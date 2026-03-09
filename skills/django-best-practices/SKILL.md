---
name: django-best-practices
description: A senior Django developer's knowledge base for Claude Code. Wrong vs correct examples across 27 topics — from ORM to deployment.
compatibility: Designed for Claude Code. Requires Python and Django.
allowed-tools: Read Write Edit Bash(python:*) Bash(pip:*)
---

# Django Best Practices

## 1. Django Core

### Project Structure

❌ **Wrong:**
```python
# Everything in one directory, no separation
myproject/
    models.py      # 5000 lines, all models here
    views.py       # 3000 lines, all views here
    urls.py
    settings.py
    manage.py
```

✅ **Correct:**
```python
# config/ holds project-level settings, each app is focused
myproject/
    config/
        __init__.py
        settings/
            __init__.py
            base.py
            local.py
            production.py
        urls.py
        wsgi.py
        asgi.py
    apps/
        users/
            __init__.py
            models.py
            views.py
            urls.py
            admin.py
            tests/
                __init__.py
                test_models.py
                test_views.py
        orders/
            ...
    manage.py
    requirements/
        base.txt
        local.txt
        production.txt
```

> 💡 **Why:** Separating config from apps keeps the project navigable as it grows. Each app owns its own domain, making it testable and reusable independently.

### App Structure

❌ **Wrong:**
```python
# One giant app doing everything
# shop/models.py
from django.db import models

class User(models.Model): ...
class Product(models.Model): ...
class Order(models.Model): ...
class Payment(models.Model): ...
class Review(models.Model): ...
class Notification(models.Model): ...
class BlogPost(models.Model): ...
```

✅ **Correct:**
```python
# Small, focused apps with clear boundaries
# users/models.py
from django.db import models

class User(models.Model): ...

# products/models.py
from django.db import models

class Product(models.Model): ...

# orders/models.py
from django.db import models
from django.conf import settings

class Order(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    ...
```

> 💡 **Why:** Small apps are easier to test, reuse, and reason about. When one app has 20+ models, it's a sign you need to split it.

### Settings Configuration

❌ **Wrong:**
```python
# settings.py — everything in one file, secrets hardcoded
SECRET_KEY = 'django-insecure-abc123realkey'
DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PASSWORD': 'mydbpassword',
    }
}
```

✅ **Correct:**
```python
# config/settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
]

# config/settings/local.py
from .base import *  # noqa: F401,F403

DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# config/settings/production.py
from .base import *  # noqa: F401,F403

DEBUG = False
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.environ['DB_HOST'],
    }
}
```

> 💡 **Why:** Splitting settings per environment prevents accidental DEBUG=True in production and keeps secrets out of version control.

### URL Configuration

❌ **Wrong:**
```python
# config/urls.py — all URLs in one flat file
from django.urls import path
from users.views import login_view, profile_view
from products.views import product_list, product_detail
from orders.views import order_list

urlpatterns = [
    path('login/', login_view),
    path('profile/', profile_view),
    path('products/', product_list),
    path('products/<int:pk>/', product_detail),
    path('orders/', order_list),
]
```

✅ **Correct:**
```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('apps.users.urls', namespace='users')),
    path('products/', include('apps.products.urls', namespace='products')),
    path('orders/', include('apps.orders.urls', namespace='orders')),
]

# apps/products/urls.py
from django.urls import path
from . import views

app_name = 'products'

urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
]
```

> 💡 **Why:** Namespaced includes keep URL definitions close to their apps and prevent name collisions. `reverse('products:detail', kwargs={'pk': 1})` is unambiguous.

### WSGI and ASGI

❌ **Wrong:**
```python
# Using WSGI when you need WebSocket or async support
# wsgi.py serving a project with Django Channels
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
application = get_wsgi_application()
# Then wondering why WebSocket connections fail
```

✅ **Correct:**
```python
# wsgi.py — for traditional HTTP-only projects
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
application = get_wsgi_application()

# asgi.py — when you need async views, WebSocket, or Channels
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
application = get_asgi_application()

# With Channels:
# import os
# from channels.routing import ProtocolTypeRouter, URLRouter
# from channels.auth import AuthMiddlewareStack
# from django.core.asgi import get_asgi_application
# os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
# application = ProtocolTypeRouter({
#     "http": get_asgi_application(),
#     "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
# })
```

> 💡 **Why:** Use WSGI with Gunicorn for standard HTTP apps. Switch to ASGI with Daphne/Uvicorn only when you need async views, WebSockets, or long-lived connections.

### Custom Management Commands

❌ **Wrong:**
```python
# Putting scripts in random .py files and running them with exec
# scripts/cleanup.py
import django
django.setup()
from myapp.models import Order
Order.objects.filter(status='expired').delete()
# Run with: python scripts/cleanup.py
```

✅ **Correct:**
```python
# apps/orders/management/commands/cleanup_expired_orders.py
from django.core.management.base import BaseCommand
from apps.orders.models import Order


class Command(BaseCommand):
    help = 'Delete orders that have been expired for more than 30 days'

    def add_arguments(self, parser):
        parser.add_argument(
            '--dry-run',
            action='store_true',
            help='Show what would be deleted without deleting',
        )

    def handle(self, *args, **options):
        from django.utils import timezone
        cutoff = timezone.now() - timezone.timedelta(days=30)
        expired = Order.objects.filter(status='expired', updated_at__lt=cutoff)
        count = expired.count()

        if options['dry_run']:
            self.stdout.write(f'Would delete {count} expired orders')
            return

        expired.delete()
        self.stdout.write(self.style.SUCCESS(f'Deleted {count} expired orders'))
```

> 💡 **Why:** Management commands integrate with Django's setup, support arguments, and can be scheduled via cron or Celery beat. Random scripts break when settings paths change.

## 2. Models (ORM)

### Model Definition

❌ **Wrong:**
```python
from django.db import models

class product(models.Model):
    n = models.CharField(max_length=100)
    d = models.CharField(max_length=5000)
    p = models.FloatField()
    active = models.CharField(max_length=5, default='yes')
```

✅ **Correct:**
```python
from django.db import models
from django.utils.translation import gettext_lazy as _


class Product(models.Model):
    name = models.CharField(_('name'), max_length=255)
    description = models.TextField(_('description'), blank=True)
    price = models.DecimalField(_('price'), max_digits=10, decimal_places=2)
    is_active = models.BooleanField(_('active'), default=True)

    class Meta:
        verbose_name = _('product')
        verbose_name_plural = _('products')

    def __str__(self):
        return self.name
```

> 💡 **Why:** Use descriptive field names, correct field types (DecimalField for money, BooleanField for flags, TextField for long text), and verbose_name for admin/i18n support.

### Field Types

❌ **Wrong:**
```python
from django.db import models

class Event(models.Model):
    title = models.TextField()  # Short text in TextField
    price = models.FloatField()  # Float for money — rounding errors
    event_date = models.DateTimeField()  # Only need date, not time
    email = models.CharField(max_length=255)  # No validation
```

✅ **Correct:**
```python
from django.db import models


class Event(models.Model):
    title = models.CharField(max_length=255)  # CharField for short text
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Exact precision
    event_date = models.DateField()  # DateField when time isn't needed
    email = models.EmailField()  # Built-in email validation
```

> 💡 **Why:** CharField has max_length enforced at DB level. FloatField causes rounding errors with currency. DateField vs DateTimeField — pick what you actually need.

### ForeignKey

❌ **Wrong:**
```python
from django.db import models

class Comment(models.Model):
    post = models.ForeignKey('Post', on_delete=models.CASCADE)
    # No related_name — default is comment_set
    # No db_index consideration
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    # Hardcoded auth.User — breaks with custom user model
```

✅ **Correct:**
```python
from django.conf import settings
from django.db import models


class Comment(models.Model):
    post = models.ForeignKey(
        'blog.Post',
        on_delete=models.CASCADE,
        related_name='comments',
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='comments',
    )
```

> 💡 **Why:** Always use `settings.AUTH_USER_MODEL` for user references. Explicit `related_name` makes reverse queries readable: `post.comments.all()` instead of `post.comment_set.all()`.

### ManyToMany with Through Model

❌ **Wrong:**
```python
from django.db import models

class Project(models.Model):
    name = models.CharField(max_length=255)
    members = models.ManyToManyField('auth.User')
    # No way to store role, date_joined, or any extra data
```

✅ **Correct:**
```python
from django.conf import settings
from django.db import models


class Project(models.Model):
    name = models.CharField(max_length=255)
    members = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        through='ProjectMembership',
        related_name='projects',
    )


class ProjectMembership(models.Model):
    class Role(models.TextChoices):
        OWNER = 'owner', 'Owner'
        EDITOR = 'editor', 'Editor'
        VIEWER = 'viewer', 'Viewer'

    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    project = models.ForeignKey(Project, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=Role.choices, default=Role.VIEWER)
    joined_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'project')
```

> 💡 **Why:** Use a through model when you need metadata on the relationship (role, timestamps, permissions). You almost always need it eventually — add it early.

### OneToOneField

❌ **Wrong:**
```python
from django.conf import settings
from django.db import models

class Profile(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    # ForeignKey allows multiple profiles per user — probably a bug
    bio = models.TextField(blank=True)
```

✅ **Correct:**
```python
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
```

> 💡 **Why:** OneToOneField enforces the one-to-one constraint at the database level. Access is cleaner too: `user.profile` instead of `user.profile_set.first()`.

### Model Meta

❌ **Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
    # No ordering, no indexes, no constraints
```

✅ **Correct:**
```python
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['slug']),
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['slug'],
                condition=models.Q(status='published'),
                name='unique_published_slug',
            ),
        ]
        verbose_name = 'article'
        verbose_name_plural = 'articles'
```

> 💡 **Why:** Indexes speed up queries you run often. Constraints enforce data integrity at the DB level. Default ordering saves repeating `.order_by()` everywhere.

### Model Methods

❌ **Wrong:**
```python
from django.db import models

class Order(models.Model):
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)
    # No __str__ — admin shows "Order object (1)"
    # No get_absolute_url — templates hardcode URLs
```

✅ **Correct:**
```python
from django.db import models
from django.urls import reverse


class Order(models.Model):
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)

    def __str__(self):
        return f'Order #{self.pk} — {self.total}'

    def get_absolute_url(self):
        return reverse('orders:detail', kwargs={'pk': self.pk})

    @property
    def is_paid(self):
        return self.status == 'paid'

    def mark_as_shipped(self):
        if self.status != 'paid':
            raise ValueError('Only paid orders can be shipped')
        self.status = 'shipped'
        self.save(update_fields=['status'])
```

> 💡 **Why:** `__str__` makes admin and debugging readable. `get_absolute_url` is used by Django's redirect shortcuts and admin. Methods encapsulate business logic on the model.

### Model Managers

❌ **Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    is_published = models.BooleanField(default=False)

# In views — filtering repeated everywhere
# Article.objects.filter(is_published=True)
# Article.objects.filter(is_published=True, category='tech')
```

✅ **Correct:**
```python
from django.db import models


class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_published=True)


class Article(models.Model):
    title = models.CharField(max_length=255)
    is_published = models.BooleanField(default=False)

    objects = models.Manager()  # Default manager
    published = PublishedManager()  # Custom manager

# Usage: Article.published.all()
# Usage: Article.published.filter(category='tech')
```

> 💡 **Why:** Custom managers encapsulate common filters, keeping views DRY. Always keep `objects` as the default manager to avoid surprises with admin and related lookups.

### Abstract Models

❌ **Wrong:**
```python
from django.db import models

class Article(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    title = models.CharField(max_length=255)

class Comment(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)  # Duplicated
    updated_at = models.DateTimeField(auto_now=True)       # Duplicated
    body = models.TextField()
```

✅ **Correct:**
```python
from django.db import models


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Article(TimeStampedModel):
    title = models.CharField(max_length=255)


class Comment(TimeStampedModel):
    body = models.TextField()
```

> 💡 **Why:** Abstract models eliminate field duplication across models. `abstract = True` means no database table is created for the base class — fields are added to each child table.

### Proxy Models

❌ **Wrong:**
```python
from django.db import models

class Order(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)

# Separate model with duplicate fields just for a different admin view
class PendingOrder(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)
```

✅ **Correct:**
```python
from django.db import models


class Order(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2)


class PendingOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status='pending')


class PendingOrder(Order):
    objects = PendingOrderManager()

    class Meta:
        proxy = True
        verbose_name = 'pending order'
        verbose_name_plural = 'pending orders'
```

> 💡 **Why:** Proxy models share the same database table but allow different Python behavior, managers, and admin registrations. No data duplication, no migration overhead.

### Custom QuerySet Methods

❌ **Wrong:**
```python
from django.db import models

# Filtering logic scattered across views
# views.py
def active_premium_users(request):
    users = User.objects.filter(is_active=True, plan='premium')
    ...

def dashboard(request):
    users = User.objects.filter(is_active=True, plan='premium')  # Duplicated
    ...
```

✅ **Correct:**
```python
from django.db import models


class UserQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def premium(self):
        return self.filter(plan='premium')

    def with_order_count(self):
        return self.annotate(order_count=models.Count('orders'))


class User(models.Model):
    is_active = models.BooleanField(default=True)
    plan = models.CharField(max_length=20)

    objects = UserQuerySet.as_manager()

# Usage — chainable: User.objects.active().premium().with_order_count()
```

> 💡 **Why:** QuerySet methods are chainable and reusable. `as_manager()` turns a QuerySet into a manager, giving you both custom methods and full QuerySet API.

## 3. Django ORM

### QuerySet Lazy Evaluation

❌ **Wrong:**
```python
from myapp.models import Product

# Hits the DB immediately — fetches ALL products into memory
all_products = list(Product.objects.all())
# Then filters in Python
cheap = [p for p in all_products if p.price < 10]
```

✅ **Correct:**
```python
from myapp.models import Product

# No DB hit yet — just builds the query
products = Product.objects.filter(price__lt=10)
# Still no DB hit
products = products.select_related('category')
# DB hit happens here — when you iterate, slice, or evaluate
for product in products:
    print(product.name)
```

> 💡 **Why:** QuerySets are lazy — they don't hit the database until evaluated (iteration, slicing, `list()`, `len()`, `bool()`). Chain filters freely without performance cost.

### filter() vs get()

❌ **Wrong:**
```python
from myapp.models import User

# get() without exception handling — crashes on missing or duplicate rows
user = User.objects.get(email=email)

# filter() when you expect exactly one result
users = User.objects.filter(pk=user_id)
user = users[0]  # IndexError if empty
```

✅ **Correct:**
```python
from django.shortcuts import get_object_or_404
from myapp.models import User

# In views — returns 404 if not found
user = get_object_or_404(User, pk=user_id)

# In business logic — handle the exception
from django.core.exceptions import ObjectDoesNotExist

try:
    user = User.objects.get(email=email)
except User.DoesNotExist:
    user = None

# Or use filter + first when "not found" is a normal case
user = User.objects.filter(email=email).first()  # Returns None if not found
```

> 💡 **Why:** `get()` raises `DoesNotExist` or `MultipleObjectsReturned`. Use `get_object_or_404` in views, `.first()` when absence is expected, and explicit try/except in services.

### exclude() Usage

❌ **Wrong:**
```python
from myapp.models import Product

# Filtering in Python instead of at the database level
products = Product.objects.all()
active_products = [p for p in products if p.status != 'archived']
```

✅ **Correct:**
```python
from myapp.models import Product

# Let the database do the filtering
active_products = Product.objects.exclude(status='archived')

# Combine with filter
featured = Product.objects.filter(is_featured=True).exclude(status='archived')
```

> 💡 **Why:** `exclude()` generates a SQL WHERE NOT clause, keeping filtering in the database where it's fast and memory-efficient.

### Q Objects

❌ **Wrong:**
```python
from myapp.models import Product

# Cannot do OR with chained filter calls
# This is AND, not OR:
products = Product.objects.filter(category='electronics').filter(category='books')
```

✅ **Correct:**
```python
from django.db.models import Q
from myapp.models import Product

# OR query
products = Product.objects.filter(
    Q(category='electronics') | Q(category='books')
)

# Complex: (category=electronics AND price<100) OR is_featured=True
products = Product.objects.filter(
    (Q(category='electronics') & Q(price__lt=100)) | Q(is_featured=True)
)

# NOT
products = Product.objects.filter(~Q(status='archived'))
```

> 💡 **Why:** Q objects enable OR, AND, and NOT combinations that chained `.filter()` calls cannot express. Use `|` for OR, `&` for AND, `~` for NOT.

### F Expressions

❌ **Wrong:**
```python
from myapp.models import Product

# Race condition — reads, modifies in Python, writes back
product = Product.objects.get(pk=1)
product.view_count = product.view_count + 1
product.save()
```

✅ **Correct:**
```python
from django.db.models import F
from myapp.models import Product

# Atomic update — database does the increment
Product.objects.filter(pk=1).update(view_count=F('view_count') + 1)

# Compare fields against each other
# Products where sale_price < original_price
discounted = Product.objects.filter(sale_price__lt=F('original_price'))
```

> 💡 **Why:** F expressions perform operations at the database level, avoiding race conditions from concurrent requests. Two users incrementing simultaneously won't lose a count.

### Aggregation

❌ **Wrong:**
```python
from myapp.models import Order

# Aggregating in Python — loads all rows into memory
orders = Order.objects.all()
total = sum(o.total for o in orders)
average = total / len(orders)
```

✅ **Correct:**
```python
from django.db.models import Avg, Count, Sum, Max, Min
from myapp.models import Order

# aggregate() returns a dict — single result for the whole queryset
result = Order.objects.aggregate(
    total_revenue=Sum('total'),
    avg_order=Avg('total'),
    order_count=Count('id'),
)
# result = {'total_revenue': Decimal('50000.00'), 'avg_order': ..., 'order_count': 500}

# annotate() adds a computed column to each row
from myapp.models import Customer
customers = Customer.objects.annotate(
    total_spent=Sum('orders__total'),
    order_count=Count('orders'),
).filter(order_count__gte=5)
```

> 💡 **Why:** `aggregate()` produces a single summary dict. `annotate()` adds computed columns per row. Both execute in SQL — no Python-side iteration needed.

### Annotation with Subquery

❌ **Wrong:**
```python
from myapp.models import Author, Book

# N+1 — one query per author to get their latest book
for author in Author.objects.all():
    latest = Book.objects.filter(author=author).order_by('-published').first()
    print(author.name, latest.title if latest else 'No books')
```

✅ **Correct:**
```python
from django.db.models import OuterRef, Subquery
from myapp.models import Author, Book

latest_book = Book.objects.filter(
    author=OuterRef('pk')
).order_by('-published')

authors = Author.objects.annotate(
    latest_book_title=Subquery(latest_book.values('title')[:1])
)
for author in authors:
    print(author.name, author.latest_book_title)
```

> 💡 **Why:** Subquery runs a correlated subquery inside a single SQL statement — one query instead of N+1.

### Subquery and Exists

❌ **Wrong:**
```python
from myapp.models import User, Order

# Fetches all orders just to check existence
users_with_orders = []
for user in User.objects.all():
    if Order.objects.filter(user=user).count() > 0:
        users_with_orders.append(user)
```

✅ **Correct:**
```python
from django.db.models import Exists, OuterRef
from myapp.models import User, Order

# Exists — efficient boolean subquery
users_with_orders = User.objects.filter(
    Exists(Order.objects.filter(user=OuterRef('pk')))
)

# Annotate with existence flag
users = User.objects.annotate(
    has_orders=Exists(Order.objects.filter(user=OuterRef('pk')))
)
```

> 💡 **Why:** `Exists` generates a SQL EXISTS subquery, which stops scanning as soon as it finds one match — far more efficient than COUNT.

### Raw SQL

❌ **Wrong:**
```python
from django.db import connection

# SQL injection vulnerability — string formatting user input
def search_products(query):
    cursor = connection.cursor()
    cursor.execute(f"SELECT * FROM products WHERE name LIKE '%{query}%'")
    return cursor.fetchall()
```

✅ **Correct:**
```python
from django.db import connection
from myapp.models import Product

# Parameterized raw SQL — safe from injection
def search_products(query):
    return Product.objects.raw(
        "SELECT * FROM products_product WHERE name LIKE %s",
        [f'%{query}%']
    )

# Or with connection.cursor for non-model queries
def get_stats():
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT category, COUNT(*) FROM products_product GROUP BY category"
        )
        return cursor.fetchall()
```

> 💡 **Why:** Never use string formatting with SQL. Always pass parameters separately so the database driver handles escaping. This is your primary defense against SQL injection.

### select_related

❌ **Wrong:**
```python
from myapp.models import Comment

# N+1 problem — each comment triggers a query for post and user
comments = Comment.objects.all()
for comment in comments:
    print(comment.post.title, comment.user.username)
# This generates 1 + N*2 queries
```

✅ **Correct:**
```python
from myapp.models import Comment

# Single query with JOINs
comments = Comment.objects.select_related('post', 'user').all()
for comment in comments:
    print(comment.post.title, comment.user.username)
# This generates 1 query with JOINs
```

> 💡 **Why:** `select_related` uses SQL JOINs to fetch related ForeignKey/OneToOne objects in a single query. Use it whenever you access FK relations in a loop.

### prefetch_related

❌ **Wrong:**
```python
from myapp.models import Author

# N+1 — each author triggers a separate query for their books
authors = Author.objects.all()
for author in authors:
    for book in author.books.all():  # One query per author
        print(book.title)
```

✅ **Correct:**
```python
from django.db.models import Prefetch
from myapp.models import Author, Book

# Two queries total: one for authors, one for all related books
authors = Author.objects.prefetch_related('books')

# Custom Prefetch for filtering or ordering the related set
authors = Author.objects.prefetch_related(
    Prefetch(
        'books',
        queryset=Book.objects.filter(is_published=True).order_by('-published'),
        to_attr='published_books',
    )
)
for author in authors:
    for book in author.published_books:
        print(book.title)
```

> 💡 **Why:** `prefetch_related` does a separate query for the related objects and joins them in Python. Use it for ManyToMany and reverse ForeignKey relations. `Prefetch` objects let you customize the related query.

### bulk_create and bulk_update

❌ **Wrong:**
```python
from myapp.models import LogEntry

# One INSERT per iteration — extremely slow for large batches
for data in large_dataset:
    LogEntry.objects.create(message=data['message'], level=data['level'])
```

✅ **Correct:**
```python
from myapp.models import LogEntry

# Single query with batch INSERT
entries = [
    LogEntry(message=data['message'], level=data['level'])
    for data in large_dataset
]
LogEntry.objects.bulk_create(entries, batch_size=1000)

# bulk_update for existing objects
products = list(Product.objects.filter(category='sale'))
for product in products:
    product.price = product.price * Decimal('0.9')
Product.objects.bulk_update(products, ['price'], batch_size=1000)
```

