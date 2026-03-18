Django Admin best practices: ModelAdmin configuration, inlines, actions, fieldsets, custom views, security, and performance.

## ModelAdmin Basics

**Wrong:**
```python
from django.contrib import admin
from .models import Product

admin.site.register(Product)
# Bare registration — no list display, no filters, no search
```

**Correct:**
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

> **Why:** A well-configured admin with `list_display`, `list_filter`, and `search_fields` turns the admin into a usable internal tool instead of a list of "Object (1)" links.

## Inline Models

**Wrong:**
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

**Correct:**
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

> **Why:** Inlines let you edit parent and child models on the same page. Use `TabularInline` for compact rows, `StackedInline` for larger forms with more fields.

## Custom Admin Actions

**Wrong:**
```python
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ('title', 'status')
    # No bulk actions — admin users must edit articles one by one to publish them
```

**Correct:**
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

> **Why:** Admin actions let staff perform bulk operations. Always show feedback via `messages` so users know what happened.

## Readonly Fields

**Wrong:**
```python
from django.contrib import admin
from .models import Order

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'total', 'status')
    # Staff can edit total and created_at — these should be computed/automatic
```

**Correct:**
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

> **Why:** Use `readonly_fields` for computed values, timestamps, and fields that should not be manually edited. You can include method names to display derived data.

## Fieldsets

**Wrong:**
```python
from django.contrib import admin
from .models import Product

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    pass  # All fields in a single flat form — hard to navigate with many fields
```

**Correct:**
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

> **Why:** Fieldsets organize forms into logical groups. Use `collapse` for rarely-used sections. `prepopulated_fields` auto-fills slugs from titles in the admin.

## Admin Forms

**Wrong:**
```python
from django.contrib import admin
from .models import Event

@admin.register(Event)
class EventAdmin(admin.ModelAdmin):
    pass
    # No custom validation — admin allows end_date before start_date
```

**Correct:**
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

> **Why:** Custom admin forms add validation rules that the database constraints alone can't express. The admin is a data entry tool — validate accordingly.

## Custom Admin Views

**Wrong:**
```python
# Creating a separate Django view and URL outside the admin for a report
# urls.py
from myapp.views import sales_report
urlpatterns = [
    path('admin/sales-report/', sales_report),  # No auth, no admin integration
]
```

**Correct:**
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

> **Why:** `get_urls()` adds custom views inside the admin with proper authentication. `admin_site.admin_view()` wraps the view with admin permission checks.

## Admin Security

**Wrong:**
```python
from django.contrib import admin
from .models import User

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('username', 'email', 'is_superuser')
    # All staff can see all users, modify superuser status, etc.
```

**Correct:**
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

> **Why:** Override permission methods to restrict what staff users can see and do. Non-superusers shouldn't be able to grant superuser access or delete accounts.

## Admin Performance

**Wrong:**
```python
from django.contrib import admin
from .models import Order

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ('id', 'customer_name', 'total')

    def customer_name(self, obj):
        return obj.customer.name  # N+1 query — one per row
```

**Correct:**
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

> **Why:** `list_select_related` prevents N+1 queries in list views. `show_full_result_count = False` avoids a slow COUNT(*) query on tables with millions of rows.
