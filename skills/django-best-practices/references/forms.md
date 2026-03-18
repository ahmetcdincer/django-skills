Django Forms best practices: form classes, ModelForms, validation, custom validators, formsets, and form security.

## Django Forms

**Wrong:**
```python
# Manually parsing request.POST without validation
def contact(request):
    if request.method == 'POST':
        name = request.POST.get('name', '')
        email = request.POST.get('email', '')
        # No validation, no error messages, no CSRF
        send_email(name, email)
```

**Correct:**
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

> **Why:** Django forms validate input, provide error messages, and render HTML widgets. Always access `cleaned_data` after validation — never trust raw `request.POST`.

## ModelForms

**Wrong:**
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

**Correct:**
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

> **Why:** Always use explicit `fields` — never `exclude` or `'__all__'`. Exclude can accidentally expose sensitive fields added later. Override `save()` to set computed fields.

## Form Validation

**Wrong:**
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

**Correct:**
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

> **Why:** `clean_<field>()` validates individual fields. `clean()` validates cross-field logic. Both raise `ValidationError` which Django renders as error messages on the form.

## Custom Validators

**Wrong:**
```python
from django.db import models

class Product(models.Model):
    price = models.DecimalField(max_digits=10, decimal_places=2)
    # No validation — allows negative prices
```

**Correct:**
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

> **Why:** Validators are reusable across models and forms. Use Django's built-in validators (`MinValueValidator`, `MaxLengthValidator`, `RegexValidator`) and write custom ones for domain-specific rules.

## Formsets

**Wrong:**
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

**Correct:**
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

> **Why:** Formsets handle multiple instances of the same form with proper validation. Always render `management_form` — it contains the hidden fields Django needs to track form count.

## Form Security

**Wrong:**
```python
# Disabling CSRF for convenience
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt  # Security vulnerability!
def payment_webhook(request):
    # Accepting file uploads without validation
    uploaded = request.FILES['document']
    uploaded.save('/uploads/' + uploaded.name)  # Path traversal + no type check
```

**Correct:**
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

> **Why:** Never disable CSRF unless the endpoint is called by external systems (webhooks with their own auth). Validate file uploads by extension, size, and MIME type to prevent malicious uploads.
