# Part 60: LINE Things - IoT Integration กับ LINE

## บทนำ

LINE Things คือ Platform ของ LINE ที่ช่วยให้อุปกรณ์ IoT สามารถเชื่อมต่อกับ LINE ได้โดยตรง ผู้ใช้สามารถควบคุมอุปกรณ์ IoT ผ่าน LINE Chat ได้อย่างง่ายดาย ทั้งยังรับการแจ้งเตือนจากอุปกรณ์ผ่าน LINE ได้เช่นกัน

### ความสามารถหลัก

- เชื่อมต่ออุปกรณ์ BLE (Bluetooth Low Energy) กับ LINE
- ส่งข้อมูลจากอุปกรณ์ไปยัง LINE
- ควบคุมอุปกรณ์ผ่าน LINE Bot
- Scenario-based automation
- Device Link/Unlink Management

---

## 1. LINE Things Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     LINE Things Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐   BLE    ┌─────────────────┐                  │
│  │  IoT Device  │◄────────►│  LINE App        │                  │
│  │ (Arduino/RPi)│          │  (Smartphone)    │                  │
│  └──────┬───────┘          └────────┬────────┘                  │
│         │                           │                            │
│         │ GATT Service              │ LINE Things SDK            │
│         ▼                           ▼                            │
│  ┌──────────────┐         ┌─────────────────┐                   │
│  │  Scenario    │         │  LINE Platform   │                   │
│  │  Engine      │◄───────►│  (Things API)    │                   │
│  └──────────────┘         └────────┬────────┘                   │
│                                    │                             │
│                           ┌────────▼────────┐                   │
│                           │  Webhook/Bot     │                   │
│                           │  Server          │                   │
│                           └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. LINE Things Trial (Development Setup)

### 2.1 สร้าง LINE Things Trial

LINE Things Trial ช่วยให้นักพัฒนาทดสอบ LINE Things ได้โดยไม่ต้องผ่านกระบวนการ Review

```javascript
// config/line-things.js
const lineThingsConfig = {
  // LINE Things Trial Product ID
  productId: process.env.LINE_THINGS_PRODUCT_ID,

  // Service UUID สำหรับ BLE GATT
  serviceUUID: process.env.LINE_THINGS_SERVICE_UUID,

  // Characteristic UUIDs
  characteristics: {
    notify: process.env.LINE_THINGS_NOTIFY_UUID,   // อุปกรณ์ -> LINE
    write: process.env.LINE_THINGS_WRITE_UUID,     // LINE -> อุปกรณ์
    indicate: process.env.LINE_THINGS_INDICATE_UUID
  }
};

module.exports = lineThingsConfig;
```

### 2.2 LINE Bot SDK สำหรับ Things

```bash
npm install @line/bot-sdk axios express dotenv
```

### 2.3 Things Webhook Handler

