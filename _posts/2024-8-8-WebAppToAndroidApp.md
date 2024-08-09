---
title: Web App To Android native App
author: Zpekii
tags: [android,web,ionic,capacitor,svelte]
categories: [dev,android,web,ionic,capacitor]
description: 如何使用capaitor将web app 封装成android app,即封装成可安装的apk
---

# Web APP到Android 原生APP

## 前言:

### 环境:`OS:Windows 11`

### 前端框架:`Sveltekit`

## 需求/准备:

- 一个可正常构建的项目(即可正常通过执行 `npm run build`构建并打包成一个`build`或`dist`的输出目录)
- `nodejs`
- `android studio`
- `android sdk`
- `vscode ide`（可选）

## 安装`android studio`并安装`android sdk`

### 访问官网:[下载 Android Studio 和应用工具 - Android 开发者  | Android Developers](https://developer.android.com/studio?hl=zh-cn)

### 下载`android studio`

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220041619.png" alt="image-20240808220041619" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220100867.png" alt="image-20240808220100867" />

### 安装并启动`android studio`

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220820112.png" alt="image-20240808220820112" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220843176.png" alt="image-20240808220843176" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220849897.png" alt="image-20240808220849897" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220855438.png" alt="image-20240808220855438" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220904573.png" alt="image-20240808220904573" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808220915832.png" alt="image-20240808220915832" />



### 打开`SDK manager`

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221237189.png" alt="image-20240808221237189" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221249489.png" alt="image-20240808221249489" />

### 选择一个安卓版本的`sdk`,这里我选择`Android 14`

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221406978.png" alt="image-20240808221406978" />

### 其他安装（可选）

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221457136.png" alt="image-20240808221457136" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221514899.png" alt="image-20240808221514899" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221529149.png" alt="image-20240808221529149" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221542378.png" alt="image-20240808221542378" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221644243.png" alt="image-20240808221644243" />

### 点击"Apply"确认安装

### 因网络问题无法正常下载sdk等解决方案

- 打开设置

  <img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221842952.png" alt="image-20240808221842952" />

- 找到代理设置，设置代理端口(与自己使用的代理软件监听端口一致)

  <img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808221950198.png" alt="image-20240808221950198" />

  <img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808222056667.png" alt="image-20240808222056667" />

## 在项目中安装`capacitor`

**官方文档**:[Capacitor Android Documentation | Capacitor Documentation (capacitorjs.com)](https://capacitorjs.com/docs/android)

### 在项目目录下执行`npm install`安装项目依赖(如果已安装可忽略)

```bat
npm install
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808213939141.png" alt="image-20240808213939141" />

### 执行`npm i @capacitor/core`安装`capacitor`核心包

```
npm i @capacitor/core
```

### 执行`npm i -D @capacitor/cli`安装`capacitor command line interface` （命令行接口）包

```
npm i -D @capacitor/cli
```

### 执行`npx cap init`初始化`capacitor`配置,依次输入app名和项目ID,回车确认

```
npx cap init
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808215213709.png" alt="image-20240808215213709" />

### 执行`npm install @capacitor/android`安装`@capacitor/android`包

```bat
npm install @capacitor/android
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808214245668.png" alt="image-20240808214245668" />

### 执行`npx cap add android`创建安卓项目,成功创建后将生成`android`项目目录

```
npx cap add android
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808215557219.png" alt="image-20240808215557219" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808215706026.png" alt="image-20240808215706026" />

## 构建项目并同步到安卓项目中

### 修改`svelte.config.js`配置文件，禁用预压缩，目的是在`vite build`过程中不要生成`.gz`和`.br`压缩文件，避免在后续安卓构建过程中出现"Duplicate resources"的问题（因文件重名导致）

**官方文档相关说明**：[Static site generation • Docs • SvelteKit](https://kit.svelte.dev/docs/adapter-static#options-precompress)

> precompress
>
> If `true`, precompresses files with brotli and gzip. This will generate `.br` and `.gz` files.

**// svelte.config.js**

```js
import adapter from '@sveltejs/adapter-static';

import preprocess from 'svelte-preprocess';

/** @type {import('@sveltejs/kit').Config} */
const config = {
	// Consult https://github.com/sveltejs/svelte-preprocess
	// for more information about preprocessors
	preprocess: preprocess(),

	onwarn: (warning, handler) => {
		if (warning.code.startsWith('a11y-')) {
			return;
		}
		handler(warning);
	},

	kit: {
		adapter: adapter({
			// default options are shown. On some platforms
			// these options are set automatically — see below
			pages: 'build',
			assets: 'build',
			fallback: 'index.html',
			precompress: false, // 禁用预压缩
			strict: true,
		}),

	}
};

export default config;

```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808225709443.png" alt="image-20240808225709443" />

### 执行`npm run build`构建web项目

```
npm run build
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808225541397.png" alt="image-20240808225541397" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808225550738.png" alt="image-20240808225550738" />

### 修改`capacitor.config.ts`web项目目录配置

```ts
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.zpekii.app',
  appName: 'myapp',
  webDir: 'build'// 默认为"dist",因为sveltekit构建后生成的是"build"目录，所以我需要做相应更改以正确执行后续的拷贝操作
};

export default config;
```



### 执行`npx cap sync`同步到安卓项目中,此步骤将会把构建好的web项目拷贝到安卓项目指定目录下

```
npx cap sync
```

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808230816735.png" alt="image-20240808230816735" />

**执行成功后可见目录"`/android/app/src/main/assetss/public/`"下拷贝过来的web项目**

## 在Android Studio中执行安卓apk打包

### 回到Android studio,打开安卓项目，打开项目文件后会自动执行build操作

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808231225765.png" alt="image-20240808231225765" />

**可点击下方"Build"查看build情况，若在build过程中遇到网络问题，可参照上面提供的<a href="#因网络问题无法正常下载sdk等解决方案">方案</a>尝试解决**

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808231336005.png" alt="image-20240808231336005" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808231413195.png" alt="image-20240808231413195" />

### 打包成发布版本apk

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808232501498.png" alt="image-20240808232501498" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808232507805.png" alt="image-20240808232507805" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808232515825.png" alt="image-20240808232515825" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808232535833.png" alt="image-20240808232535833" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808231508262.png" alt="image-20240808231508262" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808231511006.png" alt="image-20240808231511006" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240808232156964.png" alt="image-20240808232156964" />

**Apk生成位置: `/android/app/build/outputs/apk/release/*.apk`**

### 生成的发布Apk是未签名的，不可直接安装，需要进行签名

**官方文档（需科学上网）:[Sign your app  | Android Studio  | Android Developers](https://developer.android.com/studio/publish/app-signing#sign-apk)**

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121156622.png" alt="image-20240809121156622" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121233797.png" alt="image-20240809121233797" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121311962.png" alt="image-20240809121311962" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121622323.png" alt="image-20240809121622323" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121605244.png" alt="image-20240809121605244" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121702618.png" alt="image-20240809121702618" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121707206.png" alt="image-20240809121707206" />

<img src="/assets/img/2024-8-8-WebAppToAndroidApp.assets/image-20240809121825621.png" alt="image-20240809121825621" />

### 最后，即可使用真机或安卓模拟器安装已签名的apk进行测试或使用