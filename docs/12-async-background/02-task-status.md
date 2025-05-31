# Task Status Endpoint

## Problem
How do you let API clients check the status of a background task (e.g., Celery job) in Django REST Framework?

## Solution
Return the Celery task ID when triggering a job, and provide an endpoint to check the task's status using the ID.

## Code

### views.py (triggering a task)
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .tasks import long_running_task

class StartTaskView(APIView):
    def post(self, request):
        task = long_running_task.delay()
        return Response({'task_id': task.id})
```

### views.py (task status endpoint)
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from celery.result import AsyncResult
from django.conf import settings

class TaskStatusView(APIView):
    def get(self, request, task_id):
        result = AsyncResult(task_id)
        return Response({
            'task_id': task_id,
            'status': result.status,
            'result': result.result if result.ready() else None,
        })
```

### urls.py
```python
from django.urls import path
from .views import StartTaskView, TaskStatusView

urlpatterns = [
    path('start-task/', StartTaskView.as_view(), name='start-task'),
    path('task-status/<str:task_id>/', TaskStatusView.as_view(), name='task-status'),
]
```

## Notes
- The `status` can be `PENDING`, `STARTED`, `SUCCESS`, `FAILURE`, etc.
- Only expose task results that are safe for the client to see.
- For more advanced tracking, store task metadata in your database. 