```javascript
// handlers/things-webhook.js
const line = require('@line/bot-sdk');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class ThingsWebhookHandler {
  /**
   * จัดการ Event ทั้งหมดจาก LINE Things
   */
  async handleEvent(event) {
    console.log('LINE Things Event:', JSON.stringify(event, null, 2));

    switch (event.type) {
      case 'things':
        return this.handleThingsEvent(event);
      case 'message':
        return this.handleMessage(event);
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
  }

  /**
   * จัดการ Things Events (link, unlink, scenario result)
   */
  async handleThingsEvent(event) {
    const { things } = event;

    switch (things.type) {
      case 'link':
        return this.handleDeviceLink(event);
      case 'unlink':
        return this.handleDeviceUnlink(event);
      case 'scenarioResult':
        return this.handleScenarioResult(event);
      default:
        console.log(`Unknown things type: ${things.type}`);
    }
  }

  /**
   * อุปกรณ์เชื่อมต่อสำเร็จ
   */
  async handleDeviceLink(event) {
    const { replyToken, source, things } = event;
    const userId = source.userId;
    const deviceId = things.deviceId;

    console.log(`อุปกรณ์ ${deviceId} เชื่อมต่อกับผู้ใช้ ${userId}`);

    // บันทึกการเชื่อมต่อลงฐานข้อมูล
    await this.saveDeviceLink(userId, deviceId);

    // ส่งข้อความยืนยัน
    await client.replyMessage(replyToken, {
      type: 'flex',
      altText: 'อุปกรณ์เชื่อมต่อสำเร็จ!',
      contents: this.buildDeviceLinkMessage(deviceId)
    });

    return { type: 'link', userId, deviceId };
  }

  /**
   * อุปกรณ์ยกเลิกการเชื่อมต่อ
   */
  async handleDeviceUnlink(event) {
    const { source, things } = event;
    const userId = source.userId;
    const deviceId = things.deviceId;

    console.log(`อุปกรณ์ ${deviceId} ยกเลิกการเชื่อมต่อจากผู้ใช้ ${userId}`);

    // ลบข้อมูลการเชื่อมต่อ
    await this.removeDeviceLink(userId, deviceId);

    // Push แจ้งเตือน (ไม่มี replyToken)
    await client.pushMessage(userId, {
      type: 'text',
      text: `อุปกรณ์ของคุณถูกยกเลิกการเชื่อมต่อแล้ว`
    });

    return { type: 'unlink', userId, deviceId };
  }

  /**
   * รับผลลัพธ์จาก Scenario
   */
  async handleScenarioResult(event) {
    const { source, things } = event;
    const userId = source.userId;
    const result = things.result;
    const actionResults = result.bleNotificationPayload;

    console.log(`Scenario Result จาก ${userId}:`, result);

    // แปลงข้อมูล BLE Payload
    const data = this.parseBlePayload(actionResults);

    // ประมวลผลข้อมูล
    await this.processDeviceData(userId, data);

    return { type: 'scenarioResult', userId, data };
  }

  /**
   * แปลง BLE Payload เป็นข้อมูล
   */
  parseBlePayload(payload) {
    if (!payload) return null;

    // BLE ส่งข้อมูลเป็น Base64
    const buffer = Buffer.from(payload, 'base64');

    return {
      raw: buffer,
      temperature: buffer.readFloatLE(0),
      humidity: buffer.readFloatLE(4),
      timestamp: Date.now()
    };
  }

  async saveDeviceLink(userId, deviceId) {
    const db = require('../database/db');
    db.prepare(`
      INSERT OR REPLACE INTO device_links (user_id, device_id, linked_at)
      VALUES (?, ?, CURRENT_TIMESTAMP)
    `).run(userId, deviceId);
  }

  async removeDeviceLink(userId, deviceId) {
    const db = require('../database/db');
    db.prepare(`
      UPDATE device_links SET unlinked_at = CURRENT_TIMESTAMP
      WHERE user_id = ? AND device_id = ?
    `).run(userId, deviceId);
  }

  async processDeviceData(userId, data) {
    if (!data) return;

    const db = require('../database/db');
    db.prepare(`
      INSERT INTO sensor_readings (user_id, temperature, humidity, recorded_at)
      VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    `).run(userId, data.temperature, data.humidity);

    // ตรวจสอบ Alert conditions
    if (data.temperature > 35) {
      await client.pushMessage(userId, {
        type: 'text',
        text: `⚠️ คำเตือน! อุณหภูมิสูงเกินไป: ${data.temperature.toFixed(1)}°C`
      });
    }
  }

  buildDeviceLinkMessage(deviceId) {
    return {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: '#06C755',
        contents: [{
          type: 'text',
          text: '✅ เชื่อมต่อสำเร็จ!',
          color: '#FFFFFF',
          size: 'xl',
          weight: 'bold'
        }]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'อุปกรณ์ IoT ของคุณเชื่อมต่อกับ LINE แล้ว',
            wrap: true
          },
          {
            type: 'text',
            text: `Device ID: ${deviceId}`,
            size: 'sm',
            color: '#999999',
            margin: 'md'
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'button',
          action: {
            type: 'postback',
            label: 'ดูข้อมูลอุปกรณ์',
            data: `action=deviceStatus&deviceId=${deviceId}`
          },
          style: 'primary',
          color: '#06C755'
        }]
      }
    };
  }

  async handleMessage(event) {
    const { replyToken, message, source } = event;
    const userId = source.userId;
    const text = message.text;

    if (text === 'สถานะ' || text === 'status') {
      return this.sendDeviceStatus(replyToken, userId);
    }

    if (text === 'เปิด' || text === 'on') {
      return this.controlDevice(replyToken, userId, 'on');
    }

    if (text === 'ปิด' || text === 'off') {
      return this.controlDevice(replyToken, userId, 'off');
    }
  }

  async sendDeviceStatus(replyToken, userId) {
    const db = require('../database/db');
    const device = db.prepare(`
      SELECT * FROM device_links WHERE user_id = ? AND unlinked_at IS NULL
    `).get(userId);

    if (!device) {
      return client.replyMessage(replyToken, {
        type: 'text',
        text: 'ยังไม่มีอุปกรณ์ที่เชื่อมต่อ กรุณาเชื่อมต่ออุปกรณ์ก่อน'
      });
    }

    const lastReading = db.prepare(`
      SELECT * FROM sensor_readings WHERE user_id = ?
      ORDER BY recorded_at DESC LIMIT 1
    `).get(userId);

    await client.replyMessage(replyToken, {
      type: 'flex',
      altText: 'สถานะอุปกรณ์',
      contents: this.buildStatusMessage(device, lastReading)
    });
  }

  async controlDevice(replyToken, userId, action) {
    // ส่งคำสั่งไปยังอุปกรณ์ผ่าน LINE Things Scenario
    await client.replyMessage(replyToken, {
      type: 'text',
      text: action === 'on' ? '✅ เปิดอุปกรณ์แล้ว' : '⛔ ปิดอุปกรณ์แล้ว'
    });
  }

  buildStatusMessage(device, reading) {
    return {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '📊 สถานะอุปกรณ์',
            weight: 'bold',
            size: 'lg'
          },
          {
            type: 'separator',
            margin: 'md'
          },
          {
            type: 'box',
            layout: 'vertical',
            margin: 'md',
            spacing: 'sm',
            contents: [
              {
                type: 'box',
                layout: 'baseline',
                contents: [
                  { type: 'text', text: 'Device ID', flex: 1, size: 'sm', color: '#999' },
                  { type: 'text', text: device.device_id, flex: 2, size: 'sm' }
                ]
              },
              reading ? {
                type: 'box',
                layout: 'baseline',
                contents: [
                  { type: 'text', text: 'อุณหภูมิ', flex: 1, size: 'sm', color: '#999' },
                  {
                    type: 'text',
                    text: `${reading.temperature?.toFixed(1) || 'N/A'} °C`,
                    flex: 2,
                    size: 'sm',
                    color: reading.temperature > 30 ? '#FF0000' : '#000000'
                  }
                ]
              } : { type: 'text', text: 'ยังไม่มีข้อมูล', size: 'sm', color: '#999' },
              reading ? {
                type: 'box',
                layout: 'baseline',
                contents: [
                  { type: 'text', text: 'ความชื้น', flex: 1, size: 'sm', color: '#999' },
                  {
                    type: 'text',
                    text: `${reading.humidity?.toFixed(1) || 'N/A'} %`,
                    flex: 2,
                    size: 'sm'
                  }
                ]
              } : { type: 'text', text: '', size: 'sm' }
            ]
          }
        ]
      }
    };
  }
}

module.exports = ThingsWebhookHandler;
```

---

## 3. Arduino BLE สำหรับ LINE Things

### 3.1 Arduino Circuit Diagram Description

