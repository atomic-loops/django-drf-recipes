# ModelViewSet CRUD Operations

## Problem

You need to implement a complete set of CRUD (Create, Read, Update, Delete) operations for a resource with minimal code while still maintaining flexibility.

## Solution

Use Django REST Framework's `ModelViewSet` to quickly create a full-featured API endpoint with all standard CRUD operations.

## Code

### 1. Basic ModelViewSet

Here's a simple `ModelViewSet` implementation for a Book model:

```python
# models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.CharField(max_length=255)
    publication_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    def __str__(self):
        return self.title

# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'isbn', 'price']

# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing book instances.
    """
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

With this minimal code, you get these endpoints:
- `GET /books/` - List all books
- `POST /books/` - Create a new book
- `GET /books/{id}/` - Retrieve a specific book
- `PUT /books/{id}/` - Update a book (complete update)
- `PATCH /books/{id}/` - Partial update of a book
- `DELETE /books/{id}/` - Delete a book

### 2. ModelViewSet with Filtering and Searching

Enhance your viewset with filtering and searching capabilities:

```python
# views.py
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    # Enable filtering and searching
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['author', 'publication_date']
    search_fields = ['title', 'author', 'isbn']
    ordering_fields = ['title', 'author', 'publication_date', 'price']
    ordering = ['title']  # Default ordering
```

With these additions, you can now:
- Filter books: `GET /books/?author=Tolkien`
- Search books: `GET /books/?search=hobbit`
- Order results: `GET /books/?ordering=-publication_date`

Note: To use `DjangoFilterBackend`, you need to install and configure `django-filter`:

```bash
pip install django-filter
```

And add it to your settings:

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
    # ...
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
    # ...
}
```

### 3. ModelViewSet with Permissions

Add permission control to your viewset:

```python
# views.py
from rest_framework import viewsets, permissions
from .models import Book
from .serializers import BookSerializer
from .permissions import IsOwnerOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    # Different permissions for different actions
    def get_permissions(self):
        if self.action in ['list', 'retrieve']:
            permission_classes = [permissions.AllowAny]
        elif self.action == 'create':
            permission_classes = [permissions.IsAuthenticated]
        else:  # update, partial_update, destroy
            permission_classes = [permissions.IsAuthenticated, IsOwnerOrReadOnly]
        
        return [permission() for permission in permission_classes]
    
    # Associate the book with the user when creating
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

With the corresponding permission class:

```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions are only allowed to the owner
        return obj.owner == request.user
```

### 4. ModelViewSet with Pagination

Add pagination to handle large datasets:

```python
# views.py
from rest_framework import viewsets
from rest_framework.pagination import PageNumberPagination
from .models import Book
from .serializers import BookSerializer

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 100

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    pagination_class = StandardResultsSetPagination
```

With this configuration, your list endpoint will return paginated results:

```json
{
    "count": 42,
    "next": "http://example.com/api/books/?page=2",
    "previous": null,
    "results": [
        { "id": 1, "title": "Book 1", ... },
        { "id": 2, "title": "Book 2", ... },
        ...
    ]
}
```

You can also configure pagination globally:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

### 5. ModelViewSet with Multiple Serializers

Use different serializers for different actions:

```python
# serializers.py
from rest_framework import serializers
from .models import Book

class BookListSerializer(serializers.ModelSerializer):
    """Serializer for listing books (fewer fields)"""
    class Meta:
        model = Book
        fields = ['id', 'title', 'author']

class BookDetailSerializer(serializers.ModelSerializer):
    """Serializer for detailed book view (all fields)"""
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'isbn', 'price', 'created_at', 'updated_at']

# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookListSerializer, BookDetailSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        if self.action == 'list':
            return BookListSerializer
        return BookDetailSerializer
```

This optimizes your API by using a lighter serializer for list views while providing full details for individual book views.

### 6. ModelViewSet with Custom Queryset

Customize the queryset based on the current user or action:

```python
# views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookSerializer
    
    def get_queryset(self):
        user = self.request.user
        
        # Base queryset
        queryset = Book.objects.all()
        
        # Filter by user if not staff
        if not user.is_staff:
            queryset = queryset.filter(public=True)
        
        # Apply additional filters from query parameters
        author = self.request.query_params.get('author', None)
        if author:
            queryset = queryset.filter(author=author)
        
        # Optimize with select_related for detail views
        if self.action in ['retrieve', 'update', 'partial_update']:
            queryset = queryset.select_related('publisher')
        
        return queryset
```

### 7. ModelViewSet with Custom Methods

Customize specific CRUD methods:

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def create(self, request, *args, **kwargs):
        """Custom create method with additional logging"""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Add additional data
        self.perform_create(serializer)
        
        # Log the creation
        print(f"Book created: {serializer.instance.title} by user {request.user}")
        
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)
    
    def update(self, request, *args, **kwargs):
        """Custom update method with partial data handling"""
        instance = self.get_object()
        
        # Get the current data and update with request data
        current_data = self.get_serializer(instance).data
        for key, value in request.data.items():
            current_data[key] = value
        
        serializer = self.get_serializer(instance, data=current_data)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)
        
        return Response(serializer.data)
    
    def destroy(self, request, *args, **kwargs):
        """Custom destroy method with soft delete"""
        instance = self.get_object()
        
        # Instead of deleting, mark as inactive
        instance.is_active = False
        instance.save()
        
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 8. ModelViewSet with URL Configuration

Configure your URLs to use the ModelViewSet:

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet

# Create a router and register our viewsets with it
router = DefaultRouter()
router.register(r'books', BookViewSet)

# The API URLs are automatically determined by the router
urlpatterns = [
    path('api/', include(router.urls)),
]
```

This will create all the necessary URL patterns for your CRUD operations.

## Notes

1. **ModelViewSet Inheritance**:
   - `ModelViewSet` inherits from `GenericViewSet` and includes mixins for all CRUD operations
   - The class hierarchy is: `ModelViewSet` → `mixins` → `GenericViewSet` → `ViewSetMixin` → `APIView`

2. **Available Actions**:
   - `list`: Get all objects
   - `create`: Create a new object
   - `retrieve`: Get a single object
   - `update`: Update an object (full update)
   - `partial_update`: Update an object (partial update)
   - `destroy`: Delete an object

3. **Performance Optimization**:
   - Use `select_related()` and `prefetch_related()` in your queryset
   - Consider using different serializers for list and detail views
   - Implement pagination for large datasets

4. **Security Considerations**:
   - Always define appropriate permissions
   - Consider using object-level permissions for fine-grained control
   - Validate input data properly

5. **Customization Points**:
   - `get_queryset()`: Customize the queryset (filtering, ordering)
   - `get_serializer_class()`: Use different serializers based on action
   - `get_permissions()`: Apply different permissions based on action
   - `perform_create()`, `perform_update()`, `perform_destroy()`: Customize object creation/modification

6. **When to Use ModelViewSet**:
   - When you need all standard CRUD operations
   - When you want to minimize boilerplate code
   - When your API follows standard REST conventions

7. **When Not to Use ModelViewSet**:
   - When you need very custom behavior for most actions
   - When you only need a subset of CRUD operations (consider other viewsets)
   - When your API doesn't follow standard REST conventions
