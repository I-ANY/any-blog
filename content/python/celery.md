+++
title = "celery"
date = "2026-05-28T00:01:08+08:00"
draft = false
weight = 5
+++

# celery

Celery是一个简单、灵活且可靠的，处理大量消息的分布式系统，专注于实时处理的异步任务队列，同时也支持任务调度。

- **消息中间件**

​	Celery本身不提供消息服务，但是可以方便的和第三方提供的消息中间件集成。包括，RabbitMQ, Redis等等

- **任务执行单元**

​	Worker是任务执行单元，负责从消息队列中取出任务执行，它可以启动一个或者多个，也可以启动在不同的机器节点，这就是其实现分布式的核心。

- **任务结果存储**

​	Backend结果存储官方也提供了诸多的存储方式支持：RabbitMQ、 Redis、Memcached,SQLAlchemy, Django ORM、Apache Cassandra、Elasticsearch

Celery还支持不同的并发和序列化的手段：

- 并发：Prefork, Eventlet, gevent, threads/single threaded
- 序列化：pickle, json, yaml, msgpack. zlib, bzip2 compression， Cryptographic message signing 等等

使用场景：

​	1.web应用：当用户在网站进行某个操作需要很长时间完成时，我们可以将这种操作交给Celery执行，直接返回给用户，等到Celery执行完成以后通知用户，大大提好网站的并发以及用户的体验感。

　　2.任务场景：比如在运维场景下需要批量在几百台机器执行某些命令或者任务，此时Celery可以轻松搞定。

　　3.定时任务：向定时导数据报表、定时发送通知类似场景，虽然Linux的计划任务可以帮我实现，但是非常不利于管理，而Celery可以提供管理接口和丰富的API。

工作原理：

1. 任务模块Task包含异步任务和定时任务。其中，异步任务通常在业务逻辑中被触发并发往消息队列，而定时任务由Celery Beat进程周期性地将任务发往消息队列；
2. 任务执行单元Worker实时监视消息队列获取队列中的任务执行；
3. Woker执行完任务后将结果保存在Backend中;

原理图

<img src="./images/celery/image-20240625151001633.png" alt="image-20240625151001633" style="zoom:50%;" />



场景1： 有一个耗时的任务，为了不让用户hang住等待，可先接受业务，返回业务号，然后让后台处理之后再返回结果，处理耗时任务

```text
celery生成一个列表，他不负责处理，只是把任务扔入这个列表，表明任务在等待，下面有worker，结果放入另一个队列，对于耗时的任务可以将任务列入broker中，给用户返回一个任务id, 由worker去领取任务并处理。 
app只负责从celery获取任务，任务完成后放入backends
用户提供id就可以上backends中查询执行情况，这比crontab要好用 
```

场景2   有一个定时需要执行的任务

```python
定时任务： 放在任务列表但定时领取任务，定时发布和定时拍卖等 
设定任务的执行时间 
```

## 安装

1. pip3安装celery（或者源码安装）

   ```python
   # 注意5版本以后必须要python3.5的支持
   pip install -U Celery
   ```

2. redis或者rabitMQ（作为borker）

## 简单使用

创建项目celerypro目录

创建异步任务执行文件celery_task.py

```python
import celery
import time

# 消息中间件的配置
backend='redis://192.168.9.30:6379/1'
broker='redis://192.168.9.30:6379/2'
cel=celery.Celery('test',backend=backend,broker=broker)

@cel.task
def send_email(name):
    print("向%s发送邮件..."%name)
    time.sleep(5)
    print("向%s发送邮件完成"%name)
    return "ok"
```

启动异步任务(worker)，启动之后，worker将会持续去监听中间件消息队列的情况，如产生任务直接执行任务

```python
celery worker -A celerypro -l debug
#　worker: 代表第启动的角色是work当然还有beat等其他角色；
#　-A ：项目路径，这里我的目录是project
#　-l：启动的日志级别，更多参数使用celery --help查看

# 备注： 3.0版本按老命令 celery worker -A task -l info 会报错， 5.0以后命令顺序改了，按提示得到正确的结果, 应该是： celery -A task  worker  -l info
# 因为Celery 4.0+的版本官方不支持Windows, 而默认的并发模式Prefork是基于Linux实现的，如何需要在win上运行，可行的解决方案是使用gevent模块
pip install gevent
celery -A server worker -l Info -P gevent
```

启动之后，查看日志输出，会发现我们定义的任务，以及相关配置

```python
C:\codeProject\python\celery>celery -A project worker -l INFO 
 -------------- celery@WWX90099A v5.4.0 (opalescent)
--- ***** -----
-- ******* ---- Windows-10-10.0.22631-SP0 2024-06-25 16:40:12
- *** --- * ---
- ** ---------- [config]
- ** ---------- .> app:         project:0x1b492f7fb10
- ** ---------- .> transport:   redis://192.168.9.30:6379/0
- ** ---------- .> results:     redis://192.168.9.30:6379/1
- *** --- * --- .> concurrency: 20 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery


[tasks]
  . project.tasks.add
```

