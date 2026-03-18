# Background task best practices covering Celery setup, task definition, retries, routing, periodic tasks, Django-Q, Huey, Django 6.0 built-in tasks, and idempotency.

## Celery Setup and Configuration

### Proper Celery project structure and settings integration

**Wrong:**
```python
# celery.py in the wrong location, missing autodiscover
from celery import Celery
app = Celery('myproject')
# Forgetting to configure the broker, or hardcoding credentials
app.conf.broker_url = 'redis://localhost:6379/0'
```

**Correct:**
```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')

app = Celery('config')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()


# config/__init__.py
from .celery import app as celery_app
__all__ = ('celery_app',)


# settings.py
CELERY_BROKER_URL = os.environ.get('CELERY_BROKER_URL', 'redis://127.0.0.1:6379/0')
CELERY_RESULT_BACKEND = os.environ.get('CELERY_RESULT_BACKEND', 'redis://127.0.0.1:6379/1')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
```

> **Why:** `config_from_object` with `namespace='CELERY'` reads all `CELERY_*` settings from Django settings. `autodiscover_tasks` finds `tasks.py` in each installed app automatically.

## Task Definition

### Using shared_task with JSON-serializable arguments

**Wrong:**
```python
from celery import Celery
app = Celery()

@app.task
def send_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # Passing the whole user object would fail — objects aren't JSON-serializable
```

**Correct:**
```python
from celery import shared_task


@shared_task(bind=True, name='orders.send_confirmation')
def send_order_confirmation(self, order_id):
    from apps.orders.models import Order
    from django.core.mail import send_mail

    order = Order.objects.select_related('user').get(pk=order_id)
    send_mail(
        subject=f'Order #{order.pk} Confirmation',
        message=f'Your order total: ${order.total}',
        from_email='shop@example.com',
        recipient_list=[order.user.email],
    )
```

> **Why:** Use `@shared_task` to avoid importing the Celery app directly. Pass IDs, not objects — task arguments must be JSON-serializable. `bind=True` gives access to `self` for retries.

## Task Retry

### Exponential backoff with autoretry

**Wrong:**
```python
from celery import shared_task

@shared_task
def charge_payment(order_id):
    # No retry logic — if payment gateway is temporarily down, the charge is lost
    import stripe
    order = Order.objects.get(pk=order_id)
    stripe.Charge.create(amount=int(order.total * 100), currency='usd')
```

**Correct:**
```python
from celery import shared_task


@shared_task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
    retry_backoff_max=600,
    max_retries=5,
    retry_jitter=True,
)
def charge_payment(self, order_id):
    from apps.orders.models import Order
    import stripe

    order = Order.objects.get(pk=order_id)
    try:
        charge = stripe.Charge.create(
            amount=int(order.total * 100),
            currency='usd',
            source=order.payment_token,
        )
        order.status = 'paid'
        order.charge_id = charge.id
        order.save(update_fields=['status', 'charge_id'])
    except stripe.error.CardError:
        order.status = 'payment_failed'
        order.save(update_fields=['status'])
        # Don't retry card errors — they're permanent failures
```

> **Why:** `autoretry_for` retries on specific exceptions. `retry_backoff=True` uses exponential backoff. `retry_jitter=True` adds randomness to prevent thundering herd.

## Task Routing

### Separate queues for different task priorities

**Wrong:**
```python
# All tasks on the same queue — a slow report blocks email sending
@shared_task
def send_email(user_id): ...

@shared_task
def generate_large_report(report_id): ...  # Takes 30 minutes, blocks the queue
```

**Correct:**
```python
from celery import shared_task

@shared_task(queue='emails')
def send_email(user_id): ...

@shared_task(queue='reports')
def generate_large_report(report_id): ...

# settings.py
CELERY_TASK_ROUTES = {
    'apps.notifications.tasks.*': {'queue': 'emails'},
    'apps.reports.tasks.*': {'queue': 'reports'},
}

# Start workers per queue:
# celery -A config worker -Q emails -c 4
# celery -A config worker -Q reports -c 2
```

> **Why:** Route tasks to separate queues so slow tasks don't block fast ones. Run dedicated workers per queue with appropriate concurrency.

## Periodic Tasks

### Celery Beat for scheduled tasks

**Wrong:**
```python
# Using a cron job that calls manage.py — outside Celery's control
# crontab: */5 * * * * cd /app && python manage.py cleanup_expired
```

