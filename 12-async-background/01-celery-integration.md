# Using Celery with DRF

## Problem
How do you run background tasks (e.g., sending emails, processing files) asynchronously in a Django REST Framework API?

## Solution
Use Celery, a distributed task queue, to offload long-running or resource-intensive work from your API requests.

## Code

### Install Celery and a message broker (e.g., Redis)
```bash
pip install celery redis
```

### project/celery.py
```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')

app = Celery('project')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

### settings.py (additions)
```python
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

### __init__.py (project root)
```python
from .celery import app as celery_app
__all__ = ('celery_app',)
```

### tasks.py (in your app)
```python
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    # Your email sending logic here
    pass
```

### views.py (triggering a task)
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .tasks import send_welcome_email

class RegisterView(APIView):
    def post(self, request):
        # ... user creation logic ...
        send_welcome_email.delay(user.id)
        return Response({'status': 'User registered, email will be sent.'})
```

## Notes
- Run a Celery worker with: `celery -A project worker -l info`
- Use Redis or RabbitMQ as your broker.
- Celery tasks are executed outside the request/response cycle, so do not access request objects directly in tasks.
- For periodic tasks, use Celery Beat. 