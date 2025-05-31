# Status Codes in DRF

DRF makes it easy to use and test HTTP status codes in your API.

## Usage
- Use the `rest_framework.status` module for readable status codes.

**Example:**
```python
from rest_framework.response import Response
from rest_framework import status

return Response({'error': 'Not found'}, status=status.HTTP_404_NOT_FOUND)
```

## Common Status Codes
- `status.HTTP_200_OK` (Success)
- `status.HTTP_201_CREATED` (Resource created)
- `status.HTTP_204_NO_CONTENT` (Deleted)
- `status.HTTP_400_BAD_REQUEST` (Validation error)
- `status.HTTP_401_UNAUTHORIZED` (Auth required)
- `status.HTTP_403_FORBIDDEN` (Permission denied)
- `status.HTTP_404_NOT_FOUND` (Not found)
- `status.HTTP_405_METHOD_NOT_ALLOWED` (Wrong HTTP method)
- `status.HTTP_429_TOO_MANY_REQUESTS` (Throttled)

**References in this repo:**
- See usage in `08-testing/03-api-client.md` and `03-views-viewsets/04-function-based-views.md`. 