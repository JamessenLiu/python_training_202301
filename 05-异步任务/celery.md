## 使用Celery实现任务异步化

Celery 是一个简单、灵活且可靠的，处理大量消息的分布式系统，并且提供维护这样一个系统的必需工具。它是一个专注于实时处理的任务队列，同时也支持任务调度。

![image](https://user-images.githubusercontent.com/49837274/225781602-92b087aa-1b2c-4460-83b1-cd4b579d0a85.png)

Celery是一个本身不提供队列服务，官方推荐使用RabbitMQ或Redis来实现消息队列服务

    ```
    
- 创建celery实例
    
    ```python
    import os
    import django

    from celery import Celery
    from django.conf import settings
    config = os.getenv("MODE")

    if config is None:
        config = "django_study2.settings"

    os.environ.setdefault('DJANGO_SETTINGS_MODULE', config)
    django.setup()


    MQ_HOST = settings.REDIS_MQ_HOST
    MQ_PORT = settings.REDIS_MQ_PORT
    MQ_PASSWORD = settings.REDIS_MQ_PASSWORD
    MQ_DB = settings.REDIS_MQ_DB

    app = Celery(
        'celery',
        broker=f"redis://:{MQ_PASSWORD}@{MQ_HOST}:{MQ_PORT}/{MQ_DB}"
    )

    app.config_from_object(
        'tasks.celery_config', silent=True, force=True
    )
    ```
    
- celery配置
    
    ```python
    from kombu import Exchange, Queue

    # timezone
    timezone = 'UTC'

    # default exchange
    default_exchange = Exchange('default', type='direct')


    imports = ("tasks.async_tasks",)

    # create queue
    task_queues = (
        Queue('default', default_exchange, routing_key='default', max_priority=10),
    )

    worker_concurrency = 2  # celery worker number

    # create broker if not exists
    task_create_missing_queues = True

    worker_max_tasks_per_child = 100  # max tasks number per celery worker

    CELERYD_FORCE_EXECV = True  # avoid deadlock

    task_acks_late = True

    worker_prefetch_multiplier = 4

    # speed limit
    worker_disable_rate_limits = True
    task_serializer = "pickle"
    accept_content = ["json", "pickle"]

    task_default_queue = 'default'
    task_default_exchange = 'default'
    task_default_routing_key = 'default'



    ```
    

- 定义异步任务
    
    
    ```python
    @app.task
    def export_users():
        users = Users.objects.all().values("id", "email", "first_name", "last_name")
        with open('users.csv', 'w', encoding='utf-8') as f:
            user_file = csv.DictWriter(f, fieldnames=["id", "email", "first_name", "last_name"])
            user_file.writeheader()
            for user in users:
                user_file.writerow(user)
        return
    ```
    
- 启动celery worker
    
    ```python
    celery -A tasks.task worker -Q default --loglevel=debug
    ```
    

