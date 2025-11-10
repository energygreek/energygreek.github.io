---
draft: true
comments: true
date: 2025-11-03
tags: []
---

# run vllm Qwen3 locally

* torch==2.5.1+cu121  https://mirrors.aliyun.com/pytorch-wheels/cu121/torch-2.5.1+cu121-cp313-cp313-linux_x86_64.whl
* vllm_pascal-0.7.2   https://github.com/sasha0552/pascal-pkgs-ci/releases/download/wheels/vllm_pascal-0.7.2-cp38-abi3-linux_x86_64.whl
* python 3.9
* uv 0.9.0
* Tesla P40
* Ubuntu 22

## cuda版本
系统的cuda开发编译环境（/usr/local/cuda）可以不安装，除非需要本地编译。为了避免编译，就下载whl包。 whl包是别人用nvcc编译的二进制包，所以固定了cuda版本。例如torch-2.5.1+cu121表示用了cuda 12.1的编译工具编译的。
而且为了vllm能调用pytorch, vllm就必须和pytorch一样的cu版本，所以我两个都下载了cu121。

## 安装vllm_pascal后，还需要执行下面的命令，来生成一个新的vllm 包
npx 相当于npm -g, 来安装一个全局的包+cli。
```
npx install transient-package
transient-package create --source vllm --source-version 0.7.2 --target vllm-pascal --target-version 0.7.2 --output-directory .
uv pip install vllm-0.7.2-py3-none-any.whl
```
如果没有这一步, vllm命令提示找不到cuda
```
WARNING 03-14 23:02:38 init.py:26] The vLLM package was not found, so its version could not be inspected. This may cause platform detection to fail.
INFO 03-14 23:02:38 init.py:211] No platform detected, vLLM is running on UnspecifiedPlatform
0.7.2
```

## 安装其他依赖
`uv pip install ./*.whl`命令不会安装依赖，运行一个一个这样报缺包再安装包很烦，所以直接到vllm的代码里找requrement,一次安装。还有到pytorch官网看对应torch==2.5.1的torch-audio torch-vision包。

## 验证和验证和修改参数
* 参数类型不匹配，模型配置文件里设定了`"torch_dtype": "bfloat16"`, 而显卡只能支持float16, 所以修改了模型配置。
* 输出token长度太大，导致vllm不支持， 修改模型配置"max_position_embeddings"，从40960改成了10240。 

##  Install CUDA 11.7 toolkit and driver
sudo apt install -y cuda-11-7

## uv install 和 uv add 的区别
新项目应该用 uv add, 在用 uv add 之前，需要先执行
* uv init 初始化项目
* uv venv ./.venv 创造虚拟环境

## nvidia-smi 输出
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |

* Driver Version 是GPU硬件驱动
* CUDA Version 是最大支持的cuda toolkits，也叫运行时。而实际的cuda toolkit version 通过`nvcc --version`来确认

## nvcc 
nvcc 是/usr/local/cuda/bin/下的，也就是cuda toolkits，cuda目录通常是软链接到实际的cuda toolkits
```
lrwxrwxrwx  1 root root   22 Nov  3 18:23 cuda -> /etc/alternatives/cuda
lrwxrwxrwx  1 root root   25 Nov  3 18:23 cuda-11 -> /etc/alternatives/cuda-11
drwxr-xr-x 15 root root 4096 Nov  3 18:23 cuda-11.7
```
上面的cuda和cuda-11 都链接到cuda-11.7 也就是实际的toolkit version, 并且通过`nvcc --version`确认版本
nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2022 NVIDIA Corporation
Built on Wed_Jun__8_16:49:14_PDT_2022
Cuda compilation tools, release 11.7, V11.7.99
Build cuda_11.7.r11.7/compiler.31442593_0

11.7 小于 nvidia-smi打印的 13.0, 所以这个toolkit是支持的。

## cuda capabilities
虽然GPU driver 可以向前兼容，所以 Driver 535可以替代Driver 515。但是硬件架构限定了cuda capabilities，无法向前兼容的。例如Tesla P40的硬件架构是Pascal sm_6.1，而pytorch 的2.7.0 版本不支持sm_6.1了。

## Install pytorch
因为 cuda capability限制，知道最高版本支持sm 6.1的pytorch版本是2.6.0, 而pytroch有专门对cuda特定版本的二进制包，在官网上能找到 https://pytorch.org/get-started/previous-versions/  
下面可见，2.6.0版本有 cuda-11.8 cuda-12.4 cuda-12.6版本的二进制安装包。 
```
# ROCM 6.1 (Linux only)
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/rocm6.1
# ROCM 6.2.4 (Linux only)
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/rocm6.2.4
# CUDA 11.8
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cu118
# CUDA 12.4
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cu124
# CUDA 12.6
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cu126
# CPU only
pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cpu

```
然后搜索得知 11.8 要求最低的cuda capability 版本是 5.0, 所以也支持6.1的Tesla P40, 于是安装安装命令是
```
uv pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cu118
```

## Install vllm
vllm 0.9 开始不支持pytorch 2.6了，所以只能用vllm 0.8.5。从githup release找到了二进制安装包 vllm-0.8.5.post1+cu118-cp38-abi3-manylinux1_x86_64.whl  
下载到本地后运行  uv pip install ~/vllm-0.8.5.post1+cu118-cp38-abi3-manylinux1_x86_64.whl

### vllm serve Qwen3-8B fail: RuntimeError: CUDA error: no kernel image is available for execution on the device
可惜，vllm 0.8.5 不支持 pytorch-cuda-6.1, 不知道这跟pytroch 支持 sm-6.1有什么区别。好在可以手动编译vllm来支持
```
git clone https://github.com/vllm-project/vllm.git
cd vllm
git checkout v0.8.5
export TORCH_CUDA_ARCH_LIST="6.1"
pip install -e . --no-build-isolation
```

### 

## 计算VRAM(Video RAM)
dtype 是参数类型，所需的大小分别是
* FP32  4B 单精度
* BP16  2B 双精度
* FP16  2B 单精度
* INT8 1B
* INT4 0.5B 量化
vllm 可以运行时指定量化

而大模型的参数常见有32B，7B。7B的大模型，使用FP32时，需要VRAM=4*7G。 使用FP16时，只需要2*7G。

***注意***，模型的config.torch_dtype是可以改的，最大输出token也是可以改的。

## 参考

https://github.com/vllm-project/vllm/pull/4290

pytroch 2.7 不支持 6.1: https://www.reddit.com/r/LocalLLaMA/comments/1lrerwe/pytorch_27x_no_longer_supports_pascal_architecture/
vllm 0.11.5 才能支持Qwen/Qwen3-VL-32B-Instruct，否则报错 "'Qwen3VLConfig' object has no attribute 'vocab_size'" https://huggingface.co/Qwen/Qwen3-VL-32B-Instruct/discussions/3
