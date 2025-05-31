# Filtering with django-filter

## Problem

You need to add flexible filtering capabilities to your DRF API endpoints, allowing users to filter results based on model fields with various lookup types (exact, contains, greater than, etc.).

## Solution

Use the `django-filter` package to create declarative filters that integrate seamlessly with DRF's filtering system. This provides a clean, reusable way to handle complex filtering scenarios.

## Installation

```bash
pip install django-filter
```

Add to your Django settings:

```python
# settings.py
INSTALLED_APPS = [
    # ... other apps
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

## Basic FilterSet

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=100)
    
    def __str__(self):
        return self.name

class Product(models.Model):
    CONDITION_CHOICES = [
        ('new', 'New'),
        ('used', 'Used'),
        ('refurbished', 'Refurbished'),
    ]
    
    name = models.CharField(max_length=200)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    condition = models.CharField(max_length=20, choices=CONDITION_CHOICES)
    is_available = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
    
    def __str__(self):
        return self.name
```

```python
# filters.py
import django_filters
from django_filters import rest_framework as filters
from .models import Product, Category

class ProductFilter(filters.FilterSet):
    # Exact match filters
    category = filters.ModelChoiceFilter(queryset=Category.objects.all())
    condition = filters.ChoiceFilter(choices=Product.CONDITION_CHOICES)
    is_available = filters.BooleanFilter()
    
    # Range filters
    price_min = filters.NumberFilter(field_name='price', lookup_expr='gte')
    price_max = filters.NumberFilter(field_name='price', lookup_expr='lte')
    price_range = filters.RangeFilter(field_name='price')
    
    # Text search filters
    name = filters.CharFilter(lookup_expr='icontains')
    description_contains = filters.CharFilter(
        field_name='description', 
        lookup_expr='icontains'
    )
    
    # Date filters
    created_after = filters.DateTimeFilter(
        field_name='created_at', 
        lookup_expr='gte'
    )
    created_before = filters.DateTimeFilter(
        field_name='created_at', 
        lookup_expr='lte'
    )
    created_date = filters.DateFilter(field_name='created_at__date')
    
    # Custom method filters
    owner = filters.CharFilter(method='filter_by_owner')
    
    class Meta:
        model = Product
        fields = {
            'price': ['exact', 'lt', 'gt', 'gte', 'lte'],
            'created_at': ['exact', 'year', 'month', 'day'],
        }
    
    def filter_by_owner(self, queryset, name, value):
        """Custom filter method to search by username"""
        return queryset.filter(created_by__username__icontains=value)
```

```python
# serializers.py
from rest_framework import serializers
from .models import Product, Category

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class ProductSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    created_by = serializers.StringRelatedField()
    
    class Meta:
        model = Product
        fields = [
            'id', 'name', 'description', 'price', 'category', 
            'condition', 'is_available', 'created_at', 'created_by'
        ]
```

```python
# views.py
from rest_framework import generics
from django_filters.rest_framework import DjangoFilterBackend
from .models import Product
from .serializers import ProductSerializer
from .filters import ProductFilter

class ProductListView(generics.ListAPIView):
    queryset = Product.objects.select_related('category', 'created_by')
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = ProductFilter
    
    # Alternative: Use filterset_fields for simple filtering
    # filterset_fields = ['category', 'condition', 'is_available']
```

## Advanced FilterSet Examples

### Multiple Field Lookup

```python
class AdvancedProductFilter(filters.FilterSet):
    # Search across multiple fields
    search = filters.CharFilter(method='filter_search')
    
    # Multiple choice filter
    categories = filters.ModelMultipleChoiceFilter(
        field_name='category',
        queryset=Category.objects.all()
    )
    
    # Date range with custom method
    date_range = filters.DateFromToRangeFilter(field_name='created_at')
    
    class Meta:
        model = Product
        fields = []
    
    def filter_search(self, queryset, name, value):
        """Search across name and description"""
        return queryset.filter(
            models.Q(name__icontains=value) | 
            models.Q(description__icontains=value)
        )
```