```
วงจร Arduino + BLE สำหรับ LINE Things
=====================================

Components:
- Arduino Nano 33 BLE / ESP32 with BLE
- DHT22 (Temperature & Humidity Sensor)
- LED (Status Indicator)
- Push Button
- 10kΩ Resistor (Pull-up for DHT22)
- 220Ω Resistor (Current limiting for LED)

Connections:
┌──────────────────────────────────────────────────────┐
│  Arduino Nano 33 BLE                                  │
│                                                       │
│  3.3V ──────────────── DHT22 Pin 1 (VCC)             │
│  GND  ──────────────── DHT22 Pin 4 (GND)             │
│  D2   ──── 10kΩ ──── DHT22 Pin 2 (DATA)              │
│             │                                         │
│            3.3V (Pull-up)                             │
│                                                       │
│  D13  ── 220Ω ── LED (+) ── LED (-) ── GND           │
│                                                       │
│  D7   ──── Button ──── GND                           │
│         (Internal Pull-up enabled)                    │
└──────────────────────────────────────────────────────┘
```

### 3.2 Arduino Sketch

```cpp
// line_things_ble.ino
#include <ArduinoBLE.h>
#include <DHT.h>

// === Configuration ===
#define DHT_PIN       2
#define DHT_TYPE      DHT22
#define LED_PIN       13
#define BUTTON_PIN    7

// LINE Things Service UUID (ต้องตรงกับที่ตั้งค่าใน LINE Developers Console)
#define SERVICE_UUID          "E625601E-9E55-4597-A598-76018A0D293D"
#define WRITE_CHAR_UUID       "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"
#define NOTIFY_CHAR_UUID      "772AE377-B3D2-4F8E-4042-5481D1E0098C"
#define INDICATE_CHAR_UUID    "0F86F11F-9171-4C55-8A39-7DF0D849E455"

// BLE Service
BLEService lineThingsService(SERVICE_UUID);

// BLE Characteristics
BLECharacteristic writeChar(WRITE_CHAR_UUID,
  BLEWrite | BLEWriteWithoutResponse, 512);
BLECharacteristic notifyChar(NOTIFY_CHAR_UUID,
  BLENotify, 20);
BLECharacteristic indicateChar(INDICATE_CHAR_UUID,
  BLEIndicate, 20);

// Sensors
DHT dht(DHT_PIN, DHT_TYPE);

// State
bool isConnected = false;
unsigned long lastSendTime = 0;
const unsigned long SEND_INTERVAL = 5000; // 5 วินาที

void setup() {
  Serial.begin(9600);

  // ตั้งค่า Pins
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // ตั้งค่า Sensor
  dht.begin();

  // ตั้งค่า BLE
  if (!BLE.begin()) {
    Serial.println("BLE เริ่มต้นล้มเหลว!");
    while (1);
  }

  // ตั้งค่าชื่ออุปกรณ์
  BLE.setLocalName("LINE Things IoT Device");
  BLE.setAdvertisedService(lineThingsService);

  // เพิ่ม Characteristics
  lineThingsService.addCharacteristic(writeChar);
  lineThingsService.addCharacteristic(notifyChar);
  lineThingsService.addCharacteristic(indicateChar);

  // เพิ่ม Service
  BLE.addService(lineThingsService);

  // ตั้งค่า Event Handlers
  writeChar.setEventHandler(BLEWritten, onWriteReceived);
  BLE.setEventHandler(BLEConnected, onBLEConnected);
  BLE.setEventHandler(BLEDisconnected, onBLEDisconnected);

  // เริ่ม Advertising
  BLE.advertise();
  Serial.println("LINE Things Device พร้อมแล้ว - รอการเชื่อมต่อ");

  // LED กะพริบเพื่อแสดงว่าพร้อม
  blinkLED(3);
}

void loop() {
  BLE.poll();

  if (isConnected) {
    // ส่งข้อมูล Sensor ทุก 5 วินาที
    if (millis() - lastSendTime > SEND_INTERVAL) {
      sendSensorData();
      lastSendTime = millis();
    }

    // ตรวจสอบปุ่ม
    if (digitalRead(BUTTON_PIN) == LOW) {
      delay(50); // Debounce
      if (digitalRead(BUTTON_PIN) == LOW) {
        onButtonPress();
        while (digitalRead(BUTTON_PIN) == LOW); // รอปล่อยปุ่ม
      }
    }
  }
}

// === BLE Event Handlers ===

void onBLEConnected(BLEDevice central) {
  isConnected = true;
  Serial.print("เชื่อมต่อกับ: ");
  Serial.println(central.address());
  digitalWrite(LED_PIN, HIGH);
}

void onBLEDisconnected(BLEDevice central) {
  isConnected = false;
  Serial.println("ยกเลิกการเชื่อมต่อ");
  digitalWrite(LED_PIN, LOW);
  BLE.advertise(); // เริ่ม Advertising ใหม่
}

void onWriteReceived(BLEDevice central, BLECharacteristic characteristic) {
  int dataLength = characteristic.valueLength();
  const uint8_t* data = characteristic.value();

  Serial.print("รับข้อมูล: ");
  for (int i = 0; i < dataLength; i++) {
    Serial.print(data[i], HEX);
    Serial.print(" ");
  }
  Serial.println();

  // คำสั่ง: 0x01 = เปิด LED, 0x00 = ปิด LED
  if (dataLength > 0) {
    if (data[0] == 0x01) {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("เปิด LED แล้ว");
    } else if (data[0] == 0x00) {
      digitalWrite(LED_PIN, LOW);
      Serial.println("ปิด LED แล้ว");
    }
  }
}

// === Sensor Functions ===

void sendSensorData() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();

  if (isnan(temperature) || isnan(humidity)) {
    Serial.println("ไม่สามารถอ่านค่า DHT22 ได้");
    return;
  }

  Serial.print("อุณหภูมิ: ");
  Serial.print(temperature);
  Serial.print("°C, ความชื้น: ");
  Serial.print(humidity);
  Serial.println("%");

  // แปลงเป็น BLE Payload (Float = 4 bytes each)
  uint8_t payload[8];
  memcpy(payload, &temperature, 4);
  memcpy(payload + 4, &humidity, 4);

  notifyChar.writeValue(payload, 8);
}

void onButtonPress() {
  Serial.println("ปุ่มถูกกด - ส่งการแจ้งเตือน");

  // ส่ง Alert ไปยัง LINE
  uint8_t alertPayload[2] = {0xFF, 0x01}; // Alert Code
  notifyChar.writeValue(alertPayload, 2);

  blinkLED(2);
}

// === Utility Functions ===

void blinkLED(int times) {
  for (int i = 0; i < times; i++) {
    digitalWrite(LED_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
    delay(200);
  }
}
```

