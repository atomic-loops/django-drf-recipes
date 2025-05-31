# Project Setup and Installation

## Problem

You need to set up a new Django REST Framework project with the proper configuration to start building APIs.

## Solution

Set up a new Django project with DRF, configure the database, and prepare the environment.

## Code

### 1. Create a virtual environment and install dependencies

```bash
# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install Django and DRF
pip install django djangorestframework
pip install psycopg2-binary  # For PostgreSQL
pip install python-dotenv  # For environment variables

# Save dependencies
pip freeze > requirements.txt
```

### 2. Create a new Django project and app

```bash
# Create project
django-admin startproject core .

# Create your first app
python manage.py startapp api
```

### 3. Update settings.py to include DRF and your app

```python
# core/settings.py

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
]

# DRF settings
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

### 4. Configure database settings

```python
# For SQLite (default)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# For PostgreSQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'your_db_name',
        'USER': 'your_db_user',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

### 5. Set up environment variables (optional but recommended)

Create a `.env` file in your project root:

```
# .env
DEBUG=True
SECRET_KEY=your-secret-key
DATABASE_URL=postgres://user:password@localhost:5432/dbname
```

Add to settings.py:

```python
# At the top of settings.py
import os
from dotenv import load_dotenv

load_dotenv()

# Then use environment variables
DEBUG = os.getenv('DEBUG', 'False') == 'True'
SECRET_KEY = os.getenv('SECRET_KEY')
```

### 6. Configure URLs

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    path('api-auth/', include('rest_framework.urls')),  # DRF browsable API login
]
```

Create `api/urls.py`:

```python
# api/urls.py
from django.urls import path

urlpatterns = [
    # Your API endpoints will go here
]
```

### 7. Run initial migrations and server

```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

## Notes

1. **Security**: 
   - Always keep secret keys and sensitive data in environment variables
   - Never commit `.env` files to version control
   - Add `.env` to your `.gitignore` file

2. **Database Choice**:
   - SQLite is great for development but not recommended for production
   - PostgreSQL is often preferred for production Django applications

3. **Project Structure**:
   - Consider splitting settings.py into development/production files for larger projects
   - Create a dedicated folder for API versioning as your project grows

4. **Dependencies**:
   - Consider pinning versions in requirements.txt for reproducible builds
   - Use tools like pip-tools to manage dependencies effectively
