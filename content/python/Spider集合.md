+++
title = "Spider集合"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 9
aliases = ["/python/spider/spider集合/"]
+++

# 网站分析

1、先找到对应的网站的数据信息，看Headers找到对应的url、请求方式以及请求头相关信息以及响应数据等

![image-20240121232145617](images/image-20240121232145617.png)

![image-20240121232204182](images/image-20240121232204182.png)



2、查看payload请求参数是什么格式

json

data



# 数据解析

## xpath

原理：
	1、实例化一个etree对象，且需要被解析的页面源码数据加载到该对象当中；
	2、调用etree对象中的xpath方法结合着xpath表达式实现标签的定义以及内容的捕获；
依赖包安装：

```python 
pip3 install lxml
```

实例化一个etree对象：
	1、将本地的html源码数据加载到rtree对象中：

​		etree.parse(filepath)
​	2、将从互联网上捕获的源码数据加载到etree对象中：
​		etree.HTML('page_text')
xpath('xpath表达式')
​	/：表示从根节点开始定位；表示的是一个层级
​	//：表示多个层级，也表示从任意位置开始定位
​	定位：
​		属性定位：//tag[@attrName="attrValue"]
​		索引定位：//tag[@attrName="attrValue"]/attrName[n] n从1开始
​	取文本：
​		/text()：标签中直系标签的文本内容

```python
# 取出div标签的文本数据
/div/text()
```

​		//text()：标签中所有标签的文本内容
​	取属性：/@attrName   ---img/src
注意：对于相同的标签，一定要区分开，取到自己想要的标签值



```python
from lxml import etree
text = resp.text

html = etree.HTML(text)    #使用text构造一个XPath解析对象,etree模块可以自动修正HTML文本
html = etree.parse('./ex.html',etree.HTMLParser())    #直接读取文本进行解析
result = html.xpath('//*')      #选取所有节点
result = html.xpath('//li')      #获取所有li节点
result = html.xpath('//li/a')     #获取所有li节点的直接a子节点
result = html.xpath('//li//a')    #获取所有li节点的所有a子孙节点
result = html.xpath('//a[@href="link.html"]/../@class')     #获取所有href属性为link.html的a节点的父节点的class属性
result = html.xpath('//li[@class="ni"]')             #获取所有class属性为ni的li节点
result = html.xpath('//li/text()')  #获取所有li节点的文本
result = html.xpath('//li/a/@href')  #获取所有li节点的a节点的href属性
result = html.xpath('//li[contains(@class,"li")]/a/text())    #当li的class属性有多个值时，需用contains函数完成匹配
result = html.xpath('//li[contains(@class,"li") and @name="item"]/a/text()')  #多属性匹配
#按序选择，中括号内为XPath提供的函数
result = html.xpath('//li[1]/a/text()')
result = html.xpath('//li[last()]/a/text()')
result = html.xpath('//li[position()<3]/a/text()')
result = html.xpath('//li[last()-2]/a/text()')
result = html.xpath('//li[1]/ancestor::*')   #获取祖先节点
#获取属性值
result = html.xpath('//li[1]/ancestor::div')
result = html.xpath('//li[1]/attribute::*')
result = html.xpath('//li[1]/child::a[@href="link1.html"]')  #获取直接子节点
result = html.xpath('//li[1]/descendant::span')     #获取所有子孙节点
result = html.xpath('//li[1]/following::*[2]')      #获取当前节点之后的所有节点的第二个
result = html.xpath('//li[1]/following-sibling::*')   #获取后续所有同级节点
```



正则表达式：

| 规则             | 案例                                                         |
| ---------------- | ------------------------------------------------------------ |
| 匹配指定的字符   | /code/g                                                      |
| 匹配字符组及多个 | [Pp]thon  [Pp]th[Oo][Nn]                                     |
| 区间             | [0-9]、[a-z]、[A-Z]、[0-9a-zA-Z]  快捷：  \w：与任意单词字符匹配，任意单词字符表示 [A-Z]、 [a-z]、[0-9]、_（取反：\W）  \d：（digit）与任意数字匹配（取反：\D） |
| 转译             | \                                                            |
| 取反             | [^]                                                          |
| 空白             | \s                                                           |
| 边界字符         | \bcode\b                                                     |
| 开始结束         | ^-----$                                                      |
| 任意字符         | .（不能匹配换行符\n）                                        |
| 可选字符         | ？：匹配前一个字符零次或者多次  a？：匹配a一次或者多次  .？：匹配任意字符 |
| 重复匹配         | 在一个字符组后加上{N} 就可以表示在它之前的字符组出现N次      |
| 重复区间         | {M,N}，M是下界而，N是上界（优先取N个，如取消此优先需在表达式后加上：?） |
| 开闭区间         | +：匹配1个到无数个，等价于{1,}  *：代表0个到无数个，等价于{0,} |
| 分组             | ()，用来捕获数据，从匹配好的数据当中提取关键的数据           |
| 或者             | \|：分组中使用                                               |
| 非捕获组         | (?:code) ，不捕获数据，还能使用分组的功能                    |
| 分组的回溯引用   | \N：可以引用编号为N的分组  (a)(b)(c) \3\2\1 ：\3表示(c)，\2表示(b)，\1表示(a) |
| 先行断言         | **正向先行断言：**(?=code)，是指某个位置向右看，code所在位置右侧必须能匹配code（不存在顺序的约束）<br />例如：**我爱 我爱python 我不爱java 爱python**<br />如果要取出''**爱**''这个字，并且后面有**java**，需要这么写：**爱(?=java)**<br />**反向先行断言：**(?!code)，保证右边不能出现code<br />如果要取出''**爱**''这个字，并且后面没有**java**，需要这么写：**爱(?!java)** |
| 后行断言         | **正向后向断言：**(?<=code)，指在某个位置向左看，表示所在位置左侧必须能匹配code<br />例如:如果要取出**’爱‘**字，要求爱的前面有我，后面有你，这个时候就要这么写: (?=我)爱(?=你)。<br />**反向后行断言：**(?<!code)，指在某个位置向左看，表示所在位置左侧不能匹配code<br />例如:如果要取出**’爱‘**字，要求爱的前面没我，后面没你，这个时候就要这么写: **(?<!我)爱(?!你)** |



## re

语法：s = re.方法(r"partten(正则)", "需要从中提取的源数据内容")

| findall()            | 匹配字符串中所有符合正则的内容，返回的是list                 |
| -------------------- | ------------------------------------------------------------ |
| finditer()           | 匹配字符串中所有符合的内容，返回的是迭代器，从迭代器中拿到的内容需要进行.group() |
| search()             | 找到一个结果就返回，返回的结果是match对象，拿数据需要进行.group() |
| match()              | 从头开始匹配                                                 |
| re.S                 | 让.能匹配换行符                                              |
| (?P<分组名称>[正则]) | ![image-20240109165621259](./images/image-20240109165621259.png) |



预加载正则表达式：

```python
obj = re.compile(r"partten[正则]")
s = obj.finditer("源数据")
```

<img src="./images/image-20240109165717792.png" alt="image-20240109165717792" style="zoom:50%;" />



## bs4

基于bs4可以实现去HTML格式的包裹的数据库中快速提取我们想要的数据。

安装

