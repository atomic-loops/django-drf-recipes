# Pagination Techniques

## Problem

You need to implement efficient pagination for large datasets in your API to improve performance, reduce memory usage, and provide a better user experience when dealing with thousands or millions of records.

## Solution

Use DRF's built-in pagination classes or create custom pagination to handle large result sets efficiently. Choose the right pagination style based on your use case: page-based, limit/offset, or cursor pagination.

## Page Number Pagination

### Basic Implementation

```python
# pagination.py
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'page_size': self.page_size,
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'results': data
        })
```

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    view_count = models.PositiveIntegerField(default=0)
    is_published = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['is_published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title
```

```python
# views.py
from rest_framework import generics
from .models import Post
from .serializers import PostSerializer
from .pagination import StandardResultsSetPagination

class PostListView(generics.ListAPIView):
    queryset = Post.objects.filter(is_published=True).select_related('author')
    serializer_class = PostSerializer
    pagination_class = StandardResultsSetPagination
```

### Custom Page Number Pagination

```python
class CustomPageNumberPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = 'size'
    max_page_size = 1000
    
    def get_paginated_response(self, data):
        return Response({
            'pagination': {
                'count': self.page.paginator.count,
                'page_size': self.get_page_size(self.request),
                'current_page': self.page.number,
                'total_pages': self.page.paginator.num_pages,
                'has_next': self.page.has_next(),
                'has_previous': self.page.has_previous(),
                'next_page': self.page.next_page_number() if self.page.has_next() else None,
                'previous_page': self.page.previous_page_number() if self.page.has_previous() else None,
                'links': {
                    'next': self.get_next_link(),
                    'previous': self.get_previous_link(),
                }
            },
            'data': data
        })
    
    def get_page_size(self, request):
        """Override to add custom page size logic"""
        page_size = super().get_page_size(request)
        
        # Adjust page size based on user type
        if hasattr(request.user, 'profile'):
            if request.user.profile.is_premium:
                return min(page_size, 100)
            else:
                return min(page_size, 25)
        
        return page_size
```

## Limit Offset Pagination

```python
from rest_framework.pagination import LimitOffsetPagination

class CustomLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    limit_query_param = 'limit'
    offset_query_param = 'offset'
    max_limit = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'limit': self.limit,
            'offset': self.offset,
            'results': data
        })

class PostLimitOffsetView(generics.ListAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    pagination_class = CustomLimitOffsetPagination
```

## Cursor Pagination

### Basic Cursor Pagination

```python
from rest_framework.pagination import CursorPagination

class PostCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must be an orderable field
    cursor_query_param = 'cursor'
    page_size_query_param = 'page_size'
    
    def get_paginated_response(self, data):
        return Response({
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data
        })

class PostCursorView(generics.ListAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    pagination_class = PostCursorPagination
```

### Advanced Cursor Pagination

```python
class AdvancedCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'
    cursor_query_param = 'cursor'
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'page_size': self.page_size,
            'ordering': self.ordering,
            'results': data
        })
    
    def get_ordering(self, request, queryset, view):
        """Custom ordering logic"""
        ordering_param = request.query_params.get('ordering')
        
        if ordering_param:
            # Validate and transform ordering parameter
            valid_orderings = {
                'newest': '-created_at',
                'oldest': 'created_at',
                'popular': '-view_count',
                'title': 'title',
            }
            
            if ordering_param in valid_orderings:
                return valid_orderings[ordering_param]
        
        return self.ordering
```

## Custom Pagination Classes

### Keyset Pagination

```python
class KeysetPagination(PageNumberPagination):
    """Efficient pagination for large datasets using keyset pagination"""
    page_size = 50
    max_page_size = 200
    
    def paginate_queryset(self, queryset, request, view=None):
        """Override to implement keyset pagination"""
        self.request = request
        self.view = view
        
        # Get cursor parameters
        after_id = request.query_params.get('after')
        before_id = request.query_params.get('before')
        
        # Apply keyset filtering
        if after_id:
            queryset = queryset.filter(id__gt=after_id)
        elif before_id:
            queryset = queryset.filter(id__lt=before_id).order_by('-id')
        
        # Get one extra item to check if there's a next page
        page_size = self.get_page_size(request)
        queryset = queryset[:page_size + 1]
        
        items = list(queryset)
        
        # Check if there are more items
        self.has_next = len(items) > page_size
        if self.has_next:
            items = items[:-1]
        
        # Reverse if we were going backwards
        if before_id:
            items = list(reversed(items))
        
        return items
    
    def get_paginated_response(self, data):
        next_cursor = None
        previous_cursor = None
        
        if data:
            if self.has_next:
                next_cursor = data[-1]['id']
            if self.request.query_params.get('after'):
                previous_cursor = data[0]['id']
        
        return Response({
            'next': next_cursor,
            'previous': previous_cursor,
            'results': data
        })
