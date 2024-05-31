
## 原理
Celery 是一个异步任务队列，一个Celery有三个核心组件：

- Celery 客户端: 用于发布后台作业；其也可以是一个beat。
 
- Celery workers: 运行后台作业的进程。Celery 支持本地和远程的 workers，可以在本地服务器上启动一个单独的 worker，也可以在远程服务器上启动worker，来进行任务的消费；

- 消息代理: 客户端通过消息队列和 workers 进行通信，Celery 支持多种方式来实现这些队列。最常用的代理就是 RabbitMQ 和 Redis。

##  安装配置
celery是python中用于异步/定时任务的模块，可集成于django当中


### celery依赖安装
```python
pip install celery
pip install django-celery-beat

pip install redis
```



### celery.py的配置

```python
import os  
from celery import Celery  
from django.conf import settings  
  
# 设置环境变量  
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'djangoExample.settings')  
  
# 实例化  
app = Celery('djangoExample')  
  
# namespace='CELERY'作用是允许你在Django配置文件中对Celery进行配置  
# 但所有Celery配置项必须以CELERY开头，防止冲突  
app.config_from_object('django.conf:settings', namespace='CELERY')  
  
  
# 自动从Django的已注册app中发现任务  
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)  
  
# 一个测试任务  
@app.task(bind=True)  
def debug_task(self):  
 print(f'Request: {self.request!r}')
```

### settings.py当中配置broker
```python
CELERY_RESULT_BACKEND = "django-db"  
# celery内容等消息的格式设置，默认json  
CELERY_ACCEPT_CONTENT = ['application/json', ]  
CELERY_TASK_SERIALIZER = 'json'  
CELERY_RESULT_SERIALIZER = 'json'  

#redis配置
CELERY_BROKER_URL = "redis://:yyh6301@120.25.172.211:6379/0"  
USE_TZ=True  
TIME_ZONE = 'Asia/Shanghai'  
CELERY_TIMEZONE = TIME_ZONE
```



### 使用 celery 来测试celery和redis是否配置完成
```shell
celery -A djangoExample worker -l info 
```


出现xxx ready 则表示配置成功
![[Pasted image 20240531092931.png]]


## 任务
编写任务主要有两种方式，一种是直接在项目当中直接配置，专属于某个项目，另一种是在app下配置，可以在各个app下进行复用

### 专属某个项目的task
```python
 # myproject/tasks.py
 # 专属于myproject项目的任务
 app = Celery('myproject'）
 @ app.task
 def test()：
     pass
```


### 可复用的task
```python​
 # app/tasks.py, 可以复用的task
 from celery import shared_task
 import time
 ​
 @shared_task
 def add(x, y):
     time.sleep(2)
     return x + y
```



## 异步调用方式

celery异步调用有两种方式:
```python
 # 方法一：delay方法
 task_name.delay(args1, args2, kwargs=value_1, kwargs2=value_2)
 ​
 # 方法二： apply_async方法，与delay类似，但支持更多参数
 task.apply_async(args=[arg1, arg2], kwargs={key:value, key:value})
```




## 定时任务的配置
1. 直接在settings.py中使用代码来配置
2. 在django admin中去配置

这里是settings.py当中的编写方式，task表示要执行的任务，schedule表示执行周期，args表示传入的参数
```python
#settings.py
from datetime import timedelta  
CELERY_BEAT_SCHEDULE = {  
    "add-every-3s": {  
        "task": "app01.tasks.add",  
        'schedule': 3.0, # 每3秒执行1次  
        'args': (3, 8) # 传递参数-  
    },  
}
```


启动方式：
```shell
#启动worker
celery -A djangoExample worker -l info

#启动beat
celery -A djangoExample beat -l info

#静默启动：
nohup celery -A djangoExample worker -l info > worker.log 2>&1 &
nohup celery -A djangoExample beat -l info > beat.log 2>&1 &

```


beat开始生产任务：
![[Pasted image 20240531093407.png]]


worker进行任务的消费
![[Pasted image 20240531093439.png]]