```python
pip3 install beautifulsoup4
```

使用：

```python
from bs4 import BeautifulSoup

html_string = """<div>
    <h1 class="item">any</h1>
    <ul class="item">
        <li>篮球</li>
        <li>足球</li>
    </ul>
    <div id='x3'>
        <span>abc.cn</span>
        <a href="www.xxx.com" class='info'>any.com</a>
    </div>
</div>"""

soup = BeautifulSoup(html_string, features="html.parser")
tag = soup.find(name='a')

# 1、根据标签名称，获取标签（只获取找到的第1个）
print(tag)       	# 标签对象
print(tag.name)  	# 标签名字 a
print(tag.text)  	# 标签文本 any.com
print(tag.attrs) 	# 标签属性 {'href': 'www.xxx.com', 'class': ['info']}

# 2、根据属性获取标签（只获取找到的第1个）
tag = soup.find(name='div', attrs={"id": "x3"})
print(tag)      # 标签对象

# 3、嵌套读取，先找到某个标签，然后再去孩子标签中寻找
parent_tag = soup.find(name='div', attrs={"id": "x3"})
child_tag = parent_tag.find(name="span", attrs={"class": "xx1"})
print(child_tag)  # 标签对象

# 3、读取所有标签（多个）
tag_list = soup.find_all(name="li")    # 找出所有的li标签，返回一个列表
print(tag_list)                        # 输出：[<li>篮球</li>, <li>足球</li>]

for tag in tag_list:     # 循环列表
    print(tag.text)
# 输出
篮球
足球
```



# requests模块

​	requests url和参数，返回的对象是Response对象，打印出来是显示响应状态码，通过.text 方法可以返回是unicode 型的数据，一般是在网页的header中定义的编码形式，而content返回的是bytes，二级制型的数据，还有 .json方法也可以返回json字符串。如果想要提取文本就用text，但是如果你想要提取图片、文件等二进制文件，就要用content，当然decode之后，中文字符也会正常显示啦 。

不是python的自带库，需要安装：

```bash
pip3 install requests
```

常用方法：

```python
requests.request(url)	# 构造一个请求，支持以下各种方法
requests.get()		# 发送一个Get请求
requests.post()		# 发送一个Post请求
requests.head()		# 获取HTML的头部信息
requests.put()		# 发送Put请求
requests.patch()	#提交局部修改的请求
requests.delete()	#提交删除请求
```



无论是在发送GET/POST请求时，网址URL都可能会携带参数，

```python
# 不带参数
res = requests.get(
	url="https://www.5xclass.cn?age=19&name=wupeiqi"
)

# 携带参数
res = requests.get(
	url="https://www.5xclass.cn",
    params={
        "age":19,
        "name":"wupeiqi"
    }
)
```



## **请求体格式**

在发送POST请求时候，常见的请求体格式一般有二种：**form表单格式、json格式**

**1、form表单格式（抽屉新热榜）**

使用&符号拼接起来：name:xxx&age:99&passwd:xxx

![image-20240109200516353](./images/image-20240109200516353.png)

**传参方式**

``` python
# 1、直接拼接传
resp = requests.post(
	url='xxx',
    data="name:xxx&age:99&passwd:xxx",
    headers={
        xxx
    }
)

# 2、使用字典形式传递
resp = requests.post(
	url='xxx',
    data={
        "name":xxx,
        "age":99,
        "passwd":xxx
    },
    headers={
        xxx
    }
)
```

**2、json格式（腾讯课堂）**

{'name':xxx,'age':99,'passwd':xxx}

![](./images/image-20240109195235400.png)

**传参方式**

```python
# 1、json.dumps()方式
resp = requests.post(
	url='xxx',
    data=json.dumps({"name":xxx, "age":99,"passwd":xxx}),
    headers={
        xxx
    }
)



# 2、直接使用json传参
resp = requests.post(
	url='xxx',
    json={"name":xxx, "age":99,"passwd":xxx},
    headers={
        xxx
    }
)
```

- **示例：**

```python
# 请求：
import requests

url = ''
# 自定义请求头信息
header = {
    'user-Agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'referer':'https://www.abidu.com',
	}
params = {
    'key1':'value1',
}

# 1、get请求
resp = requests.get(url=url, params=params, headers=header)

# 2、post请求，一般用于提交参数，所以直接进行有参数的POST请求测试
resp = requests.get(url=url, params=params, headers=header)

# 输出
resp.status_code	# 响应状态码
resp.content		# 把response对象转换为二进制数据
resp.text			# 把response对象转换为字符串数据,有时会有中文乱码，需要先进行编码：res.encoding = 'utf-8'
resp.encoding		# 定义response对象的编码
resp.cookie			# 获取请求后的cookie
resp.url			# 获取请求网址
resp.json()	 		# 内置的JSON解码器
Resp.headers		# 以字典对象存储服务器响应头，字典键不区分大小写

```

## cookie

Cookie本质上是浏览器存储的键值对，一般用于用户凭证的保存。

- 浏览器向后端发送请求时，后端可以返回cookie（自动保存在浏览器）。
- 后续浏览器再次返送请求时，会自动携带cookie。

![image-20240121230616886](images/image-20240121230616886.png)

![image-20231201093209106](images/image-20231201093209106.png)



读取返回的cookie，并且下一个请求时携带cookie

```python
import requests

res = requests.get(
    url="https://www.bilibili.com/"
)

# 获取到cookie
cookie_dict = res.cookies.get_dict()
print(cookie_dict)   # 字典格式：{"v1":xxxx,"v3":yyyy}

# 请求携带cookie
res = requests.get(
    url="https://www.bilibili.com/",
    headers={
        "Cookie":"innersign=0; buvid3=8427E089-F4D7-CCF7-4997-0087D04B3C9810575infoc"
    }
)


res = requests.get(
    url="https://www.bilibili.com/",
    # cookies = cookie_dict     # 获取到cookie后存到cookie_dict
  	cookies={
        "innersign":"0",
        "buvid3":"8427E089-F4D7-CCF7-4997-0087D04B3C9810575infoc"
    }
)
```



## 响应体格式

基于requests发送请求后，返回的数据都封装在了res对象中

``` python
import requests
import json

res = requests.get(
    url="https://api.huaban.com/search/file?text=%E7%BE%8E%E5%A5%B3&sort=all&limit=40&page=1&position=search_pin&fields=pins:PIN,total,facets,split_words,relations"
)

# 原始响应体（字节类型）
res.content

# 原始文本，将字节转换成字符串形式
res.text

# 如果返回是JSON格式，可以自动转化json格式。   即：data = json.loads(res.text)   注意：{"xxx":123}   <html></asdfasdfadf</>
data = res.json()
```

### 原始字节

一般用于 图片、文件、视频 等下载时，获取原始数据。

https://huaban.com/boards/32310016

```python
import requests

res = requests.get(
    url="https://gd-hbimg.huaban.com/b93fcc5bb4751934bbd56918bdab8184966dca2974df1-bo7qSF",
)
print(res.content)

with open("v1.png", mode='wb') as f:
    f.write(res.content)
```

https://www.douyin.com/video/7291625134530579747?modeFrom=

