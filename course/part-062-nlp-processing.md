# Part 62: NLP Processing สำหรับ LINE Bot

## บทนำ

Natural Language Processing (NLP) คือเทคโนโลยีที่ช่วยให้คอมพิวเตอร์เข้าใจ ตีความ และประมวลผลภาษามนุษย์ การนำ NLP มาใช้กับ LINE Bot ทำให้บอทสามารถเข้าใจเจตนาของผู้ใช้ ดึงข้อมูลสำคัญ และตอบสนองได้อย่างฉลาด บทนี้ครอบคลุม NLP Pipeline ทั้งหมดตั้งแต่การวิเคราะห์ Intent จนถึงการสนทนาหลายภาษา

## เนื้อหาที่จะเรียนรู้

1. Intent Detection (การตรวจจับเจตนา)
2. Entity Extraction (การดึงข้อมูลสำคัญ)
3. Sentiment Analysis (การวิเคราะห์ความรู้สึก)
4. Thai Language NLP
5. Google Cloud Natural Language API
6. การสร้าง Custom Intent Classifier
7. การเทรน Model จากประวัติข้อความ LINE
8. Multi-language Support
9. Response Generation
10. NLP Pipeline สมบูรณ์

---

## สถาปัตยกรรม NLP Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                         NLP Pipeline                                │
│                                                                     │
│  Input Text                                                         │
│      │                                                              │
│      ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Preprocessing                            │   │
│  │  Tokenization → Normalization → Stop Words → Stemming       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      │                                                              │
│      ▼                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │   Language   │  │   Intent     │  │   Entity                 │  │
│  │  Detection   │  │  Detection   │  │   Extraction             │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│      │                    │                       │                 │
│      ▼                    ▼                       ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Sentiment   │  │   Context    │  │   Response               │  │
│  │  Analysis    │  │  Manager     │  │   Generator              │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│                        Final Response                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. ติดตั้ง Dependencies

```bash
npm install natural compromise franc-min @google-cloud/language
npm install ml-classify-text node-nlp axios

# สำหรับ Thai NLP
npm install js-nlp-utils

# Python dependencies (ถ้าใช้ Python)
pip install pythainlp transformers torch scikit-learn
pip install google-cloud-language nltk spacy
```

---

## 2. Thai Language Preprocessing

```javascript
// src/nlp/thaiPreprocessor.js

class ThaiPreprocessor {
  constructor() {
    // Thai stop words
    this.stopWords = new Set([
      'และ', 'หรือ', 'แต่', 'เพราะ', 'เนื่องจาก', 'ดังนั้น',
      'ซึ่ง', 'ที่', 'ของ', 'ใน', 'ออก', 'มา', 'ไป', 'ได้',
      'จะ', 'ก็', 'ยัง', 'แล้ว', 'อยู่', 'ไม่', 'มี', 'คือ',
      'เป็น', 'ให้', 'กับ', 'จาก', 'โดย', 'เพื่อ', 'ว่า',
      'นี้', 'นั้น', 'นั่น', 'ตาม', 'ต่อ', 'พร้อม', 'ทั้ง',
    ]);

    // Thai number normalization
    this.thaiNumbers = {
      '๐': '0', '๑': '1', '๒': '2', '๓': '3', '๔': '4',
      '๕': '5', '๖': '6', '๗': '7', '๘': '8', '๙': '9',
    };

    // Informal Thai to formal mapping
    this.informalMap = {
      'อยากได้': 'ต้องการ',
      'เอา': 'ต้องการ',
      'หน่อย': '',
      'นะ': '',
      'จ้า': '',
      'ครับ': '',
      'ค่ะ': '',
      'คะ': '',
      'นะคะ': '',
      'นะครับ': '',
      'จ้า': '',
      'จ้าา': '',
      'เน้อ': '',
      'อ่ะ': '',
    };
  }

  // Tokenize Thai text (simple rule-based)
  tokenize(text) {
    // Simple Thai tokenizer
    // ในระบบจริงควรใช้ PyThaiNLP หรือ API
    const tokens = [];
    let i = 0;

    while (i < text.length) {
      // Check for Thai word patterns
      let word = '';
      while (i < text.length && this.isThaiChar(text[i])) {
        word += text[i];
        i++;
      }

      if (word.length > 0) {
        tokens.push(word);
      } else if (text[i] !== ' ' && text[i] !== '\n') {
        tokens.push(text[i]);
        i++;
      } else {
        i++;
      }
    }

    return tokens;
  }

  isThaiChar(char) {
    const code = char.charCodeAt(0);
    return code >= 0x0E00 && code <= 0x0E7F;
  }

  // Normalize text
  normalize(text) {
    let normalized = text.toLowerCase().trim();

    // แปลงเลขไทย
    for (const [thai, arabic] of Object.entries(this.thaiNumbers)) {
      normalized = normalized.replace(new RegExp(thai, 'g'), arabic);
    }

    // ลบ informal words
    for (const [informal, formal] of Object.entries(this.informalMap)) {
      normalized = normalized.replace(new RegExp(informal, 'g'), formal);
    }

    // ลบ Emoji
    normalized = normalized.replace(/[\u{1F000}-\u{1FFFF}]/gu, '');

    // ลบ Multiple spaces
    normalized = normalized.replace(/\s+/g, ' ').trim();

    return normalized;
  }

  // Remove Stop Words
  removeStopWords(tokens) {
    return tokens.filter((token) => !this.stopWords.has(token));
  }

  // Detect Language
  detectLanguage(text) {
    const thaiPattern = /[฀-๿]/;
    const englishPattern = /[a-zA-Z]/;
    const thaiCount = (text.match(/[฀-๿]/g) || []).length;
    const englishCount = (text.match(/[a-zA-Z]/g) || []).length;

    if (thaiCount > englishCount) return 'th';
    if (englishCount > thaiCount) return 'en';
    return 'mixed';
  }

  // Full preprocessing pipeline
  preprocess(text) {
    const language = this.detectLanguage(text);
    const normalized = this.normalize(text);
    const tokens = this.tokenize(normalized);
    const filtered = this.removeStopWords(tokens);

    return {
      original: text,
      normalized,
      tokens,
      filteredTokens: filtered,
      language,
    };
  }
}

module.exports = new ThaiPreprocessor();
```

---

## 3. Intent Detection