---

## 4. Raspberry Pi สำหรับ LINE Things

### 4.1 Python BLE Server บน Raspberry Pi

```python
# rpi_ble_server.py
"""
Raspberry Pi BLE Server สำหรับ LINE Things
ใช้ BlueZ และ DBus สำหรับ BLE
"""

import asyncio
import struct
import logging
from datetime import datetime
import subprocess
import json

# ติดตั้ง: pip install bleak gpiozero
from bleak import BleakServer
import RPi.GPIO as GPIO

# ตั้งค่า Logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)

# UUIDs
SERVICE_UUID = "E625601E-9E55-4597-A598-76018A0D293D"
NOTIFY_UUID  = "772AE377-B3D2-4F8E-4042-5481D1E0098C"
WRITE_UUID   = "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"

# GPIO Pins
TEMPERATURE_SENSOR_PIN = 4
LED_PIN = 17
BUTTON_PIN = 27

class LineThingsDevice:
    def __init__(self):
        self.is_connected = False
        self.notify_char = None
        self.setup_gpio()

    def setup_gpio(self):
        GPIO.setmode(GPIO.BCM)
        GPIO.setup(LED_PIN, GPIO.OUT)
        GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
        GPIO.output(LED_PIN, GPIO.LOW)
        GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING,
                               callback=self.on_button_press,
                               bouncetime=300)
        logger.info("GPIO ตั้งค่าแล้ว")

    def on_button_press(self, channel):
        logger.info("ปุ่มถูกกด!")
        if self.is_connected and self.notify_char:
            asyncio.create_task(self.send_button_event())

    async def send_button_event(self):
        """ส่ง Alert เมื่อปุ่มถูกกด"""
        payload = struct.pack('<BB', 0xFF, 0x01)
        try:
            await self.notify_char.write_value(list(payload))
            logger.info("ส่ง Button Event แล้ว")
        except Exception as e:
            logger.error(f"ส่ง Button Event ล้มเหลว: {e}")

    def read_temperature(self):
        """อ่านอุณหภูมิจาก DS18B20"""
        try:
            result = subprocess.run(
                ['cat', f'/sys/bus/w1/devices/28-*/w1_slave'],
                capture_output=True,
                text=True,
                shell=True
            )
            if 'YES' in result.stdout:
                lines = result.stdout.strip().split('\n')
                temp_str = lines[1].split('t=')[1]
                return float(temp_str) / 1000.0
        except Exception as e:
            logger.error(f"อ่านอุณหภูมิไม่ได้: {e}")
        return 0.0

    def read_humidity(self):
        """อ่านความชื้น (simulation)"""
        import random
        return random.uniform(40, 80)

    async def sensor_loop(self):
        """ส่งข้อมูล Sensor ทุก 5 วินาที"""
        while True:
            if self.is_connected and self.notify_char:
                temperature = self.read_temperature()
                humidity = self.read_humidity()

                # Pack เป็น Binary: 4 bytes temp + 4 bytes humidity
                payload = struct.pack('<ff', temperature, humidity)

                try:
                    await self.notify_char.write_value(list(payload))
                    logger.info(f"ส่งข้อมูล: {temperature:.1f}°C, {humidity:.1f}%")
                except Exception as e:
                    logger.error(f"ส่งข้อมูลล้มเหลว: {e}")

            await asyncio.sleep(5)

    async def on_write(self, value):
        """จัดการคำสั่งจาก LINE"""
        logger.info(f"รับคำสั่ง: {value}")

        if len(value) > 0:
            command = value[0]
            if command == 0x01:
                GPIO.output(LED_PIN, GPIO.HIGH)
                logger.info("เปิด LED")
            elif command == 0x00:
                GPIO.output(LED_PIN, GPIO.LOW)
                logger.info("ปิด LED")

    def cleanup(self):
        GPIO.cleanup()
        logger.info("GPIO Cleanup แล้ว")


device = LineThingsDevice()
```

---

## 5. Scenario GATT Service

### 5.1 Scenario Definition (JSON)

```json
{
  "autoLinkService": {
    "serviceUuid": "E625601E-9E55-4597-A598-76018A0D293D"
  },
  "scenario": {
    "id": "sensor-reading-scenario",
    "name": "อ่านข้อมูล Sensor",
    "trigger": {
      "type": "autoCondition",
      "autoConditionParameters": {
        "bleNotificationParameters": [
          {
            "serviceUuid": "E625601E-9E55-4597-A598-76018A0D293D",
            "characteristicUuid": "772AE377-B3D2-4F8E-4042-5481D1E0098C"
          }
        ]
      }
    },
    "condition": {
      "type": "none"
    },
    "actions": [
      {
        "type": "bleAction",
        "bleActionParameters": {
          "type": "notify",
          "serviceUuid": "E625601E-9E55-4597-A598-76018A0D293D",
          "characteristicUuid": "772AE377-B3D2-4F8E-4042-5481D1E0098C"
        }
      }
    ]
  }
}
```

### 5.2 Scenario Result Handler