```

### Infinite Scroll Pagination

```python
class InfiniteScrollPagination(CursorPagination):
    """Pagination optimized for infinite scroll interfaces"""
    page_size = 20
    ordering = '-created_at'
    
    def get_paginated_response(self, data):
        return Response({
            'has_more': self.has_next,
            'next_cursor': self.get_next_link(),
            'results': data,
            'count': len(data)
        })

class InfiniteScrollView(generics.ListAPIView):
    queryset = Post.objects.filter(is_published=True)
    serializer_class = PostSerializer
    pagination_class = InfiniteScrollPagination
```

### Metadata-Enhanced Pagination

```python
class MetadataPagination(PageNumberPagination):
    """Pagination with additional metadata for frontend optimization"""
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        # Calculate pagination metadata
        start_index = (self.page.number - 1) * self.page_size + 1
        end_index = min(start_index + len(data) - 1, self.page.paginator.count)
        
        # Generate page range for pagination UI
        page_range = self.get_page_range()
        
        return Response({
            'count': self.page.paginator.count,
            'page_size': self.page_size,
            'current_page': self.page.number,
            'total_pages': self.page.paginator.num_pages,
            'start_index': start_index,
            'end_index': end_index,
            'page_range': page_range,
            'links': {
                'first': self.get_first_link(),
                'last': self.get_last_link(),
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
            },
            'results': data
        })
    
    def get_page_range(self):
        """Generate page range for pagination UI"""
        current_page = self.page.number
        total_pages = self.page.paginator.num_pages
        
        # Show 5 pages around current page
        start = max(1, current_page - 2)
        end = min(total_pages + 1, current_page + 3)
        
        return list(range(start, end))
    
    def get_first_link(self):
        """Generate link to first page"""
        if self.page.number <= 1:
            return None
        
        url = self.request.build_absolute_uri()
        return self.replace_query_param(url, self.page_query_param, 1)
    
    def get_last_link(self):
        """Generate link to last page"""
        if self.page.number >= self.page.paginator.num_pages:
            return None
        
        url = self.request.build_absolute_uri()
        return self.replace_query_param(
            url, 
            self.page_query_param, 
            self.page.paginator.num_pages
        )
```

## ViewSet Integration

```python
from rest_framework import viewsets
from django.core.cache import cache

class PostViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = PostSerializer
    pagination_class = StandardResultsSetPagination
    
    def get_queryset(self):
        """Optimize queryset for pagination"""
        queryset = Post.objects.filter(is_published=True)
        
        # Optimize based on pagination type
        if isinstance(self.pagination_class, CursorPagination):
            # For cursor pagination, ensure ordering field is indexed
            queryset = queryset.select_related('author')
        else:
            # For page number pagination, optimize count queries
            queryset = queryset.select_related('author')
        
        return queryset
    
    def list(self, request, *args, **kwargs):
        """Add custom pagination metadata"""
        response = super().list(request, *args, **kwargs)
        
        # Add additional metadata
        if hasattr(response, 'data') and 'results' in response.data:
            response.data['meta'] = {
                'total_items': response.data.get('count', 0),
                'items_per_page': self.pagination_class.page_size,
                'request_time': timezone.now().isoformat(),
            }
        
        return response
```

## Performance Optimization

### Optimized Count Queries

```python
class OptimizedPagination(PageNumberPagination):
    """Pagination with optimized count queries for large datasets"""
    page_size = 20
    
    def get_paginated_response(self, data):
        # Use approximate count for very large datasets
        count = self.get_count()
        
        return Response({
            'count': count,
            'count_estimate': count > 100000,  # Flag if count is estimated
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data
        })
    
    def get_count(self):
        """Get count with optimization for large datasets"""
        queryset = self.page.paginator.object_list
        
        # For very large datasets, use estimated count
        try:
            # Try exact count first
            count = queryset.count()
            
            # If count is very large, consider using estimated count
            if count > 1000000:
                # For PostgreSQL, you could use estimated count
                # from django.db import connection
                # with connection.cursor() as cursor:
                #     cursor.execute("SELECT reltuples FROM pg_class WHERE relname = %s", [queryset.model._meta.db_table])
                #     return int(cursor.fetchone()[0])
                pass
            
            return count
        except Exception:
            return 0
```

### Cached Pagination

```python
class CachedPagination(PageNumberPagination):
    """Pagination with caching for expensive count queries"""
    page_size = 20
    cache_timeout = 300  # 5 minutes
    
    def get_count(self):
        """Get cached count for expensive queries"""
        queryset = self.page.paginator.object_list
        
        # Generate cache key based on queryset
        cache_key = f"pagination_count_{hash(str(queryset.query))}"
        
        count = cache.get(cache_key)
        if count is None:
            count = queryset.count()
            cache.set(cache_key, count, self.cache_timeout)
        
        return count