```javascript
// src/nlp/intentDetector.js
const { NlpManager } = require('node-nlp');
const logger = require('../utils/logger');

class IntentDetector {
  constructor() {
    this.manager = new NlpManager({
      languages: ['th', 'en'],
      forceNER: true,
      nlu: { log: false },
    });
    this.isTraining = false;
    this.isTrained = false;
  }

  // กำหนด Training Data
  setupTrainingData() {
    // ===== GREETING INTENT =====
    this.manager.addDocument('th', 'สวัสดี', 'greeting');
    this.manager.addDocument('th', 'หวัดดี', 'greeting');
    this.manager.addDocument('th', 'ดีครับ', 'greeting');
    this.manager.addDocument('th', 'ดีค่ะ', 'greeting');
    this.manager.addDocument('th', 'สวัสดีครับ', 'greeting');
    this.manager.addDocument('th', 'สวัสดีค่ะ', 'greeting');
    this.manager.addDocument('th', 'hello', 'greeting');
    this.manager.addDocument('th', 'hi', 'greeting');
    this.manager.addDocument('en', 'hello', 'greeting');
    this.manager.addDocument('en', 'hi', 'greeting');
    this.manager.addDocument('en', 'hey', 'greeting');
    this.manager.addDocument('en', 'good morning', 'greeting');

    // ===== ORDER INTENT =====
    this.manager.addDocument('th', 'สั่งอาหาร', 'order.food');
    this.manager.addDocument('th', 'อยากสั่งอาหาร', 'order.food');
    this.manager.addDocument('th', 'ขอสั่งข้าว', 'order.food');
    this.manager.addDocument('th', 'สั่งกาแฟหน่อย', 'order.food');
    this.manager.addDocument('th', 'ต้องการสั่ง', 'order.food');
    this.manager.addDocument('th', 'เอา', 'order.food');
    this.manager.addDocument('th', 'ขอ', 'order.food');

    // ===== PRODUCT INQUIRY INTENT =====
    this.manager.addDocument('th', 'มีอะไรขายบ้าง', 'inquiry.product');
    this.manager.addDocument('th', 'สินค้ามีอะไรบ้าง', 'inquiry.product');
    this.manager.addDocument('th', 'ราคาเท่าไหร่', 'inquiry.price');
    this.manager.addDocument('th', 'ราคาเท่าไร', 'inquiry.price');
    this.manager.addDocument('th', 'ขายเท่าไหร่', 'inquiry.price');
    this.manager.addDocument('th', 'กี่บาท', 'inquiry.price');
    this.manager.addDocument('th', 'แพงไหม', 'inquiry.price');

    // ===== LOCATION INTENT =====
    this.manager.addDocument('th', 'อยู่ที่ไหน', 'inquiry.location');
    this.manager.addDocument('th', 'สาขาอยู่ที่ไหน', 'inquiry.location');
    this.manager.addDocument('th', 'ที่อยู่', 'inquiry.location');
    this.manager.addDocument('th', 'เส้นทาง', 'inquiry.location');
    this.manager.addDocument('th', 'แผนที่', 'inquiry.location');
    this.manager.addDocument('th', 'ไปยังไง', 'inquiry.location');

    // ===== HOURS INTENT =====
    this.manager.addDocument('th', 'เปิดกี่โมง', 'inquiry.hours');
    this.manager.addDocument('th', 'ปิดกี่โมง', 'inquiry.hours');
    this.manager.addDocument('th', 'เวลาทำการ', 'inquiry.hours');
    this.manager.addDocument('th', 'เปิดวันไหนบ้าง', 'inquiry.hours');
    this.manager.addDocument('th', 'ปิดวันอะไร', 'inquiry.hours');

    // ===== COMPLAINT INTENT =====
    this.manager.addDocument('th', 'ไม่พอใจ', 'complaint');
    this.manager.addDocument('th', 'บ่น', 'complaint');
    this.manager.addDocument('th', 'แย่มาก', 'complaint');
    this.manager.addDocument('th', 'ห่วยแตก', 'complaint');
    this.manager.addDocument('th', 'ไม่ดี', 'complaint');
    this.manager.addDocument('th', 'ผิดพลาด', 'complaint');
    this.manager.addDocument('th', 'ของหาย', 'complaint');
    this.manager.addDocument('th', 'ของเสีย', 'complaint');

    // ===== CANCEL INTENT =====
    this.manager.addDocument('th', 'ยกเลิก', 'order.cancel');
    this.manager.addDocument('th', 'ไม่เอาแล้ว', 'order.cancel');
    this.manager.addDocument('th', 'เลิกสั่ง', 'order.cancel');
    this.manager.addDocument('th', 'cancel', 'order.cancel');

    // ===== PAYMENT INTENT =====
    this.manager.addDocument('th', 'จ่ายเงิน', 'payment');
    this.manager.addDocument('th', 'ชำระเงิน', 'payment');
    this.manager.addDocument('th', 'โอนเงิน', 'payment');
    this.manager.addDocument('th', 'พร้อมเพย์', 'payment');
    this.manager.addDocument('th', 'บัตรเครดิต', 'payment');

    // ===== GOODBYE INTENT =====
    this.manager.addDocument('th', 'ลาก่อน', 'goodbye');
    this.manager.addDocument('th', 'bye', 'goodbye');
    this.manager.addDocument('th', 'บาย', 'goodbye');
    this.manager.addDocument('th', 'ขอบคุณ', 'goodbye');
    this.manager.addDocument('th', 'ขอบคุณมาก', 'goodbye');
    this.manager.addDocument('en', 'bye', 'goodbye');
    this.manager.addDocument('en', 'goodbye', 'goodbye');

    // ===== RESPONSES =====
    this.manager.addAnswer('th', 'greeting', 'สวัสดีครับ! ยินดีให้บริการ 😊');
    this.manager.addAnswer('th', 'greeting', 'หวัดดีครับ มีอะไรให้ช่วยไหม?');
    this.manager.addAnswer('th', 'goodbye', 'ขอบคุณที่ใช้บริการครับ 😊');
    this.manager.addAnswer('th', 'goodbye', 'ยินดีให้บริการครับ แวะมาใหม่นะครับ');
    this.manager.addAnswer('th', 'inquiry.hours', 'เปิดทำการทุกวัน จันทร์-ศุกร์ 9:00-18:00 น. เสาร์-อาทิตย์ 10:00-17:00 น.');
  }

  // เทรน Model
  async train() {
    if (this.isTraining) {
      logger.info('Training already in progress...');
      return;
    }

    logger.info('Starting NLP training...');
    this.isTraining = true;

    try {
      this.setupTrainingData();
      await this.manager.train();
      this.isTrained = true;
      logger.info('NLP training completed');
    } catch (error) {
      logger.error('NLP training failed', { error: error.message });
      throw error;
    } finally {
      this.isTraining = false;
    }
  }

  // บันทึก Model
  async save(path = './models/nlp-model.json') {
    await this.manager.save(path);
    logger.info('NLP model saved', { path });
  }

  // โหลด Model
  async load(path = './models/nlp-model.json') {
    try {
      await this.manager.load(path);
      this.isTrained = true;
      logger.info('NLP model loaded', { path });
      return true;
    } catch (error) {
      logger.warn('Could not load NLP model, will train fresh', { error: error.message });
      return false;
    }
  }

  // ตรวจจับ Intent
  async detect(text, language = 'th') {
    if (!this.isTrained) {
      await this.train();
    }

    const result = await this.manager.process(language, text);

    return {
      intent: result.intent,
      score: result.score,
      entities: result.entities,
      sentiment: result.sentiment,
      answer: result.answer,
      language: result.language,
      utterance: result.utterance,
    };
  }

  // เพิ่ม Training Data จากประวัติ LINE
  async addFromHistory(messages) {
    let added = 0;

    for (const msg of messages) {
      if (msg.intent && msg.text && msg.confidence > 0.8) {
        this.manager.addDocument(msg.language || 'th', msg.text, msg.intent);
        added++;
      }
    }

    logger.info(`Added ${added} documents from history`);
    return added;
  }
}

module.exports = new IntentDetector();
```

