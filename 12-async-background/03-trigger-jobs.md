# Triggering Jobs from API

## Problem
How do you trigger background jobs (e.g., sending emails, processing data) from a Django REST Framework API endpoint?

## Solution
Call a Celery task (or other background job) from your API view, using `.delay()` to enqueue the job asynchronously.

## Code

### tasks.py
```python
from celery import shared_task

@shared_task
def process_data_task(data_id):
    # Your background processing logic here
    pass
```

### views.py
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .tasks import process_data_task

class ProcessDataView(APIView):
    def post(self, request):
        data_id = request.data.get('data_id')
        process_data_task.delay(data_id)
        return Response({'status': 'Processing started'})
```

### urls.py
```python
from django.urls import path
from .views import ProcessDataView

urlpatterns = [
    path('process-data/', ProcessDataView.as_view(), name='process-data'),
]
```

## Notes
- Always validate and sanitize input before passing to background jobs.
- Use `.delay()` or `.apply_async()` to enqueue Celery tasks.
- For jobs that need to notify the user when done, combine with a task status endpoint or notifications. 