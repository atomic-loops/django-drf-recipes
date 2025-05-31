# Custom Actions with ViewSets

## Problem

You need to add custom endpoints to your ViewSet beyond the standard CRUD operations (list, create, retrieve, update, delete).

## Solution

Use the `@action` decorator provided by Django REST Framework to add custom actions to your ViewSets.

## Code

### 1. Basic Custom Action

Here's a ViewSet with a custom action that marks a book as featured:

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=True, methods=['post'])
    def mark_featured(self, request, pk=None):
        book = self.get_object()
        book.is_featured = True
        book.save()
        
        serializer = self.get_serializer(book)
        return Response(serializer.data)
```

This creates an endpoint at `/books/{pk}/mark_featured/` that accepts POST requests.

### 2. Custom Action on Collection (List)

You can also add actions that apply to the entire collection:

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Avg
from .models import Book
from .serializers import BookSerializer, BookStatsSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=False, methods=['get'])
    def stats(self, request):
        books_count = Book.objects.count()
        avg_rating = Book.objects.aggregate(average_rating=Avg('rating'))
        
        data = {
            'count': books_count,
            'average_rating': avg_rating['average_rating'] or 0
        }
        
        return Response(data)
```

This creates an endpoint at `/books/stats/` that returns statistics about all books.

### 3. Custom Action with Different Serializer

You can use a different serializer for your custom action:

```python
# serializers.py
from rest_framework import serializers
from .models import Book, Review

class ReviewSerializer(serializers.ModelSerializer):
    class Meta:
        model = Review
        fields = ['id', 'user', 'rating', 'comment', 'created_at']

# views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book, Review
from .serializers import BookSerializer, ReviewSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=True, methods=['get'])
    def reviews(self, request, pk=None):
        book = self.get_object()
        reviews = book.reviews.all()
        
        serializer = ReviewSerializer(reviews, many=True)
        return Response(serializer.data)
```

This creates an endpoint at `/books/{pk}/reviews/` that returns all reviews for a specific book.

### 4. Custom Action with Custom Permissions

You can apply custom permissions to specific actions:

```python
# views.py
from rest_framework import viewsets, permissions
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer
from .permissions import IsAdminOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [permissions.IsAuthenticated]
    
    @action(
        detail=True, 
        methods=['post'], 
        permission_classes=[permissions.IsAdminUser]
    )
    def publish(self, request, pk=None):
        book = self.get_object()
        book.is_published = True
        book.published_date = timezone.now()
        book.save()
        
        serializer = self.get_serializer(book)
        return Response(serializer.data)
```

In this example, only admin users can access the `/books/{pk}/publish/` endpoint.

### 5. Custom Action with URL Parameters

You can access URL parameters in your action:

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
    def by_author(self, request):
        author_name = request.query_params.get('name', None)
        
        if author_name:
            books = Book.objects.filter(author__name__icontains=author_name)
            serializer = self.get_serializer(books, many=True)
            return Response(serializer.data)
        
        return Response(
            {"error": "Author name parameter is required"}, 
            status=status.HTTP_400_BAD_REQUEST
        )
```

This creates an endpoint at `/books/by_author/?name=Rowling` that filters books by author name.

### 6. Adding Actions to Nested ViewSets

You can add actions to nested ViewSets:

```python
# views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Author, Book
from .serializers import AuthorSerializer, BookSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer
    
    @action(detail=True, methods=['get'])
    def bestsellers(self, request, pk=None):
        author = self.get_object()
        books = author.books.filter(sales__gt=10000).order_by('-sales')
        
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
```

This creates an endpoint at `/authors/{pk}/bestsellers/` that returns the bestselling books by a specific author.

### 7. Custom Action with File Upload

You can handle file uploads in custom actions:

```python
# views.py
from rest_framework import viewsets, parsers
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(
        detail=True, 
        methods=['post'],
        parser_classes=[parsers.MultiPartParser]
    )
    def cover_image(self, request, pk=None):
        book = self.get_object()
        
        if 'file' not in request.data:
            return Response(
                {"error": "No file provided"}, 
                status=status.HTTP_400_BAD_REQUEST
            )
            
        file = request.data['file']
        book.cover_image = file
        book.save()
        
        serializer = self.get_serializer(book)
        return Response(serializer.data)
```

This creates an endpoint at `/books/{pk}/cover_image/` that accepts file uploads for book covers.

### 8. Custom Action with Data Validation

You can implement custom validation for action data:

```python
# serializers.py
from rest_framework import serializers

class RateBookSerializer(serializers.Serializer):
    rating = serializers.IntegerField(min_value=1, max_value=5)
    comment = serializers.CharField(required=False, allow_blank=True)

