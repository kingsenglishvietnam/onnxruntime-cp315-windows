# onnxruntime 1.32.0 - cp315 / CUDA 12.9 / Windows x64 (unofficial)

Unofficial CUDA-enabled onnxruntime 1.32.0 wheel for CPython 3.15 on Windows x64.

**Asset:** `onnxruntime_gpu-1.32.0-cp315-cp315-win_amd64.whl` (52,966,101 bytes)

## Requirements

- Windows 10/11 x64, CPython 3.15
- CUDA Toolkit 12.9
- cuDNN 9.x for CUDA 12.x

## Install

```powershell
uv pip install --python <target-python> --force-reinstall onnxruntime_gpu-1.32.0-cp315-cp315-win_amd64.whl
[Environment]::SetEnvironmentVariable("PATH", "$([Environment]::GetEnvironmentVariable('PATH','User'));C:\cudnn\9.28.1.4_cuda12\bin\x64", "User")
```

Verify from outside the onnxruntime source tree (import shadowing is real):

```powershell
python -c "import onnxruntime as ort; print(ort.get_available_providers())"
```

Expected: `['CUDAExecutionProvider', 'CPUExecutionProvider']`

## Caveats

- Compiled for SM 7.5 only. Rebuild for native Ampere/Ada/Blackwell kernels.
- Not tested upstream. cuDNN and CUDA are not bundled.
- See README.md and TROUBLESHOOTING.md in the repo.

