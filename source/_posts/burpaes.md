---
title: BurpSuite 请求/响应解密插件开发
tags: [插件开发]
excerpt: 这篇文章主要记录如何利用burp官方的新版API即MontoyaApi 写一个请求/响应的解密插件。
sticky: 4
---
### 简介

这篇文章主要记录如何利用burp官方的新版API即MontoyaApi 写一个请求/响应的解密插件。


### 开发背景
业务接口中对请求/响应body的某些特殊字段做了加密处理，抓包后发现只能看到一堆加密后的信息，对于排查定位问题来讲 不是很方便，所以开发了一个解密插件。


### 开发前的准备工作

1. 新建项目 取名为 BurpExtendMH，GroupId 设置为butp，JDK 使用的17，在该项目的目录 src->main->java->下面新建五个文件分别为

```
BurpExtensionMH、#主入口
MyExtensionProvidedHttpRequestEditor、#请求相关
MyExtensionProvidedHttpResponseEditor、# 响应相关
MyHttpRequestEditorProvider、# 请求相关
MyHttpResponseEditorProvider # 响应相关

```

2. pom.xml详见如下配置

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>burp</groupId>
    <artifactId>BurpExtendMH</artifactId>
    <version>1.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <build>
        <sourceDirectory>src</sourceDirectory>
        <plugins>

            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <addDefaultImplementationEntries>
                                true<!--to get Version from pom.xml -->
                            </addDefaultImplementationEntries>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>


    <dependencies>
        <dependency>
            <groupId>net.portswigger.burp.extensions</groupId>
            <artifactId>montoya-api</artifactId>
            <version>2023.10.3</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.78</version>
        </dependency>
    </dependencies>

</project>

```

### 开始写代码

1. 在BurpExtensionMH.java里面写如下代码

```
package main.java;

import burp.api.montoya.BurpExtension;
import burp.api.montoya.MontoyaApi;
import burp.api.montoya.logging.Logging;


public class BurpExtensionMH implements BurpExtension {
    @Override
    public void initialize(MontoyaApi api)
{
        Logging logging = api.logging();
        logging.logToOutput("Hello output.");
        logging.logToError("Hello error.");

        api.extension().setName("Decode Data");

        api.userInterface().registerHttpRequestEditorProvider(new MyHttpRequestEditorProvider(api));
        api.userInterface().registerHttpResponseEditorProvider(new MyHttpResponseEditorProvider(api));
    }
}
```

2. 在MyExtensionProvidedHttpRequestEditor.java里面写如下代码

```
package main.java;

import burp.api.montoya.MontoyaApi;
import burp.api.montoya.core.ByteArray;
import burp.api.montoya.http.message.HttpRequestResponse;
import burp.api.montoya.http.message.requests.HttpRequest;
import burp.api.montoya.ui.Selection;
import burp.api.montoya.ui.editor.EditorOptions;
import burp.api.montoya.ui.editor.RawEditor;
import burp.api.montoya.ui.editor.extension.EditorCreationContext;
import burp.api.montoya.ui.editor.extension.EditorMode;
import burp.api.montoya.ui.editor.extension.ExtensionProvidedHttpRequestEditor;
import burp.api.montoya.utilities.Base64Utils;
import burp.api.montoya.utilities.URLUtils;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;




import static burp.api.montoya.core.ByteArray.byteArray;

class MyExtensionProvidedHttpRequestEditor implements ExtensionProvidedHttpRequestEditor {
    private final RawEditor requestEditor;
    private final Base64Utils base64Utils;
    private final URLUtils urlUtils;
    private HttpRequestResponse requestResponse;
    private final MontoyaApi api;
    String parsedHttpParameter;



    MyExtensionProvidedHttpRequestEditor(MontoyaApi api, EditorCreationContext creationContext) {
        this.api = api;
        base64Utils = api.utilities().base64Utils();
        urlUtils = api.utilities().urlUtils();

        if (creationContext.editorMode() == EditorMode.READ_ONLY) {
            requestEditor = api.userInterface().createRawEditor(EditorOptions.READ_ONLY);
        } else {
            requestEditor = api.userInterface().createRawEditor();
        }
    }