---

## 4. Entity Extraction

```javascript
// src/nlp/entityExtractor.js

class EntityExtractor {
  constructor() {
    // Thai date patterns
    this.datePatterns = [
      /(\d{1,2})[\/\-](\d{1,2})[\/\-](\d{2,4})/,
      /วันที่\s*(\d{1,2})\s*(มกราคม|กุมภาพันธ์|มีนาคม|เมษายน|พฤษภาคม|มิถุนายน|กรกฎาคม|สิงหาคม|กันยายน|ตุลาคม|พฤศจิกายน|ธันวาคม)/i,
      /(จันทร์|อังคาร|พุธ|พฤหัส|ศุกร์|เสาร์|อาทิตย์)/,
      /พรุ่งนี้|วันนี้|เมื่อวาน|ตะกี้/,
    ];

    // Thai time patterns
    this.timePatterns = [
      /(\d{1,2}):(\d{2})/,
      /(\d{1,2})\s*(โมง|นาฬิกา)/,
      /บ่าย\s*(\d{1,2})\s*โมง/,
      /เช้า\s*(\d{1,2})\s*โมง/,
    ];

    // Phone patterns
    this.phonePatterns = [
      /0[0-9]{1,2}[- ]?[0-9]{3,4}[- ]?[0-9]{4}/,
      /\+66[- ]?[0-9]{1,2}[- ]?[0-9]{3,4}[- ]?[0-9]{4}/,
    ];

    // Amount/price patterns
    this.amountPatterns = [
      /(\d+(?:,\d{3})*(?:\.\d{2})?)\s*บาท/,
      /฿\s*(\d+(?:,\d{3})*(?:\.\d{2})?)/,
      /(\d+)\s*฿/,
    ];

    // Thai months
    this.monthMap = {
      'มกราคม': 1, 'กุมภาพันธ์': 2, 'มีนาคม': 3, 'เมษายน': 4,
      'พฤษภาคม': 5, 'มิถุนายน': 6, 'กรกฎาคม': 7, 'สิงหาคม': 8,
      'กันยายน': 9, 'ตุลาคม': 10, 'พฤศจิกายน': 11, 'ธันวาคม': 12,
    };
  }

  // ดึง Entities ทั้งหมด
  extract(text) {
    return {
      dates: this.extractDates(text),
      times: this.extractTimes(text),
      phones: this.extractPhones(text),
      amounts: this.extractAmounts(text),
      locations: this.extractLocations(text),
      emails: this.extractEmails(text),
      urls: this.extractUrls(text),
      quantities: this.extractQuantities(text),
      productNames: this.extractProductNames(text),
    };
  }

  extractDates(text) {
    const dates = [];

    // วันที่แบบตัวเลข
    const dateRegex = /(\d{1,2})[\/\-](\d{1,2})[\/\-](\d{2,4})/g;
    let match;
    while ((match = dateRegex.exec(text)) !== null) {
      dates.push({
        raw: match[0],
        day: parseInt(match[1]),
        month: parseInt(match[2]),
        year: parseInt(match[3]) < 100
          ? parseInt(match[3]) + 2000
          : parseInt(match[3]),
        type: 'numeric',
      });
    }

    // วันในสัปดาห์
    const dayPattern = /(จันทร์|อังคาร|พุธ|พฤหัส(?:บดี)?|ศุกร์|เสาร์|อาทิตย์)/g;
    while ((match = dayPattern.exec(text)) !== null) {
      dates.push({ raw: match[0], dayOfWeek: match[1], type: 'dayOfWeek' });
    }

    // วันพิเศษ
    const relativePattern = /(พรุ่งนี้|วันนี้|เมื่อวาน|ตะกี้)/g;
    while ((match = relativePattern.exec(text)) !== null) {
      const today = new Date();
      let date = new Date(today);

      if (match[1] === 'พรุ่งนี้') date.setDate(today.getDate() + 1);
      else if (match[1] === 'เมื่อวาน') date.setDate(today.getDate() - 1);

      dates.push({
        raw: match[0],
        resolved: date.toISOString().split('T')[0],
        type: 'relative',
      });
    }

    return dates;
  }

  extractTimes(text) {
    const times = [];

    const timeRegex = /(\d{1,2}):(\d{2})(?::(\d{2}))?/g;
    let match;
    while ((match = timeRegex.exec(text)) !== null) {
      times.push({
        raw: match[0],
        hours: parseInt(match[1]),
        minutes: parseInt(match[2]),
        seconds: match[3] ? parseInt(match[3]) : 0,
        type: 'hhmm',
      });
    }

    // เวลาภาษาไทย
    const thaiTimeRegex = /(\d{1,2})\s*โมง/g;
    while ((match = thaiTimeRegex.exec(text)) !== null) {
      times.push({
        raw: match[0],
        hours: parseInt(match[1]),
        minutes: 0,
        type: 'thai_hour',
      });
    }

    return times;
  }

  extractPhones(text) {
    const phones = [];
    const phoneRegex = /(?:\+66|0)[0-9]{1,2}[- ]?[0-9]{3,4}[- ]?[0-9]{4}/g;
    let match;
    while ((match = phoneRegex.exec(text)) !== null) {
      phones.push({
        raw: match[0],
        normalized: match[0].replace(/[^0-9+]/g, ''),
      });
    }
    return phones;
  }

  extractAmounts(text) {
    const amounts = [];
    const amountRegex = /(\d+(?:,\d{3})*(?:\.\d{2})?)\s*(?:บาท|฿)/g;
    let match;
    while ((match = amountRegex.exec(text)) !== null) {
      amounts.push({
        raw: match[0],
        value: parseFloat(match[1].replace(/,/g, '')),
        currency: 'THB',
      });
    }
    return amounts;
  }

  extractLocations(text) {
    const locations = [];

    // จังหวัด
    const provinces = [
      'กรุงเทพ', 'เชียงใหม่', 'เชียงราย', 'ขอนแก่น', 'อุดรธานี',
      'นครราชสีมา', 'ภูเก็ต', 'สุราษฎร์ธานี', 'พัทยา', 'นนทบุรี',
      'ปทุมธานี', 'สมุทรปราการ', 'ระยอง', 'ชลบุรี',
    ];

    for (const province of provinces) {
      if (text.includes(province)) {
        locations.push({ name: province, type: 'province' });
      }
    }

    return locations;
  }

  extractEmails(text) {
    const emails = [];
    const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
    let match;
    while ((match = emailRegex.exec(text)) !== null) {
      emails.push({ raw: match[0], email: match[0].toLowerCase() });
    }
    return emails;
  }

  extractUrls(text) {
    const urls = [];
    const urlRegex = /https?:\/\/[^\s]+/g;
    let match;
    while ((match = urlRegex.exec(text)) !== null) {
      urls.push({ raw: match[0], url: match[0] });
    }
    return urls;
  }

  extractQuantities(text) {
    const quantities = [];
    const qtyRegex = /(\d+)\s*(ชิ้น|อัน|กล่อง|ถุง|แพ็ค|ลิตร|กิโล|กรัม|แก้ว|จาน|ชุด)/g;
    let match;
    while ((match = qtyRegex.exec(text)) !== null) {
      quantities.push({
        raw: match[0],
        value: parseInt(match[1]),
        unit: match[2],
      });
    }
    return quantities;
  }

  extractProductNames(text) {
    // ในระบบจริงควรใช้ dictionary จาก database
    const products = [];
    const productKeywords = ['กาแฟ', 'ชา', 'น้ำ', 'อาหาร', 'ข้าว', 'ก๋วยเตี๋ยว'];

    for (const keyword of productKeywords) {
      if (text.includes(keyword)) {
        products.push({ name: keyword, type: 'food_beverage' });
      }
    }

    return products;
  }
}

module.exports = new EntityExtractor();
```

