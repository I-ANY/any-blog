+++
title = "Django-bak"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 6
+++

# 框架

![image-20240331155628960](images/image-20240331155628960.png)



# 安装

1、conda 安装：

​	去到对应的虚拟环境下：conda install django==版本号  （如出现报错：missPackageNotFoundError: Packages missing in current channels，首先要保证当前使用的源有安装包，使用命令查找： conda search BeautifulSoup）
2、pip3安装：直接执行：pip3 install python==版本号



# 开始项目

## 创建项目

**1、命令行创建（不会自动创建templates目录，需要手工创建）：**

创建项目：

```bash
django-admin startproject [项目名称] （创建的时候先切换到对应的盘符下，默认是直接创建在当前目录下）
```

创建应用：

```bash
python manage.py startapp [应用名]
```

启动：

```bash
#切换到项目的目录下后执行：
python3 manage.py runserver [ip:port]
```

**注意：**
1、需要手动添加更改templates的文件路径settings.py：'DIRS' : os.path.join(BASE_DIR, 'templates')
2、创建应用之后需要在settings.py中注册，不注册的话django不会识别：

![image-20240331160222067](images/image-20240331160222067.png)



**2、pycharm创建（会自动创建templates目录）：**

创建：new project ----> django ---> 选择环境及编译器 ---> create

启动：直接在界面上点击运行：run

如果有报错需要更改settings.py中的tempates的路径：

![image-20240331160435865](images/image-20240331160435865.png)

创建应用：菜单Tools ---> Run Manage.py Task ，然后直接执行：startapp

![image-20240331160540286](images/image-20240331160540286.png)

更改端口号：

![image-20240331160553600](images/image-20240331160553600.png)



项目文件目录构成：

```bash
web01项目文件夹：
--web01                文件夹
  ---settings.py     配置文件
  ---urls.py             路由与视图对应关系（路由层）
  ---wsgi          wsgiref模块（不考虑）
--app01               应用文件夹
  ---admin.py      后台管理
  ---apps.py           注册使用
  ---migration.py   文件夹（数据库迁移记录）
  ---models.py     数据库相关的模型类（orm）
  ---tests.py      测试文件
  ---views.py         视图文件
--manage.py            django入口文件
--db.sqllite3            django自带的sqlite3小型数据库
```



## url 配置

| url方法 | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| url     | 支持正则匹配，实际上就是return re_path, django1.x默认方法，需要导入：from django.conf.urls import url  <br />无名分组：（正则）  <br />有名分组：(?P<组名>正则表达式) |
| path    | 只能绝对匹配路径地址，不支持正则匹配，django2.x以上默认方法，需要导入：from  django.urls import re_path, path`  `<br />例：path('index/<str:info>/'  ,views.index, name = index)<br /><>中转换器种类：<br />     str：匹配除路径分隔符外的任何非空字符串。<br />     int：匹配0或者任意正整数。<br />     slug：匹配任意一个由字母或数字组成的字符串。<br />     uuid：匹配格式化后的UUID。<br />     path：能够匹配完整的URL路径 |
| re_path | 支持正则匹配，django  2.x版本常用                            |

django1.x版本中默认使用 url：

```python
from django.conf.urls import url

url('^index/$', views.index) 
url('^home/(\d+)', views.index)      #支持正则
 
```

django2.x后默认采用 path:

```python
from django.urls import re_path, path

path("index/", views.index),      #不支持正则
re_path('^index/(\d+\w+)/$', views.index),     #支持正则
```

**多个app的urls设置**

1、在每一个app应用目录里面建有单独的urls.py文件，并在里面编辑的对应的urls地址（这里的url是各自app下面自己的url）；

![image-20240331161608285](images/image-20240331161608285.png)

2、设置总的urls:
注意：path('single/',include('TestPlatform.urls'))  # 这里引入子应用名(TestPlatform).urls

![image-20240331161628936](images/image-20240331161628936.png)



**有名无名反向解析：**

- 无名：path类似

  在urls.py文件中的地址块进行配置：

![image-20240331161717298](images/image-20240331161717298.png)

后端在views.py文件中进行接收： 

![image-20240331161755952](images/image-20240331161755952.png)

前端在html文件中进行接收：

![image-20240331161822056](images/image-20240331161822056.png)



- 有名：（其他用法跟无名一样，只是指定了其中的变量名）

![image-20240331161910698](images/image-20240331161910698.png)

在 urls.py 中给路由起别名，name="路由别名"：

```bash
re_path(r"^login/(?P<year>[0-9]{4})/$", views.login, name="login") 
```

在 views.py 中，从 django.urls 中引入 reverse，利用 reverse("路由别名"，kwargs={"分组名":符合正则匹配的参数}) 反向解析：

```bash
return redirect(reverse("login",kwargs={"year":3333}))
```

在模板 templates 中的 HTML 文件中利用 {% url "路由别名" 符合正则匹配的参数 %} 反向解析：

```python
<form action="{% url 'login' year=3333 %}" method="post">
```



## 三板斧

1、三板斧：

- HttpRespones：HttpRespones('xxxx')

​		用来返回字符串类型数据的，直接：return HttpRespones('Hello world!')

- render ：render('request', 'xxx.html')

​		用来返回html文件

- redirect ：redirect('/xxx/')

​		重定向（跳转网页，可以跳自己的也可以跳外部的，跳自己的可以直接写目录就好）

## 静态文件配置

静态文件：前段已经写好了的能够直接调用的文件（写好的js文件，css文件，图片文件，第三方前段框架）

网站所使用的html文件默认都是放在templates文件夹下；

网站所使用的静态文件都放在static文件夹下（但是django不会默认创建，需要手工进行创建）；

```bash
--static
  ---js
  ---css
  ---img
  ---plugins  #其他的第三方文件
```

在浏览器中输入url能够访问到对应的资源，说明后端提前开设了对应的接口；

在setings.py文件中配置路径：

![image-20240331161033639](images/image-20240331161033639.png)



# uWSGI

