# SerializerMethodField

## Problem

You need to include custom fields in your serialized data that don't exist in your model or require custom logic to compute.

## Solution

Use DRF's `SerializerMethodField` to add custom, read-only fields to your serializers based on methods in your serializer class.

## Code

### 1. Basic Usage

Here's a simple example of using `SerializerMethodField` to add a custom field:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.CharField(max_length=255)
    publication_date = models.DateField()
    price = models.DecimalField(max_digits=6, decimal_places=2)
    
    def __str__(self):
        return self.title

# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    # Add a custom field that doesn't exist in the model
    is_new_release = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'price', 'is_new_release']
    
    def get_is_new_release(self, obj):
        """
        Return True if the book was published within the last 30 days.
        The method name must match the pattern 'get_<field_name>'.
        """
        import datetime
        today = datetime.date.today()
        days_since_publication = (today - obj.publication_date).days
        return days_since_publication <= 30
```

### 2. Computing Values Based on Multiple Fields

You can use `SerializerMethodField` to calculate values based on multiple fields:

```python
# models.py
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    discount = models.DecimalField(max_digits=4, decimal_places=2, default=0)  # Percentage
    
    def __str__(self):
        return self.name

# serializers.py
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    final_price = serializers.SerializerMethodField()
    savings = serializers.SerializerMethodField()
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'discount', 'final_price', 'savings']
    
    def get_final_price(self, obj):
        """Calculate the final price after discount."""
        discount_amount = obj.price * (obj.discount / 100)
        return obj.price - discount_amount
    
    def get_savings(self, obj):
        """Calculate the amount saved due to discount."""
        return obj.price * (obj.discount / 100)
```

### 3. Using Context Data

You can access the serializer's context to use request information or other contextual data:

```python
# views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer

class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_serializer_context(self):
        """Add user to serializer context."""
        context = super().get_serializer_context()
        context['user'] = self.request.user
        return context

# serializers.py
from rest_framework import serializers
from .models import Book, UserBookStatus

class BookSerializer(serializers.ModelSerializer):
    user_has_purchased = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'price', 'user_has_purchased']
    
    def get_user_has_purchased(self, obj):
        """
        Check if the current user has purchased this book.
        Requires 'user' to be passed in the context.
        """
        user = self.context.get('user')
        if user and user.is_authenticated:
            return UserBookStatus.objects.filter(user=user, book=obj, purchased=True).exists()
        return False
```

### 4. Accessing Related Objects

Use `SerializerMethodField` to include data from related objects without the overhead of nested serializers:

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Author(models.Model):
    name = models.CharField(max_length=100)
    bio = models.TextField(blank=True)
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    publication_date = models.DateField()
    
    def __str__(self):
        return self.title

class Review(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    rating = models.IntegerField()
    comment = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"Review for {self.book.title} by {self.user.username}"

# serializers.py
from rest_framework import serializers
from django.db.models import Avg, Count
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    author_name = serializers.ReadOnlyField(source='author.name')
    review_stats = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author_name', 'publication_date', 'review_stats']
    
    def get_review_stats(self, obj):
        """
        Calculate and return review statistics.
        """
        stats = obj.reviews.aggregate(
            average_rating=Avg('rating'),
            count=Count('id')
        )
        return {
            'average_rating': stats['average_rating'] or 0,
            'count': stats['count'],
            'has_reviews': stats['count'] > 0
        }
```

### 5. Customizing Output Format

Use `SerializerMethodField` to format data in a specific way:

```python
# models.py
from django.db import models

class Event(models.Model):
    title = models.CharField(max_length=255)
    start_time = models.DateTimeField()
    end_time = models.DateTimeField()
    location = models.CharField(max_length=255)
    
    def __str__(self):
        return self.title

# serializers.py
from rest_framework import serializers
from .models import Event

class EventSerializer(serializers.ModelSerializer):
    formatted_date = serializers.SerializerMethodField()
    duration = serializers.SerializerMethodField()
    
    class Meta:
        model = Event
        fields = ['id', 'title', 'start_time', 'end_time', 'location', 'formatted_date', 'duration']
    
    def get_formatted_date(self, obj):
        """
        Return the date in a user-friendly format.
        """
        return obj.start_time.strftime("%A, %B %d, %Y at %I:%M %p")
    
    def get_duration(self, obj):
        """
        Calculate and return the event duration in hours and minutes.
        """
        duration = obj.end_time - obj.start_time
        total_seconds = duration.total_seconds()
        hours = int(total_seconds // 3600)
        minutes = int((total_seconds % 3600) // 60)
        
        if hours == 0:
            return f"{minutes} minutes"
        elif minutes == 0:
            return f"{hours} hours"
        else:
            return f"{hours} hours and {minutes} minutes"
```

### 6. Conditionally Including Fields

Use `SerializerMethodField` to conditionally include or format fields:

```python
# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    price_display = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'price', 'price_display']
    
    def get_price_display(self, obj):
        """
        Return the price in different formats based on the context.
        """
        format_type = self.context.get('price_format', 'standard')
        
        if format_type == 'standard':
            return f"${obj.price:.2f}"
        elif format_type == 'rounded':
            return f"${round(obj.price)}"
        elif format_type == 'with_tax':
            tax_rate = self.context.get('tax_rate', 0.1)  # Default 10%
            with_tax = obj.price * (1 + tax_rate)
            return f"${with_tax:.2f} (inc. tax)"
        return f"${obj.price:.2f}"
```

### 7. Using SerializerMethodField with Annotations

Combine `SerializerMethodField` with Django's query annotations for optimized performance:

```python
# views.py
from django.db.models import Count, Avg
from rest_framework import viewsets
from .models import Author
from .serializers import AuthorSerializer

class AuthorViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = AuthorSerializer
    
    def get_queryset(self):
        return Author.objects.annotate(
            book_count=Count('books'),
            avg_book_rating=Avg('books__reviews__rating')
        )

# serializers.py
from rest_framework import serializers
from .models import Author

class AuthorSerializer(serializers.ModelSerializer):
    stats = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'bio', 'stats']
    
    def get_stats(self, obj):
        """
        Return statistics from annotations.
        This avoids N+1 query issues since the values are computed in the queryset.
        """
        return {
            'total_books': getattr(obj, 'book_count', 0),
            'average_rating': getattr(obj, 'avg_book_rating', None)
        }
```

## Notes

1. **Performance Considerations**:
   - Method fields can cause N+1 query problems if not careful
   - Use `select_related()` and `prefetch_related()` in your views
   - Consider using annotations in the queryset for computed values
   - For complex calculations on large datasets, consider pre-computing values

2. **Read-Only Fields**:
   - `SerializerMethodField` is always read-only
   - For writeable custom fields, you'll need to override `create()` and `update()` methods

3. **Method Naming Convention**:
   - The method must follow the pattern `get_<field_name>`
   - If you need a different method name, you can specify it:
   ```python
   calculated_value = serializers.SerializerMethodField(method_name='calculate_value')
   
   def calculate_value(self, obj):
       # Your custom calculation
       return value
   ```

4. **Accessing Request Data**:
   - Request object is available in `self.context['request']`
   - Useful for user-specific calculations or permission checks

5. **Error Handling**:
   - Include proper error handling in your methods
   - Return a default value if data is missing or calculations fail

6. **Documentation**:
   - Document your method fields since their behavior isn't obvious from the model
   - Include details on what the field represents and how it's calculated

7. **Testing**:
   - Write unit tests specifically for your serializer methods
   - Test edge cases and default values
