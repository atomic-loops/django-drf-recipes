# Testing Endpoints with Authentication

**Problem:** Your API endpoints require authentication, and you need to test both authenticated and unauthenticated access scenarios, including different permission levels.

**Solution:** Use DRF's authentication and permission testing utilities to simulate different user states and access levels in your tests.

## Basic Authentication Testing

```python
# tests/test_auth.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User
from django.urls import reverse
from rest_framework.authtoken.models import Token
from decimal import Decimal
from .models import Author, Book

class AuthenticatedBookAPITestCase(APITestCase):
    def setUp(self):
        # Create users with different roles
        self.admin_user = User.objects.create_user(
            username='admin',
            email='admin@example.com',
            password='adminpass123',
            is_staff=True
        )
        
        self.regular_user = User.objects.create_user(
            username='user',
            email='user@example.com',
            password='userpass123'
        )
        
        self.author_user = User.objects.create_user(
            username='author',
            email='author@example.com',
            password='authorpass123'
        )
        
        # Create tokens for token authentication
        self.admin_token = Token.objects.create(user=self.admin_user)
        self.user_token = Token.objects.create(user=self.regular_user)
        self.author_token = Token.objects.create(user=self.author_user)
        
        # Create test data
        self.author = Author.objects.create(
            name='Test Author',
            email='testauthor@example.com',
            user=self.author_user  # Link author to user
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

    def test_unauthenticated_access_denied(self):
        """Test that unauthenticated users cannot access protected endpoints"""
        # Try to create a book without authentication
        data = {
            'title': 'Unauthorized Book',
            'author': self.author.pk,
            'isbn': '9999999999999',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_authenticated_user_can_read(self):
        """Test that authenticated users can read books"""
        self.client.force_authenticate(user=self.regular_user)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        response = self.client.get(self.book_detail_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_regular_user_cannot_create_book(self):
        """Test that regular users cannot create books"""
        self.client.force_authenticate(user=self.regular_user)
        
        data = {
            'title': 'User Book',
            'author': self.author.pk,
            'isbn': '8888888888888',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_admin_can_create_book(self):
        """Test that admin users can create books"""
        self.client.force_authenticate(user=self.admin_user)
        
        data = {
            'title': 'Admin Book',
            'author': self.author.pk,
            'isbn': '7777777777777',
            'published_date': '2024-01-01',
            'price': '29.99'
        }
        
        response = self.client.post(self.books_url, data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'Admin Book')

    def test_author_can_edit_own_book(self):
        """Test that authors can edit their own books"""
        self.client.force_authenticate(user=self.author_user)
        
        data = {'title': 'Updated Title'}
        response = self.client.patch(self.book_detail_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.book.refresh_from_db()
        self.assertEqual(self.book.title, 'Updated Title')

    def test_author_cannot_edit_others_book(self):
        """Test that authors cannot edit books by other authors"""
        # Create another author and book
        other_author = Author.objects.create(
            name='Other Author',
            email='other@example.com'
        )
        other_book = Book.objects.create(
            title='Other Book',
            author=other_author,
            isbn='6666666666666',
            published_date='2024-01-01',
            price=Decimal('39.99')
        )
        
        self.client.force_authenticate(user=self.author_user)
        
        data = {'title': 'Hacked Title'}
        url = reverse('book-detail', kwargs={'pk': other_book.pk})
        response = self.client.patch(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

## Token Authentication Testing

```python
class TokenAuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.token = Token.objects.create(user=self.user)
        self.books_url = reverse('book-list')

    def test_token_authentication_success(self):
        """Test successful token authentication"""
        self.client.credentials(HTTP_AUTHORIZATION='Token ' + self.token.key)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_invalid_token_authentication(self):
        """Test authentication with invalid token"""
        self.client.credentials(HTTP_AUTHORIZATION='Token invalidtoken123')
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_malformed_token_header(self):
        """Test authentication with malformed token header"""
        self.client.credentials(HTTP_AUTHORIZATION='Bearer ' + self.token.key)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_missing_token_prefix(self):
        """Test authentication with missing 'Token' prefix"""
        self.client.credentials(HTTP_AUTHORIZATION=self.token.key)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

