---
title: 交叉编译Nginx到Android Arm
author: Zpekii
tags: [android,nginx,cross compile]
categories: [android,cross compile,nginx]
---



# Nginx Cross Compile For Android Arm

## 主机环境

- OS: Windows 11
- 使用Docker Ubuntu容器完成交叉编译过程

## 准备工作

- 源代码下载
  - Nginx: https://nginx.org/download/nginx-1.26.2.tar.gz
  - Android NDK: [NDK 下载 Android NDK Android Developers](https://developer.android.com/ndk/downloads?hl=zh-cn)
  - Openssl:[openssl/openssl: TLS/SSL and crypto library (github.com)](https://github.com/openssl/openssl)
  - PCRE-8.45: https://sourceforge.net/projects/pcre/ 正则表达式库
  -  Zlib: https://github.com/madler/zlib/ 数据压缩库
- 源代码压缩包都下载好后放在一个合适的目录下，这里演示将上述压缩包统一先放在了`B:\temp source`，即我在我电脑的可用磁盘B盘的根目录下创建了一个`temp source`文件夹，然后将下载好的压缩包全部拷贝到了这个文件夹
  - temp source 下就有一下压缩包
    - `nginx-1.26.2.tar.gz`
    - `pcre-8.45.zip`
    - `zlib-1.3.1.tar.gz`
    - `openssl-openssl-3.3.2.tar.gz`
    - `android-ndk-r27-linux.zip`
  - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925212645734.png" alt="image-20240925212645734" />

- Docker 容器准备

  - Docker Desktop下载: [Docker: Accelerated Container Application Development](https://www.docker.com/)

  - 将一下的Dockerfile内容拷贝到本地，命名文件为`Dockerfile`

    - `Dockerfile`:

    - ```dockerfile
      # 使用官方ubuntu作为父镜像
      FROM ubuntu:latest
      
      # 设置镜像维护者信息
      LABEL maintainer="Zpekii" description="预安装各种工具，用于做集成工作的ubuntu镜像"
      
      # 预安装工具
      RUN apt update && apt install -y \
              vim \
              nano \
              curl    \
              wget \
              git     \
              rsync \
              unzip \
              build-essential \
              && rm -rf /var/lib/apt/lists/*
      
      # 指定挂载卷
      VOLUME ["/data"]
      
      # 默认执行命令
      CMD ["/bin/bash"]
      ```

  - 执行 `docker pull ubuntu:latest` 拉取最新ubuntu镜像

    - ```bat
      docker pull ubuntu:latest
      ```

  - 在`Dockerfile`所在目录下执行`docker build -t ubuntu:ci .`进行创建自定义镜像

    - ```bat
      docker build -t ubuntu:ci .
      ```

  - 在源代码压缩包存放的文件夹下运行CMD命令行窗口

    - 执行`docker run --rm -it --name temp -v ".\:/mnt" ubuntu:ci bash` 创建容器

      - ```bat
        docker run --rm -it --name temp -v ".\:/mnt" ubuntu:ci bash
        ```

      - 如果创建成功则就会直接进入到容器中，(root@XXXX:/# 

        <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925213421935.png" alt="image-20240925213421935" />

  - 执行`cd /mnt`进入挂载的本地电脑目录中，

    - ```bat
      cd /mnt
      ```

  - 执行`mkdir -p /var/data/src`创建存放解压后的源代码

  - 解压源代码

    - 执行`tar -xzvf ./nginx-1.26.2.tar.gz -C /var/data/src`解压`Nginx`源代码至指定目录(`/var/data/src`)

      - ```shell
        tar -xzvf ./nginx-1.26.2.tar.gz -C /var/data/src
        ```

    - 执行`tar -xzvf ./openssl-openssl-3.3.2.tar.gz -C /var/data/src`解压`openssl`

      - ```shell
        tar -xzvf ./openssl-openssl-3.3.2.tar.gz -C /var/data/src
        ```

    - 执行`tar -xzvf ./zlib-1.3.1.tar.gz -C /var/data/src`解压`zlib`

      - ```shell
        tar -xzvf ./zlib-1.3.1.tar.gz -C /var/data/src
        ```

    - 执行`unzip ./android-ndk-r27-linux.zip -d /var/data/src` 解压android ndk

      - ```shell
        unzip ./android-ndk-r27-linux.zip -d /var/data/src
        ```

    - 执行`unzip ./pcre-8.45.zip -d /var/data/src`解压 pcre

      - ```shell
        unzip ./pcre-8.45.zip -d /var/data/src
        ```

## 交叉编译Nginx源代码

### 设置环境变量

- 执行到这一步仍保持在`/mnt`目录下

- 执行`cd /var/data/src`进入接下来的工作目录

  - ```shell
    cd /var/data/src
    ```

- 执行`ls`查看当前各源代码文件夹

  - ```shell
    ls
    ```

  - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925214547210.png" alt="image-20240925214547210" />

- 执行`export NDK=/var/data/src/android-ndk-r27`设置NDK环境变量（**注: 需要根据实际解压后的源代码文件夹名进行设置!**）

  - ```shell
    export NDK=/var/data/src/android-ndk-r27
    ```

- 执行 `export TOOLCHAINS=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin` 设置NDK工具链环境变量，便于后面的环境变量设置

  - ```shell
    export TOOLCHAINS=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin
    ```

- 执行`export CC=$TOOLCHAINS/armv7a-linux-androideabi34-clang`设置C语言编译器（C Compiler）环境变量

  - Arm 32:

    - ```shell
      export CC=$TOOLCHAINS/armv7a-linux-androideabi34-clang
      ```

  - Arm 64:

    - ```shell
      export CC=$TOOLCHAINS/aarch64-linux-android34-clang
      ```

- 执行`export CPP=$TOOLCHAINS/armv7a-linux-androideabi34-clang++`设置C++编译器环境变量

  - Arm 32:

    - ```shell
      export CPP=$TOOLCHAINS/armv7a-linux-androideabi34-clang++
      ```

  - Arm 64:

    - ```shell
      export CPP=$TOOLCHAINS/aarch64-linux-android34-clang++
      ```

- 执行`export ZLIB=/var/data/src/zlib-1.3.1`设置压缩库环境变量

  - ```shell
    export ZLIB=/var/data/src/zlib-1.3.1
    ```

- 一条指令设置所有环境变量

  - Arm 32位:

    - ```shell
      export NDK=/var/data/src/android-ndk-r27 \
      export TOOLCHAINS=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin \
      export CC=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi34-clang \
      export CPP=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi34-clang++ \
      export ZLIB=/var/data/src/zlib-1.3.1
      ```


  - Arm 64位

    - ```shell
      export NDK=/var/data/src/android-ndk-r27 \
      export TOOLCHAINS=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin \
      export CC=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android34-clang \
      export CPP=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android34-clang++ \
      export ZLIB=/var/data/src/zlib-1.3.1	
      ```


### 设置Nginx交叉编译配置

- 执行`cd nginx-1.26.2`进入Nginx源代码目录（**注: 需要根据实际解压后的源代码文件夹名进行设置!**）

  - ```shell
    cd nginx-1.26.2
    ```

- 修改部分文件内容，以确保之后可以正常进行配置

  - 执行`nano ./auto/cc/name`使用`nano`文本编辑工具修改`./auto/cc/name`文件

    - 修改`ngx_feature_run`值为`no`

      - ```
        ngx_feature_run=yes
        修改为==> ngx_feature=no
        ```

      - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925221523432.png" alt="image-20240925221523432" />

    - 修改完成按下快捷键Ctrl+X，然后按下y键确认修改，最后按回车键返回命令行

  - 执行`nano ./auto/types/sizeof`修改`./auto/types/sizeof`文件

    - 设置`ngx_size`值为4

      - ```
        ngx_size=
        修改为==>ngx_size=4
        ```

      - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925221458453.png" alt="image-20240925221458453" />

    - **如果是arm64则修改为8**

    - 注释掉原来的ngx_size=`$NGX_AUTOTEST` ,然后写上`ngx_size=4`

      - ```
        ngx_size=`$NGX_AUTOTEST`
        修改为==> ngx_size=4
        ```

      - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925221434072.png" alt="image-20240925221434072" />

    - **如果是arm64则修改为8**

    - 修改`ngx_test`

      - ```
        ngx_test="$CC $CC_TEST_FLAGS $CC_AUX_FLAGS \
                  -o $NGX_AUTOTEST $NGX_AUTOTEST.c $NGX_LD_OPT $ngx_feature_libs"
        修改为==> ngx_test="gcc $CC_TEST_FLAGS $CC_AUX_FLAGS \
                  -o $NGX_AUTOTEST $NGX_AUTOTEST.c $NGX_LD_OPT $ngx_feature_libs"
        ```

        

      - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925222338113.png" alt="image-20240925222338113" />

  - 执行`nano ./auto/options`修改

    - Ctrl+W快捷键进行搜索`PCRE_CONF_OPT`
    - 设置`PCRE_CONF_OPT`值为`--host=armv7a-linux`
    - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925221931174.png" alt="image-20240925221931174" />

- 然后执行以下指令

  - ```
    ./configure --with-http_ssl_module --with-cc=$CC --with-cpp=$CPP --with-pcre=/var/data/src/pcre-8.45 --with-openssl=/var/data/src/openssl-openssl-3.3.2 --with-zlib=$ZLIB --without-http_gzip_module --without-http_upstream_zone_module --without-stream_upstream_zone_module --with-cc-opt="-Wsign-compare"
    ```

  - 执行成功截图

    - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925222525673.png" alt="image-20240925222525673" />

### 修改部分Nginx配置文件使编译成功

- #### 执行`nano ./objs/Makefile`修改Makefile

  - 移除`Werror`
    - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925222803630.png" alt="image-20240925222803630" />

  - Crtl+W搜索`./config --prefix=/var/data/src/openssl-openssl-3.3.2/.openssl no-shared no-threads`
    - 添加`no-asm`参数
      - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925222959137.png" alt="image-20240925222959137" />

  - Ctrl+X加输入y加回车退出编辑，返回命令行

  - 执行`nano ./objs/ngx_auto_config.h`在最后添加以下代码

    - ```c
      #ifndef NGX_SYS_NERR
      #define NGX_SYS_NERR  132
      #endif
      
      #ifndef NGX_HAVE_SYSVSHM
      #define NGX_HAVE_SYSVSHM 1
      #endif
      ```

      <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925223410627.png" alt="image-20240925223410627" />

- 执行`nano ./src/os/unix/ngx_user.c`修改Nginx源代码文件`src/os/unix/ngx_user.c`

  - 引入头文件`#include <openssl/des.h>`

    - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925224311840.png" alt="image-20240925224311840" />	

  - 修改调用,Ctrl+W搜索``

    - ```
      value = crypt((char *) key, (char *) salt);
      修改为==> value = DES_crypt((char *) key, (char *) salt);
      ```

      <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925224531595.png" alt="image-20240925224531595" />

- 执行`nano ./src/os/unix/ngx_linux_config.h`给`crypt.h`引入注释
  - <img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925225104289.png" alt="image-20240925225104289" />

### 执行`make`进行编译nginx

```shell
make
```

- 执行成功截图

<img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925225145726.png" alt="image-20240925225145726" />

### 执行`make install`进行安装

- ```shell
  make install
  ```

<img src="assets/img/2024-9-25-NginxCrossCompileForAndroidArm.assets/image-20240925225225996.png" alt="image-20240925225225996" />

### 安装路径默认在`/usr/local/nginx`,至此就成功完成Nginx的交叉编译Android Arm架构版本，可执行`cp -r /usr/local/nginx /mnt`将nginx拷贝到主机文件系统

## 参考

- [交叉编译带RTMP模块的Nginx到Android Tom's Blog (zhangtom.com)](https://zhangtom.com/2017/07/11/交叉编译带RTMP模块的Nginx到Android/)
- [交叉编译Hi3536上面使用的nginx - 简书 (jianshu.com)](https://www.jianshu.com/p/5d9b60f7b262)