# Middleware in Django & DRF

Middleware is a way to process requests and responses globally before they reach your views or after your views return a response.

## What is Middleware?
- Middleware is a class with hooks for request/response processing.
- Used for logging, authentication, rate limiting, etc.

## Example: Simple Logging Middleware
```python
# middleware.py
from django.utils.deprecation import MiddlewareMixin

class SimpleLoggingMiddleware(MiddlewareMixin):
    def process_request(self, request):
        print(f"Request: {request.method} {request.path}")
        return None
    def process_response(self, request, response):
        print(f"Response: {response.status_code} {request.path}")
        return response
```

**To use:**
Add to your `settings.py`:
```python
MIDDLEWARE = [
    # ...
    'path.to.SimpleLoggingMiddleware',
    # ...
]
```

## DRF-Specific Middleware
- You can also use middleware for caching, OAuth2, and more (see repo docs for examples).

**References in this repo:**
- See `09-performance/02-caching-responses.md` and `05-authentication-permissions/05-oauth2.md` for advanced middleware patterns. 