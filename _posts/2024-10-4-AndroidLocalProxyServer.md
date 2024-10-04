---
title: 安卓本地服务器
author: Zpekii
tags: [android,Golang,server,reverse proxy]
categories: [Golang,Android,reverse proxy]
---

# Android Local Server

## 前言:

在花了比较长时间捣鼓如何在Android 上跑Nginx以实现本地Web服务器，进行静态文件服务器以及反向代理，最终因为Nginx无法在Android 14上调用`shmget()`系统调用而放弃，然后转向尝试使用Golang去写一个这样的Web服务器，使用Golang的标准库简简单单200行代码不到就搞定了这个需求，不得不说，Golang真的挺强大的。

## Web服务器

### 项目创建

#### 使用`cobra`创建命令行应用(即可以通过在命令行界面输入相应的指令进行运行相应的应用)

- 执行`go mod init LocalServer`初始化一个Go项目

  - ```shell
    go mod init LocalServer
    ```

- 执行`go get -u github.com/spf13/cobra@latest`获取`cobra`库

  - ```shell
    go get -u github.com/spf13/cobra@latest
    ```

- 执行`cobra-cli init`创建`cobra`项目

  - ```shell
    cobra-cli init
    ```

- 执行`cobra-cli add serve`添加`serve`运行服务器指令

  - ```shell
    cobra-cli add serve
    ```

### 配置文件读取

#### 创建配置文件

- 在项目根目录下创建`cfg_android.json`文件

  - 内容如下: 其中`targetHost`为指定的远程主机

  - ```json
    {
        "appServe": {
    		"host": "localhost",
    		"port": 8080
    	},
        "workDir":"/data/data/com.example.www/files/local_server/server",
    	"targetHost": "www.example.com:8080"
    
    }
    ```

- 接着在项目根目录下创建`config`包

  - `config.go`内容如下:

  - ```go
    package config
    
    import (
    	"fmt"
    	"os"
    	"path/filepath"
    	"runtime"
    
    	"github.com/spf13/viper"
    	"go.uber.org/zap"
    )
    
    // GetConfig 读取配置文件中的指定配置信息
    func GetConfig(key string) string {
    	// 获取调用者信息(包名/文件名:行数)
    	_, file, line, ok := runtime.Caller(1)
    	if !ok {
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("zap.L() create logger failed, please check zap logger")
    			os.Exit(-1)
    		}
    		logger.Error("获取调用者信息失败,处理调用失败")
    
    		return ""
    	}
    
    	callFrom := fmt.Sprintf("%s/%s:%d", filepath.Base(filepath.Dir(file)), filepath.Base(file), line)
    
    	// 检查参数是否为空
    	if key == "" {
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("zap.L() create logger failed, please check zap logger")
    			os.Exit(-1)
    		}
    		logger.Error("'" + callFrom + "'" + "调用时传入的参数为空,参数示例: server.host")
    		return ""
    	}
    
    	// 读取配置文件
    	viper.SetConfigName("cfg_android")
    	viper.AddConfigPath(".")
    	viper.AddConfigPath("..")
    	viper.AddConfigPath("../..")
    	viper.AddConfigPath("../../..")
    	viper.AddConfigPath("/data/data/com.example.www/files/local_server/server")
    	viper.SetConfigType("json")
    
    	err := viper.ReadInConfig()
    	if err != nil {
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("zap.L() create logger failed, please check zap logger")
    			os.Exit(-1)
    		}
    		logger.Error("读取配置文件失败,处理" + "'" + callFrom + "'" + "的调用失败,错误详情:" + err.Error())
    		return ""
    	}
    
    	// 检查配置文件中是否有指定配置信息
    	if !viper.IsSet(key) {
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("zap.L() create logger failed, please check zap logger")
    			os.Exit(-1)
    		}
    		logger.Error("配置文件中没有找到指定的配置信息(" + key + "),请检查(" + "'" + callFrom + "'" + ")调用时传入参数是否正确,参数示例: server.host")
    		return ""
    	}
    
    	// 读取指定配置信息
    	config := viper.GetString(key)
    
    	logger := zap.L()
    	if logger == nil {
    		fmt.Println("zap.L() create logger failed, please check zap logger")
    		os.Exit(-1)
    	}
    
    	logger.Info("成功向" + "'" + callFrom + "'" + "返回(" + key + ")的值:" + config)
    
    	return config
    }
    
    ```

### 日志输出配置

- 日志输出使用`zap`标准库