# views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book, Review
from .serializers import BookSerializer, RateBookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=True, methods=['post'])
    def rate(self, request, pk=None):
        book = self.get_object()
        
        # Validate input data with a dedicated serializer
        serializer = RateBookSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Create the review
        review = Review.objects.create(
            book=book,
            user=request.user,
            rating=serializer.validated_data['rating'],
            comment=serializer.validated_data.get('comment', '')
        )
        
        # Return the created review
        return Response({
            'success': True,
            'rating': review.rating,
            'comment': review.comment
        })
```

This creates an endpoint at `/books/{pk}/rate/` that validates the rating input before creating a review.

### 9. Actions that Modify Multiple Objects

You can create actions that operate on multiple objects:

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
    
    @action(detail=False, methods=['post'])
    def bulk_discount(self, request):
        # Get discount percentage from request
        discount_percent = request.data.get('discount', None)
        book_ids = request.data.get('book_ids', [])
        
        if not discount_percent:
            return Response(
                {"error": "Discount percentage is required"}, 
                status=status.HTTP_400_BAD_REQUEST
            )
        
        try:
            discount_percent = float(discount_percent)
            if not 0 <= discount_percent <= 100:
                raise ValueError("Discount must be between 0 and 100")
        except ValueError as e:
            return Response(
                {"error": str(e)}, 
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Apply discount to all selected books
        books = Book.objects.filter(id__in=book_ids)
        count = books.count()
        
        for book in books:
            discount_amount = book.price * (discount_percent / 100)
            book.discounted_price = book.price - discount_amount
            book.save()
        
        return Response({
            'success': True,
            'message': f"Discount of {discount_percent}% applied to {count} books"
        })
```

This creates an endpoint at `/books/bulk_discount/` that applies a discount to multiple books.

### 10. Combining Multiple HTTP Methods in One Action

You can handle multiple HTTP methods in a single action:

```python
# views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer, BookNoteSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=True, methods=['get', 'post', 'delete'])
    def notes(self, request, pk=None):
        book = self.get_object()
        
        if request.method == 'GET':
            # Return the book notes
            return Response({
                'notes': book.notes
            })
        
        elif request.method == 'POST':
            # Add a note to the book
            serializer = BookNoteSerializer(data=request.data)
            serializer.is_valid(raise_exception=True)
            
            note = serializer.validated_data['note']
            
            # Assuming notes is a JSONField or similar
            if not book.notes:
                book.notes = []
            
            book.notes.append({
                'text': note,
                'created_by': request.user.username,
                'created_at': timezone.now().isoformat()
            })
            
            book.save()
            
            return Response({
                'success': True,
                'notes': book.notes
            })
        
        elif request.method == 'DELETE':
            # Clear all notes
            book.notes = []
            book.save()
            
            return Response({
                'success': True,
                'message': 'All notes cleared'
            })
```

This creates an endpoint at `/books/{pk}/notes/` that supports multiple operations based on the HTTP method.

## Notes

1. **URL Patterns**:
   - Detail actions (`detail=True`) are accessible at `/resourcename/{pk}/actionname/`
   - List actions (`detail=False`) are accessible at `/resourcename/actionname/`
   - The URL name will be `{resource}-{action}`, e.g., `book-reviews`

2. **HTTP Methods**:
   - By default, actions only accept GET requests
   - Specify other methods with the `methods` parameter
   - You can allow multiple methods: `methods=['get', 'post']`

3. **Router Integration**:
   - Custom actions are automatically registered with DRF's `DefaultRouter`
   - If using SimpleRouter, you need to set `trailing_slash=True` for consistent URLs

4. **Serializers**:
   - Use `self.get_serializer()` for consistency with your ViewSet's main serializer
   - Or specify a different serializer for the action
   - You can create dedicated serializers for validating action inputs

5. **Permissions**:
   - Action-specific permissions override the ViewSet's default permissions
   - Useful for actions that require higher access levels

6. **Response Formats**:
   - Custom actions follow the same content negotiation as regular endpoints
   - You can return different response formats using renderer classes

7. **Common Use Cases**:
   - State transitions (publish, approve, reject)
   - Aggregated data (stats, metrics)
   - Related object access (reviews, comments)
   - Bulk operations (batch updates, imports)
   - Custom reports or exports

8. **Testing Custom Actions**:
   - Use the `reverse()` function to get action URLs:
     ```python
     from rest_framework.reverse import reverse
     url = reverse('book-mark-featured', args=[book_id])
     ```
   - Test each custom action specifically in your test cases
