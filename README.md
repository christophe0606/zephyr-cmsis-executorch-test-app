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
it loads the model embedded in `ai_layer/model_pte.c` with `EmbeddedModule`, supplies
the same deterministic 16x16 RGB float ramp, and runs `forward()`. Input shape
and dimension order come from the model metadata: this PTE exposes NCHW
`[1, 3, 16, 16]`, while the original CMSIS runner hard-codes NHWC.
The Ethos-U backend executes the delegated work using Zephyr's initialized
driver. The runner requires one output tensor with ten float logits, prints
them, and reports `Test_result: PASS` or `Test_result: FAIL`. This is an execution
smoke test; it does not compare the logits against a numerical reference.

## Building

Populate the workspace dependencies as described below before building.

In VS Code, select the `DevKit-E8` target and build the CMSIS solution. For a direct
west build, run this from the west workspace root (`zephyr-repo`):

```powershell
.\.venv\Scripts\west.exe build -b ensemble_e8_dk/ae822fa0e5597ls0/rtss_hp -d build/my_app applications/my_app
```

## Shared west manifest and dependencies

[`../west.yml`](../west.yml), at `zephyr-repo/applications/west.yml`, is the
root manifest for the whole workspace, including every application under
`applications/`. It imports the module definitions from Zephyr v4.4.0 and
declares these application dependencies without modifying `zephyr/west.yml`:

- `executorch`, pinned to the existing commit
  `3b60683923245cf472b7323426920e15623ba361` (reports version 1.5.1) at
  `modules/lib/executorch`. Only its
  `third-party/flatbuffers` submodule is requested; the other upstream
  submodules are not needed for this firmware integration.
- `cmsis-executorch`, pinned to commit
  `3f4f09f8567a204cd5727e12a5f4bdb7fe941d18` from the `zephyr` branch of
  `git@github.com:christophe0606/CMSIS-Executorch.git`, at
  `modules/lib/cmsis-executorch`. Fetching this module uses GitHub SSH access.

For another application using this runtime, keep both entries in the shared
manifest. For additional dependencies, add a `projects` entry with its unique
`name`, repository `url`, pinned `revision`, and workspace-relative `path`.
Also enable that exact project name in `.west/config`'s
`manifest.project-filter`. Adding a project to the manifest alone does not
enable it with this workspace's filter.

The workspace configuration selects the shared manifest and limits updates to
the explicitly enabled projects:

```ini
[manifest]
path = applications
file = west.yml
project-filter = -.*,+zephyr,+cmsis,+cmsis_6,+hal_ethos_u,+hal_alif,+executorch,+cmsis-executorch
```

`hal_alif` is retained in the filter for compatibility with the existing
configuration; the pinned Zephyr manifest currently declares no project with
that name. A filter entry does not create or download a project on its own.

After changing the manifest or setting up this layout, run the following from
`zephyr-repo` (the workspace root):

```powershell
.\.venv\Scripts\west.exe update -n
```

This is required to install `cmsis-executorch` in its new workspace location.
The previous sibling checkout at `../CMSIS-Executorch` is no longer registered
by the application. The shared manifest's module metadata makes west-managed
modules discoverable by Zephyr without an `EXTRA_ZEPHYR_MODULES` entry in the
application's CMake file.

The project filter controls which repositories are updated. `-n` (`--narrow`)
reduces fetching to the requested revision and avoids fetching all tags; it
does not select fewer projects. West's default smart fetching reuses pinned
tags and commits already present locally. The three projects declared directly
in this manifest also use `clone-depth: 1` to limit history fetched when needed.
These settings do not remove submodules or history that were downloaded earlier.

Once the Zephyr manifest import has been initialized, you can update only the
new module instead of all enabled projects:

```powershell
.\.venv\Scripts\west.exe update -n cmsis-executorch
```

If the upstream ExecuTorch checkout is also missing, include it explicitly:

```powershell
.\.venv\Scripts\west.exe update -n executorch cmsis-executorch
```

For the first build after moving the module, select a pristine rebuild so an
old CMake cache cannot retain the sibling module path. With the CMSIS solution,
use its clean/rebuild operation; for a direct west build use `west build -p always`
with the board, build-directory and application arguments shown above.