- 在项目根目录下创建`log`包

  - `log.go`内容如下:

  - ```go
    package log
    
    import (
    	"fmt"
    	"os"
    	"regexp"
    
    	"go.uber.org/zap"
    	Conf "android_local_server/config"
    )
    
    func LoggerInit() error {
    	config := zap.NewDevelopmentConfig()
    	config.DisableStacktrace = true
    	
    	//检测是否有目录
    	r, _ := regexp.Compile(`(.*\/)(.*)`)
    
    	logsPath := Conf.GetConfig("workDir") + "/logs/"
    
    	matches := r.FindStringSubmatch(logsPath) //此处用配置文件获取，最后一定要有'/'
    	fmt.Println(matches)
    	
    	// 检查目录是否存在
    	if _, err := os.Stat(matches[1]); os.IsNotExist(err) {
    		
    		// 目录不存在，创建目录
    		err := os.MkdirAll(matches[1], os.ModePerm)
    		if err != nil {
    			fmt.Printf("无法创建目录：%v\n", err)
    			return err
    		}
    		fmt.Println("目录已创建：")
    	} else {
    		// 目录已存在
    		fmt.Println("目录已存在：")
    	}
    	
    	// 日志输出文件
    	config.OutputPaths = []string{"stdout", logsPath + "/logs.txt"}
    	
    	//设置日志级别
    	level := zap.DebugLevel
    	config.Level = zap.NewAtomicLevelAt(level)
    	config.Encoding = "json"
    	Logger, err := config.Build()
    	if err != nil {
    		return err
    	}
    	zap.ReplaceGlobals(Logger) // 替换全局Logger
    	Logger.Info("log init success")
    	return nil
    }
    
    ```

- 接着在项目根目录下`cmd`文件夹中的`serve.go`进行调用日志初始化函数

  - `serve.go`内容如下:

  - ```go
    /*
    Copyright © 2024 NAME HERE <EMAIL ADDRESS>
    */
    package cmd
    
    import (
    	"fmt"
    	"os"
    
    	"local_server/log"
    
    	"github.com/spf13/cobra"
    	"go.uber.org/zap"
    )
    
    // serveCmd represents the serve command
    var serveCmd = &cobra.Command{
    	Use:   "serve",
    	Short: "A brief description of your command",
    	Long: `A longer description that spans multiple lines and likely contains examples
    and usage of using your command. For example:
    
    Cobra is a CLI library for Go that empowers applications.
    This application is a tool to generate the needed files
    to quickly create a Cobra application.`,
    	Run: func(cmd *cobra.Command, args []string) {
    		fmt.Println("serve called")
    		
            // -----在这里添加命令执行操作-----
            // 初始化日志配置
    		if err := log.LoggerInit(); err != nil {
    			fmt.Printf("logger init error:%v", err)
    			os.Exit(-1)
    		}
    
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("logger is nil while it's shouldn't")
    			os.Exit(-1)
    		}
    
    	},
    }
    
    func init() {
    	rootCmd.AddCommand(serveCmd)
    
    	// Here you will define your flags and configuration settings.
    
    	// Cobra supports Persistent Flags which will work for this command
    	// and all subcommands, e.g.:
    	// serveCmd.PersistentFlags().String("foo", "", "A help for foo")
    
    	// Cobra supports local flags which will only run when this command
    	// is called directly, e.g.:
    	// serveCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
    }
    
    ```

    

### 静态文件返回服务器

- 在项目根目录下创建`server`包

  - `server.go`内容:

  - ```go
    package server
    
    import (
        "mime"
    	"net/http"
    	"os"
    	"path/filepath"
    
    	"local_server/config"
    
    	"go.uber.org/zap"
    )
    
    func Run() {
    
        logger := zap.L().Sugar()
    
        // 获取当前工作目录
        currentWorkDir := config.GetConfig("workDir"); 
        if(currentWorkDir == ""){
            logger.Error("获取当前工作目录失败")
            return
        }
    
    	
        // 处理静态文件返回
        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    		staticDir := currentWorkDir + "/static"
           
            // 获取请求文件路径
    		filePath := filepath.Join(staticDir, r.URL.Path)
    		
            // 检查文件是否存在
    		_, err := os.Stat(filePath)
    		if os.IsNotExist(err) || r.URL.Path == "/" {
    			
                // 文件不存在或请求路径为根路径，返回根目录下的 index.html
    			http.ServeFile(w, r, filepath.Join(staticDir, "index.html"))
    			
    			return
    		}
    
    		// 手动设置 Content-Type
    		ext := filepath.Ext(filePath)
    
    		contentType := mime.TypeByExtension(ext)
    
    		if contentType != "" {
    			w.Header().Set("Content-Type", contentType)
    		}
    
    		// 返回请求的静态文件
    		http.ServeFile(w, r, filePath)
    	})
    
    	serverHost := config.GetConfig("appServe.host")
    	if(serverHost == ""){
    		serverHost = "localhost"
    	}
    
    	serverPort := config.GetConfig("appServe.port")
    	if(serverPort == ""){
    		serverPort = "8080"
    	}
    
        // 启动服务器
        logger.Info("Serving static files on "+ serverHost + ":" + serverPort)
        err := http.ListenAndServe(serverHost + ":" + serverPort, nil)
        if err != nil {
            logger.Fatal("Failed to start server: ", err)
        }
    }
    ```