```python
import requests

res = requests.get(
    url="https://v3-web.douyinvod.com/4f17c475df0a484a41fa1abe00f43aaa/65695cf6/video/tos/cn/tos-cn-ve-15c001-alinc2/oIrIAZg9ChRRquVyAYQdESIxNWAQzBACtzJemf/?a=6383&ch=0&cr=0&dr=0&er=0&cd=0%7C0%7C0%7C0&cv=1&br=1354&bt=1354&cs=0&ds=4&ft=GN7rKGVVyw3XRZ_8emo~xj7ScoAp9656EvrK-iBTkto0g3&mime_type=video_mp4&qs=0&rc=ZWU7NTo3NDc0ZDM2ODQ6OkBpM2c8ODw6Zm9qbjMzNGkzM0AxNS9eY2BjNTQxYGIvYDMwYSNgZzVpcjRnamxgLS1kLS9zcw%3D%3D&btag=e00030000&dy_q=1701400015&feature_id=46a7bb47b4fd1280f3d3825bf2b29388&l=20231201110655C26B991C0F271857AB12",
)
# print(res.content)

with open("v1.mp4", mode='wb') as f:
    f.write(res.content)
```



### 普通文本

```python
import requests

res = requests.get(
	url="https://www.5xclass.cn?age=19&name=wupeiqi"
)
print(res.text)

# 输出
<!DOCTYPE html>
<html lang="en">
<head>
...
```

```python
import requests

res = requests.get(
    url="https://movie.douban.com/j/search_subjects?type=movie&tag=%E8%B1%86%E7%93%A3%E9%AB%98%E5%88%86&page_limit=50&page_start=0",
    headers={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36"
    }
)

print(res.text)

# 输出
{"subjects":[{"episodes_info":"","rate":"9.7","cover_x":2000,"title":"肖申克的救赎"...
```



### 转换格式

 对于json格式，为了更方便的获取内部元素，可以转换成python的字典或列表等类型。

```python
import requests

res = requests.get(
    url="https://movie.douban.com/j/search_subjects?type=movie&tag=%E8%B1%86%E7%93%A3%E9%AB%98%E5%88%86&page_limit=50&page_start=0",
    headers={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36"
    }
)

# 手动转换
import json
data_dict = json.loads(res.text)

# 内部自动转换
data_dict = res.json()
```





## session

​	使用sessionl来发起请求，session实例在请求了一个网站后，对方服务器设置在本地的cookie会保存在session中，下一次再使用session请求对方服务器的时候，会带上前一次的cookie，从而实现回话保持。

```python
# 导入session库
import requests

#创建session对象：
session = requests.Session()

data1 = {
    "name" : "test",
    "passwd" : "passwd"
}

#请求：
session.get(url = url,timeout=1,headers = headers，data=data1)
session.post()
```

作用：
1、可以进行请求发送；
2、如果请求中产生了cookie，则该cookie就会被自动存储到/携带在该session对象中



# 请求登录

## 表单提交

```html
<form action="https://www.xxxx.com/login" method='post'>
    <input type='text' name='user' />
    <input type='password' name='pwd' />
    <input type="submit" value="登录" />
</form>
```

例如github网站登录：https://github.com

![image-20240114215247686](images/image-20240114215247686.png)



## Ajax提交

基于jQuery实现的发送Ajax请求示例代码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div>
    <input type='text' name='user' id="user"/>
    <input type='password' name='pwd' id="pwd"/>
    <input type="button" value="登录" onclick="doLogin()"/>
</div>

<script src="http://code.jquery.com/jquery-2.1.1.min.js"></script>
<script>
    function doLogin() {
        $.ajax({
            url: "提交地址",
            type: "POST",
            data: {
                username: $("#user").val(),
                password: $("#pwd").val(),
            },
            success: function (res) {
                if (res.status) {
                    // 编写登录成功的代码
                    // 主动让页面刷新
                } else {
                    // 编写登录失败的代码
                }
            }
        })
    }
</script>
</body>
</html>
```

例如：北京大学BBS论坛 https://bbs.pku.edu.cn/v2/home.php

![image-20240114215615617](images/image-20240114215615617.png)



## 登录流程

### 方式1

正常请求流程：

- **第1次访问**，后台会返回内容+Cookie，在cookie中保存当前用户凭证（此时凭证没啥用）
- **第2次访问**，输入用户名+密码提交，此时浏览器会自动将第1次返回的凭证携带到后台； 后台校验成功，此时给凭证赋予登录权限（还是原来的凭证，只不过此时的凭证是用户已登录的标识了）。
- **第n次访问**，携带Cookie中的凭证去访问，后台就会根据凭证（用户标识）返回词用户的相关信息。



如果我们基于爬虫去模拟请求实现时：

- 第1次访问，读取返回Cookie并保存
- 第2次访问，携带用户名+密码+上次的Cookie进行登录
- 第n次访问，携带Cookie去访问，获取当前用户信息。



### 方式2

正常请求流程：

- **第1次访问**，后台仅返回页面。
- **第2次访问**，输入**用户名+密码**提交，后台校验成功后，在  **响应体** 或 **Cookie** 返回 用户登录凭证。【网页一般在Cookie中居多】
- **第n次访问**，携带之前返回的凭证去访问，后台就会根据凭证（用户标识）返回词用户的相关信息。



如果我们基于爬虫去模拟请求实现时：

- 第2次访问，携带用户名+密码去登录，在 响应体 或 Cookie中读取用户凭证。【网页一般在Cookie中居多】
- 第n次访问，携带凭证去访问，获取当前用户信息。



### 案例：某军事网

https://www.china.com/

抓包分析：

- 第1个请求，cookie不像凭证，所以断应该是属于方式2

  ![image-20240114220516244](images/image-20240114220516244.png)

  

- 登录请求，返回好多cookie

  ![image-20240114220850888](images/image-20240114220850888.png)

- 其他请求，携带登录请求的cookie即可

```python
import requests
from bs4 import BeautifulSoup


# 目标网站URL
url = "https://passport.china.com/logon"
hearder = {
    "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Referer":"https://passport.china.com/logon",
    "Origin": "https://passport.china.com",
    "X-Requested-With":"XMLHttpRequest"
}
data = {
    "userName": "手机号",
    "password": "密码"
}

# 发送HTTP请求
response = requests.post(
    url=url,
    headers=hearder,
    data=data
)

cookie_dict = response.cookies.get_dict()
print(cookie_dict)

response = requests.get(
    url="https://passport.china.com/main",
    headers=hearder,
    cookies=cookie_dict
)

soup = BeautifulSoup(response.text, features="html.parser")
print(soup)
tag = soup.find(name="a",attrs={"target":"_blank"})
print(tag.text)
print(tag.attrs["title"])

