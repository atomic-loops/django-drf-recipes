# Activity Streams

## Problem
How do you implement an activity stream (feed of user or system actions) in a Django REST Framework API?

## Solution
Create an Activity model to log actions, and expose an endpoint to retrieve a user's or global activity feed.

## Code

### models.py
```python
from django.db import models
from django.contrib.auth.models import User

class Activity(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='activities')
    verb = models.CharField(max_length=100)  # e.g., 'created', 'liked', 'commented'
    target = models.CharField(max_length=100, blank=True)  # e.g., 'Post', 'Comment'
    target_id = models.PositiveIntegerField(null=True, blank=True)
    description = models.TextField(blank=True)
    timestamp = models.DateTimeField(auto_now_add=True)
```

### serializers.py
```python
from rest_framework import serializers
from .models import Activity

class ActivitySerializer(serializers.ModelSerializer):
    class Meta:
        model = Activity
        fields = ['id', 'user', 'verb', 'target', 'target_id', 'description', 'timestamp']
```

### views.py
```python
from rest_framework import generics, permissions
from .models import Activity
from .serializers import ActivitySerializer

class UserActivityFeedView(generics.ListAPIView):
    serializer_class = ActivitySerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Activity.objects.filter(user=self.request.user).order_by('-timestamp')

class GlobalActivityFeedView(generics.ListAPIView):
    serializer_class = ActivitySerializer
    permission_classes = [permissions.IsAdminUser]
    queryset = Activity.objects.all().order_by('-timestamp')
```

### urls.py
```python
from django.urls import path
from .views import UserActivityFeedView, GlobalActivityFeedView

urlpatterns = [
    path('activity/', UserActivityFeedView.as_view(), name='user-activity-feed'),
    path('activity/global/', GlobalActivityFeedView.as_view(), name='global-activity-feed'),
]
```

## Notes
- Use signals or custom logic to create Activity records on relevant events (e.g., post created, comment added).
- For more advanced features (following, aggregation), consider using `django-activity-stream`.
- Always paginate activity feeds for performance. 