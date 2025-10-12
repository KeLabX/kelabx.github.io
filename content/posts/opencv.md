---
date: 2023-11-29T18:28:45+08:00
draft: false
title: 'Ubuntu 非管理员权限安装 OpenCV 与 FFmpeg'
showToc: true
TocOpen: false
---

## 背景

最近在搞一个视频相关的项目，需要用到视频处理库，目前开源的视频工具例如 moviepy，Katna 关键帧抽取库等，都依赖于 OpenCV 的 Python 环境。而 OpenCV 的 Python 库叫 OpenCV-Python，这个库需要依赖原生的 OpenCV 库，因此在使用之前需要先安装 OpenCV，一般的安装方式是使用 Linux 发行版自带的安装工具，例如 apt-get，不过在公共服务器的环境下，普通用户账号是没有权限直接全局安装 OpenCV 的，因此需要把 OpenCV 安装到当前用户路径下，本文采用的方案是从 OpenCV 的源代码开始编译，并指定安装的路径 prefix 来确保安装路径是在当前用户有执行权限的路径中。

在编译 OpenCV 之前，还需要安装 ffmpeg，ffmpeg 是流行的视频处理工具，OpenCV 中的 video-io 部分就依赖 ffmpeg，为了让编译出的 OpenCV 可以调用系统中的 ffmpeg，需要在 Ubuntu 的 pkg-config 的环境路径中添加 ffmpeg 的可执行路径。不过在公共服务器环境中安装 ffmpeg 同样存在权限的问题，由于普通用户权限不足无法直接使用安装工具安装 ffmpeg，因此也要从源码开始编译 ffmpeg。

## 无 root 权限编译 ffmpeg

部分资料显示 ffmpeg 的部分版本调整了 Api 导致 OpenCV 无法与 ffmpeg 一起编译成功，需要调整代码，不过我这次使用的都是最新版本的 OpenCV 和 ffmpeg，分别是 4.8.0 的 OpenCV 和 6.1 的 ffmpeg，在 Ubuntu 22.0 系统下成功编译，并可以在 Python 环境中进行调用。

