+++
title = "爬取网页云评论"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# 分析网站

第一步，查看网站信息

![image-20240616150202086](images/爬取网页云评论/image-20240616150202086.png)

第二步，通过一个一个接口检查，发现最终返回的评论信息在这个请求返回中

![image-20240616150408972](images/爬取网页云评论/image-20240616150408972.png)

第三步，分析这个请求

发现这个请求需要携带参数，并且是加密的

![image-20240616150514534](images/爬取网页云评论/image-20240616150514534.png)

第四步，获取加密信息

- 查看调用栈信息

![image-20240616150924789](images/爬取网页云评论/image-20240616150924789.png)

点击最后一个调用栈，跳转到对应的js代码位置，打一个点，重新刷新页面，一个一个请求查看到对应的请求信息

![image-20240616151739232](images/爬取网页云评论/image-20240616151739232.png)

刷新网页，一步一步调试分析，直到找到对应的url为止

从上面网页第二步请求分析可知所要调的url为：

![image-20240616152152816](images/爬取网页云评论/image-20240616152152816.png)

通过几次调试，找到了返回评论数据的请求url

![image-20240616152934212](images/爬取网页云评论/image-20240616152934212.png)

下一步，开始往回找，找到data数据加密的位置

![image-20240616153525435](images/爬取网页云评论/image-20240616153525435.png)

![image-20240616154049924](images/爬取网页云评论/image-20240616154049924.png)

<img src="images/爬取网页云评论/image-20240616154306734.png" alt="image-20240616154306734" style="zoom:150%;" />

找参数：

![image-20240616160331745](images/爬取网页云评论/image-20240616160331745.png)

![image-20240616160506241](images/爬取网页云评论/image-20240616160506241.png)

![image-20240616161017018](images/爬取网页云评论/image-20240616161017018.png)

![image-20240616161525803](images/爬取网页云评论/image-20240616161525803.png)

![image-20240616162538224](images/爬取网页云评论/image-20240616162538224.png)

![image-20240616171517517](images/爬取网页云评论/image-20240616171517517.png)



![image-20240616172834727](images/爬取网页云评论/image-20240616172834727.png)

代码实现：

```python
import requests
from lxml import etree
#AES加密函数,需要安装pycrypto
from Crypto.Cipher import AES
#进行base64数据转换
from base64 import b64encode
import json

url = 'https://music.163.com/weapi/comment/resource/comments/get?csrf_token='
header = {
    'user-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36'
}

#对js代码进行解析然后把data进行加密（必须要使用网易云的逻辑），params >>encTEXT   encSecKsy >>encSecKey  (都是在window.asrsea中加的密)
#真实参数
data = {    
    'csrf_token': "",
    'cursor': "-1",
    'offset': "0",
    'orderType': "1",
    'pageNo': "2",
    'pageSize': "20",
    'rid': "R_SO_4_2158973221",
    'threadId': "R_SO_4_2158973221",
    }

#找到对应调用的函数并且处理加密过程
"""
function a(a) { 返回随机的16位字符串
        var d, e, b = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", c = "";
        for (d = 0; a > d; d += 1)  #循环16次
            e = Math.random() * b.length,  #随机数
            e = Math.floor(e), #取整
            c += b.charAt(e);  #去字符串中的xxx位置
        return c
function b(a, b) {
        var c = CryptoJS.enc.Utf8.parse(b)
          , d = CryptoJS.enc.Utf8.parse("0102030405060708")
          , e = CryptoJS.enc.Utf8.parse(a)
          , f = CryptoJS.AES.encrypt(e, c, {
            iv: d,
            mode: CryptoJS.mode.CBC
        });
        return f.toString()
    }
    function c(a, b, c) {
        var d, e;
        return setMaxDigits(131),
        d = new RSAKeyPair(b,"",c),
        e = encryptedString(d, a)
    }
    #实际调用的加密函数算法
    四个参数：d:对应data数据,e:'010001',f:'00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7',g:'0CoJUm6Qyw8W8jud'
    function d(d, e, f, g) {   
        var h = {} #空对象
          , i = a(16);   #16位的随机值
        return h.encText = b(d, g),
        h.encText = b(h.encText, i), #返回的就是encTEXT
        h.encSecKey = c(i, e, f),  #返回的就是encSecKey，e，f是定死的，i是变的，此时把i固定
        h
    }
    function e(a, b, d, e) {
        var f = {};
        return f.encText = c(a + e, b, d),
        f
    }
"""
e = '010001'
f = '00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7'
g = '0CoJUm6Qyw8W8jud'
i = "nJaSvmCuZFEAORCi"

def get_encSecKey():
    return "036401bcdf2daa471574782cc5d5a1d3a9b6ee4a347134ba1a84a0fbf68c4d493fe4f3e9ccf3467f5af8e7249165d87bea59af6539f964ec7023d600d48a732be2c8a6f7111ec11ced3ad3bc072442cbfc563fb767da6fe7d6addba5752fdddc7963e486d49bc1c0fa68f10e4c3dbc1269be054fa0b54a92c5f404bfd7d11547"

#对paragms进行加密
def get_params(data):
    first = enc_paragms(data,g)
    second = enc_paragms(first,i)
    return second

def to_16(data):
    pad  = 16 - len(data) % 16
    data += chr(pad) * pad
    return data
#机密函数
def enc_paragms(data,key):
    iv = "0102030405060708"
    data = to_16(data)
    aes = AES.new(key=key.encode('utf-8') ,IV=iv.encode('utf-8'),mode=AES.MODE_CBC) #定义加密器
    bs = aes.encrypt(data.encode('utf-8')) #加密，加密的字符串的长度必须为16的倍数,见to_16函数   “123456char(10)”
    return str(b64encode(bs),'utf-8')   #转化称字符串


res = requests.post(url=url,data={'params':get_params(json.dumps(data)),'encSecKey':get_encSecKey()},headers=header).json()
for line in res["data"]['comments']:
    if line['user']['nickname']:
        print("{}---{}".format(line['user']['nickname'],line['content']))
    else:
        print(line['content'])


```