> 💡 **Why:** `bulk_create` and `bulk_update` reduce thousands of queries to a handful. Use `batch_size` to control memory usage and avoid exceeding DB parameter limits.

### Transactions

❌ **Wrong:**
```python
from myapp.models import Account

# No transaction — partial failure leaves inconsistent data
def transfer(from_id, to_id, amount):
    sender = Account.objects.get(pk=from_id)
    sender.balance -= amount
    sender.save()
    # If this crashes here, money is gone but not received
    receiver = Account.objects.get(pk=to_id)
    receiver.balance += amount
    receiver.save()
```

✅ **Correct:**
```python
from django.db import transaction
from django.db.models import F
from myapp.models import Account


def transfer(from_id, to_id, amount):
    with transaction.atomic():
        Account.objects.filter(pk=from_id).update(
            balance=F('balance') - amount
        )
        Account.objects.filter(pk=to_id).update(
            balance=F('balance') + amount
        )

# select_for_update for row-level locking
def transfer_safe(from_id, to_id, amount):
    with transaction.atomic():
        sender = Account.objects.select_for_update().get(pk=from_id)
        if sender.balance < amount:
            raise ValueError('Insufficient funds')
        Account.objects.filter(pk=from_id).update(balance=F('balance') - amount)
        Account.objects.filter(pk=to_id).update(balance=F('balance') + amount)
```

> 💡 **Why:** `atomic()` ensures all-or-nothing execution. `select_for_update()` locks rows to prevent concurrent modifications. Both are essential for financial operations.

### N+1 Problem Detection and Solution

❌ **Wrong:**
```python
from myapp.models import Order

# View that looks simple but generates 101 queries
def order_list(request):
    orders = Order.objects.all()[:100]
    data = []
    for order in orders:
        data.append({
            'id': order.id,
            'customer': order.customer.name,      # +1 query each
            'product_count': order.items.count(),  # +1 query each
        })
    return JsonResponse(data, safe=False)
```

✅ **Correct:**
```python
from django.db.models import Count
from myapp.models import Order


def order_list(request):
    orders = (
        Order.objects
        .select_related('customer')
        .annotate(product_count=Count('items'))
        [:100]
    )
    data = [
        {
            'id': order.id,
            'customer': order.customer.name,       # No extra query
            'product_count': order.product_count,   # No extra query
        }
        for order in orders
    ]
    return JsonResponse(data, safe=False)
```

> 💡 **Why:** Detect N+1 by counting queries with Django Debug Toolbar or `assertNumQueries`. Fix with `select_related` (FK/O2O), `prefetch_related` (M2M/reverse FK), and `annotate` (aggregates).

## 4. Migrations

### makemigrations Best Practices

❌ **Wrong:**
```python
# Making one giant migration after weeks of model changes
# 0001_initial.py — 50 operations, impossible to review or rollback
python manage.py makemigrations  # After changing 15 models at once
```

✅ **Correct:**
```bash
# Small, focused migrations after each logical change
python manage.py makemigrations users --name add_phone_field
python manage.py makemigrations orders --name add_shipping_address
```

```python
# Review the generated migration before committing
# users/migrations/0003_add_phone_field.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('users', '0002_add_email_verified'),
    ]
    operations = [
        migrations.AddField(
            model_name='user',
            name='phone',
            field=models.CharField(blank=True, max_length=20),
        ),
    ]
```

> 💡 **Why:** Small migrations are easy to review, debug, and rollback. Name them descriptively so you can understand the migration history at a glance.

### Safe Migration Usage in Production

❌ **Wrong:**
```bash
# Running migrate blindly on production without checking
python manage.py migrate --run-syncdb
# Or running migrations that aren't backward compatible while old code is still serving
```

✅ **Correct:**
```bash
# Always check what will run first
python manage.py showmigrations
python manage.py migrate --plan

# In production deployments:
# 1. Deploy new code (backward compatible)
# 2. Run migrations
# 3. Deploy code that uses new schema
```

```python
# Make AddField migrations backward-compatible
migrations.AddField(
    model_name='order',
    name='tracking_number',
    field=models.CharField(max_length=100, blank=True, default=''),
    # blank=True + default='' means old code won't break
)
```

> 💡 **Why:** Never assume migrations are safe to run blindly. Use `--plan` to preview, and ensure backward compatibility so old code keeps working during the migration window.

### Custom Migrations

❌ **Wrong:**
```python
# Running raw SQL without a reverse operation
from django.db import migrations

class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL("UPDATE users SET is_active = true WHERE last_login IS NOT NULL"),
        # No reverse — can't roll back this migration
    ]
```

✅ **Correct:**
```python
from django.db import migrations


def activate_recent_users(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.filter(last_login__isnull=False).update(is_active=True)


def deactivate_recent_users(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.filter(last_login__isnull=False).update(is_active=False)


class Migration(migrations.Migration):
    dependencies = [
        ('users', '0005_add_is_active'),
    ]
    operations = [
        migrations.RunPython(activate_recent_users, deactivate_recent_users),
    ]
```

> 💡 **Why:** Always provide both forward and reverse functions. Use `apps.get_model()` to access the model as it exists at that migration point, not the current model.

### Data Migrations

❌ **Wrong:**
```python
from django.db import migrations
from myapp.models import User  # Importing the current model directly

def populate_data(apps, schema_editor):
    # This breaks if the model changes later
    for user in User.objects.all():
        user.full_name = f'{user.first_name} {user.last_name}'
        user.save()
```

✅ **Correct:**
```python
from django.db import migrations


def populate_full_name(apps, schema_editor):
    User = apps.get_model('users', 'User')
    users = User.objects.all()
    for user in users:
        user.full_name = f'{user.first_name} {user.last_name}'
    User.objects.bulk_update(users, ['full_name'], batch_size=1000)


def reverse_full_name(apps, schema_editor):
    User = apps.get_model('users', 'User')
    User.objects.all().update(full_name='')


class Migration(migrations.Migration):
    dependencies = [
        ('users', '0006_add_full_name'),
    ]
    operations = [
        migrations.RunPython(populate_full_name, reverse_full_name),
    ]
```

> 💡 **Why:** Always use `apps.get_model()` in data migrations — it returns the model at that point in migration history. Direct imports reference the current model, which may differ.

### Migration Dependencies

❌ **Wrong:**
```python
# Migration in orders app references users app without declaring dependency
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('orders', '0001_initial'),
        # Missing: ('users', '0003_add_email') — this will fail if run out of order
    ]
    operations = [
        migrations.AddField(
            model_name='order',
            name='user_email',
            field=models.CharField(max_length=255),
        ),
    ]
```

✅ **Correct:**
```python
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [
        ('orders', '0001_initial'),
        ('users', '0003_add_email'),  # Explicit cross-app dependency
    ]
    operations = [
        migrations.AddField(
            model_name='order',
            name='user_email',
            field=models.CharField(max_length=255),
        ),
    ]
```

> 💡 **Why:** Django auto-detects most dependencies, but cross-app data migrations need explicit dependencies. Missing dependencies cause failures when migrations run in unexpected order.

### Migration Squashing

❌ **Wrong:**
```bash
# Deleting old migrations and recreating from scratch
rm myapp/migrations/0001_*.py myapp/migrations/0002_*.py
python manage.py makemigrations myapp
# Breaks all existing databases that already applied those migrations
```

✅ **Correct:**
```bash
# Squash a range of migrations safely
python manage.py squashmigrations myapp 0001 0010
# This creates 0001_squashed_0010_auto_*.py
# Old migrations are kept until all databases have migrated past them
# Then remove old migrations and update the squashed migration's replaces field
```

```python
# The squashed migration replaces the old ones
class Migration(migrations.Migration):
    replaces = [
        ('myapp', '0001_initial'),
        ('myapp', '0002_add_field'),
        # ...
        ('myapp', '0010_add_index'),
    ]
    operations = [
        # Consolidated operations
    ]
```

> 💡 **Why:** Squashing reduces migration count without breaking existing databases. The `replaces` attribute tells Django to skip individual migrations if the squashed one has been applied.

### Dangerous Migration Operations

❌ **Wrong:**
```python
# Dropping a column that's still referenced by running code
class Migration(migrations.Migration):
    operations = [
        migrations.RemoveField(model_name='user', name='legacy_email'),
        # If old code is still deployed, this crashes immediately
    ]
```

✅ **Correct:**
```python
# Step 1: Deploy code that stops reading/writing the field
# Step 2: Make the field nullable (safe, backward compatible)
class Migration(migrations.Migration):
    operations = [
        migrations.AlterField(
            model_name='user',
            name='legacy_email',
            field=models.CharField(max_length=255, null=True, blank=True),
        ),
    ]

# Step 3: After confirming no code uses it, remove the field
class Migration(migrations.Migration):
    operations = [
        migrations.RemoveField(model_name='user', name='legacy_email'),
    ]
```

> 💡 **Why:** Zero-downtime deployments require multi-step migrations: first make the field optional, deploy code that doesn't use it, then remove the column. Never drop a column that running code depends on.

## 5. Django Admin

### ModelAdmin Basics

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Product

admin.site.register(Product)
# Bare registration — no list display, no filters, no search
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import Product


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ('name', 'category', 'price', 'is_active', 'created_at')
    list_filter = ('category', 'is_active', 'created_at')
    search_fields = ('name', 'description')
    list_editable = ('is_active',)
    date_hierarchy = 'created_at'
    ordering = ('-created_at',)
```

> 💡 **Why:** A well-configured admin with `list_display`, `list_filter`, and `search_fields` turns the admin into a usable internal tool instead of a list of "Object (1)" links.

### Inline Models

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Order, OrderItem

# Separate admin pages for Order and OrderItem
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'customer')

@admin.register(OrderItem)
class OrderItemAdmin(admin.ModelAdmin):
    list_display = ('order', 'product', 'quantity')
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import Order, OrderItem


class OrderItemInline(admin.TabularInline):
    model = OrderItem
    extra = 1
    min_num = 1
    fields = ('product', 'quantity', 'unit_price')
    readonly_fields = ('unit_price',)


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'customer', 'total', 'status', 'created_at')
    inlines = [OrderItemInline]
```

> 💡 **Why:** Inlines let you edit parent and child models on the same page. Use `TabularInline` for compact rows, `StackedInline` for larger forms with more fields.

### Custom Admin Actions

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ('title', 'status')
    # No bulk actions — admin users must edit articles one by one to publish them
```

✅ **Correct:**
```python
from django.contrib import admin
from django.contrib import messages
from .models import Article


@admin.action(description='Mark selected articles as published')
def publish_articles(modeladmin, request, queryset):
    updated = queryset.update(status='published')
    messages.success(request, f'{updated} articles published.')


@admin.action(description='Mark selected articles as draft')
def unpublish_articles(modeladmin, request, queryset):
    updated = queryset.update(status='draft')
    messages.success(request, f'{updated} articles reverted to draft.')


@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ('title', 'status', 'created_at')
    actions = [publish_articles, unpublish_articles]
```

> 💡 **Why:** Admin actions let staff perform bulk operations. Always show feedback via `messages` so users know what happened.

### Readonly Fields

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Order

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'total', 'status')
    # Staff can edit total and created_at — these should be computed/automatic
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import Order


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'total', 'status', 'created_at')
    readonly_fields = ('total', 'created_at', 'updated_at', 'item_summary')

    @admin.display(description='Item Summary')
    def item_summary(self, obj):
        items = obj.items.select_related('product')
        return ', '.join(f'{i.product.name} x{i.quantity}' for i in items)
```

> 💡 **Why:** Use `readonly_fields` for computed values, timestamps, and fields that should not be manually edited. You can include method names to display derived data.

### Fieldsets

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Product

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    pass  # All fields in a single flat form — hard to navigate with many fields
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import Product


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    fieldsets = (
        (None, {
            'fields': ('name', 'slug', 'description'),
        }),
        ('Pricing', {
            'fields': ('price', 'sale_price', 'cost'),
        }),
        ('Inventory', {
            'fields': ('sku', 'stock_quantity', 'is_active'),
        }),
        ('SEO', {
            'classes': ('collapse',),
            'fields': ('meta_title', 'meta_description'),
        }),
    )
    prepopulated_fields = {'slug': ('name',)}
```

> 💡 **Why:** Fieldsets organize forms into logical groups. Use `collapse` for rarely-used sections. `prepopulated_fields` auto-fills slugs from titles in the admin.

### Admin Forms

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Event

@admin.register(Event)
class EventAdmin(admin.ModelAdmin):
    pass
    # No custom validation — admin allows end_date before start_date
```

✅ **Correct:**
```python
from django import forms
from django.contrib import admin
from .models import Event


class EventAdminForm(forms.ModelForm):
    class Meta:
        model = Event
        fields = '__all__'

    def clean(self):
        cleaned_data = super().clean()
        start = cleaned_data.get('start_date')
        end = cleaned_data.get('end_date')
        if start and end and end < start:
            raise forms.ValidationError('End date must be after start date.')
        return cleaned_data


@admin.register(Event)
class EventAdmin(admin.ModelAdmin):
    form = EventAdminForm
    list_display = ('title', 'start_date', 'end_date')
```

> 💡 **Why:** Custom admin forms add validation rules that the database constraints alone can't express. The admin is a data entry tool — validate accordingly.

### Custom Admin Views

❌ **Wrong:**
```python
# Creating a separate Django view and URL outside the admin for a report
# urls.py
from myapp.views import sales_report
urlpatterns = [
    path('admin/sales-report/', sales_report),  # No auth, no admin integration
]
```

✅ **Correct:**
```python
from django.contrib import admin
from django.template.response import TemplateResponse
from django.urls import path
from .models import Order


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'total', 'status')

    def get_urls(self):
        custom_urls = [
            path('sales-report/', self.admin_site.admin_view(self.sales_report_view),
                 name='order-sales-report'),
        ]
        return custom_urls + super().get_urls()

    def sales_report_view(self, request):
        from django.db.models import Sum, Count
        stats = Order.objects.aggregate(
            total_revenue=Sum('total'),
            total_orders=Count('id'),
        )
        context = {**self.admin_site.each_context(request), 'stats': stats}
        return TemplateResponse(request, 'admin/orders/sales_report.html', context)
```

> 💡 **Why:** `get_urls()` adds custom views inside the admin with proper authentication. `admin_site.admin_view()` wraps the view with admin permission checks.

### Admin Security

❌ **Wrong:**
```python
from django.contrib import admin
from .models import User

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('username', 'email', 'is_superuser')
    # All staff can see all users, modify superuser status, etc.
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import User


@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('username', 'email', 'is_active')

    def has_delete_permission(self, request, obj=None):
        return request.user.is_superuser

    def get_readonly_fields(self, request, obj=None):
        if not request.user.is_superuser:
            return ('is_superuser', 'is_staff', 'user_permissions', 'groups')
        return ()

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if not request.user.is_superuser:
            qs = qs.exclude(is_superuser=True)
        return qs
```

> 💡 **Why:** Override permission methods to restrict what staff users can see and do. Non-superusers shouldn't be able to grant superuser access or delete accounts.

### Admin Performance

❌ **Wrong:**
```python
from django.contrib import admin
from .models import Order

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'customer_name', 'total')

    def customer_name(self, obj):
        return obj.customer.name  # N+1 query — one per row
```

✅ **Correct:**
```python
from django.contrib import admin
from .models import Order


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'customer_name', 'total')
    list_select_related = ('customer',)
    show_full_result_count = False  # Disables COUNT(*) on large tables

    @admin.display(description='Customer', ordering='customer__name')
    def customer_name(self, obj):
        return obj.customer.name
```

> 💡 **Why:** `list_select_related` prevents N+1 queries in list views. `show_full_result_count = False` avoids a slow COUNT(*) query on tables with millions of rows.

## 6. Views

### Function-Based View Structure

❌ **Wrong:**
```python
from django.http import HttpResponse
from .models import Product

def products(request):
    # No method check, no decorator, mixes GET and POST
    if request.POST:
        name = request.POST['name']
        Product.objects.create(name=name)
    products = Product.objects.all()
    html = '<ul>' + ''.join(f'<li>{p.name}</li>' for p in products) + '</ul>'
    return HttpResponse(html)
```

✅ **Correct:**
```python
from django.shortcuts import render, redirect
from django.views.decorators.http import require_http_methods
from .models import Product
from .forms import ProductForm


@require_http_methods(["GET"])
def product_list(request):
    products = Product.objects.select_related('category').all()
    return render(request, 'products/list.html', {'products': products})


@require_http_methods(["GET", "POST"])
def product_create(request):
    form = ProductForm(request.POST or None)
    if form.is_valid():
        form.save()
        return redirect('products:list')
    return render(request, 'products/create.html', {'form': form})
```

> 💡 **Why:** Separate views per action, use `require_http_methods` to restrict HTTP methods, use forms for validation, and always return proper responses from templates.

### Class-Based View Structure

❌ **Wrong:**
```python
from django.views import View
from django.http import JsonResponse
from .models import Product

class ProductView(View):
    # One view handling everything — list, detail, create, update, delete
    def get(self, request, pk=None):
        if pk:
            return JsonResponse(Product.objects.get(pk=pk).__dict__)
        return JsonResponse(list(Product.objects.values()), safe=False)

    def post(self, request, pk=None):
        # Create and update in same method
        ...
```

✅ **Correct:**
```python
from django.views import View
from django.shortcuts import render, get_object_or_404, redirect
from .models import Product
from .forms import ProductForm


class ProductListView(View):
    def get(self, request):
        products = Product.objects.all()
        return render(request, 'products/list.html', {'products': products})


class ProductDetailView(View):
    def get(self, request, pk):
        product = get_object_or_404(Product, pk=pk)
        return render(request, 'products/detail.html', {'product': product})
```

> 💡 **Why:** One view per action. CBVs map HTTP methods to class methods (`get`, `post`, `put`, `delete`). Don't overload a single view to handle everything.

### TemplateView, ListView, DetailView

❌ **Wrong:**
```python
from django.shortcuts import render
from .models import Article

# Re-implementing what generic views already do
def article_list(request):
    articles = Article.objects.all()
    page = request.GET.get('page', 1)
    # Manual pagination logic...
    return render(request, 'articles/list.html', {'articles': articles})
```

✅ **Correct:**
```python
from django.views.generic import TemplateView, ListView, DetailView
from .models import Article


class HomeView(TemplateView):
    template_name = 'home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['featured'] = Article.objects.filter(is_featured=True)[:5]
        return context


class ArticleListView(ListView):
    model = Article
    template_name = 'articles/list.html'
    context_object_name = 'articles'
    paginate_by = 20
    ordering = '-created_at'


class ArticleDetailView(DetailView):
    model = Article
    template_name = 'articles/detail.html'
    context_object_name = 'article'
```

> 💡 **Why:** Generic views handle pagination, 404s, and context setup out of the box. Override `get_queryset()` or `get_context_data()` to customize behavior.

### CreateView, UpdateView, DeleteView

❌ **Wrong:**
```python
from django.shortcuts import render, redirect, get_object_or_404
from .models import Article
from .forms import ArticleForm

def article_create(request):
    if request.method == 'POST':
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user
            article.save()
            return redirect('/articles/')  # Hardcoded URL
    else:
        form = ArticleForm()
    return render(request, 'articles/form.html', {'form': form})
```

✅ **Correct:**
```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, UpdateView, DeleteView
from .models import Article
from .forms import ArticleForm


class ArticleCreateView(LoginRequiredMixin, CreateView):
    model = Article
    form_class = ArticleForm
    template_name = 'articles/form.html'
    success_url = reverse_lazy('articles:list')

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)


class ArticleUpdateView(LoginRequiredMixin, UpdateView):
    model = Article
    form_class = ArticleForm
    template_name = 'articles/form.html'
    success_url = reverse_lazy('articles:list')

    def get_queryset(self):
        return super().get_queryset().filter(author=self.request.user)


class ArticleDeleteView(LoginRequiredMixin, DeleteView):
    model = Article
    template_name = 'articles/confirm_delete.html'
    success_url = reverse_lazy('articles:list')

    def get_queryset(self):
        return super().get_queryset().filter(author=self.request.user)
```

> 💡 **Why:** Generic editing views handle form rendering, validation, and redirects. Override `get_queryset()` to enforce ownership — users should only edit their own objects.

### FormView

❌ **Wrong:**
```python
from django.shortcuts import render
from .forms import ContactForm

def contact(request):
    if request.method == 'POST':
        form = ContactForm(request.POST)
        if form.is_valid():
            # send email
            return render(request, 'contact/success.html')
        return render(request, 'contact/form.html', {'form': form})
    return render(request, 'contact/form.html', {'form': ContactForm()})
```

✅ **Correct:**
```python
from django.urls import reverse_lazy
from django.views.generic import FormView
from .forms import ContactForm


class ContactView(FormView):
    template_name = 'contact/form.html'
    form_class = ContactForm
    success_url = reverse_lazy('contact:success')

    def form_valid(self, form):
        form.send_email()
        return super().form_valid(form)
```

> 💡 **Why:** FormView handles the GET/POST branching and re-rendering with errors. Override `form_valid()` for the success action and `form_invalid()` for custom error handling.

### RedirectView

❌ **Wrong:**
```python
from django.shortcuts import redirect

# A whole view function just to redirect
def old_page(request):
    return redirect('/new-page/')
```

✅ **Correct:**
```python
from django.views.generic import RedirectView
from django.urls import path

urlpatterns = [
    path('old-page/', RedirectView.as_view(pattern_name='pages:new', permanent=True)),
]
```

> 💡 **Why:** RedirectView is declarative and can be defined directly in URL config. Use `permanent=True` for 301 (SEO redirect) and `permanent=False` for 302 (temporary).

### Mixin Usage

❌ **Wrong:**
```python
from django.views.generic import ListView
from .models import Article

class ArticleListView(ListView):
    model = Article

    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            from django.shortcuts import redirect
            return redirect('/login/')
        if not request.user.has_perm('articles.view_article'):
            from django.http import HttpResponseForbidden
            return HttpResponseForbidden()
        return super().dispatch(request, *args, **kwargs)
```

✅ **Correct:**
```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView
from .models import Article


class ArticleListView(LoginRequiredMixin, PermissionRequiredMixin, ListView):
    model = Article
    permission_required = 'articles.view_article'
    login_url = '/accounts/login/'
    raise_exception = True  # 403 instead of redirect for permission denied
```

> 💡 **Why:** Mixins are reusable and declarative. `LoginRequiredMixin` must come before the view class in MRO. `PermissionRequiredMixin` handles both authentication and permission checks.

