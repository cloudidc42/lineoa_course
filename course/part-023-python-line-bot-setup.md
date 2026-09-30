# Part 23: การสร้าง LINE Bot ด้วย Python

## สารบัญ

1. [Prerequisites](#prerequisites)
2. [Python Virtual Environment](#python-virtual-environment)
3. [การติดตั้ง line-bot-sdk](#การติดตั้ง-line-bot-sdk)
4. [Flask vs FastAPI](#flask-vs-fastapi)
5. [Project Structure](#project-structure)
6. [Flask Webhook Handler](#flask-webhook-handler)
7. [FastAPI Webhook Handler](#fastapi-webhook-handler)
8. [Signature Verification ใน Python](#signature-verification)
9. [Echo Bot ด้วย Flask](#echo-bot-flask)
10. [Echo Bot ด้วย FastAPI](#echo-bot-fastapi)
11. [Environment Variables ด้วย python-dotenv](#environment-variables)
12. [Deploy บน Render/Railway](#deployment)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Prerequisites

### ซอฟต์แวร์ที่ต้องมี

| ซอฟต์แวร์ | Version | Download |
|-----------|---------|----------|
| Python | 3.10+ | https://python.org |
| pip | 22.0+ | (มากับ Python) |
| Git | 2.x+ | https://git-scm.com |
| VS Code | Latest | https://code.visualstudio.com |
| Ngrok | Latest | https://ngrok.com |

**ตรวจสอบการติดตั้ง:**

```bash
# ตรวจสอบ Python
python --version
# Python 3.12.0

# หรือ
python3 --version
# Python 3.12.0

# ตรวจสอบ pip
pip --version
# pip 23.3 from /usr/local/lib/python3.12/site-packages/pip (python 3.12)

# ตรวจสอบ Git
git --version
# git version 2.43.0
```

### VS Code Extensions ที่แนะนำ

- **Python** (Microsoft)
- **Python Indent**
- **Pylance**
- **REST Client** (สำหรับทดสอบ API)
- **GitLens**

---

## 2. Python Virtual Environment

### วิธีสร้าง Virtual Environment

```bash
# สร้างโฟลเดอร์โปรเจกต์
mkdir my-python-line-bot
cd my-python-line-bot

# สร้าง Virtual Environment
python -m venv venv

# Activate Virtual Environment
# macOS/Linux
source venv/bin/activate

# Windows (Command Prompt)
venv\Scripts\activate

# Windows (PowerShell)
venv\Scripts\Activate.ps1
```

**เมื่อ Active แล้วจะเห็น:**
```
(venv) user@machine:~/my-python-line-bot$
```

### การ Deactivate

```bash
deactivate
```

### การใช้ pyenv (แนะนำ)

```bash
# ติดตั้ง pyenv
curl https://pyenv.run | bash

# ติดตั้ง Python version ที่ต้องการ
pyenv install 3.12.0
pyenv local 3.12.0

# ตรวจสอบ
python --version
# Python 3.12.0
```

### .gitignore สำหรับ Python

```gitignore
# Virtual Environment
venv/
.venv/
env/
.env/

# Python cache
__pycache__/
*.py[cod]
*$py.class
*.pyc
*.pyo

# Distribution
dist/
build/
*.egg-info/

# Environment variables
.env
.env.local
.env.*.local

# Logs
*.log
logs/

# OS files
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp

# pytest
.pytest_cache/
htmlcov/
.coverage

# mypy
.mypy_cache/
```

---

## 3. การติดตั้ง line-bot-sdk

### ติดตั้ง Dependencies หลัก

```bash
# ตรวจสอบว่า venv active
(venv) $ pip install line-bot-sdk

# สำหรับ Flask
(venv) $ pip install flask flask-cors

# สำหรับ FastAPI
(venv) $ pip install fastapi uvicorn[standard]

# Environment variables
(venv) $ pip install python-dotenv

# HTTP Client
(venv) $ pip install requests httpx

# Additional utilities
(venv) $ pip install python-dateutil pytz
```

### สร้าง requirements.txt

```bash
# Save dependencies
pip freeze > requirements.txt
```

**requirements.txt (Flask version):**
```
line-bot-sdk==3.11.0
Flask==3.0.0
Flask-Cors==4.0.0
python-dotenv==1.0.0
requests==2.31.0
gunicorn==21.2.0
```

**requirements.txt (FastAPI version):**
```
line-bot-sdk==3.11.0
fastapi==0.109.0
uvicorn[standard]==0.27.0
python-dotenv==1.0.0
httpx==0.26.0
gunicorn==21.2.0
```

### ติดตั้งจาก requirements.txt

```bash
pip install -r requirements.txt
```

---

## 4. Flask vs FastAPI

### เปรียบเทียบ

| Feature | Flask | FastAPI |
|---------|-------|---------|
| ความง่าย | ง่ายมาก | ง่าย |
| Performance | ปานกลาง | สูง (async) |
| Type Hints | ไม่บังคับ | ใช้จริง |
| Auto Docs | ไม่มี | Swagger/ReDoc |
| Async Support | ต้องใช้ extension | Native |
| Community | ใหญ่มาก | กำลังโต |
| Production | ใช้กันมาก | ใช้กันมากขึ้น |

### เมื่อไหรควรเลือกอะไร?

**ใช้ Flask เมื่อ:**
- ต้องการความเรียบง่าย
- Project ขนาดเล็ก
- ทีมคุ้นเคยกับ Flask
- ไม่ต้องการ Async

**ใช้ FastAPI เมื่อ:**
- ต้องการ Performance สูง
- ต้องการ Auto Documentation
- Project ขนาดใหญ่
- ต้องการ Type Safety

---

## 5. Project Structure

### โครงสร้างโปรเจกต์ Flask

```
my-python-line-bot/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── message_handler.py
│   │   ├── follow_handler.py
│   │   └── postback_handler.py
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── webhook.py
│   │   └── health.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── line_service.py
│   └── utils/
│       ├── __init__.py
│       └── logger.py
├── tests/
│   ├── __init__.py
│   ├── test_handlers.py
│   └── test_webhook.py
├── venv/
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
├── Procfile
└── wsgi.py
```

---

## 6. Flask Webhook Handler

### app/config.py

```python
# app/config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    """Application Configuration"""
    
    # LINE Configuration
    LINE_CHANNEL_ACCESS_TOKEN = os.environ.get('LINE_CHANNEL_ACCESS_TOKEN')
    LINE_CHANNEL_SECRET = os.environ.get('LINE_CHANNEL_SECRET')
    
    # Server Configuration
    PORT = int(os.environ.get('PORT', 5000))
    DEBUG = os.environ.get('DEBUG', 'False').lower() == 'true'
    ENV = os.environ.get('FLASK_ENV', 'production')
    
    # Webhook
    WEBHOOK_PATH = os.environ.get('WEBHOOK_PATH', '/webhook')
    
    def __init__(self):
        """Validate required configuration"""
        required = [
            'LINE_CHANNEL_ACCESS_TOKEN',
            'LINE_CHANNEL_SECRET',
        ]
        
        missing = [key for key in required if not getattr(self, key)]
        
        if missing:
            raise ValueError(
                f"Missing required environment variables: {', '.join(missing)}\n"
                f"Please check your .env file."
            )


config = Config()
```

### app/utils/logger.py

```python
# app/utils/logger.py
import logging
import os
import sys
from datetime import datetime


def setup_logger(name: str = __name__) -> logging.Logger:
    """Setup logger with proper formatting"""
    
    logger = logging.getLogger(name)
    
    if logger.handlers:
        return logger
    
    log_level = getattr(logging, os.environ.get('LOG_LEVEL', 'INFO').upper(), logging.INFO)
    logger.setLevel(log_level)
    
    # Console handler
    handler = logging.StreamHandler(sys.stdout)
    handler.setLevel(log_level)
    
    # Formatter
    formatter = logging.Formatter(
        '%(asctime)s [%(levelname)s] %(name)s: %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    
    return logger


logger = setup_logger('linebot')
```

### app/services/line_service.py

```python
# app/services/line_service.py
from linebot.v3.messaging import (
    ApiClient,
    Configuration,
    MessagingApi,
    ReplyMessageRequest,
    PushMessageRequest,
    BroadcastMessageRequest,
    MulticastMessageRequest,
    TextMessage,
    ImageMessage,
    VideoMessage,
    AudioMessage,
    LocationMessage,
    StickerMessage,
    FlexMessage,
    TemplateMessage,
)
from linebot.v3 import WebhookParser
from linebot.v3.exceptions import InvalidSignatureError
from app.config import config
from app.utils.logger import logger
from typing import Union


# สร้าง LINE API Client
configuration = Configuration(
    access_token=config.LINE_CHANNEL_ACCESS_TOKEN
)

api_client = ApiClient(configuration)
line_bot_api = MessagingApi(api_client)

# สร้าง Webhook Parser
parser = WebhookParser(config.LINE_CHANNEL_SECRET)


def verify_signature(body: bytes, signature: str) -> bool:
    """ตรวจสอบ LINE Signature"""
    import hmac
    import hashlib
    import base64
    
    hash_value = hmac.new(
        config.LINE_CHANNEL_SECRET.encode('utf-8'),
        body,
        hashlib.sha256
    ).digest()
    
    expected = base64.b64encode(hash_value).decode('utf-8')
    
    return hmac.compare_digest(expected, signature)


def reply_message(reply_token: str, messages: Union[list, dict]) -> None:
    """Reply ข้อความ"""
    if not isinstance(messages, list):
        messages = [messages]
    
    if len(messages) > 5:
        raise ValueError("Cannot send more than 5 messages at once")
    
    try:
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=messages
            )
        )
        logger.debug(f"Reply sent successfully")
    except Exception as e:
        logger.error(f"Failed to reply message: {e}")
        raise


def push_message(to: str, messages: Union[list, dict]) -> None:
    """Push ข้อความ"""
    if not isinstance(messages, list):
        messages = [messages]
    
    try:
        line_bot_api.push_message(
            PushMessageRequest(
                to=to,
                messages=messages
            )
        )
        logger.debug(f"Push message sent to {to}")
    except Exception as e:
        logger.error(f"Failed to push message: {e}")
        raise


def broadcast_message(messages: Union[list, dict]) -> None:
    """Broadcast ข้อความ"""
    if not isinstance(messages, list):
        messages = [messages]
    
    try:
        line_bot_api.broadcast(
            BroadcastMessageRequest(messages=messages)
        )
        logger.info("Broadcast message sent")
    except Exception as e:
        logger.error(f"Failed to broadcast: {e}")
        raise


def multicast_message(to: list, messages: Union[list, dict]) -> None:
    """Multicast ข้อความ (สูงสุด 500 ผู้รับ)"""
    if not isinstance(messages, list):
        messages = [messages]
    
    # แบ่ง batch ละ 500
    for i in range(0, len(to), 500):
        batch = to[i:i + 500]
        try:
            line_bot_api.multicast(
                MulticastMessageRequest(
                    to=batch,
                    messages=messages
                )
            )
        except Exception as e:
            logger.error(f"Failed to multicast to batch: {e}")
            raise


def get_user_profile(user_id: str) -> dict:
    """ดึงข้อมูล User Profile"""
    try:
        profile = line_bot_api.get_profile(user_id)
        return {
            'userId': profile.user_id,
            'displayName': profile.display_name,
            'pictureUrl': profile.picture_url,
            'statusMessage': profile.status_message,
            'language': profile.language,
        }
    except Exception as e:
        logger.error(f"Failed to get user profile: {e}")
        raise


def get_bot_info() -> dict:
    """ดึงข้อมูล Bot"""
    try:
        info = line_bot_api.get_bot_info()
        return {
            'userId': info.user_id,
            'basicId': info.basic_id,
            'displayName': info.display_name,
            'pictureUrl': info.picture_url,
        }
    except Exception as e:
        logger.error(f"Failed to get bot info: {e}")
        raise
```

### app/handlers/message_handler.py

```python
# app/handlers/message_handler.py
from linebot.v3.webhooks import (
    MessageEvent,
    TextMessageContent,
    ImageMessageContent,
    VideoMessageContent,
    AudioMessageContent,
    FileMessageContent,
    LocationMessageContent,
    StickerMessageContent,
)
from linebot.v3.messaging import (
    TextMessage,
    ImageMessage,
    VideoMessage,
    LocationMessage,
    StickerMessage,
)
from app.services import line_service
from app.utils.logger import logger


def handle_message(event: MessageEvent) -> None:
    """จัดการ Message Events ทุกประเภท"""
    
    message = event.message
    reply_token = event.reply_token
    
    handlers = {
        TextMessageContent: handle_text,
        ImageMessageContent: handle_image,
        VideoMessageContent: handle_video,
        AudioMessageContent: handle_audio,
        FileMessageContent: handle_file,
        LocationMessageContent: handle_location,
        StickerMessageContent: handle_sticker,
    }
    
    handler = handlers.get(type(message))
    
    if handler:
        handler(event, reply_token, message)
    else:
        logger.warning(f"Unknown message type: {type(message)}")
        line_service.reply_message(
            reply_token,
            TextMessage(text='ขออภัย ฉันยังไม่รองรับข้อความประเภทนี้')
        )


def handle_text(event: MessageEvent, reply_token: str, message: TextMessageContent) -> None:
    """จัดการ Text Message"""
    text = message.text
    user_id = event.source.user_id
    
    logger.info(f"Text message from {user_id}: {text}")
    
    # Echo กลับ
    reply_text = process_text_command(text)
    
    line_service.reply_message(
        reply_token,
        TextMessage(text=reply_text)
    )


def process_text_command(text: str) -> str:
    """ประมวลผลคำสั่ง"""
    text_lower = text.lower().strip()
    
    commands = {
        'help': '📋 คำสั่งที่รองรับ:\n/help - ช่วยเหลือ\n/time - เวลาปัจจุบัน\n/echo <ข้อความ> - Echo ข้อความ',
        'สวัสดี': 'สวัสดีครับ! 😊 มีอะไรให้ช่วยไหม?',
        'hello': 'Hello! 👋 How can I help you?',
    }
    
    if text_lower in commands:
        return commands[text_lower]
    
    if text_lower.startswith('/time'):
        from datetime import datetime
        import pytz
        bangkok_tz = pytz.timezone('Asia/Bangkok')
        now = datetime.now(bangkok_tz)
        return f'🕐 เวลาปัจจุบัน: {now.strftime("%d/%m/%Y %H:%M:%S")}'
    
    if text_lower.startswith('/echo '):
        return text[6:]  # ตัด '/echo ' ออก
    
    # Default: Echo
    return f'คุณพิมพ์ว่า: "{text}"'


def handle_image(event: MessageEvent, reply_token: str, message: ImageMessageContent) -> None:
    """จัดการ Image Message"""
    logger.info(f"Image message received: {message.id}")
    
    line_service.reply_message(
        reply_token,
        TextMessage(text='📷 ขอบคุณสำหรับรูปภาพ!\nฉันได้รับแล้ว 😊')
    )


def handle_video(event: MessageEvent, reply_token: str, message: VideoMessageContent) -> None:
    """จัดการ Video Message"""
    line_service.reply_message(
        reply_token,
        TextMessage(text='🎥 ขอบคุณสำหรับวิดีโอ!')
    )


def handle_audio(event: MessageEvent, reply_token: str, message: AudioMessageContent) -> None:
    """จัดการ Audio Message"""
    line_service.reply_message(
        reply_token,
        TextMessage(text='🎵 ขอบคุณสำหรับไฟล์เสียง!')
    )


def handle_file(event: MessageEvent, reply_token: str, message: FileMessageContent) -> None:
    """จัดการ File Message"""
    line_service.reply_message(
        reply_token,
        TextMessage(text=f'📎 ขอบคุณสำหรับไฟล์: {message.file_name}')
    )


def handle_location(event: MessageEvent, reply_token: str, message: LocationMessageContent) -> None:
    """จัดการ Location Message"""
    title = message.title or 'ตำแหน่งที่ส่งมา'
    address = message.address or ''
    
    line_service.reply_message(
        reply_token,
        [
            TextMessage(text=f'📍 ได้รับตำแหน่ง:\n{title}\n{address}'),
            LocationMessage(
                title=title,
                address=address,
                latitude=message.latitude,
                longitude=message.longitude
            )
        ]
    )


def handle_sticker(event: MessageEvent, reply_token: str, message: StickerMessageContent) -> None:
    """จัดการ Sticker Message"""
    line_service.reply_message(
        reply_token,
        StickerMessage(
            package_id='446',
            sticker_id='1988'
        )
    )
```

### app/handlers/follow_handler.py

```python
# app/handlers/follow_handler.py
from linebot.v3.webhooks import FollowEvent, UnfollowEvent
from linebot.v3.messaging import TextMessage
from app.services import line_service
from app.utils.logger import logger


def handle_follow(event: FollowEvent) -> None:
    """จัดการเมื่อมีคนกด Follow"""
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    logger.info(f"New follower: {user_id}")
    
    # ดึงชื่อผู้ใช้
    display_name = 'คุณ'
    try:
        profile = line_service.get_user_profile(user_id)
        display_name = profile['displayName']
    except Exception as e:
        logger.error(f"Cannot get profile: {e}")
    
    # ส่งข้อความต้อนรับ
    line_service.reply_message(
        reply_token,
        [
            TextMessage(
                text=f'ยินดีต้อนรับ {display_name}! 🎉\n\n'
                     f'ฉันคือ LINE Bot ของเรา ยินดีที่ได้รู้จัก!'
            ),
            TextMessage(
                text='📋 สิ่งที่ฉันทำได้:\n'
                     '• ตอบคำถาม\n'
                     '• ให้ข้อมูลสินค้า\n'
                     '• รับออเดอร์\n\n'
                     'พิมพ์ "help" เพื่อดูคำสั่งทั้งหมด'
            )
        ]
    )


def handle_unfollow(event: UnfollowEvent) -> None:
    """จัดการเมื่อมีคน Unfollow"""
    user_id = event.source.user_id
    
    logger.info(f"User unfollowed: {user_id}")
    
    # บันทึกข้อมูลใน Database (ถ้ามี)
    # await db.mark_user_unfollowed(user_id)
```

---

## 7. FastAPI Webhook Handler

### การตั้งค่า FastAPI

```python
# main_fastapi.py
from fastapi import FastAPI, Request, Header, HTTPException
from fastapi.responses import JSONResponse
import asyncio
from linebot.v3 import WebhookParser
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.webhooks import (
    MessageEvent, FollowEvent, UnfollowEvent,
    PostbackEvent, JoinEvent, LeaveEvent,
    MemberJoinedEvent, MemberLeftEvent,
)
from app.config import config
from app.utils.logger import logger
from app.handlers import message_handler, follow_handler, postback_handler

app = FastAPI(
    title="LINE Bot API",
    description="LINE Messaging API Bot",
    version="1.0.0"
)

parser = WebhookParser(config.LINE_CHANNEL_SECRET)


@app.get("/")
async def root():
    return {"message": "LINE Bot is running", "status": "ok"}


@app.get("/health")
async def health_check():
    return {
        "status": "ok",
        "version": "1.0.0"
    }


@app.post("/webhook")
async def webhook(
    request: Request,
    x_line_signature: str = Header(alias="X-Line-Signature", default=None)
):
    """Webhook endpoint สำหรับรับ Events จาก LINE"""
    
    if not x_line_signature:
        raise HTTPException(status_code=401, detail="Missing X-Line-Signature")
    
    body = await request.body()
    body_text = body.decode('utf-8')
    
    # Verify signature
    try:
        events = parser.parse(body_text, x_line_signature)
    except InvalidSignatureError:
        logger.warning("Invalid signature")
        raise HTTPException(status_code=403, detail="Invalid signature")
    
    logger.info(f"Received {len(events)} events")
    
    # Process events asynchronously
    tasks = [handle_event(event) for event in events]
    await asyncio.gather(*tasks, return_exceptions=True)
    
    return JSONResponse(content={"message": "OK"})


async def handle_event(event):
    """Dispatch event to appropriate handler"""
    try:
        if isinstance(event, MessageEvent):
            await asyncio.to_thread(message_handler.handle_message, event)
        elif isinstance(event, FollowEvent):
            await asyncio.to_thread(follow_handler.handle_follow, event)
        elif isinstance(event, UnfollowEvent):
            await asyncio.to_thread(follow_handler.handle_unfollow, event)
        elif isinstance(event, PostbackEvent):
            await asyncio.to_thread(postback_handler.handle_postback, event)
        elif isinstance(event, JoinEvent):
            logger.info(f"Bot joined: {event.source}")
        elif isinstance(event, LeaveEvent):
            logger.info(f"Bot left: {event.source}")
        elif isinstance(event, MemberJoinedEvent):
            logger.info(f"Members joined: {event.joined.members}")
        elif isinstance(event, MemberLeftEvent):
            logger.info(f"Members left: {event.left.members}")
        else:
            logger.warning(f"Unknown event type: {type(event)}")
    except Exception as e:
        logger.error(f"Error handling event {type(event).__name__}: {e}")
```

---

## 8. Signature Verification ใน Python

### การ Verify แบบ Manual

```python
import hmac
import hashlib
import base64


def verify_line_signature(
    channel_secret: str,
    body: bytes,
    signature: str
) -> bool:
    """
    ตรวจสอบ LINE Webhook Signature
    
    Args:
        channel_secret: LINE Channel Secret
        body: Request body bytes
        signature: X-Line-Signature header value
        
    Returns:
        bool: True ถ้า signature ถูกต้อง
    """
    hash_value = hmac.new(
        channel_secret.encode('utf-8'),
        body,
        hashlib.sha256
    ).digest()
    
    expected = base64.b64encode(hash_value).decode('utf-8')
    
    # ใช้ hmac.compare_digest เพื่อป้องกัน Timing Attack
    return hmac.compare_digest(expected, signature)


# ทดสอบ
if __name__ == '__main__':
    channel_secret = 'test_secret'
    body = b'{"events": []}'
    
    # สร้าง signature จำลอง
    import hashlib
    hash_val = hmac.new(
        channel_secret.encode('utf-8'),
        body,
        hashlib.sha256
    ).digest()
    signature = base64.b64encode(hash_val).decode('utf-8')
    
    # ตรวจสอบ
    result = verify_line_signature(channel_secret, body, signature)
    print(f"Valid signature: {result}")  # True
    
    # ตรวจสอบ signature ผิด
    result = verify_line_signature(channel_secret, body, "invalid_signature")
    print(f"Invalid signature: {result}")  # False
```

---

## 9. Echo Bot ด้วย Flask (สมบูรณ์)

```python
# flask_echo_bot.py
import os
import hmac
import hashlib
import base64
import json
from flask import Flask, request, abort, jsonify
from linebot.v3.messaging import (
    ApiClient,
    Configuration,
    MessagingApi,
    ReplyMessageRequest,
    TextMessage,
    StickerMessage,
)
from linebot.v3 import WebhookParser
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.webhooks import (
    MessageEvent,
    FollowEvent,
    UnfollowEvent,
    TextMessageContent,
    ImageMessageContent,
    StickerMessageContent,
)
from dotenv import load_dotenv

# โหลด Environment Variables
load_dotenv()

# สร้าง Flask App
app = Flask(__name__)

# LINE Configuration
channel_access_token = os.environ['LINE_CHANNEL_ACCESS_TOKEN']
channel_secret = os.environ['LINE_CHANNEL_SECRET']

# สร้าง LINE Client
configuration = Configuration(access_token=channel_access_token)
api_client = ApiClient(configuration)
line_bot_api = MessagingApi(api_client)

# สร้าง Webhook Parser
parser = WebhookParser(channel_secret)


@app.route('/')
def index():
    return jsonify({'status': 'LINE Echo Bot is running!'})


@app.route('/health')
def health():
    return jsonify({
        'status': 'ok',
        'message': 'LINE Bot Server is healthy'
    })


@app.route('/webhook', methods=['POST'])
def webhook():
    # รับ Signature Header
    signature = request.headers.get('X-Line-Signature', '')
    
    if not signature:
        app.logger.warning('Missing X-Line-Signature header')
        abort(401)
    
    # รับ Body
    body = request.get_data(as_text=True)
    
    # Parse และ Verify Events
    try:
        events = parser.parse(body, signature)
    except InvalidSignatureError:
        app.logger.warning('Invalid signature')
        abort(403)
    
    app.logger.info(f'Received {len(events)} events')
    
    # Process Events
    for event in events:
        try:
            handle_event(event)
        except Exception as e:
            app.logger.error(f'Error handling event: {e}')
    
    return jsonify({'message': 'OK'})


def handle_event(event):
    """Dispatch event to handler"""
    if isinstance(event, MessageEvent):
        handle_message(event)
    elif isinstance(event, FollowEvent):
        handle_follow(event)
    elif isinstance(event, UnfollowEvent):
        app.logger.info(f'User unfollowed: {event.source.user_id}')
    else:
        app.logger.info(f'Unknown event type: {type(event).__name__}')


def handle_message(event: MessageEvent):
    """จัดการ Message Events"""
    reply_token = event.reply_token
    message = event.message
    
    if isinstance(message, TextMessageContent):
        # Echo text message
        reply_text = f'Echo: {message.text}'
        
        # Handle special commands
        text = message.text.lower().strip()
        if text == 'help':
            reply_text = (
                '📋 คำสั่งที่รองรับ:\n'
                'help - ช่วยเหลือ\n'
                'time - เวลาปัจจุบัน\n'
                'ping - ทดสอบ'
            )
        elif text == 'time':
            from datetime import datetime
            import pytz
            tz = pytz.timezone('Asia/Bangkok')
            now = datetime.now(tz)
            reply_text = f'🕐 {now.strftime("%d/%m/%Y %H:%M:%S")}'
        elif text == 'ping':
            reply_text = '🏓 Pong!'
        
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text=reply_text)]
            )
        )
    
    elif isinstance(message, ImageMessageContent):
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text='📷 ได้รับรูปภาพแล้ว!')]
            )
        )
    
    elif isinstance(message, StickerMessageContent):
        # ส่ง Sticker กลับ
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[
                    StickerMessage(
                        package_id='446',
                        sticker_id='1988'
                    )
                ]
            )
        )
    
    else:
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text='ขออภัย ฉันยังไม่รองรับข้อความประเภทนี้')]
            )
        )


def handle_follow(event: FollowEvent):
    """จัดการ Follow Event"""
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    app.logger.info(f'New follower: {user_id}')
    
    # ดึงชื่อผู้ใช้
    display_name = 'คุณ'
    try:
        profile = line_bot_api.get_profile(user_id)
        display_name = profile.display_name
    except Exception as e:
        app.logger.error(f'Cannot get profile: {e}')
    
    line_bot_api.reply_message(
        ReplyMessageRequest(
            reply_token=reply_token,
            messages=[
                TextMessage(text=f'ยินดีต้อนรับ {display_name}! 🎉'),
                TextMessage(text='พิมพ์ "help" เพื่อดูคำสั่งทั้งหมด')
            ]
        )
    )


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(
        host='0.0.0.0',
        port=port,
        debug=os.environ.get('DEBUG', 'False').lower() == 'true'
    )
```

---

## 10. Echo Bot ด้วย FastAPI (สมบูรณ์)

```python
# fastapi_echo_bot.py
import os
import asyncio
from fastapi import FastAPI, Request, Header, HTTPException
from fastapi.responses import JSONResponse
from linebot.v3 import WebhookParser
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.webhooks import (
    MessageEvent,
    FollowEvent,
    UnfollowEvent,
    TextMessageContent,
    ImageMessageContent,
    StickerMessageContent,
)
from linebot.v3.messaging import (
    ApiClient,
    Configuration,
    MessagingApi,
    ReplyMessageRequest,
    TextMessage,
    StickerMessage,
)
from dotenv import load_dotenv
import logging

# โหลด Environment Variables
load_dotenv()

# Setup Logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# FastAPI App
app = FastAPI(
    title="LINE Echo Bot",
    description="LINE Bot with FastAPI",
    version="1.0.0"
)

# LINE Configuration
channel_access_token = os.environ['LINE_CHANNEL_ACCESS_TOKEN']
channel_secret = os.environ['LINE_CHANNEL_SECRET']

# LINE Client
configuration = Configuration(access_token=channel_access_token)
api_client = ApiClient(configuration)
line_bot_api = MessagingApi(api_client)

# Webhook Parser
parser = WebhookParser(channel_secret)


@app.get("/")
async def root():
    return {"message": "LINE Echo Bot with FastAPI"}


@app.get("/health")
async def health():
    return {"status": "ok", "framework": "FastAPI"}


@app.post("/webhook")
async def webhook(
    request: Request,
    x_line_signature: str = Header(alias="X-Line-Signature", default=None)
):
    """LINE Webhook Endpoint"""
    
    if not x_line_signature:
        raise HTTPException(status_code=401, detail="Missing X-Line-Signature")
    
    # รับ body
    body = await request.body()
    body_text = body.decode('utf-8')
    
    # Parse events
    try:
        events = parser.parse(body_text, x_line_signature)
    except InvalidSignatureError:
        logger.warning("Invalid signature received")
        raise HTTPException(status_code=403, detail="Invalid signature")
    
    logger.info(f"Processing {len(events)} events")
    
    # Process all events concurrently
    tasks = [process_event(event) for event in events]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # Log errors
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            logger.error(f"Event {i} failed: {result}")
    
    return JSONResponse(content={"message": "OK"})


async def process_event(event):
    """Process a single event"""
    # Run synchronous LINE SDK calls in thread pool
    if isinstance(event, MessageEvent):
        await asyncio.to_thread(handle_message, event)
    elif isinstance(event, FollowEvent):
        await asyncio.to_thread(handle_follow, event)
    elif isinstance(event, UnfollowEvent):
        logger.info(f"User unfollowed: {event.source.user_id}")
    else:
        logger.info(f"Unhandled event type: {type(event).__name__}")


def handle_message(event: MessageEvent):
    """Handle message events synchronously"""
    reply_token = event.reply_token
    message = event.message
    
    if isinstance(message, TextMessageContent):
        text = message.text
        
        # Process commands
        response_text = process_command(text)
        
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text=response_text)]
            )
        )
    
    elif isinstance(message, ImageMessageContent):
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text='📷 ได้รับรูปภาพ!')]
            )
        )
    
    elif isinstance(message, StickerMessageContent):
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[StickerMessage(package_id='446', sticker_id='1988')]
            )
        )


def process_command(text: str) -> str:
    """Process text commands"""
    text_lower = text.lower().strip()
    
    if text_lower == 'help':
        return '📋 คำสั่ง:\nhelp, time, ping'
    elif text_lower == 'time':
        from datetime import datetime
        import pytz
        now = datetime.now(pytz.timezone('Asia/Bangkok'))
        return f'🕐 {now.strftime("%H:%M:%S")}'
    elif text_lower == 'ping':
        return '🏓 Pong!'
    else:
        return f'Echo: {text}'


def handle_follow(event: FollowEvent):
    """Handle follow events"""
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    display_name = 'คุณ'
    try:
        profile = line_bot_api.get_profile(user_id)
        display_name = profile.display_name
    except Exception:
        pass
    
    line_bot_api.reply_message(
        ReplyMessageRequest(
            reply_token=reply_token,
            messages=[TextMessage(text=f'ยินดีต้อนรับ {display_name}! 🎉')]
        )
    )


if __name__ == '__main__':
    import uvicorn
    uvicorn.run(
        'fastapi_echo_bot:app',
        host='0.0.0.0',
        port=int(os.environ.get('PORT', 8000)),
        reload=os.environ.get('DEBUG', 'False').lower() == 'true'
    )
```

---

## 11. Environment Variables ด้วย python-dotenv

### .env.example

```env
# LINE Configuration
LINE_CHANNEL_ACCESS_TOKEN=your_token_here
LINE_CHANNEL_SECRET=your_secret_here

# Server
PORT=5000
DEBUG=False
FLASK_ENV=production

# Logging
LOG_LEVEL=INFO

# Database (Optional)
# DATABASE_URL=postgresql://user:pass@localhost/linebot

# Redis (Optional)
# REDIS_URL=redis://localhost:6379
```

### การโหลดและ Validate

```python
# config.py
import os
from typing import Optional
from dotenv import load_dotenv

# โหลด .env
load_dotenv()


class Settings:
    """Application Settings"""
    
    # Required
    LINE_CHANNEL_ACCESS_TOKEN: str = os.environ['LINE_CHANNEL_ACCESS_TOKEN']
    LINE_CHANNEL_SECRET: str = os.environ['LINE_CHANNEL_SECRET']
    
    # Optional with defaults
    PORT: int = int(os.environ.get('PORT', '5000'))
    DEBUG: bool = os.environ.get('DEBUG', 'False').lower() == 'true'
    LOG_LEVEL: str = os.environ.get('LOG_LEVEL', 'INFO').upper()
    
    # Optional
    DATABASE_URL: Optional[str] = os.environ.get('DATABASE_URL')
    REDIS_URL: Optional[str] = os.environ.get('REDIS_URL')


settings = Settings()


# ตรวจสอบค่าที่จำเป็น
def validate_settings():
    """ตรวจสอบ Settings ที่จำเป็น"""
    errors = []
    
    if not settings.LINE_CHANNEL_ACCESS_TOKEN:
        errors.append('LINE_CHANNEL_ACCESS_TOKEN is required')
    
    if not settings.LINE_CHANNEL_SECRET:
        errors.append('LINE_CHANNEL_SECRET is required')
    
    if len(settings.LINE_CHANNEL_SECRET) != 32:
        errors.append('LINE_CHANNEL_SECRET must be 32 characters')
    
    if errors:
        raise ValueError('\n'.join(errors))
    
    return True
```

---

## 12. Deployment

### Deploy บน Render

**render.yaml:**
```yaml
services:
  - type: web
    name: python-line-bot
    env: python
    plan: free
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn -w 2 -b 0.0.0.0:$PORT flask_echo_bot:app
    envVars:
      - key: LINE_CHANNEL_ACCESS_TOKEN
        sync: false
      - key: LINE_CHANNEL_SECRET
        sync: false
      - key: FLASK_ENV
        value: production
```

### Deploy FastAPI บน Render

```yaml
services:
  - type: web
    name: fastapi-line-bot
    env: python
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn fastapi_echo_bot:app --host 0.0.0.0 --port $PORT
```

### Procfile สำหรับ Heroku

**Flask:**
```
web: gunicorn -w 2 flask_echo_bot:app
```

**FastAPI:**
```
web: uvicorn fastapi_echo_bot:app --host 0.0.0.0 --port $PORT
```

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/health || exit 1

# Run with gunicorn
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "flask_echo_bot:app"]
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Thai Quote Bot
สร้าง Bot ที่ส่งคำคมภาษาไทย:
- พิมพ์ "คำคม" รับคำคมแบบ Random
- พิมพ์ "คำคม ชีวิต" รับคำคมเกี่ยวกับชีวิต

### แบบฝึกหัดที่ 2: Unit Converter Bot
สร้าง Bot แปลงหน่วย:
- "100 kg to lbs"
- "30 celsius to fahrenheit"
- "10 km to miles"

### แบบฝึกหัดที่ 3: Weather Bot (Mock)
สร้าง Weather Bot:
- พิมพ์ "อากาศ กรุงเทพ" ดูสภาพอากาศ
- ใช้ข้อมูล Mock หรือ OpenWeatherMap API

### แบบฝึกหัดที่ 4: Todo Bot
สร้าง Bot จัดการ To-do list (ใน memory):
- "เพิ่ม ซื้อของ" - เพิ่ม Task
- "รายการ" - ดูรายการทั้งหมด
- "ลบ 1" - ลบ Task ที่ 1

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Python Virtual Environment** - การจัดการ Dependencies
2. **line-bot-sdk** - SDK ภาษา Python
3. **Flask** - Framework ง่ายสำหรับ Webhook
4. **FastAPI** - Framework ที่มี Performance สูง
5. **Event Handlers** - จัดการ Events ต่างๆ
6. **Signature Verification** - ใช้ hmac ใน Python
7. **Echo Bot** - ตัวอย่างสมบูรณ์ทั้ง 2 Framework
8. **Environment Variables** - python-dotenv
9. **Deployment** - Render/Railway/Heroku/Docker

ในบทต่อไปเราจะเรียนรู้การสร้าง LINE Bot ด้วย PHP!
