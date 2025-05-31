# Optimizing Related Fields

**Problem:** Your API endpoints with related fields are causing N+1 query problems, slow response times, and unnecessary database hits when serializing related objects.

**Solution:** Use Django's `select_related`, `prefetch_related`, and DRF's optimization techniques to minimize database queries and improve performance.

## Understanding the N+1 Query Problem

```python
# models.py
from django.db import models

class Category(models.Model):
    name = models.CharField(max_length=50)
    description = models.TextField(blank=True)

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
    bio = models.TextField(blank=True)

class Publisher(models.Model):
    name = models.CharField(max_length=100)
    website = models.URLField(blank=True)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE, related_name='books')
    categories = models.ManyToManyField(Category, related_name='books')
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()
    price = models.DecimalField(max_digits=10, decimal_places=2)

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    reviewer_name = models.CharField(max_length=100)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

## Basic Optimization with select_related

```python
# serializers.py - PROBLEMATIC VERSION
from rest_framework import serializers
from .models import Book, Author, Publisher, Category, Review

# This will cause N+1 queries!
class BookSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.name', read_only=True)
    publisher_name = serializers.CharField(source='publisher.name', read_only=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author_name', 'publisher_name', 'isbn', 'published_date', 'price']

# views.py - PROBLEMATIC VERSION
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()  # This will cause N+1 queries
    serializer_class = BookSerializer
```

```python
# views.py - OPTIMIZED VERSION
class BookViewSet(viewsets.ModelViewSet):
    # Use select_related for ForeignKey fields
    queryset = Book.objects.select_related('author', 'publisher')
    serializer_class = BookSerializer
    
    def get_queryset(self):
        # You can also apply optimizations conditionally
        queryset = Book.objects.select_related('author', 'publisher')
        
        # Add prefetch_related for ManyToMany if needed
        if self.action == 'list':
            queryset = queryset.prefetch_related('categories')
        
        return queryset
```

## Advanced Optimization with prefetch_related

```python
# serializers.py - Advanced version with related fields
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class ReviewSerializer(serializers.ModelSerializer):
    class Meta:
        model = Review
        fields = ['id', 'reviewer_name', 'rating', 'comment', 'created_at']

class BookDetailSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.name', read_only=True)
    publisher_name = serializers.CharField(source='publisher.name', read_only=True)
    categories = CategorySerializer(many=True, read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)
    reviews_count = serializers.SerializerMethodField()
    average_rating = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = [
            'id', 'title', 'author_name', 'publisher_name', 'categories',
            'reviews', 'reviews_count', 'average_rating', 'isbn', 
            'published_date', 'price'
        ]
    
    def get_reviews_count(self, obj):
        # This will use prefetched data if available
        return obj.reviews.count()
    
    def get_average_rating(self, obj):
        reviews = obj.reviews.all()
        if reviews:
            return sum(review.rating for review in reviews) / len(reviews)
        return None

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookDetailSerializer
    
    def get_queryset(self):
        queryset = Book.objects.select_related('author', 'publisher')
        
        if self.action in ['retrieve', 'list']:
            # Prefetch related many-to-many and reverse foreign keys
            queryset = queryset.prefetch_related(
                'categories',
                'reviews'
            )
        
        return queryset
```

## Using Prefetch Objects for Complex Queries

```python
from django.db.models import Prefetch, Avg, Count

class BookViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Base queryset with select_related
        queryset = Book.objects.select_related('author', 'publisher')
        
        if self.action == 'list':
            # Prefetch only recent reviews (last 30 days)
            recent_reviews = Review.objects.filter(
                created_at__gte=timezone.now() - timedelta(days=30)
            ).order_by('-created_at')
            
            queryset = queryset.prefetch_related(
                'categories',
                Prefetch('reviews', queryset=recent_reviews, to_attr='recent_reviews')
            )
        
        elif self.action == 'retrieve':
            # For detail view, prefetch all reviews ordered by date
            all_reviews = Review.objects.order_by('-created_at')
            
            queryset = queryset.prefetch_related(
                'categories',
                Prefetch('reviews', queryset=all_reviews)
            )
        
        return queryset

# Update serializer to use prefetched data
class BookDetailSerializer(serializers.ModelSerializer):
    # ... other fields ...
    recent_reviews = ReviewSerializer(many=True, read_only=True)
    
    def get_reviews_count(self, obj):
        # Use prefetched recent_reviews if available
        if hasattr(obj, 'recent_reviews'):
            return len(obj.recent_reviews)
        return obj.reviews.count()
```

## Optimizing with Annotations

```python
from django.db.models import Avg, Count, F, Q

class BookListSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.name', read_only=True)
    publisher_name = serializers.CharField(source='publisher.name', read_only=True)
    reviews_count = serializers.IntegerField(read_only=True)
    average_rating = serializers.FloatField(read_only=True)
    categories_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Book
        fields = [
            'id', 'title', 'author_name', 'publisher_name', 
            'reviews_count', 'average_rating', 'categories_count',
            'isbn', 'published_date', 'price'
        ]

class BookViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Book.objects.select_related('author', 'publisher')
        
        if self.action == 'list':
            # Annotate with computed fields to avoid additional queries
            queryset = queryset.annotate(
                reviews_count=Count('reviews'),
                average_rating=Avg('reviews__rating'),
                categories_count=Count('categories')
            )
        
        return queryset
    
    def get_serializer_class(self):
        if self.action == 'list':
            return BookListSerializer
        return BookDetailSerializer
```

## Conditional Field Loading

```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """
    A serializer that takes an additional `fields` argument to
    control which fields should be displayed.
    """
    def __init__(self, *args, **kwargs):
        fields = kwargs.pop('fields', None)
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)

