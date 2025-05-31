# Using APITestCase

**Problem:** You need to write comprehensive tests for your DRF API endpoints to ensure they work correctly, handle edge cases, and maintain functionality as your codebase evolves.

**Solution:** Use Django REST Framework's `APITestCase` class to create structured, maintainable tests for your API endpoints.

## Basic APITestCase Setup

```python
# tests/test_api.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User
from django.urls import reverse
from decimal import Decimal
from .models import Author, Book, Category

class BookAPITestCase(APITestCase):
    def setUp(self):
        """Set up test data that will be used across multiple tests"""
        # Create test user
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        # Create test data
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com',
            bio='Test author bio'
        )
        
        self.category = Category.objects.create(
            name='Programming',
            description='Programming books'
        )
        
        self.book = Book.objects.create(
            title='Test Book',
            author=self.author,
            isbn='1234567890123',
            published_date='2024-01-01',
            price=Decimal('29.99')
        )
        self.book.categories.add(self.category)
        
        # Store URLs for easy reuse
        self.books_url = reverse('book-list')
        self.book_detail_url = reverse('book-detail', kwargs={'pk': self.book.pk})

    def test_get_books_list(self):
        """Test retrieving list of books"""
        response = self.client.get(self.books_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
        self.assertEqual(response.data[0]['title'], 'Test Book')
        self.assertEqual(response.data[0]['author_name'], 'Test Author')

    def test_get_book_detail(self):
        """Test retrieving a specific book"""
        response = self.client.get(self.book_detail_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Book')
        self.assertEqual(response.data['isbn'], '1234567890123')
        self.assertIn('categories', response.data)

    def test_create_book(self):
        """Test creating a new book"""
        data = {
            'title': 'New Book',
            'author': self.author.pk,
            'isbn': '9876543210123',
            'published_date': '2024-02-01',
            'price': '39.99'
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Book.objects.count(), 2)
        self.assertEqual(response.data['title'], 'New Book')

    def test_update_book(self):
        """Test updating an existing book"""
        data = {
            'title': 'Updated Book Title',
            'price': '34.99'
        }
        
        response = self.client.patch(self.book_detail_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.book.refresh_from_db()
        self.assertEqual(self.book.title, 'Updated Book Title')
        self.assertEqual(str(self.book.price), '34.99')

    def test_delete_book(self):
        """Test deleting a book"""
        response = self.client.delete(self.book_detail_url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Book.objects.count(), 0)

    def test_book_not_found(self):
        """Test 404 response for non-existent book"""
        url = reverse('book-detail', kwargs={'pk': 999})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)
```

## Testing Validation and Error Cases

```python
class BookValidationTestCase(APITestCase):
    def setUp(self):
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com'
        )
        self.books_url = reverse('book-list')

    def test_create_book_missing_required_fields(self):
        """Test validation when required fields are missing"""
        data = {
            'title': 'Incomplete Book'
            # Missing author, isbn, published_date, price
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('author', response.data)
        self.assertIn('isbn', response.data)
        self.assertIn('published_date', response.data)
        self.assertIn('price', response.data)

    def test_create_book_invalid_isbn(self):
        """Test validation for invalid ISBN format"""
        data = {
            'title': 'Test Book',
            'author': self.author.pk,
            'isbn': '123',  # Too short
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('isbn', response.data)

    def test_create_book_duplicate_isbn(self):
        """Test validation for duplicate ISBN"""
        # Create first book
        Book.objects.create(
            title='First Book',
            author=self.author,
            isbn='1234567890123',
            published_date='2024-01-01',
            price=Decimal('29.99')
        )
        
        # Try to create second book with same ISBN
        data = {
            'title': 'Second Book',
            'author': self.author.pk,
            'isbn': '1234567890123',  # Duplicate ISBN
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('isbn', response.data)

    def test_create_book_invalid_price(self):
        """Test validation for invalid price"""
        data = {
            'title': 'Test Book',
            'author': self.author.pk,
            'isbn': '1234567890123',
            'published_date': '2024-01-01',
            'price': '-10.00'  # Negative price
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('price', response.data)

    def test_create_book_future_date(self):
        """Test validation for future publication date"""
        from datetime import date, timedelta
        
        future_date = date.today() + timedelta(days=365)
        
        data = {
            'title': 'Future Book',
            'author': self.author.pk,
            'isbn': '1234567890123',
            'published_date': future_date.isoformat(),
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        
        # Assuming you have validation that prevents future dates
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('published_date', response.data)
```

## Testing Query Parameters and Filtering