### FBV vs CBV Decision Criteria

❌ **Wrong:**
```python
# Using CBV for a simple one-off view that doesn't benefit from inheritance
from django.views import View
from django.http import JsonResponse

class HealthCheckView(View):
    def get(self, request):
        return JsonResponse({'status': 'ok'})
```

✅ **Correct:**
```python
# Simple views — use FBV
from django.http import JsonResponse
from django.views.decorators.http import require_GET

@require_GET
def health_check(request):
    return JsonResponse({'status': 'ok'})


# CRUD and standard patterns — use CBV with generics
from django.views.generic import ListView
from .models import Product

class ProductListView(ListView):
    model = Product
    paginate_by = 20
```

> 💡 **Why:** Use FBVs for simple, one-off views (health checks, webhooks, custom logic). Use CBVs when generic views save boilerplate (CRUD, list/detail). Don't force CBV on everything.

## 7. URL Routing

### path() and re_path()

❌ **Wrong:**
```python
from django.urls import re_path
from . import views

# Using regex when simple path converters would work
urlpatterns = [
    re_path(r'^products/(?P<pk>[0-9]+)/$', views.product_detail),
    re_path(r'^products/(?P<slug>[\w-]+)/$', views.product_by_slug),
]
```

✅ **Correct:**
```python
from django.urls import path, re_path
from . import views

urlpatterns = [
    # path() with built-in converters — cleaner syntax
    path('products/<int:pk>/', views.product_detail, name='detail'),
    path('products/<slug:slug>/', views.product_by_slug, name='by-slug'),

    # re_path() only when you need complex regex
    re_path(r'^archive/(?P<year>[0-9]{4})-(?P<month>[0-9]{2})/$',
            views.archive, name='archive'),
]
```

> 💡 **Why:** `path()` is simpler and less error-prone. Only use `re_path()` when you need regex patterns that path converters can't express (like date formats or complex patterns).

### Separating App URLs with include()

❌ **Wrong:**
```python
# config/urls.py — importing all views from all apps
from users.views import login_view, register_view, profile_view
from products.views import product_list, product_detail, product_create

urlpatterns = [
    path('login/', login_view),
    path('register/', register_view),
    path('profile/', profile_view),
    path('products/', product_list),
    path('products/<int:pk>/', product_detail),
    path('products/create/', product_create),
]
```

✅ **Correct:**
```python
# config/urls.py
from django.urls import path, include

urlpatterns = [
    path('', include('apps.pages.urls')),
    path('users/', include('apps.users.urls')),
    path('products/', include('apps.products.urls')),
    path('api/v1/', include('apps.api.urls')),
]

# apps/products/urls.py
from django.urls import path
from . import views

app_name = 'products'
urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
    path('create/', views.product_create, name='create'),
]
```

> 💡 **Why:** Each app owns its URLs. The root `urls.py` stays clean — just a list of `include()` calls. This makes apps portable and self-contained.

### URL Namespaces

❌ **Wrong:**
```python
# No app_name, no namespaces — name collisions between apps
# users/urls.py
urlpatterns = [
    path('', views.list_view, name='list'),  # Conflicts with products:list
]

# products/urls.py
urlpatterns = [
    path('', views.list_view, name='list'),  # Same name!
]
```

✅ **Correct:**
```python
# users/urls.py
from django.urls import path
from . import views

app_name = 'users'
urlpatterns = [
    path('', views.user_list, name='list'),
    path('<int:pk>/', views.user_detail, name='detail'),
]

# products/urls.py
from django.urls import path
from . import views

app_name = 'products'
urlpatterns = [
    path('', views.product_list, name='list'),
    path('<int:pk>/', views.product_detail, name='detail'),
]

# Usage in templates: {% url 'users:list' %} {% url 'products:detail' pk=1 %}
```

> 💡 **Why:** `app_name` creates a namespace so `reverse('users:list')` and `reverse('products:list')` are unambiguous. Required when using `include()` with named URLs.

### reverse() and reverse_lazy()

❌ **Wrong:**
```python
from django.urls import reverse

class ArticleCreateView(CreateView):
    model = Article
    # reverse() is called at import time — before URL config is loaded
    success_url = reverse('articles:list')  # This crashes!
```

✅ **Correct:**
```python
from django.urls import reverse, reverse_lazy
from django.views.generic import CreateView
from .models import Article


class ArticleCreateView(CreateView):
    model = Article
    success_url = reverse_lazy('articles:list')  # Evaluated when needed, not at import

    def get_success_url(self):
        # Or use reverse() inside a method — it's called at request time
        return reverse('articles:detail', kwargs={'pk': self.object.pk})
```

> 💡 **Why:** `reverse_lazy()` delays URL resolution until first access. Use it in class attributes and module-level code. `reverse()` is fine inside functions/methods that run at request time.

### URL Parameters and Custom Converters

❌ **Wrong:**
```python
from django.urls import path
from . import views

urlpatterns = [
    # Accepting any string then validating in the view
    path('products/<slug>/', views.product_detail, name='detail'),
    # slug converter allows characters you might not want
]
```

✅ **Correct:**
```python
# converters.py
class FourDigitYearConverter:
    regex = '[0-9]{4}'

    def to_python(self, value):
        return int(value)

    def to_url(self, value):
        return f'{value:04d}'


# urls.py
from django.urls import path, register_converter
from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')

urlpatterns = [
    path('archive/<yyyy:year>/', views.archive_view, name='archive'),
    path('products/<uuid:id>/', views.product_detail, name='detail'),
]
```

> 💡 **Why:** Built-in converters (`int`, `str`, `slug`, `uuid`, `path`) handle common cases. Custom converters enforce URL patterns at the routing level, before your view code runs.

## 8. Templates

### Template Inheritance

❌ **Wrong:**
```html
<!-- Every template repeats the full HTML structure -->
<!-- products/list.html -->
<!DOCTYPE html>
<html>
<head><title>Products</title><link rel="stylesheet" href="/static/style.css"></head>
<body>
  <nav>...</nav>
  <h1>Products</h1>
  <!-- content -->
  <footer>...</footer>
</body>
</html>

<!-- articles/list.html — same boilerplate duplicated -->
<!DOCTYPE html>
<html>
<head><title>Articles</title><link rel="stylesheet" href="/static/style.css"></head>
<body>
  <nav>...</nav>
  <h1>Articles</h1>
  <!-- content -->
  <footer>...</footer>
</body>
</html>
```

✅ **Correct:**
```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}My Site{% endblock %}</title>
  {% block extra_css %}{% endblock %}
</head>
<body>
  <nav>{% include "includes/nav.html" %}</nav>
  <main>
    {% block content %}{% endblock %}
  </main>
  <footer>{% include "includes/footer.html" %}</footer>
  {% block extra_js %}{% endblock %}
</body>
</html>

<!-- templates/products/list.html -->
{% extends "base.html" %}

{% block title %}Products{% endblock %}

{% block content %}
  <h1>Products</h1>
  {% for product in products %}
    <div>{{ product.name }}</div>
  {% endfor %}
{% endblock %}
```

> 💡 **Why:** Template inheritance eliminates duplication. `base.html` defines the skeleton, child templates override specific blocks. Use `{% include %}` for reusable fragments.

### Template Tags

❌ **Wrong:**
```html
<!-- Hardcoded URLs and static paths -->
<a href="/products/{{ product.id }}/">{{ product.name }}</a>
<img src="/static/images/logo.png">

<!-- No empty list handling -->
{% for item in items %}
  <li>{{ item.name }}</li>
{% endfor %}
```

✅ **Correct:**
```html
{% load static %}

<!-- Named URLs — won't break when URL patterns change -->
<a href="{% url 'products:detail' pk=product.pk %}">{{ product.name }}</a>
<img src="{% static 'images/logo.png' %}" alt="Logo">

<!-- Handle empty lists -->
{% for item in items %}
  <li>{{ item.name }}</li>
{% empty %}
  <li>No items found.</li>
{% endfor %}

<!-- Use {% with %} to avoid repeated expensive lookups -->
{% with total=order.get_total %}
  <p>Total: ${{ total }}</p>
  <p>Tax: ${{ total|floatformat:2 }}</p>
{% endwith %}
```

> 💡 **Why:** `{% url %}` and `{% static %}` generate correct URLs regardless of deployment config. `{% empty %}` handles the no-results case. `{% with %}` caches computed values.

### Template Filters

❌ **Wrong:**
```html
<!-- Formatting in the view instead of the template -->
<!-- views.py: context['date'] = obj.created_at.strftime('%B %d, %Y') -->
<p>{{ date }}</p>

<!-- Truncating in Python -->
<!-- views.py: context['desc'] = obj.description[:100] + '...' -->
<p>{{ desc }}</p>
```

✅ **Correct:**
```html
<!-- Built-in filters handle formatting -->
<p>{{ article.created_at|date:"F j, Y" }}</p>
<p>{{ article.description|truncatewords:20 }}</p>
<p>{{ article.body|linebreaks }}</p>
<p>{{ price|floatformat:2 }}</p>
<p>{{ name|default:"Anonymous" }}</p>
<p>{{ count|pluralize:"y,ies" }}</p>
```

> 💡 **Why:** Template filters keep presentation logic in templates where it belongs. The view provides raw data, the template formats it for display.

### Custom Template Tags

❌ **Wrong:**
```python
# Passing computed data through every single view's context
# views.py
def product_list(request):
    return render(request, 'products/list.html', {
        'products': Product.objects.all(),
        'cart_count': request.session.get('cart', {}).items().__len__(),  # Repeated everywhere
    })
```

✅ **Correct:**
```python
# templatetags/shop_tags.py
from django import template
from django.utils.safestring import mark_safe

register = template.Library()


@register.simple_tag(takes_context=True)
def cart_count(context):
    request = context['request']
    cart = request.session.get('cart', {})
    return len(cart)


@register.inclusion_tag('includes/recent_articles.html')
def recent_articles(count=5):
    from articles.models import Article
    return {'articles': Article.objects.order_by('-created_at')[:count]}


@register.filter
def currency(value):
    return f'${value:,.2f}'
```

```html
{% load shop_tags %}
<span>Cart: {% cart_count %}</span>
{% recent_articles 3 %}
<p>{{ product.price|currency }}</p>
```

> 💡 **Why:** Custom tags and filters encapsulate reusable template logic. `simple_tag` for computed values, `inclusion_tag` for reusable template fragments, `filter` for value transformation.

### Context Processors

❌ **Wrong:**
```python
# Adding site_name to every view manually
def home(request):
    return render(request, 'home.html', {'site_name': 'My Site', ...})

def about(request):
    return render(request, 'about.html', {'site_name': 'My Site', ...})
```

✅ **Correct:**
```python
# context_processors.py
from django.conf import settings


def site_context(request):
    return {
        'site_name': settings.SITE_NAME,
        'support_email': settings.SUPPORT_EMAIL,
    }


# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'myapp.context_processors.site_context',  # Custom
            ],
        },
    },
]
```

> 💡 **Why:** Context processors inject variables into every template automatically. Use them for truly global data (site name, feature flags) — not for view-specific data.

### Template Security

❌ **Wrong:**
```html
<!-- Disabling autoescaping carelessly -->
{% autoescape off %}
  {{ user_comment }}  <!-- XSS vulnerability if comment contains <script> -->
{% endautoescape %}

{{ user_bio|safe }}  <!-- Also dangerous — marks raw HTML as safe -->
```

✅ **Correct:**
```html
<!-- Django autoescapes by default — trust it -->
<p>{{ user_comment }}</p>  <!-- <script> tags are escaped automatically -->

<!-- Only use |safe for content YOU control, never user input -->
{{ admin_announcement|safe }}  <!-- Only if set by trusted staff in admin -->

<!-- For user-generated HTML, use bleach to sanitize first -->
<!-- In the view: cleaned = bleach.clean(user_input, tags=['b', 'i', 'a']) -->
<p>{{ cleaned_html|safe }}</p>
```

> 💡 **Why:** Django's autoescaping prevents XSS by default. Never use `|safe` or `{% autoescape off %}` on user-supplied content. Sanitize HTML with a library like bleach before marking it safe.

## 9. Forms

### Django Forms

❌ **Wrong:**
```python
# Manually parsing request.POST without validation
def contact(request):
    if request.method == 'POST':
        name = request.POST.get('name', '')
        email = request.POST.get('email', '')
        # No validation, no error messages, no CSRF
        send_email(name, email)
```

✅ **Correct:**
```python
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100, widget=forms.TextInput(attrs={
        'class': 'form-control',
        'placeholder': 'Your name',
    }))
    email = forms.EmailField(widget=forms.EmailInput(attrs={
        'class': 'form-control',
    }))
    message = forms.CharField(widget=forms.Textarea(attrs={
        'rows': 5,
        'class': 'form-control',
    }))

    def send_email(self):
        # Use cleaned_data, not raw POST data
        from django.core.mail import send_mail
        send_mail(
            subject=f'Contact from {self.cleaned_data["name"]}',
            message=self.cleaned_data['message'],
            from_email=self.cleaned_data['email'],
            recipient_list=['support@example.com'],
        )
```

> 💡 **Why:** Django forms validate input, provide error messages, and render HTML widgets. Always access `cleaned_data` after validation — never trust raw `request.POST`.

### ModelForms

❌ **Wrong:**
```python
from django import forms
from .models import Product

class ProductForm(forms.ModelForm):
    class Meta:
        model = Product
        exclude = []  # Exposes every field including internal ones
        # Or:
        # fields = '__all__'  # Same problem
```

✅ **Correct:**
```python
from django import forms
from .models import Product


class ProductForm(forms.ModelForm):
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'category', 'is_active']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 4}),
        }

    def save(self, commit=True):
        product = super().save(commit=False)
        product.slug = slugify(product.name)
        if commit:
            product.save()
        return product
```

> 💡 **Why:** Always use explicit `fields` — never `exclude` or `'__all__'`. Exclude can accidentally expose sensitive fields added later. Override `save()` to set computed fields.

### Form Validation

❌ **Wrong:**
```python
from django import forms

class RegistrationForm(forms.Form):
    username = forms.CharField()
    password = forms.CharField()
    password_confirm = forms.CharField()

    # Validation in the view instead of the form
    # if form.cleaned_data['password'] != form.cleaned_data['password_confirm']:
    #     messages.error(request, 'Passwords do not match')
```

✅ **Correct:**
```python
from django import forms
from django.core.exceptions import ValidationError


class RegistrationForm(forms.Form):
    username = forms.CharField(max_length=150)
    email = forms.EmailField()
    password = forms.CharField(widget=forms.PasswordInput)
    password_confirm = forms.CharField(widget=forms.PasswordInput)

    def clean_username(self):
        from django.contrib.auth import get_user_model
        User = get_user_model()
        username = self.cleaned_data['username']
        if User.objects.filter(username=username).exists():
            raise ValidationError('This username is already taken.')
        return username

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        confirm = cleaned_data.get('password_confirm')
        if password and confirm and password != confirm:
            raise ValidationError('Passwords do not match.')
        return cleaned_data
```

> 💡 **Why:** `clean_<field>()` validates individual fields. `clean()` validates cross-field logic. Both raise `ValidationError` which Django renders as error messages on the form.

### Custom Validators

❌ **Wrong:**
```python
from django.db import models

class Product(models.Model):
    price = models.DecimalField(max_digits=10, decimal_places=2)
    # No validation — allows negative prices
```

✅ **Correct:**
```python
from django.core.exceptions import ValidationError
from django.core.validators import MinValueValidator
from django.db import models


def validate_not_profanity(value):
    profanity_list = ['badword1', 'badword2']
    for word in profanity_list:
        if word in value.lower():
            raise ValidationError(f'"{word}" is not allowed.')


class Product(models.Model):
    name = models.CharField(max_length=255, validators=[validate_not_profanity])
    price = models.DecimalField(
        max_digits=10, decimal_places=2,
        validators=[MinValueValidator(0.01)],
    )
```

> 💡 **Why:** Validators are reusable across models and forms. Use Django's built-in validators (`MinValueValidator`, `MaxLengthValidator`, `RegexValidator`) and write custom ones for domain-specific rules.

### Formsets

❌ **Wrong:**
```python
# Manually handling multiple forms with numbered field names
# <input name="item_0_name"> <input name="item_1_name">
def handle_items(request):
    i = 0
    while f'item_{i}_name' in request.POST:
        name = request.POST[f'item_{i}_name']
        # No validation, easy to break
        i += 1
```

✅ **Correct:**
```python
from django.forms import formset_factory, modelformset_factory
from .forms import ItemForm
from .models import OrderItem


# Regular formset
ItemFormSet = formset_factory(ItemForm, extra=3, min_num=1, validate_min=True)

def add_items(request):
    formset = ItemFormSet(request.POST or None)
    if formset.is_valid():
        for form in formset:
            if form.cleaned_data:
                form.save()
    return render(request, 'items/form.html', {'formset': formset})


# Model formset — tied to a queryset
OrderItemFormSet = modelformset_factory(
    OrderItem, fields=['product', 'quantity'], extra=1
)
```

```html
<form method="post">
  {% csrf_token %}
  {{ formset.management_form }}
  {% for form in formset %}
    <div>{{ form.as_p }}</div>
  {% endfor %}
  <button type="submit">Save</button>
</form>
```

> 💡 **Why:** Formsets handle multiple instances of the same form with proper validation. Always render `management_form` — it contains the hidden fields Django needs to track form count.

### Form Security

❌ **Wrong:**
```python
# Disabling CSRF for convenience
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # Security vulnerability!
def payment_webhook(request):
    # Accepting file uploads without validation
    uploaded = request.FILES['document']
    uploaded.save('/uploads/' + uploaded.name)  # Path traversal + no type check
```

✅ **Correct:**
```python
from django import forms
from django.core.validators import FileExtensionValidator


class DocumentUploadForm(forms.Form):
    document = forms.FileField(
        validators=[FileExtensionValidator(allowed_extensions=['pdf', 'docx'])],
    )

    def clean_document(self):
        doc = self.cleaned_data['document']
        if doc.size > 10 * 1024 * 1024:  # 10 MB limit
            raise forms.ValidationError('File too large. Max 10 MB.')
        # Verify content type matches extension
        import magic
        mime = magic.from_buffer(doc.read(2048), mime=True)
        doc.seek(0)
        allowed_mimes = ['application/pdf', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document']
        if mime not in allowed_mimes:
            raise forms.ValidationError('Invalid file type.')
        return doc
```

```html
<!-- Always include CSRF token -->
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Upload</button>
</form>
```

> 💡 **Why:** Never disable CSRF unless the endpoint is called by external systems (webhooks with their own auth). Validate file uploads by extension, size, and MIME type to prevent malicious uploads.

## 10. Authentication System

### Default User Model

❌ **Wrong:**
```python
# Starting a new project and immediately using auth.User directly everywhere
from django.contrib.auth.models import User

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    # Hardcoded to auth.User — can't switch to custom user later
```

✅ **Correct:**
```python
from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='profile',
    )
```

> 💡 **Why:** Always reference `settings.AUTH_USER_MODEL` instead of importing User directly. The default User is fine for simple projects, but referencing via settings lets you swap later without rewriting ForeignKeys.

### Custom User Model

❌ **Wrong:**
```python
# Deciding to customize the user model mid-project
# This requires complex migration surgery and often a database rebuild
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    phone = models.CharField(max_length=20)
    # Adding this after tables exist requires recreating the auth tables
```

✅ **Correct:**
```python
# Do this BEFORE the first migrate — at the very start of the project
# users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    phone = models.CharField(max_length=20, blank=True)
    date_of_birth = models.DateField(null=True, blank=True)

    class Meta:
        verbose_name = 'user'
        verbose_name_plural = 'users'


# settings.py
AUTH_USER_MODEL = 'users.User'
```

```python
# For maximum control — AbstractBaseUser
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin, BaseUserManager


class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('Email is required')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        return self.create_user(email, password, **extra_fields)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['name']
```

> 💡 **Why:** Define a custom user model at the start of every project — even if you don't need it yet. Switching later requires recreating the database. Use `AbstractUser` to extend, `AbstractBaseUser` to fully control.

### Authentication Backend

❌ **Wrong:**
```python
# Manually checking passwords in views
from django.contrib.auth.hashers import check_password
from django.contrib.auth.models import User

def login_view(request):
    user = User.objects.get(email=request.POST['email'])
    if check_password(request.POST['password'], user.password):
        # Manual session management...
        pass
```

✅ **Correct:**
```python
# Custom backend for email-based login
# backends.py
from django.contrib.auth import get_user_model
from django.contrib.auth.backends import ModelBackend

User = get_user_model()


class EmailBackend(ModelBackend):
    def authenticate(self, request, username=None, password=None, **kwargs):
        email = kwargs.get('email', username)
        try:
            user = User.objects.get(email=email)
        except User.DoesNotExist:
            return None
        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        return None


# settings.py
AUTHENTICATION_BACKENDS = [
    'apps.users.backends.EmailBackend',
    'django.contrib.auth.backends.ModelBackend',
]
```

> 💡 **Why:** Custom backends let you authenticate by email, LDAP, OAuth, etc. Django tries each backend in order. `user_can_authenticate` checks `is_active`.

### Login/Logout Views

❌ **Wrong:**
```python
from django.contrib.auth import authenticate, login

def login_view(request):
    if request.method == 'POST':
        user = authenticate(username=request.POST['username'],
                          password=request.POST['password'])
        if user:
            login(request, user)
            return redirect('/')
    return render(request, 'login.html')
    # No CSRF, no form validation, no error messages, no rate limiting
```

✅ **Correct:**
```python
# urls.py — use Django's built-in auth views
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(template_name='users/login.html'), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]

# settings.py
LOGIN_URL = '/users/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

```html
<!-- users/login.html -->
{% extends "base.html" %}
{% block content %}
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Login</button>
</form>
{% endblock %}
```

> 💡 **Why:** Django's auth views handle CSRF, form validation, error messages, and the `next` parameter for post-login redirect. Don't rewrite what's already built.

### Password Reset Flow

❌ **Wrong:**
```python
# Building password reset from scratch
def reset_password(request):
    email = request.POST['email']
    user = User.objects.get(email=email)
    new_password = 'temporary123'  # Sending plaintext password!
    user.set_password(new_password)
    user.save()
    send_mail('Your new password', f'Password: {new_password}', ...)