### Custom Widget and Form Field

```python
from django import forms

class ProductFilterWithCustomWidget(filters.FilterSet):
    price_range = filters.RangeFilter(
        field_name='price',
        widget=filters.widgets.RangeWidget(attrs={'class': 'form-control'})
    )
    
    created_year = filters.NumberFilter(
        field_name='created_at__year',
        widget=forms.NumberInput(attrs={'class': 'form-control'})
    )
    
    class Meta:
        model = Product
        fields = []
```

## URL Examples

With the filter setup above, your API will accept these URL patterns:

```
# Basic filtering
GET /api/products/?category=1
GET /api/products/?condition=new
GET /api/products/?is_available=true

# Price filtering
GET /api/products/?price_min=100&price_max=500
GET /api/products/?price_range_after=100&price_range_before=500
GET /api/products/?price__gte=100

# Text search
GET /api/products/?name=laptop
GET /api/products/?description_contains=wireless

# Date filtering
GET /api/products/?created_after=2024-01-01
GET /api/products/?created_date=2024-01-15
GET /api/products/?created_at__year=2024

# Custom filters
GET /api/products/?owner=john
GET /api/products/?search=gaming laptop

# Multiple categories
GET /api/products/?categories=1,3,5

# Combined filters
GET /api/products/?category=1&condition=new&price_min=100&is_available=true
```

## ViewSet Integration

```python
from rest_framework import viewsets

class ProductViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Product.objects.select_related('category', 'created_by')
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = ProductFilter
    
    def get_queryset(self):
        """Additional queryset customization"""
        queryset = super().get_queryset()
        # Add any additional filtering logic here
        return queryset
```

## Filter Validation

```python
class ValidatedProductFilter(filters.FilterSet):
    price_min = filters.NumberFilter(
        field_name='price', 
        lookup_expr='gte',
        help_text="Minimum price filter"
    )
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
    @property
    def qs(self):
        """Add custom validation"""
        parent = super().qs
        
        price_min = self.form.cleaned_data.get('price_min')
        price_max = self.form.cleaned_data.get('price_max')
        
        if price_min and price_max and price_min > price_max:
            # Return empty queryset for invalid ranges
            return parent.none()
            
        return parent
    
    class Meta:
        model = Product
        fields = []
```

## Notes

### Performance Considerations

- **Select Related**: Always use `select_related()` for foreign keys to avoid N+1 queries
- **Prefetch Related**: Use for many-to-many and reverse foreign key relationships
- **Database Indexes**: Add indexes on commonly filtered fields

```python
# In your model
class Product(models.Model):
    # ... fields ...
    
    class Meta:
        indexes = [
            models.Index(fields=['price']),
            models.Index(fields=['created_at']),
            models.Index(fields=['category', 'condition']),
        ]
```

### Security Tips

- **Validate Input**: Always validate filter inputs to prevent SQL injection
- **Limit Exposed Fields**: Only expose fields that should be filterable
- **Rate Limiting**: Consider rate limiting for expensive filter operations

### Common Patterns

1. **Nested Field Filtering**: Use double underscores for related field filtering
2. **Case-Insensitive Search**: Use `icontains`, `iexact` lookups
3. **Multiple Value Filtering**: Use `ModelMultipleChoiceFilter` or `in` lookup
4. **Custom Logic**: Implement custom filter methods for complex business logic

### Error Handling

```python
# Handle invalid filter values gracefully
class RobustProductFilter(filters.FilterSet):
    def filter_queryset(self, queryset):
        try:
            return super().filter_queryset(queryset)
        except (ValueError, ValidationError):
            # Return empty queryset for invalid filters
            return queryset.none()
```

This recipe provides a comprehensive foundation for implementing flexible, performant filtering in your DRF APIs using django-filter.
