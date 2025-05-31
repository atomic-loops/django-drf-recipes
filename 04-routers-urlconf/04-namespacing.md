# Namespacing APIs

## Problem

You need to organize your API endpoints into logical groups, avoid URL name conflicts, and make URL reversing more explicit in a large Django REST Framework project.

## Solution

Use Django's URL namespacing to organize your API endpoints into logical groups and provide clear, unambiguous URL names.

## Code

### 1. Basic URL Namespacing

The simplest form of namespacing involves using the `namespace` parameter when including URLs:

```python
# main urls.py
from django.urls import path, include

urlpatterns = [
    path('api/books/', include('books.urls', namespace='books')),
    path('api/authors/', include('authors.urls', namespace='authors')),
]
```

```python
# books/urls.py
from django.urls import path
from . import views

app_name = 'books'  # This is required for namespacing

urlpatterns = [
    path('', views.book_list, name='list'),
    path('<int:pk>/', views.book_detail, name='detail'),
]
```

Now you can refer to these URLs using the namespace:

```python
# In a view or template
from django.urls import reverse

url = reverse('books:list')  # Returns '/api/books/'
detail_url = reverse('books:detail', kwargs={'pk': 1})  # Returns '/api/books/1/'
```

### 2. Namespacing with Routers

You can also use namespaces with DRF routers:

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from . import views

router = routers.DefaultRouter()
router.register(r'books', views.BookViewSet)

urlpatterns = [
    path('api/', include((router.urls, 'books'), namespace='books')),
]
```

Now you can reverse router URLs with namespaces:

```python
from rest_framework.reverse import reverse

url = reverse('books:book-list')  # Returns '/api/books/'
detail_url = reverse('books:book-detail', kwargs={'pk': 1})  # Returns '/api/books/1/'
```

### 3. Nested Namespaces

For more complex APIs, you can nest namespaces:

```python
# main urls.py
from django.urls import path, include

urlpatterns = [
    path('api/v1/', include('api.v1.urls', namespace='v1')),
    path('api/v2/', include('api.v2.urls', namespace='v2')),
]
```

```python
# api/v1/urls.py
from django.urls import path, include

app_name = 'v1'

urlpatterns = [
    path('books/', include('books.urls', namespace='books')),
    path('authors/', include('authors.urls', namespace='authors')),
]
```

```python
# books/urls.py
from django.urls import path
from . import views

app_name = 'books'

urlpatterns = [
    path('', views.book_list, name='list'),
    path('<int:pk>/', views.book_detail, name='detail'),
]
```

To reverse these nested namespaces:

```python
from django.urls import reverse

# For API v1
url = reverse('v1:books:list')  # Returns '/api/v1/books/'

# For API v2 (assuming similar structure)
url = reverse('v2:books:list')  # Returns '/api/v2/books/'
```

### 4. API Versioning with Namespaces

You can use namespaces for API versioning:

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from .views_v1 import BookViewSetV1
from .views_v2 import BookViewSetV2

# Router for API v1
router_v1 = routers.DefaultRouter()
router_v1.register(r'books', BookViewSetV1)

# Router for API v2
router_v2 = routers.DefaultRouter()
router_v2.register(r'books', BookViewSetV2)

urlpatterns = [
    path('api/v1/', include((router_v1.urls, 'v1'), namespace='v1')),
    path('api/v2/', include((router_v2.urls, 'v2'), namespace='v2')),
]
```

Now you can use version-specific URL reversing:

```python
from rest_framework.reverse import reverse

# For API v1
url_v1 = reverse('v1:book-list')  # Returns '/api/v1/books/'

# For API v2
url_v2 = reverse('v2:book-list')  # Returns '/api/v2/books/'
```

### 5. Accessing Current Namespace in Views

You can access the current namespace in your views:

```python
# views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from django.urls import reverse

class BookList(APIView):
    def get(self, request, format=None):
        # Get the current namespace
        current_namespace = request.resolver_match.namespace
        
        # Create URLs using the current namespace
        example_url = reverse(f'{current_namespace}:book-detail', kwargs={'pk': 1})
        
        return Response({
            'message': f'You are using the {current_namespace} API',
            'example_url': example_url
        })
```

### 6. Using Namespaces in Serializers

You can use namespaces in serializers to create URLs:

```python
# serializers.py
from rest_framework import serializers
from rest_framework.reverse import reverse
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    url = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'url']
    
    def get_url(self, obj):
        # Get the namespace from the request
        request = self.context.get('request')
        if request is None:
            return None
        
        namespace = request.resolver_match.namespace
        return reverse(f'{namespace}:book-detail', kwargs={'pk': obj.pk}, request=request)
```

### 7. Resource-Specific Namespaces

You can organize namespaces by resource type:

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from . import views

# Books router
books_router = routers.DefaultRouter()
books_router.register(r'', views.BookViewSet)

# Authors router
authors_router = routers.DefaultRouter()
authors_router.register(r'', views.AuthorViewSet)

urlpatterns = [
    path('api/books/', include((books_router.urls, 'books'), namespace='books')),
    path('api/authors/', include((authors_router.urls, 'authors'), namespace='authors')),
]
```

### 8. Feature-Based Namespaces

You can organize namespaces by feature or module:

```python
# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/catalog/', include('catalog.urls', namespace='catalog')),
    path('api/accounts/', include('accounts.urls', namespace='accounts')),
    path('api/orders/', include('orders.urls', namespace='orders')),
]
```

### 9. Company-Specific API Namespaces

For multi-tenant applications, you can namespace by company:

```python
# urls.py
from django.urls import path, include
from . import views

urlpatterns = [
    # Dynamic company-specific APIs
    path('api/<str:company_slug>/', views.company_router, name='company_api'),
    
    # General APIs
    path('api/common/', include('common.urls', namespace='common')),
]
```

```python
# views.py
from django.urls import include
from rest_framework import routers
from . import viewsets

def company_router(request, company_slug):
    # Create a router specific to this company
    router = routers.DefaultRouter()
    router.register(r'users', viewsets.CompanyUserViewSet)
    router.register(r'projects', viewsets.CompanyProjectViewSet)
    
    # Set company context
    request.company_slug = company_slug
    
    # Include the router URLs with a company-specific namespace
    return include((router.urls, company_slug), namespace=company_slug)(request)
```

### 10. Documenting Namespaced APIs

You can include namespaces in your API documentation:

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from rest_framework.schemas import get_schema_view
from rest_framework.documentation import include_docs_urls
from . import views

router = routers.DefaultRouter()
router.register(r'books', views.BookViewSet)

# Schema view with namespace
schema_view = get_schema_view(
    title="Books API",
    description="API for books",
    patterns=[path('api/', include((router.urls, 'books'), namespace='books'))]
)

urlpatterns = [
    path('api/', include((router.urls, 'books'), namespace='books')),
    path('api/schema/', schema_view, name='schema'),
    path('api/docs/', include_docs_urls(title="Books API")),
]
```

## Notes

1. **Benefits of Namespacing**:
   - Avoids URL name conflicts in large projects
   - Makes URL reversing more explicit and less error-prone
   - Helps organize APIs into logical groups
   - Simplifies versioning and modular development

2. **Naming Conventions**:
   - Use consistent, descriptive namespace names
   - Consider using plural nouns for resource collections (e.g., 'books', 'authors')
   - Use semantic naming for feature-based namespaces (e.g., 'auth', 'admin')
   - Use version indicators for version namespaces (e.g., 'v1', 'v2')

3. **Namespace Inheritance**:
   - Child namespaces inherit their parent's namespace
   - The full namespace is constructed by joining parent and child with a colon
   - Example: 'v1:books:detail'

4. **URL Reversing Best Practices**:
   - Always use namespace-qualified URLs in `reverse()`
   - Import `reverse` from the appropriate module (Django or DRF)
   - Include the `request` parameter when reversing in serializers

5. **Common Pitfalls**:
   - Forgetting to set `app_name` in included URL modules
   - Using incorrect format for router includes (`(router.urls, 'namespace')`)
   - Not accounting for namespace changes when refactoring

6. **Testing Namespaced URLs**:
   - Use `reverse()` in tests to ensure URL patterns work correctly
   - Test URL reversing for all namespaces
   - Consider using `assertURLEqual` for URL comparison

7. **Namespace Organization Strategies**:
   - By resource type (books, authors)
   - By API version (v1, v2)
   - By feature or module (auth, admin, public)
   - By tenant or company (for multi-tenant applications)
