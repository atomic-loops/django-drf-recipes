# Caching Responses

## Problem

API responses can be expensive to generate due to complex database queries, serialization, or external API calls. You need to cache responses to improve performance.

## Solution

Implement multiple caching strategies including view-level caching, cache decorators, and cache versioning for efficient response caching.

## Code Examples

### Basic Response Caching

```python
# views.py
from django.core.cache import cache
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book, Author
from .serializers import BookSerializer, AuthorSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @method_decorator(cache_page(60 * 15))  # Cache for 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
    
    @method_decorator(cache_page(60 * 60))  # Cache for 1 hour
    def retrieve(self, request, *args, **kwargs):
        return super().retrieve(request, *args, **kwargs)

# Alternative using cache_page decorator
from django.views.decorators.vary import vary_on_headers

@method_decorator(vary_on_headers('Authorization'), name='dispatch')
@method_decorator(cache_page(60 * 15), name='list')
class AuthorViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer
```

### Manual Cache Management

```python
# views.py
import hashlib
from django.core.cache import cache
from rest_framework import viewsets, status
from rest_framework.response import Response

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_cache_key(self, request, view_name, *args, **kwargs):
        """Generate a unique cache key based on request parameters."""
        # Include user ID for user-specific caching
        user_id = request.user.id if request.user.is_authenticated else 'anonymous'
        
        # Include query parameters
        query_params = sorted(request.query_params.items())
        query_string = '&'.join([f"{k}={v}" for k, v in query_params])
        
        # Create cache key
        cache_data = f"{view_name}:{user_id}:{query_string}"
        return hashlib.md5(cache_data.encode()).hexdigest()
    
    def list(self, request, *args, **kwargs):
        cache_key = self.get_cache_key(request, 'book_list')
        cached_response = cache.get(cache_key)
        
        if cached_response:
            return Response(cached_response)
        
        response = super().list(request, *args, **kwargs)
        
        # Cache the response data for 15 minutes
        if response.status_code == status.HTTP_200_OK:
            cache.set(cache_key, response.data, timeout=60 * 15)
        
        return response
    
    def create(self, request, *args, **kwargs):
        response = super().create(request, *args, **kwargs)
        
        # Invalidate list cache when new book is created
        if response.status_code == status.HTTP_201_CREATED:
            self.invalidate_list_cache()
        
        return response
    
    def invalidate_list_cache(self):
        """Invalidate all list caches."""
        # This is a simple approach - in production, you might want
        # more sophisticated cache invalidation
        cache.delete_many([
            'book_list_page_1',
            'book_list_page_2',
            # Add more specific keys as needed
        ])
```

### Cache Middleware for ViewSets

```python
# middleware.py
import hashlib
from django.core.cache import cache
from django.utils.deprecation import MiddlewareMixin

class ViewSetCacheMiddleware(MiddlewareMixin):
    """Cache middleware specifically for DRF ViewSets."""
    
    CACHE_TIMEOUT = 60 * 15  # 15 minutes
    CACHEABLE_METHODS = ['GET']
    CACHEABLE_PATHS = ['/api/books/', '/api/authors/']
    
    def process_request(self, request):
        if not self._should_cache(request):
            return None
        
        cache_key = self._get_cache_key(request)
        cached_response = cache.get(cache_key)
        
        if cached_response:
            return cached_response
        
        return None
    
    def process_response(self, request, response):
        if not self._should_cache(request):
            return response
        
        if response.status_code == 200:
            cache_key = self._get_cache_key(request)
            cache.set(cache_key, response, timeout=self.CACHE_TIMEOUT)
        
        return response
    
    def _should_cache(self, request):
        return (
            request.method in self.CACHEABLE_METHODS and
            any(request.path.startswith(path) for path in self.CACHEABLE_PATHS)
        )
    
    def _get_cache_key(self, request):
        key_data = f"{request.path}:{request.GET.urlencode()}"
        if hasattr(request, 'user') and request.user.is_authenticated:
            key_data += f":{request.user.id}"
        return hashlib.md5(key_data.encode()).hexdigest()
```

### Cached Property for Expensive Serializer Fields

