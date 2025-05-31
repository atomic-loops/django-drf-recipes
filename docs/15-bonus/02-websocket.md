# WebSocket APIs with Channels

## Problem
How do you add real-time WebSocket support to your Django project for features like chat, notifications, or live updates?

## Solution
Use Django Channels to add asynchronous WebSocket handling to your project.

## Code

### Install channels
```bash
pip install channels
```

### settings.py (add to INSTALLED_APPS and ASGI config)
```python
INSTALLED_APPS += [
    'channels',
]
ASGI_APPLICATION = 'project.asgi.application'
```

### asgi.py (project root)
```python
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from django.urls import path
from myapp import consumers

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')

application = ProtocolTypeRouter({
    'http': get_asgi_application(),
    'websocket': AuthMiddlewareStack(
        URLRouter([
            path('ws/chat/', consumers.ChatConsumer.as_asgi()),
        ])
    ),
})
```

### consumers.py (in your app)
```python
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()
    async def disconnect(self, close_code):
        pass
    async def receive(self, text_data):
        data = json.loads(text_data)
        await self.send(text_data=json.dumps({'message': data['message']}))
```

## Notes
- Use Channels for chat, notifications, and other real-time features.
- You need a channel layer backend (e.g., Redis) for production.
- WebSocket endpoints are defined in `asgi.py` and handled by consumers. 