```javascript
// handlers/scenario-result-handler.js

class ScenarioResultHandler {
  /**
   * จัดการผลลัพธ์จาก Scenario
   * @param {Object} event - LINE Things Scenario Result Event
   */
  async handle(event) {
    const { source, things } = event;
    const userId = source.userId;
    const result = things.result;

    // แปลงข้อมูลจาก BLE Notification
    const payload = result.bleNotificationPayload;

    if (!payload) {
      console.log('ไม่มี BLE Payload');
      return;
    }

    const buffer = Buffer.from(payload, 'base64');
    const dataType = this.detectDataType(buffer);

    switch (dataType) {
      case 'sensor':
        return this.handleSensorData(userId, buffer);
      case 'alert':
        return this.handleAlert(userId, buffer);
      case 'status':
        return this.handleStatusUpdate(userId, buffer);
      default:
        console.log(`ไม่รู้จัก Data Type: ${dataType}`);
    }
  }

  detectDataType(buffer) {
    if (buffer.length >= 2 && buffer[0] === 0xFF) {
      return 'alert';
    }
    if (buffer.length === 8) {
      return 'sensor'; // Float (4) + Float (4)
    }
    if (buffer.length === 1) {
      return 'status';
    }
    return 'unknown';
  }

  async handleSensorData(userId, buffer) {
    const temperature = buffer.readFloatLE(0);
    const humidity = buffer.readFloatLE(4);

    console.log(`Sensor Data - User: ${userId}, Temp: ${temperature.toFixed(1)}°C, Humidity: ${humidity.toFixed(1)}%`);

    // บันทึกข้อมูล
    await this.saveSensorReading(userId, temperature, humidity);

    // ตรวจสอบ Threshold
    await this.checkThresholds(userId, temperature, humidity);

    return { temperature, humidity };
  }

  async handleAlert(userId, buffer) {
    const alertCode = buffer[1];
    const alertMessages = {
      0x01: 'ปุ่ม Emergency ถูกกด',
      0x02: 'อุณหภูมิสูงเกินกำหนด',
      0x03: 'แบตเตอรี่ต่ำ',
      0x04: 'ประตูเปิด'
    };

    const message = alertMessages[alertCode] || `Alert Code: ${alertCode}`;
    console.log(`Alert - User: ${userId}, Message: ${message}`);

    const lineClient = require('../config/line-client');
    await lineClient.pushMessage(userId, {
      type: 'text',
      text: `🚨 แจ้งเตือน: ${message}`
    });

    return { alert: message };
  }

  async handleStatusUpdate(userId, buffer) {
    const status = buffer[0] === 0x01 ? 'online' : 'offline';
    console.log(`Status Update - User: ${userId}, Status: ${status}`);
    return { status };
  }

  async saveSensorReading(userId, temperature, humidity) {
    const db = require('../database/db');
    db.prepare(`
      INSERT INTO sensor_readings (user_id, temperature, humidity, recorded_at)
      VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    `).run(userId, temperature, humidity);
  }

  async checkThresholds(userId, temperature, humidity) {
    const lineClient = require('../config/line-client');
    const alerts = [];

    if (temperature > 35) {
      alerts.push(`🌡️ อุณหภูมิสูงมาก: ${temperature.toFixed(1)}°C`);
    }
    if (temperature < 10) {
      alerts.push(`🥶 อุณหภูมิต่ำมาก: ${temperature.toFixed(1)}°C`);
    }
    if (humidity > 80) {
      alerts.push(`💧 ความชื้นสูงมาก: ${humidity.toFixed(1)}%`);
    }

    if (alerts.length > 0) {
      await lineClient.pushMessage(userId, {
        type: 'text',
        text: `⚠️ คำเตือน!\n${alerts.join('\n')}`
      });
    }
  }
}

module.exports = ScenarioResultHandler;
```

---

## 6. Real-World Projects

### 6.1 Project 1: Smart Home Control

```javascript
// projects/smart-home/smart-home-controller.js

class SmartHomeController {
  constructor(lineClient, mqttClient) {
    this.line = lineClient;
    this.mqtt = mqttClient;
    this.devices = new Map();
  }

  /**
   * ลงทะเบียนอุปกรณ์
   */
  registerDevice(deviceId, config) {
    this.devices.set(deviceId, {
      id: deviceId,
      name: config.name,
      type: config.type, // 'light', 'fan', 'ac', 'lock'
      topic: config.topic,
      status: 'off'
    });
  }

  /**
   * สั่งการอุปกรณ์
   */
  async controlDevice(userId, deviceId, command) {
    const device = this.devices.get(deviceId);
    if (!device) {
      await this.line.pushMessage(userId, {
        type: 'text',
        text: `ไม่พบอุปกรณ์ ID: ${deviceId}`
      });
      return;
    }

    // ส่งคำสั่งผ่าน MQTT
    this.mqtt.publish(device.topic, JSON.stringify({
      command,
      timestamp: Date.now()
    }));

    device.status = command;

    await this.line.pushMessage(userId, {
      type: 'flex',
      altText: `${device.name} ${command === 'on' ? 'เปิดแล้ว' : 'ปิดแล้ว'}`,
      contents: this.buildDeviceControlMessage(device, command)
    });
  }

  /**
   * แสดงสถานะบ้านทั้งหมด
   */
  async showHomeStatus(replyToken) {
    const deviceList = Array.from(this.devices.values());

    await this.line.replyMessage(replyToken, {
      type: 'flex',
      altText: 'สถานะบ้านอัจฉริยะ',
      contents: {
        type: 'carousel',
        contents: deviceList.map(device => this.buildDeviceCard(device))
      }
    });
  }

  buildDeviceCard(device) {
    const icons = {
      light: '💡',
      fan: '🌀',
      ac: '❄️',
      lock: '🔒'
    };

    return {
      type: 'bubble',
      size: 'micro',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: device.status === 'on' ? '#06C755' : '#CCCCCC',
        contents: [{
          type: 'text',
          text: `${icons[device.type] || '🔌'} ${device.name}`,
          color: '#FFFFFF',
          weight: 'bold'
        }]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'text',
          text: device.status === 'on' ? '🟢 เปิดอยู่' : '🔴 ปิดอยู่',
          align: 'center',
          size: 'sm'
        }]
      },
      footer: {
        type: 'box',
        layout: 'horizontal',
        spacing: 'sm',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: 'เปิด',
              data: `action=deviceOn&deviceId=${device.id}`
            },
            style: 'primary',
            color: '#06C755',
            flex: 1
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: 'ปิด',
              data: `action=deviceOff&deviceId=${device.id}`
            },
            style: 'secondary',
            flex: 1
          }
        ]
      }
    };
  }

  buildDeviceControlMessage(device, command) {
    return {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: command === 'on' ? '✅ เปิดสำเร็จ' : '⛔ ปิดสำเร็จ',
            weight: 'bold',
            size: 'xl',
            align: 'center'
          },
          {
            type: 'text',
            text: device.name,
            align: 'center',
            margin: 'md',
            color: '#666'
          }
        ]
      }
    };
  }
}

module.exports = SmartHomeController;
```