```



# selenium模块

## 安装

1）安装模块：pip install selenium -i https://pypi.tuna.tsinghua.edu.cn/simple/
2）下载浏览器驱动：
	Chrome： （注意保持和自己的chrome本版保持一致）

	114及之前版本： http://chromedriver.storage.googleapis.com/index.html
	117/118/119版本及以上： https://googlechromelabs.github.io/chrome-for-testing/

​	Edge： https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/ （根据自己的电脑型号进行选择下载）
3）把解压缩的浏览器驱动 chromedriver放在python解释器(一般为安装的路径)所在的文件夹（当前：F:\ruanjian\python\chromedriver.exe）
​	查看：print(os.sys.executable)
4）让selenium简单拉起谷歌浏览器
​	from selenium.webdriver import Chrome
​	web = Chrome()
​	web.get("https://www.baidu.com/")



各个方法：

| find_element_by_*          | 查找单个元素                                                 |
| -------------------------- | ------------------------------------------------------------ |
| find_elements_by_*         | 查找多个元素                                                 |
| click()                    | 点击                                                         |
| send_keys()                | 输入键值                                                     |
| switch_to                  | 切换网页窗口                                                 |
| window(web.window_handles) | 网页窗口选项卡                                               |
| close()                    | 关闭当前窗口                                                 |
| get_cookie(name)           | 获取指定cookie（name：为cookie的名称）                       |
| get_cookies()              | 获取本网站所有本地cookies                                    |
| get_cookie(cookie_dict)    | 添加cookie（cookie_dict：一个字典对象，必选的键包括："nama" and  "value"） |



## 请求

``` python
import time
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
                from selenium.webdriver.common.by import By      # 匹配标签使用


service = Service("chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://passport.bilibili.com/login')

time.sleep(5)
driver.close()

```



## 无头请求

不想显示展示在浏览器上的操作，只想偷偷的在后台运行

浏览器options配置（个性化）：

```python
opt.add_argument("--headless")
```

```python
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service

service = Service("driver/chromedriver.exe")
# 添加配置
opt = webdriver.ChromeOptions()
opt.add_argument('--headless')
driver = webdriver.Chrome(service=service, options=opt)

driver.get('https://www.5xclass.cn')
tag = driver.find_element(
    By.XPATH,
    '/html/body/div/div[2]/div/div[2]/div/div[2]/div[2]/div/div/div/div/div[2]/a[1]'
)
print(tag.text)
print(tag.get_attribute("target"))
print(tag.get_attribute("data-toggle"))

driver.close()
```

其他配置：

```python
opt.add_argument('--disable-infobars')                    # 禁止策略化
opt.add_argument('--no-sandbox')                          # 解决DevToolsActivePort文件不存在的报错
opt.add_argument('window-size=1920x3000')                 # 指定浏览器分辨率
opt.add_argument('--disable-gpu')                         # 谷歌文档提到需要加上这个属性来规避bug
opt.add_argument('--incognito')                           # 隐身模式（无痕模式）
opt.add_argument('--disable-javascript')                  # 禁用javascript
opt.add_argument('--start-maximized')                     # 最大化运行（全屏窗口）,不设置，取元素会报错
opt.add_argument('--hide-scrollbars')                     # 隐藏滚动条, 应对一些特殊页面
opt.add_argument('lang=en_US')                            # 设置语言
opt.add_argument('blink-settings=imagesEnabled=false')    # 不加载图片, 提升速度
opt.add_argument('User-Agent=Mozilla/5.0 (Linux; U; Androi....')      # 设置User-Agent
opt.binary_location = r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"  # 手动指定使用的浏览器位置
```



实例化浏览器并规避检测selenium：

![image-20240109162440108](./images/image-20240109162440108.png)

```python
web.execute_cdp_cmd("Page.addScriptToEvaluateOnDecument",{
    "source": """
    navigator.webdriver = undefinded
    Object.defineProperty(navigator, 'webdriver',{
        get: () =>undefined
        })
    """
})


# 浏览器版本大于88的：
opt = Options()
opt.add_argument('--disable-blink-features=AutomationControlled')
opt.add_experimental_option("excludeSwitches", ['enable-automation', 'enable-logging'])
```



## 寻找标签

```python
import time
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By      # 需要导入的此模块，才能进行匹配

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://www.5xclass.cn/')

# 根据ID寻找
tag = driver.find_element(By.ID, "bs-example-navbar-collapse-1")
print(tag.text)
print(10 * "-")

# 根据类名寻找
tags = driver.find_elements(By.CLASS_NAME, "panel-heading")
for tag in tags:
    print(tag.text)
print(10 * "-")

# 根据标签名称寻找
tags = driver.find_elements(By.TAG_NAME, "li")
for tag in tags:
    print(tag.text)
print(10 * "-")

# 根据XPATH寻找
tag = driver.find_element(By.XPATH, "/html/body/div/div[2]/div/div[2]/div/div[2]/div[1]")
print(tag.text)
print(10 * "-")

# 根据XPATH寻找
tag = driver.find_element(By.XPATH, '//*[@id="bs-example-navbar-collapse-1"]/ul[1]/li[1]/a')
print(tag.text)
print(10 * "-")

# 根据XPATH寻找多个
tags = driver.find_elements(By.XPATH, '/html/body/div/div[2]/div/div[2]/div/div[2]/div[2]/div/div/div/div/div[2]/a')
for tag in tags:
    print(tag.text)
print(10 * "-")

# 根据父子关系嵌套寻找
parent = driver.find_element(By.XPATH, '/html/body/div/div[2]/div/div[2]/div/div[2]/div[2]/div/div/div/div')
tags = parent.find_elements(By.XPATH, "div[@class='course']/a")
for tag in tags:
    print(tag.text)

time.sleep(5)
driver.close()
```



## 结合bs4寻找标签

``` python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from bs4 import BeautifulSoup


service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)
driver.implicitly_wait(10)

driver.get('https://car.yiche.com/')

html_string = driver.page_source

soup = BeautifulSoup(html_string, features="html.parser")
tag_list = soup.find_all(name="div", attrs={"class": "item-brand"})
for tag in tag_list:
    child = tag.find(name='div', attrs={"class": "brand-name"})
    print(child.text)

driver.close()

```





## 获取值

当找到某个标签之后，获取标签内部的值

### 文本和属性

```python
# 例如：`<a id='x1' class="info mine" href="any.cn">any</a>`

tag = driver.find_element(
    By.XPATH,
    '/html/body/div/div[2]/div/div[2]/div/div[2]/div[2]/div/div/div/div/div[2]/a[1]'
)

# tag为所匹配的标签对象
print(tag.text)      # 获取文本
print(tag.get_attribute("class"))        # 获取target属性的值
print(tag.get_attribute("href"))   # 获取data-toggle属性的值

driver.close()
```



### 选择相关

```python
# html
<input type="radio" name="findcar" value="1" checked="">新车
<input type="radio" name="findcar" value="2">二手机


# 1.单独找到每一个
tag = driver.find_element(
    By.XPATH,
    '/html/body/div[1]/div[11]/div[2]/div[1]/div[1]/label[1]/span/input'
)
print(tag.get_property("checked")) # True

tag = driver.find_element(
    By.XPATH,
    '/html/body/div[1]/div[11]/div[2]/div[1]/div[1]/label[2]/span/input'
)
print(tag.get_property("checked")) # False


# 2.循环找到每一个
parent = driver.find_element(
    By.XPATH,
    '/html/body/div[1]/div[11]/div[2]/div[1]/div[1]'
)

tag_list = parent.find_elements(
    By.XPATH,
    'label/span/input'
)
for tag in tag_list:
    print( tag.get_property("checked"), tag.get_attribute("value") )

driver.close()
```





## 操作动作

常见的操作：点击，输入，移动

- 点击、输入

```python
driver.get('https://passport.bilibili.com/login')

