# Writable Nested Serializers

## Problem

You need to create, update, or delete related objects through nested serializers in a single API call, but by default, DRF's nested serializers are read-only.

## Solution

Implement custom `create()` and `update()` methods in your serializer to handle creating, updating, and deleting nested related objects.

## Code

### 1. Basic Model Structure

Let's work with a typical set of related models:

```python
# models.py
from django.db import models

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

class Chapter(models.Model):
    book = models.ForeignKey(Book, on_delete=models.CASCADE, related_name='chapters')
    title = models.CharField(max_length=255)
    order = models.PositiveIntegerField()
    content = models.TextField()
    
    class Meta:
        ordering = ['order']
    
    def __str__(self):
        return f"{self.book.title} - Chapter {self.order}: {self.title}"
```

### 2. One-to-One Nested Serializer

Let's implement a writable nested serializer for a one-to-one relationship:

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book, BookMetadata

class BookMetadataSerializer(serializers.ModelSerializer):
    class Meta:
        model = BookMetadata
        fields = ['isbn', 'page_count', 'language', 'publisher']

class BookWithMetadataSerializer(serializers.ModelSerializer):
    metadata = BookMetadataSerializer()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'metadata']
    
    def create(self, validated_data):
        metadata_data = validated_data.pop('metadata')
        book = Book.objects.create(**validated_data)
        BookMetadata.objects.create(book=book, **metadata_data)
        return book
    
    def update(self, instance, validated_data):
        metadata_data = validated_data.pop('metadata', None)
        
        # Update book fields
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.publication_date = validated_data.get('publication_date', instance.publication_date)
        instance.save()
        
        # Update metadata fields
        if metadata_data and hasattr(instance, 'metadata'):
            metadata = instance.metadata
            metadata.isbn = metadata_data.get('isbn', metadata.isbn)
            metadata.page_count = metadata_data.get('page_count', metadata.page_count)
            metadata.language = metadata_data.get('language', metadata.language)
            metadata.publisher = metadata_data.get('publisher', metadata.publisher)
            metadata.save()
        elif metadata_data:
            BookMetadata.objects.create(book=instance, **metadata_data)
            
        return instance
```

### 3. One-to-Many Nested Serializer

Now let's implement a serializer for a one-to-many relationship (Book with Chapters):

```python
# serializers.py
from rest_framework import serializers
from .models import Book, Chapter

class ChapterSerializer(serializers.ModelSerializer):
    class Meta:
        model = Chapter
        fields = ['id', 'title', 'order', 'content']

class BookWithChaptersSerializer(serializers.ModelSerializer):
    chapters = ChapterSerializer(many=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'chapters']
    
    def create(self, validated_data):
        chapters_data = validated_data.pop('chapters')
        book = Book.objects.create(**validated_data)
        
        for chapter_data in chapters_data:
            Chapter.objects.create(book=book, **chapter_data)
            
        return book
    
    def update(self, instance, validated_data):
        chapters_data = validated_data.pop('chapters', None)
        
        # Update book fields
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.publication_date = validated_data.get('publication_date', instance.publication_date)
        instance.save()
        
        # Update chapters
        if chapters_data is not None:
            # Option 1: Delete and recreate all chapters
            instance.chapters.all().delete()
            for chapter_data in chapters_data:
                Chapter.objects.create(book=instance, **chapter_data)
                
        return instance
```

### 4. Advanced: Updating Existing Children by ID

A more sophisticated approach would be to update existing children if they have an ID, or create them if they don't:

```python
# serializers.py
from rest_framework import serializers
from .models import Book, Chapter

class ChapterSerializer(serializers.ModelSerializer):
    class Meta:
        model = Chapter
        fields = ['id', 'title', 'order', 'content']

