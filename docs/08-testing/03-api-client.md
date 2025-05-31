# Using APIClient

**Problem:** You need more control over your API requests during testing, including setting custom headers, handling file uploads, and testing different content types and request formats.

**Solution:** Use Django REST Framework's `APIClient` class which provides advanced features for testing API endpoints with full control over request parameters.

## Basic APIClient Usage

```python
# tests/test_client.py
from rest_framework.test import APIClient, APITestCase
from rest_framework import status
from django.contrib.auth.models import User
from django.urls import reverse
from django.core.files.uploadedfile import SimpleUploadedFile
import json
from decimal import Decimal
from .models import Author, Book, BookCover

class APIClientTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com'
        )
        
        self.book = Book.objects.create(
            title='Test Book',
            author=self.author,
            isbn='1234567890123',
            published_date='2024-01-01',
            price=Decimal('29.99')
        )
        
        self.books_url = reverse('book-list')
        self.book_detail_url = reverse('book-detail', kwargs={'pk': self.book.pk})

    def test_get_request_with_headers(self):
        """Test GET request with custom headers"""
        headers = {
            'HTTP_ACCEPT': 'application/json',
            'HTTP_USER_AGENT': 'test-client/1.0',
            'HTTP_CUSTOM_HEADER': 'custom-value'
        }
        
        response = self.client.get(self.books_url, **headers)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.get('Content-Type'), 'application/json')

    def test_post_request_with_json(self):
        """Test POST request with JSON data"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'JSON Book',
            'author': self.author.pk,
            'isbn': '9876543210123',
            'published_date': '2024-02-01',
            'price': '39.99'
        }
        
        response = self.client.post(
            self.books_url,
            data=json.dumps(data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'JSON Book')

    def test_patch_request_with_form_data(self):
        """Test PATCH request with form data"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'Updated Title',
            'price': '34.99'
        }
        
        response = self.client.patch(
            self.book_detail_url,
            data=data,
            format='multipart'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Updated Title')

    def test_delete_request(self):
        """Test DELETE request"""
        self.client.force_authenticate(user=self.user)
        
        response = self.client.delete(self.book_detail_url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Book.objects.count(), 0)
```

## File Upload Testing

