# Ordering Results

## Problem

You need to allow API users to sort and order query results by different fields, providing flexible sorting options including multiple fields, ascending/descending order, and custom sorting logic.

## Solution

Use DRF's built-in `OrderingFilter` to provide URL-based result ordering. This filter allows users to specify which fields to sort by and the sort direction through query parameters.

## Basic Ordering Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=100)
    priority = models.IntegerField(default=0)
    
    def __str__(self):
        return self.name

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock_quantity = models.PositiveIntegerField(default=0)
    rating = models.DecimalField(max_digits=3, decimal_places=2, default=0.0)
    review_count = models.PositiveIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
    is_featured = models.BooleanField(default=False)
    
    class Meta:
        ordering = ['-created_at']  # Default ordering
    
    def __str__(self):
        return self.name
```

```python
# serializers.py
from rest_framework import serializers
from .models import Product, Category

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'priority']

class ProductSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    created_by = serializers.StringRelatedField()
    
    class Meta:
        model = Product
        fields = [
            'id', 'name', 'price', 'stock_quantity', 'rating', 
            'review_count', 'created_at', 'updated_at', 'category', 
            'created_by', 'is_featured'
        ]
```

```python
# views.py
from rest_framework import generics, filters
from .models import Product
from .serializers import ProductSerializer

class ProductListView(generics.ListAPIView):
    queryset = Product.objects.select_related('category', 'created_by')
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = [
        'name', 'price', 'stock_quantity', 'rating', 
        'review_count', 'created_at', 'updated_at'
    ]
    ordering = ['-created_at']  # Default ordering
```

## Advanced Ordering Options

### Custom Ordering Fields

```python
class AdvancedProductListView(generics.ListAPIView):
    queryset = Product.objects.select_related('category', 'created_by')
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    
    # Include related fields and computed fields
    ordering_fields = [
        'name', 'price', 'rating', 'review_count',
        'created_at', 'updated_at', 'stock_quantity',
        'category__name',           # Related field
        'category__priority',       # Related field
        'created_by__username',     # Related field
        'popularity',               # Custom computed field
        'value_score',              # Custom computed field
    ]
    
    # Multiple default ordering
    ordering = ['-is_featured', '-rating', 'name']
    
    def get_queryset(self):
        """Add computed annotations for ordering"""
        from django.db.models import Case, When, F, Value, DecimalField
        from django.db.models.functions import Coalesce
        
        queryset = super().get_queryset()
        
        # Add computed fields for ordering
        queryset = queryset.annotate(
            # Popularity score based on rating and review count
            popularity=Case(
                When(review_count=0, then=Value(0)),
                default=F('rating') * F('review_count'),
                output_field=DecimalField(max_digits=10, decimal_places=2)
            ),
            # Value score: rating / price ratio
            value_score=Case(
                When(price=0, then=Value(0)),
                default=F('rating') / F('price'),
                output_field=DecimalField(max_digits=10, decimal_places=4)
            ),
            # Availability score
            availability_score=Case(
                When(stock_quantity=0, then=Value(0)),
                When(stock_quantity__lte=5, then=Value(1)),
                When(stock_quantity__lte=20, then=Value(2)),
                default=Value(3),
                output_field=models.IntegerField()
            )
        )
        
        return queryset
```

### Custom Ordering Filter

```python
# filters.py
from rest_framework import filters
from django.db.models import Case, When, F, Value

class CustomOrderingFilter(filters.OrderingFilter):
    """Custom ordering filter with additional logic"""
    
    def get_ordering(self, request, queryset, view):
        """Override to add custom ordering logic"""
        ordering = super().get_ordering(request, queryset, view)
        
        # Always put featured items first if no explicit ordering
        if not ordering:
            ordering = ['-is_featured', '-created_at']
        elif ordering and '-is_featured' not in ordering and 'is_featured' not in ordering:
            # Prepend featured ordering to user's choice
            ordering = ['-is_featured'] + list(ordering)
        
        return ordering
    
    def filter_queryset(self, request, queryset, view):
        """Add custom ordering annotations before filtering"""
        # Add smart ordering annotations
        ordering_param = request.query_params.get(self.ordering_param)
        
        if ordering_param:
            if 'smart_price' in ordering_param:
                # Smart price considers discounts, shipping, etc.
                queryset = queryset.annotate(
                    smart_price=F('price') * Case(
                        When(is_featured=True, then=Value(0.9)),  # 10% featured discount
                        default=Value(1.0)
                    )
                )
            
            if 'relevance' in ordering_param:
                # Custom relevance scoring
                search_term = request.query_params.get('search', '')
                if search_term:
                    queryset = queryset.annotate(
                        relevance=Case(
                            When(name__icontains=search_term, then=Value(3)),
                            When(category__name__icontains=search_term, then=Value(2)),
                            default=Value(1)
                        )
                    )
        
        return super().filter_queryset(request, queryset, view)

