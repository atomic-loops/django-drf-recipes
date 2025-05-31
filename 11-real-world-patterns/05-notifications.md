# Notifications API

## Problem
How do you implement a simple notifications system for users in a Django REST Framework API?

## Solution
Create a Notification model, serializer, and views to allow users to retrieve and mark notifications as read.

## Code

### models.py
```python
from django.db import models
from django.contrib.auth.models import User

class Notification(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='notifications')
    message = models.CharField(max_length=255)
    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

### serializers.py
```python
from rest_framework import serializers
from .models import Notification

class NotificationSerializer(serializers.ModelSerializer):
    class Meta:
        model = Notification
        fields = ['id', 'message', 'is_read', 'created_at']
```

### views.py
```python
from rest_framework import generics, permissions
from .models import Notification
from .serializers import NotificationSerializer

class NotificationListView(generics.ListAPIView):
    serializer_class = NotificationSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Notification.objects.filter(user=self.request.user)

class NotificationMarkReadView(generics.UpdateAPIView):
    serializer_class = NotificationSerializer
    permission_classes = [permissions.IsAuthenticated]
    queryset = Notification.objects.all()

    def get_queryset(self):
        return Notification.objects.filter(user=self.request.user)
```

### urls.py
```python
from django.urls import path
from .views import NotificationListView, NotificationMarkReadView

urlpatterns = [
    path('notifications/', NotificationListView.as_view(), name='notifications-list'),
    path('notifications/<int:pk>/read/', NotificationMarkReadView.as_view(), name='notification-mark-read'),
]
```

## Notes
- Extend the Notification model for more complex use cases (e.g., types, links, actions).
- Use signals to create notifications on relevant events (e.g., new message, comment).
- For real-time notifications, consider using Django Channels and WebSockets. 