```

✅ **Correct:**
```python
# urls.py — use Django's built-in password reset views
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path('password-reset/',
         auth_views.PasswordResetView.as_view(template_name='users/password_reset.html'),
         name='password_reset'),
    path('password-reset/done/',
         auth_views.PasswordResetDoneView.as_view(template_name='users/password_reset_done.html'),
         name='password_reset_done'),
    path('password-reset/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(template_name='users/password_reset_confirm.html'),
         name='password_reset_confirm'),
    path('password-reset/complete/',
         auth_views.PasswordResetCompleteView.as_view(template_name='users/password_reset_complete.html'),
         name='password_reset_complete'),
]
```

> 💡 **Why:** Django's reset flow uses cryptographically signed tokens that expire. Never send plaintext passwords. The built-in views handle the full flow: request, email, confirmation, and completion.

### Permissions

❌ **Wrong:**
```python
# Checking permissions with hardcoded role strings
def delete_article(request, pk):
    if request.user.role != 'admin':  # Custom role field — doesn't use Django's permission system
        return HttpResponseForbidden()
    Article.objects.get(pk=pk).delete()
```

✅ **Correct:**
```python
from django.contrib.auth.decorators import permission_required
from django.shortcuts import get_object_or_404
from .models import Article


@permission_required('articles.delete_article', raise_exception=True)
def delete_article(request, pk):
    article = get_object_or_404(Article, pk=pk)
    article.delete()
    return redirect('articles:list')


# Object-level permissions with django-guardian
from guardian.shortcuts import assign_perm

def create_article(request):
    article = Article.objects.create(author=request.user, ...)
    assign_perm('articles.change_article', request.user, article)
    assign_perm('articles.delete_article', request.user, article)
```

> 💡 **Why:** Django auto-creates add/change/delete/view permissions per model. Use them with `@permission_required` or `has_perm()`. For object-level permissions, use django-guardian.

### Groups Usage

❌ **Wrong:**
```python
# Assigning permissions to users individually
from django.contrib.auth.models import Permission

user = User.objects.get(username='editor')
perms = Permission.objects.filter(codename__in=[
    'add_article', 'change_article', 'view_article'
])
user.user_permissions.set(perms)
# Repeat for every new editor — tedious and error-prone
```

✅ **Correct:**
```python
from django.contrib.auth.models import Group, Permission

# Create groups once (in a migration or management command)
editors_group, _ = Group.objects.get_or_create(name='Editors')
editor_perms = Permission.objects.filter(
    content_type__app_label='articles',
    codename__in=['add_article', 'change_article', 'view_article'],
)
editors_group.permissions.set(editor_perms)

# Assign users to groups
user.groups.add(editors_group)

# Check in templates: {% if perms.articles.change_article %}
# Check in code: user.has_perm('articles.change_article')
```

> 💡 **Why:** Groups let you manage permissions as roles. Change the group's permissions once, and all members are updated. Much easier than managing per-user permissions.

### @login_required and @permission_required Decorators

❌ **Wrong:**
```python
def dashboard(request):
    if not request.user.is_authenticated:
        return redirect('/login/')
    if not request.user.has_perm('reports.view_dashboard'):
        return HttpResponseForbidden()
    # ...
```

✅ **Correct:**
```python
from django.contrib.auth.decorators import login_required, permission_required


@login_required
def dashboard(request):
    return render(request, 'dashboard.html')


@permission_required('reports.view_dashboard', raise_exception=True)
def admin_dashboard(request):
    return render(request, 'admin_dashboard.html')


# Stacking decorators
@login_required
@permission_required('reports.export', raise_exception=True)
def export_report(request):
    ...
```

> 💡 **Why:** Decorators are declarative and consistent. `login_required` redirects to `LOGIN_URL`. `raise_exception=True` returns 403 instead of redirecting to login (for already-authenticated users without permission).

### LoginRequiredMixin and PermissionRequiredMixin

❌ **Wrong:**
```python
from django.views.generic import ListView

class ArticleListView(ListView):
    model = Article

    def dispatch(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect('login')
        return super().dispatch(request, *args, **kwargs)
```

✅ **Correct:**
```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView, UpdateView
from .models import Article


class ArticleListView(LoginRequiredMixin, ListView):
    model = Article
    login_url = '/accounts/login/'


class ArticleUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Article
    fields = ['title', 'body']
    permission_required = 'articles.change_article'
    raise_exception = True
```

> 💡 **Why:** Mixins must come before the view class in the inheritance chain (MRO). They handle redirect-to-login and 403 responses automatically.

## 11. Sessions & Cookies

### Session Middleware Configuration

❌ **Wrong:**
```python
# settings.py — forgetting session middleware or putting it in wrong order
MIDDLEWARE = [
    'django.middleware.common.CommonMiddleware',
    # SessionMiddleware missing — request.session won't work
    'django.contrib.auth.middleware.AuthenticationMiddleware',
]
```

✅ **Correct:**
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',  # Before AuthenticationMiddleware
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # Depends on SessionMiddleware
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

> 💡 **Why:** `SessionMiddleware` must come before `AuthenticationMiddleware` because auth reads from the session. Middleware order matters — Django processes them top-to-bottom on requests.

### Session Backend Selection

❌ **Wrong:**
```python
# Using database sessions on a high-traffic site
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
# Every request hits the database for session lookup
```

✅ **Correct:**
```python
# Cache-based sessions for high-traffic sites (with Redis)
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
    },
    'sessions': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    },
}

# Or cached_db for durability — cache first, DB fallback
# SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
```

> 💡 **Why:** Database sessions add a query per request. Cache-based sessions (Redis/Memcached) are faster. Use `cached_db` if you need sessions to survive cache restarts.

### Cookie Security

❌ **Wrong:**
```python
# settings.py
SESSION_COOKIE_SECURE = False  # Cookie sent over HTTP
SESSION_COOKIE_HTTPONLY = False  # JavaScript can read the session cookie
CSRF_COOKIE_HTTPONLY = False
```

✅ **Correct:**
```python
# settings.py — production
SESSION_COOKIE_SECURE = True        # Only sent over HTTPS
SESSION_COOKIE_HTTPONLY = True       # Not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Lax'     # CSRF protection for cross-site requests
CSRF_COOKIE_SECURE = True           # CSRF cookie also HTTPS-only
CSRF_COOKIE_HTTPONLY = True          # CSRF cookie not accessible via JS
```

> 💡 **Why:** `Secure` prevents cookie theft over HTTP. `HttpOnly` prevents JavaScript access (XSS protection). `SameSite=Lax` blocks most cross-site request forgery attacks.

### Session Expiry

❌ **Wrong:**
```python
# Default session never expires until browser closes
# No explicit expiry configuration — sessions persist indefinitely on server
```

✅ **Correct:**
```python
# settings.py
SESSION_COOKIE_AGE = 60 * 60 * 24 * 7  # 1 week in seconds
SESSION_EXPIRE_AT_BROWSER_CLOSE = False  # Persist across browser sessions
SESSION_SAVE_EVERY_REQUEST = True  # Reset expiry on every request (sliding window)

# Per-view session expiry
def login_view(request):
    # ...
    if form.cleaned_data.get('remember_me'):
        request.session.set_expiry(60 * 60 * 24 * 30)  # 30 days
    else:
        request.session.set_expiry(0)  # Expire when browser closes
```

> 💡 **Why:** Set explicit session lifetimes. `SESSION_SAVE_EVERY_REQUEST` creates a sliding window — active users stay logged in. Use `set_expiry()` per-session for remember-me features.

### Session Security

❌ **Wrong:**
```python
# Not rotating session after login — session fixation vulnerability
from django.contrib.auth import login

def login_view(request):
    user = authenticate(request, ...)
    if user:
        # Reuses the same session ID from before login
        login(request, user)  # Django does rotate by default, but...
        request.session.cycle_key = lambda: None  # DON'T disable rotation
```

✅ **Correct:**
```python
from django.contrib.auth import login, authenticate


def login_view(request):
    if request.method == 'POST':
        user = authenticate(request, username=request.POST['username'],
                          password=request.POST['password'])
        if user is not None:
            # Django's login() calls request.session.cycle_key() automatically
            # This rotates the session ID to prevent session fixation
            login(request, user)
            return redirect('dashboard')

# For sensitive operations, manually flush the session
def change_password(request):
    # After password change, invalidate all other sessions
    from django.contrib.auth import update_session_auth_hash
    form = PasswordChangeForm(request.user, request.POST)
    if form.is_valid():
        form.save()
        update_session_auth_hash(request, form.user)  # Keep current session valid
```

> 💡 **Why:** Django's `login()` rotates session keys automatically. `update_session_auth_hash()` prevents logout after password change. Never disable session rotation.

## 12. Middleware

### Request/Response Lifecycle

❌ **Wrong:**
```python
# Putting timing logic directly in every view
import time

def my_view(request):
    start = time.time()
    # ... view logic ...
    duration = time.time() - start
    print(f'View took {duration}s')
```

✅ **Correct:**
```python
import time
import logging

logger = logging.getLogger(__name__)


class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = time.monotonic() - start
        response['X-Request-Duration'] = f'{duration:.3f}s'
        logger.info(f'{request.method} {request.path} — {duration:.3f}s')
        return response
```

> 💡 **Why:** Middleware intercepts every request/response. Cross-cutting concerns like logging, timing, and headers belong in middleware, not duplicated across views.

### Built-in Middleware Order

❌ **Wrong:**
```python
MIDDLEWARE = [
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.security.SecurityMiddleware',
    # Wrong order — SecurityMiddleware should be first, Session before Auth
]
```

✅ **Correct:**
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',           # 1st — security headers
    'django.contrib.sessions.middleware.SessionMiddleware',    # 2nd — session setup
    'django.middleware.common.CommonMiddleware',               # 3rd — URL normalization
    'django.middleware.csrf.CsrfViewMiddleware',               # 4th — CSRF check
    'django.contrib.auth.middleware.AuthenticationMiddleware',  # 5th — needs session
    'django.contrib.messages.middleware.MessageMiddleware',     # 6th — needs session
    'django.middleware.clickjacking.XFrameOptionsMiddleware',  # 7th — security header
]
```

> 💡 **Why:** SecurityMiddleware must be first to set security headers. SessionMiddleware must precede AuthenticationMiddleware. The documented order exists for a reason — follow it.

### Custom Middleware

❌ **Wrong:**
```python
# Old-style middleware class (pre-Django 1.10)
class OldMiddleware:
    def process_request(self, request):
        ...
    def process_response(self, request, response):
        ...
```

✅ **Correct:**
```python
# Class-based middleware (modern style)
from django.http import HttpResponseForbidden


class IPBlockMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.blocked_ips = {'1.2.3.4', '5.6.7.8'}

    def __call__(self, request):
        ip = request.META.get('REMOTE_ADDR')
        if ip in self.blocked_ips:
            return HttpResponseForbidden('Blocked')
        response = self.get_response(request)
        return response


# Functional middleware (simpler for one-off cases)
def simple_cors_middleware(get_response):
    def middleware(request):
        response = get_response(request)
        response['Access-Control-Allow-Origin'] = 'https://example.com'
        return response
    return middleware
```

> 💡 **Why:** Modern middleware uses `__init__` + `__call__`. The class receives `get_response` once at startup. Functional middleware is a shortcut for simple cases.

### Exception Handling in Middleware

❌ **Wrong:**
```python
class BadMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        return response

    def process_exception(self, request, exception):
        # Silently swallowing all exceptions — hides bugs
        return HttpResponse('Something went wrong', status=500)
```

✅ **Correct:**
```python
import logging
from django.http import JsonResponse

logger = logging.getLogger(__name__)


class ExceptionLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        return self.get_response(request)

    def process_exception(self, request, exception):
        logger.exception(
            f'Unhandled exception on {request.method} {request.path}',
            exc_info=exception,
            extra={'request': request},
        )
        # Return None to let Django's default exception handling continue
        # Or return a response to short-circuit
        return None
```

> 💡 **Why:** `process_exception` is called when a view raises an exception. Log the error, then return `None` to let Django's error handling proceed (including DEBUG pages and error reporters).

### Async Middleware (Django 4.1+)

❌ **Wrong:**
```python
# Using sync middleware with async views — Django wraps it in sync_to_async
# This works but adds overhead from thread pool context switches
class SyncMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        return self.get_response(request)
```

✅ **Correct:**
```python
# Django 4.1+ — async-compatible middleware
import asyncio


class AsyncTimingMiddleware:
    async_capable = True
    sync_capable = True

    def __init__(self, get_response):
        self.get_response = get_response
        if asyncio.iscoroutinefunction(self.get_response):
            self._is_async = True
        else:
            self._is_async = False

    def __call__(self, request):
        if self._is_async:
            return self.__acall__(request)
        import time
        start = time.monotonic()
        response = self.get_response(request)
        response['X-Duration'] = f'{time.monotonic() - start:.3f}s'
        return response

    async def __acall__(self, request):
        import time
        start = time.monotonic()
        response = await self.get_response(request)
        response['X-Duration'] = f'{time.monotonic() - start:.3f}s'
        return response
```

> 💡 **Why:** Async middleware avoids the overhead of `sync_to_async` wrapping. Set `async_capable = True` and implement `__acall__` for the async path.

## 13. Static & Media Files

### STATIC_URL, STATIC_ROOT, STATICFILES_DIRS

❌ **Wrong:**
```python
# Confusing STATIC_ROOT with STATICFILES_DIRS
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
# ^ Same directory for both — collectstatic will fail with an error
```

✅ **Correct:**
```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

STATIC_URL = '/static/'

# Where collectstatic gathers files for deployment
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Additional directories for your own static files (not app-level)
STATICFILES_DIRS = [
    BASE_DIR / 'static',  # Project-level static files
]
```

> 💡 **Why:** `STATICFILES_DIRS` is where you put your source static files. `STATIC_ROOT` is where `collectstatic` copies everything for production. They must be different directories.

### MEDIA_URL and MEDIA_ROOT

❌ **Wrong:**
```python
# Serving media files from STATIC_ROOT
# Or hardcoding absolute paths
MEDIA_ROOT = 'C:/Users/dev/project/media'  # Not portable
```

✅ **Correct:**
```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# urls.py — serve media in development only
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ...
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

> 💡 **Why:** `MEDIA_ROOT` stores user-uploaded files. Serve them via Django only in development — in production, use Nginx or a CDN. Use relative paths for portability.

### collectstatic Production Workflow

❌ **Wrong:**
```bash
# Committing collected static files to git
git add staticfiles/
# Or running collectstatic on every request
```

✅ **Correct:**
```bash
# Run during deployment, not in git
python manage.py collectstatic --noinput

# .gitignore
staticfiles/
```

```python
# settings/production.py
STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.ManifestStaticFilesStorage'
# Adds content hashes to filenames for cache busting:
# style.css -> style.abc123.css
```

> 💡 **Why:** `collectstatic` gathers static files from all apps into `STATIC_ROOT` for the web server. `ManifestStaticFilesStorage` adds content hashes for browser cache invalidation.

### WhiteNoise

❌ **Wrong:**
```python
# Using Django's development server to serve static files in production
# Or configuring Nginx just for static files on a small app
```

✅ **Correct:**
```python
# pip install whitenoise

# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Right after SecurityMiddleware
    # ...
]

STORAGES = {
    'staticfiles': {
        'BACKEND': 'whitenoise.storage.CompressedManifestStaticFilesStorage',
    },
}
```

> 💡 **Why:** WhiteNoise serves static files directly from Python with compression and caching headers. Perfect for Heroku, Docker, and small-to-medium apps. No Nginx needed for static files.

### S3/CDN Integration

❌ **Wrong:**
```python
# Writing custom S3 upload code with boto3 in every view
import boto3

def upload_file(request):
    s3 = boto3.client('s3', aws_access_key_id='AKIA...',
                      aws_secret_access_key='secret...')
    s3.upload_fileobj(request.FILES['file'], 'my-bucket', 'path/file.jpg')
```

✅ **Correct:**
```python
# pip install django-storages boto3

# settings/production.py
STORAGES = {
    'default': {
        'BACKEND': 'storages.backends.s3boto3.S3Boto3Storage',
    },
    'staticfiles': {
        'BACKEND': 'storages.backends.s3boto3.S3StaticStorage',
    },
}

AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
AWS_STORAGE_BUCKET_NAME = 'my-bucket'
AWS_S3_REGION_NAME = 'us-east-1'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
AWS_DEFAULT_ACL = None  # Use bucket policy
AWS_S3_OBJECT_PARAMETERS = {'CacheControl': 'max-age=86400'}

# Models work unchanged — FileField automatically uploads to S3
```

> 💡 **Why:** django-storages abstracts the storage backend. Your models and forms don't change — `FileField.save()` and `default_storage.open()` work the same whether it's local or S3.

### Development vs Production Differences

❌ **Wrong:**
```python
# Same static file configuration for development and production
# Running collectstatic in development
# Serving media files through Nginx in development
```

✅ **Correct:**
```python
# settings/local.py
DEBUG = True
STATIC_URL = '/static/'
MEDIA_URL = '/media/'
# Django's runserver handles static files automatically when DEBUG=True

# settings/production.py
DEBUG = False
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'

STORAGES = {
    'default': {
        'BACKEND': 'storages.backends.s3boto3.S3Boto3Storage',
    },
    'staticfiles': {
        'BACKEND': 'whitenoise.storage.CompressedManifestStaticFilesStorage',
    },
}
```

> 💡 **Why:** Development uses Django's built-in static serving. Production uses WhiteNoise, Nginx, or S3/CDN. Never use `runserver` or DEBUG=True in production.

## 14. Django Security

### CSRF Protection

❌ **Wrong:**
```python
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # Disabling CSRF on a state-changing view
def update_profile(request):
    request.user.name = request.POST['name']
    request.user.save()
    return JsonResponse({'status': 'ok'})
```

✅ **Correct:**
```python
# Regular forms — just include the token
# template:
# <form method="post">{% csrf_token %}...</form>

# AJAX with fetch
from django.views.decorators.http import require_POST

@require_POST
def update_profile(request):
    request.user.name = request.POST['name']
    request.user.save()
    return JsonResponse({'status': 'ok'})
```

```javascript
// JavaScript — read CSRF token from cookie
function getCookie(name) {
    const value = document.cookie.match('(^|;)\\s*' + name + '\\s*=\\s*([^;]+)');
    return value ? value.pop() : '';
}
fetch('/api/update-profile/', {
    method: 'POST',
    headers: {
        'X-CSRFToken': getCookie('csrftoken'),
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({name: 'New Name'}),
});
```

> 💡 **Why:** Only use `@csrf_exempt` for external webhooks that have their own authentication (Stripe, GitHub). Every other POST/PUT/DELETE needs CSRF protection.

### XSS Protection

❌ **Wrong:**
```python
from django.utils.safestring import mark_safe

def render_comment(comment):
    # Marking user input as safe — XSS vulnerability
    return mark_safe(f'<div class="comment">{comment.body}</div>')
```

✅ **Correct:**
```python
from django.utils.html import format_html


def render_comment(comment):
    # format_html escapes the variable parts while keeping the HTML structure
    return format_html('<div class="comment">{}</div>', comment.body)

# In templates — autoescaping is on by default
# {{ comment.body }}  ← Escaped automatically
# {{ comment.body|escape }}  ← Explicitly escaped (same as default)
```

> 💡 **Why:** Django autoescapes template variables by default. Use `format_html()` in Python code to safely build HTML. Never use `mark_safe()` on user input.

### SQL Injection

❌ **Wrong:**
```python
from django.db import connection

def search(request):
    query = request.GET['q']
    cursor = connection.cursor()
    cursor.execute(f"SELECT * FROM products WHERE name = '{query}'")
    # SQL injection — user can input: ' OR 1=1; DROP TABLE products; --
```

✅ **Correct:**
```python
from django.db import connection
from myapp.models import Product


def search(request):
    query = request.GET.get('q', '')
    # ORM is always safe
    products = Product.objects.filter(name__icontains=query)

    # Raw SQL — use parameterized queries
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT * FROM products_product WHERE name LIKE %s",
            [f'%{query}%']
        )
```

> 💡 **Why:** The ORM parameterizes all queries automatically. For raw SQL, always use `%s` placeholders and pass parameters as a list — never use f-strings or `.format()`.

### Clickjacking Protection

❌ **Wrong:**
```python
# No X-Frame-Options — site can be embedded in iframes (clickjacking)
MIDDLEWARE = [
    # XFrameOptionsMiddleware missing
]
```

✅ **Correct:**
```python
# settings.py
MIDDLEWARE = [
    # ...
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

X_FRAME_OPTIONS = 'DENY'  # Or 'SAMEORIGIN' if you embed your own pages

# For specific views that need to be embeddable
from django.views.decorators.clickjacking import xframe_options_exempt

@xframe_options_exempt
def embeddable_widget(request):
    return render(request, 'widget.html')
```

> 💡 **Why:** `X-Frame-Options: DENY` prevents your pages from being embedded in iframes, blocking clickjacking attacks. Use `SAMEORIGIN` if you embed your own content.

### Password Hashing

❌ **Wrong:**
```python
# Using MD5 or SHA1 for passwords
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()
# Or using the default PBKDF2 when Argon2 is available
```

✅ **Correct:**
```python
# pip install argon2-cffi

# settings.py
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',  # Preferred
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',  # Fallback for existing passwords
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
]

AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
     'OPTIONS': {'min_length': 10}},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]
```

> 💡 **Why:** Argon2 is the winner of the Password Hashing Competition — memory-hard and resistant to GPU attacks. List PBKDF2 as fallback so existing passwords still verify during migration.

### HTTPS Settings

❌ **Wrong:**
```python
# Production without HTTPS enforcement
DEBUG = False
# No HSTS, no SSL redirect, cookies sent over HTTP
```

✅ **Correct:**
```python
# settings/production.py
SECURE_SSL_REDIRECT = True  # Redirect HTTP to HTTPS
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')  # Behind reverse proxy

SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

> 💡 **Why:** HSTS tells browsers to always use HTTPS. `SECURE_SSL_REDIRECT` catches HTTP requests. `SECURE_PROXY_SSL_HEADER` is needed when Nginx/ALB handles TLS termination.

### SECRET_KEY Management

❌ **Wrong:**
```python
# settings.py — committed to git
SECRET_KEY = 'django-insecure-abc123456789realkey'
```

✅ **Correct:**
```python
import os

# Read from environment variable
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']

# Generate a new key:
# python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

> 💡 **Why:** SECRET_KEY is used for signing cookies, tokens, and password reset links. If leaked, attackers can forge sessions. Store it in environment variables or a secrets manager.

### DEBUG=False Production Checks

❌ **Wrong:**
```python
# Deploying without running Django's deployment checks
# DEBUG = True in production — exposes full tracebacks, settings, SQL queries
```

✅ **Correct:**
```bash
# Run before every deployment
python manage.py check --deploy
```

```python
# settings/production.py
DEBUG = False
ALLOWED_HOSTS = ['example.com', 'www.example.com']

