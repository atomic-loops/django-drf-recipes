# Searching with SearchFilter

## Problem

You need to implement full-text search functionality across multiple fields in your API, allowing users to search for content using natural language queries with a single search parameter.

## Solution

Use DRF's built-in `SearchFilter` to provide simple, effective search functionality across multiple model fields. This filter implements case-insensitive partial matching and can search across related model fields.

## Basic Search Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Author(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    bio = models.TextField(blank=True)
    
    def __str__(self):
        return f"{self.first_name} {self.last_name}"

class Publisher(models.Model):
    name = models.CharField(max_length=200)
    website = models.URLField(blank=True)
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=300)
    subtitle = models.CharField(max_length=300, blank=True)
    description = models.TextField()
    isbn = models.CharField(max_length=13, unique=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    publication_date = models.DateField()
    authors = models.ManyToManyField(Author, related_name='books')
    publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
    tags = models.CharField(max_length=500, blank=True, help_text="Comma-separated tags")
    
    def __str__(self):
        return self.title
```

```python
# serializers.py
from rest_framework import serializers
from .models import Book, Author, Publisher

class AuthorSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'first_name', 'last_name', 'full_name', 'bio']
    
    def get_full_name(self, obj):
        return f"{obj.first_name} {obj.last_name}"

class PublisherSerializer(serializers.ModelSerializer):
    class Meta:
        model = Publisher
        fields = ['id', 'name', 'website']

class BookSerializer(serializers.ModelSerializer):
    authors = AuthorSerializer(many=True, read_only=True)
    publisher = PublisherSerializer(read_only=True)
    
    class Meta:
        model = Book
        fields = [
            'id', 'title', 'subtitle', 'description', 'isbn', 
            'price', 'publication_date', 'authors', 'publisher', 'tags'
        ]
```

```python
# views.py
from rest_framework import generics, filters
from .models import Book
from .serializers import BookSerializer

class BookSearchView(generics.ListAPIView):
    queryset = Book.objects.select_related('publisher').prefetch_related('authors')
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = [
        'title', 
        'subtitle', 
        'description',
        'isbn',
        'tags',
        'authors__first_name',
        'authors__last_name', 
        'publisher__name'
    ]
```

## Advanced Search Configurations

### Search Field Lookups

```python
class AdvancedBookSearchView(generics.ListAPIView):
    queryset = Book.objects.select_related('publisher').prefetch_related('authors')
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    
    # Different search behaviors for different fields
    search_fields = [
        '=title',           # Exact match
        '=isbn',            # Exact match for ISBN
        '^title',           # Starts with
        '^subtitle',        # Starts with  
        'description',      # Contains (default)
        'tags',             # Contains
        '@authors__bio',    # Full-text search (PostgreSQL only)
        'authors__first_name',
        'authors__last_name',
        'publisher__name',
    ]
```

### Custom Search Parameter

```python
class CustomSearchView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['title', 'description']
    
    # Change the search parameter name from 'search' to 'q'
    search_param = 'q'
```

### Multiple Search Filters

```python
class MultipleSearchView(generics.ListAPIView):
    queryset = Book.objects.select_related('publisher').prefetch_related('authors')
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    
    def get_search_fields(self):
        """Dynamically determine search fields based on query params"""
        search_type = self.request.query_params.get('search_type', 'all')
        
        if search_type == 'title_only':
            return ['title', 'subtitle']
        elif search_type == 'author':
            return ['authors__first_name', 'authors__last_name']
        elif search_type == 'content':
            return ['description', 'tags']
        else:
            return [
                'title', 'subtitle', 'description', 'tags',
                'authors__first_name', 'authors__last_name', 
                'publisher__name'
            ]
```

## Custom Search Filter

```python
# filters.py
from rest_framework import filters
from django.db import models

class CustomSearchFilter(filters.SearchFilter):
    def filter_queryset(self, request, queryset, view):
        """Custom search logic with additional features"""
        search_terms = self.get_search_terms(request)
        
        if not search_terms:
            return queryset
        
        # Get the original filtered queryset
        queryset = super().filter_queryset(request, queryset, view)
        
        # Add custom logic - e.g., boost exact title matches
        exact_title_matches = queryset.filter(title__iexact=' '.join(search_terms))
        other_matches = queryset.exclude(title__iexact=' '.join(search_terms))
        
        # Return exact matches first, then other matches
        return exact_title_matches.union(other_matches, all=True)
    
    def get_search_terms(self, request):
        """Override to add custom search term processing"""
        params = request.query_params.get(self.search_param, '')
        
        # Custom processing: remove special characters, handle quotes
        params = params.replace('"', '').replace("'", '')
        
        return super().get_search_terms(request)

class WeightedSearchFilter(filters.SearchFilter):
    """Search filter that applies different weights to different fields"""
    
    def filter_queryset(self, request, queryset, view):
        search_terms = self.get_search_terms(request)
        
        if not search_terms:
            return queryset
        
        search_fields = self.get_search_fields(view, request)
        
        if not search_fields:
            return queryset
        
        # Build weighted search
        queries = []
        for search_term in search_terms:
            term_queries = []
            
            for search_field in search_fields:
                # Title gets higher weight
                if 'title' in search_field:
                    term_queries.append(
                        models.Q(**{f"{search_field}__icontains": search_term})
                    )
                    # Add exact match boost for titles
                    term_queries.append(
                        models.Q(**{f"{search_field}__iexact": search_term})
                    )
                else:
                    term_queries.append(
                        models.Q(**{f"{search_field}__icontains": search_term})
                    )
            
            if term_queries:
                queries.append(models.Q(*term_queries, _connector=models.Q.OR))
        
        if queries:
            queryset = queryset.filter(models.Q(*queries, _connector=models.Q.AND))
        
        return queryset.distinct()
