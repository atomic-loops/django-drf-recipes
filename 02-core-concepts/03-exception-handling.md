# Exception Handling in DRF

DRF provides robust exception handling for API errors, including validation errors and custom error responses.

## Built-in Exception Handling
- DRF automatically handles common errors (e.g., 404, 400, 403) and returns appropriate responses.
- Use `serializers.ValidationError` for validation logic in serializers.

**Example:**
```python
from rest_framework import serializers

class ExampleSerializer(serializers.Serializer):
    name = serializers.CharField()
    def validate_name(self, value):
        if value == 'forbidden':
            raise serializers.ValidationError('This name is not allowed.')
        return value
```

## Custom Exception Handler
- You can define a custom exception handler to control error response formats.

**Example:**
```python
# api/exception_handlers.py
from rest_framework.views import exception_handler

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    if response is not None and response.status_code == 500:
        response.data = {'detail': 'Internal server error.'}
    return response
```

**To use:**
Add to your `settings.py`:
```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'api.exception_handlers.custom_exception_handler',
}
```

**References in this repo:**
- See `14-security/05-information-leakage.md` and validation in `02-serializers/05-custom-validation.md`. 