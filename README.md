# Alif E8 Ethos-U85 driver check

This application runs on M55_HP and queries the E8's general-purpose NPU-HG
(Ethos-U85, 256 MACs per cycle). `app.overlay` describes its register window and
interrupt; `prj.conf` enables the Zephyr integration and the U85 core driver.

`main()` checks `device_is_ready()`, reserves the initialized driver, and calls
`ethosu_get_hw_info()` to read the hardware ID and configuration. It prints the
driver version, hardware architecture/revision, MAC count, command stream version,
and custom DMA setting, then releases the driver. No model is needed.

## Building

In VS Code, select the `DevKit-E8` target and build the CMSIS solution. For a direct
west build, run this from the west workspace root (`zephyr-repo`):

```powershell
.\.venv\Scripts\west.exe build -b ensemble_e8_dk/ae822fa0e5597ls0/rtss_hp -d build/my_app applications/my_app
```

## Running

Use CMSIS Load, then start/reset the HP application. The image retains the
`0x80200000` link address used by the board's existing HP boot configuration.
The board's boot firmware must have enabled access to NPU-HG (power, clock and
security); driver initialization runs before `main()`.

Read the UART4 console at 115200 baud, 8 data bits, no parity, one stop bit.
Successful hardware identification includes these lines, along with the versions
read from your hardware:

```text
Ethos-U85 (NPU-HG) driver communication check
Device: npu@49042000, base: 0x49042000, IRQ: 366
...
MACs per cycle: 256
Ethos-U85 driver communication OK (ID/configuration read; no inference).
```

This checks initialization and register access through the driver. An inference
is still needed to exercise DMA, command execution, and interrupt completion.
