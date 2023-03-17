## 使用Celery实现任务异步化

Celery 是一个简单、灵活且可靠的，处理大量消息的分布式系统，并且提供维护这样一个系统的必需工具。它是一个专注于实时处理的任务队列，同时也支持任务调度。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/60a7d9b8-0f32-41f6-b01e-51bc2cee0db0/Untitled.png)

Celery是一个本身不提供队列服务，官方推荐使用RabbitMQ或Redis来实现消息队列服务

- 安装RabbitMQ
    
    ```python
    docker pull rabbitmq
    docker run -d -p 5672:5672 --name myrabbit rabbitmq
    docker container exec -it myrabbit /bin/bash
    ```
    
- 创建用户、资源以及分配操作权限
    
    ```python
    rabbitmqctl add_user root root
    rabbitmqctl set_user_tags root administrator
    rabbitmqctl add_vhost vhost1
    rabbitmqctl set_permissions -p vhost1 root ".*" ".*" ".*"
    ```
    
- 创建celery实例
    
    ```python
    import os
    
    from celery import Celery
    
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_app.settings")
    
    app = Celery(
        'celery',
        broker="amqp://root:root@localhost:32780/broker"
    )
    
    app.config_from_object(
        "apps.tasks.celery_config", silent=True)
    ```
    
- celery配置
    
    ```python
    from kombu import Exchange, Queue
    
    # timezone
    CELERY_TIMEZONE = 'UTC'
    
    default_exchange = Exchange('default', type='direct')
    
    CELERY_IMPORTS = ("apps.tasks.async_tasks",)
    
    CELERY_QUEUES = (
        Queue('default', default_exchange, routing_key='default', max_priority=10),
    )
    CELERYD_CONCURRENCY = 2 # celery worker number
    
    # create broker if not exists
    CELERY_CREATE_MISSING_QUEUES = True
    
    CELERYD_MAX_TASKS_PER_CHILD = 100  # max tasks number per celery worker
    
    CELERYD_FORCE_EXECV = True  # avoid deadlock
    
    CELERY_ACKS_LATE = True
    
    CELERYD_PREFETCH_MULTIPLIER = 4
    
    # speed limit
    CELERY_DISABLE_RATE_LIMITS = True
    CELERY_TASK_SERIALIZER = "pickle"
    CELERY_ACCEPT_CONTENT = ["json", "pickle"]
    
    CELERY_DEFAULT_QUEUE = 'default'
    CELERY_DEFAULT_EXCHANGE = 'default'
    CELERY_DEFAULT_ROUTING_KEY = 'default'
    ```
    

- 定义异步任务
    
    ```python
    @app.task
    def export_companies():
        from apps.modules.companies.generator import export_all_companies
        export_all_companies()
        return
    ```
    
    ```python
    import csv
    from .models import Companies
    
    def export_all_companies():
    
        companies = Companies.objects.all().values()
        with open('company.csv', 'w', encoding='utf-8') as fp:
            company_file = csv.DictWriter(fp, fieldnames=['name', 'email'])
            company_file.writeheader()
            for company in companies:
                company_file.writerow({
                    "name": company['name'],
                    "email": company['email']
                })
        return
    ```
    
- 启动celery worker
    
    ```python
    celery -A apps.tasks.task worker -Q default --loglevel=debug
    ```
    

## 定时任务

- 配置定时任务
    
    ```python
    CELERYBEAT_SCHEDULE = {
        "schedule_test": {
            "task": "apps.tasks.async_tasks.schedule_test",
            "schedule": crontab(),
            'args': ()
        }
    }
    ```
    
    ```python
    @app.task
    def schedule_test():
        print("定时任务")
        return
    ```
    
- 启动定时任务