---

## 5. Sentiment Analysis

```javascript
// src/nlp/sentimentAnalyzer.js

class ThaiSentimentAnalyzer {
  constructor() {
    // Positive words
    this.positiveWords = new Set([
      'ดี', 'เยี่ยม', 'ยอดเยี่ยม', 'ดีมาก', 'สุดยอด', 'เจ๋ง',
      'ชอบ', 'ชอบมาก', 'รัก', 'ประทับใจ', 'พอใจ', 'ขอบคุณ',
      'สวย', 'สวยมาก', 'น่ารัก', 'อร่อย', 'อร่อยมาก', 'เด็ด',
      'ถูก', 'คุ้ม', 'คุ้มมาก', 'ราคาดี', 'บริการดี', 'รวดเร็ว',
      'ประทับใจมาก', 'แนะนำ', 'perfect', 'great', 'excellent',
      'amazing', 'wonderful', 'fantastic', 'love', 'good',
    ]);

    // Negative words
    this.negativeWords = new Set([
      'แย่', 'แย่มาก', 'ห่วย', 'ห่วยแตก', 'ไม่ดี', 'ผิดหวัง',
      'ไม่พอใจ', 'โกรธ', 'เสียใจ', 'น่าเบื่อ', 'ไม่อร่อย',
      'แพง', 'แพงมาก', 'ช้า', 'ช้ามาก', 'ไม่รวดเร็ว', 'หน้าบึ้ง',
      'ไม่สุภาพ', 'หยาบคาย', 'โกหก', 'ไม่ตรงเวลา', 'ผิดพลาด',
      'bad', 'terrible', 'awful', 'horrible', 'disgusting', 'hate',
      'poor', 'worst', 'disappointing', 'useless',
    ]);

    // Intensifiers
    this.intensifiers = new Set([
      'มาก', 'มากๆ', 'สุดๆ', 'สุดยอด', 'เกินไป', 'โคตร',
      'very', 'so', 'extremely', 'absolutely', 'totally',
    ]);

    // Negators
    this.negators = new Set([
      'ไม่', 'ไม่ได้', 'ไม่มี', 'ไม่ค่อย', 'ไม่ค่อยจะ',
      'not', "don't", 'never', "won't", "can't",
    ]);
  }

  // วิเคราะห์ Sentiment
  analyze(text) {
    const words = text.split(/\s+/);
    let score = 0;
    let positiveCount = 0;
    let negativeCount = 0;

    let previousWord = '';
    let isNegated = false;

    for (const word of words) {
      const cleanWord = word.toLowerCase().trim();

      // ตรวจ Negator
      if (this.negators.has(cleanWord)) {
        isNegated = true;
        previousWord = cleanWord;
        continue;
      }

      // ตรวจ Intensifier
      const isIntensified = this.intensifiers.has(cleanWord);
      const multiplier = isIntensified ? 1.5 : 1;

      if (this.positiveWords.has(cleanWord)) {
        const wordScore = 1 * multiplier;
        score += isNegated ? -wordScore : wordScore;
        if (!isNegated) positiveCount++;
        else negativeCount++;
        isNegated = false;
      } else if (this.negativeWords.has(cleanWord)) {
        const wordScore = -1 * multiplier;
        score += isNegated ? -wordScore : wordScore;
        if (!isNegated) negativeCount++;
        else positiveCount++;
        isNegated = false;
      }

      previousWord = cleanWord;
    }

    // Normalize score to [-1, 1]
    const totalWords = positiveCount + negativeCount;
    const normalizedScore = totalWords > 0 ? score / totalWords : 0;

    return {
      score: normalizedScore,
      label: this.scoreToLabel(normalizedScore),
      positiveCount,
      negativeCount,
      magnitude: Math.abs(normalizedScore),
    };
  }

  scoreToLabel(score) {
    if (score > 0.3) return 'positive';
    if (score < -0.3) return 'negative';
    return 'neutral';
  }

  // วิเคราะห์อารมณ์หลายด้าน
  analyzeEmotions(text) {
    const emotions = {
      joy: 0,
      sadness: 0,
      anger: 0,
      fear: 0,
      surprise: 0,
      disgust: 0,
    };

    const emotionKeywords = {
      joy: ['ดีใจ', 'ยินดี', 'มีความสุข', 'ตื่นเต้น', 'สนุก', 'happy', 'joy'],
      sadness: ['เศร้า', 'เสียใจ', 'เสียดาย', 'กังวล', 'sad', 'disappointed'],
      anger: ['โกรธ', 'หัวร้อน', 'รำคาญ', 'หงุดหงิด', 'angry', 'furious'],
      fear: ['กลัว', 'กังวล', 'ตกใจ', 'nervous', 'scared', 'worried'],
      surprise: ['ประหลาดใจ', 'ตะลึง', 'ว้าว', 'wow', 'surprised', 'amazing'],
      disgust: ['รังเกียจ', 'น่าเกลียด', 'เหม็น', 'gross', 'disgusting'],
    };

    for (const [emotion, keywords] of Object.entries(emotionKeywords)) {
      for (const keyword of keywords) {
        if (text.toLowerCase().includes(keyword.toLowerCase())) {
          emotions[emotion] += 1;
        }
      }
    }

    const dominantEmotion = Object.entries(emotions)
      .sort(([, a], [, b]) => b - a)[0];

    return {
      emotions,
      dominant: dominantEmotion[0],
      confidence: dominantEmotion[1] > 0 ? 0.7 : 0,
    };
  }
}

module.exports = new ThaiSentimentAnalyzer();
```

