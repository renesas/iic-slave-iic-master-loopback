# IIC Slave

## Overview

This SBB demonstrates typical use of the Renesas FSP IIC Slave HAL APIs with an IIC Master and IIC Slave configured on the same RA board and connected in loopback. The reference application initializes both modules at the standard rate, alternates between Slave receive and Slave transmit operations, compares the transmitted and received buffers after each transaction, indicates results with the on-board LEDs, and reports status or errors through a serial terminal.

> **Board scope:** The supplied `readme.txt` explicitly lists **FPB-RA0L1** and **FPB-RA8E1** as supported and provides loopback connections for them. It does **not** list FPB-RA2E3, FPB-RA4E2, or FPB-RA6E2. Therefore, this document does not invent configurations for those three boards.

## 1. What use case does this SBB demonstrate?

The SBB demonstrates IIC Slave operation using an on-board loopback between an IIC Master channel and an IIC Slave channel.

After initialization, the reference application alternates between two operations:

1. **Slave read (receive):** the Master writes a six-byte buffer and the Slave receives it.
2. **Slave write (transmit):** the Master reads a six-byte buffer transmitted by the Slave.

After each completed transaction, the received buffer is compared with the transmitted buffer. The application blinks the on-board LED after a successful operation. On an application-detected failure, the error handler turns the LED on, prints the error, deinitializes the IIC modules, and enters a breakpoint trap.

## 2. What does the SBB core code do?

The core implementation is in `iic_slave_ep.c` and `iic_slave_ep.h`.

It provides the following functions:

- Initializes the configured IIC Master and IIC Slave drivers.
- Performs Slave receive and Slave transmit demonstrations.
- Starts the corresponding Master transaction that drives each loopback transfer.
- Uses Master and Slave callbacks to capture IIC events.
- Services `I2C_SLAVE_EVENT_RX_REQUEST` by calling the configured Slave read API.
- Services `I2C_SLAVE_EVENT_TX_REQUEST` by calling the configured Slave write API.
- Waits for the expected Master and Slave completion events.
- Detects aborted transfers, Slave API failures, and polling timeout conditions.
- Compares the source and destination buffers after each transfer.
- Closes the Master and Slave modules during deinitialization.
- Provides LED helper functions used by the demonstration.

The header maps the common SBB macros to the applicable FSP IIC/IICA APIs at compile time, including `R_IICA_MASTER`, `R_IICA_SLAVE`, `R_IIC_MASTER`, and `R_IIC_SLAVE` for the board families relevant to this document.

## 3. What is demo/reference application code versus reusable SBB code?

### SBB/core implementation

- `iic_slave_ep.c`
- `iic_slave_ep.h`

These files contain the IIC Master/Slave transaction logic, callbacks, buffers, event checking, driver initialization/deinitialization, and LED helpers.

### Demo/reference application

- `iic_slave_example_app.c`
- `iic_slave_example_app.h`

The reference application provides the demonstration orchestration. `iic_slave_example_app_run()`:

1. Initializes the UART console.
2. Retrieves the FSP version and prints the application banner.
3. Sets the LED off.
4. Initializes the IIC Master and Slave drivers.
5. Alternates between `i2c_slave_read()` and `i2c_slave_write()`.
6. Blinks the LED and prints a success message after each successful operation.
7. Applies a `TOGGLE_DELAY` between operations.
8. Routes failures through its private `handle_error()` function.

`hal_entry.c` is described by the application source as calling the single reference-application entry point `iic_slave_example_app_run()`.

## 4. Which public functions should I look at first?

### `fsp_err_t init_i2c_driver(void)`

Opens the configured IIC Master instance and then the configured IIC Slave instance.

### `fsp_err_t i2c_slave_read(void)`

Demonstrates the Slave receive path. It prepares a Master transmit buffer, initiates the Master write, waits for Master TX complete and Slave RX complete events, and compares the Slave receive buffer with the Master buffer.

### `fsp_err_t i2c_slave_write(void)`

Demonstrates the Slave transmit path. It prepares the Slave transmit buffer, initiates a Master read, waits for Master RX complete and Slave TX complete events, and compares the Master receive buffer with the Slave transmit buffer.

### `void deinit_i2c_driver(void)`

Closes the Master and Slave modules if they are open.

### Callbacks

The implementation also defines `i2c_master_callback()` and `i2c_slave_callback()`. The Master callback records the latest Master event. The Slave callback starts the Slave read/write API when an RX or TX request occurs and records other Slave events for transfer validation.

### Demo entry point

`void iic_slave_example_app_run(void)` is the place to start when reviewing the complete reference-application flow.

## 5. What configuration assumptions were made for the reference board?

### FPB-RA0L1

The reference configuration uses IICA channel 1 for the Master and IICA channel 0 for the Slave.