    @Override
    public HttpRequest getRequest() {
        HttpRequest request;
        request = requestResponse.request();
        return request;
    }

    @Override
    public void setRequestResponse(HttpRequestResponse requestResponse) {
        this.requestResponse = requestResponse;

        ByteArray output;
        output = byteArray(parsedHttpParameter);

        this.requestEditor.setContents(output);

    }

    @Override
    public boolean isEnabledFor(HttpRequestResponse requestResponse) {
        String requestDataParam = null;
        try {
            requestDataParam = requestResponse.request().bodyToString();
        } catch (Exception e) {
            System.out.println("requestDataParam为空");
        }
        if (requestDataParam != null) {
            if (//一些条件判断比如是否包含某个字段) {
                JSONObject requestJsonObject = JSON.parseObject(requestDataParam);
                // 这里省略了具体的解密逻辑         
                return true;

            } else {
                return false;
            }
        } else {
            return false;
        }
    }




    @Override
    public String caption() {
        return "Decode data";
    }

    @Override
    public Component uiComponent() {
        return requestEditor.uiComponent();
    }

    @Override
    public Selection selectedData() {
        return requestEditor.selection().isPresent() ? requestEditor.selection().get() : null;
    }

    @Override
    public boolean isModified() {
        return requestEditor.isModified();
    }
}
```

3. 在MyExtensionProvidedHttpResponseEditor.java里面写如下代码

```
package main.java;

import burp.api.montoya.MontoyaApi;
import burp.api.montoya.core.ByteArray;
import burp.api.montoya.http.message.HttpRequestResponse;
import burp.api.montoya.http.message.responses.HttpResponse;
import burp.api.montoya.ui.Selection;
import burp.api.montoya.ui.editor.EditorOptions;
import burp.api.montoya.ui.editor.RawEditor;
import burp.api.montoya.ui.editor.extension.EditorCreationContext;
import burp.api.montoya.ui.editor.extension.EditorMode;
import burp.api.montoya.ui.editor.extension.ExtensionProvidedHttpResponseEditor;
import burp.api.montoya.utilities.Base64Utils;
import burp.api.montoya.utilities.URLUtils;
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.google.gson.JsonSyntaxException;



import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;


import static burp.api.montoya.core.ByteArray.byteArray;

public class MyExtensionProvidedHttpResponseEditor implements ExtensionProvidedHttpResponseEditor {

    private final RawEditor responseEditor;
    private final Base64Utils base64Utils;
    private final URLUtils urlUtils;
    private final MontoyaApi api;
    private HttpRequestResponse requestResponse;

    String parsedHttpParameter;

    MyExtensionProvidedHttpResponseEditor(MontoyaApi api, EditorCreationContext creationContext) {
        this.api = api;
        base64Utils = api.utilities().base64Utils();
        urlUtils = api.utilities().urlUtils();

        if (creationContext.editorMode() == EditorMode.READ_ONLY) {
            responseEditor = api.userInterface().createRawEditor(EditorOptions.READ_ONLY);
        } else {
            responseEditor = api.userInterface().createRawEditor();
        }
    }

    @Override
    public HttpResponse getResponse() {
        HttpResponse response;
        response = requestResponse.response();
        api.logging().logToOutput(String.valueOf(response));
        return response;
    }

    @Override
    public void setRequestResponse(HttpRequestResponse requestResponse) {
        this.requestResponse = requestResponse;

        ByteArray output = byteArray(parsedHttpParameter);
        this.responseEditor.setContents(output);
    }