---

## 6. Google Cloud Natural Language API

```javascript
// src/nlp/googleNLP.js
const { LanguageServiceClient } = require('@google-cloud/language');
const logger = require('../utils/logger');

class GoogleNLPService {
  constructor() {
    this.client = new LanguageServiceClient({
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  // วิเคราะห์ Sentiment
  async analyzeSentiment(text) {
    try {
      const [result] = await this.client.analyzeSentiment({
        document: {
          content: text,
          type: 'PLAIN_TEXT',
          language: 'th',
        },
      });

      const sentiment = result.documentSentiment;
      return {
        score: sentiment.score, // -1 ถึง 1
        magnitude: sentiment.magnitude, // ความแรงของอารมณ์
        label: this.scoreToLabel(sentiment.score),
      };
    } catch (error) {
      logger.error('Google NLP Sentiment error', { error: error.message });
      throw error;
    }
  }

  // ดึง Entities
  async analyzeEntities(text) {
    try {
      const [result] = await this.client.analyzeEntities({
        document: {
          content: text,
          type: 'PLAIN_TEXT',
          language: 'th',
        },
      });

      return result.entities.map((entity) => ({
        name: entity.name,
        type: entity.type,
        salience: entity.salience,
        mentions: entity.mentions.map((m) => ({
          text: m.text.content,
          type: m.type,
        })),
        metadata: entity.metadata,
      }));
    } catch (error) {
      logger.error('Google NLP Entities error', { error: error.message });
      throw error;
    }
  }

  // วิเคราะห์ Syntax
  async analyzeSyntax(text) {
    try {
      const [result] = await this.client.analyzeSyntax({
        document: {
          content: text,
          type: 'PLAIN_TEXT',
          language: 'th',
        },
      });

      return {
        tokens: result.tokens.map((token) => ({
          text: token.text.content,
          partOfSpeech: token.partOfSpeech.tag,
          dependencyEdge: token.dependencyEdge,
          lemma: token.lemma,
        })),
        language: result.language,
      };
    } catch (error) {
      logger.error('Google NLP Syntax error', { error: error.message });
      throw error;
    }
  }

  // วิเคราะห์ทั้งหมด
  async analyzeAll(text) {
    try {
      const [result] = await this.client.annotateText({
        document: {
          content: text,
          type: 'PLAIN_TEXT',
          language: 'th',
        },
        features: {
          extractSyntax: true,
          extractEntities: true,
          extractDocumentSentiment: true,
          classifyText: false,
        },
      });

      return {
        sentiment: {
          score: result.documentSentiment?.score || 0,
          magnitude: result.documentSentiment?.magnitude || 0,
          label: this.scoreToLabel(result.documentSentiment?.score || 0),
        },
        entities: result.entities?.map((e) => ({
          name: e.name,
          type: e.type,
          salience: e.salience,
        })) || [],
        language: result.language,
      };
    } catch (error) {
      logger.error('Google NLP full analysis error', { error: error.message });
      throw error;
    }
  }

  scoreToLabel(score) {
    if (score > 0.25) return 'positive';
    if (score < -0.25) return 'negative';
    return 'neutral';
  }
}

module.exports = new GoogleNLPService();
```

---

## 7. Custom Intent Classifier (ML-based)

