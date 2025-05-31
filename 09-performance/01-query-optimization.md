# Query Optimization

**Problem:** Your DRF API endpoints are slow due to inefficient database queries, N+1 query problems, and lack of proper database optimization techniques.

**Solution:** Implement comprehensive query optimization strategies including select_related, prefetch_related, query annotations, and database-level optimizations.

## Identifying Query Performance Issues

```python
# settings.py - Enable query debugging
import logging

# Database query logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

# Show SQL queries in development
if DEBUG:
    LOGGING['loggers']['django.db.backends']['level'] = 'DEBUG'
```

```python
# utils/query_debugger.py
from django.db import connection
from django.conf import settings
import time
import functools

def query_debugger(func):
    """Decorator to debug queries in a function/method"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        if not settings.DEBUG:
            return func(*args, **kwargs)
        
        initial_queries = len(connection.queries)
        start_time = time.time()
        
        result = func(*args, **kwargs)
        
        end_time = time.time()
        final_queries = len(connection.queries)
        
        print(f"\n{'='*50}")
        print(f"Function: {func.__name__}")
        print(f"Queries executed: {final_queries - initial_queries}")
        print(f"Execution time: {(end_time - start_time):.4f} seconds")
        
        if final_queries - initial_queries > 0:
            print("\nQueries:")
            for query in connection.queries[initial_queries:]:
                print(f"  {query['sql'][:100]}...")
        print(f"{'='*50}\n")
        
        return result
    return wrapper

# Usage example
class BookViewSet(viewsets.ModelViewSet):
    @query_debugger
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

## Basic Query Optimization

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
    bio = models.TextField(blank=True)
    birth_date = models.DateField(null=True, blank=True)
    country = models.CharField(max_length=100, blank=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['name']),
            models.Index(fields=['country']),
        ]

class Publisher(models.Model):
    name = models.CharField(max_length=100)
    website = models.URLField(blank=True)
    established_year = models.IntegerField(null=True, blank=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['name']),
        ]

class Category(models.Model):
    name = models.CharField(max_length=50, unique=True)
    description = models.TextField(blank=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['name']),
        ]

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE, related_name='books')
    categories = models.ManyToManyField(Category, related_name='books')
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    pages = models.IntegerField(default=0)
    is_bestseller = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['title']),
            models.Index(fields=['isbn']),
            models.Index(fields=['published_date']),
            models.Index(fields=['price']),
            models.Index(fields=['is_bestseller']),
            models.Index(fields=['created_at']),
            # Composite indexes for common query patterns
            models.Index(fields=['author', 'published_date']),
            models.Index(fields=['publisher', 'is_bestseller']),
        ]

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    reviewer_name = models.CharField(max_length=100)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    helpful_votes = models.IntegerField(default=0)
    
    class Meta:
        indexes = [
            models.Index(fields=['book', 'rating']),
            models.Index(fields=['created_at']),
            models.Index(fields=['rating']),
        ]
```

## Optimized ViewSets

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Prefetch, Count, Avg, Q, F
from .models import Book, Author, Review
from .serializers import BookSerializer, BookDetailSerializer, AuthorSerializer

class OptimizedBookViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        """Optimize queryset based on the action"""
        base_queryset = Book.objects.select_related('author', 'publisher')
        
        if self.action == 'list':
            # For list view, we need basic info + counts
            return base_queryset.prefetch_related('categories').annotate(
                reviews_count=Count('reviews'),
                average_rating=Avg('reviews__rating')
            ).order_by('-created_at')
        
        elif self.action == 'retrieve':
            # For detail view, we need everything including reviews
            return base_queryset.prefetch_related(
                'categories',
                Prefetch(
                    'reviews',
                    queryset=Review.objects.select_related('book').order_by('-created_at')
                )
            )
        
        elif self.action in ['popular', 'bestsellers']:
            # For popularity-based views
            return base_queryset.prefetch_related('categories').annotate(
                reviews_count=Count('reviews'),
                average_rating=Avg('reviews__rating')
            ).filter(
                reviews_count__gte=5,
                average_rating__gte=4.0
            ).order_by('-average_rating', '-reviews_count')
        
        return base_queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'retrieve':
            return BookDetailSerializer
        return BookSerializer
    
    @action(detail=False, methods=['get'])
    def popular(self, request):
        """Get popular books with optimized query"""
        queryset = self.get_queryset()
        
        # Apply additional filters if needed
        min_rating = request.query_params.get('min_rating', 4.0)
        queryset = queryset.filter(average_rating__gte=min_rating)
        
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def by_author(self, request):
        """Get books by author with optimized query"""
        author_id = request.query_params.get('author_id')
        if not author_id:
            return Response(
                {'error': 'author_id parameter is required'}, 
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Optimized query for books by specific author
        queryset = Book.objects.filter(author_id=author_id).select_related(
            'author', 'publisher'
        ).prefetch_related('categories').annotate(
            reviews_count=Count('reviews'),
            average_rating=Avg('reviews__rating')
        ).order_by('-published_date')
        
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

class OptimizedAuthorViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = AuthorSerializer
    
    def get_queryset(self):
        """Optimize author queries"""
        if self.action == 'list':
            # For list, prefetch books count
            return Author.objects.annotate(
                books_count=Count('books'),
                total_reviews=Count('books__reviews'),
                average_book_rating=Avg('books__reviews__rating')
            ).order_by('-books_count')
        
        elif self.action == 'retrieve':
            # For detail, prefetch recent books
            recent_books = Book.objects.select_related('publisher').prefetch_related(
                'categories'
            ).annotate(
                reviews_count=Count('reviews'),
                average_rating=Avg('reviews__rating')
            ).order_by('-published_date')[:10]
            
            return Author.objects.prefetch_related(
                Prefetch('books', queryset=recent_books, to_attr='recent_books')
            )
        
        return Author.objects.all()
    
    @action(detail=True, methods=['get'])
    def statistics(self, request, pk=None):
        """Get author statistics with optimized query"""
        author = self.get_object()
        
        # Single query to get all statistics
        stats = author.books.aggregate(
            total_books=Count('id'),
            total_reviews=Count('reviews'),
            average_rating=Avg('reviews__rating'),
            highest_rated_book_rating=models.Max('reviews__rating'),
            most_expensive_book=models.Max('price'),
            total_pages=models.Sum('pages')
        )
        
        # Get genre distribution
        genres = author.books.values('categories__name').annotate(
            count=Count('categories__name')
        ).filter(categories__name__isnull=False).order_by('-count')
        
        stats['genre_distribution'] = list(genres)
        
        return Response(stats)
```

## Advanced Query Optimization Techniques

```python
# managers.py
from django.db import models
from django.db.models import Count, Avg, Q, F, Prefetch

class BookQuerySet(models.QuerySet):
    def with_reviews_stats(self):
        """Add review statistics to queryset"""
        return self.annotate(
            reviews_count=Count('reviews'),
            average_rating=Avg('reviews__rating'),
            five_star_reviews=Count('reviews', filter=Q(reviews__rating=5)),
            recent_reviews_count=Count('reviews', filter=Q(
                reviews__created_at__gte=models.F('created_at') - models.DurationField(days=30)
            ))
        )
    
    def with_author_publisher(self):
        """Include author and publisher data"""
        return self.select_related('author', 'publisher')
    
    def with_categories(self):
        """Include categories data"""
        return self.prefetch_related('categories')
    
    def with_recent_reviews(self, limit=5):
        """Include recent reviews"""
        recent_reviews = Review.objects.order_by('-created_at')[:limit]
        return self.prefetch_related(
            Prefetch('reviews', queryset=recent_reviews, to_attr='recent_reviews')
        )
    
    def bestsellers(self):
        """Filter bestsellers with high ratings"""
        return self.with_reviews_stats().filter(
            reviews_count__gte=10,
            average_rating__gte=4.0
        ).order_by('-average_rating', '-reviews_count')
    
    def recent(self, days=30):
        """Filter recent books"""
        from datetime import timedelta
        from django.utils import timezone
        
        cutoff_date = timezone.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff_date)
    
    def by_price_range(self, min_price=None, max_price=None):
        """Filter by price range"""
        queryset = self
        if min_price is not None:
            queryset = queryset.filter(price__gte=min_price)
        if max_price is not None:
            queryset = queryset.filter(price__lte=max_price)
        return queryset
    
    def search(self, query):
        """Full-text search across multiple fields"""
        return self.filter(
            Q(title__icontains=query) |
            Q(author__name__icontains=query) |
            Q(publisher__name__icontains=query) |
            Q(categories__name__icontains=query)
        ).distinct()