- 接在在根目录下`cmd`中的`serve.go`添加调用运行函数

  - `serve.go`内容如下:

  - ```go
    /*
    Copyright © 2024 NAME HERE <EMAIL ADDRESS>
    */
    package cmd
    
    import (
    	"fmt"
    	"os"
    
    	"local_server/log"
    	"local_server/server"
    
    	"github.com/spf13/cobra"
    	"go.uber.org/zap"
    )
    
    // serveCmd represents the serve command
    var serveCmd = &cobra.Command{
    	Use:   "serve",
    	Short: "A brief description of your command",
    	Long: `A longer description that spans multiple lines and likely contains examples
    and usage of using your command. For example:
    
    Cobra is a CLI library for Go that empowers applications.
    This application is a tool to generate the needed files
    to quickly create a Cobra application.`,
    	Run: func(cmd *cobra.Command, args []string) {
    		fmt.Println("serve called")
    		
            // -----在这里添加命令执行操作-----
            // 初始化日志配置
    		if err := log.LoggerInit(); err != nil {
    			fmt.Printf("logger init error:%v", err)
    			os.Exit(-1)
    		}
    
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("logger is nil while it's shouldn't")
    			os.Exit(-1)
    		}
    
    		// 运行静态文件返回服务
    		server.Run()
    
    	},
    }
    
    func init() {
    	rootCmd.AddCommand(serveCmd)
    
    	// Here you will define your flags and configuration settings.
    
    	// Cobra supports Persistent Flags which will work for this command
    	// and all subcommands, e.g.:
    	// serveCmd.PersistentFlags().String("foo", "", "A help for foo")
    
    	// Cobra supports local flags which will only run when this command
    	// is called directly, e.g.:
    	// serveCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
    }
    
    ```

### 反向代理

- 在项目根目录下创建`proxy`包

  - `proxy.go`内容如下:

  - ```go
    package proxy
    
    import (
    	"fmt"
    	"net/http"
    	"os"
    
    	"local_server/config"
    	"net/http/httputil"
    	"net/url"
    
    	"go.uber.org/zap"
    )
    
    
    
    
    const (
    	httpScheme = "https"
    	wsScheme   = "wss"
    )
    
    
    // ProxyManager反向代理服务
    type ProxyManager struct {
    	
    	//日志
    	logger *zap.Logger
    	
    	//反向代理
    	reverseProxy *httputil.ReverseProxy
    	
    	//目标服务器地址
    	targetHost   string
    }
    
    var srv *ProxyManager
    
    func NewProxy() *ProxyManager {
    	
    	logger := zap.L()
    	if logger == nil {
    		fmt.Println("NewProxy logger is nil while it's shouldn't")
    		os.Exit(-1)
    	}
    	
    	//获取目标服务器地址
    	targetHost := config.GetConfig("targetHost")
    	if targetHost == "" {
    		logger.Error("get targetUrlHttps from config failed:targetUrlHttps is empty")
    		return nil
    	}
    
    
    	targetUrl := httpScheme + "://" + targetHost
    	logger.Info(targetUrl)
    	
    	url, err := url.Parse(targetUrl)
    	if err != nil {
    		logger.Error("parse targetUrl failed:" + err.Error())
    		return nil
    	}
    
    	// 创建反向代理, 并且设置请求的目标地址
    	proxy := &httputil.ReverseProxy{
    		Director: func(req *http.Request) {
    			req.URL.Scheme = url.Scheme //设置请求的协议
    			req.URL.Host = url.Host	//设置请求的主机地址
    			req.Host = url.Host 	//设置请求的主机地址
    		},
    	}
    
    	return &ProxyManager{
    		logger:       logger,
    		reverseProxy: proxy,
    		targetHost:   targetHost,
    	}
    }
    
    func (srv *ProxyManager) ProxyClientHttpRequest(w http.ResponseWriter, r *http.Request) {
    	if srv.logger == nil {
    		fmt.Println("ProxyClientHttpRequest logger is nil while it's shouldn't")
    		os.Exit(-1)
    	}
    	srv.logger.Info("-->local_server/proxy/proxy.go:func ProxyClientHttpRequest")
    
    	srv.reverseProxy.ServeHTTP(w, r)
    
    	srv.logger.Info("ProxyClientHttpRequest end")
    }
    
    func Run() {
    	
    	// 创建反向代理服务
    	srv = NewProxy()
    	if srv == nil {
    		fmt.Println("ProxyManager server is nil while it's shouldn't")
    		os.Exit(-1)
    	}
    	
    	srv.logger.Info("ProxyManager server is running")
    	
    	//处理http请求,转发到目标服务器
    	http.Handle("/api/", http.HandlerFunc(srv.ProxyClientHttpRequest))
    	
    	//处理websocket请求,转发到目标服务器,在ServerHTTP中会处理websocket升级,所以不需要特别处理
    	http.Handle("/msg/", http.HandlerFunc(srv.ProxyClientHttpRequest))
    }
    
    
    ```