# 1.点击短信登录
time.sleep(3)
sms_btn = driver.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[1]/div[3]'
)
sms_btn.click()  # 点击


# 2.输入账号
phone_txt = driver.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[2]/div[1]/div[1]/input'
)
phone_txt.send_keys("18630087660")  # 输入

time.sleep(55)
driver.close()
```

- 移动

```python
from selenium.webdriver import ActionChains

tag =  driver.find_element(By.XPATH, '//*[@id="captcha"]/div[2]/div[1]/div[4]/div[1]/div[2]/div/div/div[2]/div/div[3]')

ActionChains(driver).click_and_hold(tag).perform()    # 点击并抓住标签
ActionChains(driver).move_by_offset(xoffset=x1,yoffset=0).perform()  # 滑动，向下向右为负
ActionChains(driver).release().perform()  # 释放
```

- 截屏

找到某个标签后，可以通过截图的形式保存图片（只截取到标签内的内容）

```python
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://www.5xclass.cn')
tag = driver.find_element(
    By.XPATH,
    '/html/body/div/div[2]/div/div[2]/div/div[2]'
)

# 截图&保存
tag.screenshot("demo.png")

# 截图&图片内容
body = tag.screenshot_as_png
print(body)

# 截图&Base64编码格式图片内容
b64_body = tag.screenshot_as_base64
print(b64_body)

driver.close()
```



## 执行javaScript

如果【选择标签】【执行操作】这种操作起来比较繁琐，也可以直接在页面上去执行js代码实现功能。

```python
import time
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://passport.bilibili.com/login')

# 1.点击短信登录 
time.sleep(3)
sms_btn = driver.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[1]/div[3]'
)
sms_btn.click()


# 2.找到框输入账号 
phone_txt = driver.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[2]/div[1]/div[1]/input'
)
phone_txt.send_keys("这是一个测试")

# 3.选择国家 
time.sleep(2)
driver.execute_script('document.querySelector(".area-code-select").children[18].click()')

# 4.读取cookie 
data_string = driver.execute_script('return document.cookie;')   # return document.title;
print(data_string)

# 5.获取cookie
cookie_list = driver.get_cookies()
print(cookie_list)

time.sleep(2550)
driver.close()
```



## 等待

页面加载比较慢的时候，需要等待某个元素加载成功后，才能执行某些操作。

### 基于lambda表达式

```python
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.support.wait import WebDriverWait

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://passport.bilibili.com/login')

# 方式1：直接点击短信登录
time.sleep(3)
sms_btn = driver.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[1]/div[3]'
)
sms_btn.click()

# 方式2：等待页面加载之后，点击短信登录（推荐）
# 使用WebDriverWait方法，后续找元素,无返回值时，等待30s，每0.5s刷一次，嵌套一个lambda函数
sms_btn = WebDriverWait(driver, 30, 0.5).until(lambda dv: dv.find_element(
    By.XPATH,
    '//*[@id="app"]/div[2]/div[2]/div[3]/div[1]/div[3]'
))
sms_btn.click()
```

### 自定义函数

```python
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.support.wait import WebDriverWait

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

driver.get('https://passport.bilibili.com/login')


def func(dv):
    print("无返回值，则间隔0.5s执行一次此函数；如有返回值，则复制给sms_btn变量")
    # <div xxx="123" id="uuu"></div>
    # <img src="..."/>
    tag = dv.find_element(
        By.XPATH,
        '//*[@id="app"]/div[2]/div[2]/div[3]/div[1]/div[3]'
    )
    img_src = tag.get_attribute("xxx")
    if img_src:
        return tag
    return

sms_btn = WebDriverWait(driver, 30, 0.5).until(func)
sms_btn.click()

time.sleep(250)
driver.close()
```

### 全局配置

```python
import time
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

# 后续找元素时，没找到时则等待10s去寻找（一旦找到则继续）
driver.implicitly_wait(10)

driver.get('https://passport.bilibili.com/login')

sms_btn = driver.find_element(
    By.XPATH,
    '//*[@id="xxxxxxxxxapp"]/div[2]/div[2]/div[3]/div[1]/div[3]'
)

sms_btn.click()
print("找到了")
time.sleep(250)
driver.close()
```



## 携带cookie

```python
driver.add_cookie({'name': 'foo', 'value': 'bar'})
```

``` python
import time

from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

# 注意：一定要先访问，不然Cookie无法生效
driver.get('https://dig.chouti.com/about')

# 加cookie
driver.add_cookie({
    'name': 'token',
    'value': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJqaWQiOiJjZHVfNDU3OTI2NDUxNTUiLCJleHBpcmUiOiIxNzA0MzI5NDY5OTMyIn0.8n_tWcEHXsBSXWIY9rBoGWwaLPF8iWIruryhKTe5_ks'
})

# 再访问
driver.get('https://dig.chouti.com/')


time.sleep(2000)
driver.close()
```



## 特征检测

有些网站为了防止selenium，会检测特征，并禁止访问，这时想要正常使用selenium访问，那就需要隐藏浏览器相关的特征。

``` python
import time
import requests
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

service = Service("driver/chromedriver.exe")

opt = webdriver.ChromeOptions()

opt.add_argument('--disable-infobars')
opt.add_experimental_option("excludeSwitches", ["enable-automation"])
opt.add_experimental_option('useAutomationExtension', False)

driver = webdriver.Chrome(service=service, options=opt)

# Selenium在打开任何页面之前，先运行这个Js文件。
with open('driver/hide.js') as f:
    driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {"source": f.read()})

driver.get('https://www.5xclass.cn')

time.sleep(2000)
driver.close()
```



## 案例：x东搜索

```python
import requests
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service

# 换成自己生成的代理
res = requests.get(url="https://dps.kdlapi.com/api/getdps/?secret_id=o60wwtxvs5ukaqqz18ai&num=1&signature=i6s9shfjfiogat5ijecbyfwwc5grwrzj&pt=1&format=json&sep=1")
proxy_string = res.json()['data']['proxy_list'][0]
print(f"获取代理：{proxy_string}")


service = Service("driver/chromedriver.exe")
opt = webdriver.ChromeOptions()

opt.add_argument(f'--proxy-server={proxy_string}')  # 代理
opt.add_argument('blink-settings=imagesEnabled=false') # 不加载图片

opt.add_argument('--disable-infobars')
opt.add_experimental_option("excludeSwitches", ["enable-automation"])
opt.add_experimental_option('useAutomationExtension', False)

driver = webdriver.Chrome(service=service, options=opt)

driver.implicitly_wait(10)

with open('driver/hide.js') as f:
    driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {"source": f.read()})

# 1.打开京东
driver.get('https://www.jd.com/')

# 2.搜索框+输入
tag = driver.find_element(
    By.XPATH,
    '//*[@id="key"]'
)
tag.send_keys("iphone手机")

# 3.点击搜索
tag = driver.find_element(
    By.XPATH,
    '//*[@id="search"]/div/div[2]/button'
)
tag.click()

# 4.查询列表
tag_list = driver.find_elements(
    By.XPATH,
    '//*[@id="J_goodsList"]/ul/li'
)
for tag in tag_list:
    # title = tag.find_element(By.XPATH, 'div/div[@class="p-name p-name-type-2"]//em').text
    title = tag.find_element(By.XPATH, 'div/div[@class="p-name p-name-type-2"]/a/em').text
    print(title)

