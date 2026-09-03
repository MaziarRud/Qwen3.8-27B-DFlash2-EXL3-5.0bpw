# README_WINDOWS.md

Complete guide to building, configuring, and deploying **Qwen3.8-27B (EXL3)** with **NVFP4 KV Cache** and **DFlash2 / MTP Speculative Decoding** natively on **Windows** using **Git Bash**.

---

## 1. System & Software Prerequisites

Ensure your host system meets these exact requirements before starting:

* **OS:** Windows 10 or Windows 11 (64-bit) with **Git Bash** (installed via Git for Windows).
* **Python:** **Python 3.12.10** (64-bit Windows installer).
  * *Crucial:* **Do not use Python 3.13**. It introduces C-API breaking changes and lacks pre-compiled PyTorch wheels for CUDA extensions, leading to build failures.
* **NVIDIA CUDA Toolkit:** **v13.3** (or v12.6+) installed to standard system paths.
* **C++ Compiler:** **Visual Studio 2026 / 2022** with the following workload component checked in the Visual Studio Installer:
  * **MSVC v143 - VS 2022 C++ x64/x86 build tools** (or v144)
* **App Execution Aliases Disabled:**
  1. Open Windows **Settings** (`Win + I`) > **Apps** > **Advanced app settings** > **App execution aliases**.
  2. Toggle **OFF** both `App Installer - python.exe` and `App Installer - python3.exe`.
* **System `python3.exe` Executable Link:**
  Linux scripts explicitly invoke `python3`. Create a binary copy in your Python installation folder using Git Bash (adjust Python version path if needed):
  ```bash
  cp /c/Users/$USER/AppData/Local/Programs/Python/Python312/python.exe /c/Users/$USER/AppData/Local/Programs/Python/Python312/python3.exe
  ```

---

## 2. Environment Configuration (.env)

Create or edit the `.env` file in the root project directory:

```env
GPU_MEM_GB=22
CUDA_HOME="C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.3"
CUDA_PATH="C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.3"
```

*Note on Compiler Flags:* When using CUDA Toolkit 13.3 alongside modern Visual Studio tools, omit `NVCC_PREPEND_FLAGS="-allow-unsupported-compiler"` as NVCC 13.3 natively supports VS 2026/2022 headers.

---

## 3. Patching start.sh for Windows

Linux shell scripts expect virtual environments inside `.venv/bin/`, whereas Windows Python creates `.venv/Scripts/`. Additionally, `start.sh` must explicitly invoke Python 3.12 to prevent falling back to system Python 3.13.

Open `start.sh` in your text editor and apply these updates:

* **Force Python 3.12 & Add NTFS Junction Creation:**
  Locate the virtual environment creation logic (around line 50) and replace `python3 -m venv .venv` with:
  ```bash
  py -3.12 -m venv .venv
  cmd //c "mklink /J .venv\\bin .venv\\Scripts"
  ```
* **Script Path Alignment:**
  Keep `.venv/bin/python` and `.venv/bin/pip` references in `start.sh` as-is. The directory junction created above automatically forwards `.venv/bin` calls to `.venv/Scripts`.
* **RAM Auto-Detection Note:**
  Harmless bash warnings regarding `free: command not found` on line 117 occur because `free` is a Linux-only memory tool; it can be safely ignored as `GPU_MEM_GB=22` in `.env` overrides the budget check.

---

## 4. Environment Setup & Step-by-Step Installation

Execute the following commands sequentially in Git Bash:

```bash
# Step 1: Remove old virtual environment and compiled CUDA extension caches
rm -rf .venv ~/.cache/torch_extensions/

# Step 2: Create a fresh virtual environment locked to Python 3.12
py -3.12 -m venv .venv

# Step 3: Create NTFS Directory Junction (.venv/bin -> .venv/Scripts)
cmd //c "mklink /J .venv\\bin .venv\\Scripts"

# Step 4: Install core build tools, server dependencies, and Windows Triton support
./.venv/Scripts/pip install aiohttp ninja setuptools wheel triton-windows

# Step 5: Install custom MiaAI-Lab ExLlamaV3 fork
# (Required for NVFP4 KV cache quantization and DFlash2 / MTP support)
./.venv/Scripts/pip install git+https://github.com/MiaAI-Lab/exllamav3.git
```

---

## 5. Launching the Server

Execute `start.sh` from Git Bash:

```bash
./start.sh
```

On first startup, Ninja and MSVC will compile the custom CUDA/Triton C++ extension kernels in the background (this may take 2–5 minutes). Once complete, you will see:

```text
-- Loading tokenizer...
== model ready; accepting requests
```

The OpenAI-compatible server endpoint is now active at:
`http://localhost:8888/v1`

---

## 6. Alternative: Manual Model Download

If `start.sh` encounters network issues or fails during the automatic Hugging Face fetch step, download the model weights manually using `huggingface-cli`:

```bash
./.venv/Scripts/pip install huggingface_hub
./.venv/Scripts/huggingface-cli download Mia-AiLab/Qwen3.8-27B-EXL3-3.5bpw --local-dir models/Qwen3.8-27B-EXL3-3.5bpw
```

---

## 7. Comprehensive Troubleshooting Guide

| Issue / Symptom | Root Cause | Solution / Fix |
| :--- | :--- | :--- |
| `Python was not found... Microsoft Store` | Windows Store execution aliases intercepting `python` / `python3`. | Disable `python.exe` and `python3.exe` in **App execution aliases** settings. |
| `python3: command not found` | Windows installation lacks a `python3.exe` binary. | Run `cp /c/.../Python312/python.exe /c/.../Python312/python3.exe`. |
| `./start.sh: line 59: .venv/bin/pip: No such file or directory` | Windows uses `.venv/Scripts/` instead of `.venv/bin/`. | Run `cmd //c "mklink /J .venv\\bin .venv\\Scripts"` in project root. |
| `ModuleNotFoundError: No module named 'aiohttp'` | `start.sh` reset `.venv` with Python 3.13 or skipped dependency installation. | Patch `start.sh` to use `py -3.12` and run `./.venv/Scripts/pip install aiohttp`. |
| `ImportError: cannot import name '_dsa_attn_split_kernel'` | Missing Triton support on Windows for Dynamic Sparse Attention (DSA). | Run `./.venv/Scripts/pip install triton-windows`. |
| `ValueError: invalid literal for int() with base 10: 'nvfp4'` | Standard PyPI `exllamav3` package expects an integer bit depth (e.g., 8), not `nvfp4`. | Reinstall fork: `./.venv/Scripts/pip install git+[https://github.com/MiaAI-Lab/exllamav3.git](https://github.com/MiaAI-Lab/exllamav3.git)`. |
| `cudafe++ crash` or C++ compilation errors | Mismatch between CUDA compiler (NVCC) and MSVC host headers. | Upgrade to CUDA Toolkit 13.3 and install MSVC v143 build tools. |
| `free: command not found` warning | `start.sh` checks Linux RAM using `free`. | Safe to ignore. Provide `GPU_MEM_GB=22` in `.env` to override memory budgeting. |
