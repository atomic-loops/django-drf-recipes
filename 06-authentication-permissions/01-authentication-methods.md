# Authentication Methods

## Problem

You need to secure your Django REST Framework API by implementing user authentication to ensure only authorized users can access your endpoints.

## Solution

Implement and compare different authentication methods in DRF: Basic Authentication, Token Authentication, and JWT (JSON Web Tokens) Authentication.

## Code

### 1. Basic Authentication

Basic Authentication is the simplest form, sending credentials with each request. It's included in DRF by default.

#### Setup

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    # ...
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

#### Usage

Basic Authentication requires no additional views. Your API will now require HTTP Basic Authentication with each request:

```
Authorization: Basic <base64-encoded-username-password>
```

Example with curl:

```bash
curl -X GET http://localhost:8000/api/books/ -H 'Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ='
```

#### View-Level Configuration

You can also configure authentication at the view level:

```python
from rest_framework.authentication import BasicAuthentication
from rest_framework.permissions import IsAuthenticated

class BookListView(APIView):
    authentication_classes = [BasicAuthentication]
    permission_classes = [IsAuthenticated]
    
    # Rest of your view...
```

### 2. Token Authentication

Token Authentication uses a token that's generated on the server and sent with each request. It's more secure than Basic Authentication.

#### Setup

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework.authtoken',  # Add this line
    # ...
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

Run migrations to create token table:

```bash
python manage.py migrate
```

#### Creating Tokens

You can create tokens in several ways:

1. Using the admin interface
2. Using a management command
3. Automatically when a user is created (using signals)
4. Using a login API endpoint

Let's implement a login endpoint:

```python
# views.py
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.authtoken.models import Token
from rest_framework.response import Response

class CustomAuthToken(ObtainAuthToken):
    def post(self, request, *args, **kwargs):
        serializer = self.serializer_class(data=request.data,
                                          context={'request': request})
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        token, created = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'user_id': user.pk,
            'email': user.email
        })
```

```python
# urls.py
from django.urls import path
from .views import CustomAuthToken

urlpatterns = [
    # ... other URL patterns
    path('api-token-auth/', CustomAuthToken.as_view(), name='api_token_auth'),
]
```

#### Usage

Once a token is obtained, it's included in the Authorization header:

```
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

Example with curl:

```bash
curl -X GET http://localhost:8000/api/books/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'
```

### 3. JWT Authentication (SimpleJWT)

JWT Authentication is a modern approach that uses signed tokens containing encoded user information.

#### Setup

First, install the package:

```bash
pip install djangorestframework-simplejwt
```

Configure settings:

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    # ...
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# JWT settings
from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'VERIFYING_KEY': None,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',
}
```

Add the token URLs:

```python
# urls.py
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    # ... other URL patterns
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

#### Usage

To get a token, send a POST request with username and password:

```bash
curl -X POST http://localhost:8000/api/token/ -d "username=admin&password=password"
```

Response:

```json
{
    "refresh": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "access": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
}
```

Then use the access token in the Authorization header:

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

To refresh the access token when it expires:

```bash
curl -X POST http://localhost:8000/api/token/refresh/ -d "refresh=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
```

### 4. Using Multiple Authentication Methods

You can combine multiple authentication methods:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

With this configuration, DRF will try each authentication class in order until one succeeds.

## Notes

1. **Security Considerations**:
   - Basic Authentication should only be used over HTTPS
   - Token Authentication is more secure than Basic but still requires HTTPS
   - JWT tokens are secure but need proper expiration times
   - Never store sensitive data in JWT tokens

2. **Token Storage**:
   - For Token Authentication, tokens are stored in the database
   - For JWT, tokens are not stored server-side (stateless)
   - Consider token expiration and refresh strategies

3. **Performance**:
   - JWT can reduce database queries since validation happens through signature
   - Token Authentication requires a database lookup per request
   - Basic Authentication requires password verification per request

4. **Logout Handling**:
   - With Token Authentication, delete the token
   - With JWT, you need a token blacklist (SimpleJWT provides this)
   - Consider shorter token lifetimes for security

5. **Use Cases**:
   - Basic Authentication: Development, simple APIs, admin interfaces
   - Token Authentication: Mobile apps, SPAs with moderate security needs
   - JWT: Microservices, distributed systems, high-scale applications

6. **Advanced JWT Features**:
   - Token blacklisting
   - Custom claims
   - Token types (access/refresh)
   - Custom token validation

7. **Additional Authentication Options**:
   - OAuth2 with django-oauth-toolkit
   - Social Authentication with django-allauth
   - Custom authentication schemes