```python
class FileUploadTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com'
        )
        
        self.books_url = reverse('book-list')

    def test_upload_book_with_cover(self):
        """Test uploading a book with cover image"""
        self.client.force_authenticate(user=self.user)
        
        # Create a simple test image
        image_content = (
            b'\x47\x49\x46\x38\x39\x61\x01\x00\x01\x00\x80\x00\x00\x05\x04\x04\x00\x00\x00\x2c\x00'
            b'\x00\x00\x00\x01\x00\x01\x00\x00\x02\x02\x44\x01\x00\x3b'
        )
        cover_image = SimpleUploadedFile(
            name='cover.gif',
            content=image_content,
            content_type='image/gif'
        )
        
        data = {
            'title': 'Book with Cover',
            'author': self.author.pk,
            'isbn': '1111111111111',
            'published_date': '2024-01-01',
            'price': '29.99',
            'cover_image': cover_image
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            format='multipart'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertIn('cover_image', response.data)
        
        # Verify file was actually uploaded
        book = Book.objects.get(title='Book with Cover')
        self.assertTrue(book.cover_image)

    def test_upload_multiple_files(self):
        """Test uploading multiple files"""
        self.client.force_authenticate(user=self.user)
        
        # Create test files
        file1 = SimpleUploadedFile(
            name='document1.txt',
            content=b'Document 1 content',
            content_type='text/plain'
        )
        file2 = SimpleUploadedFile(
            name='document2.txt',
            content=b'Document 2 content',
            content_type='text/plain'
        )
        
        data = {
            'title': 'Book with Documents',
            'author': self.author.pk,
            'isbn': '2222222222222',
            'published_date': '2024-01-01',
            'price': '29.99',
            'documents': [file1, file2]
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            format='multipart'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    def test_invalid_file_upload(self):
        """Test upload with invalid file type"""
        self.client.force_authenticate(user=self.user)
        
        # Create an invalid file (e.g., executable)
        invalid_file = SimpleUploadedFile(
            name='malicious.exe',
            content=b'not an image',
            content_type='application/x-executable'
        )
        
        data = {
            'title': 'Book with Invalid File',
            'author': self.author.pk,
            'isbn': '3333333333333',
            'published_date': '2024-01-01',
            'price': '29.99',
            'cover_image': invalid_file
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            format='multipart'
        )
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('cover_image', response.data)

    def test_file_size_limit(self):
        """Test file size validation"""
        self.client.force_authenticate(user=self.user)
        
        # Create a large file (assuming you have size limits)
        large_content = b'x' * (10 * 1024 * 1024)  # 10MB
        large_file = SimpleUploadedFile(
            name='large_cover.jpg',
            content=large_content,
            content_type='image/jpeg'
        )
        
        data = {
            'title': 'Book with Large File',
            'author': self.author.pk,
            'isbn': '4444444444444',
            'published_date': '2024-01-01',
            'price': '29.99',
            'cover_image': large_file
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            format='multipart'
        )
        
        # Assuming you have file size validation
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

## Custom Headers and Authentication Testing

```python
class CustomHeadersTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.books_url = reverse('book-list')

    def test_custom_authentication_header(self):
        """Test custom authentication header"""
        # Simulate a custom API key authentication
        api_key = 'test-api-key-12345'
        
        response = self.client.get(
            self.books_url,
            HTTP_X_API_KEY=api_key
        )
        
        # Response depends on your custom authentication implementation
        self.assertIn(response.status_code, [status.HTTP_200_OK, status.HTTP_401_UNAUTHORIZED])

    def test_content_negotiation(self):
        """Test content type negotiation"""
        # Request XML format
        response = self.client.get(
            self.books_url,
            HTTP_ACCEPT='application/xml'
        )
        
        # Check if XML is supported or if it falls back to JSON
        self.assertIn(response.status_code, [status.HTTP_200_OK, status.HTTP_406_NOT_ACCEPTABLE])

    def test_language_header(self):
        """Test language preference header"""
        response = self.client.get(
            self.books_url,
            HTTP_ACCEPT_LANGUAGE='es-ES,es;q=0.9,en;q=0.8'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        # Check if response is localized (depends on your i18n setup)

    def test_user_agent_header(self):
        """Test User-Agent header"""
        response = self.client.get(
            self.books_url,
            HTTP_USER_AGENT='TestBot/1.0 (Testing Purpose)'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_custom_request_id_header(self):
        """Test custom request ID tracking"""
        request_id = 'test-request-12345'
        
        response = self.client.get(
            self.books_url,
            HTTP_X_REQUEST_ID=request_id
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        # Check if request ID is echoed back
        self.assertEqual(response.get('X-Request-ID'), request_id)
```

## Different Content Types Testing

```python
class ContentTypeTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.client.force_authenticate(user=self.user)
        
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com'
        )
        
        self.books_url = reverse('book-list')

    def test_json_content_type(self):
        """Test JSON content type"""
        data = {
            'title': 'JSON Book',
            'author': self.author.pk,
            'isbn': '1111111111111',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(
            self.books_url,
            data=json.dumps(data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'JSON Book')

    def test_form_data_content_type(self):
        """Test form data content type"""
        data = {
            'title': 'Form Book',
            'author': self.author.pk,
            'isbn': '2222222222222',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            content_type='application/x-www-form-urlencoded'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'Form Book')

    def test_multipart_form_data(self):
        """Test multipart form data"""
        data = {
            'title': 'Multipart Book',
            'author': self.author.pk,
            'isbn': '3333333333333',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(
            self.books_url,
            data=data,
            format='multipart'
        )
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'Multipart Book')

    def test_unsupported_content_type(self):
        """Test unsupported content type"""
        data = 'title=XML Book&author=1'
        
        response = self.client.post(
            self.books_url,
            data=data,
            content_type='application/xml'
        )
        
        self.assertEqual(response.status_code, status.HTTP_415_UNSUPPORTED_MEDIA_TYPE)
```

## Testing Response Formats

```python
class ResponseFormatTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.author = Author.objects.create(
            name='Test Author',
            email='author@example.com'
        )
        
        self.book = Book.objects.create(
            title='Test Book',
            author=self.author,
            isbn='1234567890123',
            published_date='2024-01-01',
            price=Decimal('29.99')
        )
        
        self.books_url = reverse('book-list')
        self.book_detail_url = reverse('book-detail', kwargs={'pk': self.book.pk})

    def test_json_response_format(self):
        """Test JSON response format"""
        response = self.client.get(
            self.book_detail_url,
            HTTP_ACCEPT='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.get('Content-Type'), 'application/json')
        
        # Verify JSON structure
        self.assertIn('title', response.data)
        self.assertIn('author', response.data)
        self.assertIn('isbn', response.data)

    def test_format_parameter(self):
        """Test format parameter in URL"""
        response = self.client.get(f"{self.book_detail_url}?format=json")
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('application/json', response.get('Content-Type'))

    def test_format_suffix(self):
        """Test format suffix in URL"""
        url = self.book_detail_url.rstrip('/') + '.json'
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('application/json', response.get('Content-Type'))

    def test_browsable_api_format(self):
        """Test browsable API format"""
        response = self.client.get(
            self.book_detail_url,
            HTTP_ACCEPT='text/html'
        )
        
        # Should return HTML for browsable API
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('text/html', response.get('Content-Type'))
```

## Advanced APIClient Features

```python
class AdvancedAPIClientTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.books_url = reverse('book-list')

    def test_session_persistence(self):
        """Test session persistence across requests"""
        # Login using session authentication
        self.client.login(username='testuser', password='testpass123')
        
        # First request
        response1 = self.client.get(self.books_url)
        self.assertEqual(response1.status_code, status.HTTP_200_OK)
        
        # Second request should maintain session
        response2 = self.client.get(self.books_url)
        self.assertEqual(response2.status_code, status.HTTP_200_OK)
        
        # Logout
        self.client.logout()
        
        # Third request should be unauthenticated
        response3 = self.client.get(self.books_url)
        self.assertEqual(response3.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_cookie_handling(self):
        """Test cookie handling"""
        # Set a custom cookie
        self.client.cookies['custom_cookie'] = 'test_value'
        
        response = self.client.get(self.books_url)
        
        # Check if cookie was sent (depends on your view implementation)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_follow_redirects(self):
        """Test following redirects"""
        # Assuming you have a redirect view
        redirect_url = reverse('book-redirect')  # This would redirect to books_url
        
        response = self.client.get(redirect_url, follow=True)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.redirect_chain), 1)

    def test_request_factory_alternative(self):
        """Test using RequestFactory as alternative"""
        from rest_framework.test import APIRequestFactory
        from .views import BookViewSet
        
        factory = APIRequestFactory()
        view = BookViewSet.as_view({'get': 'list'})
        
        request = factory.get(self.books_url)
        response = view(request)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_custom_client_class(self):
        """Test using custom client class"""
        class CustomAPIClient(APIClient):
            def __init__(self, **defaults):
                super().__init__(**defaults)
                # Add custom headers by default
                self.defaults.update({
                    'HTTP_X_CUSTOM_HEADER': 'default-value'
                })
        
        custom_client = CustomAPIClient()
        response = custom_client.get(self.books_url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Testing Error Handling

```python
class ErrorHandlingTestCase(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.books_url = reverse('book-list')

    def test_malformed_json(self):
        """Test malformed JSON handling"""
        response = self.client.post(
            self.books_url,
            data='{"malformed": json}',
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)

    def test_empty_request_body(self):
        """Test empty request body"""
        response = self.client.post(
            self.books_url,
            data='',
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)

    def test_oversized_request(self):
        """Test oversized request handling"""
        # Create a very large payload
        large_data = {'title': 'x' * 1000000}  # 1MB string
        
        response = self.client.post(
            self.books_url,
            data=json.dumps(large_data),
            content_type='application/json'
        )
        
        # Response depends on your server configuration
        self.assertIn(response.status_code, [
            status.HTTP_400_BAD_REQUEST,
            status.HTTP_413_REQUEST_ENTITY_TOO_LARGE
        ])

    def test_special_characters_handling(self):
        """Test special characters in request"""
        data = {
            'title': 'Book with émojis 🚀 and üñíçødé',
            'description': 'Special chars: <>&"\'',
        }
        
        response = self.client.post(
            self.books_url,
            data=json.dumps(data, ensure_ascii=False),
            content_type='application/json; charset=utf-8'
        )
        
        # Should handle Unicode properly
        self.assertIn(response.status_code, [
            status.HTTP_201_CREATED,
            status.HTTP_400_BAD_REQUEST
        ])
```

**Notes:**
- `APIClient` provides more control than basic test client
- Use appropriate content types for different request formats
- Test file uploads with `SimpleUploadedFile`
- Custom headers can be set using `HTTP_` prefix
- Use `format='multipart'` for file uploads
- Test both successful and error scenarios
- Consider using `APIRequestFactory` for unit testing views directly
- Always clean up authentication state between tests
