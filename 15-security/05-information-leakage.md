# Preventing Information Leakage

## Problem
How do you prevent sensitive information (e.g., stack traces, settings, internal errors) from leaking in Django REST Framework API responses?

## Solution
Set `DEBUG = False` in production, use custom error handlers, and avoid exposing sensitive data in error messages or serializers.

## Code

### settings.py (production)
```python
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']
```

### Custom exception handler (api/exception_handlers.py)
```python
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if response is not None:
        # Remove or mask sensitive details
        if response.status_code == 500:
            response.data = {'detail': 'Internal server error.'}
    return response
```

### settings.py (add to REST_FRAMEWORK)
```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'api.exception_handlers.custom_exception_handler',
}
```

## Notes
- Never run with `DEBUG = True` in production.
- Do not include sensitive fields (e.g., passwords, tokens) in serializers or logs.
- Use custom error handlers to control error response content. 