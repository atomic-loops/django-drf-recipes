# OAuth2 Integration

## Problem

You need to implement OAuth2 authentication in your DRF API to allow users to authenticate using third-party providers (Google, Facebook, GitHub, etc.) or to provide OAuth2 authorization server capabilities for other applications.

## Solution

Use Django OAuth Toolkit (DOT) along with social authentication libraries to implement comprehensive OAuth2 support, including both client and server functionality for secure API access.

## Installation and Setup

```bash
# Install required packages
pip install django-oauth-toolkit
pip install social-auth-app-django
pip install requests-oauthlib
```

```python
# settings.py
INSTALLED_APPS = [
    # ... other apps
    'oauth2_provider',
    'social_django',
    'rest_framework',
]

MIDDLEWARE = [
    'oauth2_provider.middleware.OAuth2TokenMiddleware',
    # ... other middleware
    'social_django.middleware.SocialAuthExceptionMiddleware',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# OAuth2 Settings
OAUTH2_PROVIDER = {
    'SCOPES': {
        'read': 'Read scope',
        'write': 'Write scope',
        'read_profile': 'Read user profile',
        'write_profile': 'Write user profile',
        'admin': 'Admin access',
    },
    'ACCESS_TOKEN_EXPIRE_SECONDS': 3600,
    'REFRESH_TOKEN_EXPIRE_SECONDS': 86400,
    'AUTHORIZATION_CODE_EXPIRE_SECONDS': 600,
    'ROTATE_REFRESH_TOKEN': True,
}

# Social Auth Settings
AUTHENTICATION_BACKENDS = [
    'social_core.backends.google.GoogleOAuth2',
    'social_core.backends.github.GithubOAuth2',
    'social_core.backends.facebook.FacebookOAuth2',
    'django.contrib.auth.backends.ModelBackend',
]

# Social Auth Configuration
SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = 'your-google-client-id'
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = 'your-google-client-secret'
SOCIAL_AUTH_GOOGLE_OAUTH2_SCOPE = ['email', 'profile']

SOCIAL_AUTH_GITHUB_KEY = 'your-github-client-id'
SOCIAL_AUTH_GITHUB_SECRET = 'your-github-client-secret'
SOCIAL_AUTH_GITHUB_SCOPE = ['user:email']

SOCIAL_AUTH_FACEBOOK_KEY = 'your-facebook-app-id'
SOCIAL_AUTH_FACEBOOK_SECRET = 'your-facebook-app-secret'
SOCIAL_AUTH_FACEBOOK_SCOPE = ['email']

# Redirect URLs
SOCIAL_AUTH_LOGIN_REDIRECT_URL = '/api/auth/success/'
SOCIAL_AUTH_LOGIN_ERROR_URL = '/api/auth/error/'
```

## OAuth2 Provider (Authorization Server)

### Basic OAuth2 Application Setup

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from oauth2_provider.models import Application, AccessToken

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    avatar = models.URLField(blank=True)
    bio = models.TextField(blank=True)
    website = models.URLField(blank=True)
    
    def __str__(self):
        return f"{self.user.username}'s profile"

