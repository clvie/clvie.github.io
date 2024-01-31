---
title: DrissionPage使用实践
date: 2024-01-31 21:28:46
tags: [插件开发]
excerpt: 本文主要记录一个selenium的替代工具 -------  DrissionPage
sticky: 5
---

### 简介

本文主要记录一个selenium的替代工具 -------  DrissionPage

### 我通常采用python脚本获取数据的方式大概分三类：

1. 是使用requests发送请求获取数据，这种方式通常适用于目标数据的接口比较容易模拟的情况。

2. 使用selenium 控制浏览器进行网页交互 然后获取数据，这种一般适合网页可见数据的获取，只要不是反爬很严格的网站，用这种方式都可以搞。

3. 直接写脚本链接数据库获取数据，这种一般都是内部的一些平台数据需要汇总处理的场景会用到，毕竟外部的数据库不可能轻易链接。

### 背景介绍

平时负责的应用的一些数据需要去firebase(简称fb)和google play console(简称gp) 这两个平台去获取，由于这些数据都没有现成的api，所以采用了上述的第二种方式 即使用selenium模拟用户登录的方式去获取数据。近期这俩平台前端页面有了新的更新，gp还好，比较好适配，fb就麻烦了，更新之后发现之前的方式不好使了，导致获取效率大不如从前。本打算放弃了，隔壁老哥给我推荐了这款工具，然后就尝试了一下，确实很行。

### DrissionPage大致介绍

是一个基于 python 的网页自动化工具，它既能控制浏览器，也能收发数据包，还能把两者合而为一，可兼顾浏览器自动化的便利性和 requests 的高效率，它的语法简洁而优雅，代码量少，对新手友好。以前的版本是对 selenium 进行重新封装实现的。从 3.0 开始，作者另起炉灶，对底层进行了重新开发，摆脱对 selenium 的依赖，增强了功能，提升了运行效率。(听起来就很厉害的样子)

### 和selenium对比，亮点如下：

1. 无 webdriver 特征

2. 无需为不同版本的浏览器下载不同的驱动

3. 运行速度更快

4. 可以跨iframe查找元素，无需切入切出

5. 把iframe看作普通元素，获取后可直接在其中查找元素，逻辑更清晰

6. 可以同时操作浏览器中的多个标签页，即使标签页为非激活状态，无需切换

7. 可以直接读取浏览器缓存来保存图片，无需用 GUI 点击另存

8. 可以对整个网页截图，包括视口外的部分（90以上版本浏览器支持）

9. 可处理非open状态的 shadow-root

### 使用文档 如下链接：

```
https://g1879.gitee.io/drissionpagedocs/
```

### 我本次主要用到的功能

1. 切换iframe

2. 常规的定位标签，获取值

下面是主要的代码逻辑，可供参考

```
from DrissionPage import ChromiumPage
import time
import json
import requests

firebase_project_info = [
#  这里都是fb的项目信息
  [],
  [],
  []
]
gp_project_info = [
# 这里都是gp的项目信息
    [],
    [],
    [],
]
project_webhook = {
  # 这里都是钉钉的webhook

}



def joint_firebase_url():
    url = ""
     # 省略了拼接目标数据url的逻辑
    return url


def get_firebase_info(url):
    # 这个方法主要是获取fb平台的数据，根据实际项目替换即可，下面只展示部分逻辑
    
    # 打开目标页面
    page.get(url)
    # 加时间等待，这个网站刷新比较慢，一般不需要单独加等待时间
    time.sleep(2)
    # 目标数据在一个iframe里面，需要切换一下
    iframe = page.get_frame('#iframe的id')
    # 官方建议切换换在调用一下这个方法，听官方的
    iframe = page.get_frame(iframe)
    # 这里假设要获取这个页面的平台这个字段的数据，使用xpath定位
    platform_info = iframe.ele('xpath://div/div[2]')
    # 这里假设平台这个值实在标签的属性里面某个key对应的value,且是这个value的某一部分
    platform = platform_info.attr('属性的key').split("-")[1]
    # 这里假设需要获取这个页面的版本这个字段，同样使用xpath先定位
    version_info = iframe.ele('xpath://div//div')
    # 这里假设版本这个字段的值使用上面这个标签包裹的，直接使用.txt获取这个值
    version = version_info.text


def joint_gp_url():
    url =""
    # 省略了拼接目标数据url的逻辑
    return url


def get_gp_info():
    # 这个方法主要是获取gp平台的数据，根据实际项目替换即可


def send_requests(url, data):
    """
    将信息根据不同的项目名称及对应的钉钉webhook发到对应的群
    :param url:
    :param data:
    :return:
    """

    header = {
        "Content-Type": "application/json",
        "Charset": "UTF-8"
    }
    send_data = json.dumps(data)  # 将字典类型数据转化为json格式
    send_data = send_data.encode("utf-8")  # 编码为UTF-8格式
    requests.post(url=url, data=send_data, headers=header)


def send_message(content, project_name):
    message = {
        "msgtype": "actionCard",
        "actionCard": {
            "title": "数据详情",
            "text": content,
            "hideAvatar": "0",
            "btnOrientation": "0",
        }
    }
    try:
        url = project_webhook[project_name]
        send_requests(url, message)
    except Exception as e:
        print(e)
    finally:
       url = ''
       send_requests(url, message)


if __name__ == '__main__':
    project_info_dict = {}
    # 初始化
    page = ChromiumPage()
    fb_url = joint_firebase_url(...)
    get_firebase_info(fb_url,...)
    time.sleep(1)

    gp_url = joint_gp_url()
    get_gp_info(gp_url, ...)
    # 关闭页面
    page.quit()
    time.sleep(3)
    print(project_info_dict)

    

    for project_name in project_info_dict:
        content = ""
        content += "# 项目名称：{}".format(project_name)
        # 省略组装要发送到钉钉的数据的逻辑
        send_message(content, project_name)

```


### 总结

本次使用DrissionPage后发现获取数据的时间比之前短了很多，而且这个工具对于需要登陆的网站，只需要在本地登录一次之后，在本地浏览器的登录态失效之前可不进行登录操作，也不需要自己去琢磨保存登录态等操作，比较方便。后续遇到同类型场景不再使用selenium了。


### 文章初次发表于

[点击这里跳转查看更详细内容](https://mp.weixin.qq.com/s?__biz=MzU1MDgxNjgyMg==&mid=2247484315&idx=1&sn=7c7e1f2666ed60e7130fd81c8f879397&chksm=fb9b9ec2ccec17d47b81350ef4d36011e983379626d98672a54d262dc3f0f487ae33b7a3cdb8&token=1772209405&lang=zh_CN#rd)