class BookManager(models.Manager):
    def get_queryset(self):
        return BookQuerySet(self.model, using=self._db)
    
    def with_reviews_stats(self):
        return self.get_queryset().with_reviews_stats()
    
    def with_author_publisher(self):
        return self.get_queryset().with_author_publisher()
    
    def bestsellers(self):
        return self.get_queryset().bestsellers()
    
    def recent(self, days=30):
        return self.get_queryset().recent(days)

# Add to Book model
class Book(models.Model):
    # ... existing fields ...
    
    objects = BookManager()
    
    class Meta:
        # ... existing meta ...
        pass
```

## Using Raw SQL for Complex Queries

```python
# views.py - Complex aggregation with raw SQL
from django.db import connection

class BookAnalyticsViewSet(viewsets.ViewSet):
    @action(detail=False, methods=['get'])
    def complex_analytics(self, request):
        """Complex analytics using raw SQL for performance"""
        
        query = """
        WITH author_stats AS (
            SELECT 
                a.id,
                a.name,
                COUNT(b.id) as book_count,
                AVG(b.price) as avg_price,
                AVG(r.rating) as avg_rating,
                COUNT(r.id) as total_reviews
            FROM api_author a
            LEFT JOIN api_book b ON a.id = b.author_id
            LEFT JOIN api_review r ON b.id = r.book_id
            GROUP BY a.id, a.name
        ),
        category_stats AS (
            SELECT 
                c.name as category_name,
                COUNT(DISTINCT b.id) as book_count,
                AVG(b.price) as avg_price
            FROM api_category c
            JOIN api_book_categories bc ON c.id = bc.category_id
            JOIN api_book b ON bc.book_id = b.id
            GROUP BY c.id, c.name
        )
        SELECT 
            'author_performance' as metric_type,
            JSON_OBJECT(
                'top_authors', JSON_ARRAYAGG(
                    JSON_OBJECT(
                        'name', name,
                        'book_count', book_count,
                        'avg_price', avg_price,
                        'avg_rating', avg_rating
                    )
                )
            ) as data
        FROM (
            SELECT * FROM author_stats 
            WHERE book_count > 0 
            ORDER BY avg_rating DESC, book_count DESC 
            LIMIT 10
        ) top_authors
        
        UNION ALL
        
        SELECT 
            'category_performance' as metric_type,
            JSON_OBJECT(
                'categories', JSON_ARRAYAGG(
                    JSON_OBJECT(
                        'name', category_name,
                        'book_count', book_count,
                        'avg_price', avg_price
                    )
                )
            ) as data
        FROM category_stats
        ORDER BY book_count DESC;
        """
        
        with connection.cursor() as cursor:
            cursor.execute(query)
            results = cursor.fetchall()
        
        # Process results
        analytics_data = {}
        for row in results:
            metric_type = row[0]
            data = json.loads(row[1]) if row[1] else {}
            analytics_data[metric_type] = data
        
        return Response(analytics_data)

    @action(detail=False, methods=['get'])
    def monthly_trends(self, request):
        """Monthly trends with optimized query"""
        
        query = """
        SELECT 
            DATE_FORMAT(published_date, '%%Y-%%m') as month,
            COUNT(*) as books_published,
            AVG(price) as avg_price,
            COUNT(DISTINCT author_id) as unique_authors,
            AVG(pages) as avg_pages
        FROM api_book 
        WHERE published_date >= DATE_SUB(CURDATE(), INTERVAL 24 MONTH)
        GROUP BY DATE_FORMAT(published_date, '%%Y-%%m')
        ORDER BY month DESC;
        """
        
        with connection.cursor() as cursor:
            cursor.execute(query)
            columns = [col[0] for col in cursor.description]
            results = [dict(zip(columns, row)) for row in cursor.fetchall()]
        
        return Response({
            'trends': results,
            'summary': {
                'total_months': len(results),
                'avg_books_per_month': sum(r['books_published'] for r in results) / len(results) if results else 0
            }
        })
