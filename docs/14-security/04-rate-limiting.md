# Rate Limiting

## Problem
How do you prevent abuse and protect your Django REST Framework API from excessive requests (DoS, brute force, etc.)?

## Solution
Enable DRF's built-in throttling classes to limit the number of requests per user or IP address.

## Code

### settings.py
```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
        'rest_framework.throttling.AnonRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '1000/day',
        'anon': '100/day',
    }
}
```

### Custom throttle class (optional)
```python
from rest_framework.throttling import SimpleRateThrottle

class BurstRateThrottle(SimpleRateThrottle):
    scope = 'burst'
    def get_cache_key(self, request, view):
        return self.get_ident(request)
```

## Notes
- Adjust throttle rates to fit your application's needs.
- You can set different rates for different endpoints using custom throttle classes.
- For advanced use cases, consider third-party packages like `django-ratelimit`. 