class SmartOrderingFilter(filters.OrderingFilter):
    """Ordering filter with business logic"""
    
    def get_valid_fields(self, queryset, view, context={}):
        """Add dynamic ordering fields"""
        valid_fields = super().get_valid_fields(queryset, view, context)
        
        # Add computed fields to valid fields
        computed_fields = [
            ('popularity', 'popularity'),
            ('value_score', 'value_score'),
            ('smart_price', 'smart_price'),
            ('availability_score', 'availability_score'),
        ]
        
        valid_fields.extend(computed_fields)
        return valid_fields
```

### ViewSet with Multiple Ordering Options

```python
from rest_framework import viewsets

class ProductViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = ProductSerializer
    filter_backends = [
        filters.SearchFilter,
        filters.OrderingFilter,
        CustomOrderingFilter
    ]
    search_fields = ['name', 'category__name']
    ordering_fields = [
        'name', 'price', 'rating', 'review_count', 
        'created_at', 'stock_quantity', 'popularity', 
        'value_score', 'category__name'
    ]
    ordering = ['-is_featured', '-created_at']
    
    def get_queryset(self):
        """Dynamic queryset with ordering optimizations"""
        queryset = Product.objects.select_related(
            'category', 'created_by'
        ).annotate(
            popularity=F('rating') * F('review_count'),
            value_score=Case(
                When(price=0, then=Value(0)),
                default=F('rating') / F('price')
            )
        )
        
        # Optimize based on ordering
        ordering_param = self.request.query_params.get('ordering', '')
        
        if 'category' in ordering_param:
            queryset = queryset.select_related('category')
        
        if 'created_by' in ordering_param:
            queryset = queryset.select_related('created_by')
        
        return queryset
    
    def list(self, request, *args, **kwargs):
        """Add ordering metadata to response"""
        response = super().list(request, *args, **kwargs)
        
        # Add available ordering options to response
        response.data['ordering_options'] = {
            'price': 'Price (Low to High: price, High to Low: -price)',
            'rating': 'Rating (Best: -rating, Worst: rating)',
            'popularity': 'Popularity (Most: -popularity)',
            'created_at': 'Date Added (Newest: -created_at, Oldest: created_at)',
            'name': 'Name (A-Z: name, Z-A: -name)',
            'stock_quantity': 'Stock (Most: -stock_quantity, Least: stock_quantity)'
        }
        
        return response
```

## URL Examples

With the ordering setup, your API accepts these URL patterns:

```
# Single field ordering
GET /api/products/?ordering=price              # Ascending by price
GET /api/products/?ordering=-price             # Descending by price
GET /api/products/?ordering=name               # Ascending by name
GET /api/products/?ordering=-created_at        # Descending by creation date

# Multiple field ordering (comma-separated)
GET /api/products/?ordering=-is_featured,price # Featured first, then by price
GET /api/products/?ordering=-rating,-review_count # By rating, then review count
GET /api/products/?ordering=category__name,name   # By category, then name

# Related field ordering
GET /api/products/?ordering=category__name     # By category name
GET /api/products/?ordering=-category__priority # By category priority
GET /api/products/?ordering=created_by__username # By creator username

# Custom computed field ordering
GET /api/products/?ordering=-popularity        # By popularity score
GET /api/products/?ordering=-value_score       # By value score

# Combined with search and filters
GET /api/products/?search=laptop&ordering=-rating
GET /api/products/?category=electronics&ordering=price
```

## Complex Ordering Scenarios

### Conditional Ordering

```python
class ConditionalOrderingView(generics.ListAPIView):
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    
    def get_ordering(self):
        """Dynamic ordering based on user preferences or context"""
        user = self.request.user
        ordering_param = self.request.query_params.get('ordering')
        
        if ordering_param:
            return ordering_param.split(',')
        
        # Default ordering based on user type
        if user.is_authenticated:
            if hasattr(user, 'profile') and user.profile.is_premium:
                # Premium users see featured items first
                return ['-is_featured', '-rating', 'price']
            else:
                # Regular users see best value first
                return ['-value_score', '-rating']
        else:
            # Anonymous users see popular items
            return ['-popularity', '-review_count']
    
    def get_queryset(self):
        queryset = Product.objects.select_related('category')
        
        # Add required annotations based on potential ordering
        queryset = queryset.annotate(
            popularity=F('rating') * F('review_count'),
            value_score=F('rating') / F('price')
        )
        
        return queryset
```

### Ordering with Pagination

```python
from rest_framework.pagination import PageNumberPagination

class ProductPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class PaginatedProductView(generics.ListAPIView):
    queryset = Product.objects.select_related('category')
    serializer_class = ProductSerializer
    pagination_class = ProductPagination
    filter_backends = [filters.OrderingFilter, filters.SearchFilter]
    ordering_fields = ['name', 'price', 'rating', 'created_at']
    ordering = ['-created_at']
    search_fields = ['name', 'category__name']
    
    def get_queryset(self):
        """Optimize queryset for ordering and pagination"""
        queryset = super().get_queryset()
        
        # Add indexes hint for common ordering patterns
        ordering = self.request.query_params.get('ordering', '')
        
        if 'price' in ordering:
            queryset = queryset.order_by()  # Clear default ordering first
        
        return queryset
```

## Custom Ordering Logic

### Business-Specific Ordering

```python
class BusinessLogicOrderingView(generics.ListAPIView):
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    
    def get_queryset(self):
        """Apply business-specific ordering logic"""
        from django.db.models import Case, When, Q, F
        
        queryset = Product.objects.select_related('category')
        
        # Business logic: prioritize in-stock, highly-rated, recently-updated items
        queryset = queryset.annotate(
            business_priority=Case(
                # Out of stock items go to bottom
                When(stock_quantity=0, then=Value(0)),
                # New items with good ratings get priority
                When(
                    Q(created_at__gte=timezone.now() - timedelta(days=30)) &
                    Q(rating__gte=4.0),
                    then=Value(5)
                ),
                # Featured items
                When(is_featured=True, then=Value(4)),
                # High-rated items
                When(rating__gte=4.0, then=Value(3)),
                # Regular items with stock
                When(stock_quantity__gt=0, then=Value(2)),
                # Everything else
                default=Value(1)
            ),
            # Smart score combining multiple factors
            smart_score=F('business_priority') * F('rating') * F('review_count')
        )
        
        return queryset
    
    ordering_fields = [
        'name', 'price', 'rating', 'business_priority', 
        'smart_score', 'stock_quantity'
    ]
    ordering = ['-business_priority', '-smart_score']
```

## Performance Optimization

### Database Indexes

```python
# models.py - Add indexes for common ordering patterns
class Product(models.Model):
    # ... fields ...
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            # Single field indexes
            models.Index(fields=['price']),
            models.Index(fields=['rating']),
            models.Index(fields=['created_at']),
            models.Index(fields=['stock_quantity']),
            
            # Composite indexes for common ordering combinations
            models.Index(fields=['-is_featured', '-rating']),
            models.Index(fields=['-created_at', 'price']),
            models.Index(fields=['category', 'price']),
            models.Index(fields=['category', '-rating']),
            
            # Conditional indexes (PostgreSQL)
            models.Index(
                fields=['price'], 
                name='active_products_price_idx',
                condition=models.Q(stock_quantity__gt=0)
            ),
        ]
```

### Query Optimization

```python
class OptimizedOrderingView(generics.ListAPIView):
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    
    def get_queryset(self):
        """Optimize queryset based on ordering requirements"""
        queryset = Product.objects.all()
        
        ordering = self.request.query_params.get('ordering', '')
        
        # Optimize select_related based on ordering fields
        if any(field in ordering for field in ['category__name', 'category__priority']):
            queryset = queryset.select_related('category')
        
        if 'created_by__username' in ordering:
            queryset = queryset.select_related('created_by')
        
        # Only add expensive annotations when needed
        if any(field in ordering for field in ['popularity', 'value_score']):
            queryset = queryset.annotate(
                popularity=F('rating') * F('review_count'),
                value_score=F('rating') / F('price')
            )
        
        return queryset
```

## Notes

### Ordering Behavior

- **Multiple Fields**: Use comma-separated values for multiple field ordering
- **Direction**: Prefix with `-` for descending order (default is ascending)
- **Related Fields**: Use double underscores for related field access
- **Case Sensitivity**: Field names are case-sensitive

### Security Considerations

- **Field Validation**: Only allow ordering by whitelisted fields
- **SQL Injection**: DRF automatically prevents injection through field validation
- **Performance**: Monitor expensive ordering operations
- **Rate Limiting**: Consider limiting complex ordering queries

### Performance Tips

1. **Database Indexes**: Add indexes for commonly ordered fields
2. **Composite Indexes**: Create indexes for multi-field ordering patterns
3. **Select Related**: Optimize queries when ordering by related fields
4. **Annotations**: Only add computed fields when needed
5. **Pagination**: Always use pagination with ordering for large datasets

### Common Patterns

1. **Default Ordering**: Always provide sensible default ordering
2. **Business Priority**: Implement business-logic-based ordering
3. **User Preferences**: Allow users to save ordering preferences
4. **Smart Defaults**: Use different defaults based on context
5. **Relevance Ordering**: Combine ordering with search relevance

This recipe provides comprehensive ordering functionality that can handle simple to complex sorting requirements in your DRF APIs.
