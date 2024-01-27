---
title: 渗透工具---BurpSuite 插件开发之HelloWorld
tags: [插件开发]
---

### 简介

这篇文章主要记录如何利用burp官方的新版API即MontoyaApi 写helloworld

<!-- more -->

### 具体开发步骤

1. 使用IDEA 新建一个maven项目，采用JDK17.其中我的项目名为 BurpPlugin， GroupId 设置为burp(供参考，可以自定义)

2. 在项目的pom.xml文件添加依赖，并刷新maven，依赖如下
```
    <dependencies>
        <dependency>
            <groupId>net.portswigger.burp.extensions</groupId>
            <artifactId>montoya-api</artifactId>
            <version>2023.10.4</version>
        </dependency>
    </dependencies>
```

3. 新建HelloWorld文件，将下面代码粘贴进去
```
package burp;

import burp.api.montoya.BurpExtension;
import burp.api.montoya.MontoyaApi;
import burp.api.montoya.logging.Logging;

public class HelloWorld implements BurpExtension {

    @Override
    public void initialize(MontoyaApi api) {
        // 设置插件名称
        api.extension().setName("Hello world extension");
        // 初始化日志
        Logging logging = api.logging();

        // 在burp output界面输出 "This is output."
        logging.logToOutput("This is output.");

        // 在burp error界面输出 "This is error."
        logging.logToError("This is error.");

        // 在Burp alerts tab 写一些文案
        logging.raiseInfoEvent("Hello info event.");
        logging.raiseDebugEvent("Hello debug event.");
        logging.raiseErrorEvent("Hello error event.");
        logging.raiseCriticalEvent("Hello critical event.");

        // 在burp error界面输出一个异常文案 "Hello exception."
        throw new RuntimeException("Hello exception.");
    }
}
```
4. 使用maven打jar包，然后将打出的jar加载到burp中就可以看到 代码中输出的log信息了

### 上述步骤 3 中需要注意的三点如下：
1. 必须实现api中的BurpExtension
2. 必须实现initialize方法(这个方法相当于和burp交互的主要入口)
3. 本文主要在initialize方法里面写了几行代码


### 文章初次发表于

[点击这里跳转查看更详细内容](https://mp.weixin.qq.com/s?__biz=MzU1MDgxNjgyMg==&mid=2247484230&idx=1&sn=4053d83ed98fc5f3c43e7c51a180ae7e&chksm=fb9b9e1fccec17094552022f8e8ca29eaa47b88a9546906e4d52f61bac48c6bb5c76543d302d&token=84371561&lang=zh_CN#rd)