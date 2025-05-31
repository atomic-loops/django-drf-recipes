# Throttling

## Problem

You need to limit the rate at which users can make requests to your API to prevent abuse, reduce server load, and ensure fair usage.

## Solution

Use Django REST Framework's throttling classes to control how frequently users can make requests to your API endpoints.

## Code

### 1. Basic Throttling Configuration

First, configure throttling in your settings:

```python
# settings.py
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

This configuration:
- Limits authenticated users to 1000 requests per day
- Limits anonymous users to 100 requests per day

### 2. Built-in Throttling Classes

DRF provides several built-in throttling classes:

#### AnonRateThrottle

Limits the rate of API calls that can be made by anonymous users:

```python
# views.py
from rest_framework.throttling import AnonRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [AnonRateThrottle]
```

#### UserRateThrottle

Limits the rate of API calls that can be made by authenticated users:

```python
# views.py
from rest_framework.throttling import UserRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [UserRateThrottle]
```

#### ScopedRateThrottle

Allows you to use different throttling rates for different parts of your API:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'books': '1000/day',
        'auth': '3/min',
    }
}
```

```python
# views.py
from rest_framework.throttling import ScopedRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'books'

class LoginView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'auth'
    # Rest of the view...
```

### 3. Custom Throttle Classes

You can create custom throttle classes for more specific throttling behavior:

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'
    
class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day',
    }
}
```

```python
# views.py
from .throttles import BurstRateThrottle, SustainedRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```

### 4. Per-Action Throttling

You can apply different throttling classes to different actions in a ViewSet:

```python
# views.py
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_throttles(self):
        if self.action in ['create', 'update', 'partial_update', 'destroy']:
            throttle_classes = [UserRateThrottle]
        else:
            throttle_classes = [AnonRateThrottle]
        return [throttle() for throttle in throttle_classes]
```

### 5. IP-Based Throttling

You can create a custom throttle that limits requests based on IP address:

```python
# throttles.py
from rest_framework.throttling import AnonRateThrottle

class IPRateThrottle(AnonRateThrottle):
    """
    Limits the rate of API calls that can be made by a given IP.
    """
    scope = 'ip'
    
    def get_cache_key(self, request, view):
        ident = self.get_ident(request)
        return self.cache_format % {
            'scope': self.scope,
            'ident': ident
        }
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'ip': '100/day',
    }
}
```

### 6. Tiered Throttling for Different User Types

You can implement different throttling rates for different user types:

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle

class BasicUserRateThrottle(UserRateThrottle):
    scope = 'basic_user'

class PremiumUserRateThrottle(UserRateThrottle):
    scope = 'premium_user'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'basic_user': '100/day',
        'premium_user': '1000/day',
    }
}
```

```python
# views.py
from .throttles import BasicUserRateThrottle, PremiumUserRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_throttles(self):
        user = self.request.user
        if user.is_authenticated and hasattr(user, 'subscription'):
            if user.subscription.plan == 'premium':
                throttle_classes = [PremiumUserRateThrottle]
            else:
                throttle_classes = [BasicUserRateThrottle]
        else:
            throttle_classes = [AnonRateThrottle]
        return [throttle() for throttle in throttle_classes]
```

### 7. Custom Wait Function

You can customize the behavior when a throttle is exceeded:

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle
import random

class RandomizedWaitThrottle(UserRateThrottle):
    """
    A throttle that randomizes the retry-after time to prevent thundering herd problems.
    """
    def wait(self):
        """
        Returns the number of seconds to wait before the next request.
        Adds random jitter to the wait time.
        """
        # Get the standard wait time
        wait_time = super().wait()
        
        if wait_time is None:
            return None
        
        # Add random jitter (±20%)
        jitter = wait_time * 0.2
        return wait_time + random.uniform(-jitter, jitter)
