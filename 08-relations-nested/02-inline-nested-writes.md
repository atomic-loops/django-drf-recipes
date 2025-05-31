# Inline Nested Writes

**Problem:** You need to create or update related objects within a single API request, allowing users to submit nested data that creates both parent and child objects simultaneously.

**Solution:** Implement custom serializer methods to handle nested writes with proper validation and transaction management.

## Basic Nested Write Example

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
    
class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()

class Review(models.Model):
    book = models.ForeignKey(Book, related_name='reviews', on_delete=models.CASCADE)
    reviewer_name = models.CharField(max_length=100)
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
```

```python
# serializers.py
from rest_framework import serializers
from django.db import transaction
from .models import Author, Book, Review

class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = Author
        fields = ['id', 'name', 'email']

class ReviewSerializer(serializers.ModelSerializer):
    class Meta:
        model = Review
        fields = ['id', 'reviewer_name', 'rating', 'comment']

class BookWithNestedSerializer(serializers.ModelSerializer):
    author = AuthorSerializer()
    reviews = ReviewSerializer(many=True, required=False)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'isbn', 'published_date', 'reviews']
    
    @transaction.atomic
    def create(self, validated_data):
        author_data = validated_data.pop('author')
        reviews_data = validated_data.pop('reviews', [])
        
        # Create or get author
        author, created = Author.objects.get_or_create(
            email=author_data['email'],
            defaults=author_data
        )
        
        # Create book
        book = Book.objects.create(author=author, **validated_data)
        
        # Create reviews
        for review_data in reviews_data:
            Review.objects.create(book=book, **review_data)
        
        return book
    
    @transaction.atomic
    def update(self, instance, validated_data):
        author_data = validated_data.pop('author', None)
        reviews_data = validated_data.pop('reviews', None)
        
        # Update book fields
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        
        # Update author if provided
        if author_data:
            author_serializer = AuthorSerializer(
                instance.author, 
                data=author_data, 
                partial=True
            )
            if author_serializer.is_valid(raise_exception=True):
                author_serializer.save()
        
        # Update reviews if provided
        if reviews_data is not None:
            # Clear existing reviews and create new ones
            instance.reviews.all().delete()
            for review_data in reviews_data:
                Review.objects.create(book=instance, **review_data)
        
        return instance
```

## Handling Many-to-Many Nested Writes

```python
# models.py (additional)
class Category(models.Model):
    name = models.CharField(max_length=50, unique=True)
    
class Tag(models.Model):
    name = models.CharField(max_length=30, unique=True)

# Update Book model
class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()
    categories = models.ManyToManyField(Category, blank=True)
    tags = models.ManyToManyField(Tag, blank=True)
```

```python
# serializers.py (updated)
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class BookWithManyToManySerializer(serializers.ModelSerializer):
    author = AuthorSerializer()
    categories = CategorySerializer(many=True, required=False)
    tags = TagSerializer(many=True, required=False)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'isbn', 'published_date', 'categories', 'tags']
    
    @transaction.atomic
    def create(self, validated_data):
        author_data = validated_data.pop('author')
        categories_data = validated_data.pop('categories', [])
        tags_data = validated_data.pop('tags', [])
        
        # Create or get author
        author, created = Author.objects.get_or_create(
            email=author_data['email'],
            defaults=author_data
        )
        
        # Create book
        book = Book.objects.create(author=author, **validated_data)
        
        # Handle categories
        self._set_many_to_many_field(book, 'categories', categories_data, Category)
        
        # Handle tags
        self._set_many_to_many_field(book, 'tags', tags_data, Tag)
        
        return book
    
    def _set_many_to_many_field(self, instance, field_name, data_list, model_class):
        """Helper method to handle many-to-many field creation"""
        objects = []
        for item_data in data_list:
            if 'id' in item_data:
                # Existing object
                obj = model_class.objects.get(id=item_data['id'])
            else:
                # Create new object
                obj, created = model_class.objects.get_or_create(
                    name=item_data['name']
                )
            objects.append(obj)
        
        field = getattr(instance, field_name)
        field.set(objects)
```

## Advanced Nested Write with Validation

```python
class BookAdvancedSerializer(serializers.ModelSerializer):
    author = AuthorSerializer()
    reviews = ReviewSerializer(many=True, required=False)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'isbn', 'published_date', 'reviews']
    
    def validate(self, data):
        """Cross-field validation"""
        # Ensure at least one review if book is published this year
        if (data.get('published_date') and 
            data['published_date'].year == timezone.now().year and
            not data.get('reviews')):
            raise serializers.ValidationError(
                "Books published this year must have at least one review"
            )
        return data
    
    def validate_reviews(self, reviews):
        """Validate reviews data"""
        if len(reviews) > 10:
            raise serializers.ValidationError(
                "Cannot add more than 10 reviews at once"
            )
        
        # Check for duplicate reviewer names
        reviewer_names = [review['reviewer_name'] for review in reviews]
        if len(reviewer_names) != len(set(reviewer_names)):
            raise serializers.ValidationError(
                "Duplicate reviewer names not allowed"
            )
        
        return reviews
    
    @transaction.atomic
    def create(self, validated_data):
        try:
            return super().create(validated_data)
        except Exception as e:
            # Log the error for debugging
            import logging
            logger = logging.getLogger(__name__)
            logger.error(f"Failed to create book with nested data: {e}")
            raise serializers.ValidationError(
                "Failed to create book. Please check your data."
            )
```

## Usage in Views

```python
# views.py
from rest_framework import viewsets, status
from rest_framework.response import Response

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.select_related('author').prefetch_related('reviews', 'categories', 'tags')
    serializer_class = BookWithNestedSerializer
    
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        try:
            book = serializer.save()
            return Response(
                self.get_serializer(book).data, 
                status=status.HTTP_201_CREATED
            )
        except Exception as e:
            return Response(
                {'error': 'Failed to create book with nested data'},
                status=status.HTTP_400_BAD_REQUEST
            )
```

## Example API Usage

```bash
# POST /api/books/
{
    "title": "Django Best Practices",
    "isbn": "9781234567890",
    "published_date": "2024-01-15",
    "author": {
        "name": "John Doe",
        "email": "john@example.com"
    },
    "reviews": [
        {
            "reviewer_name": "Alice Smith",
            "rating": 5,
            "comment": "Excellent book!"
        },
        {
            "reviewer_name": "Bob Johnson",
            "rating": 4,
            "comment": "Very helpful"
        }
    ],
    "categories": [
        {"name": "Programming"},
        {"name": "Web Development"}
    ],
    "tags": [
        {"name": "python"},
        {"name": "django"}
    ]
}
```

**Notes:**
- Always use database transactions for nested writes to ensure data consistency
- Implement proper validation at both field and object levels
- Consider performance implications when creating multiple related objects
- Use `select_related` and `prefetch_related` for efficient querying
- Handle errors gracefully and provide meaningful error messages
- Be careful with unique constraints on nested objects
