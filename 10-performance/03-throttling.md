# Throttling Requests

## Problem

You need to protect your API from abuse by limiting the number of requests users can make within a specific time window, implementing different throttling rates for different user types and endpoints.

## Solution

Use DRF's built-in throttling classes and create custom throttling policies to control request rates based on user authentication status, specific users, or endpoints.

## Implementation

### Basic Throttling Setup

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
        'burst': '60/min',
        'sustained': '1000/day',
        'premium': '5000/day',
    }
}
```

### View-Level Throttling

```python
# views.py
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle
from rest_framework.viewsets import ModelViewSet
from rest_framework.decorators import action
from rest_framework.response import Response

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

class ProductViewSet(ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
    
    @action(detail=False, methods=['post'])
    def expensive_operation(self, request):
        # This endpoint has custom throttling
        return Response({'status': 'operation completed'})

class PublicProductViewSet(ModelViewSet):
    queryset = Product.objects.filter(is_public=True)
    serializer_class = ProductSerializer
    throttle_classes = [AnonRateThrottle]
    throttle_scope = 'anon'
```

### Custom Throttling Classes

```python
# throttles.py
from rest_framework.throttling import UserRateThrottle, BaseThrottle
from django.contrib.auth.models import User
from django.core.cache import cache
import time

class PremiumUserThrottle(UserRateThrottle):
    scope = 'premium'
    
    def allow_request(self, request, view):
        # Only apply to premium users
        if hasattr(request.user, 'profile') and request.user.profile.is_premium:
            return super().allow_request(request, view)
        # Non-premium users use default throttling
        return True

class EndpointSpecificThrottle(BaseThrottle):
    """
    Throttle based on specific endpoint patterns
    """
    
    def __init__(self):
        self.rates = {
            'upload': '10/hour',
            'search': '100/hour', 
            'export': '5/hour',
        }
    
    def allow_request(self, request, view):
        endpoint_type = self.get_endpoint_type(request, view)
        if not endpoint_type:
            return True
            
        return self.check_rate_limit(request, endpoint_type)
    
    def get_endpoint_type(self, request, view):
        if hasattr(view, 'action'):
            if view.action in ['upload_file', 'bulk_upload']:
                return 'upload'
            elif view.action in ['search', 'advanced_search']:
                return 'search'
            elif view.action in ['export_csv', 'export_pdf']:
                return 'export'
        return None
    
    def check_rate_limit(self, request, endpoint_type):
        ident = self.get_ident(request)
        cache_key = f'throttle_{endpoint_type}_{ident}'
        
        rate = self.rates.get(endpoint_type)
        if not rate:
            return True
            
        num_requests, period = self.parse_rate(rate)
        now = time.time()
        
        # Get request history
        history = cache.get(cache_key, [])
        
        # Remove old requests
        while history and history[-1] <= now - period:
            history.pop()
        
        # Check if limit exceeded
        if len(history) >= num_requests:
            return False
        
        # Add current request
        history.insert(0, now)
        cache.set(cache_key, history, period)
        return True
    
    def parse_rate(self, rate):
        """Parse rate string like '10/hour' into (10, 3600)"""
        num, period = rate.split('/')
        num = int(num)
        
        periods = {
            'sec': 1,
            'min': 60,
            'hour': 3600,
            'day': 86400
        }
        
        return num, periods[period]
    
    def get_ident(self, request):
        """Get unique identifier for the request"""
        if request.user.is_authenticated:
            return str(request.user.id)
        
        forwarded = request.META.get('HTTP_X_FORWARDED_FOR')
        if forwarded:
            ip = forwarded.split(',')[0].strip()
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip

class ConditionalThrottle(BaseThrottle):
    """
    Apply different throttle rates based on conditions
    """
    
    def allow_request(self, request, view):
        # Different rates for different user types
        if request.user.is_authenticated:
            if hasattr(request.user, 'profile'):
                if request.user.profile.subscription_type == 'enterprise':
                    return self.check_throttle(request, '10000/day')
                elif request.user.profile.subscription_type == 'premium':
                    return self.check_throttle(request, '5000/day')
                else:
                    return self.check_throttle(request, '1000/day')
            else:
                return self.check_throttle(request, '1000/day')
        else:
            return self.check_throttle(request, '100/day')
    
    def check_throttle(self, request, rate):
        # Implement rate checking logic here
        # This is a simplified version
        return True  # Replace with actual implementation
```

### Using Throttling with Function-Based Views

```python
# views.py
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class LoginRateThrottle(UserRateThrottle):
    scope = 'login'

@api_view(['POST'])
@throttle_classes([LoginRateThrottle])
def login_view(request):
    # Login logic here
    return Response({'token': 'abc123'})

@api_view(['POST'])
@throttle_classes([AnonRateThrottle])
def password_reset_view(request):
    # Password reset logic
    return Response({'message': 'Reset email sent'})
```

### Monitoring and Logging Throttle Events

```python
# throttles.py
import logging
from rest_framework.throttling import UserRateThrottle

logger = logging.getLogger('throttling')

class MonitoredUserRateThrottle(UserRateThrottle):
    
    def allow_request(self, request, view):
        allowed = super().allow_request(request, view)
        
        if not allowed:
            logger.warning(
                f'Request throttled for user {request.user.id if request.user.is_authenticated else "anonymous"} '
                f'on endpoint {request.path} from IP {self.get_ident(request)}'
            )
            
            # Optional: Send alert for potential abuse
            if self.is_potential_abuse(request):
                self.send_abuse_alert(request)
        
        return allowed
    
    def is_potential_abuse(self, request):
        # Check if this looks like potential abuse
        ident = self.get_ident(request)
        abuse_key = f'potential_abuse_{ident}'
        
        abuse_count = cache.get(abuse_key, 0)
        abuse_count += 1
        cache.set(abuse_key, abuse_count, 3600)  # 1 hour window
        
        return abuse_count > 10  # More than 10 throttled requests in an hour
    
    def send_abuse_alert(self, request):
        # Send alert to administrators
        from django.core.mail import send_mail
        
        send_mail(
            'Potential API Abuse Detected',
            f'IP {self.get_ident(request)} has been throttled multiple times',
            'admin@yoursite.com',
            ['security@yoursite.com'],
            fail_silently=True
        )
```

### Dynamic Throttling Based on System Load

```python
# throttles.py
import psutil
from rest_framework.throttling import BaseThrottle

class DynamicLoadThrottle(BaseThrottle):
    """
    Adjust throttling based on system load
    """
    
    def allow_request(self, request, view):
        cpu_percent = psutil.cpu_percent(interval=1)
        memory_percent = psutil.virtual_memory().percent
        
        # Adjust throttling based on system resources
        if cpu_percent > 80 or memory_percent > 85:
            # High load - strict throttling
            return self.check_strict_throttle(request)
        elif cpu_percent > 60 or memory_percent > 70:
            # Medium load - normal throttling
            return self.check_normal_throttle(request)
        else:
            # Low load - relaxed throttling
            return self.check_relaxed_throttle(request)
    
    def check_strict_throttle(self, request):
        # Implement strict rate limiting
        return self.check_rate_limit(request, '10/min')
    
    def check_normal_throttle(self, request):
        # Implement normal rate limiting
        return self.check_rate_limit(request, '30/min')
    
    def check_relaxed_throttle(self, request):
        # Implement relaxed rate limiting
        return self.check_rate_limit(request, '60/min')
    
    def check_rate_limit(self, request, rate):
        # Implementation similar to earlier examples
        return True  # Simplified for brevity
```

### Testing Throttling

```python
# tests.py
from django.test import TestCase
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from rest_framework import status
import time

class ThrottlingTestCase(TestCase):
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass'
        )
    
    def test_anonymous_throttling(self):
        """Test that anonymous users are throttled correctly"""
        # Configure throttling for testing
        with self.settings(
            REST_FRAMEWORK={
                'DEFAULT_THROTTLE_CLASSES': [
                    'rest_framework.throttling.AnonRateThrottle',
                ],
                'DEFAULT_THROTTLE_RATES': {
                    'anon': '2/min',
                }
            }
        ):
            url = '/api/products/'
            
            # First two requests should succeed
            response1 = self.client.get(url)
            self.assertEqual(response1.status_code, status.HTTP_200_OK)
            
            response2 = self.client.get(url)
            self.assertEqual(response2.status_code, status.HTTP_200_OK)
            
            # Third request should be throttled
            response3 = self.client.get(url)
            self.assertEqual(response3.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
    
    def test_user_throttling(self):
        """Test that authenticated users have different throttle rates"""
        self.client.force_authenticate(user=self.user)
        
        with self.settings(
            REST_FRAMEWORK={
                'DEFAULT_THROTTLE_CLASSES': [
                    'rest_framework.throttling.UserRateThrottle',
                ],
                'DEFAULT_THROTTLE_RATES': {
                    'user': '5/min',
                }
            }
        ):
            url = '/api/products/'
            
            # Make 5 requests - all should succeed
            for i in range(5):
                response = self.client.get(url)
                self.assertEqual(response.status_code, status.HTTP_200_OK)
            
            # 6th request should be throttled
            response = self.client.get(url)
            self.assertEqual(response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
    
    def test_throttle_headers(self):
        """Test that throttle headers are included in responses"""
        with self.settings(
            REST_FRAMEWORK={
                'DEFAULT_THROTTLE_CLASSES': [
                    'rest_framework.throttling.AnonRateThrottle',
                ],
                'DEFAULT_THROTTLE_RATES': {
                    'anon': '10/min',
                }
            }
        ):
            response = self.client.get('/api/products/')
            
            # Check for throttle headers
            self.assertIn('X-RateLimit-Limit', response)
            self.assertIn('X-RateLimit-Remaining', response)
```

## Notes

### Performance Considerations

1. **Cache Backend**: Use Redis or Memcached for better throttling performance in production
2. **Rate Calculation**: Be mindful of the calculation overhead for complex throttling logic
3. **Database Impact**: Avoid database queries in throttle classes when possible

### Security Best Practices

1. **Layer Defense**: Combine with web server rate limiting (nginx, Apache)
2. **Monitor Patterns**: Log and monitor throttling events for security analysis
3. **Graceful Degradation**: Consider degrading service quality before hard throttling

### Configuration Tips

1. **Environment-Specific**: Use different rates for development, staging, and production
2. **User Feedback**: Provide clear error messages when requests are throttled
3. **Whitelist**: Consider IP whitelisting for trusted sources

### Common Gotchas

1. **Shared Cache**: Ensure proper cache key prefixes in multi-tenant applications
2. **Time Zones**: Be consistent with time calculations across different servers
3. **Testing**: Remember to reset throttle state between tests

This throttling system provides comprehensive protection against API abuse while maintaining flexibility for different user types and usage patterns.