    @Override
    public boolean isEnabledFor(HttpRequestResponse requestResponse) {


        String requestDataParam = null;
        try {
            requestDataParam = requestResponse.request().bodyToString();
        } catch (Exception e) {
            System.out.println("requestDataParam为空");
        }
        String responseDataParam = requestResponse.response().bodyToString();
        if (requestDataParam != null) {
            if (//一些条件判断比如是否包含某个字段) {
                // 将requestDataParam转为json对象
                JSONObject responseJsonObject = JSON.parseObject(responseDataParam);
                // 这里是具体的解密逻辑 可根据具体业务进行编码
                return true;

            } else {             
                return false;
            }
        } else {
            return false;
        }
    }

    @Override
    public String caption() {
        return "Decode data";
    }

    @Override
    public Component uiComponent() {
        return responseEditor.uiComponent();
    }

    @Override
    public Selection selectedData() {
        return responseEditor.selection().isPresent() ? responseEditor.selection().get() : null;
    }

    @Override
    public boolean isModified() {
        return responseEditor.isModified();
    }
}
```

4. 在MyHttpRequestEditorProvider.java里面写如下代码

```
package main.java;

import burp.api.montoya.MontoyaApi;
import burp.api.montoya.ui.editor.extension.EditorCreationContext;
import burp.api.montoya.ui.editor.extension.ExtensionProvidedHttpRequestEditor;
import burp.api.montoya.ui.editor.extension.HttpRequestEditorProvider;

class MyHttpRequestEditorProvider implements HttpRequestEditorProvider
{
    private final MontoyaApi api;

    MyHttpRequestEditorProvider(MontoyaApi api)
    {
        this.api = api;
    }

    @Override
    public ExtensionProvidedHttpRequestEditor provideHttpRequestEditor(EditorCreationContext creationContext)
{
        return new MyExtensionProvidedHttpRequestEditor(api, creationContext);
    }
}

```

5. 在MyHttpResponseEditorProvider.java里面写如下代码

```
package main.java;

import burp.api.montoya.MontoyaApi;
import burp.api.montoya.ui.editor.extension.EditorCreationContext;
import burp.api.montoya.ui.editor.extension.ExtensionProvidedHttpResponseEditor;
import burp.api.montoya.ui.editor.extension.HttpResponseEditorProvider;

public class MyHttpResponseEditorProvider implements HttpResponseEditorProvider {
    private final MontoyaApi api;

    MyHttpResponseEditorProvider(MontoyaApi api)
    {
        this.api = api;
    }
    @Override
    public ExtensionProvidedHttpResponseEditor provideHttpResponseEditor(EditorCreationContext creationContext) {
        return new MyExtensionProvidedHttpResponseEditor(api, creationContext);
    }
}
```

写完代码后使用maven打jar包， 然后在burp中加载插件 可以实现对请求/相应内容的解密操作

### 总结几个实操过程中遇到的问题及解决方案

1. 解密逻辑中采用了三方库fastjson(gson)，最开始的时候pom.xml文件中没有正确配置 导致本地运行打包没啥问题，一加载到burp中就报没有三方库的错误，最终发现是pom.xml的配置有问题，然后将配置改为开头给出的配置信息后将问题解决

2. 解析json数据时 开始用的三方库是gson，但是发现该插件解析出来的时间戳是科学计数法的形式，在分析问题转换时间戳时比较麻烦，后来采用fastjson 解决了该问题。

3. 解密过程中 开始采用的解密算法和业务加解密算法有点细微差别(开始觉得一点写法上的小差别应该问题不大)，导致数据解密一致不正确，最终采用和业务完全一致的逻辑解决该问题。


### 文章初次发表于

[点击这里跳转查看更详细内容](https://mp.weixin.qq.com/s?__biz=MzU1MDgxNjgyMg==&mid=2247484298&idx=1&sn=281c6bc056a2c6044db1487a940695a3&chksm=fb9b9ed3ccec17c5dbeca896e6d66bcff05dc14fb777fe846d55c4c06bb98e4170d4de81f45e&token=84371561&lang=zh_CN#rd)