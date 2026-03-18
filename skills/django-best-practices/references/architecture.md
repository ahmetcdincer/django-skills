Advanced Django internals and architecture patterns covering custom fields, multi-DB, routers, template engines, ORM internals, signals, management commands, MTV, app design, service layer, repository pattern, DDD, SOLID, and anti-patterns.

## 26. Advanced Django

### Custom Model Fields

**Wrong:**
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

**Correct:**
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

> **Why:** `JSONField` is built-in since Django 3.1 — don't store JSON as text. For truly custom fields, implement `from_db_value`, `to_python`, and `get_prep_value` for the DB-to-Python-to-DB cycle.

### Multi-Database Support

**Wrong:**
```python
# Hardcoding database selection in views
from django.db import connections

def analytics_view(request):
    cursor = connections['analytics'].cursor()
    cursor.execute('SELECT ...')
    # Manual connection management — error-prone
```

**Correct:**
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

> **Why:** Database routers automate read/write routing based on models. Queries work normally — the router decides which database to use. Use `using()` for explicit override.

### Database Router

**Wrong:**
```python
# A router that doesn't handle all methods — unexpected behavior
class MyRouter:
    def db_for_read(self, model, **hints):
        return 'replica'
    # Missing db_for_write, allow_relation, allow_migrate
```

**Correct:**
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

> **Why:** A complete router implements all four methods. `allow_relation` prevents cross-database foreign keys. `allow_migrate` ensures migrations only run on the primary. `TEST.MIRROR` handles the replica in tests.

### Custom Template Engine

**Wrong:**
```python
# Overriding Django's template engine for minor customizations
TEMPLATES = [{
    'BACKEND': 'django.template.backends.django.DjangoTemplates',
    # Replacing the whole engine when a custom tag would suffice
}]
```

**Correct:**
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

> **Why:** You can use Jinja2 and Django templates side-by-side. Jinja2 is faster and has more powerful expressions. Use Django templates for admin and contrib apps, Jinja2 for your custom templates.

### ORM Internals

**Wrong:**
```python
# Overriding queryset behavior without understanding the SQL compiler
class MyQuerySet(models.QuerySet):
    def custom_filter(self):
        # Building raw SQL strings instead of using the query API
        return self.raw('SELECT * FROM ...')
```

**Correct:**
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

> **Why:** Understanding `QuerySet.query` helps debug slow queries. Custom lookups extend the ORM's filter syntax. Print `.query` to see the generated SQL before optimizing.

### Signal Architecture

**Wrong:**
```python
# Signals that are impossible to test in isolation
@receiver(post_save, sender=Order)
def complex_side_effect(sender, instance, **kwargs):
    send_email(instance)
    update_inventory(instance)
    notify_warehouse(instance)
    # Can't test the view without all side effects firing
```

**Correct:**
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

> **Why:** Keep signal handlers thin — delegate to tasks or services. Mock signals in tests that don't need them. This keeps tests fast and focused.

### Custom Management Commands

**Wrong:**
```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    def handle(self, *args, **options):
        # No arguments, no error handling, no output styling
        from myapp.models import Order
        Order.objects.filter(status='expired').delete()
        print('done')  # Not using self.stdout
```

**Correct:**
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

> **Why:** Management commands should accept arguments, support `--dry-run`, use `self.stdout` with styling, and handle edge cases. They're often run in production — make them safe.

## 27. Django Architecture

### MTV Pattern

**Wrong:**
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

**Correct:**
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

> **Why:** Model = data + business rules. Template = presentation. View = HTTP handling + glue. Keep templates dumb — they should only display data, not compute it.

### App Modularization

**Wrong:**
```python
# One monolithic app doing everything
myproject/
    shop/
        models.py    # User, Product, Order, Payment, Review, Notification, BlogPost
        views.py     # 100+ views
        admin.py     # 30+ admin classes
```

**Correct:**
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

> **Why:** Split apps along domain boundaries (bounded contexts). An app should have 3-8 models. If it has 15+ models or its `models.py` exceeds 500 lines, it's time to split.

### Service Layer

**Wrong:**
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

**Correct:**
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

> **Why:** Services encapsulate business logic in testable, reusable functions. Views handle HTTP; services handle business rules. Test services without HTTP overhead.

### Repository Pattern

**Wrong:**
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

**Correct:**
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

> **Why:** Django's ORM is already a repository + query builder. Adding another layer just adds indirection. Use custom QuerySet methods for reusable filters. Reserve the pattern for multi-source data.

### Domain Driven Design (DDD)

**Wrong:**
```python
# Forcing full DDD patterns on a simple Django project
# Separate value objects, aggregates, repositories, domain events...
# for a blog with 5 models — massive over-engineering
```

**Correct:**
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

> **Why:** Apply DDD concepts where they add clarity — aggregate roots for consistency boundaries, value objects for immutable domain concepts. Don't force full DDD on simple CRUD apps.

### SOLID Principles in Django

**Wrong:**
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

**Correct:**
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

> **Why:** SOLID prevents god classes and tangled dependencies. In Django: thin views (SRP), service classes with injectable dependencies (DIP), and new behavior via new classes rather than modifying existing ones (OCP).

### Anti-Patterns

**Wrong:**
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

**Correct:**
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

> **Why:** Fat views are untestable. Business logic in templates is invisible and undebuggable. God models violate SRP and become maintenance nightmares. Keep views thin, models focused, and logic in services.