```

### 8. Combining Throttling with Rate Limiting Headers

You can add custom headers to inform clients about rate limits:

```python
# views.py
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class RateLimitedViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [UserRateThrottle]
    
    def list(self, request, *args, **kwargs):
        response = super().list(request, *args, **kwargs)
        
        # Add rate limit headers
        throttle_instance = self.get_throttles()[0]
        if throttle_instance.rate is not None:
            rate_limit = throttle_instance.rate.split('/')[0]
            remaining = throttle_instance.num_requests - throttle_instance.history.count()
            
            response['X-Rate-Limit-Limit'] = rate_limit
            response['X-Rate-Limit-Remaining'] = max(0, remaining)
            
            if throttle_instance.history and throttle_instance.now > throttle_instance.history[-1]:
                reset_time = int(throttle_instance.history[-1] + throttle_instance.duration - throttle_instance.now)
                response['X-Rate-Limit-Reset'] = reset_time
        
        return response
```

### 9. Throttling in Function-Based Views

You can apply throttling to function-based views using the `@throttle_classes` decorator:

```python
# views.py
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
    rate = '1/day'

@api_view(['POST'])
@throttle_classes([OncePerDayUserThrottle])
def daily_action(request):
    """
    A view that can only be accessed once per day by a user.
    """
    return Response({"message": "You've used your daily action!"})
```

### 10. Custom Throttling Backend

You can create a completely custom throttling system:

```python
# throttles.py
from rest_framework.throttling import BaseThrottle
import time
from django.core.cache import cache

class CustomKeyThrottle(BaseThrottle):
    """
    A throttle that uses a custom key for rate limiting.
    """
    cache_format = 'throttle_%(scope)s_%(key)s'
    scope = 'custom'
    
    def get_cache_key(self, request, view):
        # Get a custom key from the request, e.g., API key or other identifier
        custom_key = request.META.get('HTTP_X_API_KEY', '')
        if not custom_key:
            return None
        
        return self.cache_format % {
            'scope': self.scope,
            'key': custom_key
        }
    
    def allow_request(self, request, view):
        # Get the cache key for this request
        cache_key = self.get_cache_key(request, view)
        if not cache_key:
            # No API key, so throttling is not applied
            return True
        
        # Get the current timestamp
        now = time.time()
        
        # Get the history from the cache, or an empty list if it doesn't exist
        history = cache.get(cache_key, [])
        
        # Drop any older than 1 hour (3600 seconds)
        history = [h for h in history if now - h < 3600]
        
        # Check if we've made too many requests
        if len(history) >= 100:  # 100 requests per hour
            return False
        
        # Update the history
        history.append(now)
        cache.set(cache_key, history, 3600)
        
        return True
    
    def wait(self):
        """
        Returns the number of seconds to wait before the next request.
        """
        return 60  # Wait 60 seconds if throttled
```

## Notes

1. **Throttle Rate Formats**:
   - Specify rates as `number/period` where period is one of: `second`, `minute`, `hour`, or `day`
   - Examples: `3/minute`, `60/hour`, `1000/day`
   - You can use multiple throttles with different rates simultaneously

2. **Throttling Scope**:
   - Use different scopes to apply different rates to different parts of your API
   - Define scopes in `DEFAULT_THROTTLE_RATES` in your settings
   - Reference scopes in your throttle classes

3. **Throttling Storage**:
   - By default, throttling uses Django's cache backend
   - Ensure your cache is properly configured for production use
   - For distributed systems, use a shared cache like Redis or Memcached

4. **Response Headers**:
   - When a request is throttled, DRF returns a 429 (Too Many Requests) status code
   - The response includes a `Retry-After` header indicating when to retry
   - Consider adding custom headers with rate limit information

5. **Monitoring and Logging**:
   - Monitor throttled requests to identify potential abuse
   - Log throttling events for analysis
   - Consider notifying users when they approach their rate limits

6. **Performance Considerations**:
   - Throttling adds overhead to request processing
   - Use selective throttling on critical endpoints
   - For high-traffic APIs, consider more efficient throttling implementations

7. **Testing Throttles**:
   - Use Django's `override_settings` to test throttling in unit tests
   - Test both throttled and non-throttled scenarios
   - Verify correct response status codes and headers