虽然启动了worker，但是还需要通过celery的内部方法：delay或apply_async，来将任务添加到消息队列中

produce_task.py

```python
from celery_task import send_email
from celery.result import AsyncResult

# 编写broker，传递参数，delay是celery的内部方法
result1 = send_email.delay("any")
task1_id = result1.id
print(task1_id)

result2 = send_email.delay("Zhangsan")
task2_id=result2.id
print(task2_id)

# 下面的id是从上面得到的 
result_obj = AsyncResult(id=task1_id, app=app)
print(result_obj.status )  # 结果状态

result = result_obj.get()
# result.forget() # 将结果删除
print(result )    # 结果内容
```

直接运行结果输出：

```python
[Running] python -u "c:\codeProject\python\celery_test\project\produce_tasks.py"
17088635-3b45-45ce-8f1d-eba34ff31d0e
d3690bbd-d0ca-4bde-a510-1c56c6bb07b0
PENDING
ok
```

AsyncResult除了get方法用于常用获取结果方法外还提以下常用方法或属性：

- state: 返回任务状态；
- task_id: 返回任务id；
- result: 返回任务结果，同get()方法；
- ready(): 判断任务是否以及有结果，有结果为True，否则False；
- info(): 获取任务信息，默认为结果；
- wait(t): 等待t秒后获取结果，若任务执行完毕，则不等待直接获取结果，若任务在执行中，则wait期间一直阻塞，直到超时报错；
- successfu(): 判断任务是否成功，成功为True，否则为False；

**celery命令**

```python
# 查看worker的状态：
celery -A proj status

# 查看worker的详细信息：
celery -A proj inspect stats

# 查看当前正在执行的task
celery -A proj inspect active

# 查看当前队列信息
celery -A proj inspect active_queues
```

## 多任务结构

```python
# 目录结构：
.celery_tasks
|--- celery_task.py	
|--- task01.py
|--- task02.py
|--- demo
		|--- produce_task.py	
  	|--- check_result.py
```

celery.py:

```python
from celery import Celery

cel = Celery('celery_demo',
             broker='redis://192.168.9.30:6379/1',
             backend='redis://192.168.9.30:6379/2',
             # 包含以下两个任务文件，去相应的py文件中找任务，对多个任务做分类
             include=['celery_tasks.task01',
                      'celery_tasks.task02'
                      ])

# 时区
cel.conf.timezone = 'Asia/Shanghai'
# 是否使用UTC
cel.conf.enable_utc = False
```

task01.py，task02.py:

```python
#task01
import time
from celery_tasks.celery import cel

@cel.task
def send_email(res):
    time.sleep(5)
    return "完成向%s发送邮件任务"%res



#task02
import time
from celery_tasks.celery import cel
@cel.task
def send_msg(name):
    time.sleep(5)
    return "完成向%s发送短信任务"%name
```

produce_task.py:

```python
from celery_tasks.task01 import send_email
from celery_tasks.task02 import send_msg

# 立即告知celery去执行test_celery任务，并传入一个参数
result = send_email.delay('yuan')
print(result.id)
result = send_msg.delay('yuan')
print(result.id)
```

check_result.py:

```python
from celery.result import AsyncResult
from celery_tasks.celery import cel

# 使用AsyncResult方法
async_result = AsyncResult(id="562834c6-e4be-46d2-908a-b102adbbf390", app=cel)

if async_result.successful():
    result = async_result.get()
    print(result)
    # result.forget() # 将结果删除,执行完成，结果不会自动删除
    # async.revoke(terminate=True)  # 无论现在是什么时候，都要终止
    # async.revoke(terminate=False) # 如果任务还没有开始执行呢，那么就可以终止。
elif async_result.failed():
    print('执行失败')
elif async_result.status == 'PENDING':
    print('任务等待中被执行')
elif async_result.status == 'RETRY':
    print('任务异常后正在重试')
elif async_result.status == 'STARTED':
    print('任务已经开始被执行')
```

开启work：

```python
celery worker -A celery_task -l info -P eventlet
```

添加任务（执行produce_task.py)，检查任务执行结果（执行check_result.py）

## 配置文件结构

```python
# 目录结构
celerypro/
├── celery.py
├── settings.py
├── tasks.py
└── demo.py

```

各目录文件说明：

初始化Celery以及加载配置文件：celery.py

```python
#!/usr/bin/env python3
# -*- coding:utf-8 -*-

from celery import Celery

app = Celery('project')                                # 创建 Celery 实例
app.config_from_object('project.config')               # 加载配置模块
```

Celery相关配置文件：settings.py

```python
#!/usr/bin/env python3
# -*- coding:utf-8 -*-

BROKER_URL = 'redis://192.168.9.30:6379/0' # Broker配置，使用Redis作为消息中间件

CELERY_RESULT_BACKEND = 'redis://192.168.9.30:6379/0' # BACKEND配置，这里使用redis

CELERY_RESULT_SERIALIZER = 'json' # 结果序列化方案

CELERY_TASK_RESULT_EXPIRES = 60 * 60 * 24 # 任务过期时间

CELERY_TIMEZONE='Asia/Shanghai'   # 时区配置

CELERY_IMPORTS = (     # 指定导入的任务模块,可以指定多个
    'project.tasks',
)
```

