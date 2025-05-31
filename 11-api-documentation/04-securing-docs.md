# Securing Documentation

## Problem
How do you restrict access to your API documentation (Swagger, Redoc, etc.) so it is not publicly accessible in production?

## Solution
Use Django permissions or custom authentication classes to protect documentation endpoints. You can require login, admin status, or a custom permission.

## Code

### Example: Restrict Swagger/Redoc to Admins (drf-spectacular)
```python
from django.urls import path
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView
from rest_framework.permissions import IsAdminUser

urlpatterns = [
    path('schema/', SpectacularAPIView.as_view(permission_classes=[IsAdminUser]), name='schema'),
    path('swagger/', SpectacularSwaggerView.as_view(url_name='schema', permission_classes=[IsAdminUser]), name='swagger-ui'),
    path('redoc/', SpectacularRedocView.as_view(url_name='schema', permission_classes=[IsAdminUser]), name='redoc'),
]
```

### Example: Restrict Swagger/Redoc to Authenticated Users (drf-yasg)
```python
from django.urls import path, re_path
from rest_framework.permissions import IsAuthenticated
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
    openapi.Info(
        title="My API",
        default_version='v1',
        description="API documentation",
    ),
    public=False,
    permission_classes=(IsAuthenticated,),
)

urlpatterns = [
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0), name='schema-redoc'),
]
```

## Notes
- Never expose sensitive or internal API documentation to the public.
- You can use custom permission classes for more granular control (e.g., staff only, specific groups).
- Always use HTTPS for documentation endpoints in production. 