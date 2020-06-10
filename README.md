# 장고로 채팅방 만들기


## usage
python manage.py runserver 0:8000


## 주요 컴포넌트
Channel 은 사람. Group 은 방

```python
from django.urls import re_path

from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer),
]
```

```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing

application = ProtocolTypeRouter({
    # (http->django views is added by default)
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

```python
# chat/consumers.py
import json
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name

        # Join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )

        self.accept()

    def disconnect(self, close_code):
        # Leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    # Receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # Send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )

    # Receive message from room group
    def chat_message(self, event):
        message = event['message']

        # Send message to WebSocket
        self.send(text_data=json.dumps({
            'message': message
        }))
```

### room 에서 webSocket 으로 메시지 보내기
1. html 에서 js function 이 웹소켓으로 메시지 보냄
2. 웹소켓에 연결된 chatConsumer 가 메시지를 받고 이를 group (= room) 에 전파함
 ```
 def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name

        # Join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )

        self.accept()
 
 def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # Send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )
            
            
        )
  ```
 3. 모든 consumer 는 이 메시지를 받을 수 있게 

- When a user posts a message, a JavaScript function will transmit the message over WebSocket to a ChatConsumer. The ChatConsumer will receive that message and forward it to the group corresponding to the room name. Every ChatConsumer in the same group (and thus in the same room) will then receive the message from the group and forward it over WebSocket back to JavaScript, where it will be appended to the chat log.

```javascript
document.querySelector('#chat-message-submit').onclick = function(e) {
        var messageInputDom = document.querySelector('#chat-message-input');
        var message = messageInputDom.value;
        chatSocket.send(JSON.stringify({
            'message': message
        }));

        messageInputDom.value = '';
    };
```


### Several parts of the new ChatConsumer code deserve further explanation:

- self.scope['url_route']['kwargs']['room_name']

  - Obtains the 'room_name' parameter from the URL route in chat/routing.py that opened the WebSocket connection to the consumer.

  - Every consumer has a scope that contains information about its connection, including in particular any positional or keyword arguments from the URL route and the currently authenticated user if any.

- self.room_group_name = 'chat_%s' % self.room_name
  - Constructs a Channels group name directly from the user-specified room name, without any quoting or escaping.
  - Group names may only contain letters, digits, hyphens, and periods. Therefore this example code will fail on room names that have other characters.
- async_to_sync(self.channel_layer.group_add)(...)
  - Joins a group.
  - The async_to_sync(…) wrapper is required because ChatConsumer is a synchronous WebsocketConsumer but it is calling an asynchronous channel layer method. (All channel layer methods are asynchronous.)
Group names are restricted to ASCII alphanumerics, hyphens, and periods only. Since this code constructs a group name directly from the room name, it will fail if the room name contains any characters that aren’t valid in a group name.
- self.accept()
  - Accepts the WebSocket connection.
  - If you do not call accept() within the connect() method then the connection will be rejected and closed. You might want to reject a connection for example because the requesting user is not authorized to perform the requested action.
  - It is recommended that accept() be called as the last action in connect() if you choose to accept the connection.
- async_to_sync(self.channel_layer.group_discard)(...)
  - Leaves a group.
- async_to_sync(self.channel_layer.group_send)
  - Sends an event to a group.
  - An event has a special 'type' key corresponding to the name of the method that should be invoked on consumers that receive the event.