class APIKey(models.Model):
    """Custom API key model for additional security"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    key = models.CharField(max_length=40, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    last_used = models.DateTimeField(null=True, blank=True)
    is_active = models.BooleanField(default=True)
    
    def __str__(self):
        return f"{self.user.username} - {self.name}"
```

### Custom OAuth2 Views

```python
# views.py
from rest_framework import generics, status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from oauth2_provider.contrib.rest_framework import TokenHasScope
from oauth2_provider.models import Application, AccessToken
from django.contrib.auth.models import User
from .serializers import UserProfileSerializer, UserSerializer

class ProtectedView(generics.RetrieveAPIView):
    """Example protected view requiring OAuth2 token"""
    permission_classes = [IsAuthenticated, TokenHasScope]
    required_scopes = ['read']
    serializer_class = UserSerializer
    
    def get_object(self):
        return self.request.user

class UserProfileView(generics.RetrieveUpdateAPIView):
    """User profile view with OAuth2 protection"""
    permission_classes = [IsAuthenticated, TokenHasScope]
    required_scopes = ['read_profile']
    serializer_class = UserProfileSerializer
    
    def get_object(self):
        profile, created = UserProfile.objects.get_or_create(user=self.request.user)
        return profile
    
    def get_permissions(self):
        """Different scopes for different actions"""
        if self.request.method in ['PUT', 'PATCH']:
            self.required_scopes = ['write_profile']
        else:
            self.required_scopes = ['read_profile']
        return super().get_permissions()

@api_view(['POST'])
@permission_classes([IsAuthenticated])
def revoke_token(request):
    """Revoke the current access token"""
    token = request.auth
    if token:
        token.delete()
        return Response({'message': 'Token revoked successfully'})
    return Response({'error': 'No token found'}, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET'])
@permission_classes([IsAuthenticated, TokenHasScope])
def token_info(request):
    """Get information about the current token"""
    required_scopes = ['read']
    
    token = request.auth
    if isinstance(token, AccessToken):
        return Response({
            'token_type': 'Bearer',
            'scope': token.scope,
            'expires_at': token.expires,
            'application': token.application.name,
            'user': token.user.username,
        })
    
    return Response({'error': 'Invalid token'}, status=status.HTTP_400_BAD_REQUEST)
```

### Custom OAuth2 Serializers

```python
# serializers.py
from rest_framework import serializers
from django.contrib.auth.models import User
from oauth2_provider.models import Application, AccessToken
from .models import UserProfile

class UserSerializer(serializers.ModelSerializer):
    profile = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name', 'profile']
        read_only_fields = ['id', 'username']
    
    def get_profile(self, obj):
        try:
            return UserProfileSerializer(obj.userprofile).data
        except UserProfile.DoesNotExist:
            return None

class UserProfileSerializer(serializers.ModelSerializer):
    class Meta:
        model = UserProfile
        fields = ['avatar', 'bio', 'website']

class ApplicationSerializer(serializers.ModelSerializer):
    """Serializer for OAuth2 applications"""
    client_secret = serializers.CharField(write_only=True)
    
    class Meta:
        model = Application
        fields = [
            'client_id', 'client_secret', 'name', 'client_type', 
            'authorization_grant_type', 'redirect_uris'
        ]
        read_only_fields = ['client_id']

class AccessTokenSerializer(serializers.ModelSerializer):
    """Serializer for access token information"""
    application_name = serializers.CharField(source='application.name', read_only=True)
    user_username = serializers.CharField(source='user.username', read_only=True)
    
    class Meta:
        model = AccessToken
        fields = [
            'token', 'expires', 'scope', 'application_name', 
            'user_username', 'created', 'updated'
        ]
        read_only_fields = ['token', 'created', 'updated']
```

## Social OAuth2 Integration

### Social Authentication Views

```python
# social_auth_views.py
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from social_django.utils import load_strategy, load_backend
from social_core.backends.oauth import BaseOAuth2
from social_core.exceptions import AuthException, AuthCanceled
from oauth2_provider.models import Application, AccessToken
from django.contrib.auth import login
import requests

@api_view(['POST'])
@permission_classes([AllowAny])
def social_oauth2_login(request):
    """Exchange social provider code for our OAuth2 token"""
    provider = request.data.get('provider')
    code = request.data.get('code')
    redirect_uri = request.data.get('redirect_uri')
    
    if not all([provider, code]):
        return Response(
            {'error': 'Provider and code are required'}, 
            status=status.HTTP_400_BAD_REQUEST
        )
    
    try:
        # Load social auth strategy and backend
        strategy = load_strategy(request)
        backend = load_backend(strategy, provider, redirect_uri)
        
        # Complete the social authentication
        user = backend.complete(user=None, request=request)
        
        if user and user.is_active:
            # Create or get OAuth2 application for social auth
            application = get_or_create_social_application()
            
            # Create access token
            access_token = AccessToken.objects.create(
                user=user,
                application=application,
                token=generate_token(),
                expires=timezone.now() + timedelta(seconds=3600),
                scope='read write read_profile write_profile'
            )
            
            return Response({
                'access_token': access_token.token,
                'token_type': 'Bearer',
                'expires_in': 3600,
                'scope': access_token.scope,
                'user': UserSerializer(user).data
            })
        
        return Response(
            {'error': 'Authentication failed'}, 
            status=status.HTTP_401_UNAUTHORIZED
        )
    
    except (AuthException, AuthCanceled) as e:
        return Response(
            {'error': f'Social authentication error: {str(e)}'}, 
            status=status.HTTP_400_BAD_REQUEST
        )

@api_view(['POST'])
@permission_classes([AllowAny])
def google_oauth2_login(request):
    """Specific Google OAuth2 integration"""
    access_token = request.data.get('access_token')
    
    if not access_token:
        return Response(
            {'error': 'Access token is required'}, 
            status=status.HTTP_400_BAD_REQUEST
        )
    
    try:
        # Verify Google token
        google_user_info = verify_google_token(access_token)
        
        if google_user_info:
            # Get or create user
            user = get_or_create_user_from_google(google_user_info)
            
            # Create our OAuth2 token
            application = get_or_create_social_application()
            our_token = AccessToken.objects.create(
                user=user,
                application=application,
                token=generate_token(),
                expires=timezone.now() + timedelta(seconds=3600),
                scope='read write read_profile'
            )
            
            return Response({
                'access_token': our_token.token,
                'token_type': 'Bearer',
                'expires_in': 3600,
                'user': UserSerializer(user).data
            })
        
        return Response(
            {'error': 'Invalid Google token'}, 
            status=status.HTTP_401_UNAUTHORIZED
        )
    
    except Exception as e:
        return Response(
            {'error': f'Google authentication error: {str(e)}'}, 
            status=status.HTTP_400_BAD_REQUEST
        )

def verify_google_token(access_token):
    """Verify Google access token and get user info"""
    try:
        response = requests.get(
            'https://www.googleapis.com/oauth2/v2/userinfo',
            headers={'Authorization': f'Bearer {access_token}'}
        )
        
        if response.status_code == 200:
            return response.json()
        return None
    
    except requests.RequestException:
        return None

def get_or_create_user_from_google(google_user_info):
    """Create or get user from Google user info"""
    email = google_user_info.get('email')
    
    try:
        user = User.objects.get(email=email)
    except User.DoesNotExist:
        user = User.objects.create_user(
            username=email,
            email=email,
            first_name=google_user_info.get('given_name', ''),
            last_name=google_user_info.get('family_name', '')
        )
        
        # Create profile with Google info
        UserProfile.objects.create(
            user=user,
            avatar=google_user_info.get('picture', ''),
        )
    
    return user

def get_or_create_social_application():
    """Get or create OAuth2 application for social auth"""
    try:
        return Application.objects.get(name='Social Auth App')
    except Application.DoesNotExist:
        return Application.objects.create(
            name='Social Auth App',
            client_type=Application.CLIENT_PUBLIC,
            authorization_grant_type=Application.GRANT_AUTHORIZATION_CODE,
        )

def generate_token():
    """Generate a secure random token"""
    import secrets
    return secrets.token_urlsafe(32)
```

## OAuth2 Client Integration

### Client-Side OAuth2 Flow

```python
# oauth_client.py
import requests
from urllib.parse import urlencode
from django.conf import settings

class OAuth2Client:
    """Client for making OAuth2 requests to external APIs"""
    
    def __init__(self, client_id, client_secret, authorization_url, token_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.authorization_url = authorization_url
        self.token_url = token_url
    
    def get_authorization_url(self, redirect_uri, scope=None, state=None):
        """Generate authorization URL for OAuth2 flow"""
        params = {
            'client_id': self.client_id,
            'response_type': 'code',
            'redirect_uri': redirect_uri,
            'scope': scope or 'read',
            'state': state,
        }
        
        return f"{self.authorization_url}?{urlencode(params)}"
    
    def exchange_code_for_token(self, code, redirect_uri):
        """Exchange authorization code for access token"""
        data = {
            'grant_type': 'authorization_code',
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'code': code,
            'redirect_uri': redirect_uri,
        }
        
        response = requests.post(self.token_url, data=data)
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Token exchange failed: {response.text}")
    
    def refresh_token(self, refresh_token):
        """Refresh an access token"""
        data = {
            'grant_type': 'refresh_token',
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'refresh_token': refresh_token,
        }
        
        response = requests.post(self.token_url, data=data)
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Token refresh failed: {response.text}")
    
    def make_authenticated_request(self, url, access_token, method='GET', **kwargs):
        """Make authenticated request using access token"""
        headers = kwargs.pop('headers', {})
        headers['Authorization'] = f'Bearer {access_token}'
        
        response = requests.request(method, url, headers=headers, **kwargs)
        return response

# Example usage
github_client = OAuth2Client(
    client_id=settings.SOCIAL_AUTH_GITHUB_KEY,
    client_secret=settings.SOCIAL_AUTH_GITHUB_SECRET,
    authorization_url='https://github.com/login/oauth/authorize',
    token_url='https://github.com/login/oauth/access_token'
)
```

### Token Management Views

```python
# token_management.py
from rest_framework import generics, status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from oauth2_provider.models import AccessToken, RefreshToken, Application
from django.utils import timezone

class UserTokensView(generics.ListAPIView):
    """List user's active OAuth2 tokens"""
    permission_classes = [IsAuthenticated]
    serializer_class = AccessTokenSerializer
    
    def get_queryset(self):
        return AccessToken.objects.filter(
            user=self.request.user,
            expires__gt=timezone.now()
        ).select_related('application')