class BookWithChaptersSerializer(serializers.ModelSerializer):
    chapters = ChapterSerializer(many=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'chapters']
    
    def create(self, validated_data):
        chapters_data = validated_data.pop('chapters')
        book = Book.objects.create(**validated_data)
        
        for chapter_data in chapters_data:
            Chapter.objects.create(book=book, **chapter_data)
            
        return book
    
    def update(self, instance, validated_data):
        chapters_data = validated_data.pop('chapters', None)
        
        # Update book fields
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.publication_date = validated_data.get('publication_date', instance.publication_date)
        instance.save()
        
        # Update chapters
        if chapters_data is not None:
            # Keep track of which chapters we've seen
            chapters_ids = []
            
            for chapter_data in chapters_data:
                chapter_id = chapter_data.get('id', None)
                
                if chapter_id:
                    # If we have an ID, update the existing chapter
                    try:
                        chapter = Chapter.objects.get(id=chapter_id, book=instance)
                        chapter.title = chapter_data.get('title', chapter.title)
                        chapter.order = chapter_data.get('order', chapter.order)
                        chapter.content = chapter_data.get('content', chapter.content)
                        chapter.save()
                        chapters_ids.append(chapter_id)
                    except Chapter.DoesNotExist:
                        # If the chapter doesn't exist, create a new one
                        new_chapter = Chapter.objects.create(book=instance, **chapter_data)
                        chapters_ids.append(new_chapter.id)
                else:
                    # If no ID is provided, create a new chapter
                    new_chapter = Chapter.objects.create(book=instance, **chapter_data)
                    chapters_ids.append(new_chapter.id)
            
            # Delete any chapters that weren't in the update
            for chapter in instance.chapters.all():
                if chapter.id not in chapters_ids:
                    chapter.delete()
                    
        return instance
```

### 5. Handling Many-to-Many Relationships

For many-to-many relationships, like Books with Tags:

```python
# models.py
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    
    def __str__(self):
        return self.name

class Book(models.Model):
    # ... other fields
    tags = models.ManyToManyField(Tag, related_name='books')

# serializers.py
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class BookWithTagsSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'publication_date', 'tags']
    
    def create(self, validated_data):
        tags_data = validated_data.pop('tags')
        book = Book.objects.create(**validated_data)
        
        for tag_data in tags_data:
            tag, _ = Tag.objects.get_or_create(name=tag_data['name'])
            book.tags.add(tag)
            
        return book
    
    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)
        
        # Update book fields
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.publication_date = validated_data.get('publication_date', instance.publication_date)
        instance.save()
        
        # Update tags
        if tags_data is not None:
            # Clear existing tags
            instance.tags.clear()
            
            # Add new tags
            for tag_data in tags_data:
                tag, _ = Tag.objects.get_or_create(name=tag_data['name'])
                instance.tags.add(tag)
                
        return instance
```

### 6. Using drf-writable-nested Package

For complex nested relationships, consider using the `drf-writable-nested` package:

```bash
pip install drf-writable-nested
```

```python
# serializers.py
from rest_framework import serializers
from drf_writable_nested.serializers import WritableNestedModelSerializer
from .models import Author, Book, Chapter, Tag

class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name']

class ChapterSerializer(serializers.ModelSerializer):
    class Meta:
        model = Chapter
        fields = ['id', 'title', 'order', 'content']

class BookSerializer(WritableNestedModelSerializer):
    chapters = ChapterSerializer(many=True)
    tags = TagSerializer(many=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'publication_date', 'chapters', 'tags']

class AuthorWithBooksSerializer(WritableNestedModelSerializer):
    books = BookSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'bio', 'books']
```

This package handles create, update, and delete operations for nested relationships automatically.

## Notes

1. **Transaction Management**:
   - Wrap complex operations in transactions to ensure data consistency
   - Use `@transaction.atomic` decorator or context manager:
   ```python
   from django.db import transaction
   
   @transaction.atomic
   def create(self, validated_data):
       # Your complex creation logic
   ```

2. **Performance Considerations**:
   - Deeply nested serializers can cause performance issues
   - Use select_related and prefetch_related in your ViewSet's queryset
   - Consider bulk operations for large datasets

3. **Validation**:
   - Implement custom validation for nested data:
   ```python
   def validate(self, data):
       # Validate relationships between nested data
       chapters = data.get('chapters', [])
       if len(chapters) > 0:
           orders = [c['order'] for c in chapters]
           if len(orders) != len(set(orders)):
               raise serializers.ValidationError("Chapter orders must be unique")
       return data
   ```

4. **Alternative Approaches**:
   - Consider using separate endpoints for related objects
   - For very complex scenarios, use custom actions in ViewSets
   - Use third-party packages like `drf-writable-nested` or `drf-flex-fields`

5. **Error Handling**:
   - Provide clear error messages for nested validation failures
   - Consider implementing custom exception handlers for nested data

6. **Depth Control**:
   - Be cautious of recursion in nested serializers
   - Limit nesting depth for better performance and usability
