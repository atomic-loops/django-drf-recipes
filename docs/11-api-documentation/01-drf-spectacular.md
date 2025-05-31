# Using drf-spectacular

## Problem
How do you generate OpenAPI 3.0-compliant documentation for your Django REST Framework API?

## Solution
Use the `drf-spectacular` package to automatically generate and serve OpenAPI schemas and interactive documentation (Swagger UI, Redoc).

## Code

### Install drf-spectacular
```bash
pip install drf-spectacular
```

### settings.py (additions)
```python
INSTALLED_APPS += [
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'DESCRIPTION': 'API documentation',
    'VERSION': '1.0.0',
}
```

### urls.py
```python
from django.urls import path
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView

urlpatterns = [
    path('schema/', SpectacularAPIView.as_view(), name='schema'),
    path('swagger/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

## Notes
- You can customize the schema with `extend_schema` decorators and settings.
- drf-spectacular supports advanced features like authentication, versioning, and custom components.
- Use the `/schema/` endpoint to download the raw OpenAPI schema (YAML or JSON). 