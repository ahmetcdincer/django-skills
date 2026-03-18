Django Templates best practices: template inheritance, built-in tags and filters, custom template tags, context processors, and template security.

## Template Inheritance

**Wrong:**
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

**Correct:**
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

> **Why:** Template inheritance eliminates duplication. `base.html` defines the skeleton, child templates override specific blocks. Use `{% include %}` for reusable fragments.

## Template Tags

**Wrong:**
```html
<!-- Hardcoded URLs and static paths -->
<a href="/products/{{ product.id }}/">{{ product.name }}</a>
<img src="/static/images/logo.png">

<!-- No empty list handling -->
{% for item in items %}
  <li>{{ item.name }}</li>
{% endfor %}
```

**Correct:**
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

> **Why:** `{% url %}` and `{% static %}` generate correct URLs regardless of deployment config. `{% empty %}` handles the no-results case. `{% with %}` caches computed values.

## Template Filters

**Wrong:**
```html
<!-- Formatting in the view instead of the template -->
<!-- views.py: context['date'] = obj.created_at.strftime('%B %d, %Y') -->
<p>{{ date }}</p>

<!-- Truncating in Python -->
<!-- views.py: context['desc'] = obj.description[:100] + '...' -->
<p>{{ desc }}</p>
```

**Correct:**
```html
<!-- Built-in filters handle formatting -->
<p>{{ article.created_at|date:"F j, Y" }}</p>
<p>{{ article.description|truncatewords:20 }}</p>
<p>{{ article.body|linebreaks }}</p>
<p>{{ price|floatformat:2 }}</p>
<p>{{ name|default:"Anonymous" }}</p>
<p>{{ count|pluralize:"y,ies" }}</p>
```

> **Why:** Template filters keep presentation logic in templates where it belongs. The view provides raw data, the template formats it for display.

## Custom Template Tags

**Wrong:**
```python
# Passing computed data through every single view's context
# views.py
def product_list(request):
    return render(request, 'products/list.html', {
        'products': Product.objects.all(),
        'cart_count': request.session.get('cart', {}).items().__len__(),  # Repeated everywhere
    })
```

**Correct:**
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

> **Why:** Custom tags and filters encapsulate reusable template logic. `simple_tag` for computed values, `inclusion_tag` for reusable template fragments, `filter` for value transformation.

## Context Processors

**Wrong:**
```python
# Adding site_name to every view manually
def home(request):
    return render(request, 'home.html', {'site_name': 'My Site', ...})

def about(request):
    return render(request, 'about.html', {'site_name': 'My Site', ...})
```

**Correct:**
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

> **Why:** Context processors inject variables into every template automatically. Use them for truly global data (site name, feature flags) — not for view-specific data.

## Template Security

**Wrong:**
```html
<!-- Disabling autoescaping carelessly -->
{% autoescape off %}
  {{ user_comment }}  <!-- XSS vulnerability if comment contains <script> -->
{% endautoescape %}

{{ user_bio|safe }}  <!-- Also dangerous — marks raw HTML as safe -->
```

**Correct:**
```html
<!-- Django autoescapes by default — trust it -->
<p>{{ user_comment }}</p>  <!-- <script> tags are escaped automatically -->

<!-- Only use |safe for content YOU control, never user input -->
{{ admin_announcement|safe }}  <!-- Only if set by trusted staff in admin -->

<!-- For user-generated HTML, use bleach to sanitize first -->
<!-- In the view: cleaned = bleach.clean(user_input, tags=['b', 'i', 'a']) -->
<p>{{ cleaned_html|safe }}</p>
```

> **Why:** Django's autoescaping prevents XSS by default. Never use `|safe` or `{% autoescape off %}` on user-supplied content. Sanitize HTML with a library like bleach before marking it safe.