```

## Search with Annotations

```python
from django.db.models import Case, When, IntegerField, Q

class SearchWithRankingView(generics.ListAPIView):
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['title', 'description', 'authors__first_name', 'authors__last_name']
    
    def get_queryset(self):
        queryset = Book.objects.select_related('publisher').prefetch_related('authors')
        
        search_query = self.request.query_params.get('search', '')
        
        if search_query:
            # Add relevance scoring
            queryset = queryset.annotate(
                relevance=Case(
                    # Exact title match gets highest score
                    When(title__iexact=search_query, then=100),
                    # Title starts with query gets high score
                    When(title__istartswith=search_query, then=90),
                    # Title contains query gets medium score
                    When(title__icontains=search_query, then=70),
                    # Description contains query gets lower score
                    When(description__icontains=search_query, then=50),
                    # Author name match gets medium score
                    When(
                        Q(authors__first_name__icontains=search_query) |
                        Q(authors__last_name__icontains=search_query),
                        then=60
                    ),
                    default=0,
                    output_field=IntegerField()
                )
            ).order_by('-relevance', 'title')
        
        return queryset
```

## ViewSet Integration

```python
from rest_framework import viewsets

class BookViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Book.objects.select_related('publisher').prefetch_related('authors')
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = [
        'title', 'subtitle', 'description', 'isbn',
        'authors__first_name', 'authors__last_name',
        'publisher__name', 'tags'
    ]
    ordering_fields = ['title', 'publication_date', 'price']
    ordering = ['title']
    
    def get_queryset(self):
        """Add additional search logic if needed"""
        queryset = super().get_queryset()
        
        # Example: Filter by availability
        if self.request.query_params.get('available_only'):
            queryset = queryset.filter(stock_quantity__gt=0)
        
        return queryset
```

## URL Examples

With the search setup, your API accepts these search patterns:

```
# Basic search
GET /api/books/?search=python

# Search for multiple terms (AND logic)
GET /api/books/?search=python programming

# Search with quotes (treated as phrase)
GET /api/books/?search="machine learning"

# Custom search parameter
GET /api/books/?q=django

# Combined with other filters
GET /api/books/?search=python&ordering=price

# Search type filtering
GET /api/books/?search=john&search_type=author
```

## Performance Optimization

```python
# Use database indexes for search fields
class Book(models.Model):
    title = models.CharField(max_length=300, db_index=True)
    description = models.TextField()
    
    class Meta:
        indexes = [
            models.Index(fields=['title']),
            models.Index(fields=['isbn']),
            # Composite indexes for common search combinations
            models.Index(fields=['title', 'publication_date']),
        ]

# For PostgreSQL, consider full-text search indexes
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField

class BookWithFullText(models.Model):
    # ... other fields ...
    search_vector = SearchVectorField(null=True)
    
    class Meta:
        indexes = [
            GinIndex(fields=['search_vector']),
        ]
```

## Advanced PostgreSQL Search

```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank

class PostgreSQLSearchView(generics.ListAPIView):
    serializer_class = BookSerializer
    
    def get_queryset(self):
        queryset = Book.objects.all()
        search_query = self.request.query_params.get('search')
        
        if search_query:
            # Use PostgreSQL full-text search
            search_vector = SearchVector('title', weight='A') + \
                          SearchVector('description', weight='B') + \
                          SearchVector('authors__first_name', 'authors__last_name', weight='C')
            
            search_query_obj = SearchQuery(search_query)
            
            queryset = queryset.annotate(
                search=search_vector,
                rank=SearchRank(search_vector, search_query_obj)
            ).filter(search=search_query_obj).order_by('-rank')
        
        return queryset
```

## Notes

### Search Behavior

- **Case Insensitive**: All searches are case-insensitive by default
- **Partial Matching**: Uses `icontains` lookup by default
- **Multiple Terms**: Multiple search terms are combined with AND logic
- **Related Fields**: Can search across foreign key and many-to-many relationships

### Lookup Prefixes

- `^` - Starts with search (`istartswith`)
- `=` - Exact match (`iexact`) 
- `@` - Full-text search (PostgreSQL only)
- No prefix - Contains search (`icontains`)

### Performance Tips

1. **Database Indexes**: Add indexes on frequently searched fields
2. **Select/Prefetch Related**: Always optimize querysets for related field searches
3. **Limit Search Fields**: Don't expose too many searchable fields
4. **Search Term Length**: Consider minimum search term length requirements

### Security Considerations

- Search terms are automatically escaped to prevent SQL injection
- Consider rate limiting for search endpoints
- Monitor for expensive search queries

### Common Patterns

1. **Autocomplete**: Use `^` prefix for starts-with behavior
2. **Exact Match**: Use `=` prefix for exact matching (IDs, codes)
3. **Multi-field**: Search across logical field groups
4. **Weighted Results**: Implement custom ranking for better relevance

This recipe provides comprehensive search functionality that can be easily integrated into any DRF API.