参考文档位置： [http://www.yuan316.com/post/Django3.2%E6%A1%86%E6%9E%B6/](http://www.yuan316.com/post/Django3.2框架/)

一种为了实现加载动态脚本而运行在Web服务器和Web应用程序中的通信接口，也可以理解为一份协议/规范。只有Web服务器和Web应用程序都实现了网关接口规范以后，双方的通信才能顺利完成。常见的网关接口协议：CGI，FastCGI，WSGI，ASGI。

<img src="images/image-20240331162457084.png" alt="image-20240331162457084" style="zoom: 67%;" />



- **FastCGI：**

CGI的一个扩展， 提升了性能，废除了 CGI fork-and-execute （来一个请求 fork 一个新进程处理，处理完再把进程 kill 掉）的工作方式，转而使用一种长生存期的方法，减少了进程消耗，提升了性能。

- **WSGI：**

用在 python web 框架编写的应用程序与后端服务器之间的规范（本例就是 Django 和 uWSGI 之间），让你写的应用程序可以与后端服务器顺利通信。

<img src="images/image-20240331162553471.png" alt="image-20240331162553471" style="zoom:67%;" />

- **ASGI：**

A指的是Async，异步的意思，是构建于WSGI接口规范之上的**异步服务器网关接口**，是WSGI的延伸和扩展。

django3.0，flask2.0版本 以后才支持

**补充说明：**

​	在 Python3.5 之后增加 async/await 特性之后简化了协程操作以后，异步编程变得异常火爆，越来越多开发者投入异步的怀抱。

​	3.0版本以前，django所提供的所有内部功能都是基于同步编程的。所以，在以往django开发中，针对网络请求，数据库读取等IO操作形成的阻塞，往往会导致项目运行性能的下降。虽然等待I/O操作数微秒时，但是随着流量的增加和操作的频率上升，这一点点的阻塞就会导致整个项目运作的缓慢。而如果换成异步就不会有任何阻塞，还可以同时处理其他任务，从而以较低的延迟处理更多的请求。所以在目前python开发中，越来越多的框架开始支持了异步编程。所以，3.0版本以后，django开始支持异步编程，可以让开发者在django中使用python第三方异步模块，推出了asgi异步web服务器。3.1版本推出了异步视图，当然，目前django的异步编程还不够完善，django中只有极少的功能是支持了异步操作。

​	django中运行runserver命令时，其实内部就启动了wsgiref模块作为web服务器运行的。wsgiref是python内置的一个简单地遵循了wsgi接口规范的web服务器程序。

- **uWSGI：**

​	是一个快速的，自我驱动的，对开发者和系统管理员友好的应用容器服务器，用于接收前端服务器转发的动态请求并处理后发给 web 应用程序。完全由 C 编写，实现了WSGI协议,uwsgi,http等协议。

注意：uwsgi 协议是一个 uWSGI服务器自有的协议,用于定义传输信息的类型，常用于uWSGI服务器与其他网络服务器的数据通信中。

- **uwasgi:**

​	是uWSGI服务器实现的独有的协议， 网上没有明确的说明这个协议是用在哪里的，我个人认为它是用于前端服务器与 uwsgi 的通信规范，相当于 FastCGI的作用。

![image-20240331162756087](images/image-20240331162756087.png)

![image-20240331162803566](images/image-20240331162803566.png)

![image-20240331162809762](images/image-20240331162809762.png)



# crf跨站请求

1、setttings.py文件打开验证：

<img src="images/image-20240331162945504.png" alt="image-20240331162945504" style="zoom:67%;" />

2、然后前端在form表单中任意位置添加模板语法，即可完成认证：{% csrf_token %}

<img src="images/image-20240331163003535.png" alt="image-20240331163003535" style="zoom:67%;" />

<img src="images/image-20240331163025007.png" alt="image-20240331163025007" style="zoom:67%;" />

ajax提交post请求：

第一种方法：

利用标签查找获取页面上的随机字符串：csrfmiddlewaretoken的值：

```python
'csrfmiddlewaretoken' : $('[name=csrfmiddlewaretoken]').val()
```

<img src="images/image-20240331163122330.png" alt="image-20240331163122330" style="zoom:67%;" />

第二种方法：

直接利用模板语法的快捷方式：

```python
'csrfmiddlewaretoken':'{{csrf_token}
```

第三种方法：

导入自己编写的一个js文件：mysetup.js（自定义命名）,文件中加上代码：

```js
function getCookie(name) {
    var cookieValue = null;
    if(document.cookie && document.cookie !== '') {
        var cookies = document.cookie.split(';');
        for(vari = 0; i < cookies.length; i++) {
            var cookie = jQuery.trim(cookies[i]);
            // Does this cookie string begin with the name we want?if(cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}
varcsrftoken = getCookie('csrftoken');
			
function csrfSafeMethod(method) {
  // these HTTP methods do not require CSRF protection
				return(/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
			}
			$.ajax Setup({
  beforeSend: function (xhr, settings) {
    if(!csrfSafeMethod(settings.type) && !this.crossDomain) {
      xhr.setRequestHeader("X-CSRFToken", csrftoken);
    }
  }
});

```

然后再html文件中导入：

<img src="images/image-20240331163245161.png" alt="image-20240331163245161" style="zoom:67%;" />



使用csrf的装饰器的用法实现局部校验：

导入模块：（需要校验：csrf_protect，不需要：csrf_exempt）

```python
from django.views.decorators.csrf import csrf_protect, csrf_exempt
```

1、在普通模式下直接在views.py模块中的函数前加上这个csrf装饰器(@csrf_protect || @csrf_exempt)即可实现局部校验与不校验

2、在CBV的模式下，需要注意（即便settings中没有设置全局中间件或者设置）：
		针对@csrf_protect，符合CBV加装饰的三种玩法，都生效

​		针对@csrf_exempt，只能给dispatch方法加才生效



# 跨域请求

​	跨域请求（Cross-Origin Request）指的是在浏览器端（其他端像postman请求等不影响），使用JavaScript发起HTTP请求时，请求的目标资源位于不同的域名、端口或协议。浏览器为了安全考虑，实施了同源策略（Same-Origin Policy），该策略限制了跨域请求的执行。

​	同源策略要求请求发起方的协议、域名和端口与目标资源的协议、域名和端口完全一致，否则会被浏览器拦截。然而，现实场景中经常需要进行跨域请求，比如在前端开发中调用第三方的API接口或与不同域名的服务器进行数据交互。

 请求氛围两种：

- 简单请求，复杂请求

- 简单请求（必须满足下述条件）

​		HTTP方法为这三种方法之一：HEAD、GET、POST

​		HTTP头消息不超出以下字段：

​		Accept、Accept-Language、Content-Language、Last-Event-ID

​		且Content-Type只能为下列类型中的某一个：

```bash
- application/x-www-from-urlencoded
- multipart/form-data
- text/plain.

# ==任何不满足上述要求的请求，都会被认为是复杂请求
```

为了允许跨域请求的执行，可以采取以下一些常见的解决方案：

1. **CORS（Cross-Origin Resource     Sharing）：**服务器端设置响应头，允许特定域名的请求访问资源。通过在服务器端设置Access-Control-Allow-Origin响应头，指定允许的域名，可以解决跨域请求的问题。
2. **JSONP（JSON with Padding）：**利用<script>标签的跨域特性，动态创建<script>标签引入一个包含回调函数的URL。服务器将返回的数据包裹在回调函数中，从而实现跨域请求并获取数据。
3. **代理服务器：**在同源策略下，通过在服务器端进行中转，将浏览器请求转发到目标服务器，并将响应返回给浏览器。这样，浏览器与代理服务器之间就属于同源，可以绕过跨域限制。
4. **WebSocket：**使用HTML5提供的WebSocket协议进行跨域通信。WebSocket建立持久连接后，双方可以通过WebSocket对象发送和接收数据，不受同源策略的限制。 

使用cors-headers组件处理复杂请求：

文档： https://github.com/ottoyiu/django-cors-headers/

```bash
# 安装
pip install django-cors-headers-i https://pypi.douban.com/simple/
```

```python
# 添加应用，settings.dev.py，代码：
INSTALLED_APPS= (
  ...
   'rest_framework',
   'corsheaders',
  ...
)

# 中间件设置【必须写在第一个位置】，settings.dev.py，代码：
MIDDLEWARE= [
   'corsheaders.middleware.CorsMiddleware', #放在中间件的最上面，就是给响应头加上了一个响应头跨域
  ...
]

 
# 需要添加跨域白名单，确定一下哪些客户端可以跨域。settings.dev.py，代码：
# CORS组的配置信息
CORS_ORIGIN_WHITELIST= (
   # 如果这样写不行的话，就加上协议(http://www.uric.cn:8080，因为不同的corsheaders版本可能有不同的要求)
   #'www.uric.cn:8080', 
   'http://www.uric.cn:8080',
)
# 是否允许ajax跨域请求时携带cookie，False表示不用，我们后面也用不到cookie，所以关掉它就可以了，以防有人通过cookie来搞我们的网站
CORS_ALLOW_CREDENTIALS= False 
 
  
# 允许客户端通过api.uric.cn访问Django项目，settings.dev.py，代码：
ALLOWED_HOSTS= ["api.uric.cn",]
 
```

完成了上面的步骤，我们将来就可以通过后端提供数据给前端使用ajax访问了。前端使用 axios就可以访问到后端提供给的数据接口，但是如果要附带cookie信息，前端还要设置一下，这个等我们搭建客户端项目时再配置。



# ORM

参考博客：[ORM详情参考](https://www.cnblogs.com/Neeo/articles/10967645.html#一对一添加记录)

## **建/改表**

​	在modules.py文件中新建类，用于创建表，表的名称等于项目名称+类名

```python
from django.db import models
import django.utils.timezone

# 用户信息表
class UserInfo(models.Model):
  	id = models.AutoField(primary_key=True, verbose_name="主键")
    username = models.CharField(max_length=32, db_index=True, verbose_name='用户名')
    password = models.CharField(max_length=32, verbose_name='密码')
    email = models.EmailField(max_length=32, verbose_name='邮箱')
    mobile_phone = models.CharField(max_length=32, verbose_name='手机号')
    		
    # 设置表名
    class Meta:
        db_table = 'userInfo'
    
    # 设置控制台打印的结果
    def __str__(self):
        return 'username：{0}'.format(self.username)
```

​	表结构的增删改都在modles.py里面操作，增加列或者修改列属性就直接在原来的代码上更改，删除列就直接在代码中注释或者删除对应列的代码即可（注意：一定一定一定要看好，因为注释或者删除且提交数据之后这部分的数据就丢了）修改了models之后一定要记得提交才生效：

```bash
pythonmanage.py makemigrations  	#将操作记录记录到migrations文件中
pythonmanage.py migrate          	#提交到数据库当中
```

运行的时候提示：

```bash
You have unapplied migrations; your app may not work properly until they are applied. Run 'python manage.py migrate' to apply them.
```

**常用字段及参数：**

| 方法            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| AutoField()     | int自增列，必须填入参数 primary_key=True。当model中如果没有自增列，则自动会创建一个列名为id的列 |
| IntegerField()  | 整数类型，范围在 -2147483648 to 2147483647。(一般不用它来存手机号(位数也不够)，直接用字符串存，) |
| CharField()     | 字符类型(对应mysql中的verchar)，必须提供max_length参数， max_length表示字符长度。 |
| DateField()     | 日期字段，日期格式 YYYY-MM-DD，相当于Python中的datetime.date()实例  有可选参数：  auto_now：当对象被保存时(更新或添加都行),自动将该字段的值设置为当前时间.通常用于表示 "last-modified" 时间戳。  auto_now_add：当对象首次被创建时，自动将该字段的值设置为当前时间.通常用于表示对象创建时间。 |
| DateTimeFiele() | 日期时间字段，格式 YYYY-MM-DD HH:MM[:ss[.uuuuuu]][TZ]，相当于Python中的datetime.datetime()实例。 |
| DecimalField()  | 浮点数，必须要提供两个参数：  总位数：max_digits  小数位数：decimal_places |
| BooleanField()  | True/False字段，可以用来处理checkbox                         |
| TextField()     | 容量很大的文本字段，通常用来存储textarea标签值               |
| EmailField()    | 带有检查email合法性的CharFeild，不接受max_lenght参数         |
| FileField()     | 文件上传字段，必须入参：  upload_to：保存上载文件的本地文件系统路径 |
| IamgeField()    | 图片上传，会校验上传对象是否是一个合法图片，有两个参数：（如果提供这两个参数，则图片将按提供的高度和宽度规格保存）  height_field  width_field |
| URLField()      | 用于保存 URL，若 verify_exists参数为 True(默认)，给定的 URL 会预先检查是否存在( 即URL是否被有效装入且没有返回404响应) |
| BinaryField()   | 二进制类型                                                   |

**抽象类(abstract)与代理模型(proxy)**

抽象类不会生成对应的数据库表，其他类可以继承这个类

```python
from django.db import models


class CommonModel(models.Model):
    # 自定义模型的基类
    created_at = models.DateTimeField('注册时间', auto_now_add=True)
    updated_at = models.DateTimeField('最后更新时间', auto_now=True)

    class Meta:
        # 抽象类，这个类不会生成对应的数据库表
        abstract = True


class User(CommonModel):
    """
    用户模型
    """
    name = models.CharField('姓名', max_length=64)
    sex = models.CharField('性别', max_length=1, choices=(
        ('1', '男生'),
        ('0', '女生'),
    ), default=1)
    age = models.PositiveIntegerField('年龄', default=1)
    username = models.CharField('用户名', max_length=64, unique=True)
    password = models.CharField('密码', max_length=256)
    email = models.CharField('邮箱', max_length=64, null=True, blank=True)
    remark = models.CharField('备注', max_length=64, null=True, blank=True)

    # 修改表的名
    class Meta:
        db_table = 'user'

# 代理上面自定义模型类的所有功能，可以对功能进行扩充
class Manager(User):
    class Meta:
        proxy = True
```

**自定义字段：**

```python
from django.db import models

# Create your models here.
#Django中没有对应的char类型字段，但是我们可以自己创建
class FixCharField(models.Field):
    '''
    自定义的char类型的字段类
    '''
    def __init__(self,max_length,*args,**kwargs):
        self.max_length=max_length
        super().__init__(max_length=max_length,*args,**kwargs)

    def db_type(self, connection):
        '''
        限定生成的数据库表字段类型char，长度为max_length指定的值
        :param connection:
        :return:
        '''
        return 'char(%s)'%self.max_length


#自定义及使用：应用上面自定义的char类型
class Class(models.Model):
    id=models.AutoField(primary_key=True)
    title=models.CharField(max_length=32)
    class_name=FixCharField(max_length=16)     # 自定义的char类型
    gender_choice=((1,'男'),(2,'女'),(3,'保密'))
    gender=models.SmallIntegerField(choices=gender_choice,default=3)
```

## 外键关联

基础用户表模型

```python
class User(models.Model):
    """
    用户模型
    """
    name = models.CharField('姓名', max_length=64)
    sex = models.CharField('性别', max_length=1, choices=(
        ('1', '男生'),
        ('0', '女生'),
    ), default=1)
    age = models.PositiveIntegerField('年龄', default=1)
    username = models.CharField('用户名', max_length=64, unique=True)
    password = models.CharField('密码', max_length=256)
    email = models.CharField('邮箱', max_length=64, null=True, blank=True)
    remark = models.CharField('备注', max_length=64, null=True, blank=True)

    # 修改表的名
    class Meta:
        db_table = 'user'
```

1. **ForeignKey（一对多关系）：**

     - 描述：ForeignKey用于建立一对多关系，其中一个模型拥有对另一个模型的外键引用。

     - 场景：当一个模型对象关联到另一个模型对象，并且一个模型可以拥有多个关联对象时，可以使用ForeignKey。例如，一个问题(Question)可以有多个答案(Answer)。

   ```python
   # 外键盘关联：
   class Question(models.Model):
       """ 问题 """
       name = models.CharField('问题名称', max_length=64)
   
       
   class Answer(models.Model):
       """ 答案 """
       question = models.ForeignKey(Question, on_delete=models.CASCADE,
                                    related_name='answers',
                                    verbose_name='关联的问题')
       context = models.TextField('答案的内容')
   ```

   自关联：ForeignKey(to=自身)

     - 描述：通过设置外键字段指向模型自身，实现模型对象之间的自关联关系。

     - 场景：当模型对象之间存在层级结构，需要建立父子或上下级关系时，可以使用自关联。例如，一个部门(Department)可以有多个子部门，同时每个部门也可以有一个父部门。

   ```python
   # 内键关联:
   class Department(models.Model):
       """
       分类
       """
       name = models.CharField('部门名称', max_length=64)
       parent = models.ForeignKey('self', related_name='IT',
                                  on_delete=models.CASCADE)
   ```

2. **OneToOneField（一对一关系）：**

     - 描述：OneToOneField用于建立一对一关系，其中一个模型对象与另一个模型对象唯一相关联。

     - 场景：当一个模型对象只能与另一个模型对象建立唯一的关联时，可以使用OneToOneField。例如，一个用户(User)只能有一个用户配置文件(Profile)。

   ```python
   class Profile(CommonModel):
       """
       用户详细信息
       """
       user = models.OneToOneField(User, on_delete=models.CASCADE,
                                   related_name='profile',
                                   db_column='user_id')
       nickname = models.CharField('昵称', max_length=64)
   ```

3. **ManyToManyField（多对多关系，会自动生成第三张表来进行关联）：**

     - 描述：ManyToManyField用于建立多对多关系，其中一个模型对象可以与多个另一个模型对象相关联，并且每个另一个模型对象也可以与多个该模型对象相关联。

     - 场景：当两个模型对象之间存在多对多的关联关系时，可以使用ManyToManyField。例如，一个本书可以有多个作者，而一个作者也可以与多本书相关联。

   ```python
   from django.db import models  
     
   class Author(models.Model):  
       name = models.CharField(max_length=100)  
     
       # 一个作者可以有多本书，一本书也可以有多个作者  
       books = models.ManyToManyField('Book', related_name='authors')  
     
   class Book(models.Model):  
       title = models.CharField(max_length=100)  
       # 其他字段...
       
   ```

   ```python
   # 创建一个作者和一本书  
   author = Author.objects.create(name='John Doe')  
   book = Book.objects.create(title='My Book')  
     
   # 将这本书添加到作者的书籍列表中  
   author.books.add(book)  
     
   # 从作者的书籍列表中移除这本书  
   author.books.remove(book)  
     
   # 清空作者的书籍列表  
   author.books.clear()  
     
   # 设置作者的书籍列表（这将删除之前的所有关系，并添加新的关系）  
   author.books.set([Book.objects.get(title='Another Book')])
   ```

参数

1、**on_delete**：关联删除

-  CASCADE，默认值，即级联删除。这意味着，当关联的对象被删除时，相关的对象也会被删除。
- PROTECT：阻止删除操作，如果有关联的对象存在。
- SET_NULL：将关联字段设置为 NULL。
- SET_DEFAULT：将关联字段设置为默认值。
- SET()：将关联字段设置为指定的值。
- DO_NOTHING：不执行任何操作。

2、**related_name**：反向关系

​	指定反向关联的名称，如果未指定，django会根据模型名称和关系字段自动生成  一个模型可能会有多个外键字段，就需要通过related_name参数来给这些外键字段指定不同的名称，避免引起混淆。

3、**related_query_name**

​	指定反向关联的查询名称，如果未指定，django会根据模型名称和关系字段自动生成

## **增删改查**

**增**

```python
from models import UserInfo

#使用到对象，主键值可以使用关键字pk自动匹配到当前表的主键字段  
res = UserInfo.objects.create(column1=value1, column2=value2, …)
# 或者
obj = UserInfo.objects(column1=value1, column2=value2, …)
obj.save()
```

**删**

```python
models.[对应的类名].objects.filter().delete()
```

**改**

方式1：

```python
models.[对应的类名].objects.filter().update(column1=value1,  column2=value2, …)
```

方式2：（使用对象批量提交，当数据量非常大的时候，执行效率非常低）

```python
obj = models.[类名(表名)].objects.first()
obj.column1 = value1
obj.column2 = value2
......
obj.save()
```

**查**

注意：查询返回的都是一个queryset的数据对象（类似列表的数据结构，存放一条或多条记录对象）

- 简单查询：

```python
from accounts.models import User

# 查询单条数据:get()
user_obj = User.objects.get(username='张三', id = 1)
# 查询所有数据:all()
list_user_obj = User.objects.all()

# 返回第一条/最后一条记录first、last
# 第一条记录
user_obj = User.objects.first()
# 最后一条记录
user_obj = User.objects.last()

# 有则返回，没有则创建后返回（返回值为两个：对象、是否为新创建）
ZhangSan = User.objects.get_or_create(username = 'ZhangSan',password = '123456')
# (<User: username：ZhangSan>, False)

# 根据字段返回最晚：latest，最早的一条记录：earliest
# 最晚的一条记录
la = User.objects.latest('created_time')
# 最早的一条记录
ea = User.objects.earliest('created_time')

# 总记录数:count()
user_list = User.objects.all()
user_list.count()
# 103

# 判断结果集中是否有数据:exists()
user_list = User.objects.all()
user_list.exists()


# 指定获取的字段
# 返回列表套字典形式的对象：queryset
obj = models.User.objects.values(column1, column2,  …) 
# 返回元组套字典形式的对象：queryset   
obj = models.User.objects.values_list(column1,  column2, …) 
```

- 条件查询

```python
from accounts.models import User

# 条件过滤查询：  
objs = models.User.objects.filter(status=9)

# 条件查询并取第一个
first_obj = models.User.objects.filter(status=9).first
# 或者
objs = models.User.objects.filter(status=9)
first_obj = obj[0]

# 排除查询：（它包含了与所给筛选条件不匹配的对象） 
obj = models.User.objects.exclude(status=9) 
```

- 其中filter()内部的条件：

等于

```python
# 等于（区分大小写）
User.objects.filter(id = 6)
# 等同于
User.objects.filter(id__exact = 6)

# 等于（不区分大小写）
User.objects.filter(nickname__iexact = 'Lisi')
```

大/小于

```python
# 大于: __gt
User.objects.filter(status__gt = 0)
# 大于或等于某个值: __gte
User.objects.filter(status__gte = 0)

# 小于某个值: __lt
User.objects.filter(status__lt = 10)
# 小于或等于某个值: __gte
User.objects.filter(status__lte = 10)

```

其他

```python
# 是否为空值: __isnull
User.objects.filter(nickname__isnull = True)
User.objects.filter(nickname__isnull = False)

# 是否包含某字符串(区分大小写): __contains
User.objects.filter(username__contains = 'Lisi')
# 不区分大小写: __icontains
User.objects.filter(username__icontains = 'Lisi')

# 是否在列表内容内: __in
User.objects.filter(username__in = ['Zhangsan','Lisi'])

# 以**开始/结束
# 区分大小写
User.objects.filter(username__startswith = 'Zhan')
User.objects.filter(username__endswith = 'si')
# 不区分大小写
User.objects.filter(username__istartswith = 'Zhan')
User.objects.filter(username__iendswith = 'si')
```

日期和时间

```python
from datetime import datetime

# 日期: __date
d = datetime(2025,2,22).date()
User.objects.filter(updated_at__date = d)

# 年: __year
User.objects.filter(updated_at__year = 2025)
# 月: __month
User.objects.filter(updated_at__month = 2)
# 日: __day
User.objects.filter(updated_at__day = 22)

# 时分秒: __hour/__minute/__second
# 时查询
User.objects.filter(updated_at__hour = 8)
# 分查询
User.objects.filter(updated_at__minute = 23)
# 秒查询
User.objects.filter(updated_at__second = 23)
# 时分秒查询
User.objects.filter(updated_at__hour = 8).filter(updated_at__minute = 23).filter(updated_at__second =23)

# 星期: __week/__week_day
# 查询uodated_at字段在第1周的数据
User.objects.filter(updated_at__week = 1)
# 查询uodated_at字段在周2的数据
# __week_day字段1是从周日开始计算1代表周日，2代表周一
"""
__week_day数值：   1，   2，    3，   4，   5，    6，   7
实际查询的周几值：  周日， 周一， 周二， 周三， 周四， 周五， 周六
"""
User.objects.filter(updated_at__week_day =3)
```

外键关联查询

```python
from accounts.models import UserProfile

UserProfile.objects.filter(user__nickname = '张三')
```

多个条件的查询—filter和&运算符

```python
from accounts.models import User

User.objects.filter(nickname__icontains = '管理员').filter(status = 1)
# 同下效果一样
User.objects.filter(nickname__icontains = '管理员',status = 1)

# &运算符
q1 = User.objects.filter(nickname__icontains = '管理员')
q2 = User.objects.filter(status = 1)
q1 & q2
```



查看内部sql语句的方式：

1、是queryset对象的可以直接使用：

```python
prient(obj.query)  
```

2、在setting.py文件中加上配置：



| 动作 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| 增   |                                                              |
| 删   |                                                              |
| 改   | <br />方式2：  <br />                                        |
| 查   |                                                              |
| 其他 | 1、结果排序（升序）：obj = models.[对应的类名].objects.order_by(\*field)  <br />2、结果排序（降序）：obj = models.[对应的类名].objects.reverse()  <br />3、count()：匹配查询（Queryset）的数据量  <br />4、first()：返回第一条数据  <br />5、last()：返回最后一条数据  <br />6、exists()：如果包含Qyeryset数据就返回True，不包含就返回False  <br />7、values(\*Field)：返回一个ValueQuerySet——一个特殊的QuerySet，运行后得到的并不是一系列model的实例化对象，而是一个可迭代的字典序列  <br />8、values_list(*Field)：返回的是一个元组序列，values返回的是一个字典序列  <br />9、distinct()：结果中删除重复记录 |
|      |                                                              |

参数：

- null：如果为True，Django 将用NULL 来在数据库中存储空值。 默认值是 False.

- blank：如果为True，该字段允许不填。默认为False。要注意，这与 null 不同。null纯粹是数据库范畴的，而 blank 是数据验证范畴的。如果一个字段的blank=True，表单的验证将允许该字段是空值。如果字段的blank=False，该字段就是必填的。

- default：字段的默认值。可以是一个值或者可调用对象。如果可调用     ，每有新对象被创建它都会被调用，如果你的字段没有设置可以为空，那么将来如果我们后添加一个字段，这个字段就要给一个default值。

- primary_key：如果为True，那么这个字段就是模型的主键。如果你没有指定任何一个字段的primary_key=True，Django 就会自动添加一个IntegerField字段做为主键，所以除非你想覆盖默认的主键行为，否则没必要设置任何一个字段的primary_key=True。

- unique：如果该值设置为 True, 这个数据字段的值在整张表中必须是唯一的。

- choices：由二元组组成的一个可迭代对象（例如，列表或元组），用来给字段提供选择项。     如果设置了choices ，默认的表单将是一个选择框而不是标准的文本框，而且这个选择框的选项就是choices 中的选项。

  ```bash
  # 使用：首先定义一个元组套元组的变量：
  test_choices =((),(),(),(),…) 
  # 然后直接在定义字段的时候加上参数：
  choices= test_choices，
  
  # 固定样式调用：
  *.get_[字段]_choices
  ```

- db_index：如果db_index=True 则代表着为此字段设置一个普通索引。

- auto_now_add和auto_now：DatetimeField、DateField、TimeField这个三个时间字段，都可以设置这两个属性。

- - 置auto_now_add=True，创建数据记录的时候会把当前时间添加到数据库。
  - 配置上auto_now=True，每次更新数据记录的时候会更新该字段，标识这条记录最后一次的修改时间。
  - 注意，如果使用xx.object.filter(id=1).update()去更新记录时，update不能触发自动更新时间的auto_now的"被动"！所有，你update时，需要手动给该字段传入当前时间xx.object.filter(id=1).update(pub_date=datetime.datetime.now())。或者你使用save的方式来更新记录。

- to：外键关联字段，指定被关联的模型类。

- to_field="id"：跟to搭配使用，不指定to_field参数，外键关联时会自动找主键，指定了就按照你指定的找，注意，被关联的字段，必须具  有唯一约束。

**分组查询：**

关键字：annotate() 

```python
# models后面(.)什么就是按什么分组 
models.Book.objects.annotate() 	# 这里就是按照书籍表每本书来分组
```



## Q/F查询

**F查询：**

F() 的实例可以在查询中引用字段，来比较同一个 model 实例中两个不同字段的值。还支持支持 F() 对象之间以及 F() 对象和常数之间的加减乘除和取模的操作。基于此可以对表中的数值类型进行数学运算

也就是说：F可以帮我们取到表中某个字段对应的值来当作我的筛选条件，而不是我认为自定义常量的条件了，实现了动态比较的效果；

示例：

查询出卖出数大于库存数的商品：

```python
from django.db.models import F
ret1 = models.Product.objects.filter(maichu__gt = F('kucun')) 
```

**Q查询：**

`Q` 对象用于构建复杂的查询，它允许你组合多个查询条件，并指定它们之间的逻辑关系（如AND、OR）。

filter() 等方法中逗号隔开的条件是与的关系。 如果你需要执行更复杂的查询（例如OR、AND语句），你可以使用Q对象。

示例：

假设你有一个`User`模型，其中有两个布尔字段：`is_active` 和 `is_staff`。如果你想要查询所有活跃的员工（即`is_active`为True且`is_staff`也为True）或者所有不活跃的用户（即`is_active`为False），可以这样做：

```python
from django.db.models import Q  
  
User.objects.filter(Q(is_active=True, is_staff=True) | Q(is_active=False))
# 或者
User.objects.filter(Q(is_active=True, is_staff=True) | Q(is_active=False, is_staff=False))
```

使用了两个`Q`对象，并通过`|`（OR）操作符将它们组合在一起。第一个`Q`对象表示`is_active`为True且`is_staff`也为True的条件，第二个`Q`对象表示`is_active`为False的条件。

```python
from django.db.models import Q

from accounts.models import User
# 查询昵称中包含管理员并且status是1的用户
query1 = Q(nickname__icontains = '管理员' , status = 1)
User.objects.filter(query1)
```

# FBV & CBV

针对于视图函数(views.py)，视图函数编写逻辑可以使用两种模式：

- FBV：function+base+views（直接在views里面写函数）
- CBV：class+base+views（在views里面写类，继承：Views）

编写方式的区别：

url.py

```python
from django.conf.urls import path
from app01 import views

urlpatterns = [
  path('register/', views.register, name='register'),  # FBV  请求进来执行的是register()
  
  path('user/', views.UserView.as_view()),  # CBV
  #path('user/<int:id>', views.UserView.as_view()),  # CBV，带有参数：id
]
```

view.py

```python
from django.http import JsonResponse
from django.views import View

from rest_framework.views import APIView

# Django FBV 写法
def register(request):
    '''	
    request.method
    request.POST
    request.GET
    request.body
    '''
    if request.method == "GET":
      	return JsonResponse({'status': True, 'msg': 'GET'})
    elif request.method == "POST":
      	return JsonResponse({'status': True, 'msg': 'POST'})
    elif request.method == "PUT":
      	return JsonResponse({'status': True, 'msg': 'PUT'})
  
# Django CBV写法：使用django内部的view
# 导入模块：from django.views import Vie
class UserView(View):
    def dispatch(self, request, *args, **kwargs):         # 这里我们也可以重写父类的dispatch方法
        print('dispatch')
        obj=super().dispatch(request,*args,**kwargs)
        return obj

    # 定义get方法
    #def get(self, request, id):    						# 接收id参数
    #def get(self, request, *args, **kwargs):   # 或者直接写成接收不定参数
    def get(self, request):
      	pass

    # 定义post方法
    def post(self, request, *args, **kwargs):
      	pass

    def put(self, request, *args, **kwargs):
      	pass

    def delete(self, request, *args, **kwargs):
      	pass
```



# Form & Moduleform

form与moduleform的区别：

1、form 强大的数据验证功能（没有数据库保存数据操作的情况）

2、modelform 强大是数据验证，适中的数据操作（有数据库数据写入操作的情况）

 

- **Form：**

其实form组件的主要功能如下:

1. 生成页面可用的HTML标签；

2. 对用户提交的数据进行校验；

3. 保留上次输入内容；

views.py模块中进行导入然后定义类：

简单示例：

```python
class UserForm(forms.Form): # 类名随意，但必须继承forms.Form
  # min_length、max_length都是规则
  # 字段名字和数据库中的表字段保持一致
  user = forms.CharField(min_length=3, max_length=6)
  pwd = forms.CharField()
  email = forms.EmailField()
```

其他方法：

```bash
is_valid()：form组件类中定义的所有字段，都通过其规则校验的话，返回True，有一项或者若干项校验失败，返回False。
cleaned_data：只有先经过is_valid校验后，才能使用该方法进行获取干净的数据。
errors：校验失败的错误信息，是个字典，存放一个或多个错误信息。
errors.as_json()：校验失败的错误信息直接转为json数据。c
```

示例：
<img src="images/image-20240331222608317.png" alt="image-20240331222608317" style="zoom:67%;" />

**Form所有内置字段**

| 字段                                          | 说明                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| Field                                         | required=True,        是否允许为空    <br />widget=None,         HTML插件    <br />label=None,         用于生成Label标签或显示内容    <br />initial=None,        初始值   <br />help_text='',        帮助信息(在标签旁边显示)    <br />error_messages=None,     错误信息 {'required':  '不能为空', 'invalid': '格式错误'}    <br />validators=[],        自定义验证规则   <br /> localize=False,       是否支持本地化    <br />disabled=False,       是否可以编辑   <br /> label_suffix=None      Label内容后缀 |
| CharField(Field)                              | max_length=None,       最大长度    <br />min_length=None,       最小长度    <br />strip=True          是否移除用户输入空白 |
| IntegerField(Field)                           | max_value=None,       最大值    <br />min_value=None,       最小值 |
| FloatField(IntegerField)                      | ...                                                          |
| DecimalField(IntegerField)                    | max_value=None,       最大值    <br />min_value=None,       最小值    <br />max_digits=None,       总长度    <br />decimal_places=None,     小数位长度 |
| BaseTemporalField(Field)                      | input_formats=None     时间格式化      <br />DateField(BaseTemporalField)  格式：2015-09-01  TimeField(BaseTemporalField)  格式：11:12  DateTimeField(BaseTemporalField)格式：2015-09-01 11:12     DurationField(Field)      时间间隔：%d  %H:%M:%S.%f    ... |
| RegexField(CharField)                         | regex,           自定制正则表达式    <br />max_length=None,      最大长度    <br />min_length=None,      最小长度    <br />error_message=None,     忽略，错误信息使用  <br />error_messages={'invalid': '...'} |
| EmailField(CharField)                         | ...                                                          |
| FileField(Field)                              | allow_empty_file=False   是否允许空文件                      |
| ImageField(FileField)                         | ...    <br />注：需要PIL模块，pip3 install Pillow    <br />以上两个字典使用时，需要注意两点：      <br />- form表单中  enctype="multipart/form-data"      <br />- view函数中 obj = MyForm(request.POST,  request.FILES) |
| URLField(Field)                               | ...                                                          |
| BooleanField(Field)                           | ...                                                          |
| NullBooleanField(BooleanField)                | ...                                                          |
| ChoiceField(Field)                            | ...    <br />choices=(),        选项，如：choices = ((0,'上海'),(1,'北京'),)    <br />required=True,       是否必填    <br />widget=None,        插件，默认select插件    <br />label=None,        Label内容    <br />initial=None,       初始值    <br />help_text='',       帮助提示 |
| ModelChoiceField(ChoiceField)                 | ...                <br />django.forms.models.ModelChoiceField    queryset,         # 查询数据库中的数据    <br />empty_label="---------",  # 默认空显示内容    <br />to_field_name=None,    # HTML中value的值对应的字段    <br />limit_choices_to=None   # ModelForm中对queryset二次筛选 |
| ModelMultipleChoiceField(ModelChoiceField)    | django.forms.models.ModelMultipleChoiceField                 |
| TypedChoiceField(ChoiceField)                 | coerce = lambda val: val  对选中的值进行一次转换    empty_value= ''      空值的默认值 |
| MultipleChoiceField(ChoiceField)              | ...                                                          |
| TypedMultipleChoiceField(MultipleChoiceField) | coerce = lambda val: val  对选中的每一个值进行一次转换    <br />empty_value= ''      空值的默认值 |
| ComboField(Field)                             | fields=()         使用多个验证，如下：即验证最大长度20，又验证邮箱格式                   fields.ComboField(fields=[fields.CharField(max_length=20),  fields.EmailField(),]) |
| MultiValueField(Field)                        | PS: 抽象类，子类中可以实现聚合多个字典去匹配一个值，要配合MultiWidget使用 |
| SplitDateTimeField(MultiValueField)           | input_date_formats=None,  格式列表：['%Y--%m--%d', '%m%d/%Y', '%m/%d/%y']    <br />input_time_formats=None  格式列表：['%H:%M:%S', '%H:%M:%S.%f', '%H:%M'] |
| FilePathField(ChoiceField)                    | 文件选项，目录下文件显示在页面中    path,           <br />文件夹路径    match=None,        <br />正则匹配    recursive=False,      <br />递归下面的文件夹    allow_files=True,     <br />允许文件    allow_folders=False,    <br />允许文件夹    required=True,    widget=None,    label=None,    initial=None,    help_text='' |
| GenericIPAddressField                         | protocol='both',      both,ipv4,ipv6支持的IP格式    <br />unpack_ipv4=False     解析ipv4地址，如果是::ffff:192.0.2.1时候，可解析为192.0.2.1， PS：protocol必须为both才能启用 |
| SlugField(CharField)                          | 数字，字母，下划线，减号（连字符）    …                      |
| UUIDField(CharField)                          | uuid类型                                                     |

- **moduleform**

modelform 强大是数据验证，适中的数据操作（有数据库数据写入操作的情况）

示例：

1、编辑views.py文件使用moduleform

<img src="images/image-20240331223435591.png" alt="image-20240331223435591" style="zoom:67%;" />

insrance的使用：

<img src="images/image-20240331223452808.png" alt="image-20240331223452808" style="zoom:67%;" />

使用moduleform提交文件的时候需要注意：
数据库中对应字段：

```bash
models.FileField(upload_to="",max_lenght=32)   #其中upload_to必填字段，上传到哪个目录
```

views.py文件中需要使用request.FIELS来接受文件

<img src="images/image-20240331223535105.png" alt="image-20240331223535105" style="zoom:67%;" />

前端要是传文件，form表单必须要有属性：enctype="multipart/form-data"

<img src="images/image-20240331223552643.png" alt="image-20240331223552643" style="zoom:67%;" />

最终效果

<img src="images/image-20240331223610212.png" alt="image-20240331223610212" style="zoom:67%;" />



# 钩子函数(HOOK)

在特定的节点自动触发，完成相应的操作 

钩子函数在forms组件中就类似于第二道管卡，能够让我们自定义校验规则 

在forms组件中有两类钩子：

​	1.局部钩子 当你需要给某个字段添加校验规则的时候可以使用局部钩子 
​	2.全局钩子 当你需要给多个字段添加校验规则的时候可以使用全局钩子

## 局部钩子

```python
# 关键字：clean_局部字段名 
def clean_username(self): 
    # 数据通过了forms组件条件的数据都会存储在cleaned_data里，所以我们拿出这里面(过了第一道关卡)的数据，进行添加第二道关卡 
    username = self.cleaned_data.get('username') 
    if 'sb' in username: # 如果数据的用户名含有'sb' 
        # 提示前端展示错误信息(关键字：add_error) 
        self.add_error('username','不能骂人呀~~') 
    # 将钩子函数沟取出来的数据再放回去 
    return username
```

## 全局钩子

```python
# 2.校验密码和确认密码两次密码是否一致
# 判断：需要校验password和confirm_password两个字段 (全局钩子)
# 关键字：clean
def clean(self): # 定义全局钩子
    # 拿到通过forms组件的两次密码
    password = self.cleaned_data.get('password')     confirm_password = 		self.cleaned_data.get('confirm_password')     if password != confirm_password: # 如果两次密码不一致
    # 告诉前端展示错误信息
    self.add_error('confirm_password','两次密码不一致')        # 将钩子函数够出来的数据返回
    return self.cleaned_data
```



# cookie & session

## cookie 

工作原理:

```
1.浏览器第一次发送请求到浏览器端；
2.服务器端创建Cookie，该cookie中包含用户的信息，然后将该cookie发送到浏览器端。
3.浏览器端再次访问服务器端时会携带服务器端创建的Cookie。
4.服务器端通过cookie中携带的数据区分不同的用户。
```

## session 

工作原理:

```
1.浏览器端第一次发送请求到服务器端，服务器端创建一个session，同时会创建一个特殊的cookie，然后将cookie发送至浏览器端
2.浏览器端发送第N（N>1）次请求到服务器端，浏览器端访问服务器端的时候会携带cookie对象
3.服务器端会根据cookie的value值去查询Session对象，从而区分不同用户。
```

cookie与session的区别:

```
1)cookie数据存放在客户的浏览器上，session数据放在服务器上
2)cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗,如果主要考虑到安全应当使用session
3)session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，如果主要考虑到减轻服务器性能方面，应当使用COOKIE
4)单个cookie在客户端的限制是4K，就是说一个站点在客户端存放的COOKIE不能4K。
5)将登陆信息等重要信息存放为SESSION;其他信息如果需要保留，可以放在COOKIE中
```



**在django中体现：**

生成的用户的session数据保存在django_session表中

<img src="images/image-20240331224258629.png" alt="image-20240331224258629" style="zoom:67%;" />

注：django session表中的数据条数是取决于浏览器的同一个计算机上同一个浏览器只会有一条数据生效（当session过期的时候可能会出现多条数据对应一个浏览器，但是该现象不会持续很久，内部会自动识别过期的数据清除 你也可以通过代码清除）主要是为了节省服务端数据库资源。

## 装饰器实现登录校验(cookie或者session)

1、加在CBV视图的get或者post方法上：

```python
# 需要导入装饰器模块：
from django.utils.decorators import method_decorator

fromdjango.utils.decorators import method_decorator
class HomeView(View):
  defdispatch(self, request, *args, **kwargs):
    returnsuper(HomeView, self).dispatch(request, *args, **kwargs)

def get(self, request):
  returnrender(request, "home.html")

@method_decorator(check_login)    #装饰器函数：check_login
def post(self, request):
  print("Home View POST method...")
  returnredirect("/index/")

```

2、加载dispatch方法上：

因为CBV中首先执行的就是dispatch方法，所以这么写相当于给get和post方法都加上了登录校验。

```python
fromdjango.utils.decorators import method_decorator

class HomeView(View):
  @method_decorator(check_login)
  defdispatch(self, request, *args, **kwargs):
    returnsuper(HomeView, self).dispatch(request, *args, **kwargs)

def get(self, request):
  returnrender(request, "home.html")
 
def post(self, request):
  print("Home View POST method...")
  returnredirect("/index/")
 
```

3、直接加载类视图上，method_decorrator必须加上那么参数：name：

注意：如果get方法和post方法都需要登录校验的话就写两个装饰器

```python
fromdjango.utils.decorators import method_decorator

@method_decorator(check_login, name="get")
@method_decorator(check_login, name="post")
class HomeView(View):
  defdispatch(self, request, *args, **kwargs):
    returnsuper(HomeView, self).dispatch(request, *args, **kwargs)
    
def get(self, request):
  returnrender(request, "home.html")

def post(self, request):
  print("Home View POST method...")
  returnredirect("/index/")

```



# 中间件

作用：

- 修改请求，即传送到 view 中的 HttpRequest 对象。
- 修改响应，即view 返回的 HttpResponse 对象。

**process_request与 process_response：**

- process_request:

  ```bash
  1、请求来的时候需要经过每一个中间件里面的process_request方法，结果的顺序是按照配置文件中注册的中间件从上往下顺序依次执行；
  2、如果中间中没有定义该方法，那么就直接执行下一个中间件；
  3、如果该方法返回了HttpResponse队形，你么请求将不再继续往后继续执行，而是直接原路返回（校验失败不允许访问...）
  ```

- process_response

  ```
  1、响应走的时候需要经过每一个中间件里面的process_response方法，该方法有两个额外的参数：request，response
  2、该方法必须返回一个HttpResponse对象
  	·默认返回的就是形参：response
  	·你也可以自定义返回自己的
  3、顺序是按照从配置文件中的顺序依次从下往上经过，如果没有定义的话，直接跳过执行下一个中间件
  ```

- 其他

  ```python
  # 路由匹配成功之后执行视图函数之前，会自动执行中间件里面的该放法顺序是按照配置文件中注册的中间件从上往下的顺序依次执行
  process_view
  ```
  
  ```bash
  # 返回的ttpResponse对象有render属性的时候才会触发，顺序按照配置文件中注册了的中间件从下往上依次经过
  process template response
  ```
  
  ```bash
  # 当视图函数中出现异常的情况下触发
  process exception
  ```
  
  

<img src="images/image-20240331230303626.png" alt="image-20240331230303626" style="zoom: 80%;" />

自定义中间件：

自定义中间件类的方法有：`process_request` 和 `process_response`

<img src="images/image-20240331230337194.png" alt="image-20240331230337194" style="zoom:80%;" />



**process_request 方法**

process_request 方法有一个参数 request，这个 request 和视图函数中的 request 是一样的。

process_request 方法的返回值可以是 None 也可以是 HttpResponse 对象。

- 返回值是     None 的话，按正常流程继续走，交给下一个中间件处理。
- 返回值是     HttpResponse 对象，Django     将不执行后续视图函数之前执行的方法以及视图函数，直接以该中间件为起点，倒序执行中间件，且执行的是视图函数之后执行的方法。



**process_response 方法**

​	有两个参数，一个是 request，一个是 response，request 是请求对象，response 是视图函数返回的 HttpResponse 对象，该方法必须要有返回值，且必须是response。

process_response 方法是在视图函数之后执行的。



**示例：process_request**

登录验证（没登录之前禁止访问其他的目录，当程序走到这里的时候就开始对请求进行控制，当然也能使用auth模块的装饰函数进行实现，但是使用中间件的方法比较方便）

1、自定义中间件：

<img src="images/image-20240331230453588.png" alt="image-20240331230453588" style="zoom:80%;" />

实际代码

<img src="images/image-20240331230501019.png" alt="image-20240331230501019" style="zoom:80%;" />

```python
from django.utils.deprecation import MiddlewareMixinfrom django.shortcuts 
import HttpResponse, redirect

class AuthMiddleware(MiddlewareMixin):
  	# 0.排除那些不需要登灵就能访问的页面request.path_info 获取当前用户请求的URL /Login/
		def process_request(self，request):
    if request.path_info in ["/login/", "/image/code/"]:
      	return
		# 1.读取当前访问的用户的session信息，如果能读到，说明已登陆过，就可以继续向后走
    # print(info_dict)
		info_dict = request.session.get("info")  
		if info_dict:
				return
		# 2.没有登录过，重新回到登录页面
		return redirect('/login/')
  
```



# auth模块

Django自带的用户认证模块

**auth_user表:**

在创建项目，执行数据库迁移命令的时候就会自动创建有

基本用处：django在启动之后就可以直接访问admin路由，需要输入用户名和密码，数据参考的就是auth_user表，并且还必须是管理员用户才能进入 
创建超级用户(管理员)：
	1、首先要数据库迁移 makemigrations migrate
	2、创建超级用户命令： python3 manage.py createsuperuser（然后会提示要输入超级用户名称、邮箱以及密码（邮箱可以不用填写，但是用户名跟密码必须要填写））

<img src="images/image-20240331230925042.png" alt="image-20240331230925042" style="zoom:80%;" />

<img src="images/image-20240331230935210.png" alt="image-20240331230935210" style="zoom: 67%;" />



**扩展auyh_user表：**

```python
# auth_user表所在的位置
from django.contrib.auth.models import User 
```

<img src="images/image-20240331231203988.png" alt="image-20240331231203988" style="zoom: 80%;" />

\# 所以在扩展auth_user表是只需要继承AbstractUser即可

举例：

```python
# 需导入模块：
from django.contrib.auth.models import User,AbstractUser
 
class UserInfo(AbstractUser):
  phone = models.BigIntegerField()

'''
# 需要注意：
如果继承了AbstractUser，那么在执行数据库迁移命令的时候auth_user表就不会再创建出来了
  而UserInfo表中会出现auth_user所有的字段外加自己扩展的字段，这么做的好处在于我们能够直接点击我们自己的表更加快捷的完成操作以及扩展
  
  - 前提：
      1、在继承之前没有执行过数据库迁移的命令，即auth_user表没有被创建出来，如果当前库已经创建了那么就需要重新创建一个库
    2、继承的类里面不要覆盖AbstractUser里面的字段名，表里面原有的字段不要动，只扩展额外的字段即可
    3、需要再配置文件中告诉django你要用UserInfo代替auth_user
        AUTH_USER_MODEL = 'app01.UserInfo'  # 应用名.表名
  
  
  如果自己写表替代了auth_user，那么auth模块的功能还是照常使用，参考的表由原来的auth_user变成了UserInfo
	# 方法不变只是操作的表换成了自己指定的表
	# 比如：
		- 原先：
        from django.contrib.auth.models import User
        User.objects.create(username=username,password=password)
  	- 现在:
        from app01 import models
        models.UserInfo.objects.create(username=username,password=password)
	
 '''
```



##  使用auth_user实现功能

## **注册**

```python
# 原理：操作auth_user表写入数据：
# 关键字：User.objects.create(username=username,password=password)

# 需导入模块：
from django.contrib.auth.models import User

def register(request):
  if request.method == 'POST':
  		username = request.POST.get('username')
    	password = request.POST.get('password')
    	# 创建普通用户，使用create_auth，不能用create 密码没有加密处理
    	User.objects.create_user(username=username,password=password)
    	return render(request,'register.html')

# 创建超级用户(了解):使用代码创建超级用户 邮箱是必填的 而用命令创建则可以不填
User.objects.create_superuser(username=username,email='123@qq.com',password=password)
```



## **登录**

```python
# 导入auth模块
from django.contrib import auth

def login(request):
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        # 登录密码认证：使用auth模块获取到加密的密码并与前端输入的进行对比，返回的是一个user对象
        user_obj = auth.authenticate(request,username=username,password=password)
        '''
            他都干了那些事：
                自动查找auth_user表
                自动给密码加密后再进行比对
            注意事项：
                括号内必须同时传入用户名和密码
                不能只传用户名
        '''
        if user_obj:
            # 之前我们保存用户登录状态要通过session来保存，现在可通过auth来直接帮助我们保存到session表
            # 保存用户登录状态，保存之后，用户的数据就会记录到request中
            auth.login(request,user_obj)   #类似于request_session[key] = user_obj
            return redirect('/home/')      #跳转到home页面
	
        print(user_obj,type(res))
        print(user_obj.username)      #如果的用户名或者密码错误返回的是none
        print(user_obj.password)
    return render(request,'login.html')

def home(request):
	#可以拿到当前登录的用户对象，那么如果没有登录，直接访问home页面，requeest.user返回是AnonymousUser匿名用户
        print(request.user)   
        return HttpResponse('home')
```

<img src="images/image-20240331231702597.png" alt="image-20240331231702597" style="zoom: 67%;" />

<img src="images/image-20240331231714068.png" alt="image-20240331231714068" style="zoom:67%;" />



## **注销**

```python
# 注销则是将session表中的用户数据删除即可。
# 关键字：auth.logout(request)

@login_required   # 登录才能注销所以也需要验证是否登录。
def logout(request):
  auth.logout(request)
  return redirect('/login/')
```



## 校验用户是否登录

判断用户是否登录：

```python
# 关键字：
request.user.is_authenticated
# 返回结果：布尔类型：
#    - 未登录(匿名用户)	 ：返回False
#    - 已登录			：返回True
```

使用装饰器

```python
# 需导入内置模块：
from django.contrib.auth.decorators import login_required

'''
# 关键字：添加装饰器
- 局部配置：@login_required(login_url='/指定未登录跳转页面/')
- 全局配置：@login_required   # 无需指定跳转路径
# 需要在settings.py配置文件中配置：LOGIN_URL = '/指定未登录跳转页面/'

局部和全局区别：
全局的好处在于无需重复写代码 但是跳转的页面却很单一
局部的好处在于不同的视图函数在用户没有登陆的情况下可以跳转到不同的页面
'''
```



## **修改密码**

```python
'''
- 需要校验旧密码是否正确
- 需要将新密码从auth_user表中修改并保存

# 校验旧密码：
# 关键字：request.user.check_password('需要校验的数据(旧密码)')
		- 相当于：将当前表的password字段和用户输入的旧密码进行比对

# 修改密码并同步到数据库：
# 关键字：request.user.set_password('新密码')
# 保存：request.user.save()  # 同步到数据库（不提交就不生效）
'''
```

示例：

前端：setpassword.html

```html
<form action="" method="post">
  {% csrf_token %}
  <p>username<input type="text" name="username" disabled value="{{ request.user.username }}"></p>
  <p>old_password:<input type="text" name="old_password"></p>
  <p>new_password:<input type="text" name="new_password"></p>
  <p>confirm_password:<input type="text" name="confirm_password"></p>
  <input type="submit">
</form>
```

后端：views.py

```python
from django.contrib import auth

@login_required       #使用内部装饰器校验登录
def set_password(request):
  if request.method == 'POST':
      old_password = request.POST.get('old_password')
      new_password = request.POST.get('new_password')
      confirm_password = request.POST.get('confirm_password')
      # 校验两次密码是否一致
      if new_password == confirm_password:
          # 校验老密码：
          is_right =  request.user.check_password(old_password)   # 返回的是布尔值
          if is_right:
              # 修改密码
              request.user.set_password(new_password)  # 修改对象的属性
              request.user.save()   # 同步到数据库
       return redirect('/login/')   # 注册完之后跳转到登录页面
  return render(request,'setpassword.html',locals())
```

<img src="images/image-20240331232442437.png" alt="image-20240331232442437" style="zoom:80%;" />



# DRF

 参考： http://www.yuan316.com/post/DRF/

<img src="images/image-20240410234818622.png" alt="image-20240410234818622" style="zoom:67%;" />

RESTful是一种专门为Web 开发而定义API接口的设计风格，尤其适用于前后端分离的应用模式中。这种风格的理念认为后端开发任务就是提供数据的，对外提供的是数据资源的访问接口，所以在定义接口时，客户端访问的URL路径就表示这种要操作的数据资源。而对于数据资源分别使用POST、DELETE、GET、UPDATE等请求动作来表达对数据的增删查改。

一个建立在Django基础之上的Web 应用开发框架，可以快速的开发REST API接口应用。在REST framework中，提供了序列化器Serialzier的定义，可以帮助我们简化序列化与反序列化的过程，不仅如此，还提供丰富的类视图、扩展类、视图集来简化视图的编写工作。REST framework还提供了认证、权限、限流、过滤、分页、接口文档等功能支持。REST framework提供了一个API 的Web可视化界面来方便查看测试接口。

特点：

- 提供了定义序列化器Serializer的方法，可以快速根据     Django ORM 或者其它库自动序列化/反序列化；
- 提供了丰富的类视图、Mixin扩展类，简化视图的编写；
- 丰富的定制层级：函数视图、类视图、视图集合到自动生成     API，满足各种需要；
- 多种身份认证和权限认证方式的支持；[jwt]
- 内置了限流系统；
- 直观的 API web 界面；【方便我们调试开发api接口】
- 可扩展性，插件丰富

## DRF安装

依赖的环境：

- Python (3.5     以上)
- Django (2.2     以上)

前提是已经安装了django

```bash
pip3 install djangorestframework -i https://pypi.douban.com/simple
```

快速体验：

创建好django项目之后，在**settings.py**的**INSTALLED_APPS**中添加 `rest_framework`。

接下来就可以使用DRF提供的功能进行api接口开发了。在项目中如果使用rest_framework框架实现API接口，主要有以下三个步骤：

- 将请求的数据（如JSON格式）转换为模型类对象
- 操作数据库
- 将模型类对象转换为响应的数据（如JSON格式）

## 修改配置

修改配置setting.py配置：

```python
INSTALLED_APPS = [
    ………
    'rest_framework',
] 
```



## drf视图

继承关系图

![image-20240623171221353](images/Django/image-20240623171221353.png)

### 一级视图类:APIView

1、基于CBV的写法，APIView是REST framework提供的所有视图的基类，继承自Django的View父类。
2、APIView与View的不同之处在于：

- 传入到视图方法中的是REST framework的Request对象，而不是Django的HttpRequeset对象；
- 视图方法可以返回REST framework的Response对象，视图会为响应数据设置（render）符合前端期望要求的格式；
- 任何APIException异常都会被捕获到，并且处理成合适格式的响应信息返回给客户端；
- 重新声明了一个新的as_views方法并在dispatch()进行路由分发前，会对请求的客户端进行身份认证、权限检查、流量控制。

3、APIView新增了类属性：

- authentication_classes 列表或元组，身份认证类
- permissoin_classes 列表或元组，权限检查类
- throttle_classes 列表或元祖，流量控制类

4、在APIView中仍以常规的类视图定义方法来实现get() 、post() 或者其他请求方式的方法。

#### 请求对象

REST framework 传入视图的request对象不再是Django默认的Request对象，而是REST framework提供的扩展了Request类的Request类的对象。REST framework 提供了Parser解析器，在接收到请求后会自动根据Content-Type指明的请求数据类型（如JSON、表单等）将请求数据进行parse解析，解析为类字典[QueryDict]对象保存到Request对象中。

**常用属性：**

**request.data**

返回解析之后的请求体数据。类似于Django中标准的`request.POST`和 `request.FILES`属性，但提供如下特性：

- 包含了解析之后的文件和非文件数据
- 包含了对POST、PUT、PATCH请求方式解析后的数据
- 利用了REST framework的parsers解析器，不仅支持表单类型数据，也支持JSON数据

**request.query_params**

与Django标准的`request.GET`相同，只是更换了更正确的名称而已。

**request._request：**

获取django内部的Request对象

**总结**

- GET请求：如果想获取GET请求的所有参数，使用request.query_params即可
- POST请求：使用request.data就可以处理传入的json请求，或者其他格式请求。

**使用示例一：**

```python
#基于原来的View编写
from django.views import View
from django.http.response import HttpResponse
from django.http.request import HttpRequest
from django.core.handlers.wsgi import WSGIRequest

class ReqView(View):
  def get(self,request):
    print(request)
    return HttpResponse("ok")
 
"""
默认情况下, 编写视图类时，如果继承的是django内置的django.view.View视图基类，
则视图方法中得到的request对象，是django默认提供的django.core.handlers.wsgi.WSGIRequest
WSGIRequest这个请求处理对象，无法直接提供的关于json数据数据处理。
在编写api接口时很不方便，所以drf为了简写这块内容，在原来的HttpRequest的基础上面，新增了一个Request对象
这个Request对象是单独声明的和原来django的HttpRequest不是父子关系。
同时注意：
  要使用drf提供的Request请求处理对象，必须在编写视图类时继承drf提供的视图基类
  from rest_framework.views import APIView
  
  如果使用drf提供的视图基类APIView编写类视图，则必须使用来自drf提供的Request请求对象和Response响应对象
"""
 
#基于drf模块的APIView
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
 
class ReqAPIView(APIView):
  def get(self,request):
    # rest_framework.request.Request对象
    print(request) # <rest_framework.request.Request: GET '/req/req2?name=xiaoming&age=17&lve=swim&lve =code'>
    # 获取查询字符串
    print(request.query_params)
    # 没有参数情况下： <QueryDict: {}>
    # 有参数的情况下： <QueryDict: {'name': ['xiaoming'], 'age': ['17'], 'lve': ['swim', 'code']}>
    # 所以，request.query_params的返回值操作和原来在django里面是一模一样的
    print(request.query_params.get("name")) # xiaoming
    print(request.query_params.getlist("lve")) # ['swim', 'code']

    return Response("ok")

  def post(self, request):
    # 获取请求体
    print(request.data) # {'name': 'xiaoming', 'age': 16, 'lve': ['swim', 'code']}
    """直接从请求体中提取数据转
    # 客户端如果上传了json数据，直接返回字典
    {'name': '灰太狼', 'age': 20, 'sex': 1, 'classmate': '301', 'description': '我还会再回来的~'}
    # 客户端如果上传了表单数据，直接返回QueryDict
    <QueryDict: {'name': ['xiaohui'], 'age': ['18']}>
    """
    print(request.FILES) # 获取上传文件列表
    
    # 要获取django原生提供的HttpRequest对象，可以通过request._request来获取到
    print(request._request.META.get("Accept")) # 当值为None时，drf默认在响应数据时按json格式返回
    # response = Response(data="not ok", status=204, headers={"Company":"ANY"})
    response = Response(data="not ok", status=status.HTTP_400_BAD_REQUEST, headers={"Company":"ANY"})
    return response
```

**CBV的使用**

url.py

```python
from django.conf.urls import path
from app01 import views

urlpatterns = [
  path('info/', views.UserView.as_view()),	# CBV
  #path('user/<str:pk>', views.UserView.as_view()),  # CBV，带有参数：pk
  path('info/', views.InfoView.as_view()),	# CBV
  #path('user/<str:dt>', views.InfoView.as_view()),  # CBV，带有参数：dt
]
```

views.py

```python
from django.http import JsonResponse
from django.views import View
from rest_framework.views import APIView


# Django CBV写法：使用django内部的view
# 导入模块：from django.views import Vie
class UserView(View):
    def dispatch(self, request, *args, **kwargs):         # 这里我们也可以重写父类的dispatch方法
        print('dispatch')
        obj=super().dispatch(request,*args,**kwargs)
        return obj

    # 定义get方法
    #def get(self, request, id):    						# 接收id参数
    #def get(self, request, *args, **kwargs):   # 或者直接写成接收不定参数
    def get(self, request):
      	pass

    # 定义post方法
    def post(self, request, *args, **kwargs):
      	pass

    def put(self, request, *args, **kwargs):
      	pass

    def delete(self, request, *args, **kwargs):
      	pass
  
# 使用RESTframework提供的所有视图的基类：APIView
# 导入模块：from rest_framework.views import APIView
Class InfoView(APIView):
    #def get(self, request, dt):    						# 接收dt参数
    #def get(self, request, *args, **kwargs):   # 或者直接写成接收不定参数
    def get(self, request, *args, **kwargs):
      	pass

    def post(self, request, *args, **kwargs):
      	pass

    def put(self, request, *args, **kwargs):
      	pass

    def delete(self, request, *args, **kwargs):
      	pass

```

**示例二：**

```python
from django.http import HttpResponse
from rest_framework.renderers import JSONRenderer
from rest_framework.decorators import api_view
from quickstart.models import BookInfo
from quickstart.serializers import BookInfoSerializer


class JSONResponse(HttpResponse):
    """
    将内容渲染成JSON的HttpResponse
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)

# 使用函数修饰器修改GET和POST请求
@api_view(['GET', 'POST'])
def BookInfoView(request):
    """
    列出所有的book信息，或创建一个新book。
    """
    if request.method == 'GET':
        print(request.query_params)
        books = BookInfo.objects.all()
        serializer = BookInfoSerializer(books, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        # book = JSONParser().parse(request)
        # serializer = BookInfoSerializer(data=book)
        # 使用request.data自动将请求内容数据部分处理
        print(request.data)
        serializer = BookInfoSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)

      
@api_view(['GET', 'PUT', 'DELETE'])
def BookInfoDetailView(request, book_id):
    """
    获取，更新或删除一个指定ID的book。
    """
    try:
        book = BookInfo.objects.get(pk=book_id)
    except BookInfo.DoesNotExist:
        return JSONResponse(status=404)

    if request.method == 'GET':
        serializer = BookInfoSerializer(book)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        # data = JSONParser().parse(request)
        # serializer = BookInfoSerializer(book, data=data)
        # 使用request.data自动将请求内容数据部分处理
        serializer = BookInfoSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        book.delete()
        return JSONResponse(status=204)
```

CBV方式请求

1. 对于CBV来说（是基于**反射**来实现的，根据**请求方式**不同，执行不同的方法）

2. 原理（以下先后顺序）：

   - 根据urls.py中的路由，找到对应的视图类

   - 视图类继承django中的**View**类

   - 执行继承类**View**里面的**as_view**方法，

   - 再执行**as_view**方法里面有一个方法**view，**此时**as_view**方法返回**view**方法

   - **view**方法里面调用了**dispatch方法** （反射执行GET/POST/DELETE....方法）

**django内置as_view()源码解析：**

```python
class View(object):
    """
    Intentionally simple parent class for all views. Only implements
    dispatch-by-method and simple sanity checking.
    """
......

    @classonlymethod
    def as_view(cls, **initkwargs):   # cls代表当前请求的类UserView，类后面加括号[cls(**initkwargs)]代表将这个类实例化，即相当于self = UserView()
        """
        Main entry point for a request-response process.
        """
        for key in initkwargs:
            if key in cls.http_method_names:
                raise TypeError("You tried to pass in the %s method name as a "
                                "keyword argument to %s(). Don't do that."
                                % (key, cls.__name__))
            if not hasattr(cls, key):
                raise TypeError("%s() received an invalid keyword %r. as_view "
                                "only accepts arguments that are already "
                                "attributes of the class." % (cls.__name__, key))

        def view(request, *args, **kwargs):   
          	# 实例化了一个视图类对象，上面例子里是：UserView
            # 传入携带参数的reques对象，包含请求相关的所有信息
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.request = request
            self.args = args
            self.kwargs = kwargs
            # 返回dispatch（此时会先去子类找，没有就到父类）
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        view.view_initkwargs = initkwargs

        # take name and docstring from class
        update_wrapper(view, cls, updated=())

        # and possible attributes set by decorators
        # like csrf_exempt from dispatch
        update_wrapper(view, cls.dispatch, assigned=())
        return view						# 返回view函数
      
......
        def dispatch(self, request, *args, **kwargs):
            # Try to dispatch to the right method; if a method doesn't exist,
            # defer to the error handler. Also defer to the error handler if the
            # request method isn't on the approved list.
            # 获取到请求方式，判断是否在请求的方法内
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
            # 执行方法并返回，就例如UserView定义了get方法，这里相当于是执行get(request, *args, **kwargs)
            return handler(request, *args, **kwargs)
```

**drf 的APIView源码：**

```python
class APIView(View):

    # The following policies may be set at either globally, or per-view.
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
    metadata_class = api_settings.DEFAULT_METADATA_CLASS
    versioning_class = api_settings.DEFAULT_VERSIONING_CLASS

    # Allow dependency injection of other settings to make testing easier.
    settings = api_settings

    schema = DefaultSchema()

    @classmethod
    def as_view(cls, **initkwargs):   # 封装了as_view
        """
        Store the original class on the view function.

        This allows us to discover information about the view when we do URL
        reverse lookups.  Used for breadcrumb generation.
        """
        if isinstance(getattr(cls, 'queryset', None), models.query.QuerySet):
            def force_evaluation():
                raise RuntimeError(
                    'Do not evaluate the `.queryset` attribute directly, '
                    'as the result will be cached and reused between requests. '
                    'Use `.all()` or call `.get_queryset()` instead.'
                )
            cls.queryset._fetch_all = force_evaluation

        view = super().as_view(**initkwargs)        # 返回的还是django内部的view
        view.cls = cls
        view.initkwargs = initkwargs

        # Note: session based authentication is explicitly CSRF validated,
        # all other authentication is CSRF exempt.
        return csrf_exempt(view)
......

    def dispatch(self, request, *args, **kwargs):
        """
        `.dispatch()` is pretty much the same as Django's regular dispatch,
        but with extra hooks for startup, finalize, and exception handling.
        """
      	# 这里也是传入携带参数的reques对象，包含请求相关的所有信息
        self.args = args
        self.kwargs = kwargs
        
        # 与django内部的dispatch方法不同的是这里对request对象进行了封装
        # 类似于request = DrfRquest(request, ...)，这里是Drf内写了DrfRquest方法传入django中的request对象
        # 封装之后，request._request -->才是django中的request对象
        # 此时要使用GET方法，需要这么写：request._request.GET
        request = self.initialize_request(request, *args, **kwargs)
        self.request = request
        self.headers = self.default_response_headers  # deprecate?

        try:
            self.initial(request, *args, **kwargs)

            # Get the appropriate handler method
            if request.method.lower() in self.http_method_names:
              	# 通过反射来获取到请求方法
                handler = getattr(self, request.method.lower(),
                                  self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
						# 执行相应的请求方法
            response = handler(request, *args, **kwargs)

        except Exception as exc:
            response = self.handle_exception(exc)

        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response

```

#### 响应对象

 ```python
rest_framework.response.Response
 ```

REST framework提供了一个响应类Response，使用该类构造响应对象时，响应的具体数据内容会被转换（render渲染器）成符合前端需求的类型。

REST framework提供了Renderer 渲染器，用来根据请求头中的Accept（接收数据类型声明）来自动转换响应数据到对应格式。如果前端请求中未进行Accept声明，则会采用Content-Type方式处理响应数据，我们可以通过配置来修改默认响应格式。

 drf的响应处理类和请求处理类不一样，Response就是django的HttpResponse响应处理类的子类。

```python
Response(data, status=None, template_name=None, headers=None, content_type=None)
'''
- data: 为响应准备的序列化处理后的数据；
- status: 状态码，默认200；
- template_name: 模板名称，如果使用HTMLRenderer 时需指明；
- headers: 用于存放响应头信息的字典；
- content_type:     响应数据的Content-Type，通常此参数无需传递，REST framework会根据前端所需类型数据来设置该参数
'''

from rest_framework.response import Response
from rest_framework import status
```

**状态码**

为了方便设置状态码，REST framewrok在`rest_framework.status`模块中提供了常用状态码常量。

不再需要JSONResponse类，所有响应通过response即可，使用命名状态代码，使得响应意义更加明显

### 二级视图类：GenericAPIView

通用视图类主要作用就是把视图中的独特的代码抽取出来，让视图方法中的代码更加通用，方便把通用代码进行简写。

继承自`APIView`，主要增加了操作序列化器和数据库查询的方法，作用是为下面Mixin扩展类的执行提供方法支持。通常在使用时，可搭配一个或多个Mixin扩展类。

提供的关于序列化器使用的属性与方法：

**（1）get_serializer_class(self)**

​	当出现一个视图类中调用多个序列化器时,那么可以通过条件判断在get_serializer_class方法中通过返回不同的序列化器类名就可以让视图方法执行不同的序列化器对象了。

​	返回序列化器类，默认返回serializer_class，可以重写。

**（2）get_serializer(self, `*args`, `**kwargs`)**

​	返回序列化器对象，主要用来提供给Mixin扩展类使用，如果我们在视图中想要获取序列化器对象，也可以直接调用此方法。 

​	**注意：**该方法在提供序列化器对象的时候，会向序列化器对象的context属性补充三个数据：request、format、view，这三个数据对象可以在定义序列化器时使用。

​		view 当前请求的类视图对象

​		request 当前视图的请求对象

​		format 当前请求期望返回的数据格式

**（3）get_queryset(self)**

​	返回视图使用的查询集，主要用来提供给Mixin扩展类使用，是列表视图与详情视图获取数据的基础，默认返回queryset属性，可以重写，例如：

 ```python
 def get_queryset(self):
   user = self.request.user
   return user.accounts.all()
 ```

**（4）get_object(self)**

​	返回详情视图所需的模型类数据对象，主要用来提供给Mixin扩展类使用。

 		在试图中可以调用该方法获取详情信息的模型类对象。

​	若详情访问的模型类对象不存在，会返回404。

 		该方法会默认使用APIView提供的check_object_permissions方法检查当前对象是否有权限被访问。

```python
# url(r'^books/(?P<pk>\d+)/$', views.BookDetailView.as_view()),
class BookDetailView(GenericAPIView):
  queryset = BookInfo.objects.all()
  serializer_class = BookInfoSerializer
 
  def get(self, request, pk):
    book = self.get_object() # get_object()方法根据pk参数查找queryset中的数据对象
    serializer = self.get_serializer(book)
    return Response(serializer.data)
```

 ```python
 from rest_framework import status
 from quickstart.models import BookInfo
 from quickstart.serializers import BookInfoSerializer
 from rest_framework.response import Response
 from rest_framework import generics
 
 
 class BookInfoView(generics.GenericAPIView):
     """
     列出所有的book信息，或创建一个新book。
     """
     # 通用的属性(查询集，序列化器)
     # 原来一个类只能对一个对象和序列化器进行操作，改写完成后根据需求，填写不同的对象和序列化器即可
     queryset = BookInfo.objects.all()
     serializer_class = BookInfoSerializer
 
     def get(self, request):
         # books = self.queryset
         books = self.get_queryset()
         # serializer = self.serializer_class(books, many=True)
         # serializer = self.get_serializer_class()(books, many=True)
         serializer = self.get_serializer(books, many=True)
         return Response(serializer.data)
 
     def post(self, request):
         serializer = self.get_serializer(data=request.data)
         if serializer.is_valid():
             serializer.save()
             return Response(serializer.data, status=status.HTTP_201_CREATED)
         return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
 
 
 class BookInfoDetailView(generics.GenericAPIView):
     """
     获取，更新或删除一个指定ID的book。
     """
     # 通用属性
     queryset = BookInfo.objects.all()
     serializer_class = BookInfoSerializer
     # 默认传入id名称为pk，可以自定义其他名称
     # lookup_url_kwarg = "book_id"
 
     def get(self, request, pk):
         book = self.get_object()  # 根据book_id到queryset中取出指定对象,传入id名称必须为pk
         serializer = self.get_serializer(book)
         return Response(serializer.data)
 
     def put(self, request, pk):
         book = self.get_object()
         serializer = self.get_serializer(book, data=request.data, partial=True)
         if serializer.is_valid():
             serializer.save()
             return Response(serializer.data)
         return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
 
     def delete(self, request, pk):
         book = self.get_object()
         book.delete()
         return Response(status=status.HTTP_204_NO_CONTENT)
 ```



### 混合视图(Mixins)

提供了几种后端视图（对数据资源进行增删改查）处理流程的实现，如果需要编写的视图属于这五种，则视图可以通过继承相应的扩展类来复用代码，减少自己编写的代码量。

**1、ListModeMixin**
	列表视图扩展类，提供list(request, `*args`, `**kwargs`)方法快速实现列表视图，返回200状态码。
	该Mixin的list方法会对数据进行过滤和分页。

**2、CreatModeMixin**
	创建视图扩展类，提供create(request, `*args`, `**kwargs`)方法快速实现创建资源的视图，成功返回201状态码。
	如果序列化器对前端发送的数据验证失败，返回400错误。

**3、RetrieveModelMixin**
	详情视图扩展类，提供retrieve(request, *args, **kwargs)方法，可以快速实现返回一个存在的数据对象。
	如果存在，返回200， 否则返回404。

**4、UpdateModelMixin**
	更新视图扩展类，提供update(request, `*args`, `**kwargs`)方法，可以快速实现更新一个存在的数据对象。
	同时也提供partial_update(request, `*args`, `**kwargs`)方法，可以实现局部更新。
	成功返回200，序列化器校验数据失败时，返回400错误。

**5、DestroyModelMixin**
	删除视图扩展类，提供destroy(request, `*args`, `**kwargs`)方法，可以快速实现删除一个存在的数据对象。
	成功返回204，不存在返回404。

```python 
from quickstart.models import BookInfo
from quickstart.serializers import BookInfoSerializer
from rest_framework import generics
from rest_framework import mixins


class BookInfoView(mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView):
    """
    列出所有的book信息，或创建一个新book。
    """
    queryset = BookInfo.objects.all()
    serializer_class = BookInfoSerializer

    def get(self, request):
        return self.list(request)

    def post(self, request):
        return self.create(request)


class BookInfoDetailView(mixins.RetrieveModelMixin, mixins.UpdateModelMixin, mixins.DestroyModelMixin,
                         generics.GenericAPIView):
    """
    获取，更新或删除一个指定ID的book。
    """
    queryset = BookInfo.objects.all()
    serializer_class = BookInfoSerializer

    def get(self, request, pk):
        return self.retrieve(request)

    def put(self, request, pk):
        return self.update(request)

    def delete(self, request, pk):
        return self.destroy(request)
```

### 三级视图：generics

| 类名称                       | 父类                                                         | 提供方法                              | 作用           |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------- | -------------- |
| CreateAPIView                | GenericAPIView<br />CreateModelMixin                         | post                                  | 创建单个对象   |
| ListAPIView                  | GenericAPIView<br />ListModelMixin                           | get                                   | 查询所有的数据 |
| RetrieveAPIView              | GenericAPIView<br />RetrieveModelMixin                       | get                                   | 获取单个对象   |
| DestroyAPIView               | GenericAPIView<br />DestroyModelMixin                        | delete                                | 删除单个对象   |
| UpdateAPIView                | GenericAPIView<br />UpdateModelMixin                         | put                                   | 更新单个对象   |
| ListCreateAPIView            | GenericAPIView<br />ListModelMixin<br />CreateModelMixin     | get、post，内部调用了list和create方法 |                |
| RetrieveUpdateAPIView        | GenericAPIView<br />RetrieveModelMixin<br />UpdateModelMixin | get、put、patch                       |                |
| RetrieveDestoryAPIView       | GenericAPIView<br />RetrieveModelMixin<br />DestoryModelMixin | get、delete                           |                |
| RetrieveUpdateDestoryAPIView | GenericAPIView<br />RetrieveModelMixin<br />UpdateModelMixin<br />DestoryModelMixin | get、put、patch、delete               |                |

**ListAPIView自定义序列化与自定义分页示例：**

1. 创建模型：model

```python
# models.py
from django.db import models

class Test(models.Model):
    name = models.CharField(max_length=100)
    version = models.CharField(max_length=100)
    detail = models.PositiveIntegerField()
    package = models.ForeignKey(u"Package", verbose_name=u"xxxx", on_delete=models.CASCADE)
```

2. 创建序列化器

```python
# serializers.py
from rest_framework import serializers
from .models import Test

class TestSerializer(serializers.ModelSerializer):
    class Meta:
        model = Test
        fields = '__all__'    # 可以自定义限制返回的字段

# 自定义返回字段
class TestSerializer(serializers.ModelSerializer):
    project_name = SerializerMethodField()
    
    class Meta:
        model = Test
        fields = '__all__'
    
    # 视图中返回的queryset
		def get_project_name(obj):
      	return obj.package.name
    
```

3. 创建一个分页类

```python
# pagination.py

from rest_framework.pagination import PageNumberPagination
class PagePagination(PageNumberPagination):
    page_size = 15
    max_page_size = 500
    page_size_query_param = 'size'
    page_query_param = 'page'
    
    # 可根据需要重写get_paginated_response方法，自定义返回的字段信息
    def get_paginated_response(self, data):
    if self.request.query_params.get('size'):
        self.page_size = int(self.request.query_params.get('size'))
    page_parse = divmod(self.page.paginator.count, int(self.request.query_params.get("size", 0)) or self.page_size)
    if page_parse[1] > 0:
        total = page_parse[0] + 1
    else:
        total = page_parse[0]
    return Response(collections.OrderedDict([
        ('count', self.page.paginator.count),
        ('size', self.page_size),
        ('current', self.page.number),
        ('next', self.get_next_link()),
        ('previous', self.get_previous_link()),
        ('total', total),
        ('total_size', total),
        ('results', data)
    ]))
```

3. 视图

```python
# views.py
from rest_framework import generics
from .models import Test
from .serializers import CarSerializer

class CarListView(generics.ListAPIView):
    queryset = Test.objects.all()
    serializer_class = PagePagination     # 自定义的分页类
    pagination_class = SmallPagesPagination  # 自定义分页类
		
    # 添加额外的上下文数据
    def get_serializer_context(self):
    context = super().get_serializer_context()
    # 添加额外的上下文信息
    context['extra_data'] = MyModel.objects.extra_data_method()
    return context
    
    # 如果需要进一步筛选数据，可以重写 get_queryset 方法
    # def get_queryset(self):
    #     return Car.objects.filter(name='zhangsan')

```







### 视图集

| 类名称               | 父类                                                       | 作用                                             |
| -------------------- | ---------------------------------------------------------- | ------------------------------------------------ |
| ViewSet              | APIView<br />ViewSetMixin                                  | 可以做路由映射                                   |
| GenericViewSet       | GenericAPIView<br />ViewSetMixin                           | 可以做路由映射,可以使用二级视图三个属性,三个方法 |
| ModelViewSet         | GenericAPIView <br />5个mixin类                            |                                                  |
| ReadOnlyModelViewSet | GenericAPIView<br />RetrieveModelMixin<br />ListModelMixin |                                                  |

#### ViewSet

继承自`APIView`与`ViewSetMixin`，作用也与APIView基本类似，提供了身份认证、权限校验、流量管理等。

ViewSet主要通过继承ViewSetMixin来实现在调用as_view()时传入字典{“http请求”：“视图方法”}的映射处理工作，如{‘get’:’list’}，

在ViewSet中，没有提供任何动作action方法，需要我们自己实现action方法。

使用视图集ViewSet，可以将一系列视图相关的代码逻辑和相关的http请求动作封装到一个类中：

- list() 提供一组数据
- retrieve()     提供单个数据
- create() 创建数据
- update() 保存数据
- destory()     删除数据

url.py

```python
from django.urls import path
from quickstart import views

urlpatterns = [
    path('books/', views.BookInfoViewSet.as_view({'get': 'list'})),
    path('books/<int:pk>', views.BookInfoViewSet.as_view({'get': 'retrieve'}))
]
```

view.py

 ```python
 from rest_framework.generics import get_object_or_404
 from rest_framework.response import Response
 from quickstart.models import BookInfo
 from quickstart.serializers import BookInfoSerializer
 from rest_framework import viewsets
 class BookInfoViewSet(viewsets.ViewSet):
     """
     获取所有图书和单个图书信息
     """
     def list(self, request):
         queryset = BookInfo.objects.all()
         serializer = BookInfoSerializer(queryset, many=True)
         return Response(serializer.data)
     def retrieve(self, request, pk=None):
         queryset = BookInfo.objects.all()
         book = get_object_or_404(queryset, pk=pk)
         serializer = BookInfoSerializer(book)
         return Response(serializer.data)
 ```

#### GenericViewSet

继承自GenericAPIView和ViewSetMixin，作用让视图集的视图代码变得更加通用，抽离独特代码作为视图类的属性。

使用ViewSet通常并不方便，因为list、retrieve、create、update、destory等方法都需要自己编写，而这些方法与前面讲过的Mixin扩展类提供的方法同名，所以我们可以通过继承Mixin扩展类来复用这些方法而无需自己编写。但是Mixin扩展类依赖与GenericAPIView，所以还需要继承GenericAPIView。

GenericViewSet就帮助我们完成了这样的继承工作，继承自GenericAPIView与ViewSetMixin，在实现了调用as_view()时传入字典（如{'get':'list'}）的映射处理工作的同时，还提供了GenericAPIView提供的基础方法，可以直接搭配Mixin扩展类使用。

#### ModuleViewSet

ModelViewSet继承自GenericViewSet，同时包括了ListModelMixin、RetrieveModelMixin、CreateModelMixin、UpdateModelMixin、DestoryModelMixin。

ReadOnlyModelViewSet承自GenericViewSet，同时包括了ListModelMixin、RetrieveModelMixin。

url.py

```python
from django.urls import path
from rest_framework import routers

app_name = "public"
router = routers.DefaultRouter()
# 示例数据接口
router.register('bookInfo', views.BookInfoModelViewSet, 'bookInfo')
urlpatterns += router.urls
```

view.py

```python
from quickstart.models import BookInfo
from quickstart.serializers import BookInfoSerializer
from rest_framework import viewsets

class BookInfoModelViewSet(viewsets.ModelViewSet):
    """
    获取所有图书和单个图书信息的增删改查
    """
    queryset = BookInfo.objects.all()
    serializer_class = BookInfoSerializer
```

 #### ReadOnlyModelViewSet

url.py

```python
from django.urls import path
from quickstart import views

urlpatterns = [
    path('books/', views.BookInfoReadOnlyModelViewSet.as_view({'get': 'list'})),
    path('books/<int:pk>/', views.BookInfoReadOnlyModelViewSet.as_view({'get': 'retrieve'}))
]
```

view.py

```python
from quickstart.models import BookInfo
from quickstart.serializers import BookInfoSerializer
from rest_framework import viewsets


class BookInfoReadOnlyModelViewSet(viewsets.ReadOnlyModelViewSet):
    """
    获取所有图书和单个图书信息
    """
    queryset = BookInfo.objects.all()
    serializer_class = BookInfoSerializer
```

#### 重写增删改查方法

- 重写queryset查询方法

```python
class HistoryReadOnlyModelViewSet(viewsets.ReadOnlyModelViewSet):
    """
    获取操作记录
    """
    serializer_class = OperateHistorySerializer

    # 重写queryset方法
    def get_queryset(self):
        # 获取查询参数
        operate = self.request.query_params.get('operate')
        goods_id = self.request.query_params.get('goods_id')
        user_id = self.request.query_params.get('user_id')
        position_id = self.request.query_params.get('position_id')
        start_time = self.request.query_params.get('start_time')
        end_time = self.request.query_params.get('end_time')
        q1 = Q()
        q1.connector = 'AND'
        if operate:
            q1.children.append(('operate', operate))
        if goods_id:
            q1.children.append(('goods_id', goods_id))
        if user_id:
            q1.children.append(('user_id', user_id))
        if position_id:
            q1.children.append(('goods__position', position_id))
        if start_time and end_time:
            q1.children.append(('time__range', [start_time, end_time]))
        return OperateHistory.objects.filter(q1).order_by('-time')
```

- 重写get方法

```python
class ArticleModelViewSet(viewsets.ModelViewSet):
    """
    博客文章增删改查
    """
    queryset = Article.objects.all()
    pagination_class = MyPageNumber
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
    ordering_fields = ['is_recommend', 'created_time', 'view', 'comment', 'like', 'collect']
    filter_fields = ['category', 'tags']

    # 重写get文章方法（阅读量+1,更新留言数）
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        instance.comment = ArticleComment.objects.filter(article=instance.id).count()
        instance.view = instance.view + 1
        instance.save()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
```

- 重写序列化器方法

```python
class ArticleModelViewSet(viewsets.ModelViewSet):
    """
    博客文章增删改查
    """
    queryset = Article.objects.all()
    pagination_class = MyPageNumber
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
    ordering_fields = ['is_recommend', 'created_time', 'view', 'comment', 'like', 'collect']
    filter_fields = ['category', 'tags']

    # 重写序列化器方法(查询全部和查询指定id返回不同的字段)
    def get_serializer_class(self):
        if self.action == 'list':
            return ArticleListSerializer
        else:
            return ArticleRetrieveSerializer
```

- 重写update方法

```python
class LeaveMessageModelViewSet(viewsets.ModelViewSet):
    """
    留言增删改查
    """
    serializer_class = LeaveMessageSerializer

    # 重写更新方法
    def perform_update(self, serializer):
        serializer.save()
        name = serializer.data['name']

    # 重写更新的response
    def update(self, request, *args, **kwargs):
        try:
            instance = self.get_object()
            instance.like = instance.like + 1
            instance.save()
            return Response({'msg': '点赞成功'}, status=status.HTTP_200_OK)
        except Exception as e:
            print(e)
            return Response({'msg': '点赞失败'}, status=status.HTTP_400_BAD_REQUEST)
```

- 重写delete方法

```python
class CmdbModelViewSet(viewsets.ModelViewSet):
    """
    资产信息的增删改查
    """
    serializer_class = CmdbSerializer


    # 重写删除方法
    def perform_destroy(self, instance):
        instance.is_delete = True
        instance.save()

    # 也可以重写删除的response
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return Response(status=status.HTTP_204_NO_CONTENT)
```

-  重写create方法

```python
from django.db import transaction
class GoodsModelVIewSet(viewsets.ModelViewSet):
    """
    资产信息增删改查
    """
    queryset = Goods.objects.all()
    serializer_class = GoodsSerializer

    # 重写create方法
    @transaction.atomic  # 开启事务
    def perform_create(self, serializer):
        # 设置保存点
        sid = transaction.savepoint()
        try:
            Goods.objects.create(**serializer.data)
            OperateHistory.objects.create(operate=1, goods_id=goodsObj.id, user_id=serializer.data.get('user_id'),
                                          number=serializer.data.get('number'))
        except Exception as e:
            print(e)
            transaction.savepoint_rollback(sid)  # 回滚
        transaction.savepoint_commit(sid)  # 提交
        
	# 重写create和response
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

- 重写patch方法

```python
class CmdbModelViewSet(viewsets.ModelViewSet):
    """
    资产信息的增删改查
    """
    serializer_class = CmdbSerializer


    # 重写patch方法
    def perform_update(self, serializer):
        serializer.save()
        name = serializer.data['name']

    # 也可以重写patch的response
    def update(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)
```

## 认证组件

### 认证配置

#### 局部配置

使用步骤：

1. 先定义一个**Authtication**认证类
2. 再在需要认证的试图类中使用：

```python
from rest_framework import exceptions
from rest_framework.authentication import BaseAuthentication
from api import models

class TokenAuthenticate(BaseAuthentication):

    def authenticate(self,request):
        token = request.GET.get('token')
        token_obj = models.UserToken.objects.filter(token=token).first()
        if not token_obj:
            raise exceptions.AuthenticationFailed('用户认证失败')
        # 在rest framework内部，会将整个两个字段赋值给request，以供后续操作使用
        return (token_obj.user,token_obj)

    def authenticate_header(self, request):
        pass
```

**注意：**

- **每个认证类必须实现这两个方法**：`authenticate`和`authenticate_header`，只有authenticate方法自己写，authenticate_header方法直接照抄，不用具体实现
- **为了规范，每个认证类最好继承BaseAuthentication**

- `def authenticate(self,request):`自定义认证操作
- `def authenticate_header(self, request):`认证失败时给浏览器返回的响应头

在其他视图中进行使用：

```python
# 在视图类OrderView中使用
class OrderView(APIView):
    """订单相关业务"""

    authentication_classes = [TokenAuthenticate,]  # 使用上面定义的认证类

    def get(self,request,*args,**kwargs):

        # request.user
        # request.auth
        ret = {'code':1000,'msg':None}
        try:
            ret['data'] = ORDER_DICT
        except Exception as e:
            pass
        return JsonResponse(ret)
```

#### 全局配置

**注意：全局配置的认证类不能直接写在view.py中，需要另外创建目录**

在当前应用下新建一个utils目录，然后新建一个auth.py文件，将认证类写入**auth.py**文件中

```python
from rest_framework import exceptions
from rest_framework.authentication import BaseAuthentication
from api import models

class TokenAuthenticate(BaseAuthentication):

    def authenticate(self,request):
        token = request.GET.get('token')
        token_obj = models.UserToken.objects.filter(token=token).first()
        if not token_obj:
            raise exceptions.AuthenticationFailed('用户认证失败')
        # 在rest framework内部，会将整个两个字段赋值给request，以供后续操作使用
        return (token_obj.user,token_obj)

    def authenticate_header(self, request):
        pass
```

- 然后在配置文件中配置全局默认的认证方案（常用）

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',   			# 基本认证
        'rest_framework.authentication.SessionAuthentication',  		# session认证
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',  # POST请求的Token验证
      	'api.utils.auth.TokenAuthenticate',         # 自定义的认认证类                        
    )
}
```

也可以在每个视图中通过设置authentication_classess属性来设置

```python
from rest_framework.authentication import SessionAuthentication, BasicAuthentication
from rest_framework.views import APIView

class ExampleView(APIView):
    authentication_classes = (SessionAuthentication, BasicAuthentication)
    ...
```

- 当有视图不需要认证就可以直接使用时，使用如下配置实现：

```python
class AuthView(APIView):
    """用户登录认证"""
    # 这里直接配置为空，就可以不用通过认证类
    authentication_classes = []

    def post(self,request,*args,**kwargs):
        ret = {'code':1000,'msg':None}
        try:
            user = request.POST.get('username')
            pwd = request.POST.get('password')
            obj = models.UserInfo.objects.filter(username=user,password=pwd).first()
            # 没有该用户
            if not obj:
                ret['code'] = 1001
                ret['msg'] = '用户名或密码错误'

            token = md5(user)
            models.UserToken.objects.update_or_create(user=obj,defaults={'token':token})
            ret['data'] = token
        except Exception as e:
            ret['code'] = 1002
            ret['msg'] = '请求异常'
        return JsonResponse(ret)
```

**认证失败会有两种可能的返回值：**

- 401 Unauthorized 未认证

- 403 Permission Denied 权限被禁止

在apache的httpd.conf中加入一行配置，传递uwsgi的认证信息

```python
WSGIPassAuthorization On
```

### 认证组件加载源码

认证总体流程：

```shell
1. 当请求进来，运行了APIView类的as_view()方法，进入了APIView类的dispatch方法
2. 执行self.initialize_request这个方法，封装request和认证对象列表等其他参数
3. 执行self.initial方法中的self.perform_authentication，里面运行了user方法
4. 再执行了user方法里面的self._authenticate()方法
5. 然后执行了自己定义的类中的authenticate方法，自己定义的类继承了BaseAuthentication类，里面有  authenticate方法，如果自己定义的类中没有authenticate方法会报错
6. 把从authenticate方法得到的user和auth赋值给user和auth方法
7. 这两个方法把user和auth的值分别赋值给request.user：是登录用户的对象；request.auth：是认证的信息字典
8. 再执行自定义类中get/post方法
```

- 第一步：调用**initialize_request**方法，对原生(django内)的request进行加工，将返回值赋值给request

  `self.request = request`：request已经不是原来django dispatch中的request，而是django rest framework加工后的request

```python
# APIView源码中的dispatch函数:
  	def dispatch(self, request, *args, **kwargs):

        self.args = args
        self.kwargs = kwargs
        request = self.initialize_request(request, *args, **kwargs)
        self.request = request
        self.headers = self.default_response_headers  # deprecate?

        try:
            self.initial(request, *args, **kwargs)
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(),
                                  self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
            response = handler(request, *args, **kwargs)
        except Exception as exc:
            response = self.handle_exception(exc)
        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response

```

```python
# 对比django 中dispatch方法
    def dispatch(self, request, *args, **kwargs):
        # Try to dispatch to the right method; if a method doesn't exist,
        # defer to the error handler. Also defer to the error handler if the
        # request method isn't on the approved list.
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
```

initialize_request方法源码：

```python
# 在上面的APIView源码中的dispatch函数中调用的重写django内部的request方法：`self.initialize_request(request, *args, **kwargs)`
......
    def initialize_request(self, request, *args, **kwargs):
        """
        Returns the initial request object.
        """
        parser_context = self.get_parser_context(request)
				
        # Request是一个类，类后面加括号，是一个对象，这个对象里面封装了多个属性
        # 其中有一个属性authenticators，self.get_authenticators()：首先会从当前类UserInfoView中寻找get_authenticators（）对象，如果没有，再从父类APIView中寻找
        return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),	 # [对象, 对象, ...]
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )   
......

# 调用的父类中APIView中的get_authenticators方法
    def get_authenticators(self):
        """
        Instantiates and returns the list of authenticators that this view can use.
        """
        # auth() for auth in self.authentication_classes：列表生成式实例化对象
      	# 返回的数据类型为：[认证的类对象, 认证的类对象, 认证的类对象, ...]
        # 其中会从当前类中寻找authentication_classes属性，如果没有，从父类APIView中寻找
        return [auth() for auth in self.authentication_classes]

```

```python
# 父类APIView中的authentication_classes属性
authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
```

加载认证组件，本质就是实例化每个认证类的对象，并封装到request对象。

- 第二步：执行dispatch方法中的try中的initial方法（先从当前类中寻找，没有的话再从父类中寻找）

`initial`方法对应的源码：

传入的request是封装后的的request

```python
    def initial(self, request, *args, **kwargs):
        """
        Runs anything that needs to occur prior to calling the method handler.
        """
        self.format_kwarg = self.get_format_suffix(**kwargs)

        # Perform content negotiation and store the accepted info on the request
        neg = self.perform_content_negotiation(request)
        request.accepted_renderer, request.accepted_media_type = neg

        # Determine the API version, if versioning is in use.
        version, scheme = self.determine_version(request, *args, **kwargs)
        request.version, request.versioning_scheme = version, scheme

        # Ensure that the incoming request is permitted
        self.perform_authentication(request)
        self.check_permissions(request)
        self.check_throttles(request)

        
   	# 调用perform_authentication方法，实现认证
		def perform_authentication(self, request):
        """
        Perform authentication on the incoming request.

        Note that if you override this and simply 'pass', then authentication
        will instead be performed lazily, the first time either
        `request.user` or `request.auth` is accessed.
        """
        request.user    # 这里的user是一个类方法在drf 的Request封装类中
```

调用封装后的request中的**user**方法（因为user方法有**@property**，所以调用时不用加括号，即不用这样写：request.user( )）：

```python
#drf 的Request方法
class Request:
    def __init__(self, request, parsers=None, authenticators=None,
                 negotiator=None, parser_context=None):
        assert isinstance(request, HttpRequest), (
            'The `request` argument must be an instance of '
            '`django.http.HttpRequest`, not `{}.{}`.'
            .format(request.__class__.__module__, request.__class__.__name__)
        )
......
        
    @property
    def user(self):
        """
        Returns the user associated with the current request, as authenticated
        by the authentication classes provided to the request.
        """
        if not hasattr(self, '_user'):
            with wrap_attributeerrors():
                self._authenticate()
        return self._user
      
......
    # 看其中的_authenticate方法
    def _authenticate(self):
        """
        Attempt to authenticate the request using each authentication instance
        in turn.
        """
        # 获取每个认证组件的对象，执行authenticator方法
        for authenticator in self.authenticators:
            try:
                user_auth_tuple = authenticator.authenticate(self)
            except exceptions.APIException:
                self._not_authenticated()
                raise

            if user_auth_tuple is not None:								# 如果获取到数据，就在这里执行
                self._authenticator = authenticator
                self.user, self.auth = user_auth_tuple    # 这里进行赋值
                return

        self._not_authenticated()
```

`authenticator.authenticate(self)`：执行认证类的**authenticate**方法

- 如果authenticate方法抛出异常，self._not_authenticated()执行
- 如果有返回值，必须是元组，`self.user, self.auth = user_auth_tuple：`返回一个元组，第一个值给user，第二个值给auth
- 没报错，返回None，表示当前认证类不处理，交给下一个认证类进行处理

### 权限控制源码

权限控制可以限制用户对于视图的访问和对于具体数据对象的访问。

- 在执行视图的dispatch()方法前，会先进行视图访问权限的判断
- 在通过get_object()获取具体对象时，会进行对象访问权限的判断

流程：

1. 请求进来，进入了APIView类的dispatch方法
2. 执行self.initialize_request这个方法，封装request和认证对象列表等其他参数
3. 先执行self.initial方法中的`self.perform_authentication(request)`进行认证
4. 再执行self.initial方法中的`self.check_permissions`进行权限认证

```python
def check_permissions(self, request):
  """
        Check if the request should be permitted.
        Raises an appropriate exception if the request is not permitted.
        """
  for permission in self.get_permissions():
    if not permission.has_permission(request, self):
      self.permission_denied(
        request,
        message=getattr(permission, 'message', None),
        code=getattr(permission, 'code', None)
      )
```

从当前类中执行**get_permissions**方法，如果没有，从父类**ApiView**中寻找，父类中如下：

```python
    def get_permissions(self):
        """
        Instantiates and returns the list of permissions that this view requires.
        """
        return [permission() for permission in self.permission_classes]
```

- `[permission() for permission in self.permission_classes]：`列表生成式
- `permission_classes`来自`permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES`
  - 如果当前类中没有设置，默认从配置文件中寻找，如果设置了，从当前类中拿到

- `self.permission_classes`：拿到权限类，`permission() `：将权限类实例化-->变成对象
- ` return [permission() for permission in self.permission_classes]`：返回的是权限的对象列表

如果权限类中没有**has_permission**方法就抛出异常

- `permission.has_permission(request, self)`：如果返回True，not True就是false，就不执行if里面的代码，表示权限通过；如果返回false,not false就是true，就执行if里面的代码，`self.permission_denied`方法抛出异常，权限认证失败

```python
def permission_denied(self, request, message=None, code=None):
    """
        If request is not permitted, determine what kind of exception to raise.
        """
    if request.authenticators and not request.successful_authenticator:
        raise exceptions.NotAuthenticated()
        raise exceptions.PermissionDenied(detail=message, code=code)
```

### 控制访问频率过程源码分析

1. 请求进来，进入了APIView类的dispatch方法
2. 执行self.initialize_request这个方法，封装request和认证对象列表等其他参数
3. 先执行self.initial方法中的`self.perform_authentication(request)`进行认证
4. 执行self.initial方法中的`self.check_permissions`进行权限认证
5. 执行self.initial方法中的`self.check_throttles`进行节流

```python
def check_throttles(self, request):
    """
        Check if request should be throttled.
        Raises an appropriate exception if the request is throttled.
        """
    throttle_durations = []
    for throttle in self.get_throttles():
        if not throttle.allow_request(request, self):
            throttle_durations.append(throttle.wait())

            if throttle_durations:
                # Filter out `None` values which may happen in case of config / rate
                # changes, see #1438
                durations = [
                    duration for duration in throttle_durations
                    if duration is not None
                ]

                duration = max(durations, default=None)
                self.throttled(request, duration)
```

执行**get_throttles**方法，获取访问控制类的实例化对象

```python
    def get_throttles(self):
        """
        Instantiates and returns the list of throttles that this view uses.
        """
        return [throttle() for throttle in self.throttle_classes]
```

执行allow_request方法，如果该方法返回false，那么 not false为true，那么执行check_throttles方法里面的if语句，抛出异常

```python
    def allow_request(self, request, view):
        """
        Return `True` if the request should be allowed, `False` otherwise.
        """
        raise NotImplementedError('.allow_request() must be overridden')
```



## 序列化器

序列化类的主要功能：

1.   序列化：跟form和formset类似，作为客户和服务器传递数据的格式载体，处理客户端请求时，对数据进行反序列化解析为模型实例，，做校验，然后把cleaned_data交给视图来处理，以便在数据库中创建或更新记录。
2. 反序列化：处理服务器响应时，serializer把视图从数据库取到的模型实例，反序列化为json数据序列从而便于传输给客户端

模拟post发请求，可以这么发：

1. 把数据放在body里面，以form_data的格式发送键值对
2. 或者在body里面发raw json，同时在header里面加上`content-type :  application/json` 

### Serializer

基本序列化器，相当于  Form 

```text
基本用法也是类似的，只有一下几点区别:
 1. extra_kwargs 是定义 read_only, write_only的地方，这两个参数控制是否需要用户提供信息，是否需要展示信息  
 2. 一般自定义的字段 level_text = serializers.CharField(),  如果与model中字段同名，则覆盖，source参数可以从数据库中获取对象应赋值到对象，有两种情况：
   如果可执行，自动执行 
      level_text = serializers.CharField(source="get_level_display") 
   如果不可执行，则直接获取属性 
	 		depart = serializers.CharField(source="depart.title") 
 3. 用钩子方法自定义的字段 
      extra = serializers.SerializerMethodField(read_only=True)
 4. 用类嵌套定义的自定义字段 depart = DepartModelSerializer(many=False)
 5. 新添加数据时，触发create() , 对于外键等字段一定要定义create方法来处理post请求，否则会报错，也就是说上行和下行都要手动处理，哪么除了create方法，应该还有update方法才对 
```

Django REST framework中的Serializer使用类来定义，须继承自`rest_framework.serializers.Serializer`

**反序列化：**

使用序列化器进行反序列化时，需要对数据进行验证后，才能获取验证成功的数据或保存成模型类对象。在获取反序列化的数据前，必须调用**is_valid()**方法进行验证，验证成功返回True，否则返回False。

验证失败，可以通过序列化器对象的errors属性获取错误信息，返回字典，包含了字段和字段的错误。如果是非字段错误，可以通过修改REST framework配置中的NON_FIELD_ERRORS_KEY来控制错误字典中的键名。

验证成功，可以通过序列化器对象的validated_data属性获取数据。

```python
from rest_framework import serializers

# 定义序列化器
class TestSerializer(serializers.Serializer):
    name = serializers.CharField() 
    age = serializers.IntegerField()
    email = serializers.EmailField(label="邮箱")


# 创建序列化对象，基本语法：
serializer = Serializer(instance=None, data=empty, **kwarg)
'''
说明：
1）用于序列化时，将模型类对象传入instance参数
2）用于反序列化时，将要被反序列化的数据传入data参数=
3）除了instance和data参数外，在构造Serializer对象时，还可通过context参数额外添加数据，如:
serializer = AccountSerializer(account, context={'request': request})
'''
```

综合使用：

```python
from rest_framework import serializers

class TestSerializer(serializers.Serializer):
    name = serializers.CharField() 
    age = serializers.IntegerField()
    email = serializers.EmailField(label="邮箱")
		
    # 钩子函数，判断数据的有效性
    def validate_email(self,value):
          if re.match(r"^\w+@\w+\.\w+$",value):
              return value
          else:
              raise serializers.ValidationError("邮箱格式错误")

# 在查询中使用
class UserView(APIView):
    def get(self, request, *args, **kwargs):
        users_obj = models.user.objects.all()
        # 如果要被序列化的是包含多条数据的查询集QuerySet，可以通过添加many=True参数补充说明
        ser = RoleSerializer(instance=users_obj, many=True)
        # ser.data就是序列化的结果，.data属性可以获取序列化后的数据为字典格式
        ret = json.dumps(ser.data,ensure_ascii=False)
        return HttpResponse(ret)
              
# 反序列化
class CustomerView(APIView):  
    def post(self, request,*args, **kwargs):
        ser = CustomerSerializer(data=request.data)  
        # modelserializer的获取用户提交的方式相同，is_valid校验之后就生成validated_data or errors. 
        if ser.is_valid():
            print(ser.validated_data)
            # 手动存储数据，然后返回, 这里没有ser.save()方法，因为没有关联model 
            
            return Response({"code":200, "data":"创建成功"})
        else:
            return Response({"code":400, "data":ser.errors})
          
```

其他方法：

```python
#testSer = TestSerializer(data={k:v,…})
testSer = TestSerializer(data=data_dict)   # 这里的data_dict需为字典类型
# 判断有效性
if testSer.is_valid():
		...
 
    # 还可以使用类似于钩子的方法对数据进行校验：
    # 多个数据：
    def validate(self, data)：
      ...


    def validate_字段名(self):
      ...


    # 反序列化-保存数据：create()和update()
    def create(self, validated_data):    
     # 新建
      return instance

    # 更新保存
    def update(self, instance, validated_data):         
      return instance

    # 实现了上述两个方法后，在反序列化数据的时候，就可以通过save()方法返回一个数据对象实例了
    test = serializer.save()
    '''
    注意：
      1、如果创建序列化器对象的时候，没有传递instance实例，则调用save()方法的时候，create()被调用
      2、相反，如果传递了instance实例，则调用save()方法的时候，update()被调用。
    '''
```

### 模型化序列化器

ModelSerializer与常规的Serializer相同，但提供了：

- 基于模型类自动生成一系列字段
- 基于模型类自动为Serializer生成validators，比如unique_together
- 包含默认的create()和update()的实现

简单定义一个序列化器：

```python
class TestSerializer(serializers.ModelSerializer):
  """序列化器"""
 
  # 添加额外字段
  # fields = ('id', 'title', 'pub_date', 'bread', 'bcomment')
  class Meta:
    model = Test
    fields = '__all__'
 	 	# exclude = ['排除哪个字段']
```



# celery

参自：https://www.cnblogs.com/wdliu/p/9530219.html

## 配置使用

​	celery很容易集成到Django框架中，当然如果想要实现定时任务的话还需要安装django-celery-beta插件，后面会说明。需要注意的是Celery4.0只支持Django版本>=1.8的，如果是小于1.8版本需要使用Celery3.1。

```python
# 项目目录结构
celeryproj
├── app01
│   ├── __init__.py
│   ├── apps.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tasks.py
│   └── views.py
├── manage.py
├── celeryproj
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── templates
```

按以下步骤执行 

1、配置 settings 中的celery

settings.py

```python
############## celery的配置 ##############
CELERY_BROKER_URL = 'redis://192.168.9.30:6379/0' 				# Broker配置，使用Redis作为消息中间件
CELERY_RESULT_BACKEND = 'redis://192.168.9.30:6379/1' 		# BACKEND配置，这里使用redis
CELERY_TASK_SERIALIZER = 'pickle'													# 任务序列化方式
CELERY_RESULT_SERIALIZER = 'pickle' 											# 任务执行结果序列化方案，也可以是json
CELERY_ACCEPT_CONTENT = ['pickle', 'json']								# 指定任务接受的序列化类型
```

**扩展：**使用django的orm作为结果存储：

​	除了redis、rabbitmq能做结果存储外，还可以使用Django的orm作为结果存储，当然需要安装依赖插件，这样的好处在于我们可以直接通过django的数据查看到任务状态，同时为可以制定更多的操作，下面介绍如何使用orm作为结果存储。

```python
pip install django-celery-results
```

- 配置settings.py，注册app

```python
INSTALLED_APPS = (
    ...,
    'django_celery_results',
)
```

- 修改backend配置，将redis改为django-db

```python
#	CELERY_RESULT_BACKEND = 'redis://192.168.9.30:6379/0' 	# BACKEND配置，这里使用redis
CELERY_RESULT_BACKEND = 'django-db'  	# 使用django orm 作为结果存储
```

- 修改数据库

```
python3 manage.py migrate django_celery_results
```

此时数据库会多创建一个表：

<img src="./images/Django/image-20240626141219144.png" alt="image-20240626141219144" style="zoom:50%;" />

有时候需要对task表进行操作，以下源码的表结构定义：

```python
class TaskResult(models.Model):
    """Task result/status."""

    task_id = models.CharField(_('task id'), max_length=255, unique=True)
    task_name = models.CharField(_('task name'), null=True, max_length=255)
    task_args = models.TextField(_('task arguments'), null=True)
    task_kwargs = models.TextField(_('task kwargs'), null=True)
    status = models.CharField(_('state'), max_length=50,
                              default=states.PENDING,
                              choices=TASK_STATE_CHOICES
                              )
    content_type = models.CharField(_('content type'), max_length=128)
    content_encoding = models.CharField(_('content encoding'), max_length=64)
    result = models.TextField(null=True, default=None, editable=False)
    date_done = models.DateTimeField(_('done at'), auto_now=True)
    traceback = models.TextField(_('traceback'), blank=True, null=True)
    hidden = models.BooleanField(editable=False, default=False, db_index=True)
    meta = models.TextField(null=True, default=None, editable=False)

    objects = managers.TaskResultManager()

    class Meta:
        """Table information."""

        ordering = ['-date_done']

        verbose_name = _('task result')
        verbose_name_plural = _('task results')

    def as_dict(self):
        return {
            'task_id': self.task_id,
            'task_name': self.task_name,
            'task_args': self.task_args,
            'task_kwargs': self.task_kwargs,
            'status': self.status,
            'result': self.result,
            'date_done': self.date_done,
            'traceback': self.traceback,
            'meta': self.meta,
        }

    def __str__(self):
        return '<Task: {0.task_id} ({0.status})>'.format(self)
```

2、在项目目录celeryproj/celeryproj/目录下新建celery.py：

功能：发现配置文件，发现apps下的tasks文件 

celery.py

```python

#!/usr/bin/env python
# -*- coding:utf-8 -*-

import os
from celery import Celery

# 把celery和django进行组合，识别和加载django的配置文件
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'celerytest.settings')
app = Celery('celerytest')

# 通过app对象加载配置，使用CELERY_ 作为前缀，在settings中写配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 加载任务，发现任务文件每个app下的task.py
# 参数必须必须是一个列表，里面的每一个任务都是任务的路径名称：app.autodiscover_tasks(["任务1","任务2"])
app.autodiscover_tasks()

```

3、在celeryproj/celeryproj/下的`__init__.py`中加入以下内容，意在让django启动时执行celery.py下的代码

```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

4、在apps中建立tasks.py文件，加入任务 

tasks.py

```python
from celery import shared_task

# 使用shared_task进行装饰，添加任务
@shared_task
def add(x, y):
    return x + y

@shared_task
def mul(x, y):
    return x * y

@shared_task
def xsum(numbers):
    return sum(numbers)
```

@app.task与@shared_task的区别

```python
@app.task：
	1、Celery库中的一个装饰器，用于将函数注册为Celery任务。
	2、依赖于一个特定的Celery实例（由Celery('myproject')创建），其中'myproject'是项目名。
	3、使用bind=True时，任务函数的第一个参数将是任务实例本身（通常命名为self），这使得在任务函数内部可以访问任务实例的属性和方法。
	使用场景：当任务仅在单个Celery应用程序中定义和使用时，通常使用@app.task。
@shared_task：
	1、Celery的另一个装饰器，用于将函数注册为共享任务。
	2、共享任务是一种特殊类型的任务，可以跨多个Celery应用程序共享和调用。
	3、它不依赖于某个特定的Celery实例，而是加载到内存后自动添加到Celery对象中。
	4、使用base=MyHookTask, bind=True时，可以进一步自定义任务的基类并允许任务函数内部访问任务实例。
	使用场景：当任务需要在多个Celery应用程序中共享，或者希望利用共享任务的特性时，使用@shared_task更为合适。
```

5、进入项目的celeryproj目录启动worker，准备接活 

```python
# 备注： 此处只需要支出celery.py所在文件的目录名，会自动查找celery.py文件并执行  
celery worker -A celeryproj -l info
```

6、给worker派活，在view中启动任务 

```python
class testView(APIView):
    def get(self,request, *args, **kwargs):

        result = tasks.add.delay(89,20)
        return  Response({"status":True, "msg":"连接成功,result id:" + str(result.id)})

    def post(self,request,*args,**kwargs):
        print(request.POST)

        nid = request.data.get("nid")   
				# 注意drf中request.POST的内容是空的，转到了request.data中

        from celerytest import celery_app
        from celery.result import AsyncResult

        # 下面的id是从第三步中得到的，异步执行结果 
        result_obj = AsyncResult(id=nid, app=celery_app)

        print(result_obj.status)  # 结果状态

        print(result_obj.get())  # 结果内容

        return Response({"status": True, "msg": "结果:" + str(result_obj.get())})
```

## 定义与触发任务

任务定义在每个tasks文件中，app01／tasks.py：

```python
from celery import shared_task

@shared_task
def add(x, y):
    return x + y

@shared_task
def mul(x, y):
    return x * y
```

视图中触发任务

```python
from django.http import JsonResponse
from app01 import tasks

# Create your views here.

def index(request,*args,**kwargs):
    res=tasks.add.delay(1,3)
    #任务逻辑
    return JsonResponse({'status':'successful','task_id':res.task_id})
```

若想获取任务结果，可以通过task_id使用AsyncResult获取结果

```python
from celery.result import AsyncResult
result = result_obj.get()
print(result )    # 结果内容
```



## 定时任务

如果想要在django中使用定时任务功能同样是靠beat完成任务发送功能，当在Django中使用定时任务时，需要安装django-celery-beat插件。

### 安装配置

1.beat插件安装

```python
pip3 install django-celery-beat
```

2.注册APP

```python
INSTALLED_APPS = [
    ....   
    'django_celery_beat',
]
```

3.数据库变更

```python
python3 manage.py migrate django_celery_beat
```

4.分别启动woker和beta

```python
#启动beta 调度器使用数据库
celery -A proj beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler  

#启动woker
celery worker -A taskproj -l info
```

5.配置admin

urls.py

```python
# urls.py
from django.conf.urls import url
from django.contrib import admin
 
urlpatterns = [
    url(r'^admin/', admin.site.urls),
]
```

6.创建用户

```python
python3 manage.py createsuperuser 
```





```python
# 这里加载是apply_async,  eta是开始执行时间 ， target_time必须是utc时间 
result = tasks.add.apply_async(args=[8,9], eta=target_time)

# utc时间处理如下：
# 定时任务celery 4.4版本默认使用utc时间，即使指定时区也是这样 
ctime = datetime.datetime.now()
utc_time = datetime.datetime.utcfromtimestamp(ctime.timestamp())
target_time =utc_time + datetime.timedelta(seconds=20)
result = tasks.add.apply_async(args=[8,9], eta=target_time)
```

完整代码

```python

class testView(APIView):
    def get(self,request, *args, **kwargs):
        # 定时任务   celery 4.4版本默认使用utc时间，即使指定时区也是这样 
        ctime = datetime.datetime.now()
        utc_time = datetime.datetime.utcfromtimestamp(ctime.timestamp())
        target_time =utc_time + datetime.timedelta(seconds=20)

        result = tasks.add.apply_async(args=[8,9], eta=target_time)
        return  Response({"status":True, "msg":"连接成功,定时任务 result id:" + str(result.id)})

    def post(self,request,*args,**kwargs):
        print(request.POST)
        nid = request.data.get("nid")

        from celerytest import celery_app
        from celery.result import AsyncResult

        # 下面的id是从第三步中得到的
        result_obj = AsyncResult(id=nid, app=celery_app)
        print(result_obj.status)  # 结果状态
        if result_obj.successful():
            result = result_obj.get()   # 只有状态成功才可以获取结果，否则报错
            result_obj.forget()         # 移除redis中存贮的结果
            # result_obj.revoke()       # 取消任务 （未执行前）  
            # result_obj.revoke(terminate=True)  # 强制取消
        elif result_obj.failed():
            result = " failed"
        else:
            result = "working "

        return Response({"status": True, "msg": "结果:" + str(result)})

```

