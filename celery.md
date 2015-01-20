用到的：Django Celery Redis supervisor

celery是由python写的Distributed Task Queue。

django可以说是他的座上宾了，对其支持很好，以前django使用djcelery来使用celery，但是最新的celery版本对django支持更好，可以不用（当然也可以用来记录结果等）。

我目前项目是用celery来做数据统计和消息推送，这些都是最好异步完成的，对于每个task的结果，我看看log就可以，不需要记录，所以我没有用djcelery。

broker官方推荐的是用RabbitMQ和redis，RabbitMQ是erlang写的，比较难懂，要安装erlang等，比较麻烦，所以我用redis（如果对稳定性等要求很高，建议用RabbitMQ）。

然后用supervisor来管理celery的work进程。

#########分割线####################

以下是如何在django的代码中使用celery：

1. 安装相应的软件

    apt-get/yum安装redis-server；

    easy_install/pip安装 redis、celery、supervisor

2. 比如现在django 代码目录叫test,那么在test/test目录下建立celery.py文件，然后编辑：
```
# -*- coding: utf-8  

from __future__ import absolute_import

import os

from celery import Celery

from django.conf import settings

# set the default Django settings module for the 'celery' program.

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'test.settings')

#redis://:password@hostname:port/db_number

app = Celery('celery_test', broker='redis://localhost:6379/0')

# Using a string here means the worker will not have to

# pickle the object when using Windows.

app.config_from_object('django.conf:settings')

app.autodiscover_tasks(lambda: settings.INSTALLED_APPS) 
``` 

在test/test/__init__.py添加：

```
# -*- coding: utf-8  -*-

from __future__ import absolute_import

# This will make sure the app is always imported when

# Django starts so that shared_task will use this app.

from .celery import app as celery_app
```

这样 就可以在每个app下面建立tasks.py文件，写相应的task函数，比如：

```
# -*- coding: utf-8  -*-

from __future__ import absolute_import

from celery import shared_task

 

@shared_task

def celery_test(a=0):

    return 2*a

 ```

其他地方只需要引入这个函数，比如引入celery_test，然后celery_test.delay(a=3)

执行celery  -A celery_test worker -l info 这样就可以在shell里面看到执行情况


3. 用supervisor来控制celery进程

    部署的时候，还是需要将celery work进程像daemon这样来执行的，所以需要配置supervisor来管理这些进程（除了supervisor还有其他方式，不是唯一）

    echo_supervisord_conf > /etc/supervisord.conf （如果已经创建过/etc/supervisord.conf，跳过）

    编辑/etc/supervisord.conf，添加如下(具体参数根据实际情况调整)
```
[program:celery_test]

; Set full path to celery program if using virtualenv

command=celery worker -A celery_test –loglevel=INFO

process_name=%(program_name)s_%(process_num)02d

directory= /test/test

user=root

numprocs = 4

stdout_logfile=/opt/love/logs/celery_worker.log

stderr_logfile=/opt/love/logs/celery_error.log

autostart=true

autorestart=true

startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.

; Increase this if you have very long running tasks.

stopwaitsecs = 600

; When resorting to send SIGKILL to the program to terminate it

; send SIGKILL to its whole process group instead,

; taking care of its children as well.

killasgroup=true
```

然后启动supervisord（如果已经在运行了，supervisorctl来启动新加的进程）


参考：http://docs.celeryproject.org/en/latest/django/index.html