```javascript
// src/nlp/intentClassifier.js
// Naive Bayes Classifier สำหรับ Intent Detection

class NaiveBayesClassifier {
  constructor() {
    this.classes = {};
    this.vocabulary = new Set();
    this.totalDocuments = 0;
    this.classCounts = {};
  }

  // เทรน Model
  train(text, label) {
    const words = this.tokenize(text);

    if (!this.classes[label]) {
      this.classes[label] = {};
      this.classCounts[label] = 0;
    }

    this.classCounts[label]++;
    this.totalDocuments++;

    for (const word of words) {
      this.vocabulary.add(word);
      this.classes[label][word] = (this.classes[label][word] || 0) + 1;
    }
  }

  // ทำนาย Intent
  classify(text) {
    const words = this.tokenize(text);
    const scores = {};

    for (const label of Object.keys(this.classes)) {
      // Prior probability
      const prior = Math.log(this.classCounts[label] / this.totalDocuments);
      let likelihood = 0;

      const classWordCounts = this.classes[label];
      const totalWordsInClass = Object.values(classWordCounts).reduce(
        (a, b) => a + b, 0
      );

      for (const word of words) {
        // Laplace smoothing
        const wordCount = (classWordCounts[word] || 0) + 1;
        const vocabSize = this.vocabulary.size;
        likelihood += Math.log(wordCount / (totalWordsInClass + vocabSize));
      }

      scores[label] = prior + likelihood;
    }

    // หา label ที่มี score สูงสุด
    const bestLabel = Object.entries(scores).sort(([, a], [, b]) => b - a)[0];

    // Softmax normalization
    const maxScore = bestLabel[1];
    const expScores = Object.fromEntries(
      Object.entries(scores).map(([k, v]) => [k, Math.exp(v - maxScore)])
    );
    const sumExp = Object.values(expScores).reduce((a, b) => a + b, 0);
    const probabilities = Object.fromEntries(
      Object.entries(expScores).map(([k, v]) => [k, v / sumExp])
    );

    return {
      label: bestLabel[0],
      confidence: probabilities[bestLabel[0]],
      all: probabilities,
    };
  }

  tokenize(text) {
    return text
      .toLowerCase()
      .replace(/[^฀-๿a-z0-9\s]/g, ' ')
      .split(/\s+/)
      .filter((w) => w.length > 1);
  }

  // บันทึก Model
  toJSON() {
    return {
      classes: this.classes,
      vocabulary: [...this.vocabulary],
      totalDocuments: this.totalDocuments,
      classCounts: this.classCounts,
    };
  }

  // โหลด Model
  static fromJSON(data) {
    const classifier = new NaiveBayesClassifier();
    classifier.classes = data.classes;
    classifier.vocabulary = new Set(data.vocabulary);
    classifier.totalDocuments = data.totalDocuments;
    classifier.classCounts = data.classCounts;
    return classifier;
  }
}

// สร้างและเทรน Classifier
function createIntentClassifier() {
  const classifier = new NaiveBayesClassifier();

  // Training data
  const trainingData = [
    // Greetings
    { text: 'สวัสดีครับ', label: 'greeting' },
    { text: 'หวัดดีค่ะ', label: 'greeting' },
    { text: 'ดีครับ', label: 'greeting' },
    { text: 'hello', label: 'greeting' },
    { text: 'hi there', label: 'greeting' },

    // Product inquiry
    { text: 'มีสินค้าอะไรบ้าง', label: 'product_inquiry' },
    { text: 'ขายอะไรบ้างครับ', label: 'product_inquiry' },
    { text: 'แนะนำสินค้าหน่อย', label: 'product_inquiry' },
    { text: 'มีโปรโมชั่นอะไรบ้าง', label: 'product_inquiry' },

    // Price inquiry
    { text: 'ราคาเท่าไหร่', label: 'price_inquiry' },
    { text: 'กี่บาทครับ', label: 'price_inquiry' },
    { text: 'ราคาเท่าไร', label: 'price_inquiry' },
    { text: 'ขายราคาเท่าไหร่', label: 'price_inquiry' },

    // Order
    { text: 'ขอสั่ง', label: 'order' },
    { text: 'อยากสั่ง', label: 'order' },
    { text: 'สั่งได้เลยไหม', label: 'order' },
    { text: 'จะซื้อ', label: 'order' },

    // Support
    { text: 'ของหาย', label: 'support' },
    { text: 'ปัญหาการสั่ง', label: 'support' },
    { text: 'ไม่ได้รับสินค้า', label: 'support' },
    { text: 'ของเสีย', label: 'support' },

    // Goodbye
    { text: 'ขอบคุณครับ', label: 'goodbye' },
    { text: 'บาย', label: 'goodbye' },
    { text: 'ลาก่อนนะครับ', label: 'goodbye' },
    { text: 'ขอบคุณมาก', label: 'goodbye' },
  ];

  for (const data of trainingData) {
    classifier.train(data.text, data.label);
  }

  return classifier;
}

module.exports = { NaiveBayesClassifier, createIntentClassifier };
```

---

## 8. Response Generator

```javascript
// src/nlp/responseGenerator.js

class ResponseGenerator {
  constructor() {
    this.templates = {
      greeting: [
        'สวัสดีครับ! ยินดีให้บริการ มีอะไรให้ช่วยไหม? 😊',
        'หวัดดีครับ! วันนี้มีอะไรให้ช่วยไหมครับ?',
        'ยินดีต้อนรับครับ! ต้องการอะไรไหมครับ?',
      ],
      goodbye: [
        'ขอบคุณที่ใช้บริการครับ ยินดีให้บริการเสมอนะครับ 😊',
        'สวัสดีครับ แวะมาใหม่นะครับ',
        'ขอบคุณมากครับ ยินดีให้บริการครับ!',
      ],
      price_inquiry: [
        'ขอทราบชื่อสินค้าที่ต้องการทราบราคาได้เลยครับ',
        'สินค้าที่สนใจคืออะไรครับ จะได้แจ้งราคาให้ถูกต้อง',
      ],
      product_inquiry: [
        'เรามีสินค้าหลากหลายนะครับ ต้องการดูหมวดหมู่ไหนครับ?',
        'ยินดีแนะนำสินค้าครับ สนใจประเภทไหนเป็นพิเศษครับ?',
      ],
      order: [
        'ยินดีรับออเดอร์ครับ! กรุณาระบุสินค้าและจำนวนที่ต้องการครับ',
        'รับทราบครับ! ต้องการสินค้าอะไรบ้างครับ?',
      ],
      support: [
        'ขอโทษสำหรับความไม่สะดวกนะครับ กรุณาแจ้งรายละเอียดเพิ่มเติมครับ',
        'เข้าใจครับ ขอทราบรายละเอียดปัญหาเพิ่มเติมเพื่อช่วยได้ถูกต้องนะครับ',
      ],
      default: [
        'ขอโทษนะครับ ไม่เข้าใจคำถาม กรุณาถามใหม่อีกครั้งครับ',
        'กรุณาอธิบายเพิ่มเติมได้เลยครับ',
        'ขอโทษครับ ไม่แน่ใจว่าต้องการอะไร ช่วยอธิบายได้ไหมครับ?',
      ],
    };
  }

  // สร้าง Response
  generate(intent, context = {}) {
    const templates = this.templates[intent] || this.templates.default;
    const template = templates[Math.floor(Math.random() * templates.length)];

    return this.fillTemplate(template, context);
  }

  // แทนค่าใน Template
  fillTemplate(template, context) {
    let response = template;

    for (const [key, value] of Object.entries(context)) {
      response = response.replace(new RegExp(`{${key}}`, 'g'), value);
    }

    return response;
  }

  // สร้าง Response จาก Sentiment
  generateWithSentiment(intent, sentiment, context = {}) {
    const base = this.generate(intent, context);

    if (sentiment.label === 'negative' && sentiment.magnitude > 0.5) {
      return `ขอโทษสำหรับความไม่สะดวกนะครับ ${base}`;
    }

    if (sentiment.label === 'positive') {
      return `${base} 😊`;
    }

    return base;
  }

  // เพิ่ม Template
  addTemplate(intent, template) {
    if (!this.templates[intent]) {
      this.templates[intent] = [];
    }
    this.templates[intent].push(template);
  }
}

module.exports = new ResponseGenerator();
```

