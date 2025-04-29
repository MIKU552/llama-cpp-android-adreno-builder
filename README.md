# 自动化 Android llama.cpp 构建器 (Adreno OpenCL + mbedTLS)

[![Build Status](https://github.com/MIKU552/llama-cpp-android-adreno-builder/actions/workflows/build_android_mbedtls.yml/badge.svg)](https://github.com/MIKU552/llama-cpp-android-adreno-builder/actions/workflows/build_android_mbedtls.yml)

## 概述

本仓库使用 GitHub Actions 自动化构建 [llama.cpp](https://github.com/ggerganov/llama.cpp) 项目，专门为运行在搭载高通 Adreno GPU 的 Android 设备上进行优化。

构建流程每日自动触发，并将编译产物发布到本仓库的 Releases 页面。

## 特性

当前工作流构建的 `llama.cpp` 包含以下特性：

* **目标平台:** Android
* **目标 ABI:** `arm64-v8a` (64位 ARM)
* **目标 API Level:** 28 (或根据工作流配置)
* **GPU 加速:** 启用 OpenCL 后端 (`GGML_OPENCL=ON`)，针对 Qualcomm Adreno GPU 进行了优化。
* **网络支持:** 包含 CURL 支持，并静态链接了 **mbed TLS** 作为 SSL/TLS 后端，以支持 HTTPS 功能。
* **静态链接:** 依赖库（mbed TLS, libcurl）和 `llama.cpp` 本身编译为静态库/可执行文件，以减少运行时依赖。

## 工作流

核心的自动化构建流程定义在 `.github/workflows/build_android.yml` 文件中。

* **触发器:**
    * 每日定时触发（北京时间凌晨 4:00）。
    * 支持手动触发 (`workflow_dispatch`)。
* **主要步骤:**
    1.  设置 Ubuntu Runner 环境。
    2.  安装必要的构建依赖（CMake, Ninja, NDK, Git, Python3 等）。
    3.  下载并安装指定版本的 Android NDK。
    4.  准备 OpenCL 相关文件（头文件、编译 ICD Loader）。
    5.  交叉编译 **mbed TLS** for Android (包括其子模块)。
    6.  交叉编译 **libcurl** for Android，并配置其使用上一步编译好的 mbed TLS。
    7.  克隆最新的 `llama.cpp` 源代码。
    8.  使用 NDK 工具链配置并编译 `llama.cpp`，启用 OpenCL 并链接到编译好的 libcurl 和 mbed TLS。
    9.  将编译生成的可执行文件打包成 `.zip` 压缩包。
    10. 创建或更新 GitHub Release (使用固定标签 `daily-android-opencl-adreno-mbedtls`) 并上传打包好的 `.zip` 文件作为附件。

## 如何获取构建产物

每日自动构建的最新产物可以在本仓库的 [Releases 页面](https://github.com/MIKU552/llama-cpp-android-adreno-builder/releases) 找到。

* 通常会有一个标签为 `daily-android-opencl-adreno-mbedtls` 的 Release。
* 这个 Release 会被每日构建覆盖更新（标记为 `pre-release`）。
* 每个 Release 包含一个 `.zip` 附件，例如 `llama-android-opencl-mbedtls-YYYY-MM-DD.zip`，解压后即可获得编译好的 `llama.cpp` 相关可执行文件。

## 使用方法 (示例)

获取到 Release 中的 `.zip` 附件 (例如 `llama-android-opencl-mbedtls-YYYY-MM-DD.zip`) 并解压后，你会得到包含以下主要可执行文件的目录（我们假设你解压到了 `llama_bin`）：

* `llama-cli`
* `llama-server`
* `llama-quantize`
* `llama-embedding`
* *(其他辅助工具)*

你可以通过 `adb` 将这些文件推送到你的 Android 设备上运行（通常 `/data/local/tmp/` 是一个可用的临时目录）：

```bash
# 解压下载的 zip 文件
unzip llama-android-*.zip -d llama_bin

# 将解压后的 bin 目录中的所有内容推送到设备
# (注意：根据你的连接和权限，可能需要逐个推送或调整目标路径)
adb push llama_bin/. /data/local/tmp/llama_bin/

# 进入设备的 shell
adb shell

# 导航到目标目录
cd /data/local/tmp/llama_bin

# 给予需要的可执行文件执行权限
chmod +x llama-cli llama-server llama-quantize llama-embedding

# 运行指定的程序，例如 'llama-cli' 进行推理
# 请确保将 <模型文件路径> 替换为模型在设备上的实际路径
# 并根据需要添加其他参数 (如 -ngl 层数等)
LD_LIBRARY_PATH=LD_LIBRARY_PATH=/vendor/lib64:$LD_LIBRARY_PATH ./llama-cli -m <模型文件路径> [其他参数...]

# 或者运行 'llama-server' 启动服务
# LD_LIBRARY_PATH=LD_LIBRARY_PATH=/vendor/lib64:$LD_LIBRARY_PATH ./llama-server -m <模型文件路径> -c 2048 [其他服务器参数...]
```
同样的，可以使用Termux直接在手机上运行，方法大同小异，不再过多赘述。