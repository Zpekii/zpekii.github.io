---
title: Automatically package Web app as APK
author: Zpekii
tags: [ci, docker, android, web, capacitor, alpine linux, shell, release]
categories: [ci, docker, android, alpine linux, shell]
description: 使用脚本进行自动化将Web App打包成Apk
---

# Web App打包Apk自动化(linux)

## 环境

OS: Windows 11

## 需要

- docker desktop
  - 含`jdk17`、`android cmdline-tools`的镜像
  
- 一个可正常构建的Web项目

  

## 准备docker镜像

### 准备Dockerfile

**可从docker hub 中获取相应版本的含jdk的Dockfile下载（需要科学上网）** [amazoncorretto - Official Image | Docker Hub](https://hub.docker.com/_/amazoncorretto)

以下Dockerfile是我在 https://github.com/corretto/corretto-docker/blob/cdcc44b8859544a47ce8c64ed0b3cc051a8c58c8/17/jdk/alpine/3.20/Dockerfile 的基础上做了一定修改后的Dockerfile:

```
FROM alpine:latest

ARG version=17.0.12.7.1

# Please note that the THIRD-PARTY-LICENSE could be out of date if the base image has been updated recently.
# The Corretto team will update this file but you may see a few days' delay.
RUN wget -O /THIRD-PARTY-LICENSES-20200824.tar.gz https://corretto.aws/downloads/resources/licenses/alpine/THIRD-PARTY-LICENSES-20200824.tar.gz && \
    echo "82f3e50e71b2aee21321b2b33de372feed5befad6ef2196ddec92311bc09becb  /THIRD-PARTY-LICENSES-20200824.tar.gz" | sha256sum -c - && \
    tar x -ovzf THIRD-PARTY-LICENSES-20200824.tar.gz && \
    rm -rf THIRD-PARTY-LICENSES-20200824.tar.gz && \
    wget -O /etc/apk/keys/amazoncorretto.rsa.pub https://apk.corretto.aws/amazoncorretto.rsa.pub && \
    SHA_SUM="6cfdf08be09f32ca298e2d5bd4a359ee2b275765c09b56d514624bf831eafb91" && \
    echo "${SHA_SUM}  /etc/apk/keys/amazoncorretto.rsa.pub" | sha256sum -c - && \
    echo "https://apk.corretto.aws" >> /etc/apk/repositories && \
    apk add --no-cache amazon-corretto-17=$version-r0 && \
    rm -rf /usr/lib/jvm/java-17-amazon-corretto/lib/src.zip

RUN apk update && apk add nano git unzip rsync file nodejs npm gcompat bash expect
	
ENV LANG C.UTF-8

ENV JAVA_HOME=/usr/lib/jvm/default-jvm
ENV PATH=$PATH:/usr/lib/jvm/default-jvm/bin
```

**说明:**

- 在构建镜像前，需要先拉取`alpine:latest`镜像
- 额外安装了其他应用包
  - 安装 `gcompat` 目的是为了能够在 `alpine linux` 执行`glibc`文件
  - 安装 `bash`是为了能够在 `alpine linux`下执行 `bash`文件

### 构建镜像

- 若还没有`alpine:latest`镜像，执行拉取镜像

- ```
  docker pull alpine:latest
  ```

- 在`Dockerfile`同目录下，执行构建镜像

- ```
  docker build -t alpine:jdk17 .
  ```

  **说明**: `-t` 参数表示命名镜像和打上标签, 格式 `name:tag` ;`.`表示`Dcokerfile`文件在当前目录下

- 构建成功后，可在`docker desktop`中的 `Images`下查看到创建成功的镜像

## 准备Android Cmdlinetools命令行工具(Linux版本)

- 前往官网:[下载 Android Studio 和应用工具 - Android 开发者  | Android Developers](https://developer.android.com/studio?hl=zh-cn)
- 滚动页面至看到"仅限命令行工具"标题，下载linux平台的SDK工具包

<img src="/assets/img/2024-8-19-WebAppToApk(Linux).assets/image-20240821173015076.png" alt="image-20240821173015076" />

## 创建容器(以下均在WSL下操作)

- 执行创建容器

- ```bash
  sudo docker run -it --rm --name java-shell \
  	-v /mnt:/mnt \
  	-v data:/var/data \
  	alpine:jdk17
  ```

  **说明:**

  - 这里创建的是一个一次性的容器（退出容器后自动删除），相关参数： `--rm`
  - 挂载`wsl`的 `/mnt`，便于容器从主机中获取下载好的Android cmdlinetools压缩包，相关参数: `-v /mnt:/mnt`
  - 创建成功后，便会直接进入到容器，进行交互， 相关参数: `-it`

- 通过访问`/mnt`目录找到下载到主句的Android Cmdlinetools 压缩包并拷贝至其他目录，这里我拷贝到我已创建好的目录`/var/data/kApps`下

- 创建目录 `android_sdk/cmdline-tools`

- ```sh
  mkdir android_sdk
  cd android_sdk
  mkdir cmdline-tools
  ```

- 使用`unzip`解压格式为`.zip`的Android Cmdlinetools压缩包`android_sdk/cmdline-tools`目录下，并重命名为 `latest`

- ```sh
  unzip -d android_sdk/cmdline-tools/ commandlinetools-linux-11076708_latest.zip
  cd android_sdk/cmdline-tools
  mv cmdline-tools latest
  ```

  **说明:**

  - 压缩包请以实际下载为准

## 将Web App打包成Apk

- 准备签名密钥,在`/var/data`目录下（最好是项目部署目录）创建存储密钥的目录 `/android_release/keystore`

- 在该目录下使用`keytool`创建密钥

- ```sh
  keytool -genkeypair -alias Key0 -keyalg RSA -keysize 2048 -validity 9125 -keystore KeyStore.jks -storepass mystorepwd -keypass mykeypwd
  ```

  **说明:**

  - 参数 `-alias`为密钥名缩写，可自行定义
  - 参数  `-validity` 为密钥有效天数，可自行定义
  - 参数 `-keystore` 为密钥存储文件名，可自行定义
  - 参数 `-storepass` 为密钥存储密码，可自行定义
  - 参数 `-keypass` 为密钥密码，可自行定义
  - 更多参数说明:[keytool (oracle.com)](https://docs.oracle.com/en/java/javase/11/tools/keytool.html)

- 创建密码存储文件

  - 创建名为 `KeyStorePwd.txt`文件用于存储 Keystore 密码，内容如下

    - ```
      mystorepwd
      ```

  - 创建名为 `KeyPwd.txt` 文件用于存储 Key 密码，内容如下

    - ```
      mykeypwd
      ```

- 现在在`/var/data/android_release/keystore`下有三个文件
  - KeyStore.jks
  - KeyStorePwd.txt
  - KeyPwd.txt

- 进入到前端项目目录下

- 创建名为`android_release.sh`的脚本文件，填入内容

- ```sh
  #!/bin/sh
  
  # capacitor app 名
  APPNAME="myapp"
  
  # capacitor app 包名
  PKGNAME="com.myapp.app"
  
  # 密钥路径
  KEYSTORE="/var/data/android_release/keystore"
  
  # 密钥存储文件名
  keyStore="KeyStore.jks"
  
  # 密钥存储密码存储文件
  keyStorePwd="KeyStorePwd.txt"
  
  # 密钥密码存储文件
  keyPwd="KeyPwd.txt"
  
  # apk 名
  app="myapp.apk"
  
  # 环境变量设置
  export ANDROID_SDK_ROOT="/var/data/kApps/android_sdk"
  export PATH="$ANDROID_SDK_ROOT/build-tools/34.0.0:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
  
  echo "install android 14 sdk"
  
  if [ -z "which sdkmanager" ];then
          echo "sdkmanager not found, please install android cmdline-tools!"
          exit 1
  fi
  
  # 安装android 14 sdk
  sdkmanager "platforms;android-34" || { echo "install android 14 sdk failed"; exit 1; }
  
  echo "start build..."
  
  # 构建web app
  if [ -d "build" ]; then
          cp -r build dist
  else
          npm install
          npm run build
  fi
  
  echo "remove *.gz and *.br file.."
  
  # 移除生成的 .gz 和 .br压缩文件
  find ./dist -type f \( -name "*.br" -o -name "*.gz" \) -exec rm {} +
  
  echo "install capacitor dependencies"
  
  # 安装capacitor核心包和命令行包
  npm i @capacitor/core
  npm i -D @capacitor/cli
  
  echo "init cap prj..."
  
  rm capacitor.config.ts 2> /dev/null
  
  # 初始化capacitor配置
  npx cap init $APPNAME $PKGNAME || { echo "init cap prj failed"; exit 1; }
  
  echo "add android prj..."
  
  # 安装capacitor android包
  npm install @capacitor/android
  
  rm -r android
  
  # 添加capacitor android 项目
  npx cap add android || { echo "add android failed"; exit 1; }
  
  echo "sync android prj.."
  
  # 同步到android项目
  npx cap sync
  
  echo "build apk..."
  
  cd ./android
  
  # 构建apk
  ./gradlew build || { echo "android build failed"; exit 1; }
  
  cd ./app/build/outputs/apk/release
  
  # 对齐apk
  zipalign -f -v 4 app-release-unsigned.apk app-release-unsigned-zipaligned.apk
  
  # 为apk签名
  apksigner sign --ks $KEYSTORE/$keyStore  --out $app -ks-pass file:$KEYSTORE/$keyStorePwd  --key-pass file:$KEYSTORE/$keyPwd  app-release-unsigned-zipaligned.apk
  
  echo "apk build SUCCESSFUL!"
  ```

- 给予脚本文件可执行权限

- ```sh
  chmod +x ./android_release.sh
  ```

- 执行脚本

- ```sh
  ./android_release.sh
  ```

- 若执行成功，最后会打印"apk build SUCCESSFUL!"，以及在 `/android/app/build/outputs/apk/build/release/`目录下可以看到构建好的 `myapp.apk`文件