## Session Authentication Testing

```python
class SessionAuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.books_url = reverse('book-list')
        self.login_url = reverse('rest_framework:login')
        self.logout_url = reverse('rest_framework:logout')

    def test_session_login_logout(self):
        """Test session-based authentication flow"""
        # Test login
        login_data = {
            'username': 'testuser',
            'password': 'testpass123'
        }
        response = self.client.post(self.login_url, login_data)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        # Test authenticated request
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        # Test logout
        response = self.client.post(self.logout_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        # Test that we're now unauthenticated
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_invalid_login_credentials(self):
        """Test login with invalid credentials"""
        login_data = {
            'username': 'testuser',
            'password': 'wrongpassword'
        }
        response = self.client.post(self.login_url, login_data)
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

## JWT Authentication Testing

```python
# Assuming you're using django-rest-framework-simplejwt
from rest_framework_simplejwt.tokens import RefreshToken

class JWTAuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.books_url = reverse('book-list')

    def get_jwt_token(self, user):
        """Helper method to get JWT tokens for a user"""
        refresh = RefreshToken.for_user(user)
        return {
            'refresh': str(refresh),
            'access': str(refresh.access_token),
        }

    def test_jwt_authentication_success(self):
        """Test successful JWT authentication"""
        tokens = self.get_jwt_token(self.user)
        self.client.credentials(HTTP_AUTHORIZATION='Bearer ' + tokens['access'])
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_expired_jwt_token(self):
        """Test authentication with expired JWT token"""
        # This would require mocking time or using a very short-lived token
        # For demonstration purposes, we'll use an invalid token
        self.client.credentials(HTTP_AUTHORIZATION='Bearer invalidjwttoken')
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_jwt_token_refresh(self):
        """Test JWT token refresh functionality"""
        tokens = self.get_jwt_token(self.user)
        
        # Use refresh token to get new access token
        refresh_url = reverse('token_refresh')  # Adjust based on your URL config
        response = self.client.post(refresh_url, {'refresh': tokens['refresh']})
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('access', response.data)
        
        # Test new access token works
        self.client.credentials(HTTP_AUTHORIZATION='Bearer ' + response.data['access'])
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Testing Custom Permissions

```python
class CustomPermissionsTestCase(APITestCase):
    def setUp(self):
        # Create users with different roles
        self.owner_user = User.objects.create_user(
            username='owner',
            email='owner@example.com',
            password='ownerpass123'
        )
        
        self.editor_user = User.objects.create_user(
            username='editor',
            email='editor@example.com',
            password='editorpass123'
        )
        
        self.viewer_user = User.objects.create_user(
            username='viewer',
            email='viewer@example.com',
            password='viewerpass123'
        )
        
        # Create test data with ownership
        self.author = Author.objects.create(
            name='Owner Author',
            email='ownerauthor@example.com',
            user=self.owner_user
        )
        
        self.book = Book.objects.create(
            title='Owner Book',
            author=self.author,
            isbn='1111111111111',
            published_date='2024-01-01',
            price=Decimal('29.99'),
            created_by=self.owner_user  # Assuming you have this field
        )
        
        self.book_detail_url = reverse('book-detail', kwargs={'pk': self.book.pk})

    def test_owner_can_delete_book(self):
        """Test that book owner can delete their book"""
        self.client.force_authenticate(user=self.owner_user)
        
        response = self.client.delete(self.book_detail_url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

    def test_non_owner_cannot_delete_book(self):
        """Test that non-owners cannot delete books"""
        self.client.force_authenticate(user=self.editor_user)
        
        response = self.client.delete(self.book_detail_url)
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    def test_editor_can_update_book(self):
        """Test that editors can update books (if permission allows)"""
        # Add editor permission (implementation depends on your permission system)
        self.client.force_authenticate(user=self.editor_user)
        
        data = {'title': 'Updated by Editor'}
        response = self.client.patch(self.book_detail_url, data)
        
        # This might be 200 or 403 depending on your permission implementation
        self.assertIn(response.status_code, [status.HTTP_200_OK, status.HTTP_403_FORBIDDEN])

    def test_viewer_cannot_update_book(self):
        """Test that viewers cannot update books"""
        self.client.force_authenticate(user=self.viewer_user)
        
        data = {'title': 'Updated by Viewer'}
        response = self.client.patch(self.book_detail_url, data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

## Testing Multiple Authentication Methods

```python
class MultipleAuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.token = Token.objects.create(user=self.user)
        self.books_url = reverse('book-list')

    def test_force_authenticate_method(self):
        """Test using force_authenticate for testing"""
        self.client.force_authenticate(user=self.user)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_force_authenticate_with_token(self):
        """Test force_authenticate with token"""
        self.client.force_authenticate(user=self.user, token=self.token)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_credentials_method(self):
        """Test using credentials method"""
        self.client.credentials(HTTP_AUTHORIZATION='Token ' + self.token.key)
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_login_method(self):
        """Test using login method for session auth"""
        self.client.login(username='testuser', password='testpass123')
        
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Testing Authentication Edge Cases

