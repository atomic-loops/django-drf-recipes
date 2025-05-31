# Nested Serializers

## Problem

You need to represent related objects in your API response, such as displaying an author's details inside a book response or showing a list of books within an author response.

## Solution

Use nested serializers to include related object data in your API responses without having to make additional API calls.

## Code

### Basic Model Setup

First, let's set up the models with relationships:

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    biography = models.TextField(blank=True, null=True)
    date_of_birth = models.DateField(blank=True, null=True)
    website = models.URLField(blank=True, null=True)
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='books')
    publication_date = models.DateField(blank=True, null=True)
    isbn = models.CharField(max_length=13, unique=True)
    
    def __str__(self):
        return self.title
```

### Approach 1: Nesting a Serializer (One-to-Many)

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'publication_date', 'isbn']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    # Nested serializer for books
    books = BookSerializer(many=True, read_only=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website', 'books']
```

The above will produce a response like:

```json
{
    "id": 1,
    "name": "J.K. Rowling",
    "biography": "British author best known for the Harry Potter series...",
    "date_of_birth": "1965-07-31",
    "website": "https://www.jkrowling.com",
    "books": [
        {
            "id": 1,
            "title": "Harry Potter and the Philosopher's Stone",
            "publication_date": "1997-06-26",
            "isbn": "9780747532699"
        },
        {
            "id": 2,
            "title": "Harry Potter and the Chamber of Secrets",
            "publication_date": "1998-07-02",
            "isbn": "9780747538493"
        }
    ]
}
```

### Approach 2: Nesting a Serializer (Many-to-One)

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website']

class BookWithAuthorSerializer(serializers.ModelSerializer):
    # Nested serializer for author
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'publication_date', 'isbn', 'author']
```

This will produce a response like:

```json
{
    "id": 1,
    "title": "Harry Potter and the Philosopher's Stone",
    "publication_date": "1997-06-26",
    "isbn": "9780747532699",
    "author": {
        "id": 1,
        "name": "J.K. Rowling",
        "biography": "British author best known for the Harry Potter series...",
        "date_of_birth": "1965-07-31",
        "website": "https://www.jkrowling.com"
    }
}
```

### Approach 3: Using StringRelatedField

For simple use cases where you just want to display the string representation:

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class AuthorWithBookTitlesSerializer(serializers.ModelSerializer):
    # Using StringRelatedField for a list of book titles
    books = serializers.StringRelatedField(many=True, read_only=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website', 'books']
```

This produces:

```json
{
    "id": 1,
    "name": "J.K. Rowling",
    "biography": "British author best known for the Harry Potter series...",
    "date_of_birth": "1965-07-31",
    "website": "https://www.jkrowling.com",
    "books": [
        "Harry Potter and the Philosopher's Stone",
        "Harry Potter and the Chamber of Secrets"
    ]
}
```

### Approach 4: Using PrimaryKeyRelatedField

When you just need the IDs:

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class AuthorWithBookIdsSerializer(serializers.ModelSerializer):
    # Using PrimaryKeyRelatedField for a list of book IDs
    books = serializers.PrimaryKeyRelatedField(many=True, read_only=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website', 'books']
```

This produces:

```json
{
    "id": 1,
    "name": "J.K. Rowling",
    "biography": "British author best known for the Harry Potter series...",
    "date_of_birth": "1965-07-31",
    "website": "https://www.jkrowling.com",
    "books": [1, 2]
}
```

### Approach 5: Using HyperlinkedRelatedField

To include URLs to the related objects:

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class AuthorWithBookLinksSerializer(serializers.ModelSerializer):
    # Using HyperlinkedRelatedField for book links
    books = serializers.HyperlinkedRelatedField(
        many=True,
        read_only=True,
        view_name='book-detail'  # This should match your URL pattern name
    )
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website', 'books']
```

This produces:

```json
{
    "id": 1,
    "name": "J.K. Rowling",
    "biography": "British author best known for the Harry Potter series...",
    "date_of_birth": "1965-07-31",
    "website": "https://www.jkrowling.com",
    "books": [
        "http://example.com/api/books/1/",
        "http://example.com/api/books/2/"
    ]
}
```

### Approach 6: Using SerializerMethodField

For custom serialization logic:

```python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class AuthorWithCustomBooksSerializer(serializers.ModelSerializer):
    # Using SerializerMethodField for custom book data
    books = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'biography', 'date_of_birth', 'website', 'books']
    
    def get_books(self, obj):
        # Custom logic for presenting books
        books_data = []
        for book in obj.books.all():
            books_data.append({
                'id': book.id,
                'title': book.title,
                'year': book.publication_date.year if book.publication_date else None,
                'url': f"/api/books/{book.id}/"
            })
        return books_data
```

This produces:

```json
{
    "id": 1,
    "name": "J.K. Rowling",
    "biography": "British author best known for the Harry Potter series...",
    "date_of_birth": "1965-07-31",
    "website": "https://www.jkrowling.com",
    "books": [
        {
            "id": 1,
            "title": "Harry Potter and the Philosopher's Stone",
            "year": 1997,
            "url": "/api/books/1/"
        },
        {
            "id": 2,
            "title": "Harry Potter and the Chamber of Secrets",
            "year": 1998,
            "url": "/api/books/2/"
        }
    ]
}
```

## Notes

1. **Performance Considerations**:
   - Nested serializers can cause N+1 query problems
   - Use `select_related()` and `prefetch_related()` to optimize:
   ```python
   # In your view:
   queryset = Author.objects.prefetch_related('books')
   ```

2. **Depth Parameter**:
   - You can use the `depth` option for simple nesting:
   ```python
   class BookSerializer(serializers.ModelSerializer):
       class Meta:
           model = Book
           fields = ['id', 'title', 'author']
           depth = 1  # Will automatically serialize related fields one level deep
   ```

3. **Circular Nesting**:
   - Be careful with circular dependencies (author -> books -> author)
   - Consider creating different serializers for different scenarios

4. **Writing to Nested Serializers**:
   - By default, nested serializers are read-only
   - For writable nested serializers, see the recipe for [Writable Nested Serializers](03-writable-nested-serializers.md)

5. **Controlling Fields**:
   - It's a good practice to create specific serializers for different use cases
   - Consider creating a separate lightweight serializer for list views vs. detail views