---

## 9. NLP Pipeline รวม

```javascript
// src/nlp/pipeline.js
const thaiPreprocessor = require('./thaiPreprocessor');
const intentDetector = require('./intentDetector');
const entityExtractor = require('./entityExtractor');
const sentimentAnalyzer = require('./sentimentAnalyzer');
const responseGenerator = require('./responseGenerator');
const logger = require('../utils/logger');

class NLPPipeline {
  constructor() {
    this.isReady = false;
  }

  async initialize() {
    try {
      // โหลด NLP Model หรือเทรนใหม่
      const loaded = await intentDetector.load();
      if (!loaded) {
        await intentDetector.train();
        await intentDetector.save();
      }
      this.isReady = true;
      logger.info('NLP Pipeline initialized');
    } catch (error) {
      logger.error('NLP Pipeline initialization failed', { error: error.message });
      throw error;
    }
  }

  // ประมวลผลข้อความ
  async process(text, userId = null, conversationContext = {}) {
    if (!this.isReady) {
      await this.initialize();
    }

    const startTime = Date.now();

    try {
      // Step 1: Detect Language
      const language = thaiPreprocessor.detectLanguage(text);

      // Step 2: Preprocess
      const preprocessed = thaiPreprocessor.preprocess(text);

      // Step 3: Intent Detection
      const intentResult = await intentDetector.detect(
        preprocessed.normalized,
        language
      );

      // Step 4: Entity Extraction
      const entities = entityExtractor.extract(text);

      // Step 5: Sentiment Analysis
      const sentiment = sentimentAnalyzer.analyze(text);

      // Step 6: Generate Response
      const response = responseGenerator.generateWithSentiment(
        intentResult.intent,
        sentiment,
        {
          userName: conversationContext.userName || 'คุณ',
          ...entities,
        }
      );

      const processingTime = Date.now() - startTime;

      logger.debug('NLP Processing complete', {
        userId,
        intent: intentResult.intent,
        confidence: intentResult.score,
        sentiment: sentiment.label,
        processingTimeMs: processingTime,
      });

      return {
        input: {
          original: text,
          normalized: preprocessed.normalized,
          language,
        },
        intent: intentResult,
        entities,
        sentiment,
        response,
        metadata: {
          processingTimeMs: processingTime,
          pipeline: 'v1',
        },
      };
    } catch (error) {
      logger.error('NLP Processing error', {
        error: error.message,
        text: text.substring(0, 100),
      });
      throw error;
    }
  }

  // เทรนซ้ำด้วยข้อมูลใหม่
  async retrain(newData) {
    logger.info('Retraining NLP model with new data...');

    await intentDetector.addFromHistory(newData);
    await intentDetector.train();
    await intentDetector.save();

    logger.info('NLP model retrained successfully');
  }
}

module.exports = new NLPPipeline();
```

---

## 10. Integration กับ LINE Bot

```javascript
// src/handlers/nlpHandler.js
const nlpPipeline = require('../nlp/pipeline');
const logger = require('../utils/logger');

class NLPHandler {
  async handleMessage(userId, text, replyToken, lineClient) {
    try {
      // ประมวลผลด้วย NLP Pipeline
      const result = await nlpPipeline.process(text, userId);

      // Log สำหรับ Analytics
      logger.info('NLP result', {
        userId,
        intent: result.intent.intent,
        confidence: result.intent.score,
        sentiment: result.sentiment.label,
        language: result.input.language,
      });

      // จัดการตาม Intent
      const response = await this.routeByIntent(
        result,
        userId,
        lineClient
      );

      // ส่งคำตอบ
      if (response) {
        await lineClient.replyMessage({
          replyToken,
          messages: [{ type: 'text', text: response }],
        });
      }

      return result;
    } catch (error) {
      logger.error('NLP Handler error', { error: error.message, userId });
      await lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: 'ขออภัยครับ เกิดข้อผิดพลาด กรุณาลองใหม่' }],
      });
    }
  }

  async routeByIntent(nlpResult, userId, lineClient) {
    const { intent, entities, sentiment, response } = nlpResult;

    switch (intent.intent) {
      case 'order.food':
        return this.handleFoodOrder(entities, userId, lineClient);

      case 'inquiry.price':
        return this.handlePriceInquiry(entities);

      case 'inquiry.location':
        return this.handleLocationInquiry();

      case 'inquiry.hours':
        return this.handleHoursInquiry();

      case 'complaint':
        return this.handleComplaint(sentiment, userId);

      default:
        // ใช้ response ที่ generate ไว้แล้ว
        return response;
    }
  }

  async handleFoodOrder(entities, userId, lineClient) {
    const products = entities.productNames || [];
    const quantities = entities.quantities || [];

    if (products.length === 0) {
      return 'ต้องการสั่งอะไรครับ? กรุณาระบุชื่อสินค้า';
    }

    return `รับทราบครับ! กำลังดำเนินการสั่ง ${products.map((p) => p.name).join(', ')} ให้ครับ`;
  }

  handlePriceInquiry(entities) {
    if (entities.productNames?.length > 0) {
      const product = entities.productNames[0].name;
      return `สินค้า "${product}" ราคา... กรุณาสอบถามพนักงานครับ`;
    }
    return 'สนใจสินค้าอะไรครับ? ยินดีแจ้งราคาให้ครับ';
  }

  handleLocationInquiry() {
    return `📍 ที่อยู่ร้านของเรา:
123/45 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพฯ 10110

⏰ เวลาทำการ:
จันทร์-ศุกร์: 09:00-18:00 น.
เสาร์-อาทิตย์: 10:00-17:00 น.

