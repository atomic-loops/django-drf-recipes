# DefaultRouter and SimpleRouter

## Problem

You need to create URL patterns for your API endpoints in a consistent and maintainable way, especially when working with ViewSets.

## Solution

Use Django REST Framework's routers to automatically generate URL patterns for your ViewSets, following REST conventions.

## Code

### 1. Basic Router Usage

Here's how to set up a basic router with a ViewSet:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

# Create a router and register our viewset with it
router = DefaultRouter()
router.register(r'books', BookViewSet)

# The API URLs are now determined automatically by the router
urlpatterns = [
    path('api/', include(router.urls)),
]
```

This automatically creates the following URL patterns:
- `GET/POST /api/books/` - List and create books
- `GET/PUT/PATCH/DELETE /api/books/{pk}/` - Retrieve, update, and delete a specific book
- `GET /api/` - API root which lists all available endpoints

### 2. DefaultRouter vs SimpleRouter

The two main router classes in DRF are `DefaultRouter` and `SimpleRouter`. Here's a comparison:

```python
# Using DefaultRouter
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'books', BookViewSet)
```

```python
# Using SimpleRouter
from rest_framework.routers import SimpleRouter

router = SimpleRouter()
router.register(r'books', BookViewSet)
```

Key differences:
- `DefaultRouter` includes an API root view (`/api/`) that lists all registered viewsets
- `DefaultRouter` includes a `.json` format suffix pattern by default
- `SimpleRouter` is more minimal and doesn't include these features

Choose `DefaultRouter` for more complete, browsable APIs, and `SimpleRouter` for simpler, more minimal URL patterns.

### 3. Registering Multiple ViewSets

You can register multiple ViewSets with a single router:

```python
# views.py
from rest_framework import viewsets
from .models import Book, Author, Publisher
from .serializers import BookSerializer, AuthorSerializer, PublisherSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer

class PublisherViewSet(viewsets.ModelViewSet):
    queryset = Publisher.objects.all()
    serializer_class = PublisherSerializer

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet, AuthorViewSet, PublisherViewSet

router = DefaultRouter()
router.register(r'books', BookViewSet)
router.register(r'authors', AuthorViewSet)
router.register(r'publishers', PublisherViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

This creates endpoints for all three resources.

### 4. Custom Route Names

You can specify custom URL names when registering ViewSets:

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

router = DefaultRouter()
router.register(r'books', BookViewSet, basename='library-book')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

The `basename` parameter changes the URL name patterns. For example, the list view URL name would be `'library-book-list'` instead of `'book-list'`.

### 5. Combining Routers with Regular URL Patterns

You can combine router-generated URLs with regular URL patterns:

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet, author_list, author_detail

router = DefaultRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/authors/', author_list, name='author-list'),
    path('api/authors/<int:pk>/', author_detail, name='author-detail'),
]
```

This combines ViewSet-generated URLs with function-based view URLs.

### 6. Nested Routers

For nested resources, you can use the `drf-nested-routers` package:

```bash
pip install drf-nested-routers
```

```python
# views.py
from rest_framework import viewsets
from .models import Author, Book
from .serializers import AuthorSerializer, BookSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookSerializer
    
    def get_queryset(self):
        return Book.objects.filter(author=self.kwargs['author_pk'])

# urls.py
from django.urls import path, include
from rest_framework_nested import routers
from .views import AuthorViewSet, BookViewSet

router = routers.DefaultRouter()
router.register(r'authors', AuthorViewSet)