# Configure error reporting
ADMINS = [('Admin', 'admin@example.com')]
SERVER_EMAIL = 'errors@example.com'

LOGGING = {
    'version': 1,
    'handlers': {
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['mail_admins'],
            'level': 'ERROR',
        },
    },
}
```

> 💡 **Why:** `manage.py check --deploy` catches common security misconfigurations. `ALLOWED_HOSTS` prevents HTTP Host header attacks. Set up error logging so you see exceptions without DEBUG.

### Security Middleware Order

❌ **Wrong:**
```python
# SecurityMiddleware buried in the middle or missing entirely
MIDDLEWARE = [
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',  # Too late
]
```

✅ **Correct:**
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',  # FIRST — sets security headers
    'corsheaders.middleware.CorsMiddleware',  # Before CommonMiddleware
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

> 💡 **Why:** SecurityMiddleware must be first to set security headers (HSTS, SSL redirect) before any other processing. CorsMiddleware must come before CommonMiddleware to handle preflight requests.

## 15. Django Signals

### pre_save / post_save

❌ **Wrong:**
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

✅ **Correct:**
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

> 💡 **Why:** Signals must be imported to be connected. The `ready()` method in `AppConfig` is the canonical place. `dispatch_uid` prevents duplicate connections.

### pre_delete / post_delete

❌ **Wrong:**
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

✅ **Correct:**
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

> 💡 **Why:** `post_delete` runs after the DB row is gone — use it for file cleanup. `pre_delete` runs before deletion — use it for logging. Always use `save=False` in post_delete to avoid saving a deleted instance.

### m2m_changed

❌ **Wrong:**
```python
# Trying to use post_save to detect ManyToMany changes
from django.db.models.signals import post_save
from .models import Article

@receiver(post_save, sender=Article)
def handle_tags(sender, instance, **kwargs):
    # M2M changes don't trigger post_save — this never fires for tag changes
    print(instance.tags.all())
```

✅ **Correct:**
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

> 💡 **Why:** M2M changes fire `m2m_changed`, not `post_save`. The `action` parameter tells you what happened: `pre_add`, `post_add`, `pre_remove`, `post_remove`, `pre_clear`, `post_clear`.

### request_started / request_finished

❌ **Wrong:**
```python
# Using request signals for per-request logic when middleware is more appropriate
from django.core.signals import request_started

@receiver(request_started)
def on_request(sender, **kwargs):
    # request_started doesn't give you the request object!
    # You can't access request.user, request.path, etc. here
    print('Request started')
```

✅ **Correct:**
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

> 💡 **Why:** `request_started`/`request_finished` don't provide the request object. Use middleware for request/response processing. Use these signals only for low-level hooks like resource cleanup.

### Custom Signals

❌ **Wrong:**
```python
from django.dispatch import Signal

# Old-style with providing_args (deprecated in Django 4.0, removed in 5.0)
order_completed = Signal(providing_args=['order', 'user'])
```

✅ **Correct:**
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

> 💡 **Why:** Custom signals decouple components — the order module doesn't need to know about email sending. `providing_args` was removed in Django 5.0; just document expected kwargs.

### When to Avoid Signals

❌ **Wrong:**
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

✅ **Correct:**
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

> 💡 **Why:** Signals are invisible control flow — hard to debug and test. Use them for decoupled cross-app communication. Use explicit method calls for tightly coupled logic within the same app.

### Importing Signals in apps.py ready()

❌ **Wrong:**
```python
# Importing signals at module level in models.py
# models.py
from . import signals  # Circular import risk, loaded too early
```

✅ **Correct:**
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

> 💡 **Why:** `ready()` runs after all apps are loaded, avoiding circular imports. This is the Django-recommended way to connect signals.

## 16. Caching

### Cache Framework Configuration

❌ **Wrong:**
```python
# No cache configuration — every cache.get() silently returns None
# Or using LocMemCache in production (not shared between workers)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
```

✅ **Correct:**
```python
# settings/local.py — dummy cache for development
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}

# settings/production.py — real cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ['REDIS_URL'],
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
    }
}
```

> 💡 **Why:** `DummyCache` in development means caching never hides bugs. `LocMemCache` is per-process — useless with multiple Gunicorn workers. Use Redis or Memcached in production.

### Redis Cache Backend

❌ **Wrong:**
```python
# Using the built-in Redis backend without connection pooling
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379',
        # No timeout, no key prefix, no error handling
    }
}
```

✅ **Correct:**
```python
# pip install django-redis

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/0'),
        'TIMEOUT': 300,  # Default 5 minutes
        'KEY_PREFIX': 'myapp',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'IGNORE_EXCEPTIONS': True,  # Cache failures don't crash the app
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
    }
}
```

> 💡 **Why:** django-redis provides connection pooling, key prefixing, and error handling. `IGNORE_EXCEPTIONS=True` means cache failures degrade gracefully instead of crashing.

### Memcached

❌ **Wrong:**
```python
# Using Memcached without understanding its limitations
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}
# Then storing objects larger than 1MB or expecting persistence
```

✅ **Correct:**
```python
# pip install pymemcache

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
        'OPTIONS': {
            'no_delay': True,
            'connect_timeout': 3,
            'timeout': 3,
        },
    }
}

# Use Redis instead of Memcached when you need:
# - Persistence, data structures (lists, sets), pub/sub, or values > 1MB
```

> 💡 **Why:** Memcached is fast and simple but has a 1MB value limit, no persistence, and limited data types. Redis is more versatile. Use Memcached only if you already have the infrastructure.

### Per-View Cache

❌ **Wrong:**
```python
from django.views.decorators.cache import cache_page

# Caching a view that depends on the current user
@cache_page(60 * 15)
def dashboard(request):
    return render(request, 'dashboard.html', {
        'user': request.user,  # All users see the same cached page!
    })
```

✅ **Correct:**
```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie


@cache_page(60 * 15)  # Cache for 15 minutes
def product_list(request):
    # Public page — safe to cache for all users
    products = Product.objects.filter(is_active=True)
    return render(request, 'products/list.html', {'products': products})


@cache_page(60 * 5)
@vary_on_cookie  # Different cache per session/user
def dashboard(request):
    return render(request, 'dashboard.html', {'user': request.user})
```

> 💡 **Why:** `@cache_page` caches the entire response. Use it for public pages. For user-specific pages, add `@vary_on_cookie` so each user gets their own cache entry.

### Template Fragment Cache

❌ **Wrong:**
```html
<!-- Caching a fragment that includes user-specific content -->
{% load cache %}
{% cache 500 sidebar %}
  <div>Welcome, {{ user.username }}</div>  <!-- Same for all users! -->
  {% for article in recent_articles %}
    <a href="{{ article.url }}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}
```

✅ **Correct:**
```html
{% load cache %}

<!-- Cache per user using a vary argument -->
{% cache 300 sidebar user.pk %}
  <div>Welcome, {{ user.username }}</div>
  {% for article in user_articles %}
    <a href="{{ article.url }}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}

<!-- Or cache only the expensive, non-personalized part -->
{% cache 600 recent_articles %}
  {% for article in recent_articles %}
    <a href="{% url 'articles:detail' pk=article.pk %}">{{ article.title }}</a>
  {% endfor %}
{% endcache %}
```

> 💡 **Why:** Template fragment caching is more granular than view caching. Pass a varying argument (like `user.pk`) to create per-user cache entries. Cache only expensive, non-personalized fragments.

### Low-Level Cache API

❌ **Wrong:**
```python
from django.core.cache import cache

def get_product(pk):
    product = cache.get(pk)  # Generic key — collides with other cached objects
    if not product:
        product = Product.objects.get(pk=pk)
        cache.set(pk, product)  # No timeout — cached forever
    return product
```

✅ **Correct:**
```python
from django.core.cache import cache
from .models import Product


def get_product(pk):
    cache_key = f'product:{pk}'
    product = cache.get(cache_key)
    if product is None:
        product = Product.objects.get(pk=pk)
        cache.set(cache_key, product, timeout=300)  # 5 minutes
    return product


# get_or_set — shortcut for the pattern above
def get_product_v2(pk):
    return cache.get_or_set(
        f'product:{pk}',
        lambda: Product.objects.get(pk=pk),
        timeout=300,
    )


# Cache with versioning
cache.set('product:1', data, version=2)
cache.get('product:1', version=2)
```

> 💡 **Why:** Use namespaced cache keys (`product:1`) to avoid collisions. Always set a timeout. `get_or_set` is a convenient shortcut for the cache-aside pattern.

### Cache Invalidation Strategies

❌ **Wrong:**
```python
# Never invalidating — stale data served forever
cache.set('product_list', products)  # Never updated when products change

# Or invalidating everything on any change
cache.clear()  # Nukes the entire cache
```

✅ **Correct:**
```python
from django.core.cache import cache
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from .models import Product


@receiver([post_save, post_delete], sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.pk}')
    cache.delete('product_list')  # Invalidate the list too


# Or use cache versioning for bulk invalidation
def get_product_list():
    version = cache.get('product_list_version', 1)
    return cache.get_or_set(
        'product_list',
        lambda: list(Product.objects.all()),
        timeout=600,
        version=version,
    )

def invalidate_product_list():
    cache.incr('product_list_version')  # Old version naturally expires
```

> 💡 **Why:** Targeted invalidation deletes only affected keys. Version-based invalidation avoids thundering herd — old entries expire naturally while new ones are created.

### Cache Key Design

❌ **Wrong:**
```python
# Ambiguous keys that collide across models/apps
cache.set('list', products)
cache.set('detail', product)
cache.set(str(pk), product)
```

✅ **Correct:**
```python
from django.core.cache import cache


def make_cache_key(model_name, identifier, suffix=''):
    """Consistent cache key format: app:model:id:suffix"""
    key = f'myapp:{model_name}:{identifier}'
    if suffix:
        key += f':{suffix}'
    return key


# Usage
cache.set(make_cache_key('product', pk), product, timeout=300)
cache.set(make_cache_key('product', 'list', 'page:1'), page_data, timeout=60)
cache.set(make_cache_key('user', user_id, 'dashboard'), dashboard_data, timeout=120)
```

> 💡 **Why:** Structured cache keys prevent collisions, make debugging easier, and allow pattern-based invalidation. Include the app/model name, identifier, and any variant suffixes.

## 17. File Handling

### FileField and ImageField

❌ **Wrong:**
```python
from django.db import models

class Document(models.Model):
    file = models.FileField()  # Uploads to MEDIA_ROOT root — messy
    # No upload_to, no size validation
```

✅ **Correct:**
```python
import uuid
from django.db import models
from django.core.validators import FileExtensionValidator


def document_upload_path(instance, filename):
    ext = filename.split('.')[-1]
    return f'documents/{instance.user.pk}/{uuid.uuid4().hex}.{ext}'


class Document(models.Model):
    user = models.ForeignKey('users.User', on_delete=models.CASCADE)
    file = models.FileField(
        upload_to=document_upload_path,
        validators=[FileExtensionValidator(allowed_extensions=['pdf', 'docx', 'txt'])],
    )
    uploaded_at = models.DateTimeField(auto_now_add=True)


class UserAvatar(models.Model):
    user = models.OneToOneField('users.User', on_delete=models.CASCADE)
    image = models.ImageField(upload_to='avatars/%Y/%m/', blank=True)
    # ImageField requires Pillow: pip install Pillow
```

> 💡 **Why:** Use callable `upload_to` for dynamic paths (prevents filename collisions with UUID). Validate extensions at the model level. ImageField requires Pillow to be installed.

### Storage API

❌ **Wrong:**
```python
# Hardcoding file paths and using Python's open() directly
import os

def save_file(uploaded_file):
    path = f'/var/www/media/docs/{uploaded_file.name}'
    with open(path, 'wb') as f:
        for chunk in uploaded_file.chunks():
            f.write(chunk)
```

✅ **Correct:**
```python
from django.core.files.storage import default_storage
from django.core.files.base import ContentFile


def save_file(uploaded_file):
    path = default_storage.save(f'docs/{uploaded_file.name}', uploaded_file)
    return default_storage.url(path)


def read_file(path):
    if default_storage.exists(path):
        with default_storage.open(path, 'rb') as f:
            return f.read()


def delete_file(path):
    if default_storage.exists(path):
        default_storage.delete(path)
```

> 💡 **Why:** `default_storage` abstracts the backend — works with local filesystem, S3, GCS, or any custom storage. Switching backends doesn't require code changes.

### Custom Storage Backends

❌ **Wrong:**
```python
# Writing boto3 upload code directly in views
import boto3

def upload_view(request):
    s3 = boto3.client('s3')
    s3.upload_fileobj(request.FILES['file'], 'bucket', 'key')
```

✅ **Correct:**
```python
# Use django-storages for S3, GCS, Azure
# pip install django-storages boto3

# settings.py
STORAGES = {
    'default': {
        'BACKEND': 'storages.backends.s3boto3.S3Boto3Storage',
    },
}

# For multiple storage backends on the same project
from storages.backends.s3boto3 import S3Boto3Storage

class PrivateMediaStorage(S3Boto3Storage):
    bucket_name = 'my-private-bucket'
    default_acl = 'private'
    file_overwrite = False
    querystring_auth = True  # Signed URLs

class Document(models.Model):
    file = models.FileField(storage=PrivateMediaStorage())
```

> 💡 **Why:** django-storages integrates with Django's storage API. Custom storage classes let you use different backends (public vs private) on the same project without changing model code.

### File Upload Security

❌ **Wrong:**
```python
def upload(request):
    f = request.FILES['file']
    # No type checking — user can upload .exe, .sh, etc.
    # No size checking — user can upload 10 GB files
    f.save('/uploads/' + f.name)  # Path traversal: name could be "../../../etc/passwd"
```

✅ **Correct:**
```python
from django import forms
from django.core.validators import FileExtensionValidator


class SecureUploadForm(forms.Form):
    file = forms.FileField(
        validators=[FileExtensionValidator(allowed_extensions=['pdf', 'jpg', 'png'])],
    )

    def clean_file(self):
        f = self.cleaned_data['file']

        # Size check
        if f.size > 5 * 1024 * 1024:  # 5 MB
            raise forms.ValidationError('File too large. Max 5 MB.')

        # MIME type verification (don't trust Content-Type header)
        import magic
        mime = magic.from_buffer(f.read(2048), mime=True)
        f.seek(0)
        allowed = {'application/pdf', 'image/jpeg', 'image/png'}
        if mime not in allowed:
            raise forms.ValidationError('Invalid file type.')

        return f
```

```python
# settings.py — limit upload size at the server level
DATA_UPLOAD_MAX_MEMORY_SIZE = 5 * 1024 * 1024  # 5 MB
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440  # 2.5 MB — above this, writes to temp file
```

> 💡 **Why:** Validate file extension, size, and actual MIME type (via magic bytes). Never trust the filename or Content-Type header. Set server-level size limits as a safety net.

### Pillow Usage

❌ **Wrong:**
```python
from django.db import models

class Photo(models.Model):
    image = models.ImageField(upload_to='photos/')
    # Pillow not installed — ImageField won't validate image files
    # No image processing — uploaded 10 MB photos served as-is
```

✅ **Correct:**
```python
# pip install Pillow

from django.db import models
from PIL import Image as PILImage
from io import BytesIO
from django.core.files.base import ContentFile


class Photo(models.Model):
    image = models.ImageField(upload_to='photos/%Y/%m/')
    thumbnail = models.ImageField(upload_to='photos/thumbs/', blank=True)

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        if self.image and not self.thumbnail:
            self._create_thumbnail()

    def _create_thumbnail(self):
        img = PILImage.open(self.image)
        img.thumbnail((300, 300))
        buffer = BytesIO()
        img.save(buffer, format='JPEG', quality=85)
        self.thumbnail.save(
            f'thumb_{self.image.name.split("/")[-1]}',
            ContentFile(buffer.getvalue()),
            save=True,
        )
```

> 💡 **Why:** Pillow is required for `ImageField` validation. Use it to create thumbnails and optimize images on upload instead of serving raw multi-megabyte images.

## 18. Internationalization (i18n)

### i18n Settings

❌ **Wrong:**
```python
# Ignoring timezone support
USE_I18N = False
USE_TZ = False  # Storing naive datetimes — breaks when users span timezones
```

✅ **Correct:**
```python
# settings.py
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'

USE_I18N = True   # Enable translation system
USE_L10N = True   # Locale-aware date/number formatting (deprecated in Django 5.0, on by default)
USE_TZ = True     # Store datetimes as UTC in database

LANGUAGES = [
    ('en', 'English'),
    ('es', 'Spanish'),
    ('fr', 'French'),
]

LOCALE_PATHS = [
    BASE_DIR / 'locale',
]
```

> 💡 **Why:** `USE_TZ=True` stores all datetimes as UTC in the database and converts to local time for display. This is essential for any app with users in multiple timezones.

### gettext and gettext_lazy

❌ **Wrong:**
```python
from django.utils.translation import gettext as _

class Product(models.Model):
    # gettext (non-lazy) in model definition — evaluated at import time, before language is set
    name = models.CharField(_('product name'), max_length=255)
```

✅ **Correct:**
```python
from django.utils.translation import gettext_lazy as _

class Product(models.Model):
    # gettext_lazy — evaluated when accessed, not at import time
    name = models.CharField(_('product name'), max_length=255)
    description = models.TextField(_('description'), blank=True)

    class Meta:
        verbose_name = _('product')
        verbose_name_plural = _('products')


# In views — use gettext (non-lazy) since the language is known at request time
from django.utils.translation import gettext as _

def my_view(request):
    message = _('Welcome to our store')
    return render(request, 'home.html', {'message': message})
```

> 💡 **Why:** Use `gettext_lazy` (`_()` from `gettext_lazy`) in models, forms, and any code evaluated at import time. Use `gettext` in views and functions called at request time.

### makemessages / compilemessages

❌ **Wrong:**
```bash
# Forgetting to run makemessages after adding new translatable strings
# Or running compilemessages without .po files
python manage.py compilemessages  # Error: no .po files found
```

✅ **Correct:**
```bash
# Create locale directory structure
mkdir -p locale/es/LC_MESSAGES locale/fr/LC_MESSAGES

# Extract translatable strings from Python and templates
python manage.py makemessages -l es -l fr --no-wrap

# Edit the .po files:
# locale/es/LC_MESSAGES/django.po
# msgid "Welcome to our store"
# msgstr "Bienvenido a nuestra tienda"

# Compile .po to .mo (binary format Django reads)
python manage.py compilemessages
```

> 💡 **Why:** `makemessages` scans code for `_()` and `{% trans %}` strings. `compilemessages` converts human-readable `.po` files to binary `.mo` files. Run both as part of your deployment process.

### Template Translation Tags

❌ **Wrong:**
```html
<!-- Hardcoded strings in templates -->
<h1>Welcome</h1>
<p>You have {{ count }} items in your cart.</p>
```

✅ **Correct:**
```html
{% load i18n %}

<h1>{% trans "Welcome" %}</h1>

<!-- Simple translation with variable -->
<p>{% trans "Shopping Cart" %}</p>

<!-- Translation with variables — use blocktrans -->
{% blocktrans count counter=count %}
  You have {{ counter }} item in your cart.
{% plural %}
  You have {{ counter }} items in your cart.
{% endblocktrans %}

<!-- Django 4.0+: blocktranslate (newer name) -->
{% blocktranslate with name=user.name %}
  Hello, {{ name }}!
{% endblocktranslate %}
```

> 💡 **Why:** `{% trans %}` for simple strings. `{% blocktrans %}` for strings with variables or pluralization. Always load `{% load i18n %}` at the top of templates.

### Timezone Support

❌ **Wrong:**
```python
from datetime import datetime

# Naive datetime — no timezone info
now = datetime.now()
event.start_time = now  # Django will warn about naive datetimes with USE_TZ=True
```

✅ **Correct:**
```python
from django.utils import timezone

# Always use timezone.now() instead of datetime.now()
now = timezone.now()  # Returns UTC-aware datetime

# Activate a timezone for display
from django.utils import timezone as tz

def set_user_timezone(request):
    user_tz = request.user.profile.timezone  # e.g., 'America/New_York'
    tz.activate(user_tz)

# Convert for display
from django.utils.timezone import localtime
local_dt = localtime(event.start_time)  # Converts to active timezone

# In templates — automatic conversion
# {{ event.start_time }}  ← Converted to active timezone if USE_TZ=True
```

> 💡 **Why:** `timezone.now()` returns UTC-aware datetimes. Django stores UTC in the database and converts to local time using the activated timezone for display.

### Locale File Structure

❌ **Wrong:**
```
# Locale files scattered around the project
myapp/locale/es/django.po
templates/locale/fr/django.po
# Inconsistent structure, missing LC_MESSAGES directories
```

✅ **Correct:**
```
project/
    locale/                    # Project-level translations
        es/
            LC_MESSAGES/
                django.po
                django.mo
        fr/
            LC_MESSAGES/
                django.po
                django.mo
    apps/
        orders/
            locale/            # App-level translations (optional)
                es/
                    LC_MESSAGES/
                        django.po
                        django.mo
```

```python
# settings.py
LOCALE_PATHS = [
    BASE_DIR / 'locale',  # Project-level — checked first
]
# App-level locale/ dirs are found automatically
```

> 💡 **Why:** Keep project-wide translations in a top-level `locale/` directory. App-level translations live in `<app>/locale/`. Django checks `LOCALE_PATHS` first, then app directories.

## 19. Testing

### TestCase vs SimpleTestCase vs TransactionTestCase

❌ **Wrong:**
```python
from django.test import TestCase

# Using TestCase for tests that don't need a database
class UtilsTest(TestCase):  # Unnecessary DB setup/teardown overhead
    def test_format_currency(self):
        self.assertEqual(format_currency(1000), '$10.00')
```

✅ **Correct:**
```python
from django.test import SimpleTestCase, TestCase, TransactionTestCase


# SimpleTestCase — no database access, fastest
class UtilsTest(SimpleTestCase):
    def test_format_currency(self):
        from myapp.utils import format_currency
        self.assertEqual(format_currency(1000), '$10.00')


# TestCase — wraps each test in a transaction (rolled back after), uses DB
class ProductModelTest(TestCase):
    def test_str(self):
        from myapp.models import Product
        product = Product.objects.create(name='Widget', price=9.99)
        self.assertEqual(str(product), 'Widget')


# TransactionTestCase — actually commits to DB, needed for testing transactions
class TransferTest(TransactionTestCase):
    def test_atomic_transfer(self):
        # Test code that uses transaction.atomic() or select_for_update()
        ...