```

## URL Examples

With the pagination setup, your API accepts these URL patterns:

```
# Page Number Pagination
GET /api/posts/?page=1
GET /api/posts/?page=2&page_size=50

# Limit Offset Pagination
GET /api/posts/?limit=20&offset=40
GET /api/posts/?limit=50&offset=100

# Cursor Pagination
GET /api/posts/?cursor=cD0yMDIzLTEyLTE1KzAzJTNBMDA%3D
GET /api/posts/?cursor=next_cursor_token

# Keyset Pagination
GET /api/posts/?after=12345
GET /api/posts/?before=54321

# Combined with filtering and ordering
GET /api/posts/?page=2&ordering=-created_at&author=john
GET /api/posts/?limit=25&search=django&page_size=25
```

## Settings Configuration

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'myapp.pagination.StandardResultsSetPagination',
    'PAGE_SIZE': 20,
}

# Custom pagination settings
PAGINATION_SETTINGS = {
    'DEFAULT_PAGE_SIZE': 20,
    'MAX_PAGE_SIZE': 100,
    'CACHE_TIMEOUT': 300,
    'USE_CURSOR_FOR_LARGE_DATASETS': True,
    'LARGE_DATASET_THRESHOLD': 100000,
}
```

## Advanced Use Cases

### Dynamic Pagination Strategy

```python
class DynamicPagination:
    """Choose pagination strategy based on dataset size and user preferences"""
    
    @staticmethod
    def get_pagination_class(queryset, request):
        """Select appropriate pagination based on conditions"""
        count = queryset.count()
        
        # For large datasets, use cursor pagination
        if count > 100000:
            return AdvancedCursorPagination
        
        # For mobile clients, use smaller page sizes
        user_agent = request.META.get('HTTP_USER_AGENT', '').lower()
        if 'mobile' in user_agent:
            class MobilePagination(PageNumberPagination):
                page_size = 10
                max_page_size = 25
            return MobilePagination
        
        # Default pagination
        return StandardResultsSetPagination

class DynamicPaginationView(generics.ListAPIView):
    serializer_class = PostSerializer
    
    def get_pagination_class(self):
        """Dynamically choose pagination class"""
        queryset = self.get_queryset()
        return DynamicPagination.get_pagination_class(queryset, self.request)
    
    @property
    def pagination_class(self):
        if not hasattr(self, '_pagination_class'):
            self._pagination_class = self.get_pagination_class()
        return self._pagination_class
```

### Pagination with Aggregations

```python
class AggregatedPagination(PageNumberPagination):
    """Pagination that includes aggregated data"""
    page_size = 20
    
    def get_paginated_response(self, data):
        # Calculate aggregations for the current page
        queryset = self.page.object_list
        
        aggregations = queryset.aggregate(
            total_views=models.Sum('view_count'),
            avg_views=models.Avg('view_count'),
            max_views=models.Max('view_count'),
        )
        
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'aggregations': aggregations,
            'results': data
        })
```

## Notes

### Pagination Types Comparison

| Type | Use Case | Pros | Cons |
|------|----------|------|------|
| **Page Number** | General purpose, user-friendly | Easy to understand, supports jumping to any page | Performance degrades with offset, inconsistent with real-time data |
| **Limit/Offset** | Simple API integration | Flexible, supports random access | Poor performance for large offsets |
| **Cursor** | Large datasets, real-time data | Consistent performance, handles real-time changes | No random access, more complex |
| **Keyset** | Very large datasets | Excellent performance, consistent results | Limited to sequential access |

### Performance Tips

1. **Database Indexes**: Add indexes on ordering fields
2. **Select Related**: Use `select_related()` and `prefetch_related()`
3. **Count Optimization**: Cache or estimate counts for large datasets
4. **Limit Page Size**: Set reasonable `max_page_size` limits
5. **Cursor for Large Data**: Use cursor pagination for datasets > 100k records

### Security Considerations

- **Page Size Limits**: Always set `max_page_size` to prevent abuse
- **Input Validation**: Validate page numbers and cursors
- **Rate Limiting**: Apply rate limits to pagination endpoints
- **Resource Monitoring**: Monitor memory usage for large page sizes

### Common Patterns

1. **Mobile Optimization**: Smaller page sizes for mobile clients
2. **Progressive Loading**: Implement infinite scroll with cursor pagination
3. **Cached Counts**: Cache expensive count queries
4. **Estimated Counts**: Use approximate counts for very large datasets
5. **Hybrid Approaches**: Combine different pagination strategies

This recipe provides comprehensive pagination solutions for any size dataset while maintaining good performance and user experience.
