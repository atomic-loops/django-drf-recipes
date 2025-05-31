# Nested Routing with drf-nested-routers

**Problem:** You need to create nested URL patterns for related resources (e.g., `/authors/1/books/` or `/books/1/reviews/`) while maintaining clean, RESTful API design.

**Solution:** Use the `drf-nested-routers` package to automatically generate nested routes for related resources.

## Installation

```bash
pip install drf-nested-routers
```

## Basic Nested Router Setup

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, related_name='books', on_delete=models.CASCADE)
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()
    
    def __str__(self):
        return self.title

class Review(models.Model):
    book = models.ForeignKey(Book, related_name='reviews', on_delete=models.CASCADE)
    reviewer_name = models.CharField(max_length=100)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"Review by {self.reviewer_name}"
```

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book, Review

class AuthorSerializer(serializers.ModelSerializer):
    books_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'email', 'bio', 'books_count']
    
    def get_books_count(self, obj):
        return obj.books.count()

class BookSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.name', read_only=True)
    reviews_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'author_name', 'isbn', 'published_date', 'reviews_count']
    
    def get_reviews_count(self, obj):
        return obj.reviews.count()

class ReviewSerializer(serializers.ModelSerializer):
    book_title = serializers.CharField(source='book.title', read_only=True)
    
    class Meta:
        model = Review
        fields = ['id', 'book', 'book_title', 'reviewer_name', 'rating', 'comment', 'created_at']
        read_only_fields = ['created_at']
```

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action
from django.shortcuts import get_object_or_404
from .models import Author, Book, Review
from .serializers import AuthorSerializer, BookSerializer, ReviewSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.prefetch_related('books')
    serializer_class = AuthorSerializer
    
    @action(detail=True, methods=['get'])
    def stats(self, request, pk=None):
        author = self.get_object()
        books = author.books.all()
        total_reviews = sum(book.reviews.count() for book in books)
        
        return Response({
            'books_count': books.count(),
            'total_reviews': total_reviews,
            'average_rating': sum(
                sum(review.rating for review in book.reviews.all()) / max(book.reviews.count(), 1)
                for book in books
            ) / max(books.count(), 1) if books.exists() else 0
        })

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookSerializer
    
    def get_queryset(self):
        # Check if this is a nested route
        if 'author_pk' in self.kwargs:
            return Book.objects.filter(
                author_id=self.kwargs['author_pk']
            ).select_related('author')
        return Book.objects.select_related('author')
    
    def perform_create(self, serializer):
        # If this is a nested route, set the author automatically
        if 'author_pk' in self.kwargs:
            author = get_object_or_404(Author, pk=self.kwargs['author_pk'])
            serializer.save(author=author)
        else:
            serializer.save()

class ReviewViewSet(viewsets.ModelViewSet):
    serializer_class = ReviewSerializer
    
    def get_queryset(self):
        # Check if this is a nested route under books
        if 'book_pk' in self.kwargs:
            return Review.objects.filter(
                book_id=self.kwargs['book_pk']
            ).select_related('book')
        # Check if this is a nested route under authors
        elif 'author_pk' in self.kwargs:
            return Review.objects.filter(
                book__author_id=self.kwargs['author_pk']
            ).select_related('book', 'book__author')
        return Review.objects.select_related('book')
    
    def perform_create(self, serializer):
        # If this is a nested route under books, set the book automatically
        if 'book_pk' in self.kwargs:
            book = get_object_or_404(Book, pk=self.kwargs['book_pk'])
            serializer.save(book=book)
        else:
            serializer.save()
```

## URL Configuration with Nested Routers

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_nested import routers
from .views import AuthorViewSet, BookViewSet, ReviewViewSet

# Main router
router = DefaultRouter()
router.register(r'authors', AuthorViewSet)
router.register(r'books', BookViewSet, basename='book')
router.register(r'reviews', ReviewViewSet, basename='review')

# Nested routers
authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
authors_router.register(r'books', BookViewSet, basename='author-books')

books_router = routers.NestedDefaultRouter(router, r'books', lookup='book')
books_router.register(r'reviews', ReviewViewSet, basename='book-reviews')

# Also allow reviews under authors (authors/1/reviews/ - all reviews for author's books)
authors_reviews_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
authors_reviews_router.register(r'reviews', ReviewViewSet, basename='author-reviews')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(authors_router.urls)),
    path('api/', include(books_router.urls)),
    path('api/', include(authors_reviews_router.urls)),
]
```

## Advanced Nested Router with Custom Filtering

```python
# filters.py
import django_filters
from .models import Book, Review

class BookFilter(django_filters.FilterSet):
    title = django_filters.CharFilter(lookup_expr='icontains')
    published_year = django_filters.NumberFilter(field_name='published_date__year')
    published_after = django_filters.DateFilter(field_name='published_date', lookup_expr='gte')
    
    class Meta:
        model = Book
        fields = ['title', 'published_year', 'published_after']

class ReviewFilter(django_filters.FilterSet):
    rating_min = django_filters.NumberFilter(field_name='rating', lookup_expr='gte')
    rating_max = django_filters.NumberFilter(field_name='rating', lookup_expr='lte')
    reviewer = django_filters.CharFilter(field_name='reviewer_name', lookup_expr='icontains')
    
    class Meta:
        model = Review
        fields = ['rating_min', 'rating_max', 'reviewer']
```

