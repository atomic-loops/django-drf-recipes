# Request & Response in DRF

Django REST Framework (DRF) provides powerful abstractions for handling HTTP requests and responses in your API views.

## Request Class
- DRF's `Request` extends Django's standard `HttpRequest` and provides flexible parsing of incoming data.
- Access data via `request.data`, `request.query_params`, `request.user`, etc.

**Example:**
```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['POST'])
def example_view(request):
    # Access parsed data
    data = request.data
    user = request.user
    return Response({'received': data, 'user': str(user)})
```

## Response Class
- DRF's `Response` class is used to return API responses with content negotiation and proper status codes.
- Accepts any serializable data (dict, list, etc.) and optional status, headers, etc.

**Example:**
```python
from rest_framework.response import Response
from rest_framework import status

return Response({'message': 'Success'}, status=status.HTTP_200_OK)
```

**References in this repo:**
- See usage in `03-views-viewsets/04-function-based-views.md` and `08-testing/03-api-client.md`. 