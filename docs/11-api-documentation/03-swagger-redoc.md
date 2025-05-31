# Generating Swagger & Redoc

## Problem
How do you provide interactive API documentation (Swagger UI and Redoc) for your Django REST Framework APIs?

## Solution
Use either `drf-yasg` or `drf-spectacular` to generate and serve Swagger UI and Redoc documentation endpoints.

## Code

### With drf-spectacular
```python
from django.urls import path
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView

urlpatterns = [
    path('schema/', SpectacularAPIView.as_view(), name='schema'),
    path('swagger/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

### With drf-yasg
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
- Both packages provide interactive documentation and raw schema endpoints.
- Swagger UI is great for testing endpoints; Redoc is excellent for browsing and reading API docs.
- Protect documentation endpoints in production if your API is private. 