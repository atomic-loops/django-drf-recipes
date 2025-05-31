# Basic Project Structure

## Problem

You need a well-organized Django REST Framework project structure that is modular, maintainable, and follows best practices.

## Solution

Organize your project following DRF best practices with a focus on separation of concerns, modularity, and maintainability.

## Code

### 1. Recommended Project Structure

Here's a recommended structure for a Django REST Framework project:

```
myproject/
│
├── core/                  # Project configuration
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings/          # Split settings
│   │   ├── __init__.py
│   │   ├── base.py        # Common settings
│   │   ├── local.py       # Development settings
│   │   ├── production.py  # Production settings
│   │   └── test.py        # Test settings
│   ├── urls.py            # Main URL routing
│   └── wsgi.py
│
├── api/                   # API app
│   ├── __init__.py
│   ├── apps.py
│   ├── urls.py            # API routes
│   └── v1/                # API version 1
│       ├── __init__.py
│       ├── serializers/   # Serializers for this version
│       ├── views/         # Views for this version
│       └── urls.py        # URL patterns for v1
│
├── apps/                  # Domain-specific apps
│   ├── users/             # User management
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
│   │
│   ├── products/          # Another domain app
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
│   │
│   └── ...
│
├── utils/                 # Utility modules
│   ├── __init__.py
│   ├── permissions.py     # Custom permissions
│   ├── mixins.py          # Reusable mixins
│   └── validators.py      # Custom validators
│
├── docs/                  # Project documentation
│
├── manage.py
├── requirements/
│   ├── base.txt           # Common requirements
│   ├── local.txt          # Development requirements
│   └── production.txt     # Production requirements
│
├── .env.example           # Example environment variables
├── .gitignore
└── README.md
```

### 2. Settings Organization

Split your settings into multiple files:

```python
# core/settings/base.py - Common settings
import os
from pathlib import Path
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Build paths inside the project
BASE_DIR = Path(__file__).resolve().parent.parent.parent

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.getenv('SECRET_KEY')

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third-party apps
    'rest_framework',
    
    # Local apps
    'api',
    'apps.users',
    'apps.products',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'core.urls'

# Rest of your common settings...
```

```python
# core/settings/local.py - Development settings
from .base import *

DEBUG = True

ALLOWED_HOSTS = ['localhost', '127.0.0.1']

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# DRF settings for development
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

```python
# core/settings/production.py - Production settings
from .base import *

DEBUG = False

ALLOWED_HOSTS = [os.getenv('ALLOWED_HOSTS', '').split(',')]

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

# Security settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True

# DRF production settings
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    },
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

### 3. URL Structure

Organize your URLs to support API versioning:

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    path('api-auth/', include('rest_framework.urls')),
]
```

```python
# api/urls.py
from django.urls import path, include

urlpatterns = [
    path('v1/', include('api.v1.urls')),
    # In the future: path('v2/', include('api.v2.urls')),
]
```

```python
# api/v1/urls.py
from django.urls import path, include

urlpatterns = [
    path('users/', include('apps.users.urls')),
    path('products/', include('apps.products.urls')),
]
```

## Notes

1. **Modularity**:
   - Each app should focus on a specific domain
   - Keep apps small and focused on their responsibilities

2. **API Versioning**:
   - The folder structure supports API versioning from the start
   - Makes future API evolution easier without breaking existing clients

3. **Settings Management**:
   - Split settings by environment
   - Use environment variables for sensitive information
   - Never commit secrets to version control
   - Use different settings for different environments

4. **Application Discovery**:
   - When using the nested app structure, make sure to update the INSTALLED_APPS in settings
   - Use dotted path notation: 'apps.users' instead of just 'users'

5. **Alternative Structure**:
   - For smaller projects, you might not need to split apps into a separate directory
   - The API versioning structure is still recommended, even for smaller projects