driver.close()
```

## 案例：x麦网

```python
import time

import requests
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service

# 换成自己生成的代理
res = requests.get(
    url="https://dps.kdlapi.com/api/getdps/?secret_id=o60wwtxvs5ukaqqz18ai&num=1&signature=i6s9shfjfiogat5ijecbyfwwc5grwrzj&pt=1&format=json&sep=1")
proxy_string = res.json()['data']['proxy_list'][0]
print(f"获取代理：{proxy_string}")

service = Service("driver/chromedriver.exe")
opt = webdriver.ChromeOptions()
opt.add_argument(f'--proxy-server={proxy_string}')  # 代理
opt.add_argument('blink-settings=imagesEnabled=false')
opt.add_argument('--disable-infobars')
opt.add_experimental_option("excludeSwitches", ["enable-automation"])
opt.add_experimental_option('useAutomationExtension', False)
driver = webdriver.Chrome(service=service, options=opt)
driver.implicitly_wait(10)
with open('driver/hide.js') as f:
    driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {"source": f.read()})

# 1.打开大麦网
driver.get('https://www.damai.cn/')

# 2.搜索框+输入
tag = driver.find_element(
    By.XPATH,
    '//input[@class="input-search"]'
)
tag.send_keys("周杰伦")

# 3.点击搜索
tag = driver.find_element(
    By.XPATH,
    '//div[@class="btn-search"]'
)
tag.click()

# 4.查询列表
tag_list = driver.find_elements(
    By.XPATH,
    '//div[@class="search__itemlist"]//div[@class="items"]'
)
for tag in tag_list:
    title = tag.find_element(By.XPATH, 'div[@class="items__txt"]/div[1]/a').text
    print(title)


time.sleep(2000)
driver.close()

```



# 登录验证方式

## 图片识别方式

### ddddocr

 ```python
 import ddddocr
 import requests
 
 slice_bytes = requests.get("缺口图片地址").content
 bg_bytes = requests.get("背景图片地址").content
 
 
 slide = ddddocr.DdddOcr(det=False, ocr=False, show_ad=False)
 res = slide.slide_match(slice_bytes, bg_bytes, simple_target=True)
 x1, y1, x2, y2 = res['target']
 print(x1, y1, x2, y2)  # 114 45 194 125
 ```



### opencv

```python
import cv2
import numpy as np
import requests


def get_distance(bg_bytes, slice_bytes):
    def get_image_object(byte_image):
        img_buffer_np = np.frombuffer(byte_image, dtype=np.uint8)
        img_np = cv2.imdecode(img_buffer_np, 1)
        bg_img = cv2.cvtColor(img_np, cv2.COLOR_BGR2GRAY)
        return bg_img

    bg_image_object = get_image_object(bg_bytes)
    slice_image_object = get_image_object(slice_bytes)
    # 边缘检测
    bg_edge = cv2.Canny(bg_image_object, 255, 255)
    tp_edge = cv2.Canny(slice_image_object, 255, 255)

    bg_pic = cv2.cvtColor(bg_edge, cv2.COLOR_GRAY2RGB)
    tp_pic = cv2.cvtColor(tp_edge, cv2.COLOR_GRAY2RGB)

    res = cv2.matchTemplate(bg_pic, tp_pic, cv2.TM_CCOEFF_NORMED)

    min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(res)  # 寻找最优匹配
    x = max_loc[0]
    return x


slice_bytes = requests.get("缺口图片地址").content
bg_bytes = requests.get("背景图片地址").content
distance = get_distance(bg_bytes, slice_bytes)
print(distance)
```



### 打码平台

http://www.ttshitu.com/

![image-20231206094837737](images/image-20231206094837737-17058515911321.png)

```
# 一、图片文字类型(默认 3 数英混合)：
# 1 : 纯数字
# 1001：纯数字2
# 2 : 纯英文
# 1002：纯英文2
# 3 : 数英混合
# 1003：数英混合2
#  4 : 闪动GIF
# 7 : 无感学习(独家)
# 11 : 计算题
# 1005:  快速计算题
# 16 : 汉字
# 32 : 通用文字识别(证件、单据)
# 66:  问答题
# 49 :recaptcha图片识别

# 二、图片旋转角度类型：
# 29 :  旋转类型

# 三、图片坐标点选类型：
# 19 :  1个坐标
# 20 :  3个坐标
# 21 :  3 ~ 5个坐标
# 22 :  5 ~ 8个坐标
# 27 :  1 ~ 4个坐标
# 48 : 轨迹类型

# 四、缺口识别
# 18 : 缺口识别（需要2张图 一张目标图一张缺口图）
# 33 : 单缺口识别（返回X轴坐标 只需要1张图）
# 五、拼图识别
# 53：拼图识别
```

```python
import base64
import requests

bg_bytes = requests.get("背景图地址").content
b64_string = base64.b64encode(bg_bytes).decode('utf-8')

data = {"username": "wupeiqi", "password": "自己的密码", "typeid": 33, "image":b64_string }
res = requests.post("http://api.ttshitu.com/predict", json=data)
data_dict = res.json()
distance = data_dict['data']['result']
print(distance)
# {"success":true,"code":"0","message":"success","data":{"result":"136","id":"ztAkFAn1RvOJGsFhiAPuWg"}}
```



## 滑块验证

测试平台：https://www.geetest.com/adaptive-captcha-demo

![image-20231206093404644](images/image-20231206093404644.png)

![image-20231206093434520](images/image-20231206093434520.png)

示例：

- 使用闭包，进行复用等待执行的自定义函数

```python
import re
import time
import ddddocr
import requests
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver import ActionChains

service = Service("driver/chromedriver.exe")
driver = webdriver.Chrome(service=service)

# 1.打开首页
driver.get('https://www.geetest.com/adaptive-captcha-demo')

# 2.点击【滑动拼图验证】
tag = WebDriverWait(driver, 30, 0.5).until(lambda dv: dv.find_element(
    By.XPATH,
    '//*[@id="gt-showZh-mobile"]/div/section/div/div[2]/div[1]/div[2]/div[3]/div[3]'
))
tag.click()

# 3.点击开始验证
tag = WebDriverWait(driver, 30, 0.5).until(lambda dv: dv.find_element(
    By.CLASS_NAME,
    'geetest_btn_click'
))
tag.click()

# 4.读取背景图片,使用闭包方式，方便进行多次调用
def fetch_image_func(class_name):
    def inner(dv):
        tag_object = dv.find_element(
            By.CLASS_NAME,
            class_name
        )
        style_string = tag_object.get_attribute("style")
        match_list = re.findall('url\(\"(.*)\"\);', style_string)
        if match_list:
            return match_list[0]
    return inner
bg_image_url = WebDriverWait(driver, 30, 0.5).until(fetch_image_func("geetest_bg"))
slice_image_url = WebDriverWait(driver, 30, 0.5).until(fetch_image_func("geetest_slice_bg")) 

