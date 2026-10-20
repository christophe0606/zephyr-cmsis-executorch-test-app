# Alif E8 ExecuTorch inference smoke test

This application runs on M55_HP and queries the E8's general-purpose NPU-HG
(Ethos-U85, 256 MACs per cycle). `app.overlay` describes its register window and
interrupt; `prj.conf` enables the Zephyr integration and the U85 core driver.

`main()` checks `device_is_ready()`, reserves the initialized driver, and calls
`ethosu_get_hw_info()` to read the hardware ID and configuration. It prints the
driver version, hardware architecture/revision, MAC count, command stream version,
and custom DMA setting, then releases the driver before calling
`executorch_run_test()` through its C interface.

`src/executorch_test.cpp` adapts the original CMSIS-Executorch `app_main.cpp`:
it loads the model embedded in `src/model_pte.c` with `EmbeddedModule`, supplies
the same deterministic 16x16 RGB float ramp, and runs `forward()`. Input shape
and dimension order come from the model metadata: this PTE exposes NCHW
`[1, 3, 16, 16]`, while the original CMSIS runner hard-codes NHWC.
The Ethos-U backend executes the delegated work using Zephyr's initialized
driver. The runner requires one output tensor with ten float logits, prints
them, and reports `Test_result: PASS` or `Test_result: FAIL`. This is an execution
smoke test; it does not compare the logits against a numerical reference.

## Building

In VS Code, select the `DevKit-E8` target and build the CMSIS solution. For a direct
west build, run this from the west workspace root (`zephyr-repo`):

```powershell
.\.venv\Scripts\west.exe build -b ensemble_e8_dk/ae822fa0e5597ls0/rtss_hp -d build/my_app applications/my_app
```

## Local CMSIS-ExecuTorch module

`CMakeLists.txt` registers `../../../CMSIS-Executorch` with
`EXTRA_ZEPHYR_MODULES` before `find_package(Zephyr)`. The installed upstream
`executorch` checkout is supplied by west. The local module path is relative
to this application and also applies when building through the CMSIS solution.

`prj.conf` enables C++17 and `CONFIG_CMSIS_EXECUTORCH=y`, keeps
`CONFIG_EXECUTORCH=n`. `operators.conf` selects the quantize/dequantize operator
families and is merged with `prj.conf` through `EXTRA_CONF_FILE`, set in
`CMakeLists.txt` before `find_package(Zephyr)`.
The local module uses upstream ExecuTorch sources with its own build rules.
Its generated files require ExecuTorch 1.5.1 and the pinned FlatBuffers headers.

The application compiles `src/model_pte.c`, the C++ runner, and upstream
`EthosUBackend.cpp`, `EthosUBackend_Cortex_M.cpp`, `EthosUBackend_IoMemcpy.cpp`,
and `VelaBinStream.cpp`. Backend registration is retained in Zephyr's application
library. No upstream host tools or code generators run during the firmware build.

The method allocator has a 128 KiB pool in the default RAM. The temporary
allocator has a 64 KiB pool in the devicetree's `SRAM0` section so NPU scratch
uses system SRAM instead of CPU-local DTCM. That NOLOAD pool is explicitly
cleared by the runner. `prj.conf` provides a 16 KiB main stack and kernel heap,
cache maintenance for the Ethos-U driver, and floating-point printf support.

The supplied `src/model.pte` targets `ethos-u85-256`; `src/model_pte.c` has been
refreshed from that binary (the previous C array contained a U55-128 model).
When replacing the model, refresh its C array and operator selections together.
The replacement must accept one rank-four input with 768 floats and return one
tensor with ten floats, or the runner must be adapted to its interface.

See the module's [integration guide](../../../CMSIS-Executorch/documentation/zephyr-module.md)
for operator selection and the runtime API.

## Running

Use CMSIS Load, then start/reset the HP application. The image retains the
`0x80200000` link address used by the board's existing HP boot configuration.
The solution selects `debugger.load-setup.post-reset: system`. With this board
and the bundled pyOCD 0.46.0, the default hardware post-reset resumes the loader
before asserting the reset pin; this produced a transient Zephyr HardFault
(`Debug event`, `SPSEL in thread mode does not indicate PSP`) during CMSIS Load.
System reset after loading avoids that path. CMSIS Load+Run resets after loading
and again when starting Run, so two successful startup/test sequences are normal.

The board's boot firmware must have enabled access to NPU-HG (power, clock and
security); driver initialization runs before `main()`.

Read the UART4 console at 115200 baud, 8 data bits, no parity, one stop bit.
`main()` already runs in Zephyr's main thread, after the boot banner is issued.
Synchronous logging (`CONFIG_LOG_MODE_IMMEDIATE=y`) prints that banner before
the test's direct `printf()` output. With deferred logging, the banner can appear
after the test and its characters can interleave with the runtime's output.
NPU driver initialization messages can still precede the banner, since device
initialization runs before `main()`.

Successful hardware identification includes these lines, along with the versions
read from your hardware:

```text
Ethos-U85 (NPU-HG) driver communication check
Device: npu@49042000, base: 0x49042000, IRQ: 366
...
MACs per cycle: 256
Ethos-U85 driver communication OK (ID/configuration read; no inference).
ExecuTorch Ethos-U85-256 application: 8832 byte model
Input shape: [1, 3, 16, 16]
...
Output: 10 element(s): 0.0079 0.0459 0.0475 -0.0475 0.0791 0.0411 -0.0285 -0.0744 -0.2246 -0.0016
Test_result: PASS
```

The final pass marker requires successful model loading, inference, and the
expected output type and element count. The example runs once per boot.
