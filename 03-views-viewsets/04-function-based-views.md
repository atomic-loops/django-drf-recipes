# Function-Based Views

## Problem

You need to create API endpoints with custom logic that doesn't fit well into class-based views, or you prefer the simplicity and explicit nature of function-based views.

## Solution

Use Django REST Framework's function-based view decorators to create API endpoints using regular Python functions.

## Code

### 1. Basic Function-Based View

Here's a simple function-based view for listing and creating books:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'POST'])
def book_list(request):
    """
    List all books, or create a new book.
    """
    if request.method == 'GET':
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

And a corresponding view for retrieving, updating, or deleting a specific book:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'PUT', 'DELETE'])
def book_detail(request, pk):
    """
    Retrieve, update or delete a book.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

Configure your URLs:

```python
# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('books/', views.book_list, name='book-list'),
    path('books/<int:pk>/', views.book_detail, name='book-detail'),
]
```

### 2. Adding Permissions

You can add permission checks to your function-based views:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def book_list(request):
    """
    List all books, or create a new book.
    Only authenticated users can access this view.
    """
    if request.method == 'GET':
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            # Associate the book with the user
            serializer.save(owner=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### 3. Authentication and Authorization

You can specify authentication classes and implement custom permission checks:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, authentication_classes, permission_classes
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'PUT', 'DELETE'])
@authentication_classes([TokenAuthentication])
@permission_classes([IsAuthenticated])
def book_detail(request, pk):
    """
    Retrieve, update or delete a book.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    # Check if the user is the owner or staff
    if book.owner != request.user and not request.user.is_staff:
        return Response(
            {"detail": "You do not have permission to perform this action."},
            status=status.HTTP_403_FORBIDDEN
        )
    
    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 4. Content Negotiation

Function-based views support content negotiation for different formats:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, renderer_classes
from rest_framework.renderers import JSONRenderer, BrowsableAPIRenderer, TemplateHTMLRenderer
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET'])
@renderer_classes([JSONRenderer, BrowsableAPIRenderer, TemplateHTMLRenderer])
def book_detail(request, pk):
    """
    Retrieve a book with support for HTML rendering.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    serializer = BookSerializer(book)
    
    if request.accepted_renderer.format == 'html':
        return Response(
            {'book': serializer.data},
            template_name='book_detail.html'
        )
    
    return Response(serializer.data)
```

### 5. Custom Response Headers

You can customize the response headers in function-based views:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['GET'])
def book_download(request, pk):
    """
    Download a book's information as a file.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    serializer = BookSerializer(book)
    response = Response(serializer.data)
    
    # Add headers for file download
    filename = f"book_{book.id}_{book.title.replace(' ', '_')}.json"
    response['Content-Disposition'] = f'attachment; filename="{filename}"'
    
    return response
```

### 6. Schema Decoration

You can use the `@schema` decorator to customize the schema for your view:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, schema
from rest_framework.response import Response
from rest_framework.schemas import AutoSchema
from .models import Book
from .serializers import BookSerializer

class BookSchema(AutoSchema):
    def get_description(self, path, method):
        if method == 'GET':
            return "Retrieves a book by ID"
        elif method == 'PUT':
            return "Updates an existing book"
        elif method == 'DELETE':
            return "Deletes a book"
        return super().get_description(path, method)

@api_view(['GET', 'PUT', 'DELETE'])
@schema(BookSchema())
def book_detail(request, pk):
    """
    Retrieve, update or delete a book.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 7. Throttling

You can apply throttling to function-based views:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle
from .models import Book
from .serializers import BookSerializer

class BurstRateThrottle(UserRateThrottle):
    rate = '3/min'

@api_view(['POST'])
@throttle_classes([AnonRateThrottle, BurstRateThrottle])
def book_create(request):
    """
    Create a new book, with rate limiting.
    """
    serializer = BookSerializer(data=request.data)
    if serializer.is_valid():
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### 8. Pagination

You can implement pagination in function-based views:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.pagination import PageNumberPagination
from .models import Book
from .serializers import BookSerializer

@api_view(['GET'])
def book_list(request):
    """
    List all books with pagination.
    """
    books = Book.objects.all()
    
    # Create a paginator instance
    paginator = PageNumberPagination()
    paginator.page_size = 10
    
    # Paginate the queryset
    paginated_books = paginator.paginate_queryset(books, request)
    
    # Serialize the paginated data
    serializer = BookSerializer(paginated_books, many=True)
    
    # Return the paginated response
    return paginator.get_paginated_response(serializer.data)
```

### 9. API Root View

Create an API root view to provide links to other endpoints:

```python
# views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse

@api_view(['GET'])
def api_root(request, format=None):
    """
    API root view providing links to endpoints.
    """
    return Response({
        'books': reverse('book-list', request=request, format=format),
        'authors': reverse('author-list', request=request, format=format),
        'users': reverse('user-list', request=request, format=format),
    })
```

### 10. File Upload Handling

Handle file uploads in function-based views:

```python
# views.py
from rest_framework import status
from rest_framework.decorators import api_view, parser_classes
from rest_framework.parsers import MultiPartParser, FormParser
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

@api_view(['POST'])
@parser_classes([MultiPartParser, FormParser])
def book_upload_cover(request, pk):
    """
    Upload a cover image for a book.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if 'cover' not in request.data:
        return Response(
            {'error': 'No cover image provided'},
            status=status.HTTP_400_BAD_REQUEST
        )
    
    book.cover = request.data['cover']
    book.save()
    
    serializer = BookSerializer(book)
    return Response(serializer.data)
```

## Notes

1. **When to Use Function-Based Views**:
   - For simple endpoints with custom logic
   - When you prefer explicit request method handling
   - For endpoints that don't fit well into class-based views
   - When you want to avoid the class inheritance complexity

2. **Available Decorators**:
   - `@api_view`: Specifies allowed HTTP methods (required)
   - `@permission_classes`: Specifies permission classes
   - `@authentication_classes`: Specifies authentication classes
   - `@throttle_classes`: Specifies throttling classes
   - `@parser_classes`: Specifies parser classes
   - `@renderer_classes`: Specifies renderer classes
   - `@schema`: Customizes the API schema

3. **Error Handling**:
   - Always handle common exceptions like `DoesNotExist`
   - Return appropriate status codes
   - Include descriptive error messages

4. **Content Negotiation**:
   - Function-based views support content negotiation just like class-based views
   - You can render different formats based on the request

5. **Best Practices**:
   - Keep views small and focused
   - Extract common logic into helper functions
   - Use meaningful function names
   - Document your views with docstrings

6. **Performance Considerations**:
   - Function-based views are slightly faster than class-based views
   - But the performance difference is negligible in most cases

7. **Migration Path**:
   - If a function-based view becomes too complex, consider refactoring to a class-based view
   - Start with function-based views for simpler endpoints
