# Test Data with Factory Boy

**Problem:** Creating complex test data manually is time-consuming, leads to code duplication, and makes tests harder to maintain. You need a systematic way to generate realistic, consistent test data.

**Solution:** Use Factory Boy to create flexible, reusable factories that generate test data with sensible defaults while allowing customization for specific test scenarios.

## Installation and Basic Setup

```bash
pip install factory-boy faker
```

```python
# tests/factories.py
import factory
from factory import fuzzy
from factory.django import DjangoModelFactory
from faker import Faker
from decimal import Decimal
from datetime import date, timedelta
from django.contrib.auth.models import User
from .models import Author, Publisher, Category, Book, Review

fake = Faker()

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')
    is_active = True
    is_staff = False
    
    @factory.post_generation
    def password(obj, create, extracted, **kwargs):
        """Set password after object creation"""
        if not create:
            return
        
        password = extracted or 'defaultpass123'
        obj.set_password(password)
        obj.save()

class AuthorFactory(DjangoModelFactory):
    class Meta:
        model = Author
    
    name = factory.Faker('name')
    email = factory.Faker('email')
    bio = factory.Faker('text', max_nb_chars=500)
    user = factory.SubFactory(UserFactory)
    birth_date = factory.Faker('date_of_birth', minimum_age=25, maximum_age=80)
    
class PublisherFactory(DjangoModelFactory):
    class Meta:
        model = Publisher
    
    name = factory.Faker('company')
    website = factory.Faker('url')
    established_year = fuzzy.FuzzyInteger(1900, 2020)
    country = factory.Faker('country')

class CategoryFactory(DjangoModelFactory):
    class Meta:
        model = Category
    
    name = factory.Faker('word')
    description = factory.Faker('sentence', nb_words=10)
    
    @factory.lazy_attribute
    def slug(self):
        return self.name.lower().replace(' ', '-')

class BookFactory(DjangoModelFactory):
    class Meta:
        model = Book
    
    title = factory.Faker('catch_phrase')
    author = factory.SubFactory(AuthorFactory)
    publisher = factory.SubFactory(PublisherFactory)
    isbn = factory.Sequence(lambda n: f"978{n:010d}")
    published_date = factory.Faker('date_between', start_date='-10y', end_date='today')
    price = fuzzy.FuzzyDecimal(9.99, 99.99, precision=2)
    pages = fuzzy.FuzzyInteger(100, 800)
    language = fuzzy.FuzzyChoice(['en', 'es', 'fr', 'de', 'it'])
    
    @factory.post_generation
    def categories(self, create, extracted, **kwargs):
        """Add categories after book creation"""
        if not create:
            return
        
        if extracted:
            for category in extracted:
                self.categories.add(category)
        else:
            # Add 1-3 random categories
            categories = CategoryFactory.create_batch(
                size=fake.random_int(min=1, max=3)
            )
            self.categories.add(*categories)

class ReviewFactory(DjangoModelFactory):
    class Meta:
        model = Review
    
    book = factory.SubFactory(BookFactory)
    reviewer_name = factory.Faker('name')
    rating = fuzzy.FuzzyInteger(1, 5)
    comment = factory.Faker('text', max_nb_chars=1000)
    created_at = factory.Faker('date_time_between', start_date='-1y', end_date='now')
    helpful_votes = fuzzy.FuzzyInteger(0, 100)
```

## Using Factories in Tests

```python
# tests/test_with_factories.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.urls import reverse
from .factories import UserFactory, AuthorFactory, BookFactory, ReviewFactory, CategoryFactory
from .models import Book, Review

class BookFactoryTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.author = AuthorFactory()
        self.books_url = reverse('book-list')

    def test_create_book_with_factory(self):
        """Test creating books using factory"""
        # Create a book with default factory values
        book = BookFactory()
        
        self.assertTrue(isinstance(book.title, str))
        self.assertTrue(len(book.isbn) == 13)
        self.assertIsNotNone(book.author)
        self.assertIsNotNone(book.publisher)

    def test_create_book_with_custom_values(self):
        """Test creating books with custom values"""
        # Override specific values
        book = BookFactory(
            title='Custom Title',
            price=Decimal('49.99'),
            author=self.author
        )
        
        self.assertEqual(book.title, 'Custom Title')
        self.assertEqual(book.price, Decimal('49.99'))
        self.assertEqual(book.author, self.author)

    def test_create_multiple_books(self):
        """Test creating multiple books"""
        # Create multiple books at once
        books = BookFactory.create_batch(5)
        
        self.assertEqual(len(books), 5)
        self.assertEqual(Book.objects.count(), 5)
        
        # All books should have unique ISBNs
        isbns = [book.isbn for book in books]
        self.assertEqual(len(isbns), len(set(isbns)))

    def test_create_book_with_specific_categories(self):
        """Test creating book with specific categories"""
        categories = CategoryFactory.create_batch(3)
        book = BookFactory(categories=categories)
        
        self.assertEqual(book.categories.count(), 3)
        self.assertEqual(set(book.categories.all()), set(categories))

    def test_api_endpoint_with_factory_data(self):
        """Test API endpoint using factory-generated data"""
        self.client.force_authenticate(user=self.user)
        
        # Create test data
        books = BookFactory.create_batch(10)
        
        response = self.client.get(self.books_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 10)
```

