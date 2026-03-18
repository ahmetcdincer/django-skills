# Django Models & ORM Best Practices

Reference for model definition, field types, relationships, meta options, methods, managers, abstract/proxy models, and custom QuerySet methods.

## Model Definition

**Wrong:**
```python
from django.db import models

class product(models.Model):
    n = models.CharField(max_length=100)
    d = models.CharField(max_length=5000)
    p = models.FloatField()
    active = models.CharField(max_length=5, default='yes')
```

**Correct:**
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

> **Why:** Use descriptive field names, correct field types (DecimalField for money, BooleanField for flags, TextField for long text), and verbose_name for admin/i18n support.

## Field Types

**Wrong:**
```python
from django.db import models

class Event(models.Model):
    title = models.TextField()  # Short text in TextField
    price = models.FloatField()  # Float for money — rounding errors
    event_date = models.DateTimeField()  # Only need date, not time
    email = models.CharField(max_length=255)  # No validation
```

**Correct:**
```python
from django.db import models


class Event(models.Model):
    title = models.CharField(max_length=255)  # CharField for short text
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Exact precision
    event_date = models.DateField()  # DateField when time isn't needed
    email = models.EmailField()  # Built-in email validation
```

> **Why:** CharField has max_length enforced at DB level. FloatField causes rounding errors with currency. DateField vs DateTimeField — pick what you actually need.

## ForeignKey

**Wrong:**
```python
from django.db import models

class Comment(models.Model):
    post = models.ForeignKey('Post', on_delete=models.CASCADE)
    # No related_name — default is comment_set
    # No db_index consideration
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    # Hardcoded auth.User — breaks with custom user model
```

**Correct:**
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

> **Why:** Always use `settings.AUTH_USER_MODEL` for user references. Explicit `related_name` makes reverse queries readable: `post.comments.all()` instead of `post.comment_set.all()`.

## ManyToMany with Through Model

**Wrong:**
```python
from django.db import models

class Project(models.Model):
    name = models.CharField(max_length=255)
    members = models.ManyToManyField('auth.User')
    # No way to store role, date_joined, or any extra data
```

**Correct:**
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

> **Why:** Use a through model when you need metadata on the relationship (role, timestamps, permissions). You almost always need it eventually — add it early.

## OneToOneField

**Wrong:**
```python
from django.conf import settings
from django.db import models

class Profile(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    # ForeignKey allows multiple profiles per user — probably a bug
    bio = models.TextField(blank=True)
```

**Correct:**
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

> **Why:** OneToOneField enforces the one-to-one constraint at the database level. Access is cleaner too: `user.profile` instead of `user.profile_set.first()`.

## Model Meta

**Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
    # No ordering, no indexes, no constraints
```

**Correct:**
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

> **Why:** Indexes speed up queries you run often. Constraints enforce data integrity at the DB level. Default ordering saves repeating `.order_by()` everywhere.

## Model Methods

**Wrong:**
```python
from django.db import models

class Order(models.Model):
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)
    # No __str__ — admin shows "Order object (1)"
    # No get_absolute_url — templates hardcode URLs
```

**Correct:**
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

> **Why:** `__str__` makes admin and debugging readable. `get_absolute_url` is used by Django's redirect shortcuts and admin. Methods encapsulate business logic on the model.

## Model Managers

**Wrong:**
```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=255)
    is_published = models.BooleanField(default=False)

# In views — filtering repeated everywhere
# Article.objects.filter(is_published=True)
# Article.objects.filter(is_published=True, category='tech')
```

**Correct:**
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

> **Why:** Custom managers encapsulate common filters, keeping views DRY. Always keep `objects` as the default manager to avoid surprises with admin and related lookups.

## Abstract Models

**Wrong:**
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

**Correct:**
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

> **Why:** Abstract models eliminate field duplication across models. `abstract = True` means no database table is created for the base class — fields are added to each child table.

## Proxy Models

**Wrong:**
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

**Correct:**
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

> **Why:** Proxy models share the same database table but allow different Python behavior, managers, and admin registrations. No data duplication, no migration overhead.

## Custom QuerySet Methods

**Wrong:**
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

**Correct:**
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

> **Why:** QuerySet methods are chainable and reusable. `as_manager()` turns a QuerySet into a manager, giving you both custom methods and full QuerySet API.