```python
# views.py (updated with filtering)
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.filters import SearchFilter, OrderingFilter

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = BookFilter
    search_fields = ['title', 'author__name']
    ordering_fields = ['title', 'published_date']
    ordering = ['-published_date']
    
    def get_queryset(self):
        if 'author_pk' in self.kwargs:
            return Book.objects.filter(
                author_id=self.kwargs['author_pk']
            ).select_related('author')
        return Book.objects.select_related('author')

class ReviewViewSet(viewsets.ModelViewSet):
    serializer_class = ReviewSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = ReviewFilter
    search_fields = ['reviewer_name', 'comment']
    ordering_fields = ['rating', 'created_at']
    ordering = ['-created_at']
    
    def get_queryset(self):
        if 'book_pk' in self.kwargs:
            return Review.objects.filter(
                book_id=self.kwargs['book_pk']
            ).select_related('book')
        elif 'author_pk' in self.kwargs:
            return Review.objects.filter(
                book__author_id=self.kwargs['author_pk']
            ).select_related('book', 'book__author')
        return Review.objects.select_related('book')
```

## Custom Nested Actions

```python
# views.py (additional custom actions)
from rest_framework.decorators import action
from django.db.models import Avg, Count

class AuthorViewSet(viewsets.ModelViewSet):
    # ... existing code ...
    
    @action(detail=True, methods=['get'], url_path='books/top-rated')
    def top_rated_books(self, request, pk=None):
        """Get author's top-rated books"""
        author = self.get_object()
        books = author.books.annotate(
            avg_rating=Avg('reviews__rating'),
            review_count=Count('reviews')
        ).filter(
            review_count__gte=1  # Only books with reviews
        ).order_by('-avg_rating')[:5]
        
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)

class BookViewSet(viewsets.ModelViewSet):
    # ... existing code ...
    
    @action(detail=True, methods=['get'], url_path='reviews/summary')
    def review_summary(self, request, pk=None, author_pk=None):
        """Get review summary for a book"""
        book = self.get_object()
        reviews = book.reviews.all()
        
        if not reviews.exists():
            return Response({
                'message': 'No reviews yet',
                'total_reviews': 0,
                'average_rating': None
            })
        
        rating_distribution = {}
        for i in range(1, 6):
            rating_distribution[f'{i}_star'] = reviews.filter(rating=i).count()
        
        return Response({
            'total_reviews': reviews.count(),
            'average_rating': reviews.aggregate(Avg('rating'))['rating__avg'],
            'rating_distribution': rating_distribution,
            'latest_review': ReviewSerializer(reviews.first()).data
        })
```

## Generated URL Patterns

The nested routers will generate the following URL patterns:

```
# Standard routes
GET    /api/authors/                    # List authors
POST   /api/authors/                    # Create author
GET    /api/authors/{id}/               # Retrieve author
PUT    /api/authors/{id}/               # Update author
DELETE /api/authors/{id}/               # Delete author

# Nested routes for books under authors
GET    /api/authors/{id}/books/         # List author's books
POST   /api/authors/{id}/books/         # Create book for author
GET    /api/authors/{id}/books/{id}/    # Retrieve specific book
PUT    /api/authors/{id}/books/{id}/    # Update book
DELETE /api/authors/{id}/books/{id}/    # Delete book

# Nested routes for reviews under books
GET    /api/books/{id}/reviews/         # List book's reviews
POST   /api/books/{id}/reviews/         # Create review for book
GET    /api/books/{id}/reviews/{id}/    # Retrieve specific review
PUT    /api/books/{id}/reviews/{id}/    # Update review
DELETE /api/books/{id}/reviews/{id}/    # Delete review

# Custom nested actions
GET    /api/authors/{id}/stats/         # Author statistics
GET    /api/authors/{id}/books/top-rated/ # Top-rated books
GET    /api/books/{id}/reviews/summary/ # Review summary
```

## Usage Examples

```bash
# Get all books by author 1
GET /api/authors/1/books/

# Create a new book for author 1
POST /api/authors/1/books/
{
    "title": "New Book",
    "isbn": "1234567890123",
    "published_date": "2024-01-01"
}

# Get all reviews for book 5
GET /api/books/5/reviews/

# Get reviews with filtering
GET /api/books/5/reviews/?rating_min=4&ordering=-created_at

# Get author's book statistics
GET /api/authors/1/stats/

# Get review summary for a book
GET /api/books/5/reviews/summary/
```

**Notes:**
- Nested routers automatically handle parent-child relationships in URLs
- Always validate that parent objects exist in nested views
- Use `select_related` and `prefetch_related` for efficient queries
- Consider permission implications for nested resources
- Custom actions can be added at any nesting level
- Be mindful of URL complexity - avoid too many nesting levels