class BookOptimizedSerializer(DynamicFieldsSerializer):
    author_name = serializers.CharField(source='author.name', read_only=True)
    publisher_name = serializers.CharField(source='publisher.name', read_only=True)
    categories = CategorySerializer(many=True, read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)
    
    class Meta:
        model = Book
        fields = [
            'id', 'title', 'author_name', 'publisher_name',
            'categories', 'reviews', 'isbn', 'published_date', 'price'
        ]

class BookViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Book.objects.select_related('author', 'publisher')
        
        # Get fields parameter from query params
        fields = self.request.query_params.get('fields', '').split(',')
        
        # Only prefetch if related fields are requested
        if any(field in fields for field in ['categories', 'reviews']):
            prefetch_list = []
            if 'categories' in fields:
                prefetch_list.append('categories')
            if 'reviews' in fields:
                prefetch_list.append('reviews')
            
            queryset = queryset.prefetch_related(*prefetch_list)
        
        return queryset
    
    def get_serializer(self, *args, **kwargs):
        fields = self.request.query_params.get('fields')
        if fields:
            kwargs['fields'] = fields.split(',')
        return super().get_serializer(*args, **kwargs)
```

## Measuring Query Performance

```python
import time
from django.db import connection
from django.conf import settings

class QueryCountMiddleware:
    """Middleware to count database queries in development"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if settings.DEBUG:
            initial_queries = len(connection.queries)
            start_time = time.time()
        
        response = self.get_response(request)
        
        if settings.DEBUG:
            end_time = time.time()
            query_count = len(connection.queries) - initial_queries
            response['X-Query-Count'] = str(query_count)
            response['X-Query-Time'] = f"{(end_time - start_time):.3f}s"
        
        return response

# Add to views for debugging
class BookViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        if settings.DEBUG:
            from django.db import connection
            initial_queries = len(connection.queries)
        
        response = super().list(request, *args, **kwargs)
        
        if settings.DEBUG:
            query_count = len(connection.queries) - initial_queries
            print(f"Queries executed: {query_count}")
            for query in connection.queries[-query_count:]:
                print(f"Query: {query['sql']}")
        
        return response
```

## Best Practices Summary

```python
# Good practices for related field optimization
class OptimizedBookViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Always use select_related for ForeignKey and OneToOne
        queryset = Book.objects.select_related('author', 'publisher')
        
        # Use prefetch_related for ManyToMany and reverse ForeignKey
        if self.action in ['list', 'retrieve']:
            queryset = queryset.prefetch_related('categories')
        
        # Use Prefetch objects for complex related queries
        if self.action == 'retrieve':
            recent_reviews = Review.objects.select_related('book').order_by('-created_at')[:10]
            queryset = queryset.prefetch_related(
                Prefetch('reviews', queryset=recent_reviews, to_attr='recent_reviews')
            )
        
        # Use annotations for computed fields
        if self.action == 'list':
            queryset = queryset.annotate(
                reviews_count=Count('reviews'),
                average_rating=Avg('reviews__rating')
            )
        
        return queryset
    
    def get_serializer_class(self):
        # Use different serializers for list vs detail
        if self.action == 'list':
            return BookListSerializer
        return BookDetailSerializer
```

**Notes:**
- Use `select_related` for ForeignKey and OneToOne relationships
- Use `prefetch_related` for ManyToMany and reverse ForeignKey relationships
- Use `Prefetch` objects for complex related queries with filtering/ordering
- Use annotations to compute aggregate values in the database
- Consider using different serializers for list vs detail views
- Monitor query count and performance in development
- Be careful not to over-optimize - only optimize what you actually need
- Test with realistic data volumes to ensure optimizations are effective