@api_view(['POST'])
def refresh_access_token(request):
    """Refresh an access token using refresh token"""
    refresh_token = request.data.get('refresh_token')
    
    if not refresh_token:
        return Response(
            {'error': 'Refresh token is required'}, 
            status=status.HTTP_400_BAD_REQUEST
        )
    
    try:
        refresh_token_obj = RefreshToken.objects.get(token=refresh_token)
        
        if refresh_token_obj.access_token.expires < timezone.now():
            # Create new access token
            new_access_token = AccessToken.objects.create(
                user=refresh_token_obj.user,
                application=refresh_token_obj.application,
                token=generate_token(),
                expires=timezone.now() + timedelta(seconds=3600),
                scope=refresh_token_obj.access_token.scope
            )
            
            # Optionally rotate refresh token
            if getattr(settings, 'OAUTH2_PROVIDER', {}).get('ROTATE_REFRESH_TOKEN'):
                refresh_token_obj.delete()
                new_refresh_token = RefreshToken.objects.create(
                    user=refresh_token_obj.user,
                    token=generate_token(),
                    application=refresh_token_obj.application,
                    access_token=new_access_token
                )
                refresh_token_value = new_refresh_token.token
            else:
                refresh_token_obj.access_token = new_access_token
                refresh_token_obj.save()
                refresh_token_value = refresh_token
            
            return Response({
                'access_token': new_access_token.token,
                'token_type': 'Bearer',
                'expires_in': 3600,
                'refresh_token': refresh_token_value,
                'scope': new_access_token.scope
            })
        
        return Response(
            {'error': 'Access token not expired'}, 
            status=status.HTTP_400_BAD_REQUEST
        )
    
    except RefreshToken.DoesNotExist:
        return Response(
            {'error': 'Invalid refresh token'}, 
            status=status.HTTP_401_UNAUTHORIZED
        )

