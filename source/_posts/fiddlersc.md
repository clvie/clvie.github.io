---
title: Fiddler Script 脚本实现篡改加密的请求/响应数据
tags: [插件开发]
excerpt: 这篇文章主要记录如何利用Fiddler Script 脚本实现篡改 响应前被加密的数据。
sticky: 2
---
### 简介

这篇文章主要记录如何利用Fiddler Script 脚本实现篡改 响应前被加密的数据。


### 开发背景
客户端有个活动需要上线，想要测试活动结束时客户端的一些表现情况。有两种方式：
1. 服务端改结束时间，重新部署 
2. 拦截接口 在响应之前将数据改掉 然后返回给客户端

其中：<br>方案1 比较方便但是会影响所有正在参与该活动的同学，属于利己不利他。<br>方案2 影响面小，但是实现比较复杂，需要自己解密数据-- 修改数据--加密数据 再返回。<br>本文采用了方案2

### 梳理思路
1. 响应前拦截目标接口
2. 先将数据解密 然后拿到目标字段的值进行重新赋值修改
3. 将修改后的数据进行加密 然后返回给客户端

### 具体开发步骤
下面两个步骤都是直接在Fiddler Script 中编码即可，注意需要将Fiddler Script的语言设置为js(Fiddler Script 有 c# 和 js 两种语言可选)
1. 实现加解密方法，具体代码可参考如下
   ```
    //解密
    static function AesDecrypt(decryptStr, aesKey,iv) {
        var byteKEY = System.Text.Encoding.UTF8.GetBytes(aesKey);
        var byteIv = "同上";
        var byteDecrypt = System.Convert.FromBase64String(decryptStr);
  
        var _aes = new RijndaelManaged();
        _aes.Key = byteKEY;
        _aes.iv= byteIv ;
        _aes.Mode = 项目的Mode;
        _aes.Padding = 项目的PaddingMode;
               
        var _crypto = _aes.CreateDecryptor();
        var decrypted = _crypto.TransformFinalBlock(
            byteDecrypt, 0, byteDecrypt.Length);

  
        return System.Text.Encoding.UTF8.GetString(decrypted);
    }
     //加密  
    static function AesEncrypt(byteEncrypt, aesKey，iv) {
        var byteKEY = System.Text.Encoding.UTF8.GetBytes(aesKey);
        var byteIv = "同上";
        
        var _aes = new RijndaelManaged();
        _aes.Key = byteKEY;
        _aes.iv= byteIv ;
        _aes.Mode = 项目的Mode;
        _aes.Padding = 项目的PaddingMode;
               
        var _crypto = _aes.CreateEncryptor();
        var encrypted = _crypto.TransformFinalBlock(
            byteEncrypt, 0, byteEncrypt.Length);
  
  
        return System.Convert.ToBase64String(encrypted);
    }
   ```
   
2. 拦截接口、解密、修改数据、加密、返回数据，具体代码可参考如下
   ```
   static function OnBeforeResponse(oSession: Session) {
        if (m_Hide304s && oSession.responseCode == 304) {
            oSession["ui-hide"] = "true";
        }
       
        //这个例子是获取响应body里面的目标字段1解密，然后获取解密内容里面的'目标字段2'修改值，然后加密返回
        if(oSession.fullUrl.Contains("目标接口")){
            // 获取请求参数
            var requestStringOriginal = oSession.GetRequestBodyAsString();
            var requestJSON = Fiddler.WebFormats.JSON.JsonDecode(requestStringOriginal);
            //获取响应参数
            var responseStringOriginal = oSession.GetResponseBodyAsString();
            var responseJSON = Fiddler.WebFormats.JSON.JsonDecode(responseStringOriginal);
            var key = "项目key";
            var iv = "项目iv";
            if (responseJSON.JSONObject['目标字段1'] != "") {
                var oriDara = responseJSON.JSONObject['目标字段1']
                //解密
                var data = AesDecrypt(responseJSON.JSONObject['目标字段1'], key, iv);
                
                var jsonData = Fiddler.WebFormats.JSON.JsonDecode(data);
        
                jsonData.JSONObject['目标字段2'] = '目标值';
                var strData =  Fiddler.WebFormats.JSON.JsonEncode(jsonData.JSONObject);
                // base64加密
                var inputBytes = System.Text.Encoding.UTF8.GetBytes(strData);
                //var encodedData =System.Convert.ToBase64String(inputBytes);
                // aes加密
                var aesData = AesEncrypt(inputBytes, key, iv);
                
                responseJSON.JSONObject['目标字段1'] = aesData
                
                var mod_json = Fiddler.WebFormats.JSON.JsonEncode(responseJSON.JSONObject);
                oSession.utilSetResponseBody(mod_json);

            }
        }

    }
   ```

### 总结
上述操作其实对于请求body里面的加密内容的修改同样适用，不再赘述。在写代码过程中 不太好调试的话可以将需要的内容写到本地文件 然后查看 也算是一种调试方式。

### 文章初次发表于

[点击这里跳转查看更详细内容](https://mp.weixin.qq.com/s?__biz=MzU1MDgxNjgyMg==&mid=2247484303&idx=1&sn=8d154fac856ec246559c7fe106c5993e&chksm=fb9b9ed6ccec17c068bdd4afe94e6e6e7f800510b02dd8fc73d832f15163b8fbcc6367886bec&token=1498682306&lang=zh_CN#rd)