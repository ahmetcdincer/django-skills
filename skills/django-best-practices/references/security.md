Django security best practices covering CSRF, XSS, SQL injection, clickjacking, password hashing, HTTPS, and middleware configuration.

## 14. Django Security

### CSRF Protection

**Wrong:**
```python
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # Disabling CSRF on a state-changing view
def update_profile(request):
    request.user.name = request.POST['name']
    request.user.save()
    return JsonResponse({'status': 'ok'})
```

**Correct:**
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

> **Why:** Only use `@csrf_exempt` for external webhooks that have their own authentication (Stripe, GitHub). Every other POST/PUT/DELETE needs CSRF protection.

### XSS Protection

**Wrong:**
```python
from django.utils.safestring import mark_safe

def render_comment(comment):
    # Marking user input as safe — XSS vulnerability
    return mark_safe(f'<div class="comment">{comment.body}</div>')
```

**Correct:**
```python
from django.utils.html import format_html


def render_comment(comment):
    # format_html escapes the variable parts while keeping the HTML structure
    return format_html('<div class="comment">{}</div>', comment.body)

# In templates — autoescaping is on by default
# {{ comment.body }}  ← Escaped automatically
# {{ comment.body|escape }}  ← Explicitly escaped (same as default)
```

> **Why:** Django autoescapes template variables by default. Use `format_html()` in Python code to safely build HTML. Never use `mark_safe()` on user input.

### SQL Injection

**Wrong:**
```python
from django.db import connection

def search(request):
    query = request.GET['q']
    cursor = connection.cursor()
    cursor.execute(f"SELECT * FROM products WHERE name = '{query}'")
    # SQL injection — user can input: ' OR 1=1; DROP TABLE products; --
```

**Correct:**
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

> **Why:** The ORM parameterizes all queries automatically. For raw SQL, always use `%s` placeholders and pass parameters as a list — never use f-strings or `.format()`.

### Clickjacking Protection

**Wrong:**
```python
# No X-Frame-Options — site can be embedded in iframes (clickjacking)
MIDDLEWARE = [
    # XFrameOptionsMiddleware missing
]
```

**Correct:**
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

> **Why:** `X-Frame-Options: DENY` prevents your pages from being embedded in iframes, blocking clickjacking attacks. Use `SAMEORIGIN` if you embed your own content.

### Password Hashing

**Wrong:**
```python
# Using MD5 or SHA1 for passwords
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()
# Or using the default PBKDF2 when Argon2 is available
```

**Correct:**
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

> **Why:** Argon2 is the winner of the Password Hashing Competition — memory-hard and resistant to GPU attacks. List PBKDF2 as fallback so existing passwords still verify during migration.

### HTTPS Settings

**Wrong:**
```python
# Production without HTTPS enforcement
DEBUG = False
# No HSTS, no SSL redirect, cookies sent over HTTP
```

**Correct:**
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

> **Why:** HSTS tells browsers to always use HTTPS. `SECURE_SSL_REDIRECT` catches HTTP requests. `SECURE_PROXY_SSL_HEADER` is needed when Nginx/ALB handles TLS termination.

### SECRET_KEY Management

**Wrong:**
```python
# settings.py — committed to git
SECRET_KEY = 'django-insecure-abc123456789realkey'
```

**Correct:**
```python
import os

# Read from environment variable
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']

# Generate a new key:
# python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

> **Why:** SECRET_KEY is used for signing cookies, tokens, and password reset links. If leaked, attackers can forge sessions. Store it in environment variables or a secrets manager.

### DEBUG=False Production Checks

**Wrong:**
```python
# Deploying without running Django's deployment checks
# DEBUG = True in production — exposes full tracebacks, settings, SQL queries
```

**Correct:**
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

> **Why:** `manage.py check --deploy` catches common security misconfigurations. `ALLOWED_HOSTS` prevents HTTP Host header attacks. Set up error logging so you see exceptions without DEBUG.

### Security Middleware Order

**Wrong:**
```python
# SecurityMiddleware buried in the middle or missing entirely
MIDDLEWARE = [
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',  # Too late
]
```

**Correct:**
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

> **Why:** SecurityMiddleware must be first to set security headers (HSTS, SSL redirect) before any other processing. CorsMiddleware must come before CommonMiddleware to handle preflight requests.
