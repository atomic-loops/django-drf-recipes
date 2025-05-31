# Preventing Overposting

## Problem
How do you prevent users from updating fields they shouldn't (overposting) in Django REST Framework APIs?

## Solution
Explicitly declare allowed fields in your serializers and avoid using `ModelSerializer` with `fields = '__all__'`. Use read-only fields and custom validation as needed.

## Code

### serializers.py
```python
from rest_framework import serializers
from .models import MyModel

class MyModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = MyModel
        fields = ['id', 'name', 'description']  # Only allow these fields
        read_only_fields = ['id']
```

### Example: Preventing mass assignment
```python
# BAD: allows overposting if extra fields are sent
class BadSerializer(serializers.ModelSerializer):
    class Meta:
        model = MyModel
        fields = '__all__'

# GOOD: only allows specific fields
class GoodSerializer(serializers.ModelSerializer):
    class Meta:
        model = MyModel
        fields = ['name', 'description']
```

## Notes
- Never use `fields = '__all__'` in serializers exposed to untrusted clients.
- Use `read_only_fields` for fields that should not be set by the user (e.g., `id`, `created_at`, `owner`).
- Validate user input and permissions in your views as an extra layer of protection. 