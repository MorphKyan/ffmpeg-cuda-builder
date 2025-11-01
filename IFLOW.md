# FFmpeg CUDA Builder

这是一个用于在 Windows 和 Linux 环境下构建支持 CUDA 硬件加速的 FFmpeg 的项目。项目通过 GitHub Actions 自动化整个构建流程，最终生成包含 CUDA 编解码器（NVENC/NVDEC）支持的 FFmpeg 二进制文件。

## 项目概述

- **核心目的**: 自动化构建一个集成了 NVIDIA CUDA 工具包的 FFmpeg 版本，以启用硬件加速的视频编解码。
- **主要技术**: 
  - **FFmpeg**: 流行的开源音视频处理框架。
  - **NVIDIA CUDA**: 用于 GPU 通用计算的平台和编程模型。
  - **GitHub Actions**: 用于自动化 CI/CD 流程。
  - **MSYS2/MINGW64**: 在 Windows 环境下提供类 Unix 编译环境。
- **架构**: 项目本身不包含源代码，而是一个 CI/CD 配置。它在每次推送或拉取请求时，会在 Windows 2022 和 Linux 的 GitHub Actions 运行器上执行一系列步骤来下载、配置和编译 FFmpeg。

## 构建与运行

该项目通过 GitHub Actions 自动运行。本地构建需要手动执行工作流中的步骤。

### Windows 构建步骤

1. **安装 MSYS2 环境**:
   - 下载并安装 [MSYS2](https://www.msys2.org/)
   - 启动 MSYS2 MinGW 64-bit 终端
   - 更新包数据库和核心系统包：
     ```bash
     pacman -Syu
     ```
   - 安装必要的编译工具和依赖：
     ```bash
     pacman -S --needed git make mingw-w64-x86_64-gcc mingw-w64-x86_64-nasm mingw-w64-x86_64-yasm mingw-w64-x86_64-pkg-config mingw-w64-x86_64-libtool mingw-w64-x86_64-autotools mingw-w64-x86_64-cmake mingw-w64-x86_64-pkgconf mingw-w64-x86_64-libass mingw-w64-x86_64-freetype mingw-w64-x86_64-lame mingw-w64-x86_64-opus mingw-w64-x86_64-libtheora mingw-w64-x86_64-libvorbis mingw-w64-x86_64-x264 mingw-w64-x86_64-x265 mingw-w64-x86_64-openssl mingw-w64-x86_64-zlib mingw-w64-x86_64-nv-codec-headers
     ```

2. **安装 NVIDIA CUDA Toolkit 11.8**:
   - 从 [NVIDIA 官网](https://developer.nvidia.com/cuda-11-8-0-download-archive) 下载 CUDA Toolkit 11.8
   - 按照安装向导完成安装

3. **设置环境变量**:
   - 在 MSYS2 终端中设置 CUDA 路径（使用短路径避免空格问题）：
     ```bash
     # 获取 CUDA 短路径（8.3 格式）
     export CUDA_PATH_SHORT=$(cmd.exe /c 'for %i in ("C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8") do @echo %~si' 2>/dev/null | tr -d '\r')
     export PATH="$CUDA_PATH_SHORT/bin:$PATH"
     ```

4. **克隆 FFmpeg 源码**:
   ```bash
   git clone https://git.ffmpeg.org/ffmpeg.git
   cd ffmpeg
   git checkout release/6.0
   ```

5. **配置 FFmpeg**:
   ```bash
   ./configure \
     --enable-nonfree \
     --enable-gpl \
     --enable-cuda-nvcc \
     --enable-cuvid \
     --enable-nvdec \
     --enable-nvenc \
     --enable-libnpp \
     --extra-cflags="-I$CUDA_PATH_SHORT/include" \
     --extra-ldflags="-L$CUDA_PATH_SHORT/lib/x64" \
     --nvccflags="-gencode arch=compute_52,code=sm_52 -O2" \
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

6. **编译与安装**:
   ```bash
   make -j$(nproc)
   make install PREFIX=/tmp/ffmpeg-build
   ```

7. **打包可执行文件**:
   ```bash
   mkdir ffmpeg-cuda-build
   cp /tmp/ffmpeg-build/bin/* ffmpeg-cuda-build/
   # 复制必要的 DLL 文件
   cp /mingw64/bin/libwinpthread-1.dll ffmpeg-cuda-build/
   cp /mingw64/bin/libgcc_s_seh-1.dll ffmpeg-cuda-build/
   cp /mingw64/bin/libstdc++-6.dll ffmpeg-cuda-build/
   # 复制 CUDA 运行时 DLL
   cp "$CUDA_PATH_SHORT/bin/cudart64_110.dll" ffmpeg-cuda-build/ || true
   cp "$CUDA_PATH_SHORT/bin/nvcuvid64_110.dll" ffmpeg-cuda-build/ || true
   ```

### Linux 构建步骤 (在 Ubuntu 20.04 上)

1. **安装依赖**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y autoconf automake build-essential cmake git-core libass-dev libfreetype6-dev libgnutls28-dev libmp3lame-dev libopus-dev libsdl2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev meson ninja-build pkg-config texinfo wget yasm zlib1g-dev
   ```

2. **安装 NVIDIA CUDA Toolkit 11.6**:
   ```bash
   wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
   sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
   sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/7fa2af80.pub
   sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/ /"
   sudo apt-get update
   sudo apt-get -y install cuda-toolkit-11-6
   ```

3. **设置环境变量**:
   ```bash
   export CUDA_PATH=/usr/local/cuda
   export PATH=/usr/local/cuda/bin:$PATH
   export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
   ```

4. **克隆 FFmpeg 源码**:
   ```bash
   git clone https://git.ffmpeg.org/ffmpeg.git
   cd ffmpeg
   git checkout release/6.0
   ```

5. **安装 `nv-codec-headers`**:
   ```bash
   git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
   cd nv-codec-headers
   make
   sudo make install
   ```

6. **配置 FFmpeg**:
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

7. **编译与安装**:
   ```bash
   make -j$(nproc)
   make install
   ```

## 开发规范

- **自动化**: 所有构建和发布流程都通过 `.github/workflows/build-ffmpeg-windows.yml` 文件定义。
- **版本**: 项目固定使用 FFmpeg `release/6.0` 分支进行构建。
- **CUDA 版本**: 
  - Windows 构建使用 CUDA Toolkit 11.8
  - Linux 构建使用 CUDA Toolkit 11.6
- **输出**: 
  - Windows 构建产物会被打包成 `ffmpeg-cuda-windows-x64.zip` 
  - Linux 构建产物会被打包成 `ffmpeg-cuda-linux-x64.tar.gz`
  - 构建产物作为 GitHub Actions 的构件上传。当代码推送到 `main` 分支时，还会自动创建一个 GitHub Release。
- **关键问题处理**: 
  - Windows 路径中的空格问题通过使用 8.3 短路径格式解决
  - 确保所有依赖项在 MSYS2 环境中正确安装
  - CUDA 运行时 DLL 需要与 FFmpeg 二进制文件一起分发