# Failure modes hit building this, and their fixes

Everything here was encountered during the build that produced the v1.0-cp315
wheel. Each section names the error verbatim so it greps well.

## `CMake Error: generator toolset: host=x64,cuda=12.9 does not match ... host=x64`

Stale cache from an earlier CPU-only configure. This is the **only** situation
where deleting the build tree is correct:

```powershell
Remove-Item -Recurse -Force C:\Programs\onnxruntime\build
```

## `The CUDA Toolkit v12.9 directory '' does not exist` (MSBuild)

`--cuda_home` reaches CMake but not MSBuild CUDA `.targets`. The MSBuild
property it looks at is `CudaToolkitDir`, an environment variable. Set it
and persist it:

```powershell
$env:CudaToolkitDir = "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.9"
[Environment]::SetEnvironmentVariable("CudaToolkitDir", $env:CudaToolkitDir, "User")
```

Without this, the first CUDA compiler ID test fails within 30 seconds of
CMake configure completing.

## `LLVM error : out of memory` on `onnxruntime_providers_cuda_llm_fp4`

FP4 QMoE kernels are Blackwell-targeted and compile-heavy. Disable them:

```powershell
--cmake_extra_defines "onnxruntime_USE_FP4_QMOE=OFF"
```

They are never dispatched on non-Blackwell hardware, so disabling them has
no functional cost.

## `LLVM error : out of memory` on other CUDA files

Peak nvcc memory exceeds commit limit. In order of impact:

1. Set a **fixed 64 GB page file** (commit limit becomes ~96 GB on 32 GB RAM)
2. `--nvcc_threads 1`
3. Drop `--parallel` to 2, then 1
4. Reduce `CMAKE_CUDA_ARCHITECTURES` to your single SM

A 6-architecture build needs 40-50 GB commit. A single-architecture build
needs ~24 GB and completes comfortably on 32 GB of physical RAM.

## `ptxas error : Value of threads per SM ... is out of range. .minnctapersm will be ignored`

A warning promoted to error by `-Werror all-warnings`. Add:

```powershell
--compile_no_warning_as_error
```

The affected kernels (`MoeGemvTcInt4Kernel`, etc.) are guarded by `sm < 80`
at dispatch and never execute on Turing. Dropping `-Werror` lets ptxas do what
its own message promised: ignore the constraint and emit valid PTX.

This is the most important flag in the whole recipe. Without it, `nvcc`
produces `ptxas fatal : Ptx assembly aborted due to errors` and the build
stops.

## `ModuleNotFoundError: No module named 'onnxruntime.capi'` after installing the wheel

Import shadowing. You are `cd`'d into the onnxruntime **source** tree, so
Python finds `.\onnxruntime\__init__.py` before `site-packages`. The source
`__init__.py` tries to import `onnxruntime.capi`, which is compiled into the
`.pyd` and only exists in the installed wheel. `cd` elsewhere.

## `No module named pip` inside a `uv tool` environment

uv-managed environments deliberately ship without pip. Use uv's own installer:

```powershell
uv pip install --python "<tool-env>\Scripts\python.exe" --force-reinstall <wheel>
```

## `GH007: Your push would publish a private email address`

GitHub blocks pushes whose commit author email matches a private email on the
account. Either make the email public at github.com/settings/emails, or use
the noreply address per repo:

```powershell
git config user.email "<username>@users.noreply.github.com"
git commit --amend --reset-author --no-edit
```

## Build is quiet for 5-10 minutes

Normal for XQA / flash-attention translation units. Confirm progress:

```powershell
(Get-ChildItem C:\Programs\onnxruntime\build\Windows\Release -Recurse -Filter *.obj).Count
```

Watch the number climb. If it is frozen **and** no `cl`/`nvcc`/`cicc` process
exists, the build is stuck. Ctrl-C and resume. Do **not** delete `build\`
unless the CMake toolset mismatch above applies — every completed `.obj` saves
recompile time on resume.

