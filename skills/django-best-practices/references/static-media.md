Django static files, media files, and file handling best practices including storage backends, uploads, and security.

## 13. Static & Media Files

### STATIC_URL, STATIC_ROOT, STATICFILES_DIRS

**Wrong:**
```python
# Confusing STATIC_ROOT with STATICFILES_DIRS
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
# ^ Same directory for both — collectstatic will fail with an error
```

**Correct:**
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

> **Why:** `STATICFILES_DIRS` is where you put your source static files. `STATIC_ROOT` is where `collectstatic` copies everything for production. They must be different directories.

### MEDIA_URL and MEDIA_ROOT

**Wrong:**
```python
# Serving media files from STATIC_ROOT
# Or hardcoding absolute paths
MEDIA_ROOT = 'C:/Users/dev/project/media'  # Not portable
```

**Correct:**
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

> **Why:** `MEDIA_ROOT` stores user-uploaded files. Serve them via Django only in development — in production, use Nginx or a CDN. Use relative paths for portability.

### collectstatic Production Workflow

**Wrong:**
```bash
# Committing collected static files to git
git add staticfiles/
# Or running collectstatic on every request
```

**Correct:**
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

> **Why:** `collectstatic` gathers static files from all apps into `STATIC_ROOT` for the web server. `ManifestStaticFilesStorage` adds content hashes for browser cache invalidation.

### WhiteNoise

**Wrong:**
```python
# Using Django's development server to serve static files in production
# Or configuring Nginx just for static files on a small app
```

**Correct:**
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

> **Why:** WhiteNoise serves static files directly from Python with compression and caching headers. Perfect for Heroku, Docker, and small-to-medium apps. No Nginx needed for static files.

### S3/CDN Integration

**Wrong:**
```python
# Writing custom S3 upload code with boto3 in every view
import boto3

def upload_file(request):
    s3 = boto3.client('s3', aws_access_key_id='AKIA...',
                      aws_secret_access_key='secret...')
    s3.upload_fileobj(request.FILES['file'], 'my-bucket', 'path/file.jpg')
```

**Correct:**
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

> **Why:** django-storages abstracts the storage backend. Your models and forms don't change — `FileField.save()` and `default_storage.open()` work the same whether it's local or S3.

### Development vs Production Differences

**Wrong:**
```python
# Same static file configuration for development and production
# Running collectstatic in development
# Serving media files through Nginx in development
```

**Correct:**
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

> **Why:** Development uses Django's built-in static serving. Production uses WhiteNoise, Nginx, or S3/CDN. Never use `runserver` or DEBUG=True in production.

## 17. File Handling

### FileField and ImageField

**Wrong:**
```python
from django.db import models

class Document(models.Model):
    file = models.FileField()  # Uploads to MEDIA_ROOT root — messy
    # No upload_to, no size validation
```

**Correct:**
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

> **Why:** Use callable `upload_to` for dynamic paths (prevents filename collisions with UUID). Validate extensions at the model level. ImageField requires Pillow to be installed.

### Storage API

**Wrong:**
```python
# Hardcoding file paths and using Python's open() directly
import os

def save_file(uploaded_file):
    path = f'/var/www/media/docs/{uploaded_file.name}'
    with open(path, 'wb') as f:
        for chunk in uploaded_file.chunks():
            f.write(chunk)
```

**Correct:**
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

> **Why:** `default_storage` abstracts the backend — works with local filesystem, S3, GCS, or any custom storage. Switching backends doesn't require code changes.

### Custom Storage Backends

**Wrong:**
```python
# Writing boto3 upload code directly in views
import boto3

def upload_view(request):
    s3 = boto3.client('s3')
    s3.upload_fileobj(request.FILES['file'], 'bucket', 'key')
```

**Correct:**
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

> **Why:** django-storages integrates with Django's storage API. Custom storage classes let you use different backends (public vs private) on the same project without changing model code.

### File Upload Security

**Wrong:**
```python
def upload(request):
    f = request.FILES['file']
    # No type checking — user can upload .exe, .sh, etc.
    # No size checking — user can upload 10 GB files
    f.save('/uploads/' + f.name)  # Path traversal: name could be "../../../etc/passwd"
```

**Correct:**
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

> **Why:** Validate file extension, size, and actual MIME type (via magic bytes). Never trust the filename or Content-Type header. Set server-level size limits as a safety net.

### Pillow Usage

**Wrong:**
```python
from django.db import models

class Photo(models.Model):
    image = models.ImageField(upload_to='photos/')
    # Pillow not installed — ImageField won't validate image files
    # No image processing — uploaded 10 MB photos served as-is
```

**Correct:**
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

> **Why:** Pillow is required for `ImageField` validation. Use it to create thumbnails and optimize images on upload instead of serving raw multi-megabyte images.