🗺️ Google Maps: https://maps.google.com/?q=...`;
  }

  handleHoursInquiry() {
    return `⏰ เวลาทำการของเรา:
จันทร์-ศุกร์: 09:00-18:00 น.
เสาร์: 10:00-17:00 น.
อาทิตย์: ปิดทำการ

วันหยุดนักขัตฤกษ์: ปิดทำการ`;
  }

  handleComplaint(sentiment, userId) {
    if (sentiment.label === 'negative') {
      return `ขอโทษสำหรับความไม่สะดวกอย่างยิ่งนะครับ 🙏

กรุณาแจ้งรายละเอียดปัญหาให้ทีมงานทราบครับ:
- ปัญหาที่พบ
- วันเวลาที่เกิดเหตุ
- เลขที่ออเดอร์ (ถ้ามี)

เราจะรีบแก้ไขและติดต่อกลับภายใน 1 ชั่วโมงครับ`;
    }

    return 'ขอบคุณสำหรับ feedback นะครับ เราจะนำไปปรับปรุงบริการครับ 😊';
  }
}

module.exports = new NLPHandler();
```

---

## 11. Training จาก LINE Message History

```javascript
// scripts/trainFromHistory.js
const Conversation = require('../src/models/conversation');
const intentDetector = require('../src/nlp/intentDetector');
const mongoose = require('mongoose');

async function trainFromHistory() {
  await mongoose.connect(process.env.MONGODB_URI);

  // ดึงข้อความที่มี Intent Label แล้ว
  const labeledMessages = await Conversation.aggregate([
    { $unwind: '$messages' },
    { $match: { 'messages.metadata.intent': { $exists: true } } },
    {
      $project: {
        text: '$messages.content',
        intent: '$messages.metadata.intent',
        confidence: '$messages.metadata.intentConfidence',
        language: '$messages.metadata.language',
      },
    },
    { $match: { confidence: { $gte: 0.8 } } },
  ]);

  console.log(`Found ${labeledMessages.length} labeled messages`);

  const added = await intentDetector.addFromHistory(labeledMessages);
  console.log(`Added ${added} documents to training`);

  await intentDetector.train();
  await intentDetector.save('./models/nlp-model-trained.json');

  console.log('Training complete!');
  await mongoose.disconnect();
}

trainFromHistory().catch(console.error);
```

---

## 12. Multi-language Support

```javascript
// src/nlp/multiLanguageProcessor.js
const franc = require('franc-min');

class MultiLanguageProcessor {
  constructor() {
    this.supportedLanguages = ['th', 'en', 'zh', 'ja', 'ko'];
    this.languageNames = {
      th: 'ภาษาไทย',
      en: 'English',
      zh: '中文',
      ja: '日本語',
      ko: '한국어',
    };
  }

  // ตรวจจับภาษา
  detectLanguage(text) {
    try {
      const detected = franc(text, { minLength: 3 });

      // Map franc codes to our codes
      const languageMap = {
        tha: 'th',
        eng: 'en',
        cmn: 'zh',
        jpn: 'ja',
        kor: 'ko',
        und: 'unknown',
      };

      return languageMap[detected] || 'unknown';
    } catch (error) {
      // Fallback: simple character detection
      if (/[฀-๿]/.test(text)) return 'th';
      if (/[一-鿿]/.test(text)) return 'zh';
      if (/[぀-ゟ゠-ヿ]/.test(text)) return 'ja';
      if (/[가-힯]/.test(text)) return 'ko';
      return 'en';
    }
  }

  // สร้าง System Prompt ตามภาษา
  getSystemPrompt(language) {
    const prompts = {
      th: 'คุณคือ AI Assistant ที่ตอบภาษาไทย ตอบกระชับ ชัดเจน และเป็นมิตร',
      en: 'You are an AI Assistant. Respond in English, concisely and helpfully.',
      zh: '你是一个AI助手。用中文简洁、友好地回答问题。',
      ja: 'あなたはAIアシスタントです。日本語で簡潔かつ親切に返答してください。',
      ko: '당신은 AI 어시스턴트입니다. 한국어로 간결하고 친절하게 답변해 주세요.',
    };

    return prompts[language] || prompts.en;
  }

  // ตรวจสอบว่า Language Support
  isSupported(language) {
    return this.supportedLanguages.includes(language);
  }
}

module.exports = new MultiLanguageProcessor();
```

---

## 13. Analytics Dashboard

```javascript
// src/services/nlpAnalytics.js

class NLPAnalytics {
  constructor(mongodb) {
    this.db = mongodb;
    this.collection = this.db.collection('nlp_analytics');
  }

  // บันทึก NLP Result
  async recordResult(userId, nlpResult) {
    await this.collection.insertOne({
      userId,
      intent: nlpResult.intent.intent,
      confidence: nlpResult.intent.score,
      sentiment: nlpResult.sentiment.label,
      language: nlpResult.input.language,
      processingTime: nlpResult.metadata.processingTimeMs,
      timestamp: new Date(),
    });
  }

  // ดูสถิติ Intent
  async getIntentStats(days = 30) {
    const cutoff = new Date(Date.now() - days * 24 * 3600 * 1000);

    return this.collection.aggregate([
      { $match: { timestamp: { $gte: cutoff } } },
      {
        $group: {
          _id: '$intent',
          count: { $sum: 1 },
          avgConfidence: { $avg: '$confidence' },
        },
      },
      { $sort: { count: -1 } },
    ]).toArray();
  }

  // ดูสถิติ Sentiment
  async getSentimentStats(days = 30) {
    const cutoff = new Date(Date.now() - days * 24 * 3600 * 1000);

    return this.collection.aggregate([
      { $match: { timestamp: { $gte: cutoff } } },
      {
        $group: {
          _id: '$sentiment',
          count: { $sum: 1 },
          percentage: { $avg: 1 },
        },
      },
    ]).toArray();
  }
}

module.exports = NLPAnalytics;
```

---

## สรุปบทที่ 62

ในบทนี้เราได้เรียนรู้:

1. **Thai Preprocessing** - Tokenization, Normalization สำหรับภาษาไทย
2. **Intent Detection** - การใช้ node-nlp และ Naive Bayes Classifier
3. **Entity Extraction** - ดึงวันที่ เวลา เบอร์โทร ราคา จากข้อความ
4. **Sentiment Analysis** - วิเคราะห์อารมณ์และความรู้สึกภาษาไทย
5. **Google Cloud NLP** - การใช้ API สำหรับ NLP ระดับ Enterprise
6. **Custom Classifier** - สร้างและเทรน Model เอง
7. **Training จาก History** - ปรับปรุง Model จากข้อมูลจริง
8. **Multi-language** - รองรับหลายภาษา
9. **NLP Pipeline** - Pipeline สมบูรณ์จาก Input ถึง Response

---

*ต่อไป: Part 63 - Dialogflow Integration*