### 6.2 Project 2: Air Quality Monitor

```javascript
// projects/air-quality/air-quality-monitor.js

class AirQualityMonitor {
  constructor(lineClient, db) {
    this.line = lineClient;
    this.db = db;
    this.alertThresholds = {
      pm25: { good: 12, moderate: 35.4, unhealthy: 55.4, veryUnhealthy: 150.4 },
      co2: { good: 1000, moderate: 2000, unhealthy: 5000 },
      temperature: { min: 18, max: 30 },
      humidity: { min: 40, max: 70 }
    };
  }

  /**
   * รับข้อมูลคุณภาพอากาศจากอุปกรณ์
   */
  async processAirQualityData(userId, data) {
    // บันทึกข้อมูล
    this.db.prepare(`
      INSERT INTO air_quality_readings
      (user_id, pm25, co2, temperature, humidity, aqi, recorded_at)
      VALUES (?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)
    `).run(
      userId,
      data.pm25,
      data.co2,
      data.temperature,
      data.humidity,
      this.calculateAQI(data.pm25)
    );

    // ตรวจสอบและส่งการแจ้งเตือน
    const alerts = this.checkAlerts(data);
    if (alerts.length > 0) {
      await this.sendAirQualityAlert(userId, data, alerts);
    }
  }

  /**
   * คำนวณ AQI จาก PM2.5
   */
  calculateAQI(pm25) {
    if (pm25 <= 12) return Math.round((50/12) * pm25);
    if (pm25 <= 35.4) return Math.round(50 + (50/23.4) * (pm25 - 12));
    if (pm25 <= 55.4) return Math.round(100 + (50/20) * (pm25 - 35.4));
    if (pm25 <= 150.4) return Math.round(150 + (50/94.9) * (pm25 - 55.4));
    if (pm25 <= 250.4) return Math.round(200 + (100/100) * (pm25 - 150.4));
    return Math.round(300 + (200/250) * (pm25 - 250.4));
  }

  checkAlerts(data) {
    const alerts = [];

    if (data.pm25 > this.alertThresholds.pm25.unhealthy) {
      alerts.push({ type: 'pm25', level: 'danger', value: data.pm25 });
    } else if (data.pm25 > this.alertThresholds.pm25.moderate) {
      alerts.push({ type: 'pm25', level: 'warning', value: data.pm25 });
    }

    if (data.co2 > this.alertThresholds.co2.unhealthy) {
      alerts.push({ type: 'co2', level: 'danger', value: data.co2 });
    }

    if (data.temperature > this.alertThresholds.temperature.max ||
        data.temperature < this.alertThresholds.temperature.min) {
      alerts.push({ type: 'temperature', level: 'warning', value: data.temperature });
    }

    return alerts;
  }

  /**
   * ส่ง Dashboard คุณภาพอากาศ
   */
  async sendAirQualityDashboard(replyToken, userId) {
    const latest = this.db.prepare(`
      SELECT * FROM air_quality_readings WHERE user_id = ?
      ORDER BY recorded_at DESC LIMIT 1
    `).get(userId);

    if (!latest) {
      return this.line.replyMessage(replyToken, {
        type: 'text',
        text: 'ยังไม่มีข้อมูลคุณภาพอากาศ'
      });
    }

    const aqi = this.calculateAQI(latest.pm25);
    const aqiInfo = this.getAQIInfo(aqi);

    await this.line.replyMessage(replyToken, {
      type: 'flex',
      altText: `AQI: ${aqi} - ${aqiInfo.label}`,
      contents: this.buildAirQualityMessage(latest, aqi, aqiInfo)
    });
  }

  getAQIInfo(aqi) {
    if (aqi <= 50) return { label: 'ดีมาก', color: '#00E400', emoji: '😊' };
    if (aqi <= 100) return { label: 'พอใช้', color: '#FFFF00', emoji: '😐' };
    if (aqi <= 150) return { label: 'ไม่ดีสำหรับกลุ่มเสี่ยง', color: '#FF7E00', emoji: '😷' };
    if (aqi <= 200) return { label: 'ไม่ดีต่อสุขภาพ', color: '#FF0000', emoji: '🤧' };
    if (aqi <= 300) return { label: 'เป็นอันตรายมาก', color: '#8F3F97', emoji: '☠️' };
    return { label: 'อันตราย', color: '#7E0023', emoji: '💀' };
  }

  buildAirQualityMessage(data, aqi, aqiInfo) {
    return {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: aqiInfo.color,
        contents: [
          {
            type: 'text',
            text: `${aqiInfo.emoji} AQI: ${aqi}`,
            color: '#FFFFFF',
            weight: 'bold',
            size: 'xxl',
            align: 'center'
          },
          {
            type: 'text',
            text: aqiInfo.label,
            color: '#FFFFFF',
            align: 'center',
            size: 'md'
          }
        ]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'md',
        contents: [
          this.buildMetricRow('🔴 PM2.5', `${data.pm25?.toFixed(1)} μg/m³`),
          this.buildMetricRow('💨 CO2', `${data.co2?.toFixed(0)} ppm`),
          this.buildMetricRow('🌡️ อุณหภูมิ', `${data.temperature?.toFixed(1)}°C`),
          this.buildMetricRow('💧 ความชื้น', `${data.humidity?.toFixed(1)}%`),
          {
            type: 'text',
            text: `อัพเดทล่าสุด: ${new Date(data.recorded_at).toLocaleString('th-TH')}`,
            size: 'xs',
            color: '#AAAAAA',
            margin: 'md'
          }
        ]
      }
    };
  }

  buildMetricRow(label, value) {
    return {
      type: 'box',
      layout: 'baseline',
      contents: [
        { type: 'text', text: label, flex: 1, size: 'sm', color: '#666' },
        { type: 'text', text: value, flex: 1, size: 'sm', weight: 'bold', align: 'end' }
      ]
    };
  }

  async sendAirQualityAlert(userId, data, alerts) {
    const alertMessages = alerts.map(alert => {
      if (alert.type === 'pm25') {
        return `PM2.5: ${alert.value.toFixed(1)} μg/m³`;
      }
      if (alert.type === 'co2') {
        return `CO2: ${alert.value.toFixed(0)} ppm`;
      }
      return `อุณหภูมิ: ${alert.value.toFixed(1)}°C`;
    }).join('\n');

    await this.line.pushMessage(userId, {
      type: 'text',
      text: `⚠️ คุณภาพอากาศไม่ดี!\n${alertMessages}\n\nกรุณาสวมหน้ากากอนามัย`
    });
  }
}

module.exports = AirQualityMonitor;
```