```python
class BookFilterTestCase(APITestCase):
    def setUp(self):
        self.author1 = Author.objects.create(name='Author One', email='author1@example.com')
        self.author2 = Author.objects.create(name='Author Two', email='author2@example.com')
        
        self.category1 = Category.objects.create(name='Programming')
        self.category2 = Category.objects.create(name='Science')
        
        # Create multiple books for testing
        self.book1 = Book.objects.create(
            title='Python Programming',
            author=self.author1,
            isbn='1111111111111',
            published_date='2023-01-01',
            price=Decimal('39.99')
        )
        self.book1.categories.add(self.category1)
        
        self.book2 = Book.objects.create(
            title='Data Science',
            author=self.author2,
            isbn='2222222222222',
            published_date='2024-01-01',
            price=Decimal('49.99')
        )
        self.book2.categories.add(self.category2)
        
        self.books_url = reverse('book-list')

    def test_filter_by_author(self):
        """Test filtering books by author"""
        url = f"{self.books_url}?author={self.author1.pk}"
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
        self.assertEqual(response.data[0]['title'], 'Python Programming')

    def test_search_by_title(self):
        """Test searching books by title"""
        url = f"{self.books_url}?search=Python"
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
        self.assertEqual(response.data[0]['title'], 'Python Programming')

    def test_filter_by_price_range(self):
        """Test filtering books by price range"""
        url = f"{self.books_url}?min_price=40&max_price=50"
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
        self.assertEqual(response.data[0]['title'], 'Data Science')

    def test_ordering(self):
        """Test ordering results"""
        url = f"{self.books_url}?ordering=-published_date"
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 2)
        # First book should be the one published in 2024
        self.assertEqual(response.data[0]['title'], 'Data Science')

    def test_pagination(self):
        """Test pagination parameters"""
        url = f"{self.books_url}?page_size=1&page=1"
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('results', response.data)
        self.assertEqual(len(response.data['results']), 1)
        self.assertIn('next', response.data)
        self.assertIn('previous', response.data)
```

## Testing Custom Actions

```python
class BookCustomActionsTestCase(APITestCase):
    def setUp(self):
        self.author = Author.objects.create(name='Test Author', email='author@example.com')
        self.book = Book.objects.create(
            title='Test Book',
            author=self.author,
            isbn='1234567890123',
            published_date='2024-01-01',
            price=Decimal('29.99')
        )

    def test_book_stats_action(self):
        """Test custom stats action"""
        # Create some reviews for the book
        Review.objects.create(
            book=self.book,
            reviewer_name='John Doe',
            rating=5,
            comment='Great book!'
        )
        Review.objects.create(
            book=self.book,
            reviewer_name='Jane Smith',
            rating=4,
            comment='Good read'
        )
        
        url = reverse('book-stats', kwargs={'pk': self.book.pk})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('total_reviews', response.data)
        self.assertIn('average_rating', response.data)
        self.assertEqual(response.data['total_reviews'], 2)
        self.assertEqual(response.data['average_rating'], 4.5)

    def test_mark_as_favorite_action(self):
        """Test custom action to mark book as favorite"""
        url = reverse('book-mark-favorite', kwargs={'pk': self.book.pk})
        response = self.client.post(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('message', response.data)
        self.book.refresh_from_db()
        # Assuming you have a 'is_favorite' field
        # self.assertTrue(self.book.is_favorite)
```

## Testing Error Handling

```python
class BookErrorHandlingTestCase(APITestCase):
    def test_method_not_allowed(self):
        """Test method not allowed responses"""
        books_url = reverse('book-list')
        
        # Try PATCH on list endpoint (should be not allowed)
        response = self.client.patch(books_url, {})
        self.assertEqual(response.status_code, status.HTTP_405_METHOD_NOT_ALLOWED)

    def test_invalid_json(self):
        """Test invalid JSON in request"""
        books_url = reverse('book-list')
        
        response = self.client.post(
            books_url,
            'invalid json',
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)

    def test_unsupported_media_type(self):
        """Test unsupported content type"""
        books_url = reverse('book-list')
        
        response = self.client.post(
            books_url,
            'some data',
            content_type='text/plain'
        )
        
        self.assertEqual(response.status_code, status.HTTP_415_UNSUPPORTED_MEDIA_TYPE)
```

## Testing with Different Data Formats

```python
class BookDataFormatTestCase(APITestCase):
    def setUp(self):
        self.author = Author.objects.create(name='Test Author', email='author@example.com')
        self.books_url = reverse('book-list')

    def test_json_format(self):
        """Test JSON request/response format"""
        data = {
            'title': 'JSON Book',
            'author': self.author.pk,
            'isbn': '1234567890123',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(
            self.books_url,
            data,
            format='json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'JSON Book')

    def test_form_data_format(self):
        """Test form data format"""
        data = {
            'title': 'Form Book',
            'author': self.author.pk,
            'isbn': '1234567890124',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'Form Book')
```

## Utility Methods for Testing

```python
class BookAPITestCase(APITestCase):
    def setUp(self):
        # ... setup code ...
        pass

    def create_sample_book(self, **kwargs):
        """Utility method to create a sample book with default values"""
        defaults = {
            'title': 'Sample Book',
            'author': self.author,
            'isbn': '1234567890123',
            'published_date': '2024-01-01',
            'price': Decimal('29.99')
        }
        defaults.update(kwargs)
        return Book.objects.create(**defaults)

    def assertBookEqual(self, book_data, book_instance):
        """Custom assertion to compare book data with book instance"""
        self.assertEqual(book_data['title'], book_instance.title)
        self.assertEqual(book_data['isbn'], book_instance.isbn)
        self.assertEqual(book_data['price'], str(book_instance.price))

    def test_with_utility_methods(self):
        """Example test using utility methods"""
        book = self.create_sample_book(title='Utility Test Book')
        
        response = self.client.get(reverse('book-detail', kwargs={'pk': book.pk}))
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertBookEqual(response.data, book)
```

**Notes:**
- Use `setUp()` method to create common test data
- Test both successful and error scenarios
- Use descriptive test method names that explain what is being tested
- Test all CRUD operations and custom actions
- Include tests for validation, filtering, and edge cases
- Use utility methods to reduce code duplication
- Assert on both status codes and response data
- Consider using factories (like Factory Boy) for complex test data
