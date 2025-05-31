# Using drf-yasg

## Problem
How do you generate Swagger/OpenAPI documentation for your Django REST Framework API using drf-yasg?

## Solution
Use the `drf-yasg` package to generate and serve interactive Swagger and Redoc documentation for your API.

## Code

### Install drf-yasg
```bash
pip install drf-yasg
```

### settings.py (additions)
```python
INSTALLED_APPS += [
    'drf_yasg',
]
```

### urls.py
```python
from django.urls import path, re_path
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
    openapi.Info(
        title="My API",
        default_version='v1',
        description="API documentation",
    ),
    public=True,
    permission_classes=(permissions.AllowAny,),
)

urlpatterns = [
    re_path(r'^swagger(?P<format>\.json|\.yaml)$', schema_view.without_ui(cache_timeout=0), name='schema-json'),
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0), name='schema-redoc'),
]
```

## Notes
- drf-yasg supports advanced schema customization with `@swagger_auto_schema` and `openapi.Schema`.
- Use the `/swagger/` and `/redoc/` endpoints for interactive docs, and `/swagger.json` or `/swagger.yaml` for raw schema.
- drf-yasg is widely used and supports most DRF features out of the box. 