@api_view(['POST'])
@permission_classes([IsAuthenticated])
def revoke_all_tokens(request):
    """Revoke all user's tokens"""
    AccessToken.objects.filter(user=request.user).delete()
    RefreshToken.objects.filter(user=request.user).delete()
    
    return Response({'message': 'All tokens revoked successfully'})
```

## URL Configuration

```python
# urls.py
from django.urls import path, include
from . import views, social_auth_views, token_management

urlpatterns = [
    # OAuth2 Provider URLs
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
    
    # Social Authentication
    path('auth/', include('social_django.urls', namespace='social')),
    path('auth/social-login/', social_auth_views.social_oauth2_login, name='social_oauth2_login'),
    path('auth/google-login/', social_auth_views.google_oauth2_login, name='google_oauth2_login'),
    
    # Protected API Endpoints
    path('api/profile/', views.UserProfileView.as_view(), name='user_profile'),
    path('api/me/', views.ProtectedView.as_view(), name='user_detail'),
    
    # Token Management
    path('api/token/info/', views.token_info, name='token_info'),
    path('api/token/revoke/', views.revoke_token, name='revoke_token'),
    path('api/token/refresh/', token_management.refresh_access_token, name='refresh_token'),
    path('api/tokens/', token_management.UserTokensView.as_view(), name='user_tokens'),
    path('api/tokens/revoke-all/', token_management.revoke_all_tokens, name='revoke_all_tokens'),
]
```

## Advanced OAuth2 Features

### Custom Scopes and Permissions

```python
# custom_permissions.py
from oauth2_provider.contrib.rest_framework import TokenHasScope
from rest_framework.permissions import BasePermission

