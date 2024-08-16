---
title: Mock WebSocket In Playwright(not support mock websocket exactly) Test
author: Zpekii
tags: [playwright, automation test, E2E testing]
categories: [automation test, playwright, E2E testing]
descriptions: 如何在playwright还未支持websocket mock 下进行mock websocket消息
---



# 在Playwright中Mock WebSocket

##  前言:

截止于写稿日期(2024-8-13)，据我了解到的信息,`playwright`官方还未支持`mock websocket`,现官方提供提供的`WebSocket`类仅能实现监听`page`下开启的`websocket`连接，并获取该连接下的各种事件,而无法做到`mock`自定义数据返回。官方`github`仓库相关`issue`:[[Feature\] Websocket interception · Issue #4488 · microsoft/playwright (github.com)](https://github.com/microsoft/playwright/issues/4488)

因此，只能寻找playwright官方提供的`api`以外的方法去实现`mock websocket`,我这里从`playwright`官方`github`仓库`#4488 issue`中其他人的评论提供的示例中了解到一个可行的方法，即可以在测试中创建一个`WebSocket`服务器去与前端进行连接，在测试中发送自定义数据返回，以此实现`mock websocket`

## 需要:

- `nodejs`
- `playwright`
- 一个可以正常运行的项目

## 安装依赖:

### 安装项目依赖

```
npm install
```

### 安装`ws`(WebSocket 包，提供创建WebSocketServer)

```
npm install ws
```

## 测试用例:

### 项目代码中创建`websocket`连接并打开，进行发送和接收消息

// websocket.js

```js
let ws = new WebSocket(`${document.location.protocol === 'https:' ? 'wss' : 'ws'}://${document.location.host}/msg/hello`)
        
ws.onopen = function(){
    ws.send("hello");
}

ws.onmessage = function(event){
    ws.send(JSON.stringify({
        msg: `give back: ${event.data}`
    }));

    window.isOk = true; // 全局变量，用于判断是否成功接收到消息，做测试验证
}
```

### 测试代码中创建`WebSocketServer`监听`websocket`连接，接收并返回消息

// websocket_test.js

```js
import { test } from '@playwright/test';
import { WebSocketServer } from "ws"

test.describe('webSocket test', ()=>{
    test('test',async ({ page })=>{
        test.setTimeout(2 * 60 * 1000);
        await page.goto('/');
        
        // 获取网页运行端口
        let listen_port = await page.evaluate(()=>{
            return Number(document.location.port);
        })

        // 创建websocket服务器，监听网页运行端口
        let wss = new WebSocketServer({ port: listen_port });

        // 监听连接消息
        wss.on('connection', (ws)=>{
            // 接收消息， 这里是服务端websocket，只能监听"onmessage"和"close"事件
            ws.on('message', (message)=>{
                console.log('received: %s', message);// 打印接收到的消息
                ws.send('hi');// 返回消息
            });
        })

        // 验证结果
        await page.waitForFunction(()=>{
            return window.isOk === true;
        })
    });
});
```