tasks.py ：任务定义文件

```python
#!/usr/bin/env python3
# -*- coding:utf-8 -*-

from project import app

@app.task
def show_name(name):
    return name
```

启动Worker：

```python
celery -A project worker -l INFO
```

执行demo.py

```python
from celerypro import tasks

t = tasks.show_name.delay('Zhangsan')
t.get()      # 输出Zhangsan
```



## 执行定时任务

Celery的提供的定时任务主要靠schedules来完成，通过beat组件周期性将任务发送给woker执行。

简单任务：

 设定时间让celery执行一个定时任务，produce_task.py:

```python
from celery_task import send_email
from datetime import datetime

'''
# 方式一：取固定时间
t1 = datetime(2024, 3, 11, 16, 19, 00)
print(v1)
# 需要时国标时间
t2 = datetime.utcfromtimestamp(v1.timestamp())
print(t2)
# 调用apply_async，传更多的参数，eta接收一个日期对象（有eta就是定时任务，没有就是异步任务(等同于delay)）
result = send_email.apply_async(args=["zahngsan",], eta=t2)
print(result.id)
'''

# 方式二
ctime = datetime.now()
# 当前时间，默认用utc时间
utc_ctime = datetime.utcfromtimestamp(ctime.timestamp())

from datetime import timedelta
time_delay = timedelta(seconds=10)
# 当前时间+10s
task_time = utc_ctime + time_delay

# 使用apply_async并设定时间
result = send_email.apply_async(args=["zhangsan"], eta=task_time)
print(result.id)
```

配置文件：

创建一个period_task.py文件

```python
#!/usr/bin/env python3
# -*- coding:utf-8 -*-
from project import app
from celery.schedules import crontab

@app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    sender.add_periodic_task(10.0, add.s(1,3), name='1+3=') # 每10秒执行add
    sender.add_periodic_task(
        crontab(hour=16, minute=56, day_of_week=1),      		#每周一下午四点五十六执行sayhai
        sayhi.s('zhangsan'),name='say_hi'
    )
```

settings.py文件

```python
#!/usr/bin/env python3
# -*- coding:utf-8 -*-
# Author:wd

BROKER_URL = 'redis://192.163.9.30:6379/1' # Broker配置，使用Redis作为消息中间件
CELERY_RESULT_BACKEND = 'redis://192.163.9.30:6379/2' # BACKEND配置，这里使用redis

CELERY_RESULT_SERIALIZER = 'json' # 结果序列化方案

CELERY_TASK_RESULT_EXPIRES = 60 * 60 * 24 # 任务过期时间

CELERY_TIMEZONE='Asia/Shanghai'   # 时区配置

CELERY_IMPORTS = (     # 指定导入的任务模块,可以指定多个
    'project.tasks',
    'project.period_task', 	#定时任务
)
```

启动worker和beat：

```python
#启动work，持续监听消息队列（此时注意，如果当前redis中存在历史消息，一启动，就会直接执行这些历史的消息数据，如果有很多，可直接将broker对应的redis库下面的celery键直接删除即可）
celery worker -A project -l INFO
#启动beat，周期性地将任务插入到消息队列中（注意此时对应的文件路径）
celery beat -A  project.period_task -l  INFO   
```

多任务结构：

celery.py修改如下:

```python
from datetime import timedelta
from celery import Celery
from celery.schedules import crontab

cel = Celery('tasks', broker='redis://192.168.9.30:6379/1', backend='redis://192.168.9.30:6379/2', include=[
    'celery_tasks.task01',			# 指定导入的任务模块,可以指定多个
    'celery_tasks.task02',
])
cel.conf.timezone = 'Asia/Shanghai'
cel.conf.enable_utc = False

cel.conf.beat_schedule = {
    # 名字随意命名
    'add-every-10-seconds': {
        # 任务路径，键固定为task，执行tasks01下的send_email函数
        'task': 'celery_tasks.task01.send_email',  
        # 键固定为schedule，每隔2秒执行一次
        # 'schedule': 1.0,
        # 'schedule': crontab(minute="*/1"),
        'schedule': timedelta(seconds=6),
        # 固定为args，给函数传递参数
        'args': ('张三',)
    },
  	# 其他写法
    'add-every-12-seconds': {
        'task': 'celery_tasks.task01.send_email',
        # 每年4月11号，8点42分执行
        'schedule': crontab(minute=42, hour=8, day_of_month=11, month_of_year=4),
        'args': ('张三',)
    },
}
```

启动worker和beat：

```python
#启动work，持续监听消息队列（此时注意，如果当前redis中存在历史消息，一启动，就会直接执行这些历史的消息数据，如果有很多，可直接将broker对应的redis库下面的celery键直接删除即可）
celery worker -A project -l INFO
#启动beat，周期性地将任务插入到消息队列中（注意此时对应的文件路径）
celery beat -A  project.period_task -l  INFO   
```