| Function | Master | Slave |
|---|---|---|
| SDA | SDAA1 P407 (J1:12) | SDAA0 P913 (J1:15) |
| SCL | SCLA1 P408 (J1:11) | SCLA0 P914 (J1:14) |

Connect Master SDA to Slave SDA and Master SCL to Slave SCL. Place jumper **J9 on pins 1-2** to use the on-board debug functionality.

The README states that external **3.9 kΩ or 4.7 kΩ pull-up resistors** are required for IIC Slave operation on boards not included in its exception list. FPB-RA0L1 is not in that exception list.

The README also states that DTC does not support the IICA module on FPB-RA0L1.

### FPB-RA8E1

The reference configuration uses IIC channel 1 for the Master and channel 0 for the Slave.

| Function | Master | Slave |
|---|---|---|
| SDA | SDA1 P511 (J4:4) | SDA0 P409 (J2:11) |
| SCL | SCL1 P512 (J4:3) | SCL0 P410 (J2:12) |

Connect Master SDA to Slave SDA and Master SCL to Slave SCL. Open jumpers **J6 and J9** to use the on-board debug functionality.

FPB-RA8E1 appears in the README's exception list for external I2C pull-up resistors.

### FPB-RA2E3, FPB-RA4E2, and FPB-RA6E2

The supplied README does not list these boards as supported by this IIC Slave example and provides no Master/Slave channel or pin mapping for them. No configuration for these boards is asserted in this document.

### Common pin assumptions

The README requires SDA and SCL to be configured with **n-channel open-drain** output type.

### DTC configuration

DTC transfer support is configurable, and the README states that DTC transfer is enabled in the FSP Configurator for this example. It also notes that DTC is helpful for minimizing interrupts during large transactions.

### Transfer size and test data

`BUF_LEN` is six bytes. The implementation initializes its example payload arrays with five explicit nonzero values; the remaining byte is zero-initialized by C array initialization.

### Serial terminal

The supplied README specifies:

- Port: J-Link On-Board serial port
- Speed: 115200
- Data: 8 bits
- Parity: none
- Stop bits: 1
- Flow control: none
- Receive newline: CR
- Transmit newline: CR
- Local echo: enabled

## 6. What must I change for my board, pins, clocks, timing, memory, task priorities, or middleware setup?

### Board and pins

The supplied example depends on a loopback between two IIC interfaces. When moving away from the supplied reference configuration, update the FSP pin/peripheral configuration and physical loopback connections for the intended Master and Slave channels. SDA and SCL must retain the open-drain configuration specified by the README, and external pull-ups must be provided where required by the target setup.

### Generated FSP instances

The core source directly references:

- `g_i2c_master_ctrl`
- `g_i2c_master_cfg`
- `g_i2c_slave_ctrl`
- `g_i2c_slave`

A project using different generated instance names must account for these references.

### Driver selection

`iic_slave_ep.h` selects between IICA, IIC_B, and IIC driver APIs according to peripheral availability and board definitions. The target project must therefore provide an FSP configuration compatible with the selected macro path.

### Timing

`TOGGLE_DELAY` is defined as 1000 and is passed to `R_BSP_SoftwareDelay()` with millisecond units both when blinking the LED and between reference-application operations.

Transfer completion uses a `uint16_t` counter initialized to `UINT16_MAX`. The supplied source does not define this counter as a specific elapsed-time duration, so it should not be interpreted as a documented millisecond or microsecond timeout.

### Buffers and memory

The core contains three static buffers, each sized by `BUF_LEN`:

- Slave transmit buffer
- Slave receive buffer
- Master buffer

`BUF_LEN` is six in the supplied implementation. Changes to the transfer size require corresponding review of the payload initialization, Master/Slave read/write calls, and buffer comparisons.

### Clocks

The README says the example initializes the IIC Master and Slave modules at the **standard rate**, but the supplied files do not state explicit peripheral clock frequencies or detailed clock-tree settings.

### Task priorities and RTOS

The supplied source does not define RTOS tasks or task priorities. No RTOS priority values are documented in these files.

### Middleware/console

The reference application uses `console/uart_console.h`, `printf()`, `uart_console_init()`, and `uart_console_flush()` for terminal output. These belong to the reference application's logging/error path and must be accounted for if that application layer is retained.

## 7. What dependencies does the SBB have on FSP, middleware, drivers, services, or hardware resources?

### FSP and generated configuration

The SBB includes `hal_data.h` and uses generated IIC Master, IIC Slave, and I/O port objects.

Depending on compile-time selection, the code maps to these IIC API families:

- `R_IICA_MASTER_*`
- `R_IICA_SLAVE_*`
- `R_IIC_MASTER_*`
- `R_IIC_SLAVE_*`
- `R_IIC_B_MASTER_*` and `R_IIC_B_SLAVE_*` for the separately handled board path present in the source

