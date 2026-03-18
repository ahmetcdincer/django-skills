# Django Signals - Best practices for using Django's signal dispatcher including built-in and custom signals.

## pre_save / post_save

**Wrong:**
```python
# signals.py — connected but not imported, so it never fires
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import User

@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

# Signal defined but never connected because no one imports signals.py
```

**Correct:**
```python
# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.conf import settings


@receiver(post_save, sender=settings.AUTH_USER_MODEL, dispatch_uid='create_user_profile')
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        from .models import Profile
        Profile.objects.create(user=instance)


# apps.py — import signals in ready()
from django.apps import AppConfig

class UsersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.users'

    def ready(self):
        import apps.users.signals  # noqa: F401
```

> **Why:** Signals must be imported to be connected. The `ready()` method in `AppConfig` is the canonical place. `dispatch_uid` prevents duplicate connections.

## pre_delete / post_delete

**Wrong:**
```python
from django.db.models.signals import post_delete
from django.dispatch import receiver
from .models import Document

@receiver(post_delete, sender=Document)
def delete_file(sender, instance, **kwargs):
    instance.file.delete()
    # This also fires during queryset.delete() — watch out for bulk deletes
    # Also fires during cascade deletes — may error if file already gone
```

**Correct:**
```python
from django.db.models.signals import post_delete, pre_delete
from django.dispatch import receiver
from .models import Document


@receiver(post_delete, sender=Document)
def delete_document_file(sender, instance, **kwargs):
    if instance.file:
        instance.file.delete(save=False)  # save=False because instance is already deleted


@receiver(pre_delete, sender=Document)
def log_document_deletion(sender, instance, **kwargs):
    import logging
    logger = logging.getLogger(__name__)
    logger.info(f'Deleting document: {instance.pk} — {instance.title}')
```

> **Why:** `post_delete` runs after the DB row is gone — use it for file cleanup. `pre_delete` runs before deletion — use it for logging. Always use `save=False` in post_delete to avoid saving a deleted instance.

## m2m_changed

**Wrong:**
```python
# Trying to use post_save to detect ManyToMany changes
from django.db.models.signals import post_save
from .models import Article

@receiver(post_save, sender=Article)
def handle_tags(sender, instance, **kwargs):
    # M2M changes don't trigger post_save — this never fires for tag changes
    print(instance.tags.all())
```

**Correct:**
```python
from django.db.models.signals import m2m_changed
from django.dispatch import receiver
from .models import Article


@receiver(m2m_changed, sender=Article.tags.through)
def handle_tag_changes(sender, instance, action, pk_set, **kwargs):
    if action == 'post_add':
        # Tags were added
        from .tasks import reindex_article
        reindex_article.delay(instance.pk)
    elif action == 'post_remove':
        # Tags were removed
        pass
    elif action == 'post_clear':
        # All tags were removed
        pass
```

> **Why:** M2M changes fire `m2m_changed`, not `post_save`. The `action` parameter tells you what happened: `pre_add`, `post_add`, `pre_remove`, `post_remove`, `pre_clear`, `post_clear`.

## request_started / request_finished

**Wrong:**
```python
# Using request signals for per-request logic when middleware is more appropriate
from django.core.signals import request_started

@receiver(request_started)
def on_request(sender, **kwargs):
    # request_started doesn't give you the request object!
    # You can't access request.user, request.path, etc. here
    print('Request started')
```

**Correct:**
```python
# request_started/finished are for low-level hooks (connection management, etc.)
from django.core.signals import request_finished
from django.dispatch import receiver


@receiver(request_finished)
def cleanup_temp_files(sender, **kwargs):
    import tempfile
    import os
    # Clean up any temp files created during request processing
    temp_dir = tempfile.gettempdir()
    for f in os.listdir(temp_dir):
        if f.startswith('django_upload_'):
            os.remove(os.path.join(temp_dir, f))

# For per-request logic with access to request/response, use middleware instead
```

> **Why:** `request_started`/`request_finished` don't provide the request object. Use middleware for request/response processing. Use these signals only for low-level hooks like resource cleanup.

## Custom Signals

**Wrong:**
```python
from django.dispatch import Signal

# Old-style with providing_args (deprecated in Django 4.0, removed in 5.0)
order_completed = Signal(providing_args=['order', 'user'])
```

**Correct:**
```python
# signals.py
from django.dispatch import Signal

order_completed = Signal()  # No providing_args needed
payment_failed = Signal()

# Sending the signal
from .signals import order_completed

def complete_order(order):
    order.status = 'completed'
    order.save()
    order_completed.send(sender=order.__class__, order=order, user=order.user)


# Receiving the signal
from .signals import order_completed

@receiver(order_completed)
def send_order_confirmation(sender, order, user, **kwargs):
    from .tasks import send_confirmation_email
    send_confirmation_email.delay(order.pk, user.email)
```

> **Why:** Custom signals decouple components — the order module doesn't need to know about email sending. `providing_args` was removed in Django 5.0; just document expected kwargs.

## When to Avoid Signals

**Wrong:**
```python
# Using signals for tightly coupled logic that should be explicit
@receiver(post_save, sender=Order)
def recalculate_total(sender, instance, **kwargs):
    instance.total = sum(item.price for item in instance.items.all())
    instance.save()  # Infinite loop! post_save triggers post_save

@receiver(post_save, sender=Order)
def send_email(sender, instance, created, **kwargs):
    if created:
        send_mail(...)  # Slows down every Order.save() call, even in tests
```

**Correct:**
```python
# Use explicit method calls instead of signals for tightly coupled logic
class Order(models.Model):
    def recalculate_total(self):
        self.total = self.items.aggregate(total=Sum('price'))['total'] or 0
        self.save(update_fields=['total'])

    def complete(self):
        self.status = 'completed'
        self.recalculate_total()
        # Explicit — you can see exactly what happens
        from .tasks import send_order_confirmation
        send_order_confirmation.delay(self.pk)
```

> **Why:** Signals are invisible control flow — hard to debug and test. Use them for decoupled cross-app communication. Use explicit method calls for tightly coupled logic within the same app.

## Importing Signals in apps.py ready()

**Wrong:**
```python
# Importing signals at module level in models.py
# models.py
from . import signals  # Circular import risk, loaded too early
```

**Correct:**
```python
# apps.py
from django.apps import AppConfig


class OrdersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.orders'

    def ready(self):
        from . import signals  # noqa: F401
        # Signals are now connected after all apps are loaded
```

> **Why:** `ready()` runs after all apps are loaded, avoiding circular imports. This is the Django-recommended way to connect signals.