```

## Database-Level Optimizations

```python
# migrations/xxxx_add_database_optimizations.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('api', '0001_initial'),
    ]
    
    operations = [
        # Add database-specific indexes
        migrations.RunSQL(
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_book_title_gin ON api_book USING gin(to_tsvector('english', title));",
            reverse_sql="DROP INDEX IF EXISTS idx_book_title_gin;"
        ),
        
        # Add partial indexes
        migrations.RunSQL(
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_book_bestseller_recent ON api_book (published_date) WHERE is_bestseller = true;",
            reverse_sql="DROP INDEX IF EXISTS idx_book_bestseller_recent;"
        ),
        
        # Add expression indexes
        migrations.RunSQL(
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_book_price_range ON api_book ((CASE WHEN price < 20 THEN 'budget' WHEN price < 50 THEN 'mid' ELSE 'premium' END));",
            reverse_sql="DROP INDEX IF EXISTS idx_book_price_range;"
        ),
    ]
```

## Caching Query Results

```python
# utils/cache_utils.py
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key
import hashlib
import json

def cache_query_result(timeout=300):
    """Decorator to cache query results"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            # Create cache key from function name and arguments
            key_data = {
                'func': func.__name__,
                'args': str(args),
                'kwargs': str(sorted(kwargs.items()))
            }
            cache_key = hashlib.md5(
                json.dumps(key_data, sort_keys=True).encode()
            ).hexdigest()
            
            # Try to get from cache
            result = cache.get(cache_key)
            if result is not None:
                return result
            
            # Execute function and cache result
            result = func(*args, **kwargs)
            cache.set(cache_key, result, timeout)
            return result
        
        return wrapper
    return decorator

# Usage in views
class CachedBookViewSet(viewsets.ModelViewSet):
    @cache_query_result(timeout=600)  # Cache for 10 minutes
    def get_popular_books(self):
        """Get popular books with caching"""
        return Book.objects.bestsellers().with_author_publisher().with_categories()[:20]
    
    @action(detail=False, methods=['get'])
    def popular(self, request):
        books = self.get_popular_books()
        serializer = self.get_serializer(books, many=True)
        return Response(serializer.data)
```

## Monitoring and Profiling

```python
# middleware/query_monitor.py
import time
from django.db import connection
from django.conf import settings

class QueryMonitorMiddleware:
    """Middleware to monitor database queries"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if not settings.DEBUG:
            return self.get_response(request)
        
        initial_queries = len(connection.queries)
        start_time = time.time()
        
        response = self.get_response(request)
        
        end_time = time.time()
        query_count = len(connection.queries) - initial_queries
        query_time = sum(float(q['time']) for q in connection.queries[initial_queries:])
        
        # Add performance headers
        response['X-Query-Count'] = str(query_count)
        response['X-Query-Time'] = f"{query_time:.4f}"
        response['X-Response-Time'] = f"{(end_time - start_time):.4f}"
        
        # Log slow queries
        if query_count > 10 or query_time > 1.0:
            print(f"SLOW REQUEST: {request.path}")
            print(f"Queries: {query_count}, Time: {query_time:.4f}s")
            for query in connection.queries[initial_queries:]:
                if float(query['time']) > 0.1:  # Log queries > 100ms
                    print(f"SLOW QUERY ({query['time']}s): {query['sql'][:200]}...")
        
        return response
```

**Notes:**
- Always use `select_related` for ForeignKey relationships
- Use `prefetch_related` for ManyToMany and reverse ForeignKey relationships
- Create database indexes for frequently queried fields
- Use `only()` and `defer()` to load specific fields when needed
- Implement custom managers and querysets for reusable optimizations
- Monitor query performance with middleware and debugging tools
- Consider using raw SQL for complex aggregations
- Cache expensive query results appropriately
- Use database-specific optimizations like partial and expression indexes
