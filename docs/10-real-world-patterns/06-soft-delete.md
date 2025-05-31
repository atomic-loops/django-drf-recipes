# Soft Delete

## Problem
How do you implement soft delete (marking records as deleted without actually removing them from the database) in a Django REST Framework API?

## Solution
Add an `is_deleted` field to your models and override queryset methods to filter out deleted records. Provide an endpoint to soft-delete objects.

## Code

### models.py
```python
from django.db import models

class SoftDeleteModel(models.Model):
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def delete(self, using=None, keep_parents=False):
        self.is_deleted = True
        from django.utils import timezone
        self.deleted_at = timezone.now()
        self.save()
```

### Example usage in another model
```python
class MyModel(SoftDeleteModel):
    name = models.CharField(max_length=100)
```

### managers.py (optional)
```python
from django.db import models

class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)
```

### Use the manager in your model
```python
class MyModel(SoftDeleteModel):
    name = models.CharField(max_length=100)
    objects = SoftDeleteManager()
    all_objects = models.Manager()  # includes deleted
```

### views.py
```python
from rest_framework import generics, permissions, status
from rest_framework.response import Response
from .models import MyModel
from .serializers import MyModelSerializer

class MyModelDeleteView(generics.DestroyAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
    permission_classes = [permissions.IsAuthenticated]

    def perform_destroy(self, instance):
        instance.delete()
```

## Notes
- Soft delete is useful for audit trails and data recovery.
- Always filter out deleted records in your default queryset.
- You can add endpoints to restore or permanently delete records if needed. 