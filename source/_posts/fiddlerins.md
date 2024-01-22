---
title: Fiddler Inspectors AES解密插件开发
tags: [插件开发]
---

### 简介

本文主要记录 从0开始 使用C#开发fiddler的解密插件的具体操作。

<!-- more -->


### 开发背景

业务接口中部分接口数据采取了AES加密，使用fiddler抓包看不到相关的详细字段，不方便分析定位问题，fiddler官方提供了相关接口可以根据实际场景开发自己想要的插件，使用fiddler开发插件的思路大致如本文，插件不一定通用 但是思路是差不多的。

### 具体开发步骤

一. fiddler插件使用C#语言开发，先安装visual studio，不需要太高级的功能，所以在官网直接下了个社区最新版，安装时选择.NET桌面开发

二. 安装好 创建项目 选择类库(.NET Framework),然后点击下一步 进项目

三. 在AssemblyInfo.cs文件中添加fiddler版本号

四. 上一步中如果有报错，添加fiddler引用 来解决，右键引用=>添加引用=>点击浏览找到fiddler安装路径，选择fiddler.exe添加 并添加standard.dll

五. 新增:DecryptionUtil.cs、RequestDecryption.cs、RequestDecryptionFormat.cs、ResponseDecryption.cs、ResponseDecryptionFormat.cs 这5个类，代码复制粘贴

1. DecryptionUtil.cs，这个类 处理 解密逻辑，根据项目处理为自己的逻辑即可
```
using Fiddler;
using System;
using System.Diagnostics;
using System.Security.Cryptography;
using System.Text;

namespace Util
{
    class DecryptionUtil
    {   
         public static string DecryptSDKBody(String bodyContent)
        {
            JObject encryptionData = JObject.Parse(bodyContent);
            FiddlerApplication.Log.LogString("获取到的请求数据------");
            FiddlerApplication.Log.LogString(encryptionData.ToString());
            var key = ""; 
            var iv = "";
            JObject decryptionData = new JObject();
            //遍历请求体中键值对
            foreach (var property in encryptionData.Properties())
            {
             
                if (做一些自己需要的逻辑判断)
                {
                    decryptionData[property.Name] = property.Value.ToString();

                }
                else
                {
                    FiddlerApplication.Log.LogString("获取到的值------");
                    FiddlerApplication.Log.LogString(property.Value.ToString());
                    //对值进行解密
                    string decryptedValue = AESDecryption(property.Value.ToString(), key, iv);
                    decryptionData[property.Name] = decryptedValue; // 将解密后的值赋给 cc 对象}
                }
                //myControl.setText(decryptionData.ToString());
            }
            FiddlerApplication.Log.LogString("解密后的数据------");
            FiddlerApplication.Log.LogString(decryptionData.ToString());

            return decryptionData.ToString();
        }

        // AES解密
        public static string AESDecryption(string text, string key, string iv)
        {
            try
            {
                FiddlerApplication.Log.LogString("解密算法中text");
                FiddlerApplication.Log.LogString(text);
                FiddlerApplication.Log.LogString("解密算法中AesKey");
                FiddlerApplication.Log.LogString(key);
                byte[] byteKEY = Encoding.UTF8.GetBytes(key);
                byte[] byteIV = Encoding.UTF8.GetBytes(iv);
                byte[] byteDecrypt = System.Convert.FromBase64String(text);

                var _aes = new RijndaelManaged
                {
                    Key = byteKEY,
                    iv = byteIV,
                    Mode = 项目加密Mode,
                    Padding = 项目的Padding 
                    
                };

                var _crypto = _aes.CreateDecryptor();
                byte[] decrypted = _crypto.TransformFinalBlock(byteDecrypt, 0, byteDecrypt.Length);

                _crypto.Dispose();
                return Encoding.UTF8.GetString(decrypted);

            }
            catch (Exception ex)
            {
                // Console.WriteLine(ex.ToString());
                FiddlerApplication.Log.LogString(ex.ToString());
                return null;
            }
        }

    }
}
```
2. RequestDecryption.cs这个类是解密请求数据，text格式
```
using Fiddler;
using System;
using Standard;
using System.Windows.Forms;
using Util;

namespace Request
{
    public sealed class RequestDecryption : Inspector2, IRequestInspector2, IBaseInspector2
    {
        private bool mBDirty;
        private bool mBReadOnly;
        private byte[] mBody;
        private HTTPRequestHeaders mRequestHeaders;
        private RequestTextViewer mRequestTextViewer;

       public RequestDecryption()
        {
            mRequestTextViewer = new RequestTextViewer();
        }

        public bool bDirty
        {
            get
            {
                return mBDirty;            
            }
        }

        public byte[] body
        {
            get
            {
                return mBody;        
            }

            set
            {
                mBody = value;
                byte[] decodedBody = DoDecryption();
                if (decodedBody != null)
                {
                    mRequestTextViewer.body = decodedBody;
                }
                else
                {
                    mRequestTextViewer.body = value;
                }
            }
        }


        public byte[] DoDecryption()
        {
            // 将 byte[] 转成字符串
            String rawBody = System.Text.Encoding.Default.GetString(mBody);
            String showBody = rawBody;
            FiddlerApplication.Log.LogString("rawBody: " + rawBody);
            showBody = DecryptionUtil.DecryptSDKBody(rawBody);
            if (showBody != null)
            {
                byte[] decodeBody = System.Text.Encoding.UTF8.GetBytes(showBody);
                return decodeBody;
            }
            else
            {
                Clear();
                return null;
            }
        }

        public bool bReadOnly
        {
            get
            {
                return mBReadOnly;
            }

            set
            {
                mBReadOnly = value;
            }
        }

        public HTTPRequestHeaders headers
        {
            get
            {
                FiddlerApplication.Log.LogString("headers get function.");
                return mRequestHeaders;
            }  
            set
             {
                mRequestHeaders = value;           

            }

        }

        public override void AddToTab(TabPage o)
        {
            mRequestTextViewer.AddToTab(o);
            o.Text = "Decryption";
        }

        public  void Clear()
        {
            mBody = null;
            mRequestTextViewer.Clear();
        }

        // 在 Tab 上的摆放位置
        public override int GetOrder() => 100;
    }
}
```