```

> 💡 **Why:** `SimpleTestCase` is fastest (no DB). `TestCase` wraps each test in a transaction for isolation. `TransactionTestCase` commits for real — needed when testing transaction behavior.

### Client and RequestFactory

❌ **Wrong:**
```python
from django.test import TestCase

class ViewTest(TestCase):
    def test_product_list(self):
        # Using Client for a test that only needs to test view logic, not middleware
        response = self.client.get('/products/')
        # Client runs the full middleware stack — slow for unit tests
```

✅ **Correct:**
```python
from django.test import TestCase, RequestFactory
from django.contrib.auth import get_user_model
from myapp.views import product_list

User = get_user_model()


class ViewTest(TestCase):
    def setUp(self):
        self.factory = RequestFactory()
        self.user = User.objects.create_user(username='test', password='test')

    def test_product_list_with_client(self):
        # Client — full integration test (middleware, URL routing, templates)
        self.client.login(username='test', password='test')
        response = self.client.get('/products/')
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Products')

    def test_product_list_with_factory(self):
        # RequestFactory — unit test the view function directly
        request = self.factory.get('/products/')
        request.user = self.user
        response = product_list(request)
        self.assertEqual(response.status_code, 200)
```

> 💡 **Why:** `Client` tests the full stack (middleware, URL resolution, templates). `RequestFactory` creates bare request objects for testing views in isolation — faster for unit tests.

### Fixtures

❌ **Wrong:**
```python
from django.test import TestCase

class OrderTest(TestCase):
    fixtures = ['users.json', 'products.json', 'orders.json']
    # JSON fixtures are hard to maintain, brittle, and don't evolve with model changes
    # They also create tight coupling between tests and specific data
```

✅ **Correct:**
```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from myapp.models import Product, Order

User = get_user_model()


class OrderTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Created once for the whole TestCase — fast
        cls.user = User.objects.create_user(username='buyer', password='test')
        cls.product = Product.objects.create(name='Widget', price=9.99)

    def test_create_order(self):
        order = Order.objects.create(user=self.user, product=self.product)
        self.assertEqual(order.user, self.user)
```

> 💡 **Why:** `setUpTestData` creates test data once per TestCase class (not per test method), making it fast. JSON fixtures are fragile — they break when models change and are hard to read.

### pytest-django

❌ **Wrong:**
```python
# Using unittest-style assertions with pytest
import pytest

@pytest.mark.django_db
class TestProduct:
    def test_create(self):
        from myapp.models import Product
        p = Product.objects.create(name='X', price=1)
        assert p is not None  # Too vague
```

✅ **Correct:**
```python
# conftest.py
import pytest
from django.contrib.auth import get_user_model

User = get_user_model()


@pytest.fixture
def user(db):
    return User.objects.create_user(username='testuser', password='testpass')


@pytest.fixture
def auth_client(client, user):
    client.login(username='testuser', password='testpass')
    return client


# tests/test_views.py
import pytest
from myapp.models import Product


@pytest.mark.django_db
def test_product_list(auth_client):
    Product.objects.create(name='Widget', price=9.99)
    response = auth_client.get('/products/')
    assert response.status_code == 200
    assert b'Widget' in response.content


@pytest.mark.django_db
def test_product_create(auth_client):
    response = auth_client.post('/products/create/', {'name': 'Gadget', 'price': '19.99'})
    assert response.status_code == 302
    assert Product.objects.filter(name='Gadget').exists()
```

> 💡 **Why:** pytest-django provides `db`, `client`, `rf` (RequestFactory), and `admin_client` fixtures. Use `@pytest.mark.django_db` to enable DB access. Fixtures compose naturally.

### FactoryBoy

❌ **Wrong:**
```python
# Creating test data manually in every test
def test_order_total(self):
    user = User.objects.create_user(username='u1', password='p')
    product1 = Product.objects.create(name='A', price=10)
    product2 = Product.objects.create(name='B', price=20)
    order = Order.objects.create(user=user)
    OrderItem.objects.create(order=order, product=product1, quantity=2)
    OrderItem.objects.create(order=order, product=product2, quantity=1)
    # 6 lines just for setup — repeated in every test
```

✅ **Correct:**
```python
# pip install factory-boy

# factories.py
import factory
from django.contrib.auth import get_user_model
from myapp.models import Product, Order, OrderItem

User = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda o: f'{o.username}@example.com')


class ProductFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Faker('word')
    price = factory.Faker('pydecimal', left_digits=3, right_digits=2, positive=True)


class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)


class OrderItemFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = factory.Faker('random_int', min=1, max=5)


# Usage in tests
def test_order_total():
    order = OrderFactory()
    OrderItemFactory(order=order, product__price=10, quantity=2)
    OrderItemFactory(order=order, product__price=20, quantity=1)
    assert order.calculate_total() == 40
```

> 💡 **Why:** Factories create realistic test data with sensible defaults. `SubFactory` handles relationships. Override specific fields when they matter for the test.

### Mock Usage

❌ **Wrong:**
```python
# Testing with real external services
def test_send_notification(self):
    send_sms('+1234567890', 'Hello')  # Actually sends an SMS during tests!
```

✅ **Correct:**
```python
from unittest.mock import patch, MagicMock
from django.test import TestCase, override_settings
from myapp.services import process_order


class OrderServiceTest(TestCase):
    @patch('myapp.services.send_sms')
    @patch('myapp.services.charge_payment')
    def test_process_order(self, mock_charge, mock_sms):
        mock_charge.return_value = {'status': 'success', 'charge_id': 'ch_123'}

        order = OrderFactory(total=100)
        process_order(order)

        mock_charge.assert_called_once_with(amount=100, currency='usd')
        mock_sms.assert_called_once()
        order.refresh_from_db()
        self.assertEqual(order.status, 'completed')

    @override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')
    def test_sends_confirmation_email(self):
        from django.core import mail
        order = OrderFactory()
        process_order(order)
        self.assertEqual(len(mail.outbox), 1)
        self.assertIn('confirmation', mail.outbox[0].subject.lower())
```

> 💡 **Why:** Mock external services (SMS, payments, APIs) to keep tests fast and deterministic. Use Django's `locmem` email backend to capture emails in tests.

### Test Coverage and Targets

❌ **Wrong:**
```bash
# Running tests without coverage measurement
python manage.py test
# No idea which code paths are untested
```

✅ **Correct:**
```bash
# pip install coverage pytest-cov

# With pytest
pytest --cov=apps --cov-report=html --cov-report=term-missing

# With Django's test runner
coverage run manage.py test
coverage report -m
coverage html  # Open htmlcov/index.html
```

```ini
# setup.cfg or pyproject.toml
[tool:pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
addopts = --cov=apps --cov-report=term-missing --cov-fail-under=80

[coverage:run]
omit =
    */migrations/*
    */tests/*
    manage.py
```

> 💡 **Why:** Aim for 80%+ coverage on business logic. Don't chase 100% — focus on critical paths (payments, auth, data mutations). Exclude migrations and test files from coverage reports.

### Test Organization

❌ **Wrong:**
```python
# All tests in a single tests.py file per app
# apps/orders/tests.py — 2000 lines, mixing unit/integration/e2e tests
```

✅ **Correct:**
```
apps/orders/
    tests/
        __init__.py
        test_models.py          # Unit tests — model methods, validation
        test_views.py           # Integration tests — request/response cycle
        test_services.py        # Unit tests — business logic
        test_api.py             # API endpoint tests
        test_integration.py     # Cross-app integration tests
        factories.py            # Test data factories
        conftest.py             # Shared fixtures
```

```python
# test_models.py — focused on model behavior
from django.test import TestCase
from .factories import OrderFactory


class OrderModelTest(TestCase):
    def test_calculate_total(self):
        ...

    def test_cannot_ship_unpaid_order(self):
        ...
```

> 💡 **Why:** Split tests by scope and layer. Small test files are easier to navigate and run selectively (`pytest tests/test_models.py`). Shared factories live in a dedicated file.

## 20. Django Rest Framework (DRF)

### Serializers

❌ **Wrong:**
```python
from rest_framework import serializers

class ProductSerializer(serializers.Serializer):
    # Manually defining every field — ignores the model definition
    id = serializers.IntegerField()
    name = serializers.CharField()
    price = serializers.FloatField()  # FloatField for money — rounding issues
    # Missing validation, missing many fields
```

✅ **Correct:**
```python
from rest_framework import serializers
from .models import Product


class ProductSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name', read_only=True)
    discounted_price = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category', 'category_name',
                  'discounted_price', 'is_active']
        read_only_fields = ['id']

    def get_discounted_price(self, obj):
        if obj.sale_price:
            return str(obj.sale_price)
        return str(obj.price)
```

> 💡 **Why:** `ModelSerializer` derives fields from the model, reducing duplication. Use `source` for dotted attribute access and `SerializerMethodField` for computed values.

### ModelSerializer Configuration

❌ **Wrong:**
```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Exposes password hash, permissions, etc.
```

✅ **Correct:**
```python
from rest_framework import serializers
from django.contrib.auth import get_user_model

User = get_user_model()


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'date_joined']
        read_only_fields = ['id', 'date_joined']
        extra_kwargs = {
            'email': {'required': True},
        }

    def validate_email(self, value):
        if User.objects.filter(email=value).exclude(pk=self.instance.pk if self.instance else None).exists():
            raise serializers.ValidationError('Email already in use.')
        return value
```

> 💡 **Why:** Never use `fields = '__all__'` — it exposes internal fields. List fields explicitly. Use `extra_kwargs` for field-level overrides and `validate_<field>` for custom validation.

### Nested Serializers

❌ **Wrong:**
```python
from rest_framework import serializers
from .models import Order, OrderItem

class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = '__all__'

class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)
    # Nested write is not supported by default — POST will fail
    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total']
```

✅ **Correct:**
```python
from rest_framework import serializers
from .models import Order, OrderItem


class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ['id', 'product', 'quantity', 'unit_price']


class OrderReadSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)
    user_name = serializers.CharField(source='user.get_full_name', read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'user_name', 'items', 'total', 'status', 'created_at']


class OrderWriteSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ['items']

    def create(self, validated_data):
        items_data = validated_data.pop('items')
        order = Order.objects.create(**validated_data)
        for item_data in items_data:
            OrderItem.objects.create(order=order, **item_data)
        return order
```

> 💡 **Why:** Use separate serializers for read (nested, rich) and write (flat, writable). Override `create()`/`update()` for writable nested serializers since DRF doesn't handle them automatically.

### Views (APIView and Generics)

❌ **Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer

class ProductList(APIView):
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
    # Re-implementing what generics already provide
```

✅ **Correct:**
```python
from rest_framework import generics
from .models import Product
from .serializers import ProductSerializer


class ProductListCreateView(generics.ListCreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer


class ProductDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    def get_queryset(self):
        return super().get_queryset().filter(is_active=True)
```

> 💡 **Why:** Generic views handle serialization, pagination, status codes, and error responses. Use `APIView` only when generics don't fit your use case.

### ViewSets

❌ **Wrong:**
```python
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    # Exposes create, update, partial_update, destroy without any permission checks
```

✅ **Correct:**
```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAdminUser
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer


class ProductViewSet(viewsets.ModelViewSet):
    serializer_class = ProductSerializer

    def get_queryset(self):
        return Product.objects.select_related('category').filter(is_active=True)

    def get_permissions(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            return [IsAdminUser()]
        return [IsAuthenticated()]

    @action(detail=True, methods=['post'])
    def archive(self, request, pk=None):
        product = self.get_object()
        product.is_active = False
        product.save(update_fields=['is_active'])
        return Response({'status': 'archived'})

    @action(detail=False, methods=['get'])
    def featured(self, request):
        featured = self.get_queryset().filter(is_featured=True)[:10]
        serializer = self.get_serializer(featured, many=True)
        return Response(serializer.data)
```

> 💡 **Why:** ViewSets combine list/create/retrieve/update/destroy into one class. Use `get_permissions()` to vary permissions per action. `@action` adds custom endpoints.

### Routers

❌ **Wrong:**
```python
from django.urls import path
from .views import ProductViewSet

# Manually mapping ViewSet methods to URLs
urlpatterns = [
    path('products/', ProductViewSet.as_view({'get': 'list', 'post': 'create'})),
    path('products/<int:pk>/', ProductViewSet.as_view({'get': 'retrieve', 'put': 'update', 'delete': 'destroy'})),
]
```

✅ **Correct:**
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet, OrderViewSet

router = DefaultRouter()
router.register('products', ProductViewSet, basename='product')
router.register('orders', OrderViewSet, basename='order')

urlpatterns = [
    path('api/v1/', include(router.urls)),
]
# Generates: /api/v1/products/, /api/v1/products/{pk}/, /api/v1/products/{pk}/archive/
```

> 💡 **Why:** Routers auto-generate URL patterns for ViewSets, including custom `@action` endpoints. `DefaultRouter` adds an API root view listing all endpoints.

### Authentication

❌ **Wrong:**
```python
# Using SessionAuthentication for a mobile API
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        # Session auth requires cookies — doesn't work for mobile/SPA
    ],
}
```

✅ **Correct:**
```python
# pip install djangorestframework-simplejwt

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',  # For browsable API
    ],
}

from datetime import timedelta
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

> 💡 **Why:** JWT is stateless — works for mobile, SPAs, and microservices. Keep access tokens short-lived (30 min) and use refresh tokens for renewal. Session auth is fine for the browsable API.

### Permissions

❌ **Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response

class OrderView(APIView):
    # No permissions — any anonymous user can access
    def get(self, request):
        return Response(Order.objects.values())
```

✅ **Correct:**
```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user == request.user


# settings.py — global default
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Per-view override
from rest_framework import generics

class OrderDetailView(generics.RetrieveUpdateAPIView):
    permission_classes = [permissions.IsAuthenticated, IsOwnerOrReadOnly]
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
```

> 💡 **Why:** Set `IsAuthenticated` as the global default. Create custom permissions for object-level checks. DRF checks all permission classes — all must return True.

### Throttling

❌ **Wrong:**
```python
# No rate limiting — API vulnerable to abuse
REST_FRAMEWORK = {
    # No throttle classes configured
}
```

✅ **Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
}

# Custom throttle for sensitive endpoints
from rest_framework.throttling import UserRateThrottle

class LoginRateThrottle(UserRateThrottle):
    rate = '5/minute'

class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
```

> 💡 **Why:** Throttling prevents abuse and brute-force attacks. Set lower rates for anonymous users and sensitive endpoints (login, password reset). DRF stores throttle state in the cache.

### Pagination

❌ **Wrong:**
```python
# Returning all records at once
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()  # Returns 100,000 products in one response
```

✅ **Correct:**
```python
# settings.py — global pagination
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# Custom pagination
from rest_framework.pagination import PageNumberPagination, CursorPagination


class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100


class TimelinePagination(CursorPagination):
    page_size = 50
    ordering = '-created_at'
    # CursorPagination is most efficient for large datasets — no OFFSET


class ProductViewSet(viewsets.ModelViewSet):
    pagination_class = StandardPagination
```

> 💡 **Why:** Always paginate list endpoints. `CursorPagination` is best for large datasets (no SQL OFFSET). `PageNumberPagination` is most familiar to API consumers.

### Filtering

❌ **Wrong:**
```python
class ProductViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        qs = Product.objects.all()
        # Manual filtering — tedious and error-prone
        if self.request.query_params.get('category'):
            qs = qs.filter(category=self.request.query_params['category'])
        if self.request.query_params.get('min_price'):
            qs = qs.filter(price__gte=self.request.query_params['min_price'])
        return qs
```

✅ **Correct:**
```python
# pip install django-filter

# filters.py
import django_filters
from .models import Product


class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    category = django_filters.CharFilter(field_name='category__slug')

    class Meta:
        model = Product
        fields = ['category', 'is_active']


# views.py
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend


class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = ProductFilter
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at', 'name']
    ordering = ['-created_at']
```

> 💡 **Why:** django-filter provides declarative filtering. Combine with DRF's `SearchFilter` for full-text search and `OrderingFilter` for sortable columns.

### API Versioning

❌ **Wrong:**
```python
# No versioning — breaking changes affect all clients immediately
# Or: path('api/products/', ProductListView.as_view())
```

✅ **Correct:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/<version>/', include('apps.api.urls')),
]

# views.py
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return ProductV2Serializer
        return ProductV1Serializer
```

> 💡 **Why:** URL-based versioning (`/api/v1/`, `/api/v2/`) is the most explicit and easiest for API consumers. Switch serializers per version to evolve the API without breaking clients.

### SerializerMethodField Security

❌ **Wrong:**
```python
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    role = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'role']

    def get_role(self, obj):
        # Exposes internal role to everyone — no permission check
        return obj.role
```

✅ **Correct:**
```python
from rest_framework import serializers


class UserSerializer(serializers.ModelSerializer):
    role = serializers.SerializerMethodField()
    email = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = ['id', 'username', 'role', 'email']

    def get_role(self, obj):
        request = self.context.get('request')
        if request and (request.user == obj or request.user.is_staff):
            return obj.role
        return None

    def get_email(self, obj):
        request = self.context.get('request')
        if request and (request.user == obj or request.user.is_staff):
            return obj.email
        return None  # Hide email from other users
```

> 💡 **Why:** SerializerMethodField can access the request via `self.context['request']`. Use it to conditionally expose sensitive data based on the requesting user's identity or permissions.

### DRF Performance

❌ **Wrong:**
```python
from rest_framework import viewsets
from .models import Order
from .serializers import OrderSerializer

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    # Serializer accesses order.customer.name and order.items.all()
    # N+1 queries on every list request
```

✅ **Correct:**
```python
from rest_framework import viewsets
from .models import Order
from .serializers import OrderSerializer


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer

    def get_queryset(self):
        return (
            Order.objects
            .select_related('customer')
            .prefetch_related('items__product')
            .only('id', 'total', 'status', 'created_at',
                  'customer__id', 'customer__name')
        )
```

> 💡 **Why:** Optimize `get_queryset()` with `select_related` for FK/O2O, `prefetch_related` for M2M/reverse FK, and `only()` to limit fetched columns. Profile with Django Debug Toolbar.

## 21. Background Tasks

### Celery Setup and Configuration

❌ **Wrong:**
```python
# celery.py in the wrong location, missing autodiscover
from celery import Celery
app = Celery('myproject')
# Forgetting to configure the broker, or hardcoding credentials
app.conf.broker_url = 'redis://localhost:6379/0'
```

✅ **Correct:**
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

> 💡 **Why:** `config_from_object` with `namespace='CELERY'` reads all `CELERY_*` settings from Django settings. `autodiscover_tasks` finds `tasks.py` in each installed app automatically.

### Task Definition

❌ **Wrong:**
```python
from celery import Celery
app = Celery()

@app.task
def send_email(user_id):
    from myapp.models import User
    user = User.objects.get(pk=user_id)
    # Passing the whole user object would fail — objects aren't JSON-serializable
```

✅ **Correct:**
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

> 💡 **Why:** Use `@shared_task` to avoid importing the Celery app directly. Pass IDs, not objects — task arguments must be JSON-serializable. `bind=True` gives access to `self` for retries.

### Task Retry

❌ **Wrong:**
```python
from celery import shared_task

@shared_task
def charge_payment(order_id):
    # No retry logic — if payment gateway is temporarily down, the charge is lost
    import stripe
    order = Order.objects.get(pk=order_id)
    stripe.Charge.create(amount=int(order.total * 100), currency='usd')
```

✅ **Correct:**
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

> 💡 **Why:** `autoretry_for` retries on specific exceptions. `retry_backoff=True` uses exponential backoff. `retry_jitter=True` adds randomness to prevent thundering herd.

### Task Routing

❌ **Wrong:**
```python
# All tasks on the same queue — a slow report blocks email sending
@shared_task
def send_email(user_id): ...

@shared_task
def generate_large_report(report_id): ...  # Takes 30 minutes, blocks the queue
```

✅ **Correct:**
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

> 💡 **Why:** Route tasks to separate queues so slow tasks don't block fast ones. Run dedicated workers per queue with appropriate concurrency.

### Periodic Tasks

❌ **Wrong:**
```python
# Using a cron job that calls manage.py — outside Celery's control
# crontab: */5 * * * * cd /app && python manage.py cleanup_expired
```

✅ **Correct:**
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

> 💡 **Why:** Celery Beat manages periodic tasks within the Celery ecosystem — monitoring, retries, and logging work the same as regular tasks. Use `django-celery-beat` if you need DB-managed schedules.

### Django-Q Alternative

❌ **Wrong:**
```python
# Using threading for background tasks in Django
import threading

def my_view(request):
    t = threading.Thread(target=send_email, args=(user.id,))
    t.start()  # No retry, no monitoring, lost if server restarts
    return HttpResponse('ok')
```

✅ **Correct:**
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

> 💡 **Why:** Django-Q (django-q2 fork) is simpler than Celery — no separate broker config needed if you use Redis as Django's cache. Good for projects that don't need Celery's full feature set.

### Huey Lightweight Alternative

❌ **Wrong:**
```python
# Using Celery for a simple project with 2-3 background tasks
# Celery + Redis + Beat + Flower = complex infrastructure for a small app
```

✅ **Correct:**
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

> 💡 **Why:** Huey is a lightweight alternative to Celery — single dependency, simple config. `immediate=True` in development runs tasks synchronously. Great for small-to-medium projects.

### Django 6.0 Built-in Tasks

❌ **Wrong:**
```python
# Django 6.0+ — still using Celery just for simple fire-and-forget tasks
# when the built-in task system would suffice
```

✅ **Correct:**
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

> 💡 **Why:** Django 6.0+ includes a built-in task system that eliminates the need for Celery in simple cases. Database backend requires no extra infrastructure. Use Celery when you need advanced features like routing and priorities.

### Task Idempotency

❌ **Wrong:**
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

✅ **Correct:**
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

> 💡 **Why:** Tasks may run more than once due to retries, broker redelivery, or duplicate sends. Use idempotency keys and status checks to ensure safe re-execution.

## 22. Deployment

### Gunicorn Configuration

❌ **Wrong:**
```bash
# Running Django's development server in production
python manage.py runserver 0.0.0.0:8000
# Or Gunicorn with default settings
gunicorn config.wsgi
```

✅ **Correct:**
```python
# gunicorn.conf.py
import multiprocessing

bind = '0.0.0.0:8000'
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'gthread'  # Or 'gevent' for high-concurrency
threads = 4
timeout = 30
keepalive = 5
max_requests = 1000
max_requests_jitter = 50
accesslog = '-'
errorlog = '-'
loglevel = 'info'
```

```bash
gunicorn config.wsgi:application -c gunicorn.conf.py
```

> 💡 **Why:** `workers = 2 * CPU + 1` is the rule of thumb. `max_requests` recycles workers to prevent memory leaks. `gthread` gives threads within workers for I/O-bound apps.

### uWSGI Alternative

❌ **Wrong:**
```bash
# Running uWSGI without proper configuration
uwsgi --http :8000 --module config.wsgi
# Missing worker management, logging, and process recycling
```

✅ **Correct:**
```ini
; uwsgi.ini
[uwsgi]
module = config.wsgi:application
master = true
processes = 4
threads = 2
socket = /tmp/uwsgi.sock
chmod-socket = 660
vacuum = true
die-on-term = true
max-requests = 5000
harakiri = 30
```

```bash
uwsgi --ini uwsgi.ini
```

> 💡 **Why:** uWSGI is a mature alternative to Gunicorn. Use socket mode behind Nginx. `harakiri` kills stuck workers. `max-requests` prevents memory leaks. Gunicorn is simpler; uWSGI is more configurable.

### Daphne (ASGI)

❌ **Wrong:**
```bash
# Using Gunicorn for a Django Channels project
gunicorn config.wsgi  # WebSocket connections will fail
```

✅ **Correct:**
```bash
# For ASGI apps (Channels, async views, WebSocket)
pip install daphne

# Run with Daphne
daphne -b 0.0.0.0 -p 8000 config.asgi:application

# Or with Uvicorn (faster)
pip install uvicorn
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 4
```

> 💡 **Why:** ASGI servers (Daphne, Uvicorn) handle WebSocket and async views. Uvicorn is faster; Daphne is the official Channels server. Use ASGI only when you need async features.

### Nginx Configuration

❌ **Wrong:**
```nginx
# Proxying everything through Django, including static files
server {
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
    # Static files served by Django — slow
}
```

✅ **Correct:**
```nginx
upstream django {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    client_max_body_size 10M;

    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /app/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> 💡 **Why:** Nginx serves static/media files directly (fast, with caching headers). Django only handles dynamic requests. `X-Forwarded-Proto` lets Django know it's behind HTTPS.

### Docker

❌ **Wrong:**
```dockerfile
# Single-stage Dockerfile with development dependencies
FROM python:3.12
COPY . /app
RUN pip install -r requirements.txt  # Includes dev dependencies
CMD python manage.py runserver 0.0.0.0:8000  # Development server!
```

✅ **Correct:**
```dockerfile
# Multi-stage Dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS builder
RUN pip install --no-cache-dir pip-tools
COPY requirements/production.txt .
RUN pip install --no-cache-dir -r production.txt

FROM base AS production
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
RUN python manage.py collectstatic --noinput
EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", "-c", "gunicorn.conf.py"]
```

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [db, redis]
  db:
    image: postgres:16
    volumes: [postgres_data:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
  redis:
    image: redis:7-alpine
  celery:
    build: .
    command: celery -A config worker -l info
    env_file: .env
    depends_on: [db, redis]

volumes:
  postgres_data:
```

> 💡 **Why:** Multi-stage builds keep images small (no build tools in production). docker-compose defines the full stack. Never use `runserver` in production containers.

### Environment Variables

❌ **Wrong:**
```python
# Hardcoding configuration in settings
SECRET_KEY = 'hardcoded-secret'
DATABASE_URL = 'postgres://user:pass@localhost/db'
```

✅ **Correct:**
```python
# pip install django-environ

# settings/base.py
import environ

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)
environ.Env.read_env(BASE_DIR / '.env')

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')
DATABASES = {'default': env.db('DATABASE_URL')}
EMAIL_CONFIG = env.email('EMAIL_URL', default='consolemail://')
CACHES = {'default': env.cache('CACHE_URL', default='locmemcache://')}
```

```bash
# .env
SECRET_KEY=your-random-secret-key
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgres://user:pass@db:5432/myapp
CACHE_URL=redis://redis:6379/0
EMAIL_URL=smtp://user:pass@smtp.example.com:587
```

> 💡 **Why:** django-environ parses DATABASE_URL, CACHE_URL, etc. into Django's dict format. Keep `.env` out of git. Default values keep development simple.

### CI/CD (GitHub Actions)

❌ **Wrong:**
```yaml
# No CI — tests only run manually (or never)
```

✅ **Correct:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements/test.txt
      - run: python manage.py test --parallel
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test_db
          DJANGO_SETTINGS_MODULE: config.settings.test
          SECRET_KEY: test-secret-key
```

> 💡 **Why:** CI runs tests on every push. Service containers provide Postgres and Redis. `--parallel` speeds up the test suite. Fail the pipeline before merging broken code.

### Zero-Downtime Deployment

❌ **Wrong:**
```bash
# Stop server, deploy, start server — downtime!
systemctl stop gunicorn
git pull
pip install -r requirements.txt
python manage.py migrate
systemctl start gunicorn
```

✅ **Correct:**
```bash
#!/bin/bash
# deploy.sh — zero-downtime deployment
set -e

# 1. Pull new code
git pull origin main

# 2. Install dependencies
pip install -r requirements/production.txt

# 3. Run migrations (must be backward-compatible)
python manage.py migrate --noinput

# 4. Collect static files
python manage.py collectstatic --noinput

# 5. Graceful reload — finishes existing requests, starts new workers
kill -s HUP $(cat /tmp/gunicorn.pid)
# Or with systemd: systemctl reload gunicorn
```

> 💡 **Why:** Gunicorn's HUP signal spawns new workers with new code while old workers finish serving current requests. Migrations must be backward-compatible — new and old code run simultaneously.

### Health Check Endpoint

❌ **Wrong:**
```python
# No health check — load balancer has no way to know if the app is healthy
# Or: checking only if Django responds, not if the database is reachable
```

✅ **Correct:**
```python
# views.py
from django.db import connection
from django.http import JsonResponse


def health_check(request):
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        status = 'healthy'
        code = 200
    except Exception as e:
        status = f'unhealthy: {e}'
        code = 503

    return JsonResponse({
        'status': status,
        'database': 'ok' if code == 200 else 'error',
    }, status=code)


# urls.py — no auth required
from django.urls import path
urlpatterns = [
    path('health/', health_check, name='health-check'),
]
```

> 💡 **Why:** Load balancers need a health endpoint to route traffic. Check the database connection at minimum. Return 503 when unhealthy so the load balancer stops sending traffic.

## 23. Performance

### Django Debug Toolbar

❌ **Wrong:**
```python
# No profiling tool — guessing where performance problems are
# Or: Debug Toolbar enabled in production
INSTALLED_APPS = [
    'debug_toolbar',  # In base settings — loaded in production!
]
```

✅ **Correct:**
```python
# settings/local.py only
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
INTERNAL_IPS = ['127.0.0.1']

# urls.py
from django.conf import settings
if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

> 💡 **Why:** Debug Toolbar shows SQL queries, template rendering time, cache hits, and signals per request. Only install in development — it exposes internal data and slows requests.

### General Query Optimization Principles

❌ **Wrong:**
```python
# Fetching all fields when you only need a few
users = User.objects.all()
names = [u.name for u in users]  # Loads all columns for each user

# Using .count() just to check existence
if User.objects.filter(email=email).count() > 0:
    ...
```

✅ **Correct:**
```python
# Fetch only needed fields
names = User.objects.values_list('name', flat=True)

# Use .exists() instead of .count() > 0
if User.objects.filter(email=email).exists():
    ...

# Use .iterator() for large querysets to avoid loading all into memory
for user in User.objects.all().iterator(chunk_size=2000):
    process(user)
```

> 💡 **Why:** `values_list` avoids instantiating model objects. `exists()` stops at the first match (faster than count). `iterator()` streams results instead of loading all into memory.

### N+1 Detection and Solution

❌ **Wrong:**
```python
def get_orders(request):
    orders = Order.objects.all()
    for order in orders:
        print(order.customer.name)       # N+1
        print(order.items.count())        # N+1
        for item in order.items.all():    # N+1
            print(item.product.name)      # N*M+1
```

✅ **Correct:**
```python
from django.db.models import Count, Prefetch

def get_orders(request):
    orders = (
        Order.objects
        .select_related('customer')           # JOIN for FK
        .prefetch_related(
            Prefetch('items', queryset=OrderItem.objects.select_related('product'))
        )                                      # 2 queries for M2M + FK
        .annotate(item_count=Count('items'))   # Aggregate in SQL
    )
    for order in orders:
        print(order.customer.name)       # No extra query
        print(order.item_count)          # No extra query
        for item in order.items.all():   # No extra query
            print(item.product.name)     # No extra query
```

> 💡 **Why:** `select_related` = SQL JOIN (FK/O2O). `prefetch_related` = separate query + Python join (M2M/reverse FK). `Prefetch` object lets you chain `select_related` on the prefetched queryset.

### Database Indexing

❌ **Wrong:**
```python
class Order(models.Model):
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)
    customer = models.ForeignKey('Customer', on_delete=models.CASCADE)
    # No indexes — queries on status and created_at do full table scans
```

✅ **Correct:**
```python
from django.db import models


class Order(models.Model):
    status = models.CharField(max_length=20, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    customer = models.ForeignKey('Customer', on_delete=models.CASCADE)

    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),       # Composite index
            models.Index(fields=['-created_at']),                 # Descending index
            models.Index(
                fields=['status'],
                condition=models.Q(status='pending'),
                name='pending_orders_idx',                        # Partial index
            ),
        ]