- 在`cmd`中`serve.go`中添加反向代理服务运行

  - `serve.go`内容如下:

  - ```go
    /*
    Copyright © 2024 NAME HERE <EMAIL ADDRESS>
    */
    package cmd
    
    import (
    	"fmt"
    	"os"
    
    	"local_server/log"
    	"local_server/server"
    	"local_server/proxy"
    
    	"github.com/spf13/cobra"
    	"go.uber.org/zap"
    )
    
    // serveCmd represents the serve command
    var serveCmd = &cobra.Command{
    	Use:   "serve",
    	Short: "A brief description of your command",
    	Long: `A longer description that spans multiple lines and likely contains examples
    and usage of using your command. For example:
    
    Cobra is a CLI library for Go that empowers applications.
    This application is a tool to generate the needed files
    to quickly create a Cobra application.`,
    	Run: func(cmd *cobra.Command, args []string) {
    		fmt.Println("serve called")
    		
            // -----在这里添加命令执行操作-----
            // 初始化日志配置
    		if err := log.LoggerInit(); err != nil {
    			fmt.Printf("logger init error:%v", err)
    			os.Exit(-1)
    		}
    
    		logger := zap.L()
    		if logger == nil {
    			fmt.Println("logger is nil while it's shouldn't")
    			os.Exit(-1)
    		}
    
    		// 在运行静态文件返回服务之前，先启动反向代理服务
    		proxy.Run()
    
    		// 运行静态文件返回服务
    		server.Run()
    
    	},
    }
    
    func init() {
    	rootCmd.AddCommand(serveCmd)
    
    	// Here you will define your flags and configuration settings.
    
    	// Cobra supports Persistent Flags which will work for this command
    	// and all subcommands, e.g.:
    	// serveCmd.PersistentFlags().String("foo", "", "A help for foo")
    
    	// Cobra supports local flags which will only run when this command
    	// is called directly, e.g.:
    	// serveCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
    }
    
    ```

### 编译

#### 准备