See [west update](https://docs.zephyrproject.org/latest/develop/west/built-in.html#west-update)
and [manifest imports](https://docs.zephyrproject.org/latest/develop/west/manifest.html#manifest-imports)
for the update and dependency-resolution behavior.

## CMSIS-ExecuTorch module configuration

`prj.conf` enables C++17 and the board's Ethos-U driver. The AI layer owns the
ExecuTorch selections: `ai_layer/components.conf` enables the CMSIS module,
disables the upstream runtime build, and selects Runtime, Kernel Utils, Kernel
Registration, and Backend EthosU. It also sets
`CONFIG_CMSIS_EXECUTORCH_LOG_ENABLED=n`, equivalent to the original CMSIS AI
layer's `ET_LOG_ENABLED: 0`. Set it to `y` to enable ExecuTorch diagnostics;
Zephyr and Ethos-U driver logging have their own application settings.
`ai_layer/operators.conf` selects the quantize/dequantize operator families.
Both fragments are merged with `prj.conf` through `EXTRA_CONF_FILE`, set by
`ai_layer/CMakeLists.txt` before `find_package(Zephyr)`.
The CMSIS module uses upstream ExecuTorch sources with its own build rules.
Its generated files require ExecuTorch 1.5.1 and the pinned FlatBuffers headers.

The application compiles `ai_layer/model_pte.c` and the C++ runner. The module
compiles the selected runtime, utilities, operator registration, CPU operators,
and the four Ethos-U backend sources. Zephyr's module library retains the
backend registration constructor; no backend source paths are needed in the
application CMake. No upstream host tools or code generators run during the
firmware build.

The method allocator has a 128 KiB pool in the default RAM. The temporary
allocator has a 64 KiB pool in the devicetree's `SRAM0` section so NPU scratch
uses system SRAM instead of CPU-local DTCM. That NOLOAD pool is explicitly
cleared by the runner. `prj.conf` provides a 16 KiB main stack and kernel heap,
cache maintenance for the Ethos-U driver, and floating-point printf support.

The supplied `ai_layer/model.pte` targets `ethos-u85-256`; `ai_layer/model_pte.c` has been
refreshed from that binary (the previous C array contained a U55-128 model).
When replacing the model, refresh its C array and operator selections together.
The replacement must accept one rank-four input with 768 floats and return one
tensor with ten floats, or the runner must be adapted to its interface.

See the module's [integration guide](../../modules/lib/cmsis-executorch/documentation/zephyr-module.md)
for operator selection and the runtime API.

## Reusing the AI layer

The `ai_layer` directory contains `CMakeLists.txt`, `components.conf`,
`operators.conf`, `model.pte`, and its embedded C array/header
(`model_pte.c` / `model_pte.h`). An enclosing
Zephyr application integrates these files with one call **before** Zephyr setup:

```cmake
cmake_minimum_required(VERSION 3.20.0)
# Install the runtime dependencies through the shared west manifest first.
add_subdirectory(ai_layer)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)
target_sources(app PRIVATE src/main.c)
```

The layer appends its absolute `components.conf` and `operators.conf` paths to
the existing `EXTRA_CONF_FILE` list and propagates it to the parent. It also supplies its model
sources and header directory through the `ai_layer` interface target, linked to
`app` automatically at the end of the enclosing `CMakeLists.txt`. No additional
`target_sources()`, include path, or link command is needed for the models.
The layer selects the CMSIS-ExecuTorch runtime and its backend. The enclosing
project obtains the Zephyr modules through the shared manifest and provides C++17 support, the inference
runner, and board-specific driver, devicetree and memory settings.

For future generated layers with several models, list each embedded C source in
`ai_layer/CMakeLists.txt`, use unique symbols and headers for each model, and put
the union of their required shared components/backends in
`ai_layer/components.conf` and operators in `ai_layer/operators.conf`. The `.pte`
files accompany the generated arrays; they are not themselves CMake sources.
The enclosing application's `add_subdirectory(ai_layer)` call stays the same.

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