```

> 💡 **Why:** Index columns used in `WHERE`, `ORDER BY`, and `JOIN`. Composite indexes match multi-column queries. Partial indexes are smaller and faster for common filtered queries.

### Caching Strategies

❌ **Wrong:**
```python
# Caching at the wrong level — too broad or too narrow
@cache_page(3600)
def dashboard(request):
    # Entire page cached for 1 hour — user sees stale data
    ...
```

✅ **Correct:**
```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page


# Level 1: Query-level cache for expensive computations
def get_dashboard_stats():
    stats = cache.get('dashboard_stats')
    if stats is None:
        stats = Order.objects.aggregate(
            total=Sum('total'),
            count=Count('id'),
        )
        cache.set('dashboard_stats', stats, timeout=300)
    return stats


# Level 2: Fragment cache in templates for expensive partials
# {% cache 300 sidebar_widgets user.pk %}...{% endcache %}

# Level 3: Full page cache for public, static pages
@cache_page(60 * 60)
def about_page(request):
    return render(request, 'about.html')
```

> 💡 **Why:** Cache at the most granular level that makes sense. Query caching for expensive DB operations. Fragment caching for expensive template partials. Page caching for fully static pages.

### Deferred Fields

❌ **Wrong:**
```python
# Loading a TextField with 100KB of content when you only need the title
articles = Article.objects.all()
for article in articles:
    print(article.title)  # Also loaded: body (100KB), metadata, etc.
```

✅ **Correct:**
```python
# only() — fetch only specified fields
articles = Article.objects.only('id', 'title', 'created_at')
for article in articles:
    print(article.title)  # Only these fields are loaded
    # article.body  # Would trigger an additional query (lazy load)

# defer() — fetch everything except specified fields
articles = Article.objects.defer('body', 'metadata')

# values() — returns dicts, not model instances (fastest)
articles = Article.objects.values('id', 'title')
```

> 💡 **Why:** `only()` and `defer()` reduce memory usage for large text/binary fields. `values()` skips model instantiation entirely. Use when you don't need the full model.

### Connection Pooling

❌ **Wrong:**
```python
# Django opens and closes a DB connection per request by default
# Under heavy load, this creates connection overhead
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        # No connection pooling
    }
}
```

✅ **Correct:**
```python
# Option 1: Django 5.1+ built-in connection pooling
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'OPTIONS': {
            'pool': True,  # Django 5.1+
        },
    }
}

# Option 2: PgBouncer (external connection pooler)
# pgbouncer.ini:
# [databases]
# mydb = host=127.0.0.1 port=5432 dbname=mydb
# [pgbouncer]
# pool_mode = transaction
# max_client_conn = 1000
# default_pool_size = 20

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': '127.0.0.1',
        'PORT': '6432',  # PgBouncer port
    }
}
```

> 💡 **Why:** Connection pooling reuses database connections instead of opening/closing per request. Django 5.1+ has built-in pooling. PgBouncer is the standard external pooler for PostgreSQL.

### Async Views (Django 4.1+)

❌ **Wrong:**
```python
# Using sync views for I/O-bound operations
import requests

def external_api_view(request):
    # Blocks the worker thread while waiting for external API
    response1 = requests.get('https://api1.example.com/data')
    response2 = requests.get('https://api2.example.com/data')
    # Sequential — takes sum of both response times
    return JsonResponse({...})
```

✅ **Correct:**
```python
# Django 4.1+ async views
import httpx
from django.http import JsonResponse


async def external_api_view(request):
    async with httpx.AsyncClient() as client:
        # Concurrent — takes max of both response times
        response1, response2 = await asyncio.gather(
            client.get('https://api1.example.com/data'),
            client.get('https://api2.example.com/data'),
        )
    return JsonResponse({
        'api1': response1.json(),
        'api2': response2.json(),
    })
```

```python
import asyncio
```

> 💡 **Why:** Async views don't block the worker thread during I/O waits. Use `asyncio.gather()` for concurrent external API calls. Requires ASGI server (Uvicorn/Daphne).

## 24. Django Channels

### ASGI Setup

❌ **Wrong:**
```python
# asgi.py — using get_asgi_application without routing
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
application = get_asgi_application()
# WebSocket connections will get 404
```

✅ **Correct:**
```python
# pip install channels channels-redis

# config/asgi.py
import os
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
django_asgi_app = get_asgi_application()

from apps.chat import routing  # Import after django setup

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(routing.websocket_urlpatterns)
        )
    ),
})

# settings.py
INSTALLED_APPS = [..., 'channels']
ASGI_APPLICATION = 'config.asgi.application'
```

> 💡 **Why:** `ProtocolTypeRouter` splits HTTP and WebSocket traffic. `AuthMiddlewareStack` provides `scope['user']` for WebSocket connections. `AllowedHostsOriginValidator` prevents cross-site WebSocket hijacking.

### WebSocket Consumer

❌ **Wrong:**
```python
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def receive(self, text_data):
        # Processing in the consumer directly — blocks the event loop
        import time
        time.sleep(5)  # Blocks all other connections!
        self.send(text_data=text_data)
```

✅ **Correct:**
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f'chat_{self.room_name}'

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        message = data['message']
        user = self.scope['user']

        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat.message',
                'message': message,
                'username': user.username,
            }
        )

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            'message': event['message'],
            'username': event['username'],
        }))
```

> 💡 **Why:** Use `AsyncWebsocketConsumer` to avoid blocking the event loop. Group messaging broadcasts to all room members. The `type` field maps to handler methods (dots become underscores).

### Channel Layers

❌ **Wrong:**
```python
# Using InMemoryChannelLayer in production
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
        # Only works within a single process — no cross-worker communication
    }
}
```

✅ **Correct:**
```python
# pip install channels-redis

# settings.py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/2')],
            'capacity': 1500,
            'expiry': 10,
        },
    },
}

# Test with InMemoryChannelLayer in tests
# settings/test.py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
    }
}
```

> 💡 **Why:** Redis channel layer enables cross-process communication — essential when running multiple ASGI workers. InMemoryChannelLayer is fine for development and testing only.

### Group Messaging

❌ **Wrong:**
```python
# Sending to individual connections manually
class NotificationConsumer(AsyncWebsocketConsumer):
    connected_users = {}  # Global state — breaks with multiple workers

    async def connect(self):
        self.connected_users[self.scope['user'].pk] = self.channel_name
        await self.accept()

    async def send_to_user(self, user_id, message):
        channel = self.connected_users.get(user_id)
        if channel:
            await self.channel_layer.send(channel, {'type': 'notify', 'message': message})
```

✅ **Correct:**
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class NotificationConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']
        if self.user.is_anonymous:
            await self.close()
            return

        # Add user to their personal notification group
        self.group_name = f'notifications_{self.user.pk}'
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        if hasattr(self, 'group_name'):
            await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def notify(self, event):
        await self.send(text_data=json.dumps(event['data']))


# Send notification from anywhere (views, tasks, signals)
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def send_notification(user_id, data):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'notifications_{user_id}',
        {'type': 'notify', 'data': data},
    )
```

> 💡 **Why:** Groups work across all workers via Redis. Use per-user groups for targeted notifications. `async_to_sync` lets you send from sync code (views, Celery tasks).

### WebSocket Authentication

❌ **Wrong:**
```python
# No authentication — any anonymous user can connect
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()  # Anyone can connect
```

✅ **Correct:**
```python
# routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path('ws/chat/<str:room_name>/', consumers.ChatConsumer.as_asgi()),
]

# consumers.py
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']

        # Reject anonymous users
        if self.user.is_anonymous:
            await self.close()
            return

        self.room_name = self.scope['url_route']['kwargs']['room_name']
        # Check permission
        if not await self.has_room_access():
            await self.close()
            return

        self.room_group = f'chat_{self.room_name}'
        await self.channel_layer.group_add(self.room_group, self.channel_name)
        await self.accept()

    async def has_room_access(self):
        from channels.db import database_sync_to_async
        @database_sync_to_async
        def check():
            return self.user.chat_rooms.filter(name=self.room_name).exists()
        return await check()
```

> 💡 **Why:** `AuthMiddlewareStack` in ASGI config provides `scope['user']` from the session cookie. Always check authentication and authorization in `connect()`. Reject unauthorized connections early.

### HTTP Long Polling Alternative

❌ **Wrong:**
```python
# Polling with short interval — wastes bandwidth
# JavaScript: setInterval(() => fetch('/api/notifications/'), 1000)
```

✅ **Correct:**
```python
# For simple real-time without Channels — use Server-Sent Events (SSE)
import asyncio
from django.http import StreamingHttpResponse


async def sse_notifications(request):
    async def event_stream():
        while True:
            # Check for new notifications
            from channels.db import database_sync_to_async

            @database_sync_to_async
            def get_notifications():
                return list(
                    request.user.notifications
                    .filter(is_read=False)
                    .values('id', 'message')[:10]
                )

            notifications = await get_notifications()
            if notifications:
                import json
                data = json.dumps(notifications)
                yield f'data: {data}\n\n'
            await asyncio.sleep(5)

    response = StreamingHttpResponse(event_stream(), content_type='text/event-stream')
    response['Cache-Control'] = 'no-cache'
    return response
```

> 💡 **Why:** SSE is simpler than WebSockets for server-to-client streaming. No extra infrastructure needed. Use WebSockets only when you need bidirectional communication.

## 25. Django Ecosystem

### Django REST Framework

❌ **Wrong:**
```python
# Building a REST API from scratch with JsonResponse
from django.http import JsonResponse

def api_products(request):
    products = list(Product.objects.values())
    return JsonResponse(products, safe=False)
    # No serialization, no validation, no pagination, no auth
```

✅ **Correct:**
```python
# pip install djangorestframework

# settings.py
INSTALLED_APPS = [..., 'rest_framework']
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': ['rest_framework.permissions.IsAuthenticated'],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

> 💡 **Why:** DRF provides serialization, validation, authentication, permissions, pagination, throttling, and a browsable API. Don't rebuild these from scratch.

### Django Filter

❌ **Wrong:**
```python
# Manual query parameter parsing for filtering
def product_list(request):
    qs = Product.objects.all()
    if request.GET.get('min_price'):
        qs = qs.filter(price__gte=request.GET['min_price'])
    # Repeated for every filter...
```

✅ **Correct:**
```python
# pip install django-filter

# settings.py
INSTALLED_APPS = [..., 'django_filters']

# filters.py
import django_filters
from .models import Product

class ProductFilter(django_filters.FilterSet):
    class Meta:
        model = Product
        fields = {
            'price': ['gte', 'lte'],
            'category': ['exact'],
            'name': ['icontains'],
        }
```

> 💡 **Why:** django-filter generates filter forms and querysets declaratively. Integrates with both DRF and standard Django views.

### Django Allauth

❌ **Wrong:**
```python
# Implementing OAuth from scratch with requests-oauthlib
# Handling token exchange, user creation, email verification manually
```

✅ **Correct:**
```python
# pip install django-allauth

# settings.py
INSTALLED_APPS = [
    ...,
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.github',
]

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'
ACCOUNT_AUTHENTICATION_METHOD = 'email'

# urls.py
urlpatterns = [
    path('accounts/', include('allauth.urls')),
]
```

> 💡 **Why:** Allauth handles email/password registration, email verification, password reset, and 50+ OAuth providers. Battle-tested security you shouldn't rewrite.

### Django Debug Toolbar

❌ **Wrong:**
```python
# Printing queries manually
from django.db import connection
print(len(connection.queries))
```

✅ **Correct:**
```python
# pip install django-debug-toolbar

# settings/local.py
INSTALLED_APPS += ['debug_toolbar']
MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
INTERNAL_IPS = ['127.0.0.1']

# urls.py (development only)
if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [path('__debug__/', include(debug_toolbar.urls))] + urlpatterns
```

> 💡 **Why:** Shows SQL queries, duplicates, time per query, template rendering, cache operations, and signals — all in a sidebar panel. Essential for finding N+1 queries.

### Django Storages

❌ **Wrong:**
```python
# Writing custom S3 integration per project
import boto3
# ... 50 lines of upload/download/delete code
```

✅ **Correct:**
```python
# pip install django-storages boto3

# settings.py — S3
STORAGES = {
    'default': {'BACKEND': 'storages.backends.s3boto3.S3Boto3Storage'},
}
AWS_STORAGE_BUCKET_NAME = 'my-bucket'

# For Google Cloud Storage:
# pip install django-storages google-cloud-storage
# STORAGES = {'default': {'BACKEND': 'storages.backends.gcloud.GoogleCloudStorage'}}
# GS_BUCKET_NAME = 'my-bucket'

# For Azure:
# pip install django-storages azure-storage-blob
# STORAGES = {'default': {'BACKEND': 'storages.backends.azure_storage.AzureStorage'}}
# AZURE_CONTAINER = 'my-container'
```

> 💡 **Why:** django-storages provides a unified API for S3, GCS, Azure, and more. Your models and forms work unchanged regardless of storage backend.

### Django Channels

❌ **Wrong:**
```python
# Using polling for real-time features
# setInterval(fetch('/api/messages/'), 2000)
```

✅ **Correct:**
```python
# pip install channels channels-redis

# settings.py
INSTALLED_APPS = [..., 'channels']
ASGI_APPLICATION = 'config.asgi.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {'hosts': [('127.0.0.1', 6379)]},
    },
}
```

> 💡 **Why:** Channels adds WebSocket, HTTP long-polling, and background task support to Django. Use it for chat, notifications, live updates, and any real-time feature.

### Django Celery Results

❌ **Wrong:**
```python
# No way to check if a Celery task completed or what it returned
result = my_task.delay(data)
# result.get() blocks — defeats the purpose of async tasks
```