'''
  # 不使用闭包方式就得单独获取
# 读取背景图片
def fetch_bg(dv):
    tag_object = dv.find_element(
        By.CLASS_NAME,
        'geetest_bg'
    )
    style_string = tag_object.get_attribute("style")
    match_list = re.findall('url\(\"(.*)\"\);', style_string)  # ["http..." ]     []
    if match_list:
        return match_list[0]

bg_image_url = WebDriverWait(driver, 30, 0.5).until(fetch_bg_func)  # 新的函数 = 某个函数('geetest_bg')
print("背景图：", bg_image_url)

# 读取缺口图片
def fetch_slice(dv):
    tag_object = dv.find_element(
        By.CLASS_NAME,
        'geetest_slice_bg'
    )
    style_string = tag_object.get_attribute("style")
    match_list = re.findall('url\(\"(.*)\"\);', style_string)
    if match_list:
        return match_list[0]

slice_image_url = WebDriverWait(driver, 30, 0.5).until(fetch_slice_func)  # 新的函数 = 某个函数('geetest_slice_bg')
print("缺口图：", slice_image_url)
'''

slice_bytes = requests.get(slice_image_url).content
bg_bytes = requests.get(bg_image_url).content

# 图片识别，获取到缺口的坐标
slide = ddddocr.DdddOcr(det=False, ocr=False, show_ad=False)
res = slide.slide_match(slice_bytes, bg_bytes, simple_target=True)
x1, y1, x2, y2 = res['target']
print("滑动距离",x1)

# 等待图片框弹出来
def show_func(dv):
    geetest_box_tag = dv.find_element(By.CLASS_NAME, "geetest_box")
    display_string = geetest_box_tag.get_attribute("style")
    if "block" in display_string:
        time.sleep(2)  # 等两秒
        return dv.find_element(By.CLASS_NAME, 'geetest_btn')
btn_tag = WebDriverWait(driver, 30, 0.5).until(show_func)

# 使用selenium的ActionChains动作方法移动图片
ActionChains(driver).click_and_hold(btn_tag).perform()  # 点击并抓住标签
ActionChains(driver).move_by_offset(xoffset=x1, yoffset=0).perform()  # 向右滑动114像素（向左是负数）
ActionChains(driver).release().perform()

time.sleep(10)
driver.close()
```



## 图片验证码

``` python
import requests
import ddddocr

res = requests.get(url="https://xuexi.chinabett.com/Login/GetValidateCode/1701954700567")
with open("code.png", mode='wb') as f:
    f.write(res.content)
ocr = ddddocr.DdddOcr(show_ad=False)
code = ocr.classification(res.content)
print(code)
```



# IP检测和代理

如果网站进行了IP访问限制，例如：每个IP每天只能操作5次。此时可以选择购买IP，然后在请求时添加代理IP即可，具体步骤：

- 购买IP
- 登录购买IP渠道的后台，配置自己IP白名单
- 代码携带代理

![image-20231205090942499](images/image-20231205090942499.png)

![image-20231205091034550](images/image-20231205091034550.png)

![image-20231205091232846](images/image-20231205091232846.png)

![image-20231205092003682](images/image-20231205092003682.png)

![image-20231205092030587](images/image-20231205092030587.png)

![image-20231205092420969](images/image-20231205092420969.png)

```python
import time
import requests
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

# 换成自己生成的代理
res = requests.get(url="https://dps.kdlapi.com/api/getdps/?secret_id=o60wwtxvs5ukaqqz18ai&num=1&signature=i6s9shfjfiogat5ijecbyfwwc5grwrzj&pt=1&format=json&sep=1")
proxy_string = res.json()['data']['proxy_list'][0]
print(f"获取代理：{proxy_string}") # "182.106.136.218:40192"

service = Service("driver/chromedriver.exe")

opt = webdriver.ChromeOptions()
# opt.add_argument(f'--proxy-server=222.89.70.40:40001')  # 代理
opt.add_argument(f'--proxy-server={proxy_string}')  # 代理
driver = webdriver.Chrome(service=service, options=opt)


driver.get('https://myip.ipip.net/')

time.sleep(2000)
driver.close()
```



# js逆向分析案例

**用户名密码入参加密之后才传到后端**

例如在页面上输入的密码：any，但是提交后密码居然变成：c739492f2837ed5c6927914a55467874。

这其实是，在网页中的JS代码在发送请求之前，对我们的密码进行了处理（加密）。

那么，如果我们后续想要模拟请求发送时，必须要去网站中找到他的加密方式，然后用代码实现加密+请求发送。

而我们根据现象，去网站的js代码中寻找算法的行为，就称为js逆向。

注意：一般稍微正式点的网站，都会加入加密算法，即：爬虫时都需要逆向。



大搜车智云：https://zhiyun.souche.com/login

![image-20240114231517014](images/image-20240114231517014.png)



北大末名：https://bbs.pku.edu.cn/v2/home.php

![image-20240114231739664](images/image-20240114231739664.png)



## 简单base64加密（直接找到对应的js加密算法代码）

**案例一：北大末名**

北大末名：https://bbs.pku.edu.cn/v2/home.php

1、找登录的js代码

![image-20240114232005959](images/image-20240114232005959.png)

![image-20240114232210721](images/image-20240114232210721.png)

![image-20240114232403940](images/image-20240114232403940.png)

**实现：**

```python
import time,hashlib,requests

# 1.首页
res = requests.get(url="https://bbs.pku.edu.cn/v2/home.php")
cookie_dict = res.cookies.get_dict()

# 2.登录
user = "any"
pwd = "123123"
ctime = int(time.time())
data_string = f"{pwd}{user}{ctime}{pwd}"    # 需要加密的拼接字符串

obj = hashlib.md5()    
obj.update(data_string.encode('utf-8'))  # MD5进行加密
md5_string = obj.hexdigest()   

res = requests.post(
    url="https://bbs.pku.edu.cn/v2/ajax/login.php",
    data={
        "username": user,
        "password": pwd,
        "keepalive": "0",
        "time": ctime,
        "t": md5_string
    },
    cookies=cookie_dict
)

print(res.text)
```



**案例二**

媒想到：https://www.94mxd.com.cn/signin

![image-20240114235018838](images/image-20240114235018838.png)



## 自定义js加密算法（基于pyexecjs+js）

前提：需要安装node.js环境，pyexecjs依赖

假如在逆向分析时，发现某个js加密算法比较繁琐，用Python还原同样的算法比较费劲。此时，可以不必使用Python还原，而是利用Python去直接调用JavaScript中定义的功能。

想实现Python调用JavaScript代码，需如下步骤：

- 在电脑上安装node.js（软件）
- 安装Python的第三方模块pyexecjs
- 利用 pyexecjs 调用 nodejs 去执行JavaScript代码



**安装node.js**

最新版本：https://nodejs.org/en/download

历史版本：https://nodejs.org/en/about/previous-releases

![image-20230105174627063](images/image-20230105174627063.png)



![image-20230105175621435](images/image-20230105175621435.png)

![image-20230105175626438](images/image-20230105175626438.png)

![image-20230105175632497](images/image-20230105175632497.png)

![image-20230105175639342](images/image-20230105175639342.png)



安装完成之后，再进行如下环境变量的配置：

```
>>>npm root -g
```

![image-20220323231808029](images/image-20220323231808029.png)

**然后打开环境变量添加**

![image-20220323231854955](images/image-20220323231854955.png)



**安装pyexecjs依赖**

```python
pip3 install pyexecjs
```

运行测试

```python
import execjs