### 6.3 Project 3: Attendance Tracking

```javascript
// projects/attendance/attendance-tracker.js

class AttendanceTracker {
  constructor(lineClient, db) {
    this.line = lineClient;
    this.db = db;
  }

  /**
   * บันทึกการเข้างาน (จากอุปกรณ์ RFID หรือ BLE)
   */
  async recordAttendance(deviceId, employeeId, type = 'checkIn') {
    const employee = this.db.prepare(
      'SELECT * FROM employees WHERE employee_id = ?'
    ).get(employeeId);

    if (!employee) {
      console.log(`ไม่พบพนักงาน ID: ${employeeId}`);
      return;
    }

    const now = new Date();
    const time = now.toTimeString().slice(0, 5);
    const date = now.toISOString().slice(0, 10);

    // บันทึกลงฐานข้อมูล
    this.db.prepare(`
      INSERT INTO attendance_records (employee_id, date, check_in_time, check_out_time, type)
      VALUES (?, ?, ?, ?, ?)
      ON CONFLICT(employee_id, date) DO UPDATE SET
        check_out_time = CASE WHEN ? = 'checkOut' THEN ? ELSE check_out_time END
    `).run(
      employeeId, date,
      type === 'checkIn' ? time : null,
      type === 'checkOut' ? time : null,
      type, type, time
    );

    // แจ้งเตือนพนักงานผ่าน LINE
    if (employee.line_user_id) {
      await this.notifyEmployee(employee, type, time);
    }

    // แจ้งผู้จัดการ
    await this.notifyManager(employee, type, time, date);
  }

  async notifyEmployee(employee, type, time) {
    const message = type === 'checkIn'
      ? `✅ บันทึกเข้างานแล้ว\n⏰ เวลา: ${time} น.`
      : `👋 บันทึกออกงานแล้ว\n⏰ เวลา: ${time} น.`;

    await this.line.pushMessage(employee.line_user_id, {
      type: 'text',
      text: message
    });
  }

  async notifyManager(employee, type, time, date) {
    const manager = this.db.prepare(
      'SELECT * FROM employees WHERE role = ? LIMIT 1'
    ).get('manager');

    if (!manager?.line_user_id) return;

    await this.line.pushMessage(manager.line_user_id, {
      type: 'flex',
      altText: `${employee.name} ${type === 'checkIn' ? 'เข้างาน' : 'ออกงาน'}`,
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: type === 'checkIn' ? '✅ พนักงานเข้างาน' : '👋 พนักงานออกงาน',
              weight: 'bold',
              size: 'lg'
            },
            {
              type: 'box',
              layout: 'vertical',
              margin: 'md',
              spacing: 'sm',
              contents: [
                { type: 'text', text: `👤 ${employee.name}` },
                { type: 'text', text: `🆔 ${employee.employee_id}` },
                { type: 'text', text: `⏰ ${time} น.` },
                { type: 'text', text: `📅 ${date}` }
              ]
            }
          ]
        }
      }
    });
  }

  /**
   * รายงานการเข้างานรายวัน
   */
  async getDailyReport(replyToken, date) {
    const records = this.db.prepare(`
      SELECT e.name, e.employee_id, a.*
      FROM attendance_records a
      JOIN employees e ON a.employee_id = e.employee_id
      WHERE a.date = ?
      ORDER BY a.check_in_time
    `).all(date);

    const total = this.db.prepare('SELECT COUNT(*) as count FROM employees').get();
    const present = records.filter(r => r.check_in_time).length;

    await this.line.replyMessage(replyToken, {
      type: 'flex',
      altText: `รายงานการเข้างาน ${date}`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          backgroundColor: '#06C755',
          contents: [{
            type: 'text',
            text: `📊 รายงาน ${date}`,
            color: '#FFFFFF',
            weight: 'bold'
          }]
        },
        body: {
          type: 'box',
          layout: 'vertical',
          spacing: 'md',
          contents: [
            {
              type: 'box',
              layout: 'horizontal',
              contents: [
                { type: 'text', text: 'เข้างาน:', flex: 1 },
                {
                  type: 'text',
                  text: `${present}/${total.count} คน`,
                  flex: 1,
                  align: 'end',
                  weight: 'bold',
                  color: '#06C755'
                }
              ]
            },
            { type: 'separator' },
            ...records.slice(0, 5).map(r => ({
              type: 'box',
              layout: 'horizontal',
              contents: [
                { type: 'text', text: r.name, flex: 2, size: 'sm' },
                {
                  type: 'text',
                  text: r.check_in_time || '-',
                  flex: 1,
                  size: 'sm',
                  align: 'end'
                }
              ]
            }))
          ]
        }
      }
    });
  }
}

module.exports = AttendanceTracker;
```

---

## 7. Database Schema สำหรับ LINE Things