# Create a nested router for books under authors
author_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
author_router.register(r'books', BookViewSet, basename='author-books')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(author_router.urls)),
]
```

This creates nested URLs like `/api/authors/{author_pk}/books/` and `/api/authors/{author_pk}/books/{pk}/`.

### 7. Custom Route Configuration

You can customize how routes are generated for specific ViewSets:

```python
# views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=False, methods=['get'])
    def bestsellers(self, request):
        books = self.get_queryset().filter(is_bestseller=True)
        serializer = self.get_serializer(books, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def mark_as_read(self, request, pk=None):
        book = self.get_object()
        book.mark_as_read_by(request.user)
        return Response({'status': 'book marked as read'})

# urls.py with custom route
from rest_framework.routers import DefaultRouter, Route, DynamicRoute

class CustomRouter(DefaultRouter):
    """
    Custom router to change URL patterns.
    """
    routes = [
        # List route
        Route(
            url=r'^{prefix}/$',
            mapping={'get': 'list', 'post': 'create'},
            name='{basename}-list',
            detail=False,
            initkwargs={'suffix': 'List'}
        ),
        # Detail route
        Route(
            url=r'^{prefix}/{lookup}/$',
            mapping={'get': 'retrieve', 'put': 'update', 'patch': 'partial_update', 'delete': 'destroy'},
            name='{basename}-detail',
            detail=True,
            initkwargs={'suffix': 'Detail'}
        ),
        # Custom routes
        DynamicRoute(
            url=r'^{prefix}/{lookup}/{url_path}/$',
            name='{basename}-{url_name}',
            detail=True,
            initkwargs={}
        ),
        DynamicRoute(
            url=r'^{prefix}/{url_path}/$',
            name='{basename}-{url_name}',
            detail=False,
            initkwargs={}
        ),
    ]

router = CustomRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

This customizes the URL patterns for all ViewSets registered with this router.

### 8. URL Namespace

You can add a namespace to your router URLs:

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

router = DefaultRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/v1/', include((router.urls, 'v1'), namespace='v1')),
]
```

This adds the namespace `'v1'` to all URL names. For example, the list view URL name would be `'v1:book-list'`.

### 9. API Versioning with Multiple Routers

You can use different routers for different API versions:

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views_v1 import BookViewSetV1
from .views_v2 import BookViewSetV2

# Router for API v1
router_v1 = DefaultRouter()
router_v1.register(r'books', BookViewSetV1)

# Router for API v2
router_v2 = DefaultRouter()
router_v2.register(r'books', BookViewSetV2)

urlpatterns = [
    path('api/v1/', include((router_v1.urls, 'v1'), namespace='v1')),
    path('api/v2/', include((router_v2.urls, 'v2'), namespace='v2')),
]
```

This separates URL patterns for different API versions.

### 10. Router for Read-Only Views

If you only need read operations, use `ReadOnlyModelViewSet` with a router:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A viewset that provides only 'list' and 'retrieve' actions.
    """
    queryset = Book.objects.all()
    serializer_class = BookSerializer

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

router = DefaultRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

This creates only `GET` endpoints: `/api/books/` for list and `/api/books/{pk}/` for detail.

## Notes

1. **Router Features**:
   - Automatic URL pattern generation for ViewSets
   - Consistent naming convention for URLs
   - Support for custom actions with `@action` decorator
   - API root view (with `DefaultRouter`)
   - Format suffix patterns

2. **When to Use Routers**:
   - When using ViewSets for standard CRUD operations
   - When you want consistent URL patterns across your API
   - When you want to reduce URL configuration boilerplate

3. **When Not to Use Routers**:
   - For highly custom URL patterns
   - When using function-based views or regular APIView classes
   - When you need more fine-grained control over URL patterns

4. **URL Naming Conventions**:
   - List view: `{basename}-list`
   - Detail view: `{basename}-detail`
   - Custom actions (non-detail): `{basename}-{url_name}`
   - Custom actions (detail): `{basename}-{url_name}`

5. **Advanced Router Usage**:
   - Custom router classes by subclassing `SimpleRouter` or `DefaultRouter`
   - Custom route generation by overriding the `routes` attribute
   - Dynamic URL patterns with `DynamicRoute`

6. **Combining with Other URL Patterns**:
   - Routers can be combined with regular URL patterns
   - Use `include()` to include router URLs in your main URL configuration
   - Consider using namespaces to avoid URL name conflicts

7. **Performance Considerations**:
   - Routers add minimal overhead
   - URL lookup is slightly more complex but not significantly slower
   - Benefits of consistency and maintainability outweigh any performance costs