js_string = """
function func(arg) {
    return arg + '666';
}
"""
JS = execjs.compile(js_string)

sign = JS.call("func", "any")
print(sign)  # any666
```



**案例**：**X平台登录**

地址：https://xuexi.chinabett.com/

需求：

- 账户和密码加密
- 图片验证码

![image-20231207203031744](images/image-20231207203031744-17058549460162.png)

**分析：**

- **用户名**

![image-20231207203637480](images/image-20231207203637480.png)



![image-20231207203919565](images/image-20231207203919565.png)



![image-20231207203859185](images/image-20231207203859185.png)

找到用户名的加密算法并取出来

```javascript
// 用户名加密的算法
function base64encode(str) {
    var base64EncodeChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    var base64DecodeChars = new Array(
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, 62, -1, -1, -1, 63,
    52, 53, 54, 55, 56, 57, 58, 59, 60, 61, -1, -1, -1, -1, -1, -1,
    -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14,
    15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, -1, -1, -1, -1, -1,
    -1, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40,
    41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, -1, -1, -1, -1, -1);
    var out, i, len;
    var c1, c2, c3;
    len = str.length;
    i = 0;
    out = "";
    while (i < len) {
        c1 = str.charCodeAt(i++) & 0xff;
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt((c1 & 0x3) << 4);
            out += "==";
            break;
        }
        c2 = str.charCodeAt(i++);
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
            out += base64EncodeChars.charAt((c2 & 0xF) << 2);
            out += "=";
            break;
        }
        c3 = str.charCodeAt(i++);
        out += base64EncodeChars.charAt(c1 >> 2);
        out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
        out += base64EncodeChars.charAt(((c2 & 0xF) << 2) | ((c3 & 0xC0) >> 6));
        out += base64EncodeChars.charAt(c3 & 0x3F);
    }
    return out;
}
```



- **密码**

![image-20231207212846806](images/image-20231207212846806.png)

![image-20231207212827866](images/image-20231207212827866.png)

在页面中找到s1函数的加密代码并取出来

```python
import execjs

js_string = """
// 密码的加密算法
function s1() {
    var data = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z", "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z"];
    var r = Math.floor(Math.random() * 62);
    return data[r];
}

// 因为密码的加密算法逻辑不一样，所以套到一个加密函数进行实现，函数中逻辑为原网站的加密逻辑
function encryptPwd(password){
    //base64编码的密码每隔1位插入一个随机数 最后一位后面不插入
    var newPwd = [];
    var pwdlength = password.length;
    for (i = 0; i < pwdlength; i++) {
        newPwd.push(password[i]);
        if (i < pwdlength - 1)
            newPwd.push(s1());

    }
    var res = newPwd.join('');
    return res;
}
"""
JS = execjs.compile(js_string)

pwd = JS.call("base64encode", "123")
pwd_string = JS.call("encryptPwd", pwd)
print(pwd_string)
```



- **图片验证码**

![image-20231207211844426](images/image-20231207211844426.png)

```python
import requests
import ddddocr

# 使用ddddocr模块解密
res = requests.get(url="https://xuexi.chinabett.com/Login/GetValidateCode/1701954700567")
with open("code.png", mode='wb') as f:
    f.write(res.content)
ocr = ddddocr.DdddOcr(show_ad=False)
code = ocr.classification(res.content)
print(code)
```



- **整合实现**

```python
import execjs
import requests
import ddddocr
from bs4 import BeautifulSoup

# 1.首页请求
cookie_dict = {}
res = requests.get(url="https://xuexi.chinabett.com/")
cookie_dict.update(res.cookies.get_dict())

# 2.获取验证码地址
soup = BeautifulSoup(res.text, features="html.parser")
image_tag = soup.find(name="img", attrs={"id": "imgVerifity"})
code_src = image_tag.attrs['src']

# 3.读取验证码并实现
res = requests.get(url=f"https://xuexi.chinabett.com{code_src}", cookies=cookie_dict)
cookie_dict.update(res.cookies.get_dict())
ocr = ddddocr.DdddOcr(show_ad=False)
code = ocr.classification(res.content)

# 4.处理用户名&密码
js_string = """
function base64encode(str) {
    var base64EncodeChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    var base64DecodeChars = new Array(
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
    -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, 62, -1, -1, -1, 63,
    52, 53, 54, 55, 56, 57, 58, 59, 60, 61, -1, -1, -1, -1, -1, -1,
    -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14,
    15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, -1, -1, -1, -1, -1,
    -1, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40,
    41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, -1, -1, -1, -1, -1);
    var out, i, len;
    var c1, c2, c3;
    len = str.length;
    i = 0;
    out = "";
    while (i < len) {
        c1 = str.charCodeAt(i++) & 0xff;
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt((c1 & 0x3) << 4);
            out += "==";
            break;
        }
        c2 = str.charCodeAt(i++);
        if (i == len) {
            out += base64EncodeChars.charAt(c1 >> 2);
            out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
            out += base64EncodeChars.charAt((c2 & 0xF) << 2);
            out += "=";
            break;
        }
        c3 = str.charCodeAt(i++);
        out += base64EncodeChars.charAt(c1 >> 2);
        out += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
        out += base64EncodeChars.charAt(((c2 & 0xF) << 2) | ((c3 & 0xC0) >> 6));
        out += base64EncodeChars.charAt(c3 & 0x3F);
    }
    return out;
};

function s1() {
    var data = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z", "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z"];
    var r = Math.floor(Math.random() * 62);
    return data[r];
}

function encryptPwd(password){
    //base64编码的密码每隔1位插入一个随机数 最后一位后面不插入
    var newPwd = [];
    var pwdlength = password.length;
    for (i = 0; i < pwdlength; i++) {
        newPwd.push(password[i]);
        if (i < pwdlength - 1)
            newPwd.push(s1());

    }
    var res = newPwd.join('');
    return res;
}
"""
JS = execjs.compile(js_string)

# 用户名
username = JS.call("base64encode", "173xxx")
# 密码
temp = JS.call("base64encode", "123")
password = JS.call("encryptPwd", temp)

# 5.登录，携带参数数据
res = requests.post(
    url="https://xuexi.chinabett.com/Login/Entry",
    data={
        "userAccount": username,
        "password": password,
        "returnUrl": "/PersonalCenter",
        "proVing": code,
    },
    cookies=cookie_dict
)
print(res.text)
```



## js逆向-滑块验证

示例：x哪儿



请求分析：

![image-20240128213857056](images/image-20240128213857056.png)

![image-20240128214011210](images/image-20240128214011210.png)

![image-20240128213736327](images/image-20240128213736327.png)

![image-20240128225605318](images/image-20240128225605318.png)

![image-20240128225819698](images/image-20240128225819698.png)

此时已经看到o的操作，下一步判断e.data中的data的具体的值信息

![image-20240128230402164](images/image-20240128230402164.png)

![image-20240128230739966](images/image-20240128230739966.png)

![image-20240128231702645](images/image-20240128231702645.png)

![image-20240128232235032](images/image-20240128232235032.png)

![image-20240128233424802](images/image-20240128233424802.png)
