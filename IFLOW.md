# FFmpeg CUDA Builder

这是一个用于在 Linux 环境下构建支持 CUDA 硬件加速的 FFmpeg 的项目。项目通过 GitHub Actions 自动化整个构建流程，最终生成包含 CUDA 编解码器（NVENC/NVDEC）支持的 FFmpeg 二进制文件。

## 项目概述

- **核心目的**: 自动化构建一个集成了 NVIDIA CUDA 工具包的 FFmpeg 版本，以启用硬件加速的视频编解码。
- **主要技术**: 
  - **FFmpeg**: 流行的开源音视频处理框架。
  - **NVIDIA CUDA**: 用于 GPU 通用计算的平台和编程模型。
  - **GitHub Actions**: 用于自动化 CI/CD 流程。
- **架构**: 项目本身不包含源代码，而是一个 CI/CD 配置。它在每次推送或拉取请求时，会在 Ubuntu 20.04 的 GitHub Actions 运行器上执行一系列步骤来下载、配置和编译 FFmpeg。

## 构建与运行

该项目通过 GitHub Actions 自动运行。本地构建需要手动执行工作流中的步骤。

### 本地构建步骤 (在 Ubuntu 20.04 上)

1.  **安装依赖**:
    ```bash
    sudo apt-get update
    sudo apt-get install -y autoconf automake build-essential cmake git-core libass-dev libfreetype6-dev libgnutls28-dev libmp3lame-dev libopus-dev libsdl2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev meson ninja-build pkg-config texinfo wget yasm zlib1g-dev
    ```

2.  **安装 NVIDIA CUDA Toolkit 11.6**:
    ```bash
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
    sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
    sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/7fa2af80.pub
    sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/ /"
    sudo apt-get update
    sudo apt-get -y install cuda-toolkit-11-6
    ```

3.  **设置环境变量**:
    ```bash
    export CUDA_PATH=/usr/local/cuda
    export PATH=/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    ```

4.  **克隆 FFmpeg 源码**:
    ```bash
    git clone https://git.ffmpeg.org/ffmpeg.git
    cd ffmpeg
    git checkout release/6.0
    ```

5.  **安装 `nv-codec-headers`**:
    ```bash
    git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
    cd nv-codec-headers
    make
    sudo make install
    ```

6.  **配置 FFmpeg**:
    ```bash
    cd ffmpeg
    ./configure \
      --enable-nonfree \
      --enable-gpl \
      --enable-cuda-nvcc \
      --enable-cuvid \
      --enable-nvdec \
      --enable-nvenc \
      --enable-libnpp \
      --extra-cflags=-I/usr/local/cuda/include \
      --extra-ldflags=-L/usr/local/cuda/lib64 \
      --nvccflags='-gencode arch=compute_52,code=sm_52 -O2' \
      --enable-libass \
      --enable-libfreetype \
      --enable-libmp3lame \
      --enable-libopus \
      --enable-libtheora \
      --enable-libvorbis \
      --enable-libx264 \
      --enable-libx265 \
      --enable-openssl \
      --enable-shared \
      --disable-static \
      --prefix=/tmp/ffmpeg-build
    ```

7.  **编译与安装**:
    ```bash
    make -j$(nproc)
    make install
    ```

## 开发规范

- **自动化**: 所有构建和发布流程都通过 `.github/workflows/build-ffmpeg.yml` 文件定义。
- **版本**: 项目固定使用 FFmpeg `release/6.0` 分支进行构建。
- **CUDA 版本**: 项目使用 CUDA Toolkit 11.6。
- **输出**: 构建产物会被打包成 `ffmpeg-cuda-linux-x64.tar.gz` 并作为 GitHub Actions 的构件上传。当代码推送到 `main` 分支时，还会自动创建一个 GitHub Release。