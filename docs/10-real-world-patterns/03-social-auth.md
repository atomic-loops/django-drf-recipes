# Social Authentication

## Problem
How do you allow users to authenticate via social providers (e.g., Google, Facebook) in a Django REST Framework API?

## Solution
Use the `social-auth-app-django` package to integrate social authentication. Expose endpoints for social login and handle token exchange with the provider.

## Code

### Install Dependencies
```bash
pip install social-auth-app-django djangorestframework
```

### settings.py (additions)
```python
INSTALLED_APPS += [
    'social_django',
]

AUTHENTICATION_BACKENDS = (
    'social_core.backends.google.GoogleOAuth2',
    'django.contrib.auth.backends.ModelBackend',
)

SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = '<your-client-id>'
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = '<your-client-secret>'

# Add to MIDDLEWARE
MIDDLEWARE += [
    'social_django.middleware.SocialAuthExceptionMiddleware',
]

# Add to context processors
TEMPLATES[0]['OPTIONS']['context_processors'] += [
    'social_django.context_processors.backends',
    'social_django.context_processors.login_redirect',
]
```

### urls.py
```python
from django.urls import path, include

urlpatterns = [
    path('auth/', include('social_django.urls', namespace='social')),
]
```

### Social Login Endpoint (views.py)
For DRF, you can use a custom endpoint to accept a provider access token and authenticate the user:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from social_django.utils import load_strategy, load_backend
from social_core.exceptions import MissingBackend
from django.contrib.auth import login
from django.conf import settings

class SocialLoginView(APIView):
    def post(self, request, *args, **kwargs):
        provider = request.data.get('provider')  # e.g., 'google-oauth2'
        access_token = request.data.get('access_token')
        try:
            strategy = load_strategy(request)
            backend = load_backend(strategy=strategy, name=provider, redirect_uri=None)
            user = backend.do_auth(access_token)
            if user and user.is_active:
                login(request, user)
                # Optionally, return a DRF token or JWT
                return Response({'detail': 'Login successful'})
            else:
                return Response({'error': 'Authentication failed'}, status=status.HTTP_400_BAD_REQUEST)
        except MissingBackend:
            return Response({'error': 'Invalid provider'}, status=status.HTTP_400_BAD_REQUEST)
```

## Notes
- You must set up OAuth credentials with the provider (e.g., Google) and configure redirect URIs.
- For production, consider using packages like `dj-rest-auth` or `djoser` with social auth support for a more complete solution.
- Always validate and verify tokens server-side.
- The above example is simplified; adapt for your token strategy (e.g., JWT, session, etc.). 