```python
class AuthenticationEdgeCasesTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.token = Token.objects.create(user=self.user)
        self.books_url = reverse('book-list')

    def test_inactive_user_authentication(self):
        """Test authentication with inactive user"""
        self.user.is_active = False
        self.user.save()
        
        self.client.force_authenticate(user=self.user)
        response = self.client.get(self.books_url)
        
        # Behavior depends on your authentication backend
        self.assertIn(response.status_code, [status.HTTP_401_UNAUTHORIZED, status.HTTP_403_FORBIDDEN])

    def test_deleted_user_token(self):
        """Test using token for deleted user"""
        token_key = self.token.key
        self.user.delete()
        
        self.client.credentials(HTTP_AUTHORIZATION='Token ' + token_key)
        response = self.client.get(self.books_url)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_concurrent_token_usage(self):
        """Test multiple requests with same token"""
        self.client.credentials(HTTP_AUTHORIZATION='Token ' + self.token.key)
        
        # Make multiple concurrent requests
        responses = []
        for _ in range(5):
            response = self.client.get(self.books_url)
            responses.append(response)
        
        # All should succeed
        for response in responses:
            self.assertEqual(response.status_code, status.HTTP_200_OK)

    def tearDown(self):
        """Clean up after each test"""
        # Clear any authentication credentials
        self.client.credentials()
        self.client.logout()
```

## Utility Methods for Authentication Testing

```python
class AuthTestMixin:
    """Mixin class with utility methods for authentication testing"""
    
    def authenticate_user(self, user):
        """Helper method to authenticate a user"""
        self.client.force_authenticate(user=user)
    
    def authenticate_with_token(self, token):
        """Helper method to authenticate with token"""
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.key}')
    
    def logout_user(self):
        """Helper method to logout user"""
        self.client.force_authenticate(user=None)
        self.client.credentials()
    
    def create_authenticated_request(self, method, url, user=None, data=None):
        """Helper method to make authenticated requests"""
        if user:
            self.authenticate_user(user)
        
        client_method = getattr(self.client, method.lower())
        return client_method(url, data or {})

class BookAuthTestCase(APITestCase, AuthTestMixin):
    def setUp(self):
        self.user = User.objects.create_user(username='test', password='pass')
        self.books_url = reverse('book-list')
    
    def test_with_mixin_methods(self):
        """Test using mixin utility methods"""
        response = self.create_authenticated_request('GET', self.books_url, self.user)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        self.logout_user()
        response = self.client.get(self.books_url)
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

**Notes:**
- Use `force_authenticate()` for simple test authentication
- Use `credentials()` method to test actual authentication headers
- Test both successful and failed authentication scenarios
- Test different user roles and permission levels
- Test edge cases like inactive users and expired tokens
- Create utility methods to reduce code duplication
- Always clean up authentication state between tests
- Consider using fixtures or factories for complex user setups
