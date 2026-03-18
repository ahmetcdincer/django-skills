Django Views best practices: FBV and CBV structure, generic views, mixins, and choosing between function-based and class-based views.

## Function-Based View Structure

**Wrong:**
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

**Correct:**
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

> **Why:** Separate views per action, use `require_http_methods` to restrict HTTP methods, use forms for validation, and always return proper responses from templates.

## Class-Based View Structure

**Wrong:**
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

**Correct:**
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

> **Why:** One view per action. CBVs map HTTP methods to class methods (`get`, `post`, `put`, `delete`). Don't overload a single view to handle everything.

## TemplateView, ListView, DetailView

**Wrong:**
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

**Correct:**
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

> **Why:** Generic views handle pagination, 404s, and context setup out of the box. Override `get_queryset()` or `get_context_data()` to customize behavior.

## CreateView, UpdateView, DeleteView

**Wrong:**
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

**Correct:**
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

> **Why:** Generic editing views handle form rendering, validation, and redirects. Override `get_queryset()` to enforce ownership — users should only edit their own objects.

## FormView

**Wrong:**
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

**Correct:**
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

> **Why:** FormView handles the GET/POST branching and re-rendering with errors. Override `form_valid()` for the success action and `form_invalid()` for custom error handling.

## RedirectView

**Wrong:**
```python
from django.shortcuts import redirect

# A whole view function just to redirect
def old_page(request):
    return redirect('/new-page/')
```

**Correct:**
```python
from django.views.generic import RedirectView
from django.urls import path

urlpatterns = [
    path('old-page/', RedirectView.as_view(pattern_name='pages:new', permanent=True)),
]
```

> **Why:** RedirectView is declarative and can be defined directly in URL config. Use `permanent=True` for 301 (SEO redirect) and `permanent=False` for 302 (temporary).

## Mixin Usage

**Wrong:**
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

**Correct:**
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

> **Why:** Mixins are reusable and declarative. `LoginRequiredMixin` must come before the view class in MRO. `PermissionRequiredMixin` handles both authentication and permission checks.

## FBV vs CBV Decision Criteria

**Wrong:**
```python
# Using CBV for a simple one-off view that doesn't benefit from inheritance
from django.views import View
from django.http import JsonResponse

class HealthCheckView(View):
    def get(self, request):
        return JsonResponse({'status': 'ok'})
```

**Correct:**
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

> **Why:** Use FBVs for simple, one-off views (health checks, webhooks, custom logic). Use CBVs when generic views save boilerplate (CRUD, list/detail). Don't force CBV on everything.
