# Audit Trail

## Problem
How do you track changes (create, update, delete) to important models in a Django REST Framework API for auditing purposes?

## Solution
Use an AuditLog model to record changes, and Django signals to automatically log create, update, and delete events.

## Code

### models.py
```python
from django.db import models
from django.contrib.auth.models import User

class AuditLog(models.Model):
    ACTION_CHOICES = (
        ('create', 'Create'),
        ('update', 'Update'),
        ('delete', 'Delete'),
    )
    user = models.ForeignKey(User, null=True, on_delete=models.SET_NULL)
    action = models.CharField(max_length=10, choices=ACTION_CHOICES)
    model = models.CharField(max_length=100)
    object_id = models.PositiveIntegerField()
    changes = models.TextField(blank=True)
    timestamp = models.DateTimeField(auto_now_add=True)
```

### signals.py
```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.contrib.contenttypes.models import ContentType
from .models import AuditLog, MyModel

@receiver(post_save, sender=MyModel)
def log_save(sender, instance, created, **kwargs):
    action = 'create' if created else 'update'
    AuditLog.objects.create(
        user=getattr(instance, 'modified_by', None),  # set this in your views
        action=action,
        model=sender.__name__,
        object_id=instance.pk,
        changes=str(instance.__dict__),
    )

@receiver(post_delete, sender=MyModel)
def log_delete(sender, instance, **kwargs):
    AuditLog.objects.create(
        user=getattr(instance, 'modified_by', None),
        action='delete',
        model=sender.__name__,
        object_id=instance.pk,
        changes=str(instance.__dict__),
    )
```

### serializers.py
```python
from rest_framework import serializers
from .models import AuditLog

class AuditLogSerializer(serializers.ModelSerializer):
    class Meta:
        model = AuditLog
        fields = ['id', 'user', 'action', 'model', 'object_id', 'changes', 'timestamp']
```

### views.py
```python
from rest_framework import generics, permissions
from .models import AuditLog
from .serializers import AuditLogSerializer

class AuditLogListView(generics.ListAPIView):
    queryset = AuditLog.objects.all().order_by('-timestamp')
    serializer_class = AuditLogSerializer
    permission_classes = [permissions.IsAdminUser]
```

### urls.py
```python
from django.urls import path
from .views import AuditLogListView

urlpatterns = [
    path('audit-logs/', AuditLogListView.as_view(), name='audit-log-list'),
]
```

## Notes
- You can use third-party packages like `django-simple-history` or `django-auditlog` for more advanced features.
- Always protect audit log endpoints with strict permissions.
- Store only necessary data in the `changes` field to avoid sensitive data leaks. 