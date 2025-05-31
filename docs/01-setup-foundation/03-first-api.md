# Creating Your First API

## Problem

You need to create a basic REST API that performs CRUD operations on a model.

## Solution

Build a complete API for a Book model with Django REST Framework using serializers and class-based views.

## Code

### 1. Define Your Model

First, create a model in one of your apps:

```python
# apps/books/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    publication_date = models.DateField(blank=True, null=True)
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=6, decimal_places=2, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title
```

Don't forget to run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

### 2. Create a Serializer

Create a serializer to convert your model to/from JSON:

```python
# apps/books/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'description', 'publication_date', 
                 'isbn', 'price', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

### 3. Create Views

There are multiple ways to create DRF views. Here, we'll show three approaches, from most manual to most automatic:

#### Option 1: Using APIView (more control, more code)

```python
# apps/books/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.http import Http404
from .models import Book
from .serializers import BookSerializer

class BookList(APIView):
    """
    List all books or create a new book
    """
    def get(self, request, format=None):
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    def post(self, request, format=None):
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class BookDetail(APIView):
    """
    Retrieve, update or delete a book instance
    """
    def get_object(self, pk):
        try:
            return Book.objects.get(pk=pk)
        except Book.DoesNotExist:
            raise Http404
    
    def get(self, request, pk, format=None):
        book = self.get_object(pk)
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    def put(self, request, pk, format=None):
        book = self.get_object(pk)
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk, format=None):
        book = self.get_object(pk)
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

#### Option 2: Using Generic Views (balanced approach)

```python
# apps/books/views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer

class BookList(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

#### Option 3: Using ViewSets (most concise)

```python
# apps/books/views.py
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

### 4. Configure URLs

Configure your URLs based on the view approach you've chosen:

#### For APIView or Generic Views:

```python
# apps/books/urls.py
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from . import views

urlpatterns = [
    path('books/', views.BookList.as_view(), name='book-list'),
    path('books/<int:pk>/', views.BookDetail.as_view(), name='book-detail'),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

#### For ViewSets:

```python
# apps/books/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'books', views.BookViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

Remember to include these URLs in your main URLs file:

```python
# api/v1/urls.py
from django.urls import path, include

urlpatterns = [
    path('', include('apps.books.urls')),
    # Other app URLs...
]
```

### 5. Test Your API

With your server running, you can test your API:

- List all books: GET http://127.0.0.1:8000/api/v1/books/
- Get a specific book: GET http://127.0.0.1:8000/api/v1/books/1/
- Create a new book: POST http://127.0.0.1:8000/api/v1/books/
- Update a book: PUT http://127.0.0.1:8000/api/v1/books/1/
- Delete a book: DELETE http://127.0.0.1:8000/api/v1/books/1/

## Notes

1. **View Options**:
   - **APIView**: Offers the most control but requires more code
   - **Generic Views**: Provides common patterns with less code
   - **ViewSets**: Most concise, good for standard CRUD operations
   - Choose based on the complexity of your API requirements

2. **URL Format**:
   - The router for ViewSets automatically creates URL patterns for all CRUD operations
   - `format_suffix_patterns` adds support for format suffixes like `.json` or `.api`

3. **Permissions**:
   - This example doesn't include permissions, but in real apps, you should define them
   - Add permission_classes to your views based on your requirements

4. **Validation**:
   - The serializer automatically validates incoming data based on the model fields
   - You can add custom validation by overriding `validate_<field_name>` or `validate` methods

5. **Next Steps**:
   - Add filtering, searching, and ordering functionality
   - Implement pagination
   - Add authentication
   - Set up permissions and throttling