在编译 ffmpeg 之前需要先编译 yasm，否则后续 ffmpeg 会编译失败。先前往[yasm.tortall.net/Download.html](http://yasm.tortall.net/Download.html)下载 yasm 的压缩包，由于这个库已经很久没有更新过了，所以目前最新的版本就是 1.3.0 。然后使用如下命令安装 yasm。

```bash
cd yasm
./configure --prefix=/path/to/your/yasm_build
make -j8 # 数字代表使用多少个线程并行编译
make install
```

安装完之后需要在 bash 的配置文件里添加路径，好让终端环境可以找到 yasm。

```bash
# yasm build path
export PATH=/ffmpeg/yasm_build/bin:$PATH
```

安装好 yasm 后前往 ffmpeg 的 release 页面下载 ffmpeg 的源代码压缩包文件：

![CleanShot 2023-11-29 at 16.37.47@2x.png](https://img.tanknee.cn/blogpicbed/2023/11/29/2311296566f87682ab4.png#vwid=3562&vhei=2152)

也可以使用 wget 直接下载压缩包：

```bash
wget https://ffmpeg.org/releases/ffmpeg-6.1.tar.xz
```

然后解压压缩包，得到 ffmpeg 的源代码文件夹 ffmpeg-6.1，然后在命令行中进入文件夹，配置 ffmpeg 的编译设置：

```bash
cd ffmpeg-6.1
./configure --enable-shared --prefix=/path/to/your/ffmpeg_build
```

将上面 prefix 的路径更改为当前 Linux 用户有权限读写的路径，后续将把该路径写入 pkg-config 的 PATH 中。

完成配置之后即可进行源代码的编译：

```bash
make -j64 # 根据服务器 CPU 可支持的线程数来选择数字的大小，64 个线程只需要几秒钟就能完成编译。
make install # 将编译出来的文件安装到系统中，并使用上面配置中提供的前缀路径
```

完成编译和安装之后，还需要将 ffmpeg 的路径添加到 shell 的配置文件中，bash 对应的配置文件是 .bashrc，因此需要在 .bashrc 文件中写入：

```bash
# other configurations ...
# yasm build path
export PATH=/ffmpeg/yasm_build/bin:$PATH

# 配置 ffmpeg
export PATH=/ffmpeg/ffmpeg_build/bin:$PATH
export PKG_CONFIG_PATH=/ffmpeg/ffmpeg_build/lib/pkgconfig
export LD_LIBRARY_PATH=/ffmpeg/ffmpeg_build/lib
```

到这一步 ffmpeg 应该就已经安装完成了，可以使用`ffmpeg -version`来查看是否安装成功。不过这个命令可以正确输出并不代表安装已经成功，之前我的 OpenCV 一直找不到系统环境中的 ffmpeg，导致 videoio 模块一直无法成功过启用，就是因为 pkg-config 这个工具找不到 ffmpeg，因此还需要使用`pkg-config --modversion ffmpeg`来检测是否成功。

## 非 root 用户从源码编译 OpenCV

OpenCV 的安装相对 ffmpeg 的安装更加复杂，首先从 OpenCV 的官网下载源代码压缩包：

![CleanShot 2023-11-29 at 17.08.29@2x.png](https://img.tanknee.cn/blogpicbed/2023/11/29/2311296566ff9d68e82.png#vwid=3562&vhei=2152)

下载完成后解压压缩包，进入 OpenCV 文件夹并建立一个编译结果文件夹 build：

```bash
cd opencv-4.8.0
mkdir build
cd build
```

由于 OpenCV 的许多模块已经拆分出来放到 opencv-contrib 库里，因此为了使用更多的模块可以下载 opencv-contrib 库进行一起编译，不过由于我使用到的两个库用不到这么多模块，并且服务器连接外网的速度比较感人，因此不用 opencv-contrib。

使用如下命令进行编译

```bash
cmake -D WITH_FFMPEG=ON -D BUILD_opencv_python3=yes -D BUILD_opencv_python2=no -D PYTHON3_EXECUTABLE=/anaconda3/envs/llm/bin/python3.8 -D PYTHON3_INCLUDE_DIR=/anaconda3/envs/llm/include/python3.8 -D PYTHON3_LIBRARY=/anaconda3/envs/llm/lib/libpython3.8.so -D PYTHON3_NUMPY_INCLUDE_DIRS=/anaconda3/envs/llm/lib/python3.8/site-packages/numpy/core/include -D PYTHON3_PACKAGES_PATH=/anaconda3/envs/llm/lib/python3.8/site-packages -D PYTHON_DEFAULT_EXECUTABLE=/anaconda3/envs/llm/bin/python3.8 -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/OpenCV/opencv-4.8.0 -D OPENCV_GENERATE_PKGCONFIG=ON ..
```

`WITH_FFMPEG=ON`表示要和 ffmpeg 一起编译，另外编译参数中的各类 Python 配置路径需要替换成实际的 Python 环境路径，没试过不带路径直接使用 `BUILD_opencv_python3=yes` 行不行，感觉应该也是可以的。最后要注意`CMAKE_INSTALL_PREFIX=`这个参数要改成本地实际要安装的路径参数，后续 shell 的配置文件也要用到这个路径。

在完成 cmake 的预编译之后，还需要使用 make 进行编译和安装，步骤与上面的 ffmpeg 类似：

```bash
make -j64
make install
```

完成安装之后还需要再去 `.bashrc` 文件添加路径。pkg-config 的路径使用冒号分隔，多个路径用一个环境变量就行：

```bash
# other configurations ...
# yasm build path
export PATH=/ffmpeg/yasm_build/bin:$PATH

# 配置 ffmpeg 和 OpenCV
export PATH=/ffmpeg/ffmpeg_build/bin:$PATH
export PKG_CONFIG_PATH=/ffmpeg/ffmpeg_build/lib/pkgconfig:/OpenCV/opencv-4.8.0/lib/pkgconfig
export LD_LIBRARY_PATH=/ffmpeg/ffmpeg_build/lib:/OpenCV/opencv-4.8.0/lib
```

完成以上步骤之后，可以使用`pkg-config --modversion opencv4`来检查 OpenCV 是否安装成功。之后就可以安装 OpenCV-Python 等各类语言的库来调用 OpenCV 的能力，本文中使用的所有命令都不需要 root 权限，因为所有的操作都是在当前用户的路径下进行的，不会涉及系统全局的配置，因此没有太高的风险。