## Advanced Factory Patterns

```python
# tests/factories.py (advanced patterns)

class BookWithReviewsFactory(BookFactory):
    """Factory that creates a book with reviews"""
    
    @factory.post_generation
    def reviews(self, create, extracted, **kwargs):
        if not create:
            return
        
        if extracted:
            # Use provided reviews
            for review_data in extracted:
                ReviewFactory(book=self, **review_data)
        else:
            # Create random number of reviews
            review_count = fake.random_int(min=2, max=10)
            ReviewFactory.create_batch(review_count, book=self)

class PopularBookFactory(BookFactory):
    """Factory for popular books with high ratings"""
    
    price = fuzzy.FuzzyDecimal(39.99, 79.99, precision=2)
    pages = fuzzy.FuzzyInteger(300, 600)
    
    @factory.post_generation
    def reviews(self, create, extracted, **kwargs):
        if not create:
            return
        
        # Create mostly positive reviews
        for _ in range(fake.random_int(min=10, max=20)):
            ReviewFactory(
                book=self,
                rating=fuzzy.FuzzyInteger(4, 5).fuzz(),
                helpful_votes=fuzzy.FuzzyInteger(5, 50).fuzz()
            )

class RecentBookFactory(BookFactory):
    """Factory for recently published books"""
    
    published_date = factory.Faker('date_between', start_date='-1y', end_date='today')
    
class ClassicBookFactory(BookFactory):
    """Factory for classic/old books"""
    
    published_date = factory.Faker('date_between', start_date='-100y', end_date='-20y')
    price = fuzzy.FuzzyDecimal(15.99, 39.99, precision=2)

class SciFiBookFactory(BookFactory):
    """Factory for science fiction books"""
    
    title = factory.Faker('sentence', nb_words=3)
    
    @factory.post_generation
    def categories(self, create, extracted, **kwargs):
        if not create:
            return
        
        # Always add sci-fi category
        scifi_category, _ = Category.objects.get_or_create(
            name='Science Fiction',
            defaults={'description': 'Science fiction books'}
        )
        self.categories.add(scifi_category)
        
        # Add some related categories
        related_categories = CategoryFactory.create_batch(2)
        self.categories.add(*related_categories)

class AuthorWithBooksFactory(AuthorFactory):
    """Factory that creates an author with multiple books"""
    
    @factory.post_generation
    def books(self, create, extracted, **kwargs):
        if not create:
            return
        
        book_count = extracted or fake.random_int(min=2, max=8)
        BookFactory.create_batch(book_count, author=self)
```

## Traits and Conditional Factories

```python
# tests/factories.py (traits and conditions)

class BookFactory(DjangoModelFactory):
    class Meta:
        model = Book
    
    title = factory.Faker('catch_phrase')
    author = factory.SubFactory(AuthorFactory)
    publisher = factory.SubFactory(PublisherFactory)
    isbn = factory.Sequence(lambda n: f"978{n:010d}")
    published_date = factory.Faker('date_between', start_date='-10y', end_date='today')
    price = fuzzy.FuzzyDecimal(9.99, 99.99, precision=2)
    
    class Params:
        # Define traits
        is_bestseller = factory.Trait(
            price=fuzzy.FuzzyDecimal(29.99, 59.99, precision=2),
            pages=fuzzy.FuzzyInteger(250, 500),
        )
        
        is_budget = factory.Trait(
            price=fuzzy.FuzzyDecimal(4.99, 19.99, precision=2),
            pages=fuzzy.FuzzyInteger(100, 200),
        )
        
        is_premium = factory.Trait(
            price=fuzzy.FuzzyDecimal(79.99, 149.99, precision=2),
            pages=fuzzy.FuzzyInteger(400, 800),
        )

# Usage in tests
class BookTraitsTestCase(APITestCase):
    def test_bestseller_books(self):
        """Test creating bestseller books"""
        bestseller = BookFactory(is_bestseller=True)
        
        self.assertGreaterEqual(bestseller.price, Decimal('29.99'))
        self.assertLessEqual(bestseller.price, Decimal('59.99'))
        self.assertGreaterEqual(bestseller.pages, 250)

    def test_budget_books(self):
        """Test creating budget books"""
        budget_book = BookFactory(is_budget=True)
        
        self.assertLessEqual(budget_book.price, Decimal('19.99'))
        self.assertLessEqual(budget_book.pages, 200)
```

## Using Factories with Faker Providers