The code also uses:

- `R_BSP_SoftwareDelay()`
- `R_IOPORT_PinWrite()`
- `g_bsp_leds`
- `g_ioport`

The reference application uses `R_FSP_VersionGet()`.

### Optional/configured DTC dependency

The README states that DTC transfer support is configurable and enabled in the FSP Configurator in this example, with the explicit limitation that DTC does not support the IICA module on FPB-RA0L1.

### Reference-application console dependency

The demo uses the UART console service and standard `printf()` for status and error output.

### Hardware resources

For the supported configurations documented here, the example uses:

- Two IIC channels on the same MCU, one Master and one Slave.
- Two jumper wires for SDA and SCL loopback.
- On-board LED resources.
- J-Link on-board debug/serial functionality.
- External SDA/SCL pull-up resistors where required.

## 8. What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied files demonstrate:

- IIC Master and Slave initialization.
- Six-byte Master-to-Slave data transfer.
- Six-byte Slave-to-Master data transfer.
- Slave RX and TX request processing through the Slave callback.
- Master and Slave completion-event checking.
- Buffer comparison following successful transactions.
- LED indication following successful operations.
- Error reporting and LED-on indication in the reference application's error path.
- Alternation between Slave receive and transmit operations with a 1000 ms reference delay.

### Performance measurements

The supplied README and source files do not provide measured throughput, CPU utilization, interrupt latency, transfer latency, memory footprint, power consumption, or maximum transaction rate. The documented standard-rate configuration and 1000 ms application/LED delay are reference configuration behavior, not reported performance measurements.

## 9. What files are safe to modify and what files are generated, preset, or demo-only?

### Core SBB files

- `iic_slave_ep.c`
- `iic_slave_ep.h`

These contain the IIC transaction implementation, callbacks, public core APIs, buffers, LED helpers, event handling, and compile-time FSP API mapping.

### Demo/reference application files

- `iic_slave_example_app.c`
- `iic_slave_example_app.h`

These contain reference-application sequencing, console output, success/error presentation, and the alternating demonstration loop.

### FSP-generated content

The supplied core includes `hal_data.h` and consumes generated instances, but the attached files do not enumerate all generated project files. Peripheral, channel, pin, DTC, and callback configuration associated with generated FSP instances should be treated separately from the handwritten files documented above.

The supplied material does not explicitly label individual generated files as safe or unsafe for manual modification, so this document does not assert a file-by-file generated-code modification policy beyond identifying the direct generated configuration dependency.

## 10. What happens if I modify the source and later want to adopt a newer SBB release?

The supplied README and source files do not document a formal SBB upgrade, migration, or source-compatibility procedure.

As integration guidance, keep locally changed SBB source under version control and compare those changes with a newer SBB release before replacing the local implementation. Changes to the core APIs, callbacks, compile-time driver selection, buffer length, generated-instance references, or transfer/event logic are the areas most directly coupled to this implementation and therefore deserve explicit review during an update.

Changes limited to reference-application behavior can be kept separate from changes to the core transaction implementation, which makes the distinction between application customization and SBB integration clearer when reviewing a later release.

## Reference Application Flow

```text
hal_entry()
  -> iic_slave_example_app_run()
       -> uart_console_init()
       -> R_FSP_VersionGet()
       -> set_led(LED_OFF)
       -> init_i2c_driver()
       -> repeat
            -> i2c_slave_read()
                 Master write -> Slave RX request -> Slave read
                 wait for Master TX complete + Slave RX complete
                 compare buffers
            -> blink_led()
            -> i2c_slave_write()
                 Master read -> Slave TX request -> Slave write
                 wait for Master RX complete + Slave TX complete
                 compare buffers
            -> blink_led()
```

## Integration Checklist

- [ ] Use FPB-RA0L1 or FPB-RA8E1 when following the supplied README exactly.
- [ ] Do not assume the supplied files define FPB-RA2E3, FPB-RA4E2, or FPB-RA6E2 configurations.
- [ ] Connect the configured Master SDA/SCL pins to the corresponding Slave SDA/SCL pins.
- [ ] Configure SDA and SCL as n-channel open drain.
- [ ] Add 3.9 kΩ or 4.7 kΩ pull-ups where required by the README/reference hardware.
- [ ] Verify the FSP-generated Master and Slave instances used by the core source.
- [ ] Configure the Master and Slave callbacks used by the implementation.
- [ ] Review DTC configuration; note the documented IICA limitation on FPB-RA0L1.
- [ ] Review `BUF_LEN` and the demonstration payload when changing transfer size.
- [ ] Retain the UART console dependency only if the reference terminal-output behavior is needed.
