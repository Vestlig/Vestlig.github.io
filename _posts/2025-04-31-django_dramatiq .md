---
title: django_dramatiq
date: 2025-04-31 13:48:00 +0800
categories: [三方插件, 消息队列]  # categories是文章的分类，支持至多两级分类
tags: [dramatiq配置]  # tags是文章的标签，可以有任意多个
---

# 一、环境准备与依赖安装  

## 基础依赖  

- Python 3\.8\+  

- Django 4\.0\+  

- RabbitMQ 3\.12\+（需提前安装并启动服务）  

- Erlang 25\.0\+（RabbitMQ 依赖）  



## Python 包安装  

```Plain Text
pip install django django-dramatiq dramatiq[rabbitmq]  
```



---



# 二、Django 项目配置  

## 添加应用到 `INSTALLED_APPS`  

```Plain Text
# settings.py  
INSTALLED_APPS = [  
    ...,  
    "django_dramatiq",  
]  
```



## 配置消息代理（RabbitMQ）  

```Plain Text
# settings.py  
DRAMATIQ_BROKER = {  
    "BROKER": "dramatiq.brokers.rabbitmq.RabbitmqBroker",  
    "OPTIONS": {  
        "url": "amqp://guest:guest@localhost:5672//",  # RabbitMQ 默认地址  
    },  
    "MIDDLEWARE": [  
        "dramatiq.middleware.Prometheus",  
        "dramatiq.middleware.AgeLimit",  
    ]  
}  
```



## 设置任务存储路径（可选）  

```Plain Text
DRAMATIQ_TASKS_DATABASE = "default"  # 使用 Django 数据库存储任务结果  
```



---



# 三、定义与调用异步任务  

## 创建任务函数  

```Plain Text
# tasks.py  
import dramatiq  
from django.core.mail import send_mail  

@dramatiq.actor  
def send_welcome_email(user_email):  
    send_mail(  
        "Welcome",  
        "Thank you for registering!",  
        "admin@example.com",  
        [user_email],  
        fail_silently=False,  
    )  
```



## 在视图中调用任务  

```Plain Text
# views.py  
from .tasks import send_welcome_email  

def register_user(request):  
    # 同步业务逻辑  
    user_email = request.POST.get("email")  
    send_welcome_email.send(user_email)  # 异步触发  
    return HttpResponse("Registration successful!")  
```



---



# 四、运行与管理任务  

## 启动 Worker 进程  

```Plain Text
python manage.py rundramatiq --processes 4 --threads 2  
```

- `--processes`：进程数（建议与 CPU 核心数一致）  

- `--threads`：每个进程的线程数  



## 监控任务队列  

- RabbitMQ 管理界面：访问 `http://localhost:15672` 查看队列状态  

- Django Admin：集成 `django_dramatiq` 后，可查看任务历史记录  



---



# 五、高级功能实现  

## 任务重试机制  

```Plain Text
@dramatiq.actor(max_retries=3, min_backoff=5000)  
def process_payment(order_id):  
    if payment_failed:  
        raise Exception("Retry payment")  
```



## 定时任务调度  

```Plain Text
from dramatiq import cron  

@dramatiq.actor  
@cron("0 3 * * *")  # 每天凌晨3点执行  
def generate_daily_report():  
    ...  
```



## 结果存储与查询  

```Plain Text
from django_dramatiq.models import Task  

def check_task_status(task_id):  
    task = Task.objects.get(task_id=task_id)  
    return task.status  # 可能为 "done", "failed", "pending"  
```



---



# 六、最佳实践与注意事项  

## 连接池管理  

- 使用 `CONNECTION_POOL` 配置复用 RabbitMQ 连接，避免频繁创建连接  

```Plain Text
OPTIONS = {  
    "url": "amqp://guest:guest@localhost:5672//",  
    "connection_pool": {  
        "max_size": 10,  
        "ttl": 300,  
    }  
}  
```



## 错误处理策略  

- 捕获异常并记录日志：  

```Plain Text
@dramatiq.actor  
def critical_task():  
    try:  
        ...  
    except Exception as e:  
        logging.error(f"Task failed: {str(e)}")  
        raise  
```



## 性能优化  

- 批量处理消息：通过 `pipe` 合并多个任务  

```Plain Text
from dramatiq import pipeline  

@dramatiq.actor  
def stage1(data):  
    return processed_data  

@dramatiq.actor  
def stage2(data):  
    ...  

result = pipeline([stage1.message(), stage2.message()]).run()  
```



---



# 七、常见问题解决  

## 任务未执行  

- 检查 RabbitMQ 服务状态：`systemctl status rabbitmq-server`  

- 确认 Worker 是否监听正确队列：`python manage.py rundramatiq --queues high_priority,default`  



## 消息堆积  

- 动态扩展 Worker：`--processes 8`（根据负载调整）  

- 设置消息 TTL：在 RabbitMQ 队列中配置 `x-message-ttl`  



## 跨项目通信  

- 统一 Exchange 名称：在 Django 和外部服务中使用相同 Exchange 定义  



---



# 附录  

- django\_dramatiq 官方文档：https://django\-dramatiq\.io/  

- RabbitMQ Python 客户端指南：https://www\.rabbitmq\.com/tutorials/tutorial\-one\-python\.html  



\-（示例基于 Django 4\.2 和 django\-dramatiq 2\.4 版本）\*



