Django Middleware best practices: request/response lifecycle, built-in ordering, custom middleware, exception handling, and async support.

## Request/Response Lifecycle

**Wrong:**
```python
# Putting timing logic directly in every view
import time

def my_view(request):
    start = time.time()
    # ... view logic ...
    duration = time.time() - start
    print(f'View took {duration}s')
```

**Correct:**
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

> **Why:** Middleware intercepts every request/response. Cross-cutting concerns like logging, timing, and headers belong in middleware, not duplicated across views.

## Built-in Middleware Order

**Wrong:**
```python
MIDDLEWARE = [
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.security.SecurityMiddleware',
    # Wrong order — SecurityMiddleware should be first, Session before Auth
]
```

**Correct:**
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

> **Why:** SecurityMiddleware must be first to set security headers. SessionMiddleware must precede AuthenticationMiddleware. The documented order exists for a reason — follow it.

## Custom Middleware

**Wrong:**
```python
# Old-style middleware class (pre-Django 1.10)
class OldMiddleware:
    def process_request(self, request):
        ...
    def process_response(self, request, response):
        ...
```

**Correct:**
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

> **Why:** Modern middleware uses `__init__` + `__call__`. The class receives `get_response` once at startup. Functional middleware is a shortcut for simple cases.

## Exception Handling in Middleware

**Wrong:**
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

**Correct:**
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

> **Why:** `process_exception` is called when a view raises an exception. Log the error, then return `None` to let Django's error handling proceed (including DEBUG pages and error reporters).

## Async Middleware (Django 4.1+)

**Wrong:**
```python
# Using sync middleware with async views — Django wraps it in sync_to_async
# This works but adds overhead from thread pool context switches
class SyncMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        return self.get_response(request)
```

**Correct:**
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

> **Why:** Async middleware avoids the overhead of `sync_to_async` wrapping. Set `async_capable = True` and implement `__acall__` for the async path.
