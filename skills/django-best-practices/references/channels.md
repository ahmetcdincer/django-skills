Django Channels reference covering ASGI setup, WebSocket consumers, channel layers, group messaging, authentication, and SSE alternatives.

## 24. Django Channels

### ASGI Setup

**Wrong:**
```python
# asgi.py — using get_asgi_application without routing
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
application = get_asgi_application()
# WebSocket connections will get 404
```

**Correct:**
```python
# pip install channels channels-redis

# config/asgi.py
import os
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
django_asgi_app = get_asgi_application()

from apps.chat import routing  # Import after django setup

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(routing.websocket_urlpatterns)
        )
    ),
})

# settings.py
INSTALLED_APPS = [..., 'channels']
ASGI_APPLICATION = 'config.asgi.application'
```

> **Why:** `ProtocolTypeRouter` splits HTTP and WebSocket traffic. `AuthMiddlewareStack` provides `scope['user']` for WebSocket connections. `AllowedHostsOriginValidator` prevents cross-site WebSocket hijacking.

### WebSocket Consumer

**Wrong:**
```python
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def receive(self, text_data):
        # Processing in the consumer directly — blocks the event loop
        import time
        time.sleep(5)  # Blocks all other connections!
        self.send(text_data=text_data)
```

**Correct:**
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f'chat_{self.room_name}'

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        message = data['message']
        user = self.scope['user']

        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat.message',
                'message': message,
                'username': user.username,
            }
        )

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            'message': event['message'],
            'username': event['username'],
        }))
```

> **Why:** Use `AsyncWebsocketConsumer` to avoid blocking the event loop. Group messaging broadcasts to all room members. The `type` field maps to handler methods (dots become underscores).

### Channel Layers

**Wrong:**
```python
# Using InMemoryChannelLayer in production
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
        # Only works within a single process — no cross-worker communication
    }
}
```

**Correct:**
```python
# pip install channels-redis

# settings.py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [os.environ.get('REDIS_URL', 'redis://127.0.0.1:6379/2')],
            'capacity': 1500,
            'expiry': 10,
        },
    },
}

# Test with InMemoryChannelLayer in tests
# settings/test.py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
    }
}
```

> **Why:** Redis channel layer enables cross-process communication — essential when running multiple ASGI workers. InMemoryChannelLayer is fine for development and testing only.

### Group Messaging

**Wrong:**
```python
# Sending to individual connections manually
class NotificationConsumer(AsyncWebsocketConsumer):
    connected_users = {}  # Global state — breaks with multiple workers

    async def connect(self):
        self.connected_users[self.scope['user'].pk] = self.channel_name
        await self.accept()

    async def send_to_user(self, user_id, message):
        channel = self.connected_users.get(user_id)
        if channel:
            await self.channel_layer.send(channel, {'type': 'notify', 'message': message})
```

**Correct:**
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class NotificationConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']
        if self.user.is_anonymous:
            await self.close()
            return

        # Add user to their personal notification group
        self.group_name = f'notifications_{self.user.pk}'
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        if hasattr(self, 'group_name'):
            await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def notify(self, event):
        await self.send(text_data=json.dumps(event['data']))


# Send notification from anywhere (views, tasks, signals)
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def send_notification(user_id, data):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'notifications_{user_id}',
        {'type': 'notify', 'data': data},
    )
```

> **Why:** Groups work across all workers via Redis. Use per-user groups for targeted notifications. `async_to_sync` lets you send from sync code (views, Celery tasks).

### WebSocket Authentication

**Wrong:**
```python
# No authentication — any anonymous user can connect
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()  # Anyone can connect
```

**Correct:**
```python
# routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path('ws/chat/<str:room_name>/', consumers.ChatConsumer.as_asgi()),
]

# consumers.py
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']

        # Reject anonymous users
        if self.user.is_anonymous:
            await self.close()
            return

        self.room_name = self.scope['url_route']['kwargs']['room_name']
        # Check permission
        if not await self.has_room_access():
            await self.close()
            return

        self.room_group = f'chat_{self.room_name}'
        await self.channel_layer.group_add(self.room_group, self.channel_name)
        await self.accept()

    async def has_room_access(self):
        from channels.db import database_sync_to_async
        @database_sync_to_async
        def check():
            return self.user.chat_rooms.filter(name=self.room_name).exists()
        return await check()
```

> **Why:** `AuthMiddlewareStack` in ASGI config provides `scope['user']` from the session cookie. Always check authentication and authorization in `connect()`. Reject unauthorized connections early.

### HTTP Long Polling Alternative (SSE)

**Wrong:**
```python
# Polling with short interval — wastes bandwidth
# JavaScript: setInterval(() => fetch('/api/notifications/'), 1000)
```

**Correct:**
```python
# For simple real-time without Channels — use Server-Sent Events (SSE)
import asyncio
from django.http import StreamingHttpResponse


async def sse_notifications(request):
    async def event_stream():
        while True:
            # Check for new notifications
            from channels.db import database_sync_to_async

            @database_sync_to_async
            def get_notifications():
                return list(
                    request.user.notifications
                    .filter(is_read=False)
                    .values('id', 'message')[:10]
                )

            notifications = await get_notifications()
            if notifications:
                import json
                data = json.dumps(notifications)
                yield f'data: {data}\n\n'
            await asyncio.sleep(5)

    response = StreamingHttpResponse(event_stream(), content_type='text/event-stream')
    response['Cache-Control'] = 'no-cache'
    return response
```

> **Why:** SSE is simpler than WebSockets for server-to-client streaming. No extra infrastructure needed. Use WebSockets only when you need bidirectional communication.