3. RequestDecryptionFormat.cs这个类是解密请求数据，json格式
```
using Fiddler;
using System;
using System.Windows.Forms;
using Standard;
using Util;

namespace Request
{
    public class RequestDecryptionFormat : Inspector2, IRequestInspector2, IBaseInspector2
    {
        private bool mBDirty;
        private bool mBReadOnly;
        private byte[] mBody;
        private HTTPRequestHeaders mRequestHeaders;
        private JSONRequestViewer mJsonRequestViewer;

        public RequestDecryptionFormat()
        {
            mJsonRequestViewer = new JSONRequestViewer();
        }

        public bool bDirty
        {
            get
            {
                return mBDirty;
            }
        }

        public byte[] body
        {
            get
            {
                return mJsonRequestViewer.body;
            }

            set
            {
                mBody = value;
                byte[] decodedBody = DoDecryption();
                if (decodedBody != null)
                {
                    mJsonRequestViewer.body = decodedBody;
                }
                else
                {
                    mJsonRequestViewer.body = value;
                }
            }
        }

        public byte[] DoDecryption()
        {
            // 将 byte[] 转成字符串
            String rawBoday = System.Text.Encoding.Default.GetString(mBody);
            String showBody = rawBoday;
            FiddlerApplication.Log.LogString("rawBoday: " + rawBoday);
            showBody = DecryptionUtil.DecryptSDKBody(rawBoday);
            if (showBody != null)
            {
                byte[] decodeBody = System.Text.Encoding.UTF8.GetBytes(showBody);
                return decodeBody;
            }
            else
            {
                Clear();
                return null;
            }
        }

        public bool bReadOnly
        {
            get
            {
                return mBReadOnly;
            }

            set
            {
                mBReadOnly = value;
                mJsonRequestViewer.bReadOnly = value;
            }
        }

        public HTTPRequestHeaders headers
        {
            get
            {
                return mRequestHeaders;
            }

            set
            {

                mRequestHeaders = value;
                mJsonRequestViewer.headers = value;
            }
        }

        public override void AddToTab(TabPage o)
        {
            mJsonRequestViewer.AddToTab(o);
            o.Text = "DecryptionJSON";
        }

        public void Clear()
        {
            mBody = null;
            mJsonRequestViewer.Clear();
        }

        // 在 Tab 上的摆放位置
        public override int GetOrder() => 100;
    }
}
```
4. ResponseDecryption.cs这个类是解密响应数据，text格式
```
using Fiddler;
using System;
using Standard;
using System.Windows.Forms;
using Util;

namespace Response
{
    public sealed class ResponseDecryption : Inspector2, IResponseInspector2, IBaseInspector2
    {
        private bool mBDirty;
        private bool mBReadOnly;
        private byte[] mBody;
        private HTTPResponseHeaders mResponseHeaders;
        private ResponseTextViewer mResponseTextViewer;

        public ResponseDecryption()
        {
            mResponseTextViewer = new ResponseTextViewer();
        }

        public bool bDirty
        {
            get
            {
                return mBDirty;
            }
        }

        public byte[] body
        {
            get
            {
                return mBody;
            }

            set
            {
                mBody = value;
                byte[] decodedBody = DoDecryption();
                if (decodedBody != null)
                {
                    mResponseTextViewer.body = decodedBody;
                }
                else
                {
                    mResponseTextViewer.body = value;
                }
            }
        }

        public byte[] DoDecryption()
        {
            // 将 byte[] 转成字符串
            String rawBody = System.Text.Encoding.Default.GetString(mBody);
            String showBody = rawBody;
            FiddlerApplication.Log.LogString("rawBody: " + rawBody);
            showBody = DecryptionUtil.DecryptSDKBody(rawBody);
            if (showBody != null)
            {
                byte[] decodeBody = System.Text.Encoding.UTF8.GetBytes(showBody);
                return decodeBody;
            }
            else
            {
                Clear();
                return null;
            }
        }

        public bool bReadOnly
        {
            get
            {
                return mBReadOnly;
            }

            set
            {
                mBReadOnly = value;
            }
        }

        HTTPResponseHeaders IResponseInspector2.headers
        {
            get
            {
                FiddlerApplication.Log.LogString("headers get function.");
                return mResponseHeaders;
            }
            set
            {
                FiddlerApplication.Log.LogString("headers set function.");
                mResponseHeaders = value;
            }
        }

        public override void AddToTab(TabPage o)
        {
            mResponseTextViewer.AddToTab(o);
            o.Text = "Decryption";
        }

        public void Clear()
        {
            mBody = null;
            mResponseTextViewer.Clear();
        }

        // 在 Tab 上的摆放位置
        public override int GetOrder() => 100;
    }
}
```
5. ResponseDecryptionFormat.cs这个类是解密响应数据，json格式
```
using Fiddler;
using System;
using System.Windows.Forms;
using Standard;
using Util;

namespace Response
{
    public class ResponseDecryptionFormat : Inspector2, IResponseInspector2, IBaseInspector2
    {
        private bool mBDirty;
        private bool mBReadOnly;
        private byte[] mBody;
        private HTTPResponseHeaders mResponseHeaders;
        private JSONResponseViewer mJsonResponseViewer;

        public ResponseDecryptionFormat()
        {
            mJsonResponseViewer = new JSONResponseViewer();
        }

        public bool bDirty
        {
            get
            {
                return mBDirty;
            }
        }

        public byte[] body
        {
            get
            {
                return mJsonResponseViewer.body;
            }

            set
            {
                mBody = value;
                byte[] decodedBody = DoDecryption();
                if (decodedBody != null)
                {
                    mJsonResponseViewer.body = decodedBody;
                }
                else
                {
                    mJsonResponseViewer.body = value;
                }
            }
        }

        public byte[] DoDecryption()
        {
            // 将 byte[] 转成字符串
            String rawBody = System.Text.Encoding.Default.GetString(mBody);
            String showBody = rawBody;
            FiddlerApplication.Log.LogString("rawBody: " + rawBody);
            showBody = DecryptionUtil.DecryptSDKBody(rawBody);
            if (showBody != null) 
            { 
                byte[] decodeBody = System.Text.Encoding.UTF8.GetBytes(showBody);
                return decodeBody;
            }
            else
            {
                Clear();
                return null;
            }
        }


        public bool bReadOnly
        {
            get
            {
                return mBReadOnly;
            }

            set
            {
                mBReadOnly = value;
                mJsonResponseViewer.bReadOnly = value;
            }
        }

        HTTPResponseHeaders IResponseInspector2.headers
        {
            get => mResponseHeaders;
            set
            {
                mResponseHeaders = value;
                mJsonResponseViewer.headers = value;
            }
        }

        public override void AddToTab(TabPage o)
        {
            mJsonResponseViewer.AddToTab(o);
            o.Text = "DecryptionJSON";
        }

        public void Clear()
        {
            mBody = null;
            mJsonResponseViewer.Clear();
        }

        // 在 Tab 上的摆放位置
        public override int GetOrder() => 100;
    }
}
``` 

上述内容就是这个插件的完整开发过程，这里没有放图片，更多详细内容可以参考下面链接


### 文章初次发表于

[点击这里跳转查看更详细内容](https://mp.weixin.qq.com/s?__biz=MzU1MDgxNjgyMg==&mid=2247484302&idx=1&sn=49b5938b00efa77956b54bef43fdf7be&chksm=fb9b9ed7ccec17c1685f4622eccc067bd2eeb1741881080102c410d8dadb45f68d2f09e817ff&token=1498682306&lang=zh_CN#rd)

