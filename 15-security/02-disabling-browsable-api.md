# Disabling Browsable API

## Problem
How do you prevent users from accessing the Django REST Framework Browsable API in production?

## Solution
Remove or restrict the `BrowsableAPIRenderer` in your REST framework settings for production environments.

## Code

### settings.py (production)
```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ]
}
```

### settings.py (development)
```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ]
}
```

## Notes
- The Browsable API is useful for development but should be disabled in production for security and performance.
- You can use environment variables to toggle renderers based on environment.
- Disabling the Browsable API does not affect API functionality for clients using JSON. 