```python
# serializers.py
from django.utils.functional import cached_property
from rest_framework import serializers
from .models import Author, Book

class AuthorSerializer(serializers.ModelSerializer):
    book_count = serializers.SerializerMethodField()
    average_rating = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'bio', 'book_count', 'average_rating']
    
    def get_book_count(self, obj):
        # Cache expensive computation
        cache_key = f"author_{obj.id}_book_count"
        book_count = cache.get(cache_key)
        
        if book_count is None:
            book_count = obj.books.count()
            cache.set(cache_key, book_count, timeout=60 * 60)  # 1 hour
        
        return book_count
    
    def get_average_rating(self, obj):
        cache_key = f"author_{obj.id}_avg_rating"
        avg_rating = cache.get(cache_key)
        
        if avg_rating is None:
            from django.db.models import Avg
            avg_rating = obj.books.aggregate(
                avg_rating=Avg('reviews__rating')
            )['avg_rating'] or 0
            cache.set(cache_key, avg_rating, timeout=60 * 30)  # 30 minutes
        
        return round(avg_rating, 2)

# Alternative using model cached properties
class Author(models.Model):
    name = models.CharField(max_length=100)
    bio = models.TextField()
    
    @cached_property
    def cached_book_count(self):
        return self.books.count()
    
    @cached_property
    def cached_average_rating(self):
        from django.db.models import Avg
        return self.books.aggregate(
            avg_rating=Avg('reviews__rating')
        )['avg_rating'] or 0
```

### Cache Versioning

```python
# utils/cache.py
from django.core.cache import cache
from django.conf import settings

class VersionedCache:
    """Cache with automatic versioning for cache invalidation."""
    
    def __init__(self, namespace):
        self.namespace = namespace
        self.version_key = f"{namespace}:version"
    
    def get_version(self):
        version = cache.get(self.version_key)
        if version is None:
            version = 1
            cache.set(self.version_key, version, timeout=None)
        return version
    
    def get_key(self, key):
        version = self.get_version()
        return f"{self.namespace}:v{version}:{key}"
    
    def get(self, key, default=None):
        return cache.get(self.get_key(key), default)
    
    def set(self, key, value, timeout=None):
        return cache.set(self.get_key(key), value, timeout)
    
    def delete(self, key):
        return cache.delete(self.get_key(key))
    
    def invalidate_all(self):
        """Invalidate all cached data by incrementing version."""
        version = self.get_version()
        cache.set(self.version_key, version + 1, timeout=None)

# Usage in views
book_cache = VersionedCache('books')

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def list(self, request, *args, **kwargs):
        cache_key = f"list:{request.GET.urlencode()}"
        cached_data = book_cache.get(cache_key)
        
        if cached_data:
            return Response(cached_data)
        
        response = super().list(request, *args, **kwargs)
        
        if response.status_code == 200:
            book_cache.set(cache_key, response.data, timeout=60 * 15)
        
        return response
    
    def create(self, request, *args, **kwargs):
        response = super().create(request, *args, **kwargs)
        
        if response.status_code == 201:
            # Invalidate all book caches
            book_cache.invalidate_all()
        
        return response
```

### Redis Cache Configuration

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SERIALIZER': 'django_redis.serializers.json.JSONSerializer',
            'COMPRESSOR': 'django_redis.compressors.zlib.ZlibCompressor',
        },
        'KEY_PREFIX': 'drf_cookbook',
        'TIMEOUT': 60 * 15,  # 15 minutes default
    }
}

# Cache key versioning
CACHES['default']['VERSION'] = 1

# Connection pooling
CACHES['default']['OPTIONS']['CONNECTION_POOL_KWARGS'] = {
    'max_connections': 20,
    'retry_on_timeout': True,
}
```

### Cache Invalidation Signals

```python
# signals.py
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.core.cache import cache
from .models import Book, Author, Review

@receiver([post_save, post_delete], sender=Book)
def invalidate_book_cache(sender, instance, **kwargs):
    """Invalidate book-related caches when book is modified."""
    # Invalidate specific book cache
    cache.delete(f"book_{instance.id}")
    
    # Invalidate book list caches
    cache.delete_many([
        'book_list_page_1',
        'book_list_page_2',
        'book_list_featured',
    ])
    
    # Invalidate author's book count cache
    cache.delete(f"author_{instance.author.id}_book_count")

@receiver([post_save, post_delete], sender=Review)
def invalidate_rating_cache(sender, instance, **kwargs):
    """Invalidate rating caches when review is modified."""
    book = instance.book
    author = book.author
    
    # Invalidate average rating caches
    cache.delete(f"book_{book.id}_avg_rating")
    cache.delete(f"author_{author.id}_avg_rating")

# apps.py
from django.apps import AppConfig

class BooksConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'books'
    
    def ready(self):
        import books.signals
```

## Notes

- **Cache Keys**: Use descriptive, unique cache keys that include relevant parameters
- **Cache Invalidation**: Implement proper cache invalidation when data changes
- **Memory Usage**: Monitor cache memory usage, especially with large datasets
- **Cache Timeout**: Set appropriate timeouts based on data freshness requirements
- **User-Specific Caching**: Consider user permissions when caching data
- **Cache Warming**: Implement cache warming for critical data
- **Monitoring**: Monitor cache hit/miss ratios to optimize caching strategies
- **Testing**: Test cache behavior in development and staging environments