```sql
-- database/things-schema.sql

CREATE TABLE IF NOT EXISTS device_links (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  device_id TEXT NOT NULL,
  device_name TEXT,
  device_type TEXT DEFAULT 'generic',
  linked_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  unlinked_at DATETIME,
  UNIQUE(user_id, device_id)
);

CREATE TABLE IF NOT EXISTS sensor_readings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  device_id TEXT,
  temperature REAL,
  humidity REAL,
  pressure REAL,
  co2 REAL,
  pm25 REAL,
  aqi INTEGER,
  recorded_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS attendance_records (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  employee_id TEXT NOT NULL,
  date TEXT NOT NULL,
  check_in_time TEXT,
  check_out_time TEXT,
  type TEXT DEFAULT 'checkIn',
  UNIQUE(employee_id, date)
);

CREATE TABLE IF NOT EXISTS employees (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  employee_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  line_user_id TEXT,
  role TEXT DEFAULT 'employee',
  department TEXT
);

CREATE INDEX IF NOT EXISTS idx_sensor_readings_user ON sensor_readings(user_id);
CREATE INDEX IF NOT EXISTS idx_sensor_readings_date ON sensor_readings(recorded_at);
```

---

## 8. Testing LINE Things

### 8.1 Unit Tests

```javascript
// tests/things/things-handler.test.js
const ThingsWebhookHandler = require('../../handlers/things-webhook');

describe('ThingsWebhookHandler', () => {
  let handler;
  let mockClient;

  beforeEach(() => {
    mockClient = {
      replyMessage: jest.fn().mockResolvedValue({}),
      pushMessage: jest.fn().mockResolvedValue({})
    };
    handler = new ThingsWebhookHandler(mockClient);
  });

  test('handleDeviceLink ควรส่ง Welcome Message', async () => {
    const event = {
      type: 'things',
      things: { type: 'link', deviceId: 'TEST-DEVICE-001' },
      source: { userId: 'U123', type: 'user' },
      replyToken: 'test-reply-token'
    };

    await handler.handleThingsEvent(event);

    expect(mockClient.replyMessage).toHaveBeenCalledTimes(1);
    const [replyToken, message] = mockClient.replyMessage.mock.calls[0];
    expect(replyToken).toBe('test-reply-token');
    expect(message.type).toBe('flex');
  });

  test('handleDeviceUnlink ควรส่ง Goodbye Message', async () => {
    const event = {
      type: 'things',
      things: { type: 'unlink', deviceId: 'TEST-DEVICE-001' },
      source: { userId: 'U123', type: 'user' },
      replyToken: 'test-reply-token'
    };

    await handler.handleThingsEvent(event);

    expect(mockClient.pushMessage).toHaveBeenCalledTimes(1);
  });

  test('parseBlePayload ควรแปลงข้อมูลอุณหภูมิและความชื้น', () => {
    // สร้าง Buffer ที่มีค่า temperature=25.5, humidity=65.3
    const buffer = Buffer.alloc(8);
    buffer.writeFloatLE(25.5, 0);
    buffer.writeFloatLE(65.3, 4);
    const base64 = buffer.toString('base64');

    const result = handler.parseBlePayload(base64);

    expect(result.temperature).toBeCloseTo(25.5, 1);
    expect(result.humidity).toBeCloseTo(65.3, 1);
  });
});
```

---

## 9. Complete Server Setup

```javascript
// server.js - Complete LINE Things Server
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const ThingsWebhookHandler = require('./handlers/things-webhook');
const SmartHomeController = require('./projects/smart-home/smart-home-controller');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.Client(config);
const app = express();

const thingsHandler = new ThingsWebhookHandler(client);
const smartHome = new SmartHomeController(client);

// ลงทะเบียนอุปกรณ์ Smart Home
smartHome.registerDevice('LIGHT_001', {
  name: 'ไฟห้องนั่งเล่น',
  type: 'light',
  topic: 'home/living-room/light'
});

smartHome.registerDevice('FAN_001', {
  name: 'พัดลมห้องนอน',
  type: 'fan',
  topic: 'home/bedroom/fan'
});

// Webhook
app.post('/webhook',
  line.middleware(config),
  async (req, res) => {
    res.sendStatus(200);

    const events = req.body.events;
    await Promise.all(events.map(async (event) => {
      try {
        if (event.type === 'things') {
          await thingsHandler.handleEvent(event);
        } else if (event.type === 'message') {
          await handleMessage(event);
        } else if (event.type === 'postback') {
          await handlePostback(event);
        }
      } catch (error) {
        console.error('Error processing event:', error);
      }
    }));
  }
);

async function handleMessage(event) {
  const { replyToken, message, source } = event;
  const text = message.text;

  if (text === 'บ้าน' || text === 'smart home') {
    await smartHome.showHomeStatus(replyToken);
  }
}

async function handlePostback(event) {
  const { source, postback } = event;
  const userId = source.userId;
  const params = new URLSearchParams(postback.data);
  const action = params.get('action');

  if (action === 'deviceOn') {
    await smartHome.controlDevice(userId, params.get('deviceId'), 'on');
  } else if (action === 'deviceOff') {
    await smartHome.controlDevice(userId, params.get('deviceId'), 'off');
  } else if (action === 'deviceStatus') {
    await thingsHandler.sendDeviceStatus(event.replyToken, userId);
  }
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`LINE Things Server รันที่ port ${PORT}`);
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **LINE Things Overview** - ความสามารถและ Architecture ของ LINE Things
2. **LINE Things Trial** - การตั้งค่าสำหรับ Development
3. **Device Link/Unlink** - จัดการการเชื่อมต่ออุปกรณ์
4. **Scenario GATT Service** - ตั้งค่า BLE Scenarios
5. **Arduino BLE** - เขียน Firmware สำหรับ Arduino
6. **Raspberry Pi BLE** - Python BLE Server
7. **Smart Home Control** - ควบคุมบ้านอัจฉริยะผ่าน LINE
8. **Air Quality Monitor** - ติดตามคุณภาพอากาศ
9. **Attendance Tracking** - ระบบบันทึกเวลาเข้างาน
10. **Testing** - Unit Tests สำหรับ LINE Things

LINE Things เปิดโลกใหม่ในการเชื่อมอุปกรณ์ IoT กับ LINE ทำให้สร้างประสบการณ์ที่ดีขึ้นสำหรับผู้ใช้ในชีวิตประจำวัน