✅ **Correct:**
```python
# pip install django-celery-results

# settings.py
INSTALLED_APPS = [..., 'django_celery_results']
CELERY_RESULT_BACKEND = 'django-db'  # Store results in Django's database

# Usage
result = my_task.delay(data)
# Later — check status without blocking
from django_celery_results.models import TaskResult
task = TaskResult.objects.get(task_id=result.id)
print(task.status, task.result)
```

> 💡 **Why:** django-celery-results stores task results in the database, making them queryable via Django ORM. Useful for tracking task status in admin or dashboards.

### Django Guardian

❌ **Wrong:**
```python
# Implementing object-level permissions from scratch
class Article(models.Model):
    owner = models.ForeignKey(User, on_delete=models.CASCADE)

def can_edit(user, article):
    return user == article.owner  # Doesn't scale for teams, shared access
```

✅ **Correct:**
```python
# pip install django-guardian

# settings.py
INSTALLED_APPS = [..., 'guardian']
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'guardian.backends.ObjectPermissionBackend',
]

# Usage
from guardian.shortcuts import assign_perm, get_objects_for_user

assign_perm('change_article', user, article)
assign_perm('view_article', team_group, article)

# Check: user.has_perm('change_article', article)
# Query: get_objects_for_user(user, 'articles.view_article')
```

> 💡 **Why:** Django Guardian provides per-object permissions (user X can edit article Y). Works with Django's built-in permission system and DRF.

### Django Import Export

❌ **Wrong:**
```python
# Writing custom CSV import/export views for admin
import csv
from django.http import HttpResponse

def export_products(request):
    response = HttpResponse(content_type='text/csv')
    writer = csv.writer(response)
    for p in Product.objects.all():
        writer.writerow([p.name, p.price])
    return response
```

✅ **Correct:**
```python
# pip install django-import-export

# admin.py
from import_export import resources
from import_export.admin import ImportExportModelAdmin
from .models import Product


class ProductResource(resources.ModelResource):
    class Meta:
        model = Product
        fields = ('id', 'name', 'price', 'category__name', 'is_active')


@admin.register(Product)
class ProductAdmin(ImportExportModelAdmin):
    resource_class = ProductResource
    list_display = ('name', 'price', 'is_active')
```

> 💡 **Why:** django-import-export adds CSV/Excel/JSON import and export buttons to the admin. Handles validation, preview, and rollback on import errors.

### Django Unfold

❌ **Wrong:**
```python
# Using the default Django admin UI for a client-facing admin panel
# The default admin looks dated and lacks modern UX features
```

✅ **Correct:**
```python
# pip install django-unfold

# settings.py
INSTALLED_APPS = [
    'unfold',  # Must be before django.contrib.admin
    'django.contrib.admin',
    ...
]

# admin.py
from unfold.admin import ModelAdmin

@admin.register(Product)
class ProductAdmin(ModelAdmin):
    list_display = ('name', 'price', 'is_active')
```

> 💡 **Why:** Django Unfold provides a modern, responsive admin UI with dark mode, improved navigation, and better form layouts — with minimal code changes.

### django-environ / python-decouple

❌ **Wrong:**
```python
# Hardcoded settings, no environment variable support
SECRET_KEY = 'hardcoded'
DEBUG = True
```

✅ **Correct:**
```python
# django-environ (more Django-specific)
# pip install django-environ
import environ
env = environ.Env()
environ.Env.read_env()
DATABASES = {'default': env.db()}

# python-decouple (simpler, framework-agnostic)
# pip install python-decouple
from decouple import config, Csv
SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())
```

> 💡 **Why:** Both read from `.env` files and environment variables. django-environ has Django-specific helpers (`env.db()`, `env.cache()`). python-decouple is simpler and framework-agnostic.

### django-redis

❌ **Wrong:**
```python
# Using Django's built-in Redis backend without connection pooling
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://localhost:6379',
    }
}
```

✅ **Correct:**
```python
# pip install django-redis

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {'max_connections': 50},
        },
    }
}

# Can also be used as session backend
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

> 💡 **Why:** django-redis provides connection pooling, key prefixing, compression, and can serve as both cache and session backend. More production-ready than Django's built-in Redis backend.

### Pillow

❌ **Wrong:**
```python
# Using ImageField without Pillow installed
class Photo(models.Model):
    image = models.ImageField(upload_to='photos/')
    # ImportError: No module named 'PIL'
```

✅ **Correct:**
```python
# pip install Pillow

from django.db import models

class Photo(models.Model):
    image = models.ImageField(upload_to='photos/%Y/%m/')

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Resize on upload
        from PIL import Image
        img = Image.open(self.image.path)
        if img.width > 1920 or img.height > 1080:
            img.thumbnail((1920, 1080))
            img.save(self.image.path)
```

> 💡 **Why:** Pillow is required for `ImageField`. It validates that uploaded files are actual images and provides image processing (resize, crop, format conversion).

## 26. Advanced Django

### Custom Model Fields

❌ **Wrong:**
```python
from django.db import models

class Product(models.Model):
    # Storing JSON as a TextField and parsing manually
    metadata = models.TextField(default='{}')

    def get_metadata(self):
        import json
        return json.loads(self.metadata)

    def set_metadata(self, data):
        import json
        self.metadata = json.dumps(data)
```

✅ **Correct:**
```python
from django.db import models


class CompressedTextField(models.TextField):
    """Stores text compressed with zlib in the database."""

    def from_db_value(self, value, expression, connection):
        if value is None:
            return value
        import zlib
        import base64
        return zlib.decompress(base64.b64decode(value)).decode('utf-8')

    def to_python(self, value):
        if isinstance(value, str):
            return value
        if value is None:
            return value
        import zlib
        import base64
        return zlib.decompress(base64.b64decode(value)).decode('utf-8')

    def get_prep_value(self, value):
        if value is None:
            return value
        import zlib
        import base64
        return base64.b64encode(zlib.compress(value.encode('utf-8'))).decode('ascii')


# For JSON data, just use Django's built-in JSONField
class Product(models.Model):
    metadata = models.JSONField(default=dict, blank=True)
    # Supports lookups: Product.objects.filter(metadata__color='red')
```

> 💡 **Why:** `JSONField` is built-in since Django 3.1 — don't store JSON as text. For truly custom fields, implement `from_db_value`, `to_python`, and `get_prep_value` for the DB-to-Python-to-DB cycle.

### Multi-Database Support

❌ **Wrong:**
```python
# Hardcoding database selection in views
from django.db import connections

def analytics_view(request):
    cursor = connections['analytics'].cursor()
    cursor.execute('SELECT ...')
    # Manual connection management — error-prone
```

✅ **Correct:**
```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'primary_db',
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'analytics_db',
    },
}

# Router
class AnalyticsRouter:
    analytics_models = {'AnalyticsEvent', 'PageView'}

    def db_for_read(self, model, **hints):
        if model.__name__ in self.analytics_models:
            return 'analytics'
        return None

    def db_for_write(self, model, **hints):
        if model.__name__ in self.analytics_models:
            return 'analytics'
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if model_name and model_name.capitalize() in self.analytics_models:
            return db == 'analytics'
        return db == 'default'


# settings.py
DATABASE_ROUTERS = ['config.routers.AnalyticsRouter']

# Usage — transparent
events = AnalyticsEvent.objects.all()  # Reads from analytics DB
# Or explicit:
User.objects.using('analytics').all()
```

> 💡 **Why:** Database routers automate read/write routing based on models. Queries work normally — the router decides which database to use. Use `using()` for explicit override.

### Database Router

❌ **Wrong:**
```python
# A router that doesn't handle all methods — unexpected behavior
class MyRouter:
    def db_for_read(self, model, **hints):
        return 'replica'
    # Missing db_for_write, allow_relation, allow_migrate
```

✅ **Correct:**
```python
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        # Allow relations between objects in the same database group
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        # Only run migrations on the primary
        return db == 'default'
```

```python
# settings.py
DATABASES = {
    'default': {'ENGINE': 'django.db.backends.postgresql', 'NAME': 'primary'},
    'replica': {'ENGINE': 'django.db.backends.postgresql', 'NAME': 'replica', 'TEST': {'MIRROR': 'default'}},
}
DATABASE_ROUTERS = ['config.routers.PrimaryReplicaRouter']
```

> 💡 **Why:** A complete router implements all four methods. `allow_relation` prevents cross-database foreign keys. `allow_migrate` ensures migrations only run on the primary. `TEST.MIRROR` handles the replica in tests.

### Custom Template Engine

❌ **Wrong:**
```python
# Overriding Django's template engine for minor customizations
TEMPLATES = [{
    'BACKEND': 'django.template.backends.django.DjangoTemplates',
    # Replacing the whole engine when a custom tag would suffice
}]
```

✅ **Correct:**
```python
# Use Jinja2 alongside Django templates when you need its features
# pip install Jinja2

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'jinja2_templates'],
        'APP_DIRS': False,
        'OPTIONS': {
            'environment': 'config.jinja2.environment',
        },
    },
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [...],
        },
    },
]

# config/jinja2.py
from django.urls import reverse
from django.templatetags.static import static
from jinja2 import Environment

def environment(**options):
    env = Environment(**options)
    env.globals.update({
        'static': static,
        'url': reverse,
    })
    return env
```

> 💡 **Why:** You can use Jinja2 and Django templates side-by-side. Jinja2 is faster and has more powerful expressions. Use Django templates for admin and contrib apps, Jinja2 for your custom templates.

### ORM Internals

❌ **Wrong:**
```python
# Overriding queryset behavior without understanding the SQL compiler
class MyQuerySet(models.QuerySet):
    def custom_filter(self):
        # Building raw SQL strings instead of using the query API
        return self.raw('SELECT * FROM ...')
```

✅ **Correct:**
```python
from django.db import models
from django.db.models.sql import Query


# Understanding how querysets build SQL
qs = Product.objects.filter(price__gt=10).order_by('name')
print(qs.query)  # Shows the SQL Django will execute

# Custom lookups
from django.db.models import Lookup

class NotEqual(Lookup):
    lookup_name = 'ne'

    def as_sql(self, compiler, connection):
        lhs, lhs_params = self.process_lhs(compiler, connection)
        rhs, rhs_params = self.process_rhs(compiler, connection)
        return f'{lhs} != {rhs}', lhs_params + rhs_params

models.Field.register_lookup(NotEqual)

# Usage: Product.objects.filter(status__ne='archived')
```

> 💡 **Why:** Understanding `QuerySet.query` helps debug slow queries. Custom lookups extend the ORM's filter syntax. Print `.query` to see the generated SQL before optimizing.

### Signal Architecture

❌ **Wrong:**
```python
# Signals that are impossible to test in isolation
@receiver(post_save, sender=Order)
def complex_side_effect(sender, instance, **kwargs):
    send_email(instance)
    update_inventory(instance)
    notify_warehouse(instance)
    # Can't test the view without all side effects firing
```

✅ **Correct:**
```python
# Decouple signal handlers so they can be tested independently
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Order


@receiver(post_save, sender=Order)
def on_order_created(sender, instance, created, **kwargs):
    if created:
        from .tasks import process_new_order
        process_new_order.delay(instance.pk)


# In tests — disconnect signals when testing views in isolation
from django.test import TestCase
from unittest.mock import patch


class OrderViewTest(TestCase):
    @patch('apps.orders.signals.on_order_created')
    def test_create_order(self, mock_signal):
        response = self.client.post('/orders/create/', data={...})
        self.assertEqual(response.status_code, 302)
        # Signal handler was mocked — no side effects
```

> 💡 **Why:** Keep signal handlers thin — delegate to tasks or services. Mock signals in tests that don't need them. This keeps tests fast and focused.

### Custom Management Commands

❌ **Wrong:**
```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    def handle(self, *args, **options):
        # No arguments, no error handling, no output styling
        from myapp.models import Order
        Order.objects.filter(status='expired').delete()
        print('done')  # Not using self.stdout
```

✅ **Correct:**
```python
from django.core.management.base import BaseCommand, CommandError
from django.db import transaction


class Command(BaseCommand):
    help = 'Archive orders older than N days'

    def add_arguments(self, parser):
        parser.add_argument('days', type=int, help='Archive orders older than this many days')
        parser.add_argument('--dry-run', action='store_true', help='Preview without making changes')
        parser.add_argument('--batch-size', type=int, default=1000)

    @transaction.atomic
    def handle(self, *args, **options):
        from django.utils import timezone
        from apps.orders.models import Order

        cutoff = timezone.now() - timezone.timedelta(days=options['days'])
        orders = Order.objects.filter(created_at__lt=cutoff, status='completed')
        count = orders.count()

        if count == 0:
            self.stdout.write(self.style.WARNING('No orders to archive.'))
            return

        if options['dry_run']:
            self.stdout.write(f'Would archive {count} orders.')
            return

        updated = orders.update(status='archived')
        self.stdout.write(self.style.SUCCESS(f'Archived {updated} orders.'))
```

> 💡 **Why:** Management commands should accept arguments, support `--dry-run`, use `self.stdout` with styling, and handle edge cases. They're often run in production — make them safe.

## 27. Django Architecture

### MTV Pattern

❌ **Wrong:**
```python
# Confusing MVC with MTV — putting controller logic in templates
# template.html
{% if user.orders.count > 10 and user.is_premium %}
  {% for order in user.orders.all %}
    {% if order.total > 100 %}
      <!-- Complex business logic in templates -->
    {% endif %}
  {% endfor %}
{% endif %}
```

✅ **Correct:**
```python
# Model — data + business logic
class User(models.Model):
    @property
    def is_vip(self):
        return self.orders.count() > 10 and self.is_premium

# View — request handling + coordination
def dashboard(request):
    user = request.user
    context = {
        'is_vip': user.is_vip,
        'high_value_orders': user.orders.filter(total__gt=100),
    }
    return render(request, 'dashboard.html', context)

# Template — presentation only
# {% if is_vip %}...{% endif %}
# {% for order in high_value_orders %}...{% endfor %}
```

> 💡 **Why:** Model = data + business rules. Template = presentation. View = HTTP handling + glue. Keep templates dumb — they should only display data, not compute it.

### App Modularization

❌ **Wrong:**
```python
# One monolithic app doing everything
myproject/
    shop/
        models.py    # User, Product, Order, Payment, Review, Notification, BlogPost
        views.py     # 100+ views
        admin.py     # 30+ admin classes
```

✅ **Correct:**
```python
# Bounded contexts as separate apps
myproject/
    apps/
        accounts/      # User registration, profile, authentication
        products/      # Product catalog, categories, search
        orders/        # Order lifecycle, line items
        payments/      # Payment processing, refunds
        reviews/       # Product reviews, ratings
        notifications/ # Email, push, in-app notifications
        blog/          # Blog posts, comments

# Each app has a clear boundary and minimal dependencies
# orders/ depends on accounts/ and products/ but not on blog/
```

> 💡 **Why:** Split apps along domain boundaries (bounded contexts). An app should have 3-8 models. If it has 15+ models or its `models.py` exceeds 500 lines, it's time to split.

### Service Layer

❌ **Wrong:**
```python
# Business logic scattered in views
def checkout_view(request):
    order = Order.objects.create(user=request.user)
    for item in request.user.cart.items.all():
        OrderItem.objects.create(order=order, product=item.product, quantity=item.quantity)
    order.total = sum(i.product.price * i.quantity for i in order.items.all())
    order.save()
    charge = stripe.Charge.create(amount=int(order.total * 100))
    order.payment_id = charge.id
    order.status = 'paid'
    order.save()
    send_mail(...)
    request.user.cart.items.all().delete()
    return redirect('orders:detail', pk=order.pk)
    # 15 lines of business logic — untestable without HTTP request
```

✅ **Correct:**
```python
# services.py — business logic lives here
from django.db import transaction
from .models import Order, OrderItem


class OrderService:
    @staticmethod
    @transaction.atomic
    def create_from_cart(user):
        cart = user.cart
        order = Order.objects.create(user=user)

        items = []
        for cart_item in cart.items.select_related('product'):
            items.append(OrderItem(
                order=order,
                product=cart_item.product,
                quantity=cart_item.quantity,
                unit_price=cart_item.product.price,
            ))
        OrderItem.objects.bulk_create(items)

        order.recalculate_total()
        return order

    @staticmethod
    def charge(order):
        from apps.payments.gateway import charge_order
        charge = charge_order(order)
        order.payment_id = charge.id
        order.status = 'paid'
        order.save(update_fields=['payment_id', 'status'])
        return order


# views.py — thin, just HTTP handling
def checkout_view(request):
    order = OrderService.create_from_cart(request.user)
    OrderService.charge(order)
    request.user.cart.items.all().delete()
    return redirect('orders:detail', pk=order.pk)
```

> 💡 **Why:** Services encapsulate business logic in testable, reusable functions. Views handle HTTP; services handle business rules. Test services without HTTP overhead.

### Repository Pattern

❌ **Wrong:**
```python
# Using the repository pattern when Django's ORM already provides it
class ProductRepository:
    def get_all(self):
        return Product.objects.all()

    def get_by_id(self, pk):
        return Product.objects.get(pk=pk)

    def filter_by_category(self, category):
        return Product.objects.filter(category=category)
    # This is just wrapping the ORM with no added value
```

✅ **Correct:**
```python
# The repository pattern is rarely needed in Django — the ORM IS your repository
# Use it only when you need to abstract the data source

# When repository pattern DOES make sense:
class ExternalProductRepository:
    """Abstracts data that comes from an external API or multiple sources."""

    def __init__(self, api_client):
        self.api_client = api_client

    def get_products(self, category=None):
        # Combines local DB with external API
        local = Product.objects.filter(source='local')
        if category:
            local = local.filter(category=category)

        external = self.api_client.get_products(category=category)
        return list(local) + external

# For most Django apps, use custom QuerySet methods instead:
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_active=True)

    def in_category(self, category):
        return self.filter(category=category)

    def affordable(self, max_price):
        return self.filter(price__lte=max_price)
```

> 💡 **Why:** Django's ORM is already a repository + query builder. Adding another layer just adds indirection. Use custom QuerySet methods for reusable filters. Reserve the pattern for multi-source data.

### Domain Driven Design (DDD)

❌ **Wrong:**
```python
# Forcing full DDD patterns on a simple Django project
# Separate value objects, aggregates, repositories, domain events...
# for a blog with 5 models — massive over-engineering
```

✅ **Correct:**
```python
# Apply DDD concepts pragmatically in Django

# Aggregate root — Order controls its items
class Order(models.Model):
    status = models.CharField(max_length=20)
    total = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    def add_item(self, product, quantity):
        """Only modify items through the Order aggregate."""
        item = self.items.create(product=product, quantity=quantity, unit_price=product.price)
        self.recalculate_total()
        return item

    def remove_item(self, item_id):
        self.items.filter(pk=item_id).delete()
        self.recalculate_total()

    def recalculate_total(self):
        from django.db.models import F, Sum
        self.total = self.items.aggregate(
            total=Sum(F('unit_price') * F('quantity'))
        )['total'] or 0
        self.save(update_fields=['total'])

    def submit(self):
        if self.status != 'draft':
            raise ValueError('Only draft orders can be submitted')
        if not self.items.exists():
            raise ValueError('Cannot submit an empty order')
        self.status = 'submitted'
        self.save(update_fields=['status'])


# Value object as a dataclass (not a model)
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str = 'USD'

    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError('Cannot add different currencies')
        return Money(self.amount + other.amount, self.currency)
```

> 💡 **Why:** Apply DDD concepts where they add clarity — aggregate roots for consistency boundaries, value objects for immutable domain concepts. Don't force full DDD on simple CRUD apps.

### SOLID Principles in Django

❌ **Wrong:**
```python
# Single Responsibility Violation — one view does everything
def product_view(request, pk):
    product = Product.objects.get(pk=pk)
    reviews = Review.objects.filter(product=product)
    recommendations = get_recommendations(product)
    analytics.track('product_view', product.pk)
    if request.method == 'POST':
        Review.objects.create(product=product, user=request.user, ...)
        send_review_notification(product.owner)
        recalculate_rating(product)
    return render(request, 'product.html', {...})
```

✅ **Correct:**
```python
# Single Responsibility — each class/function has one job
class ProductDetailView(DetailView):
    model = Product
    template_name = 'products/detail.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['reviews'] = self.object.reviews.select_related('user')[:20]
        return context


# Open/Closed — extend via new classes, not modifying existing
class NotificationService:
    def __init__(self, backends=None):
        self.backends = backends or [EmailBackend(), PushBackend()]

    def notify(self, user, message):
        for backend in self.backends:
            backend.send(user, message)


# Dependency Inversion — depend on abstractions
class PaymentProcessor:
    def __init__(self, gateway):  # Accept any gateway that implements charge()
        self.gateway = gateway

    def process(self, order):
        return self.gateway.charge(order.total)

# Usage:
# PaymentProcessor(StripeGateway()).process(order)
# PaymentProcessor(MockGateway()).process(order)  # In tests
```

> 💡 **Why:** SOLID prevents god classes and tangled dependencies. In Django: thin views (SRP), service classes with injectable dependencies (DIP), and new behavior via new classes rather than modifying existing ones (OCP).

### Anti-Patterns

❌ **Wrong:**
```python
# Fat View — hundreds of lines of business logic in a view
def checkout(request):
    # 200 lines of validation, payment, inventory, email, logging...
    pass

# Business Logic in Templates
# {% if user.orders.count > 10 and user.profile.tier == 'gold' and product.stock > 0 %}

# God Model — one model with 30+ fields and 20+ methods
class User(models.Model):
    # user fields, profile fields, settings, preferences, billing,
    # notification preferences, analytics data...
    # 50 fields, 30 methods
```

✅ **Correct:**
```python
# Thin View — delegates to services
def checkout(request):
    form = CheckoutForm(request.POST)
    if form.is_valid():
        order = CheckoutService.process(request.user, form.cleaned_data)
        return redirect('orders:confirmation', pk=order.pk)
    return render(request, 'checkout.html', {'form': form})


# Logic in models/services, not templates
class User(models.Model):
    @property
    def can_purchase(self):
        return self.is_active and self.profile.tier in ('gold', 'platinum')

# Template: {% if user.can_purchase %}


# Decomposed models — split god models into focused models
class User(models.Model):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=255)

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    avatar = models.ImageField(blank=True)

class UserSettings(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='settings')
    email_notifications = models.BooleanField(default=True)
    theme = models.CharField(max_length=20, default='light')
```

> 💡 **Why:** Fat views are untestable. Business logic in templates is invisible and undebuggable. God models violate SRP and become maintenance nightmares. Keep views thin, models focused, and logic in services.
