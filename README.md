# Conda

在自己的conda环境里面配置单独的cuda-toolkit和TensorRT

# Windows + MX110 搭建 YOLO TensorRT 8.5 环境

本文记录在 Windows 上为 NVIDIA GeForce MX110 搭建独立 YOLO TensorRT 环境的完整过程。

目标是：

- 不修改系统 CUDA 12.1
- 不污染 Conda `base` 和其他环境
- CUDA Toolkit 11.8 安装在独立环境
- 使用 TensorRT 8.5.3.1 支持 MX110
- 导出 Ultralytics YOLO TensorRT Engine

## 已验证配置

| 组件 | 版本 |
| --- | --- |
| GPU | NVIDIA GeForce MX110 2 GB |
| Compute Capability | SM 5.0 |
| Python | 3.10 |
| PyTorch | 2.2.2+cu118 |
| Torchvision | 0.17.2+cu118 |
| CUDA Toolkit | 11.8 |
| cuDNN | 8.7 |
| TensorRT | 8.5.3.1 |
| NumPy | 1.26.4 |
| OpenCV | 4.10.0.84 |
| ONNX | 1.16.1 |
| Ultralytics | 8.4.120 |

## 为什么选择 TensorRT 8.5.3.1

MX110 的计算能力是 SM 5.0。

- TensorRT 11.x 最低要求 SM 7.5，不兼容
- TensorRT 8.6 最低要求 SM 6.0，不兼容
- TensorRT 8.5.3.1 仍可在 MX110 上构建 Engine
- TensorRT 8.5.3.1 的 Windows Python wheel 支持 Python 3.10

因此使用：

```text
TensorRT-8.5.3.1.Windows10.x86_64.cuda-11.8.zip
```

Windows 11 也可以使用这个 Windows 10 ZIP 包。

## 1. 创建 Conda 环境

打开 Anaconda PowerShell：

```powershell
conda create -n yolov python=3.10
conda activate yolov
where.exe python
python --version
```

## 2. 安装 PyTorch CUDA 11.8

Pytorch网址：https://pytorch.org/get-started/previous-versions/

```powershell
python -m pip install --no-cache-dir torch==2.2.2+cu118 torchvision==0.17.2+cu118 torchaudio==2.2.2+cu118 --index-url https://download.pytorch.org/whl/cu118
```

验证：

```powershell
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0)); print(torch.backends.cudnn.version())"
```

预期结果包括：

```text
2.2.2+cu118
11.8
True
NVIDIA GeForce MX110
8700
```

PyTorch wheel 已包含所需 cuDNN，因此不需要另外全局安装 cuDNN。

## 3. 在环境内安装 CUDA Toolkit 11.8

```powershell
conda activate yolov
conda install -c nvidia cuda-toolkit=11.8 -y
```

验证：

```powershell
where.exe nvcc
nvcc --version
python -c "import torch; print(torch.version.cuda); print(torch.cuda.is_available())"
```

预期显示：

```text
Cuda compilation tools, release 11.8
```

让当前终端优先使用环境内 CUDA：

```powershell
$env:CUDA_PATH = $env:CONDA_PREFIX
$env:PATH = "$env:CONDA_PREFIX\Library\bin;$env:CONDA_PREFIX\bin;$env:PATH"
```

验证：（关闭环境重启环境后）

```powershell
where.exe nvcc
nvcc --version
python -c "import numpy, torch; print('NumPy:', numpy.__version__); print('PyTorch:', torch.__version__); print('CUDA:', torch.version.cuda); print('cuDNN:', torch.backends.cudnn.version()); x=torch.tensor([2.,3.],device='cuda'); print((x*x).cpu().tolist())"
```

## 4. 安装基础依赖

必须固定 NumPy 1.26.4。PyTorch 2.2.2 与 NumPy 2.x 一起使用会出现 `_ARRAY_API` 或 `multiarray failed to import`。