```python
# tests/factories.py (custom faker providers)

from faker.providers import BaseProvider

class BookProvider(BaseProvider):
    """Custom provider for book-related fake data"""
    
    book_genres = [
        'Mystery', 'Romance', 'Science Fiction', 'Fantasy', 'Thriller',
        'Biography', 'History', 'Self-Help', 'Cooking', 'Travel'
    ]
    
    book_formats = ['Hardcover', 'Paperback', 'Ebook', 'Audiobook']
    
    def book_genre(self):
        return self.random_element(self.book_genres)
    
    def book_format(self):
        return self.random_element(self.book_formats)
    
    def isbn13(self):
        """Generate a valid-looking ISBN-13"""
        import random
        prefix = '978'
        group = str(random.randint(0, 9))
        publisher = str(random.randint(10000, 99999))
        title = str(random.randint(100, 999))
        # Simplified check digit calculation
        check = str(random.randint(0, 9))
        return f"{prefix}{group}{publisher}{title}{check}"

# Add provider to faker instance
fake.add_provider(BookProvider)

class EnhancedBookFactory(BookFactory):
    """Book factory using custom faker provider"""
    
    isbn = factory.LazyFunction(lambda: fake.isbn13())
    format = factory.LazyFunction(lambda: fake.book_format())
    
    @factory.post_generation
    def categories(self, create, extracted, **kwargs):
        if not create:
            return
        
        if not extracted:
            # Use custom provider for genre
            genre_name = fake.book_genre()
            category, _ = Category.objects.get_or_create(
                name=genre_name,
                defaults={'description': f'{genre_name} books'}
            )
            self.categories.add(category)
```

## Factory Fixtures and Test Data Management

```python
# tests/fixtures.py
from .factories import (
    UserFactory, AuthorFactory, BookFactory, ReviewFactory,
    PopularBookFactory, RecentBookFactory
)

class TestDataManager:
    """Utility class to manage test data creation"""
    
    @staticmethod
    def create_basic_dataset():
        """Create basic test dataset"""
        users = UserFactory.create_batch(5)
        authors = AuthorFactory.create_batch(3)
        books = BookFactory.create_batch(10)
        
        # Add reviews to some books
        for book in books[:5]:
            ReviewFactory.create_batch(
                size=fake.random_int(min=2, max=8),
                book=book
            )
        
        return {
            'users': users,
            'authors': authors,
            'books': books
        }
    
    @staticmethod
    def create_marketplace_scenario():
        """Create a realistic marketplace scenario"""
        # Create popular authors with multiple books
        popular_authors = AuthorFactory.create_batch(3)
        
        # Create bestsellers
        bestsellers = []
        for author in popular_authors:
            bestseller = PopularBookFactory(author=author)
            bestsellers.append(bestseller)
        
        # Create recent releases
        recent_books = RecentBookFactory.create_batch(5)
        
        # Create some budget books
        budget_books = BookFactory.create_batch(
            3, 
            is_budget=True
        )
        
        return {
            'popular_authors': popular_authors,
            'bestsellers': bestsellers,
            'recent_books': recent_books,
            'budget_books': budget_books
        }

# Usage in tests
class MarketplaceTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.test_data = TestDataManager.create_marketplace_scenario()
        
    def test_bestsellers_api(self):
        """Test bestsellers endpoint"""
        self.client.force_authenticate(user=self.user)
        
        url = reverse('book-bestsellers')
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        # Should return the bestsellers we created
        self.assertEqual(len(response.data), 3)
```

## Performance Considerations

```python
# tests/test_performance.py
from django.test import TestCase, TransactionTestCase
from django.test.utils import override_settings
from django.db import transaction
from .factories import BookFactory, ReviewFactory

class FactoryPerformanceTestCase(TestCase):
    def test_bulk_creation_performance(self):
        """Test performance of bulk creation"""
        import time
        
        # Measure time for individual creation
        start_time = time.time()
        for _ in range(100):
            BookFactory()
        individual_time = time.time() - start_time
        
        # Clear database
        Book.objects.all().delete()
        
        # Measure time for batch creation
        start_time = time.time()
        BookFactory.create_batch(100)
        batch_time = time.time() - start_time
        
        # Batch should be faster (though not guaranteed in all cases)
        print(f"Individual: {individual_time:.2f}s, Batch: {batch_time:.2f}s")
    
    @override_settings(DEBUG=False)
    def test_factory_with_database_optimization(self):
        """Test factory usage with database optimizations"""
        with transaction.atomic():
            # Create related objects efficiently
            books = BookFactory.create_batch(50)
            
            # Add reviews in bulk
            reviews = []
            for book in books:
                for _ in range(5):
                    reviews.append(ReviewFactory.build(book=book))
            
            Review.objects.bulk_create(reviews)
        
        self.assertEqual(Book.objects.count(), 50)
        self.assertEqual(Review.objects.count(), 250)
```

**Notes:**
- Use `build()` instead of `create()` when you don't need database persistence
- Use `create_batch()` for creating multiple objects efficiently
- Leverage `@factory.post_generation` for complex relationships
- Create custom providers for domain-specific fake data
- Use traits to create variations of the same factory
- Consider performance implications when creating large datasets
- Organize factories in separate modules for better maintainability
- Use realistic data that matches your production patterns
