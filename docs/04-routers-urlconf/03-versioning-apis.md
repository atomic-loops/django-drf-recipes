# Versioning APIs

## Problem

You need to implement versioning for your API to allow making changes while maintaining backward compatibility for existing clients.

## Solution

Use Django REST Framework's versioning schemes to manage different versions of your API endpoints.

## Code

### 1. Setting Up API Versioning

First, configure versioning in your settings:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

### 2. URL Path Versioning

This approach includes the version in the URL path:

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from .views import BookViewSetV1, BookViewSetV2

# Router for API v1
router_v1 = routers.DefaultRouter()
router_v1.register(r'books', BookViewSetV1)

# Router for API v2
router_v2 = routers.DefaultRouter()
router_v2.register(r'books', BookViewSetV2)

urlpatterns = [
    path('api/v1/', include(router_v1.urls)),
    path('api/v2/', include(router_v2.urls)),
]
```

With corresponding ViewSets:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSetV1(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializerV1

class BookViewSetV2(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializerV2
```

### 3. Query Parameter Versioning

This approach uses a query parameter for versioning:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
```

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from .views import BookViewSet

router = routers.DefaultRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

Now you can access the API with: `/api/books/?version=v1` or `/api/books/?version=v2`.

### 4. HTTP Header Versioning

This approach uses a custom HTTP header for versioning:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}
```

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
```

With this approach, clients specify the version in the `Accept` header:
```
Accept: application/json; version=v1
```

### 5. Hostname Versioning

This approach uses different hostnames for different API versions:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.HostNameVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}
```

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
```

With this approach, clients access different versions using different hostnames:
- `v1.example.com/api/books/`
- `v2.example.com/api/books/`

### 6. Namespace Versioning

This approach uses URL namespaces for versioning:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}
```

```python
# urls.py
from django.urls import path, include
from rest_framework import routers
from .views import BookViewSet

router_v1 = routers.DefaultRouter()
router_v1.register(r'books', BookViewSet)

router_v2 = routers.DefaultRouter()
router_v2.register(r'books', BookViewSet)

urlpatterns = [
    path('api/v1/', include((router_v1.urls, 'v1'), namespace='v1')),
    path('api/v2/', include((router_v2.urls, 'v2'), namespace='v2')),
]
```

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
```

### 7. Accessing Version in ViewSets

You can access the requested version in your ViewSet:

```python
# views.py
from rest_framework import viewsets
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
    
    def list(self, request, *args, **kwargs):
        # Add version to response for debugging
        response = super().list(request, *args, **kwargs)
        response.data['api_version'] = request.version
        return response
```

### 8. Different Behaviors for Different Versions

You can implement different behaviors based on the API version:

```python
# views.py
from rest_framework import viewsets
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializerV1, BookSerializerV2
from .pagination import StandardPagination, EnhancedPagination

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return BookSerializerV1
        return BookSerializerV2
    
    def get_pagination_class(self):
        if self.request.version == 'v1':
            return StandardPagination
        return EnhancedPagination
    
    def get_queryset(self):
        queryset = Book.objects.all()
        
        # V2 adds extra filtering
        if self.request.version == 'v2':
            author = self.request.query_params.get('author', None)
            if author:
                queryset = queryset.filter(author__name__icontains=author)
        
        return queryset
```

### 9. Version-Specific Serializers

Define different serializers for different API versions:

```python
# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializerV1(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date']

class BookSerializerV2(serializers.ModelSerializer):
    # V2 adds more fields and a computed field
    author_name = serializers.ReadOnlyField(source='author.name')
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'author_name', 'publication_date', 'isbn', 'price', 'average_rating']
```

### 10. Handling Breaking Changes

For backward-incompatible changes, you can use serializer adaptations:

```python
# models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.ForeignKey('Author', on_delete=models.CASCADE)
    publication_date = models.DateField()
    # Field renamed from 'price' to 'retail_price' in a database migration
    retail_price = models.DecimalField(max_digits=6, decimal_places=2)

# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializerV1(serializers.ModelSerializer):
    # Keep using 'price' for backward compatibility
    price = serializers.DecimalField(source='retail_price', max_digits=6, decimal_places=2)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'price']

class BookSerializerV2(serializers.ModelSerializer):
    # Use the new field name in V2
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'retail_price']
```

## Notes

1. **Choosing a Versioning Scheme**:
   - URL Path Versioning: Most explicit, easiest for clients to understand
   - Query Parameter Versioning: Simple, no URL structure changes needed
   - Header Versioning: Keeps URLs clean but less visible
   - Hostname Versioning: Good for completely separate API versions
   - Namespace Versioning: Combines benefits of path versioning with namespace isolation

2. **Version Negotiation**:
   - Set a `DEFAULT_VERSION` for clients that don't specify a version
   - Use `ALLOWED_VERSIONS` to restrict which versions are valid
   - Return appropriate error responses for invalid versions

3. **Testing Versioned APIs**:
   - Write tests for each API version
   - Ensure backward compatibility
   - Test version negotiation and fallbacks

4. **Versioning Best Practices**:
   - Don't create new versions for minor changes
   - Document changes between versions clearly
   - Maintain older versions for a reasonable deprecation period
   - Consider using feature flags for more granular control

5. **Deprecation Policy**:
   - Define a clear deprecation policy (e.g., 12 months after a new version)
   - Notify users through API responses (e.g., `Deprecation` and `Sunset` headers)
   - Monitor usage of deprecated versions

6. **Performance Considerations**:
   - Multiple versions can increase code complexity
   - Consider the performance impact of supporting multiple versions
   - Use caching effectively for different versions

7. **Alternative Approaches**:
   - Content negotiation using media types (`application/vnd.myapi.v1+json`)
   - Using the "Robustness Principle" to reduce the need for versioning
   - Evolution-based APIs that maintain backward compatibility