class TokenHasReadWriteScope(TokenHasScope):
    """Custom permission requiring both read and write scopes"""
    required_scopes = ['read', 'write']

class AdminScopePermission(BasePermission):
    """Permission requiring admin scope"""
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        token = request.auth
        if token and hasattr(token, 'scope'):
            return 'admin' in token.scope
        
        return False

class ConditionalScopePermission(BasePermission):
    """Permission with conditional scope requirements"""
    
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        
        token = request.auth
        if not token or not hasattr(token, 'scope'):
            return False
        
        # Different scope requirements based on HTTP method
        if request.method in ['POST', 'PUT', 'PATCH', 'DELETE']:
            return 'write' in token.scope
        else:
            return 'read' in token.scope
```

### OAuth2 Middleware

```python
# middleware.py
from django.utils.deprecation import MiddlewareMixin
from oauth2_provider.models import AccessToken
from django.http import JsonResponse
import json

class OAuth2LoggingMiddleware(MiddlewareMixin):
    """Middleware to log OAuth2 token usage"""
    
    def process_request(self, request):
        if hasattr(request, 'auth') and isinstance(request.auth, AccessToken):
            # Log token usage
            import logging
            logger = logging.getLogger('oauth2')
            logger.info(f"OAuth2 token used: {request.auth.token[:8]}... by {request.user}")
        
        return None

class OAuth2RateLimitMiddleware(MiddlewareMixin):
    """Simple rate limiting for OAuth2 tokens"""
    
    def process_request(self, request):
        if hasattr(request, 'auth') and isinstance(request.auth, AccessToken):
            from django.core.cache import cache
            
            token = request.auth
            cache_key = f"oauth2_rate_limit_{token.token}"
            
            # Get current count
            current_count = cache.get(cache_key, 0)
            
            # Check rate limit (100 requests per hour)
            if current_count >= 100:
                return JsonResponse(
                    {'error': 'Rate limit exceeded'}, 
                    status=429
                )
            
            # Increment counter
            cache.set(cache_key, current_count + 1, 3600)  # 1 hour
        
        return None
```

## Notes

### Security Considerations

- **HTTPS Only**: Always use HTTPS in production for OAuth2 flows
- **State Parameter**: Use state parameter to prevent CSRF attacks
- **Token Storage**: Store tokens securely on the client side
- **Scope Limitation**: Use minimal required scopes
- **Token Rotation**: Implement refresh token rotation for better security

### Best Practices

1. **Short-Lived Tokens**: Use short-lived access tokens (1 hour or less)
2. **Secure Redirect URIs**: Validate redirect URIs strictly
3. **Rate Limiting**: Implement rate limiting for token endpoints
4. **Audit Logging**: Log all OAuth2 operations for security monitoring
5. **Token Introspection**: Provide token introspection endpoints for clients

### Common Patterns

1. **Mobile Apps**: Use PKCE (Proof Key for Code Exchange) for mobile OAuth2
2. **Single Sign-On**: Implement SSO using OAuth2 with multiple applications
3. **API Gateway Integration**: Use OAuth2 tokens with API gateways
4. **Microservices**: Share OAuth2 tokens across microservices
5. **Third-Party Integration**: Allow third-party applications to access your API

### Error Handling

- Return appropriate OAuth2 error responses
- Handle token expiration gracefully
- Provide clear error messages for debugging
- Implement proper exception handling for social auth failures

This recipe provides comprehensive OAuth2 integration for both provider and client scenarios, enabling secure authentication and authorization in your DRF APIs.
