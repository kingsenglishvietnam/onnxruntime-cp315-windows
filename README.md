# onnxruntime 1.32.0 - cp315 / CUDA 12.9 / Windows x64 (unofficial)

Unofficial CUDA-enabled onnxruntime 1.32.0 wheel for CPython 3.15 on Windows x64.

Upstream onnxruntime 1.32.0 publishes cp311-cp314 wheels only; the cp315 packaging
PR is still open, and CUDA on 3.15 is explicitly untested upstream. This repo
documents one configuration that works end to end, and ships the resulting wheel
as a release asset.

## Requirements

- Windows 10/11 x64
- CPython 3.15
- NVIDIA driver supporting CUDA 12.9
- CUDA Toolkit 12.9 at `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.9`
- cuDNN 9.x for CUDA 12.x at `C:\cudnn\9.28.1.4_cuda12`
  (layout: `bin\x64\cudnn64_9.dll`, `include\cudnn.h`, `lib\x64\cudnn.lib`)

## Install

1. Download the wheel from the Releases page.

2. Install it into your target environment:

```powershell
uv pip install --python <target-python> --force-reinstall onnxruntime_gpu-1.32.0-cp315-cp315-win_amd64.whl
```

3. Register cuDNN on PATH so the loader finds it at runtime (persists across reboots):

```powershell
[Environment]::SetEnvironmentVariable(
  "PATH",
  "$([Environment]::GetEnvironmentVariable('PATH','User'));C:\cudnn\9.28.1.4_cuda12\bin\x64",
  "User")
```

4. Verify (run from a directory that is **not** the onnxruntime source tree):

```powershell
python -c "import onnxruntime as ort; print(ort.__version__, ort.get_available_providers())"
```

Expected output: `1.32.0 ['CUDAExecutionProvider', 'CPUExecutionProvider']`

## Import shadowing gotcha

If you run the verify command from inside a checkout of the onnxruntime source
tree, Python imports `.\onnxruntime\__init__.py` (source) instead of the installed
package and fails with:

```
ModuleNotFoundError: No module named 'onnxruntime.capi'
```

`cd` somewhere else first. This costs people hours.

## Tested configuration

| Component | Version |
|---|---|
| OS | Windows 11 Pro x64 |
| Python | 3.15.2 (system) |
| Visual Studio | 2022 Build Tools 17.14, MSVC 19.44 |
| CUDA toolkit | 12.9.86 |
| cuDNN | 9.28.1.4 (CUDA 12.x archive) |
| GPU | GTX 1660 Ti (SM 7.5, Turing) |

Compiled for `sm_75` only. Runs natively on Turing, PTX-JITs on newer GPUs.

## The four flags that make the build work

```powershell
--cmake_extra_defines "CMAKE_CUDA_ARCHITECTURES=75-real"   # 6x less work, avoids SM100 compiler bug
--cmake_extra_defines "onnxruntime_USE_FP4_QMOE=OFF"       # FP4 QMoE OOMs nvcc on 32 GB
--parallel 3 --nvcc_threads 1                              # caps per-file nvcc memory
--compile_no_warning_as_error                              # see TROUBLESHOOTING.md
```

## Adapting for other GPUs

### Blackwell (RTX 50-series, B100/B200, SM 100+)

Preferred: CUDA 13.0, which fixes the CUDA 12.9 + Windows + SM 100+ compiler bug.

- Install CUDA 13.0 and cuDNN 9.x for CUDA 13
- `--cuda_version 13.0 --cuda_home <v13.0> --cudnn_home <cudnn-13>`
- `--cmake_extra_defines "CMAKE_CUDA_ARCHITECTURES=75;86;89;90;100;120"`

Alternative: stay on CUDA 12.9. Build as this repo does; the wheel PTX-JITs on
Blackwell (works, slower).

### Ampere / Ada (SM 80, 86, 89)

Same CUDA 12.9 toolkit. Change only:

```powershell
--cmake_extra_defines "CMAKE_CUDA_ARCHITECTURES=80-real;86-real;89-real"
```

## Caveats

- Unofficial. Not affiliated with Microsoft or the ONNX Runtime project.
- cp315 is untested upstream. This is one data point: CUDA does compile on 3.15/Windows.
- Single-architecture build (SM 7.5 only).
- cuDNN and CUDA are not bundled; install them per NVIDIA terms.

## License

MIT (this repo). ONNX Runtime is MIT.