- Android NDK 下载: [NDK 下载 Android NDK Android Developers (google.cn)](https://developer.android.google.cn/ndk/downloads?hl=zh-cn)

#### 编译脚本(Linux下运行)

- **注意:**
  - **下载好的NDK需解压到`/var/data/src`下才可执行以下脚本**
  - **NDK需替换使用实际下载的版本**
  - `Mac OS` 需要通过在`Android Studio`中下载NDK,并修改脚本相关变量(`NDK_HOME`、`TOOLCHAIN`路径)

- `build_x86-64.sh`

  - ```shell
    export NDK_HOME=/var/data/src/android-ndk-r27
    export TOOLCHAIN=$NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin
    export CGO_ENABLED=1
    export GOOS=android
    export GOARCH=amd64
    export CC=$TOOLCHAIN/x86_64-linux-android34-clang
    export CPP=$TOOLCHAIN/x86_64-linux-android34-clang++
    
    output=server_x86-64
    
    go build -o $output .
    ```

- `build_arm64.sh`

  - ```shell
    export NDK_HOME=/var/data/src/android-ndk-r27
    export TOOLCHAIN=/var/data/src/android-ndk-r27/toolchains/llvm/prebuilt/linux-x86_64/bin
    export CGO_ENABLED=1
    export GOOS=android
    export GOARCH=arm64
    export CC=$TOOLCHAIN/aarch64-linux-android34-clang
    export CPP=$TOOLCHAIN/aarch64-linux-android34-clang++
    
    output=server_arm64
    
    go build -o $output .
    ```

## 在Android下运行

### 嵌入Android项目

- 将编译好的可执行二进制文件(`server_x86-64`或`server_arm64`)、配置文件(`cfg_android.json`)拷贝到安卓项目`app/src/main/assets/`下，将这些文件统一放进一个自行创建的`server`目录下，并在`server`目录下创建`logs/log.txt`日志输出文件(程序没有权限在Android下创建目录)
  - 在`app/src/main/assets/server`下就有
    - `server_x86-64` 或 `server_arm64`
    - `cfg_android.json`
    - `logs`
      - `log.txt`
- 以及将构建好的web项目拷贝到`app/src/main/assets/`下，需要自行在创建一个目录`static`，及在`app/src/main/assets/static`下就有`index.html`web引入html

### 运行代码示例

- `app/main/res/values/styles.xml`

  - ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <resources>
    
        <!-- Base application theme. -->
        <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
            <!-- Customize your theme here. -->
            <item name="colorPrimary">@color/colorPrimary</item>
            <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
            <item name="colorAccent">@color/colorAccent</item>
        </style>
    
        <style name="AppTheme.NoActionBar" parent="Theme.AppCompat.DayNight.NoActionBar">
            <item name="windowActionBar">false</item>
            <item name="windowNoTitle">true</item>
            <item name="android:background">@null</item>
        </style>
    
    
        <style name="AppTheme.NoActionBarLaunch" parent="Theme.SplashScreen">
            <item name="android:background">@drawable/splash</item>
        </style>
    
        <!--   自定义显示主题: 全屏沉浸显示     -->
        <style name="FullscreenTheme" parent="AppTheme">
            <item name="android:windowFullscreen">true</item>
            <item name="android:windowNoTitle">true</item>
            <item name="android:windowContentOverlay">@null</item>
            <item name="android:windowActionBar">false</item>
            <item name="android:windowLayoutInDisplayCutoutMode">always</item>
        </style>
    </resources>
    ```

- `app/main/AndroidManifest.xml` 安卓清单

  - ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <manifest xmlns:android="http://schemas.android.com/apk/res/android">
    
        <application
            android:allowBackup="true"
            android:icon="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:roundIcon="@mipmap/ic_launcher_round"
            android:supportsRtl="true"
            android:extractNativeLibs="true"
            android:usesCleartextTraffic="true"
            android:theme="@style/AppTheme">
    
            <activity
                android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|smallestScreenSize|screenLayout|uiMode"
                android:name=".MainActivity"
                android:label="@string/app_name"
                android:theme="@style/FullscreenTheme"
                android:launchMode="singleTask"
                android:exported="true"
                android:windowLayoutInDisplayCutoutMode="always">
    
                <intent-filter>
                    <action android:name="android.intent.action.MAIN" />
                    <category android:name="android.intent.category.LAUNCHER" />
                </intent-filter>
    
            </activity>
    
            <provider
                android:name="androidx.core.content.FileProvider"
                android:authorities="${applicationId}.fileprovider"
                android:exported="false"
                android:grantUriPermissions="true">
                <meta-data
                    android:name="android.support.FILE_PROVIDER_PATHS"
                    android:resource="@xml/file_paths"></meta-data>
            </provider>
    
            <!-- 服务声明 -->
            <service android:name=".LocalServer"/>
    
        </application>
    
        <!-- Permissions -->
    
        <uses-permission android:name="android.permission.INTERNET" />
        <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
        <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
        <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
        <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CAMERA"/>
    
    </manifest>
    
    ```

- `app/src/main/java/com.example.www/`下有

  - `MainActivity.java`app入口类

    - 执行步骤:

      - 设置显示配置: 横屏、全屏显示、无系统UI沉浸显示
      - 运行本地服务器服务
      - 注册服务器运行广播消息监听，传入webview实例

    - 代码:

    - ```java
      package com.example.www;
      
      
      import android.content.BroadcastReceiver;
      import android.content.DialogInterface;
      import android.content.Intent;
      import android.content.IntentFilter;
      import android.content.SharedPreferences;
      import android.content.pm.ActivityInfo;
      import android.content.pm.PackageManager;
      import android.content.res.Configuration;
      import android.os.AsyncTask;
      import android.os.Build;
      import android.os.Bundle;
      import android.os.Handler;
      import android.os.Looper;
      import android.util.Log;
      import android.view.View;
      import android.view.WindowInsets;
      import android.view.WindowInsetsController;
      import androidx.annotation.NonNull;
      import androidx.annotation.RequiresApi;
      import androidx.core.app.ActivityCompat;
      import androidx.core.content.ContextCompat;
      import android.Manifest.permission;
      import android.view.WindowManager;
      import android.app.AlertDialog;
      import android.webkit.WebResourceRequest;
      import android.widget.ScrollView;
      import android.widget.TextView;
      
      
      import java.io.BufferedReader;
      import java.io.IOException;
      import java.io.InputStream;
      import java.io.InputStreamReader;
      import java.net.HttpURLConnection;
      import java.net.URL;
      
      import androidx.appcompat.app.AppCompatActivity;
      
      import android.webkit.WebSettings;
      import android.webkit.WebView;
      import android.webkit.WebViewClient;
      import android.widget.Toast;
      
      public class MainActivity extends AppCompatActivity {
          private static final int PERMISSION_REQUEST_CODE = 100;
      
      
      
          @Override
          protected void onCreate(Bundle savedInstanceState) {
              super.onCreate(savedInstanceState);
      
              getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
      
              if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                  getWindow().getAttributes().layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
              }
      
      
              requestPermission();
      
              setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE);
      
              // 隐藏系统 UI
              if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                  hideSystemUI();
              }
      
              getSupportActionBar().hide();
      
              // 启动 LocalServer 服务
              Intent intent = new Intent(this, LocalServer.class);
              startService(intent);
      
              WebView myWebView = new WebView(this);
      
              WebSettings webSettings = myWebView.getSettings();
              webSettings.setJavaScriptEnabled(true); // 启用JavaScript
              webSettings.setDomStorageEnabled(true); // 启用web localStorage,是网页可以获取并写入localStorage
      
              setContentView(myWebView);
              myWebView.setWebViewClient(new WebViewClient());
      
              ServerBroadcastReceiver serverBr = new ServerBroadcastReceiver(); // 创建服务广播
      
              IntentFilter intentFilter = new IntentFilter("com.example.www.Server_Start"); // 服务器运行广播消息
      
              WebView.setWebContentsDebuggingEnabled(true); // 启用web debug,可以通过chrome://inspect进行监视并进行调试
      
              serverBr.setWebView(myWebView);
      
              registerReceiver(serverBr,intentFilter); // 注册接受指定广播消息
          }
      
                  @Override
                  protected void onPostExecute(Boolean canConnect) {
                      if (canConnect) {
                          callback.onConnectionSuccess();
                      } else {
                          callback.onConnectionFailed();
                      }
                  }
              }.execute();
          }
      
          interface ServerConnectionCallback {
              void onConnectionSuccess();
              void onConnectionFailed();
          }
      
          @Override
          protected void onDestroy() {
              super.onDestroy();
              // 停止后台服务
              Intent stopIntent = new Intent(this, LocalServer.class);
              stopService(stopIntent);
          }
      
          public void onConfigurationChanged(@NonNull Configuration newConfig) {
              super.onConfigurationChanged(newConfig);
              // 处理配置更改，例如屏幕方向变化
              if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                  hideSystemUI();
              }
          }
      
          @RequiresApi(api = Build.VERSION_CODES.R)
          private void hideSystemUI() {
              // 获取当前窗口的装饰视图
              View decorView = getWindow().getDecorView();
      
              // 获取 WindowInsetsController
              WindowInsetsController insetsController = decorView.getWindowInsetsController();
              if (insetsController != null) {
                  // 设置系统 UI 行为
                  insetsController.setSystemBarsBehavior(WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
      
                  // 隐藏手势条
                  insetsController.hide((WindowInsets.Type.systemGestures()));
      
                  // 隐藏系统栏：状态栏、导航栏
                  insetsController.hide((WindowInsets.Type.systemBars()));
                  
              }
          }
      
          private boolean hasPermissions(String[] permissions) {
              for (String permission : permissions) {
                  if (ContextCompat.checkSelfPermission(this, permission) != PackageManager.PERMISSION_GRANTED) {
                      return false;
                  }
              }
              return true;
          }
      
          private void requestPermission() {
              String[] requirePermissions = {
                      permission.INTERNET,
                      permission.ACCESS_NETWORK_STATE,
                      permission.FOREGROUND_SERVICE,
                      permission.FOREGROUND_SERVICE_CAMERA
              };
      
              if(!hasPermissions(requirePermissions)){
                  ActivityCompat.requestPermissions(this,requirePermissions, 0);
              }else{
      
              }
      
      
          }
      
          @Override
          public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
              super.onRequestPermissionsResult(requestCode, permissions, grantResults);
              if (requestCode == PERMISSION_REQUEST_CODE) {
                  if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                      // 权限被授予，设置屏幕方向为横向
                      setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
                  } else {
                      // 权限被拒绝，显示提示信息或采取其他措施
                      // 这里可以显示一个提示对话框，告知用户需要该权限
                  }
              }
          }
      
      
      }
      
      
      
      ```

      

  - `LocalServer.java`运行服务器可执行二进制文件类

    - 执行步骤:

      - 将`assets`下的web静态文件、可执行二进制文件、配置文件拷贝到`/data/data/com.example.www/files/local_server/server`下
        - 文件结构:
        - `/data/data/com.example.www/files/local_server/server`:
          - `static` : web构建项目
            - 包含`index.html`
          - `server_arm64` / `server_x86-64`
          - `cfg_android.json`
          - `logs`
            - `log.txt`
      - 给予目录`/data/data/com.example.www/files/local_server/server`777 权限
      - 执行`/data/data/com.example.www/files/local_server/server/server_arm64 serve` 或 `/data/data/com.example.www/files/local_server/server/server_x86-64 serve`运行服务器

    - 代码:

    - ```java
      package com.example.www;
      
      import android.annotation.SuppressLint;
      import android.app.Notification;
      import android.app.NotificationChannel;
      import android.app.NotificationManager;
      import android.app.PendingIntent;
      import android.app.Service;
      import android.content.Intent;
      import android.content.res.AssetManager;
      import android.os.Build;
      import android.os.IBinder;
      import android.util.Log;
      
      import androidx.core.app.NotificationCompat;
      
      import com.getcapacitor.plugin.WebView;
      
      import java.io.BufferedReader;
      import java.io.File;
      import java.io.FileOutputStream;
      import java.io.IOException;
      import java.io.InputStream;
      import java.io.InputStreamReader;
      import java.nio.Buffer;
      
      public class LocalServer extends Service {
      
          private static final String TAG = "LocalServer";
          private static final String CHANNEL_ID = "LocalServerChannel";
          private static final int NOTIFICATION_ID = 1;
          private Process serverProcess;
          private String pkgPath = "/local_server";  // 服务器工作目录, 默认为 /data/data/com.example.www/files/local_server/server
          private String serverBin = "server_x86-64"; // 后端服务器运行可执行二进制文件, server_arm64或server_x86-64
          private Thread serverThread;
      
          @Override
          public void onCreate() {
              super.onCreate();
              Log.i(TAG,"Start Local Server");
              createNotificationChannel();
              startForegroundService();
              copyAssets();
              startServer();
          }
      
          @Override
          public void onDestroy() {
              super.onDestroy();
              if (serverProcess != null) {
                  serverProcess.destroy();
                  Log.i(TAG, "Server process destroyed");
              }
      
              if (serverThread != null) {
                  serverThread.interrupt();
                  Log.i(TAG, "Server thread interrupted");
              }
              Log.i(TAG, "Local Server stopped");
          }
      
          @Override
          public IBinder onBind(Intent intent) {
              return null;
          }
      
          private void createNotificationChannel() {
              if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                  NotificationChannel serviceChannel = new NotificationChannel(
                          CHANNEL_ID,
                          "Local Server Channel",
                          NotificationManager.IMPORTANCE_DEFAULT
                  );
                  NotificationManager manager = getSystemService(NotificationManager.class);
                  if (manager != null) {
                      manager.createNotificationChannel(serviceChannel);
                  }
              }
          }
      
          @SuppressLint("ForegroundServiceType")
          private void startForegroundService() {
              Intent notificationIntent = new Intent(this, MainActivity.class);
              PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, PendingIntent.FLAG_IMMUTABLE);
      
              Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                      .setContentTitle("Local Server")
                      .setContentText("Running local server...")
                      .setContentIntent(pendingIntent)
                      .setOngoing(true)
                      .build();
      
              startForeground(NOTIFICATION_ID, notification);
          }
      
          private void copyAssets() {
              AssetManager assetManager = getAssets();
              String[] assets = {"server", "public"};
              String outputDirPath = "/data/data/com.example.www/files" + "/local_server";
              pkgPath = outputDirPath;
      
              for (String asset : assets) {
                  String targetName = asset.equals("public") || asset.equals("static") ? "/server/static" : "/"+asset;
                  copyAsset(assetManager, asset, outputDirPath, targetName);
              }
          }
      
          private void copyAsset(AssetManager assetManager, String assetName, String outputDirPath, String targetName) {
              String outputFilePath = outputDirPath + targetName;
      
              File outputDir = new File(outputDirPath);
              if (!outputDir.exists()) {
                  if (!outputDir.mkdirs()) {
                      Log.e(TAG, "Failed to create directory: " + outputDirPath);
                      return;
                  }
              }
      
              try {
                  String[] files = assetManager.list(assetName);
                  if (files != null && files.length > 0) {
                      // It's a directory
                      File subDir = new File(outputFilePath);
                      if (!subDir.exists()) {
                          if (!subDir.mkdirs()) {
                              Log.e(TAG, "Failed to create directory: " + outputFilePath);
                              return;
                          }
                      }
                      for (String file : files) {
                          copyAsset(assetManager, assetName + "/" + file, outputDirPath, targetName + "/" + file);
                      }
                  } else {
                       // It's a file
                       try (InputStream is = assetManager.open(assetName);
                       FileOutputStream fos = new FileOutputStream(outputFilePath)) {
      
                          byte[] buffer = new byte[1024];
                          int length;
                          while ((length = is.read(buffer)) > 0) {
                              fos.write(buffer, 0, length);
                          }
      
                          Log.d(TAG, "File copied to: " + outputFilePath);
      
                          // 设置文件可执行权限
                          if (assetName.equals(serverBin)) {
                              File serverFile = new File(outputFilePath);
                              if (!serverFile.setExecutable(true)) {
                                  Log.e(TAG, "Failed to set executable permission for: " + outputFilePath);
                              }
                          }
      
                       }
                  }
              } catch (IOException e) {
                  Log.e(TAG, "Failed to copy asset: " + assetName, e);
              }
          }
      
          private void startServer() {
              String outputFilePath = pkgPath + "/server/" + serverBin;
              serverThread = new Thread(() -> {
                  try {
      
                      // 检测是否存在./logs目录
                      File logsDir = new File(pkgPath + "/server/logs");
      
                      if (!logsDir.exists()) {
                          if (!logsDir.mkdirs()) {
                              Log.e(TAG, "Failed to create directory: " + pkgPath + "/server/logs");
                              return;
                          }
      
                      }
      
                      //创建logs.txt日志文件
                      File logFile = new File(pkgPath + "/server/logs/logs.txt");
      
                      if (!logFile.exists()) {
                          if (!logFile.createNewFile()) {
                              Log.e(TAG, "Failed to create file: " + pkgPath + "/server/logs/logs.txt");
                              return;
                          }
                      }
      
                      // 读取cfg_android.json文件，打印到日志
                      File cfgFile = new File(pkgPath + "/server/cfg_android.json");
      
                      if (!cfgFile.exists()) {
                          Log.e(TAG, "Failed to find file: " + pkgPath + "/server/cfg_android.json");
                          return;
                      }
      
                      try (BufferedReader reader = new BufferedReader(new InputStreamReader(cfgFile.toURI().toURL().openStream()))) {
                          String line;
                          while ((line = reader.readLine()) != null) {
                              Log.d(TAG, "CFG: " + line);
                          }
                      }
      
      
                      executeCommand("chmod -R 777 "+ pkgPath + "/server",true);
                      serverProcess = executeCommand(outputFilePath+ " serve", false);
      
                      Intent intent = new Intent();
                      intent.setAction("com.example.www.Server_Start");
      
                      sendBroadcast(intent);
      
                  } catch (Exception e) {
                      Log.e(TAG, "Failed to start server: " + outputFilePath, e);
                  }
              });
      
              serverThread.start();
              Log.i(TAG, "Server Thread Run");
      
      
      
          }
      
          private Process executeCommand(String command, boolean enableWaitFor) {
              try {
                  Log.i(TAG,"execute command: "+command);
      
                  Process process = Runtime.getRuntime().exec(command);
      
                  // 打印标准输出
                  new Thread(() -> {
                      try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
                          String line;
                          while ((line = reader.readLine()) != null) {
                              Log.d(TAG, "STDOUT: " + line);
                          }
                      } catch (IOException e) {
                          Log.e(TAG, "Failed to read standard output", e);
                      }
                  }).start();
      
                  // 打印错误输出
                  new Thread(() -> {
                      try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getErrorStream()))) {
                          String line;
                          while ((line = reader.readLine()) != null) {
                              Log.e(TAG, "STDERR: " + line);
                          }
                      } catch (IOException e) {
                          Log.e(TAG, "Failed to read error output", e);
                      }
                  }).start();
      
                  // 等待进程完成并获取退出码
                  if(enableWaitFor){
                      int exitCode = process.waitFor();
                      Log.i(TAG, "exited with code: " + exitCode);
                  }
      
      
                  return process;
              } catch (Exception e) {
                  Log.e(TAG, "Failed to execute command: " + command, e);
                  return null;
              }
          }
      }
      ```

  - `ServerBroadcastReceiver.java`: 服务广播消息接受

    - 执行步骤:

      - 监听指定广播消息
      - 等待适当时间(等待服务器运行完毕)
      - 通过`WebView`访问本地服务器监听端口，渲染网页

    - 代码:

    - ```java
      package com.example.www;
      
      import android.content.BroadcastReceiver;
      import android.content.Context;
      import android.content.Intent;
      import android.os.Handler;
      import android.os.Looper;
      import android.util.Log;
      import android.webkit.WebView;
      
      public class ServerBroadcastReceiver extends BroadcastReceiver {
      
          private static final String TAG = "ServerBroadcastReceiver";
          private WebView webView;
      
          @Override
          public void onReceive(Context context, Intent intent) {
              StringBuilder sb = new StringBuilder();
              sb.append("Action: " + intent.getAction() + "\n");
              String log = sb.toString();
              Log.d(TAG, log);
      
              switch (intent.getAction()){
                  case "com.example.www.Server_Start":
                      if(webView != null){
      
                          // 使用 Handler(Looper.getMainLooper()) 延时加载 URL，确保服务器已启动
                          new Handler(Looper.getMainLooper()).postDelayed(() -> {
                              Log.i(TAG,"display webview");
                              webView.loadUrl("http://127.0.0.1:8080/");
                          }, 500);
      
                      }else{
                          Log.e(TAG,"webview is null!");
                      }
                      break;
                  default:
                      Log.e(TAG,"unknown Action:"+intent.getAction());
                      break;
              }
          }
      
          public void setWebView(WebView webViewInput){
              webView = webViewInput;
          }
      
      }
      
      ```

      