**Correct:**
```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'cleanup-expired-orders': {
        'task': 'orders.cleanup_expired',
        'schedule': crontab(minute='*/30'),  # Every 30 minutes
    },
    'send-daily-digest': {
        'task': 'notifications.send_digest',
        'schedule': crontab(hour=9, minute=0),  # Daily at 9 AM
    },
    'weekly-report': {
        'task': 'reports.generate_weekly',
        'schedule': crontab(hour=0, minute=0, day_of_week=1),  # Monday midnight
    },
}

# Start beat: celery -A config beat --loglevel=info
```

> **Why:** Celery Beat manages periodic tasks within the Celery ecosystem — monitoring, retries, and logging work the same as regular tasks. Use `django-celery-beat` if you need DB-managed schedules.

## Django-Q Alternative

### Simpler background tasks without separate broker config

**Wrong:**
```python
# Using threading for background tasks in Django
import threading

def my_view(request):
    t = threading.Thread(target=send_email, args=(user.id,))
    t.start()  # No retry, no monitoring, lost if server restarts
    return HttpResponse('ok')
```

**Correct:**
```python
# pip install django-q2

# settings.py
Q_CLUSTER = {
    'name': 'myproject',
    'workers': 4,
    'recycle': 500,
    'timeout': 60,
    'django_redis': 'default',  # Uses Django's cache backend
}

INSTALLED_APPS = [..., 'django_q']

# Usage
from django_q.tasks import async_task, schedule

# Fire and forget
async_task('myapp.tasks.send_email', user.id)

# With callback
async_task('myapp.tasks.process_order', order.id,
           hook='myapp.tasks.order_processed_callback')
```

> **Why:** Django-Q (django-q2 fork) is simpler than Celery — no separate broker config needed if you use Redis as Django's cache. Good for projects that don't need Celery's full feature set.

## Huey Lightweight Alternative

### Minimal background task setup for small projects

**Wrong:**
```python
# Using Celery for a simple project with 2-3 background tasks
# Celery + Redis + Beat + Flower = complex infrastructure for a small app
```

**Correct:**
```python
# pip install huey

# settings.py
INSTALLED_APPS = [..., 'huey.contrib.djhuey']

HUEY = {
    'huey_class': 'huey.RedisHuey',
    'name': 'myproject',
    'immediate': False,  # Set True for development (runs tasks synchronously)
}

# tasks.py
from huey.contrib.djhuey import task, periodic_task, crontab

@task()
def send_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # send email...

@periodic_task(crontab(minute='0', hour='*/6'))
def cleanup():
    # runs every 6 hours
    ...

# Start: python manage.py run_huey
```

> **Why:** Huey is a lightweight alternative to Celery — single dependency, simple config. `immediate=True` in development runs tasks synchronously. Great for small-to-medium projects.

## Django 6.0 Built-in Tasks

### Using Django's native task system for simple use cases

**Wrong:**
```python
# Django 6.0+ — still using Celery just for simple fire-and-forget tasks
# when the built-in task system would suffice
```

**Correct:**
```python
# Django 6.0+ — built-in background tasks
# settings.py
TASKS = {
    'default': {
        'BACKEND': 'django.tasks.backends.database.DatabaseBackend',
    }
}

# tasks.py
from django.tasks import task

@task()
def send_welcome_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # send email...

# Usage in views
from .tasks import send_welcome_email

def register(request):
    user = User.objects.create(...)
    send_welcome_email.enqueue(user.id)
    return redirect('home')

# Run worker: python manage.py taskworker
```

> **Why:** Django 6.0+ includes a built-in task system that eliminates the need for Celery in simple cases. Database backend requires no extra infrastructure. Use Celery when you need advanced features like routing and priorities.

## Task Idempotency

### Ensuring tasks are safe to run more than once

**Wrong:**
```python
from celery import shared_task

@shared_task
def charge_order(order_id):
    order = Order.objects.get(pk=order_id)
    # If this task runs twice (retry, duplicate message), customer is charged twice!
    payment_gateway.charge(order.total)
    order.status = 'paid'
    order.save()
```

**Correct:**
```python
from celery import shared_task


@shared_task(bind=True)
def charge_order(self, order_id):
    from apps.orders.models import Order

    order = Order.objects.select_for_update().get(pk=order_id)

    # Idempotency check — skip if already processed
    if order.status == 'paid':
        return

    if order.charge_id:
        # Already charged but status wasn't updated — verify with gateway
        return

    charge = payment_gateway.charge(order.total, idempotency_key=f'order-{order.id}')
    order.charge_id = charge.id
    order.status = 'paid'
    order.save(update_fields=['charge_id', 'status'])
```

> **Why:** Tasks may run more than once due to retries, broker redelivery, or duplicate sends. Use idempotency keys and status checks to ensure safe re-execution.
