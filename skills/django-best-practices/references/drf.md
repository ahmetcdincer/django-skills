# Django Rest Framework (DRF) best practices for serializers, views, viewsets, routers, authentication, permissions, throttling, pagination, filtering, versioning, and performance.

## Serializers

### Using ModelSerializer over plain Serializer

**Wrong:**
```python
from rest_framework import serializers

class ProductSerializer(serializers.Serializer):
    # Manually defining every field — ignores the model definition
    id = serializers.IntegerField()
    name = serializers.CharField()
    price = serializers.FloatField()  # FloatField for money — rounding issues
    # Missing validation, missing many fields
```

**Correct:**
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

> **Why:** `ModelSerializer` derives fields from the model, reducing duplication. Use `source` for dotted attribute access and `SerializerMethodField` for computed values.

## ModelSerializer Configuration

### Explicit field listing and custom validation

**Wrong:**
```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Exposes password hash, permissions, etc.
```

**Correct:**
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

> **Why:** Never use `fields = '__all__'` — it exposes internal fields. List fields explicitly. Use `extra_kwargs` for field-level overrides and `validate_<field>` for custom validation.

## Nested Serializers

### Separate read and write serializers for nested data

**Wrong:**
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

**Correct:**
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

> **Why:** Use separate serializers for read (nested, rich) and write (flat, writable). Override `create()`/`update()` for writable nested serializers since DRF doesn't handle them automatically.

## Views (APIView and Generics)

### Prefer generic views over raw APIView

**Wrong:**
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

**Correct:**
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

> **Why:** Generic views handle serialization, pagination, status codes, and error responses. Use `APIView` only when generics don't fit your use case.

## ViewSets

### Permission-aware ViewSets with custom actions

**Wrong:**
```python
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    # Exposes create, update, partial_update, destroy without any permission checks
```

**Correct:**
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

> **Why:** ViewSets combine list/create/retrieve/update/destroy into one class. Use `get_permissions()` to vary permissions per action. `@action` adds custom endpoints.

## Routers

### Auto-generating URL patterns for ViewSets

**Wrong:**
```python
from django.urls import path
from .views import ProductViewSet

# Manually mapping ViewSet methods to URLs
urlpatterns = [
    path('products/', ProductViewSet.as_view({'get': 'list', 'post': 'create'})),
    path('products/<int:pk>/', ProductViewSet.as_view({'get': 'retrieve', 'put': 'update', 'delete': 'destroy'})),
]
```

**Correct:**
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

> **Why:** Routers auto-generate URL patterns for ViewSets, including custom `@action` endpoints. `DefaultRouter` adds an API root view listing all endpoints.

## Authentication

### JWT for stateless API authentication

**Wrong:**
```python
# Using SessionAuthentication for a mobile API
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        # Session auth requires cookies — doesn't work for mobile/SPA
    ],
}
```

**Correct:**
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

> **Why:** JWT is stateless — works for mobile, SPAs, and microservices. Keep access tokens short-lived (30 min) and use refresh tokens for renewal. Session auth is fine for the browsable API.

## Permissions

### Global defaults and custom object-level permissions

**Wrong:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response

class OrderView(APIView):
    # No permissions — any anonymous user can access
    def get(self, request):
        return Response(Order.objects.values())
```

**Correct:**
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

> **Why:** Set `IsAuthenticated` as the global default. Create custom permissions for object-level checks. DRF checks all permission classes — all must return True.

## Throttling

### Rate limiting to prevent abuse

**Wrong:**
```python
# No rate limiting — API vulnerable to abuse
REST_FRAMEWORK = {
    # No throttle classes configured
}
```

**Correct:**
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

> **Why:** Throttling prevents abuse and brute-force attacks. Set lower rates for anonymous users and sensitive endpoints (login, password reset). DRF stores throttle state in the cache.

## Pagination

### Always paginate list endpoints

**Wrong:**
```python
# Returning all records at once
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()  # Returns 100,000 products in one response
```

**Correct:**
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

> **Why:** Always paginate list endpoints. `CursorPagination` is best for large datasets (no SQL OFFSET). `PageNumberPagination` is most familiar to API consumers.

## Filtering

### Declarative filtering with django-filter

**Wrong:**
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

**Correct:**
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

> **Why:** django-filter provides declarative filtering. Combine with DRF's `SearchFilter` for full-text search and `OrderingFilter` for sortable columns.

## API Versioning

### URL-based versioning with per-version serializers

**Wrong:**
```python
# No versioning — breaking changes affect all clients immediately
# Or: path('api/products/', ProductListView.as_view())
```

**Correct:**
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

> **Why:** URL-based versioning (`/api/v1/`, `/api/v2/`) is the most explicit and easiest for API consumers. Switch serializers per version to evolve the API without breaking clients.

## SerializerMethodField Security

### Conditionally exposing sensitive data

**Wrong:**
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

**Correct:**
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

> **Why:** SerializerMethodField can access the request via `self.context['request']`. Use it to conditionally expose sensitive data based on the requesting user's identity or permissions.

## DRF Performance

### Optimizing querysets with select_related and prefetch_related

**Wrong:**
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

**Correct:**
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

> **Why:** Optimize `get_queryset()` with `select_related` for FK/O2O, `prefetch_related` for M2M/reverse FK, and `only()` to limit fetched columns. Profile with Django Debug Toolbar.
