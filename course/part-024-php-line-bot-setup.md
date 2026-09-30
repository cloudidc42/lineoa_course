# Part 24: การสร้าง LINE Bot ด้วย PHP

## สารบัญ

1. [Prerequisites](#prerequisites)
2. [การติดตั้ง Composer](#การติดตั้ง-composer)
3. [การติดตั้ง line-bot-sdk-php](#การติดตั้ง-line-bot-sdk-php)
4. [Laravel vs Plain PHP Setup](#laravel-vs-plain-php)
5. [Webhook Handler ใน PHP](#webhook-handler-ใน-php)
6. [Composer Dependencies](#composer-dependencies)
7. [Echo Bot ด้วย Plain PHP](#echo-bot-plain-php)
8. [Echo Bot ด้วย Laravel](#echo-bot-laravel)
9. [Environment Configuration](#environment-config)
10. [Deployment บน Shared Hosting และ VPS](#deployment)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Prerequisites

### ซอฟต์แวร์ที่ต้องมี

| ซอฟต์แวร์ | Version | Download |
|-----------|---------|----------|
| PHP | 8.1+ | https://php.net |
| Composer | 2.x | https://getcomposer.org |
| Git | 2.x+ | https://git-scm.com |
| VS Code | Latest | https://code.visualstudio.com |

**ตรวจสอบการติดตั้ง:**

```bash
# ตรวจสอบ PHP
php --version
# PHP 8.2.0 (cli) (built: Nov 25 2022 20:22:55)

# ตรวจสอบ Composer
composer --version
# Composer version 2.6.5 2023-10-06 10:35:21

# ตรวจสอบ extensions ที่จำเป็น
php -m | grep -E 'curl|json|openssl|mbstring'
```

### PHP Extensions ที่จำเป็น

```bash
# ตรวจสอบ extensions
php -m

# Extensions ที่ต้องมี:
# - curl      : สำหรับ HTTP requests
# - json      : สำหรับ JSON handling
# - openssl   : สำหรับ HTTPS
# - mbstring  : สำหรับ multibyte strings
```

### VS Code Extensions ที่แนะนำ

- **PHP Intelephense**
- **PHP Debug**
- **PHP DocBlocker**
- **Laravel Extension Pack** (ถ้าใช้ Laravel)

---

## 2. การติดตั้ง Composer

### macOS

```bash
# ติดตั้งผ่าน Homebrew
brew install composer

# หรือ manual install
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
sudo mv composer.phar /usr/local/bin/composer
```

### Linux (Ubuntu/Debian)

```bash
# ดาวน์โหลดและติดตั้ง
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
chmod +x /usr/local/bin/composer
```

### Windows

```powershell
# ดาวน์โหลด Composer-Setup.exe จาก https://getcomposer.org/download/
# และรันการติดตั้ง

# ตรวจสอบ
composer --version
```

---

## 3. การติดตั้ง line-bot-sdk-php

### สร้างโปรเจกต์ใหม่

```bash
mkdir my-php-line-bot
cd my-php-line-bot

# เริ่มต้น Composer project
composer init
# กรอกข้อมูลตามที่ถามหรือกด Enter ทั้งหมด
```

### ติดตั้ง LINE Bot SDK

```bash
# ติดตั้ง line-bot-sdk
composer require linecorp/line-bot-sdk

# ติดตั้ง packages เพิ่มเติม
composer require vlucas/phpdotenv          # .env loading
composer require guzzlehttp/guzzle         # HTTP client
composer require monolog/monolog           # Logging
```

### ตรวจสอบ composer.json

```json
{
    "name": "mycompany/line-bot",
    "description": "LINE Bot ด้วย PHP",
    "type": "project",
    "require": {
        "php": ">=8.1",
        "linecorp/line-bot-sdk": "^8.0",
        "vlucas/phpdotenv": "^5.5",
        "guzzlehttp/guzzle": "^7.8",
        "monolog/monolog": "^3.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "squizlabs/php_codesniffer": "*"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    }
}
```

---

## 4. Laravel vs Plain PHP Setup

### เปรียบเทียบ

| Feature | Plain PHP | Laravel |
|---------|-----------|---------|
| ความง่ายในการเริ่ม | ง่ายมาก | ต้องเรียนรู้ |
| Database ORM | ต้องทำเอง | Eloquent (มีให้) |
| Routing | ต้องทำเอง | มี Built-in |
| Queue/Jobs | ต้องทำเอง | มี Built-in |
| Testing | ต้องตั้งค่าเอง | PHPUnit พร้อม |
| Deployment | ง่าย | ง่าย |
| Performance | สูง | ดี |

### โครงสร้าง Plain PHP

```
my-php-line-bot/
├── public/
│   └── webhook.php          # Entry point
├── src/
│   ├── Config.php
│   ├── LineClient.php
│   ├── Handlers/
│   │   ├── MessageHandler.php
│   │   ├── FollowHandler.php
│   │   └── PostbackHandler.php
│   └── Utils/
│       └── Logger.php
├── vendor/                  # Composer packages
├── .env
├── .env.example
├── .gitignore
├── composer.json
└── composer.lock
```

### โครงสร้าง Laravel

```
my-laravel-line-bot/
├── app/
│   ├── Http/
│   │   └── Controllers/
│   │       └── LineWebhookController.php
│   ├── Services/
│   │   └── LineService.php
│   └── Handlers/
│       ├── MessageHandler.php
│       └── FollowHandler.php
├── routes/
│   └── web.php
├── config/
│   └── line.php
├── .env
└── composer.json
```

---

## 5. Webhook Handler ใน PHP

### src/Config.php

```php
<?php

namespace App;

use Dotenv\Dotenv;

class Config
{
    private static ?Config $instance = null;
    private array $config = [];

    private function __construct()
    {
        // โหลด .env
        $dotenv = Dotenv::createImmutable(dirname(__DIR__));
        $dotenv->load();
        
        // Validate required variables
        $dotenv->required([
            'LINE_CHANNEL_ACCESS_TOKEN',
            'LINE_CHANNEL_SECRET',
        ])->notEmpty();
        
        $this->config = [
            'line' => [
                'channel_access_token' => $_ENV['LINE_CHANNEL_ACCESS_TOKEN'],
                'channel_secret' => $_ENV['LINE_CHANNEL_SECRET'],
            ],
            'app' => [
                'debug' => filter_var($_ENV['APP_DEBUG'] ?? false, FILTER_VALIDATE_BOOLEAN),
                'env' => $_ENV['APP_ENV'] ?? 'production',
            ],
        ];
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function get(string $key, mixed $default = null): mixed
    {
        $keys = explode('.', $key);
        $value = $this->config;
        
        foreach ($keys as $k) {
            if (!isset($value[$k])) {
                return $default;
            }
            $value = $value[$k];
        }
        
        return $value;
    }
}
```

### src/Utils/Logger.php

```php
<?php

namespace App\Utils;

use Monolog\Logger as MonologLogger;
use Monolog\Handler\StreamHandler;
use Monolog\Formatter\LineFormatter;

class Logger
{
    private static ?MonologLogger $instance = null;

    public static function getInstance(): MonologLogger
    {
        if (self::$instance === null) {
            $logger = new MonologLogger('linebot');
            
            // Console handler
            $handler = new StreamHandler('php://stdout', MonologLogger::DEBUG);
            $formatter = new LineFormatter(
                "[%datetime%] %channel%.%level_name%: %message% %context%\n",
                "Y-m-d H:i:s"
            );
            $handler->setFormatter($formatter);
            $logger->pushHandler($handler);
            
            // File handler (optional)
            if (defined('LOG_PATH')) {
                $fileHandler = new StreamHandler(LOG_PATH, MonologLogger::INFO);
                $logger->pushHandler($fileHandler);
            }
            
            self::$instance = $logger;
        }
        
        return self::$instance;
    }
}
```

### src/LineClient.php

```php
<?php

namespace App;

use LINE\Clients\MessagingApi\Api\MessagingApiApi;
use LINE\Clients\MessagingApi\Configuration;
use LINE\Clients\MessagingApi\Model\ReplyMessageRequest;
use LINE\Clients\MessagingApi\Model\PushMessageRequest;
use LINE\Clients\MessagingApi\Model\BroadcastMessageRequest;
use LINE\Clients\MessagingApi\Model\TextMessage;
use LINE\Clients\MessagingApi\Model\StickerMessage;
use LINE\Clients\MessagingApi\Model\LocationMessage;
use LINE\Clients\MessagingApi\Model\ImageMessage;
use LINE\Parser\EventRequestParser;
use LINE\Parser\Exception\InvalidSignatureException;
use GuzzleHttp\Client as GuzzleClient;
use App\Utils\Logger;

class LineClient
{
    private static ?LineClient $instance = null;
    private MessagingApiApi $api;
    private string $channelSecret;
    private \Monolog\Logger $logger;

    private function __construct()
    {
        $config = Config::getInstance();
        
        $this->channelSecret = $config->get('line.channel_secret');
        $this->logger = Logger::getInstance();
        
        // สร้าง API Client
        $configuration = new Configuration();
        $configuration->setAccessToken($config->get('line.channel_access_token'));
        
        $this->api = new MessagingApiApi(
            new GuzzleClient(),
            $configuration
        );
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    /**
     * Parse และ Verify Webhook Events
     */
    public function parseWebhookRequest(string $body, string $signature): array
    {
        try {
            $result = EventRequestParser::parseEventRequest(
                $body,
                $this->channelSecret,
                $signature
            );
            return $result->getEvents();
        } catch (InvalidSignatureException $e) {
            throw new \RuntimeException('Invalid LINE signature', 403);
        }
    }

    /**
     * Reply Message
     */
    public function replyMessage(string $replyToken, array $messages): void
    {
        if (count($messages) > 5) {
            throw new \InvalidArgumentException('Cannot send more than 5 messages');
        }

        try {
            $request = new ReplyMessageRequest([
                'replyToken' => $replyToken,
                'messages' => $messages,
            ]);
            
            $this->api->replyMessage($request);
            $this->logger->debug('Reply sent successfully');
        } catch (\Exception $e) {
            $this->logger->error('Failed to reply message', [
                'error' => $e->getMessage(),
                'replyToken' => substr($replyToken, 0, 10) . '...',
            ]);
            throw $e;
        }
    }

    /**
     * Push Message
     */
    public function pushMessage(string $to, array $messages): void
    {
        try {
            $request = new PushMessageRequest([
                'to' => $to,
                'messages' => $messages,
            ]);
            
            $this->api->pushMessage($request);
            $this->logger->debug('Push message sent', ['to' => $to]);
        } catch (\Exception $e) {
            $this->logger->error('Failed to push message', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    /**
     * Broadcast Message
     */
    public function broadcastMessage(array $messages): void
    {
        try {
            $request = new BroadcastMessageRequest([
                'messages' => $messages,
            ]);
            
            $this->api->broadcast($request);
            $this->logger->info('Broadcast sent');
        } catch (\Exception $e) {
            $this->logger->error('Failed to broadcast', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    /**
     * Get User Profile
     */
    public function getUserProfile(string $userId): array
    {
        try {
            $profile = $this->api->getProfile($userId);
            return [
                'userId' => $profile->getUserId(),
                'displayName' => $profile->getDisplayName(),
                'pictureUrl' => $profile->getPictureUrl(),
                'statusMessage' => $profile->getStatusMessage(),
            ];
        } catch (\Exception $e) {
            $this->logger->error('Failed to get profile', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    /**
     * Get Bot Info
     */
    public function getBotInfo(): array
    {
        try {
            $info = $this->api->getBotInfo();
            return [
                'userId' => $info->getUserId(),
                'basicId' => $info->getBasicId(),
                'displayName' => $info->getDisplayName(),
            ];
        } catch (\Exception $e) {
            $this->logger->error('Failed to get bot info', ['error' => $e->getMessage()]);
            throw $e;
        }
    }
}
```

---

## 6. Composer Dependencies

### การจัดการ Dependencies

```bash
# ติดตั้ง dependencies ใหม่
composer install

# อัปเดต dependencies
composer update

# เพิ่ม dependency ใหม่
composer require package/name

# ลบ dependency
composer remove package/name

# แสดง dependencies ที่ติดตั้ง
composer show
```

### composer.json สมบูรณ์

```json
{
    "name": "mycompany/line-bot",
    "description": "LINE Chatbot ด้วย PHP",
    "type": "project",
    "license": "MIT",
    "require": {
        "php": ">=8.1",
        "ext-curl": "*",
        "ext-json": "*",
        "ext-mbstring": "*",
        "ext-openssl": "*",
        "linecorp/line-bot-sdk": "^8.0",
        "vlucas/phpdotenv": "^5.5",
        "guzzlehttp/guzzle": "^7.8",
        "monolog/monolog": "^3.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.5",
        "squizlabs/php_codesniffer": "^3.8",
        "phpstan/phpstan": "^1.10"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "lint": "phpcs src/ --standard=PSR12",
        "analyze": "phpstan analyze src/"
    },
    "config": {
        "optimize-autoloader": true,
        "sort-packages": true
    }
}
```

---

## 7. Echo Bot ด้วย Plain PHP (สมบูรณ์)

### src/Handlers/MessageHandler.php

```php
<?php

namespace App\Handlers;

use LINE\Webhooks\Model\TextMessageContent;
use LINE\Webhooks\Model\ImageMessageContent;
use LINE\Webhooks\Model\StickerMessageContent;
use LINE\Webhooks\Model\LocationMessageContent;
use LINE\Webhooks\Model\VideoMessageContent;
use LINE\Webhooks\Model\FileMessageContent;
use LINE\Clients\MessagingApi\Model\TextMessage;
use LINE\Clients\MessagingApi\Model\StickerMessage;
use LINE\Clients\MessagingApi\Model\LocationMessage;
use App\LineClient;
use App\Utils\Logger;

class MessageHandler
{
    private LineClient $client;
    private \Monolog\Logger $logger;

    public function __construct()
    {
        $this->client = LineClient::getInstance();
        $this->logger = Logger::getInstance();
    }

    public function handle(object $event): void
    {
        $message = $event->getMessage();
        $replyToken = $event->getReplyToken();

        match (true) {
            $message instanceof TextMessageContent => $this->handleText($replyToken, $message),
            $message instanceof ImageMessageContent => $this->handleImage($replyToken, $message),
            $message instanceof StickerMessageContent => $this->handleSticker($replyToken, $message),
            $message instanceof LocationMessageContent => $this->handleLocation($replyToken, $message),
            $message instanceof VideoMessageContent => $this->handleVideo($replyToken),
            $message instanceof FileMessageContent => $this->handleFile($replyToken, $message),
            default => $this->handleUnknown($replyToken),
        };
    }

    private function handleText(string $replyToken, TextMessageContent $message): void
    {
        $text = $message->getText();
        $this->logger->info('Text message received', ['text' => $text]);

        $response = $this->processCommand($text);

        $this->client->replyMessage($replyToken, [
            new TextMessage(['text' => $response]),
        ]);
    }

    private function processCommand(string $text): string
    {
        $text = mb_strtolower(trim($text));

        return match (true) {
            $text === 'help' || $text === 'ช่วยเหลือ' => $this->getHelpText(),
            $text === 'time' || $text === 'เวลา' => $this->getCurrentTime(),
            $text === 'ping' => '🏓 Pong!',
            str_starts_with($text, '/echo ') => mb_substr($text, 6),
            default => "Echo: {$text}",
        };
    }

    private function getHelpText(): string
    {
        return "📋 คำสั่งที่รองรับ:\n" .
               "help / ช่วยเหลือ - แสดงคำสั่ง\n" .
               "time / เวลา - เวลาปัจจุบัน\n" .
               "ping - ทดสอบ\n" .
               "/echo <ข้อความ> - Echo ข้อความ";
    }

    private function getCurrentTime(): string
    {
        $tz = new \DateTimeZone('Asia/Bangkok');
        $now = new \DateTime('now', $tz);
        return '🕐 เวลาปัจจุบัน: ' . $now->format('d/m/Y H:i:s');
    }

    private function handleImage(string $replyToken, ImageMessageContent $message): void
    {
        $this->logger->info('Image received', ['id' => $message->getId()]);
        
        $this->client->replyMessage($replyToken, [
            new TextMessage(['text' => '📷 ขอบคุณสำหรับรูปภาพ!']),
        ]);
    }

    private function handleSticker(string $replyToken, StickerMessageContent $message): void
    {
        // ส่ง Sticker กลับ
        $this->client->replyMessage($replyToken, [
            new StickerMessage([
                'packageId' => '446',
                'stickerId' => '1988',
            ]),
        ]);
    }

    private function handleLocation(string $replyToken, LocationMessageContent $message): void
    {
        $title = $message->getTitle() ?? 'ตำแหน่งที่ส่งมา';
        $address = $message->getAddress() ?? '';

        $this->client->replyMessage($replyToken, [
            new TextMessage([
                'text' => "📍 ได้รับตำแหน่ง:\n{$title}\n{$address}",
            ]),
            new LocationMessage([
                'title' => $title,
                'address' => $address,
                'latitude' => $message->getLatitude(),
                'longitude' => $message->getLongitude(),
            ]),
        ]);
    }

    private function handleVideo(string $replyToken): void
    {
        $this->client->replyMessage($replyToken, [
            new TextMessage(['text' => '🎥 ขอบคุณสำหรับวิดีโอ!']),
        ]);
    }

    private function handleFile(string $replyToken, FileMessageContent $message): void
    {
        $this->client->replyMessage($replyToken, [
            new TextMessage([
                'text' => "📎 ขอบคุณสำหรับไฟล์: {$message->getFileName()}",
            ]),
        ]);
    }

    private function handleUnknown(string $replyToken): void
    {
        $this->client->replyMessage($replyToken, [
            new TextMessage(['text' => 'ขออภัย ฉันยังไม่รองรับข้อความประเภทนี้']),
        ]);
    }
}
```

### src/Handlers/FollowHandler.php

```php
<?php

namespace App\Handlers;

use LINE\Clients\MessagingApi\Model\TextMessage;
use App\LineClient;
use App\Utils\Logger;

class FollowHandler
{
    private LineClient $client;
    private \Monolog\Logger $logger;

    public function __construct()
    {
        $this->client = LineClient::getInstance();
        $this->logger = Logger::getInstance();
    }

    public function handleFollow(object $event): void
    {
        $userId = $event->getSource()->getUserId();
        $replyToken = $event->getReplyToken();

        $this->logger->info('New follower', ['userId' => $userId]);

        // ดึงชื่อผู้ใช้
        $displayName = 'คุณ';
        try {
            $profile = $this->client->getUserProfile($userId);
            $displayName = $profile['displayName'];
        } catch (\Exception $e) {
            $this->logger->warning('Cannot get profile', ['error' => $e->getMessage()]);
        }

        $this->client->replyMessage($replyToken, [
            new TextMessage([
                'text' => "ยินดีต้อนรับ {$displayName}! 🎉\n\nฉันคือ LINE Bot ของเรา",
            ]),
            new TextMessage([
                'text' => "📋 คำสั่งพื้นฐาน:\nhelp - ช่วยเหลือ\ntime - เวลาปัจจุบัน\npeng - ทดสอบ",
            ]),
        ]);
    }

    public function handleUnfollow(object $event): void
    {
        $userId = $event->getSource()->getUserId();
        $this->logger->info('User unfollowed', ['userId' => $userId]);
    }
}
```

### public/webhook.php (Entry Point)

```php
<?php

declare(strict_types=1);

// Autoload
require_once __DIR__ . '/../vendor/autoload.php';

use App\Config;
use App\LineClient;
use App\Handlers\MessageHandler;
use App\Handlers\FollowHandler;
use App\Utils\Logger;
use LINE\Webhooks\Model\MessageEvent;
use LINE\Webhooks\Model\FollowEvent;
use LINE\Webhooks\Model\UnfollowEvent;
use LINE\Webhooks\Model\PostbackEvent;
use LINE\Webhooks\Model\JoinEvent;
use LINE\Webhooks\Model\LeaveEvent;

$logger = Logger::getInstance();

// CORS Headers (ถ้าต้องการ)
header('Content-Type: application/json');

// รับ Request Method
$method = $_SERVER['REQUEST_METHOD'] ?? '';

// Health Check
if ($method === 'GET') {
    echo json_encode([
        'status' => 'ok',
        'message' => 'LINE Bot is running',
        'timestamp' => date('Y-m-d H:i:s'),
    ]);
    exit;
}

// Webhook ต้องเป็น POST
if ($method !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method Not Allowed']);
    exit;
}

// รับ Body และ Signature
$body = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_LINE_SIGNATURE'] ?? '';

if (empty($signature)) {
    $logger->warning('Missing X-Line-Signature header');
    http_response_code(401);
    echo json_encode(['error' => 'Unauthorized']);
    exit;
}

// Parse Events
try {
    $lineClient = LineClient::getInstance();
    $events = $lineClient->parseWebhookRequest($body, $signature);
} catch (\RuntimeException $e) {
    $logger->warning('Signature verification failed');
    http_response_code(403);
    echo json_encode(['error' => 'Forbidden']);
    exit;
} catch (\Exception $e) {
    $logger->error('Failed to parse webhook', ['error' => $e->getMessage()]);
    http_response_code(500);
    echo json_encode(['error' => 'Internal Server Error']);
    exit;
}

// ตอบกลับ 200 ทันที
http_response_code(200);
echo json_encode(['message' => 'OK']);

// ปิด Connection (ให้ PHP ประมวลผลต่อหลัง Response)
if (function_exists('fastcgi_finish_request')) {
    fastcgi_finish_request();
}

$logger->info('Processing events', ['count' => count($events)]);

// Handlers
$messageHandler = new MessageHandler();
$followHandler = new FollowHandler();

// ประมวลผล Events
foreach ($events as $event) {
    try {
        $logger->debug('Processing event', ['type' => get_class($event)]);
        
        match (true) {
            $event instanceof MessageEvent => $messageHandler->handle($event),
            $event instanceof FollowEvent => $followHandler->handleFollow($event),
            $event instanceof UnfollowEvent => $followHandler->handleUnfollow($event),
            $event instanceof PostbackEvent => handlePostback($event, $lineClient, $logger),
            $event instanceof JoinEvent => $logger->info('Bot joined group'),
            $event instanceof LeaveEvent => $logger->info('Bot left group'),
            default => $logger->warning('Unknown event type', ['type' => get_class($event)]),
        };
    } catch (\Exception $e) {
        $logger->error('Error handling event', [
            'type' => get_class($event),
            'error' => $e->getMessage(),
        ]);
    }
}

function handlePostback(object $event, LineClient $client, $logger): void
{
    $data = $event->getPostback()->getData();
    $replyToken = $event->getReplyToken();
    $logger->info('Postback received', ['data' => $data]);
    
    use LINE\Clients\MessagingApi\Model\TextMessage;
    $client->replyMessage($replyToken, [
        new TextMessage(['text' => "Postback: {$data}"]),
    ]);
}
```

---

## 8. Echo Bot ด้วย Laravel

### ติดตั้ง Laravel

```bash
# สร้างโปรเจกต์ Laravel ใหม่
composer create-project laravel/laravel laravel-line-bot

cd laravel-line-bot

# ติดตั้ง LINE SDK
composer require linecorp/line-bot-sdk
```

### config/line.php

```php
<?php

return [
    'channel_access_token' => env('LINE_CHANNEL_ACCESS_TOKEN'),
    'channel_secret' => env('LINE_CHANNEL_SECRET'),
    'webhook_path' => env('LINE_WEBHOOK_PATH', '/webhook/line'),
];
```

### app/Services/LineService.php

```php
<?php

namespace App\Services;

use LINE\Clients\MessagingApi\Api\MessagingApiApi;
use LINE\Clients\MessagingApi\Configuration;
use LINE\Clients\MessagingApi\Model\ReplyMessageRequest;
use LINE\Clients\MessagingApi\Model\PushMessageRequest;
use LINE\Clients\MessagingApi\Model\TextMessage;
use LINE\Clients\MessagingApi\Model\StickerMessage;
use LINE\Parser\EventRequestParser;
use LINE\Parser\Exception\InvalidSignatureException;
use GuzzleHttp\Client;
use Illuminate\Support\Facades\Log;

class LineService
{
    private MessagingApiApi $api;
    private string $channelSecret;

    public function __construct()
    {
        $this->channelSecret = config('line.channel_secret');
        
        $configuration = new Configuration();
        $configuration->setAccessToken(config('line.channel_access_token'));
        
        $this->api = new MessagingApiApi(
            new Client(),
            $configuration
        );
    }

    /**
     * Parse Webhook Request
     */
    public function parseWebhook(string $body, string $signature): array
    {
        try {
            $result = EventRequestParser::parseEventRequest(
                $body,
                $this->channelSecret,
                $signature
            );
            return $result->getEvents();
        } catch (InvalidSignatureException) {
            throw new \RuntimeException('Invalid signature', 403);
        }
    }

    /**
     * Reply Message
     */
    public function reply(string $replyToken, array|object $messages): void
    {
        $messages = is_array($messages) ? $messages : [$messages];
        
        $request = new ReplyMessageRequest([
            'replyToken' => $replyToken,
            'messages' => $messages,
        ]);
        
        try {
            $this->api->replyMessage($request);
        } catch (\Exception $e) {
            Log::error('Failed to reply', ['error' => $e->getMessage()]);
            throw $e;
        }
    }

    /**
     * Push Message
     */
    public function push(string $to, array|object $messages): void
    {
        $messages = is_array($messages) ? $messages : [$messages];
        
        $request = new PushMessageRequest([
            'to' => $to,
            'messages' => $messages,
        ]);
        
        $this->api->pushMessage($request);
    }

    /**
     * Get User Profile
     */
    public function getProfile(string $userId): array
    {
        $profile = $this->api->getProfile($userId);
        
        return [
            'userId' => $profile->getUserId(),
            'displayName' => $profile->getDisplayName(),
            'pictureUrl' => $profile->getPictureUrl(),
        ];
    }
}
```

### app/Http/Controllers/LineWebhookController.php

```php
<?php

namespace App\Http\Controllers;

use App\Services\LineService;
use App\Handlers\MessageHandler;
use App\Handlers\FollowHandler;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Log;
use LINE\Webhooks\Model\MessageEvent;
use LINE\Webhooks\Model\FollowEvent;
use LINE\Webhooks\Model\UnfollowEvent;
use LINE\Webhooks\Model\PostbackEvent;

class LineWebhookController extends Controller
{
    public function __construct(
        private LineService $lineService,
        private MessageHandler $messageHandler,
        private FollowHandler $followHandler,
    ) {}

    public function handle(Request $request): JsonResponse
    {
        $signature = $request->header('X-Line-Signature');
        
        if (!$signature) {
            Log::warning('Missing LINE signature');
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        $body = $request->getContent();

        try {
            $events = $this->lineService->parseWebhook($body, $signature);
        } catch (\RuntimeException $e) {
            Log::warning('Invalid LINE signature');
            return response()->json(['error' => 'Forbidden'], 403);
        }

        Log::info('LINE webhook received', ['eventCount' => count($events)]);

        // ตอบกลับ 200 ก่อน
        $response = response()->json(['message' => 'OK']);

        // ประมวลผล events
        foreach ($events as $event) {
            try {
                $this->processEvent($event);
            } catch (\Exception $e) {
                Log::error('Failed to process event', [
                    'type' => get_class($event),
                    'error' => $e->getMessage(),
                ]);
            }
        }

        return $response;
    }

    private function processEvent(object $event): void
    {
        match (true) {
            $event instanceof MessageEvent => $this->messageHandler->handle($event),
            $event instanceof FollowEvent => $this->followHandler->handleFollow($event),
            $event instanceof UnfollowEvent => $this->followHandler->handleUnfollow($event),
            $event instanceof PostbackEvent => $this->handlePostback($event),
            default => Log::info('Unhandled event', ['type' => get_class($event)]),
        };
    }

    private function handlePostback(PostbackEvent $event): void
    {
        $data = $event->getPostback()->getData();
        Log::info('Postback event', ['data' => $data]);
        
        // ประมวลผล postback data
        $this->lineService->reply(
            $event->getReplyToken(),
            new \LINE\Clients\MessagingApi\Model\TextMessage([
                'text' => "Postback: {$data}",
            ])
        );
    }
}
```

### routes/web.php

```php
<?php

use App\Http\Controllers\LineWebhookController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return response()->json([
        'status' => 'ok',
        'message' => 'LINE Bot is running',
    ]);
});

Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'timestamp' => now()->toISOString(),
    ]);
});

// LINE Webhook - ต้อง disable CSRF verification
Route::post('/webhook/line', [LineWebhookController::class, 'handle']);
```

### Disable CSRF สำหรับ Webhook

```php
// app/Http/Middleware/VerifyCsrfToken.php
<?php

namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

class VerifyCsrfToken extends Middleware
{
    protected $except = [
        '/webhook/line',  // ยกเว้น LINE Webhook
    ];
}
```

---

## 9. Environment Configuration

### .env.example

```env
# Application
APP_NAME="My LINE Bot"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

# LINE Configuration
LINE_CHANNEL_ACCESS_TOKEN=your_token_here
LINE_CHANNEL_SECRET=your_secret_here
LINE_WEBHOOK_PATH=/webhook/line

# Logging
LOG_CHANNEL=stderr
LOG_LEVEL=info
```

### .env (จริง)

```env
APP_NAME="My LINE Bot"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-bot.railway.app

LINE_CHANNEL_ACCESS_TOKEN=eyJhbGciOiJIUzI1NiJ9...
LINE_CHANNEL_SECRET=a1b2c3d4e5f6789012345678901234ab
```

### PHP .htaccess (สำหรับ Apache)

```apache
# public/.htaccess
Options -Indexes
RewriteEngine On

# Force HTTPS
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Route ทุกอย่างไป webhook.php
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ webhook.php [L]
```

### Nginx Configuration

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    root /var/www/my-php-line-bot/public;
    index webhook.php;
    
    location / {
        try_files $uri $uri/ /webhook.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index webhook.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
    
    # ปิด access ไปยังไฟล์ที่สำคัญ
    location ~ /\. {
        deny all;
    }
    
    location ~ /vendor {
        deny all;
    }
}
```

---

## 10. Deployment

### Shared Hosting

**ขั้นตอน:**

1. Upload ไฟล์ด้วย FTP/SFTP
2. ตั้งค่า Document Root ไปที่ `public/`
3. ตั้งค่า Environment Variables (ผ่าน cPanel หรือ `.env`)
4. รัน `composer install --no-dev`
5. ตั้งค่า Webhook URL บน LINE Console

```bash
# บน Shared Hosting (ผ่าน SSH ถ้ามี)
cd /home/username/public_html
git clone https://github.com/your/line-bot.git .
composer install --no-dev --optimize-autoloader
cp .env.example .env
nano .env  # แก้ไข values
```

### VPS Deployment (Ubuntu)

```bash
# ติดตั้ง PHP 8.2
sudo apt update
sudo apt install -y php8.2 php8.2-fpm php8.2-curl php8.2-json \
    php8.2-mbstring php8.2-xml php8.2-zip

# ติดตั้ง Nginx
sudo apt install -y nginx

# ติดตั้ง Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# Clone โปรเจกต์
sudo mkdir -p /var/www/linebot
cd /var/www/linebot
sudo git clone https://github.com/your/line-bot.git .

# ติดตั้ง Dependencies
sudo composer install --no-dev --optimize-autoloader

# ตั้งค่า Permissions
sudo chown -R www-data:www-data /var/www/linebot
sudo chmod -R 755 /var/www/linebot

# ตั้งค่า Environment
sudo cp .env.example .env
sudo nano .env
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM php:8.2-fpm-alpine

# ติดตั้ง extensions
RUN docker-php-ext-install pdo_mysql curl mbstring

# ติดตั้ง Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app

# Copy files
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-scripts

COPY . .

# Permissions
RUN chown -R www-data:www-data /app

EXPOSE 9000

CMD ["php-fpm"]
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  app:
    build: .
    volumes:
      - .:/app
    environment:
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./public:/app/public
    depends_on:
      - app

  certbot:
    image: certbot/certbot
    volumes:
      - ./ssl:/etc/letsencrypt
    command: certonly --webroot -w /var/www/html -d your-domain.com
```

### Deploy บน Railway (PHP)

```bash
# ติดตั้ง Railway CLI
npm install -g @railway/cli

# Login
railway login

# สร้างโปรเจกต์
railway init

# ตั้งค่า Environment Variables
railway variables set LINE_CHANNEL_ACCESS_TOKEN=your_token
railway variables set LINE_CHANNEL_SECRET=your_secret

# Deploy
railway up
```

**Procfile สำหรับ Railway:**
```
web: php -S 0.0.0.0:$PORT -t public/
```

### SSL Certificate ด้วย Let's Encrypt

```bash
# ติดตั้ง Certbot
sudo apt install -y certbot python3-certbot-nginx

# ออก SSL Certificate
sudo certbot --nginx -d your-domain.com

# Auto-renew
sudo crontab -e
# เพิ่มบรรทัดนี้:
0 12 * * * /usr/bin/certbot renew --quiet
```

---

## 11. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Thai Command Bot
สร้าง Bot ที่รองรับคำสั่ง:
- "เวลา" → ส่งเวลาปัจจุบัน
- "วันที่" → ส่งวันที่ปัจจุบัน
- "สวัสดี" → ส่งข้อความต้อนรับ

### แบบฝึกหัดที่ 2: FAQ Bot
สร้าง FAQ Bot:
- อ่านคำถาม-คำตอบจาก Array
- ค้นหาคำตอบด้วย keyword matching
- ส่งคำตอบที่ใกล้เคียงที่สุด

### แบบฝึกหัดที่ 3: Contact Form Bot
สร้าง Bot รับข้อมูลติดต่อ:
- ถามชื่อ อีเมล เบอร์โทร
- บันทึกลง Database (MySQL/SQLite)
- ส่งสรุปกลับ

### แบบฝึกหัดที่ 4: Product Catalog Bot
สร้าง Bot แสดงสินค้า:
- แสดงรายการสินค้า (Flex Message)
- กด Button เพื่อดูรายละเอียด
- รับออเดอร์ผ่าน Bot

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **PHP 8.1+** - Features ใหม่เช่น Match Expression, Named Arguments
2. **Composer** - การจัดการ PHP Dependencies
3. **line-bot-sdk-php** - Official SDK สำหรับ PHP
4. **Plain PHP Webhook** - สร้าง Webhook Server ด้วย PHP ล้วน
5. **Laravel Webhook** - ใช้ Framework สำหรับโปรเจกต์ใหญ่
6. **Config Management** - จัดการ Configuration อย่างถูกต้อง
7. **Deployment** - Shared Hosting, VPS, Docker, Railway
8. **SSL/HTTPS** - Let's Encrypt สำหรับ Production

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ Webhook Events ทุกประเภทอย่างละเอียด!