```powershell
python -m pip install --no-cache-dir "numpy==1.26.4" "opencv-python==4.10.0.84"
python -m pip install --no-cache-dir pillow pyyaml requests scipy matplotlib psutil pandas tqdm seaborn ultralytics-thop
python -m pip install --no-cache-dir "onnx==1.16.1" "protobuf<5" "onnxslim==0.1.96"
```

如果使用本地 Ultralytics 源码：

```powershell
cd D:\users\Vsode_project\ultralytics-8.4.120
python -m pip install -e . --no-deps
```

检查依赖：

```powershell
python -m pip check
```

## 5. 下载 TensorRT

从 NVIDIA TensorRT 8.x Archive 下载：https://developer.nvidia.com/tensorrt/download

```text
TensorRT 8.5 GA Update 2
Windows 10 x86_64
CUDA 11.8
ZIP Package
```

文件名应为：

```text
TensorRT-8.5.3.1.Windows10.x86_64.cuda-11.8.zip
```

## 6. 解压到 yolov 环境

假设 ZIP 位于 `D:\Downloads`：
将其放入D:\A-APP\Anaconda\A\envs\yolov
最终目录应为：

```text
D:\A-APP\Anaconda\A\envs\yolov\TensorRT-8.5.3.1
```

## 7. 安装 TensorRT Python 接口

加载 TensorRT 路径：

```powershell
$trt = "D:\A-APP\Anaconda\A\envs\yolov\TensorRT-8.5.3.1"
$env:PATH = "$env:CONDA_PREFIX\Library\bin;$trt\lib;$trt\bin;$env:PATH"
```

查找并安装 Python 3.10 对应的 wheel：

```powershell
$wheel = Get-ChildItem "$trt\python" -Filter "tensorrt-*-cp310-none-win_amd64.whl" | Select-Object -First 1
$wheel.FullName
python -m pip install --no-cache-dir $wheel.FullName
```

最后验证：

```powershell
python -c "import tensorrt as trt; print('TensorRT:', trt.__version__)"
& "$trt\bin\trtexec.exe" --version
```

## 8. 配置激活环境时自动加载

创建 Conda 激活和退出脚本：

```powershell
$act = "$env:CONDA_PREFIX\etc\conda\activate.d"
$deact = "$env:CONDA_PREFIX\etc\conda\deactivate.d"

New-Item -ItemType Directory -Force $act, $deact
```

创建激活脚本：

```powershell
@'
$env:TENSORRT_ROOT = Join-Path $env:CONDA_PREFIX "TensorRT-8.5.3.1"
$env:PATH = "$env:TENSORRT_ROOT\lib;$env:TENSORRT_ROOT\bin;$env:PATH"
'@ | Set-Content "$act\tensorrt.ps1" -Encoding ASCII
```

创建退出脚本：

```powershell
@'
if ($env:TENSORRT_ROOT) {
    $trtLib = "$env:TENSORRT_ROOT\lib"
    $trtBin = "$env:TENSORRT_ROOT\bin"
    $env:PATH = (($env:PATH -split ";") | Where-Object {
        $_ -and $_ -ne $trtLib -and $_ -ne $trtBin
    }) -join ";"
}
Remove-Item Env:TENSORRT_ROOT -ErrorAction SilentlyContinue
'@ | Set-Content "$deact\tensorrt.ps1" -Encoding ASCII
```

关闭当前 PowerShell，重新打开后验证：

```powershell
conda activate yolov

python -c "import tensorrt as trt; print('TensorRT:', trt.__version__)"
```

## 9. 完整验证

```powershell
python -c "import numpy, cv2, torch, onnx, tensorrt; print('NumPy:', numpy.__version__); print('OpenCV:', cv2.__version__); print('PyTorch:', torch.__version__); print('CUDA:', torch.version.cuda); print('cuDNN:', torch.backends.cudnn.version()); print('GPU:', torch.cuda.get_device_name(0)); print('ONNX:', onnx.__version__); print('TensorRT:', tensorrt.__version__)"
