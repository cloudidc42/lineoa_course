# Part 64: Image Recognition สำหรับ LINE Bot

## บทนำ

Image Recognition เปิดโอกาสให้ LINE Bot มีความสามารถในการ "มองเห็น" และวิเคราะห์รูปภาพที่ผู้ใช้ส่งมา ตั้งแต่การจดจำอาหาร ตรวจสอบเอกสาร ไปจนถึงการสแกน QR Code บทนี้ครอบคลุม Integration กับ Google Vision API และ AWS Rekognition พร้อม Use Cases จริงในธุรกิจ

## เนื้อหาที่จะเรียนรู้

1. Image Events ใน LINE
2. การดาวน์โหลดรูปภาพจาก LINE API
3. Google Vision API
4. AWS Rekognition
5. Food Recognition Bot
6. Document Scanner Bot
7. Product Identification Bot
8. Face Detection
9. QR/Barcode Scanning
10. Implementation ครบถ้วน

---

## สถาปัตยกรรมระบบ

```
┌───────────────────────────────────────────────────────────────────┐
│                        LINE Platform                              │
│  User ─ส่งรูป─► LINE App ─► LINE Server ─► Webhook              │
└───────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                     Node.js Backend                               │
│                                                                   │
│  ┌─────────────────┐    ┌──────────────────┐                     │
│  │  Image Event    │───▶│  Image Download  │                     │
│  │  Handler        │    │  Service         │                     │
│  └─────────────────┘    └──────────────────┘                     │
│                                 │                                 │
│           ┌─────────────────────┼─────────────────────┐          │
│           ▼                     ▼                     ▼          │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐   │
│  │  Google      │    │  AWS             │    │  OpenAI      │   │
│  │  Vision API  │    │  Rekognition     │    │  Vision API  │   │
│  └──────────────┘    └──────────────────┘    └──────────────┘   │
│           │                     │                     │           │
│           └─────────────────────┴─────────────────────┘          │
│                                 │                                 │
│                                 ▼                                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Response Builder                              │  │
│  │  Food Info | Document Data | Product Info | Face Data      │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

---

## 1. ติดตั้ง Dependencies

```bash
npm init -y

# Core
npm install @line/bot-sdk express dotenv

# Google Vision
npm install @google-cloud/vision

# AWS
npm install @aws-sdk/client-rekognition

# Image Processing
npm install sharp jimp multer

# Storage
npm install @aws-sdk/client-s3 @google-cloud/storage

# QR/Barcode
npm install jsqr

# Utilities
npm install uuid axios winston node-cache
```

```env
# .env
LINE_CHANNEL_ACCESS_TOKEN=your_line_token
LINE_CHANNEL_SECRET=your_line_secret

# Google Vision
GOOGLE_APPLICATION_CREDENTIALS=./credentials/google-creds.json

# AWS
AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
AWS_REGION=ap-southeast-1

# OpenAI (for Vision)
OPENAI_API_KEY=your_openai_key

# Storage
IMAGE_STORAGE_BUCKET=your-bucket-name

PORT=3000
```

---

## 2. Image Download Service

```javascript
// src/services/imageDownloader.js
const axios = require('axios');
const sharp = require('sharp');
const { createHash } = require('crypto');
const path = require('path');
const fs = require('fs').promises;
const logger = require('../utils/logger');

class ImageDownloaderService {
  constructor() {
    this.tempDir = './temp/images';
    this.maxImageSize = 10 * 1024 * 1024; // 10MB
    this.supportedFormats = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
  }

  async initialize() {
    await fs.mkdir(this.tempDir, { recursive: true });
  }

  // ดาวน์โหลดรูปภาพจาก LINE
  async downloadFromLINE(messageId, channelAccessToken) {
    const url = `https://api-data.line.me/v2/bot/message/${messageId}/content`;

    try {
      const response = await axios.get(url, {
        headers: {
          Authorization: `Bearer ${channelAccessToken}`,
        },
        responseType: 'arraybuffer',
        timeout: 30000,
        maxContentLength: this.maxImageSize,
      });

      const contentType = response.headers['content-type'];
      if (!this.supportedFormats.includes(contentType?.split(';')[0])) {
        throw new Error(`Unsupported image format: ${contentType}`);
      }

      const buffer = Buffer.from(response.data);
      const hash = createHash('md5').update(buffer).digest('hex');
      const filename = `${hash}.jpg`;
      const filepath = path.join(this.tempDir, filename);

      // แปลงและ optimize รูปภาพ
      const processedBuffer = await this.processImage(buffer);
      await fs.writeFile(filepath, processedBuffer);

      logger.info('Image downloaded', {
        messageId,
        size: buffer.length,
        filename,
      });

      return {
        filepath,
        buffer: processedBuffer,
        filename,
        originalSize: buffer.length,
        processedSize: processedBuffer.length,
        contentType: 'image/jpeg',
      };
    } catch (error) {
      logger.error('Image download error', {
        error: error.message,
        messageId,
      });
      throw error;
    }
  }

  // Process รูปภาพ
  async processImage(buffer, options = {}) {
    const {
      maxWidth = 1920,
      maxHeight = 1920,
      quality = 85,
      format = 'jpeg',
    } = options;

    return sharp(buffer)
      .resize(maxWidth, maxHeight, {
        fit: 'inside',
        withoutEnlargement: true,
      })
      .toFormat(format, { quality })
      .toBuffer();
  }

  // ดึง Metadata รูปภาพ
  async getMetadata(buffer) {
    const metadata = await sharp(buffer).metadata();
    return {
      width: metadata.width,
      height: metadata.height,
      format: metadata.format,
      size: metadata.size,
      channels: metadata.channels,
      hasAlpha: metadata.hasAlpha,
    };
  }

  // แปลงเป็น Base64
  async toBase64(buffer) {
    return buffer.toString('base64');
  }

  // ลบไฟล์ชั่วคราว
  async cleanup(filepath) {
    try {
      await fs.unlink(filepath);
    } catch (error) {
      logger.warn('Failed to cleanup temp file', { filepath, error: error.message });
    }
  }

  // ตรวจสอบว่าเป็นรูปภาพ
  async isValidImage(buffer) {
    try {
      const metadata = await sharp(buffer).metadata();
      return !!metadata.format;
    } catch {
      return false;
    }
  }
}

module.exports = new ImageDownloaderService();
```

---

## 3. Google Vision API Service

```javascript
// src/services/googleVision.js
const vision = require('@google-cloud/vision');
const logger = require('../utils/logger');

class GoogleVisionService {
  constructor() {
    this.client = new vision.ImageAnnotatorClient({
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  // วิเคราะห์รูปภาพแบบครบถ้วน
  async analyze(imageBuffer, features = []) {
    const defaultFeatures = [
      { type: 'LABEL_DETECTION', maxResults: 20 },
      { type: 'OBJECT_LOCALIZATION', maxResults: 10 },
      { type: 'TEXT_DETECTION' },
      { type: 'SAFE_SEARCH_DETECTION' },
      { type: 'IMAGE_PROPERTIES' },
    ];

    const usedFeatures = features.length > 0 ? features : defaultFeatures;

    try {
      const [result] = await this.client.annotateImage({
        image: { content: imageBuffer.toString('base64') },
        features: usedFeatures,
      });

      return this.parseResult(result);
    } catch (error) {
      logger.error('Google Vision error', { error: error.message });
      throw error;
    }
  }

  // ตรวจจับ Labels (วัตถุ/สถานที่/กิจกรรม)
  async detectLabels(imageBuffer) {
    const [result] = await this.client.labelDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    return result.labelAnnotations?.map((label) => ({
      description: label.description,
      score: label.score,
      topicality: label.topicality,
    })) || [];
  }

  // ตรวจจับ Text (OCR)
  async detectText(imageBuffer) {
    const [result] = await this.client.textDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    const annotations = result.textAnnotations;
    if (!annotations || annotations.length === 0) {
      return { fullText: '', blocks: [] };
    }

    return {
      fullText: annotations[0].description || '',
      blocks: annotations.slice(1).map((block) => ({
        text: block.description,
        boundingBox: block.boundingPoly?.vertices,
        confidence: block.confidence,
      })),
      language: annotations[0].locale,
    };
  }

  // ตรวจจับเอกสาร (Document OCR - ดีกว่า textDetection)
  async detectDocument(imageBuffer) {
    const [result] = await this.client.documentTextDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    const fullText = result.fullTextAnnotation;
    if (!fullText) return { text: '', pages: [] };

    return {
      text: fullText.text,
      confidence: fullText.pages?.[0]?.confidence,
      language: fullText.pages?.[0]?.property?.detectedLanguages?.[0]?.languageCode,
      pages: fullText.pages?.map((page) => ({
        width: page.width,
        height: page.height,
        blocks: page.blocks?.map((block) => ({
          text: block.paragraphs?.map((p) =>
            p.words?.map((w) =>
              w.symbols?.map((s) => s.text).join('')
            ).join(' ')
          ).join('\n'),
          confidence: block.confidence,
          blockType: block.blockType,
        })),
      })) || [],
    };
  }

  // ตรวจจับใบหน้า
  async detectFaces(imageBuffer) {
    const [result] = await this.client.faceDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    return result.faceAnnotations?.map((face) => ({
      confidence: face.detectionConfidence,
      joyLikelihood: face.joyLikelihood,
      sorrowLikelihood: face.sorrowLikelihood,
      angerLikelihood: face.angerLikelihood,
      surpriseLikelihood: face.surpriseLikelihood,
      underExposedLikelihood: face.underExposedLikelihood,
      blurredLikelihood: face.blurredLikelihood,
      headwearLikelihood: face.headwearLikelihood,
      boundingPoly: face.boundingPoly?.vertices,
      landmarks: face.landmarks?.map((l) => ({
        type: l.type,
        position: l.position,
      })),
    })) || [];
  }

  // ตรวจจับ Logos/แบรนด์
  async detectLogos(imageBuffer) {
    const [result] = await this.client.logoDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    return result.logoAnnotations?.map((logo) => ({
      description: logo.description,
      score: logo.score,
      boundingPoly: logo.boundingPoly?.vertices,
    })) || [];
  }

  // ตรวจสอบความเหมาะสม (Safe Search)
  async checkSafeSearch(imageBuffer) {
    const [result] = await this.client.safeSearchDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    const safeSearch = result.safeSearchAnnotation;
    return {
      adult: safeSearch?.adult,
      spoof: safeSearch?.spoof,
      medical: safeSearch?.medical,
      violence: safeSearch?.violence,
      racy: safeSearch?.racy,
      isUnsafe: ['LIKELY', 'VERY_LIKELY'].includes(safeSearch?.adult) ||
        ['LIKELY', 'VERY_LIKELY'].includes(safeSearch?.violence),
    };
  }

  // Web Detection (ค้นหารูปที่คล้ายกันในเว็บ)
  async detectWeb(imageBuffer) {
    const [result] = await this.client.webDetection({
      image: { content: imageBuffer.toString('base64') },
    });

    const web = result.webDetection;
    return {
      bestGuessLabels: web?.bestGuessLabels?.map((l) => l.label) || [],
      webEntities: web?.webEntities?.map((e) => ({
        description: e.description,
        score: e.score,
      })) || [],
      visuallySimilarImages: web?.visuallySimilarImages?.map((i) => i.url) || [],
      pagesWithMatchingImages: web?.pagesWithMatchingImages?.slice(0, 5).map((p) => ({
        url: p.url,
        pageTitle: p.pageTitle,
      })) || [],
    };
  }

  // ตรวจจับ Objects
  async detectObjects(imageBuffer) {
    const [result] = await this.client.objectLocalization({
      image: { content: imageBuffer.toString('base64') },
    });

    return result.localizedObjectAnnotations?.map((obj) => ({
      name: obj.name,
      score: obj.score,
      boundingPoly: obj.boundingPoly?.normalizedVertices,
    })) || [];
  }

  // Parse ผลลัพธ์รวม
  parseResult(result) {
    return {
      labels: result.labelAnnotations?.map((l) => ({
        label: l.description,
        confidence: l.score,
      })) || [],
      objects: result.localizedObjectAnnotations?.map((o) => ({
        name: o.name,
        confidence: o.score,
      })) || [],
      text: result.textAnnotations?.[0]?.description || '',
      safeSearch: result.safeSearchAnnotation,
      colors: result.imagePropertiesAnnotation?.dominantColors?.colors?.map((c) => ({
        color: c.color,
        score: c.score,
        pixelFraction: c.pixelFraction,
      })) || [],
    };
  }
}

module.exports = new GoogleVisionService();
```

---

## 4. AWS Rekognition Service

```javascript
// src/services/awsRekognition.js
const { RekognitionClient, DetectLabelsCommand,
  DetectTextCommand, DetectFacesCommand,
  DetectModerationLabelsCommand, RecognizeCelebritiesCommand,
  DetectCustomLabelsCommand } = require('@aws-sdk/client-rekognition');
const logger = require('../utils/logger');

class AWSRekognitionService {
  constructor() {
    this.client = new RekognitionClient({
      region: process.env.AWS_REGION,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      },
    });
  }

  // ตรวจจับ Labels
  async detectLabels(imageBuffer, options = {}) {
    const { maxLabels = 20, minConfidence = 70 } = options;

    try {
      const command = new DetectLabelsCommand({
        Image: { Bytes: imageBuffer },
        MaxLabels: maxLabels,
        MinConfidence: minConfidence,
      });

      const response = await this.client.send(command);

      return response.Labels?.map((label) => ({
        name: label.Name,
        confidence: label.Confidence,
        categories: label.Categories?.map((c) => c.Name) || [],
        parents: label.Parents?.map((p) => p.Name) || [],
        instances: label.Instances?.map((inst) => ({
          boundingBox: inst.BoundingBox,
          confidence: inst.Confidence,
        })) || [],
      })) || [];
    } catch (error) {
      logger.error('AWS Rekognition DetectLabels error', { error: error.message });
      throw error;
    }
  }

  // ตรวจจับ Text
  async detectText(imageBuffer) {
    const command = new DetectTextCommand({
      Image: { Bytes: imageBuffer },
    });

    const response = await this.client.send(command);

    const lines = response.TextDetections?.filter((t) => t.Type === 'LINE') || [];
    const words = response.TextDetections?.filter((t) => t.Type === 'WORD') || [];

    return {
      fullText: lines.map((l) => l.DetectedText).join('\n'),
      lines: lines.map((l) => ({
        text: l.DetectedText,
        confidence: l.Confidence,
        boundingBox: l.Geometry?.BoundingBox,
      })),
      words: words.map((w) => ({
        text: w.DetectedText,
        confidence: w.Confidence,
      })),
    };
  }

  // ตรวจจับใบหน้า
  async detectFaces(imageBuffer) {
    const command = new DetectFacesCommand({
      Image: { Bytes: imageBuffer },
      Attributes: ['ALL'],
    });

    const response = await this.client.send(command);

    return response.FaceDetails?.map((face) => ({
      confidence: face.Confidence,
      ageRange: face.AgeRange,
      gender: face.Gender,
      emotions: face.Emotions?.map((e) => ({
        type: e.Type,
        confidence: e.Confidence,
      })).sort((a, b) => b.confidence - a.confidence) || [],
      smile: face.Smile,
      eyeglasses: face.Eyeglasses,
      sunglasses: face.Sunglasses,
      beard: face.Beard,
      mustache: face.Mustache,
      eyesOpen: face.EyesOpen,
      mouthOpen: face.MouthOpen,
      pose: face.Pose,
      quality: face.Quality,
      boundingBox: face.BoundingBox,
    })) || [];
  }

  // ตรวจสอบ Content Moderation
  async detectModerationLabels(imageBuffer, minConfidence = 60) {
    const command = new DetectModerationLabelsCommand({
      Image: { Bytes: imageBuffer },
      MinConfidence: minConfidence,
    });

    const response = await this.client.send(command);

    const labels = response.ModerationLabels || [];
    const isUnsafe = labels.some((l) =>
      ['Explicit Nudity', 'Violence', 'Hate Symbols'].includes(l.ParentName)
    );

    return {
      isUnsafe,
      labels: labels.map((l) => ({
        name: l.Name,
        confidence: l.Confidence,
        parentName: l.ParentName,
      })),
    };
  }

  // จดจำดารา/บุคคลมีชื่อเสียง
  async recognizeCelebrities(imageBuffer) {
    const command = new RecognizeCelebritiesCommand({
      Image: { Bytes: imageBuffer },
    });

    const response = await this.client.send(command);

    return {
      celebrities: response.CelebrityFaces?.map((celeb) => ({
        name: celeb.Name,
        confidence: celeb.MatchConfidence,
        urls: celeb.Urls || [],
        boundingBox: celeb.Face?.BoundingBox,
      })) || [],
      unrecognized: response.UnrecognizedFaces?.length || 0,
    };
  }

  // ตรวจจับสินค้า (ต้องสร้าง Custom Model ก่อน)
  async detectCustomLabels(imageBuffer, projectVersionArn) {
    const command = new DetectCustomLabelsCommand({
      Image: { Bytes: imageBuffer },
      ProjectVersionArn: projectVersionArn,
      MinConfidence: 70,
    });

    const response = await this.client.send(command);

    return response.CustomLabels?.map((label) => ({
      name: label.Name,
      confidence: label.Confidence,
      boundingBox: label.Geometry?.BoundingBox,
    })) || [];
  }
}

module.exports = new AWSRekognitionService();
```

---

## 5. Food Recognition Bot

```javascript
// src/bots/foodRecognitionBot.js
const googleVision = require('../services/googleVision');
const awsRekognition = require('../services/awsRekognition');
const imageDownloader = require('../services/imageDownloader');
const logger = require('../utils/logger');

// Database อาหารไทย
const THAI_FOOD_DATABASE = {
  'Tom Yum': {
    thai: 'ต้มยำ',
    calories: 120,
    description: 'ซุปเปรี้ยวเผ็ดแบบไทย',
    allergens: ['กุ้ง', 'เห็ด'],
    price: '60-150 บาท',
  },
  'Pad Thai': {
    thai: 'ผัดไทย',
    calories: 450,
    description: 'ก๋วยเตี๋ยวผัดแบบไทย',
    allergens: ['เต้าหู้', 'กุ้ง', 'ถั่วลิสง'],
    price: '50-120 บาท',
  },
  'Green Curry': {
    thai: 'แกงเขียวหวาน',
    calories: 200,
    description: 'แกงเขียวหวานไก่หรือเนื้อ',
    allergens: ['นม', 'น้ำมันมะพร้าว'],
    price: '60-150 บาท',
  },
  'Mango Sticky Rice': {
    thai: 'ข้าวเหนียวมะม่วง',
    calories: 350,
    description: 'ขนมหวานไทยยอดนิยม',
    allergens: ['นม'],
    price: '60-100 บาท',
  },
  'Som Tum': {
    thai: 'ส้มตำ',
    calories: 100,
    description: 'ยำมะละกอ',
    allergens: ['กุ้งแห้ง', 'ถั่วลิสง'],
    price: '40-80 บาท',
  },
};

class FoodRecognitionBot {
  // วิเคราะห์รูปอาหาร
  async analyzeFood(imageBuffer, userId) {
    logger.info('Analyzing food image', { userId });

    // ใช้ Google Vision ตรวจจับ Labels
    const labels = await googleVision.detectLabels(imageBuffer);

    // ใช้ AWS Rekognition ตรวจจับ Labels ด้วย (สำหรับ accuracy สูงขึ้น)
    const awsLabels = await awsRekognition.detectLabels(imageBuffer, {
      minConfidence: 70,
    });

    // รวม Labels จากทั้งสอง service
    const allLabels = [
      ...labels.map((l) => ({ source: 'google', name: l.label, confidence: l.confidence })),
      ...awsLabels.map((l) => ({ source: 'aws', name: l.name, confidence: l.confidence / 100 })),
    ];

    // ค้นหาอาหารที่ตรงกับ Database
    const detectedFoods = this.matchFoodDatabase(allLabels);

    // ตรวจสอบความปลอดภัย
    const safeSearch = await googleVision.checkSafeSearch(imageBuffer);
    if (safeSearch.isUnsafe) {
      return {
        safe: false,
        message: 'ไม่สามารถวิเคราะห์รูปภาพนี้ได้',
      };
    }

    return {
      safe: true,
      detected: detectedFoods,
      labels: allLabels.slice(0, 5),
      analysis: this.buildFoodAnalysis(detectedFoods, allLabels),
    };
  }

  // จับคู่กับ Database อาหาร
  matchFoodDatabase(labels) {
    const detectedFoods = [];

    for (const [englishName, foodInfo] of Object.entries(THAI_FOOD_DATABASE)) {
      // ตรวจสอบว่า label ตรงกับ food name
      const matchedLabel = labels.find((l) =>
        l.name.toLowerCase().includes(englishName.toLowerCase()) ||
        englishName.toLowerCase().includes(l.name.toLowerCase())
      );

      if (matchedLabel) {
        detectedFoods.push({
          ...foodInfo,
          englishName,
          confidence: matchedLabel.confidence,
        });
      }
    }

    return detectedFoods.sort((a, b) => b.confidence - a.confidence);
  }

  // สร้าง Analysis Text
  buildFoodAnalysis(detectedFoods, allLabels) {
    if (detectedFoods.length > 0) {
      const topFood = detectedFoods[0];
      return {
        found: true,
        message: `🍽️ ตรวจพบ: ${topFood.thai} (${topFood.englishName})\n\n` +
          `📊 แคลอรี่: ~${topFood.calories} kcal\n` +
          `📝 คำอธิบาย: ${topFood.description}\n` +
          `⚠️ สารก่อภูมิแพ้: ${topFood.allergens.join(', ')}\n` +
          `💰 ราคาโดยประมาณ: ${topFood.price}`,
        food: topFood,
      };
    }

    // ไม่พบในฐานข้อมูล แต่อาจเป็นอาหาร
    const foodLabels = allLabels.filter((l) =>
      ['Food', 'Cuisine', 'Dish', 'Meal', 'Recipe'].some((foodWord) =>
        l.name.toLowerCase().includes(foodWord.toLowerCase())
      )
    );

    if (foodLabels.length > 0) {
      return {
        found: false,
        message: `🍽️ พบรูปอาหาร แต่ไม่สามารถระบุชนิดได้แน่ชัด\n\n` +
          `สิ่งที่ตรวจพบ:\n${foodLabels.slice(0, 3).map((l) =>
            `• ${l.name} (${(l.confidence * 100).toFixed(0)}%)`
          ).join('\n')}`,
        food: null,
      };
    }

    return {
      found: false,
      message: '❌ ไม่พบอาหารในรูปภาพ กรุณาส่งรูปอาหารที่ต้องการวิเคราะห์',
      food: null,
    };
  }

  // สร้าง Flex Message สำหรับ LINE
  buildFoodFlexMessage(analysis, userId) {
    if (!analysis.safe || !analysis.detected.length) {
      return null;
    }

    const food = analysis.detected[0];
    return {
      type: 'flex',
      altText: `วิเคราะห์อาหาร: ${food.thai}`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '🍽️ วิเคราะห์อาหาร',
            weight: 'bold',
            size: 'lg',
            color: '#ffffff',
          }],
          backgroundColor: '#FF6B35',
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: food.thai,
              weight: 'bold',
              size: 'xl',
            },
            {
              type: 'text',
              text: food.englishName,
              color: '#888888',
            },
            { type: 'separator', margin: 'md' },
            {
              type: 'box',
              layout: 'vertical',
              margin: 'md',
              contents: [
                {
                  type: 'box',
                  layout: 'horizontal',
                  contents: [
                    { type: 'text', text: '🔥 แคลอรี่', flex: 2, color: '#555' },
                    { type: 'text', text: `${food.calories} kcal`, flex: 1 },
                  ],
                },
                {
                  type: 'box',
                  layout: 'horizontal',
                  margin: 'sm',
                  contents: [
                    { type: 'text', text: '💰 ราคา', flex: 2, color: '#555' },
                    { type: 'text', text: food.price, flex: 1 },
                  ],
                },
              ],
            },
            { type: 'separator', margin: 'md' },
            {
              type: 'text',
              text: `⚠️ สารก่อภูมิแพ้: ${food.allergens.join(', ')}`,
              size: 'sm',
              color: '#ff6b6b',
              margin: 'md',
              wrap: true,
            },
          ],
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'button',
            style: 'primary',
            action: {
              type: 'message',
              label: 'วิเคราะห์อีกรูป',
              text: 'ส่งรูปอาหารอีกรูปเพื่อวิเคราะห์',
            },
            color: '#FF6B35',
          }],
        },
      },
    };
  }
}

module.exports = new FoodRecognitionBot();
```

---

## 6. Document Scanner Bot

```javascript
// src/bots/documentScannerBot.js
const googleVision = require('../services/googleVision');
const logger = require('../utils/logger');

class DocumentScannerBot {
  // สแกนเอกสารทั่วไป
  async scanDocument(imageBuffer) {
    logger.info('Scanning document');

    // ใช้ Document OCR สำหรับความแม่นยำสูง
    const ocrResult = await googleVision.detectDocument(imageBuffer);

    // ตรวจจับ Labels เพื่อดูประเภทเอกสาร
    const labels = await googleVision.detectLabels(imageBuffer);

    // ตรวจจับ Objects สำหรับตำแหน่งข้อความ
    const objects = await googleVision.detectObjects(imageBuffer);

    const documentType = this.classifyDocumentType(labels, ocrResult.text);

    return {
      text: ocrResult.text,
      documentType,
      confidence: ocrResult.confidence,
      language: ocrResult.language,
      extractedData: await this.extractStructuredData(ocrResult.text, documentType),
    };
  }

  // จำแนกประเภทเอกสาร
  classifyDocumentType(labels, text) {
    const labelNames = labels.map((l) => l.label?.toLowerCase() || '');
    const textLower = text.toLowerCase();

    // บัตรประชาชน/ID Card
    if (textLower.includes('บัตรประจำตัวประชาชน') || textLower.includes('national id') ||
      labelNames.some((l) => l.includes('identification') || l.includes('id card'))) {
      return 'thai_id_card';
    }

    // ใบสูติบัตร
    if (textLower.includes('ใบสูติบัตร') || textLower.includes('birth certificate')) {
      return 'birth_certificate';
    }

    // ใบกำกับภาษี/ใบเสร็จ
    if (textLower.includes('ใบกำกับภาษี') || textLower.includes('tax invoice') ||
      textLower.includes('ใบเสร็จ')) {
      return 'tax_invoice';
    }

    // สัญญา
    if (textLower.includes('สัญญา') || textLower.includes('contract') ||
      textLower.includes('agreement')) {
      return 'contract';
    }

    // ประวัติการรักษา
    if (textLower.includes('โรงพยาบาล') || textLower.includes('hospital') ||
      textLower.includes('ผู้ป่วย')) {
      return 'medical_document';
    }

    // ใบสมัครงาน/Resume
    if (textLower.includes('ประวัติย่อ') || textLower.includes('resume') ||
      textLower.includes('curriculum vitae')) {
      return 'resume';
    }

    return 'general_document';
  }

  // ดึงข้อมูลที่มีโครงสร้าง
  async extractStructuredData(text, documentType) {
    switch (documentType) {
      case 'thai_id_card':
        return this.extractThaiIDData(text);
      case 'tax_invoice':
        return this.extractInvoiceData(text);
      case 'resume':
        return this.extractResumeData(text);
      default:
        return this.extractGeneralData(text);
    }
  }

  // ดึงข้อมูลบัตรประชาชน
  extractThaiIDData(text) {
    const data = {};

    // ชื่อ
    const nameMatch = text.match(/(?:ชื่อ|Name)[:\s]+([^\n]+)/);
    if (nameMatch) data.name = nameMatch[1].trim();

    // เลขบัตร
    const idMatch = text.match(/\d[\d\s-]{12,14}\d/);
    if (idMatch) data.idNumber = idMatch[0].replace(/[\s-]/g, '');

    // วันเกิด
    const birthMatch = text.match(/(?:เกิด|Born)[:\s]+([^\n]+)/);
    if (birthMatch) data.birthDate = birthMatch[1].trim();

    // ที่อยู่
    const addressMatch = text.match(/(?:ที่อยู่|Address)[:\s]+([^\n]+(?:\n[^\n]+)*)/);
    if (addressMatch) data.address = addressMatch[1].trim();

    return data;
  }

  // ดึงข้อมูล Invoice
  extractInvoiceData(text) {
    const data = {};

    // เลขที่ใบกำกับ
    const invoiceNoMatch = text.match(/(?:เลขที่|Invoice No)[:\s]+([^\n]+)/i);
    if (invoiceNoMatch) data.invoiceNo = invoiceNoMatch[1].trim();

    // วันที่
    const dateMatch = text.match(/(?:วันที่|Date)[:\s]+([^\n]+)/i);
    if (dateMatch) data.date = dateMatch[1].trim();

    // ยอดรวม
    const totalMatch = text.match(/(?:ยอดรวม|Total|รวมทั้งสิ้น)[:\s]+([0-9,]+(?:\.\d{2})?)/i);
    if (totalMatch) data.total = parseFloat(totalMatch[1].replace(',', ''));

    // ภาษี
    const vatMatch = text.match(/(?:ภาษีมูลค่าเพิ่ม|VAT)[:\s]+([0-9,]+(?:\.\d{2})?)/i);
    if (vatMatch) data.vat = parseFloat(vatMatch[1].replace(',', ''));

    return data;
  }

  // ดึงข้อมูล Resume
  extractResumeData(text) {
    const data = {};

    // ชื่อ (มักอยู่บรรทัดแรก)
    const lines = text.split('\n').filter((l) => l.trim());
    if (lines.length > 0) data.name = lines[0].trim();

    // Email
    const emailMatch = text.match(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/);
    if (emailMatch) data.email = emailMatch[0];

    // โทรศัพท์
    const phoneMatch = text.match(/0[0-9]{1,2}[- ]?[0-9]{3,4}[- ]?[0-9]{4}/);
    if (phoneMatch) data.phone = phoneMatch[0];

    return data;
  }

  // ดึงข้อมูลทั่วไป
  extractGeneralData(text) {
    const data = {
      wordCount: text.split(/\s+/).length,
      lineCount: text.split('\n').length,
      hasNumbers: /\d/.test(text),
      hasThaiText: /[฀-๿]/.test(text),
      hasEnglishText: /[a-zA-Z]/.test(text),
    };

    // ค้นหา URLs
    const urls = text.match(/https?:\/\/[^\s]+/g);
    if (urls) data.urls = urls;

    // ค้นหา Emails
    const emails = text.match(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g);
    if (emails) data.emails = emails;

    return data;
  }

  // Format ผลลัพธ์สำหรับ LINE
  formatForLINE(scanResult) {
    const { text, documentType, extractedData, confidence } = scanResult;

    const typeNames = {
      thai_id_card: '🪪 บัตรประชาชน',
      tax_invoice: '🧾 ใบกำกับภาษี',
      birth_certificate: '📜 ใบสูติบัตร',
      contract: '📄 สัญญา',
      medical_document: '🏥 เอกสารทางการแพทย์',
      resume: '📋 ประวัติย่อ',
      general_document: '📃 เอกสารทั่วไป',
    };

    let response = `${typeNames[documentType] || '📃 เอกสาร'}\n`;
    response += `━━━━━━━━━━━━━━━━\n`;

    if (Object.keys(extractedData).length > 0) {
      response += '📌 ข้อมูลที่พบ:\n';

      for (const [key, value] of Object.entries(extractedData)) {
        const keyNames = {
          name: 'ชื่อ',
          idNumber: 'เลขบัตร',
          birthDate: 'วันเกิด',
          address: 'ที่อยู่',
          invoiceNo: 'เลขที่',
          date: 'วันที่',
          total: 'ยอดรวม',
          vat: 'ภาษี VAT',
          email: 'อีเมล',
          phone: 'โทรศัพท์',
        };

        if (value) {
          response += `• ${keyNames[key] || key}: ${value}\n`;
        }
      }
    } else if (text) {
      response += `📝 ข้อความในเอกสาร:\n${text.substring(0, 300)}`;
      if (text.length > 300) response += '...';
    } else {
      response += 'ไม่พบข้อความในเอกสาร';
    }

    return response;
  }
}

module.exports = new DocumentScannerBot();
```

---

## 7. QR/Barcode Scanner

```javascript
// src/bots/qrScannerBot.js
const jsQR = require('jsqr');
const sharp = require('sharp');
const logger = require('../utils/logger');

class QRScannerBot {
  // สแกน QR Code
  async scanQR(imageBuffer) {
    try {
      // แปลงรูปเป็น Raw RGBA
      const { data, info } = await sharp(imageBuffer)
        .ensureAlpha()
        .raw()
        .toBuffer({ resolveWithObject: true });

      // Scan QR Code
      const code = jsQR(
        new Uint8ClampedArray(data),
        info.width,
        info.height,
        { inversionAttempts: 'dontInvert' }
      );

      if (!code) {
        // ลองด้วย inverted
        const invertedCode = jsQR(
          new Uint8ClampedArray(data),
          info.width,
          info.height,
          { inversionAttempts: 'attemptBoth' }
        );

        if (!invertedCode) {
          return { found: false, data: null };
        }

        return this.parseQRData(invertedCode.data);
      }

      return this.parseQRData(code.data);
    } catch (error) {
      logger.error('QR scan error', { error: error.message });
      throw error;
    }
  }

  // แยกประเภท QR Data
  parseQRData(rawData) {
    logger.info('QR data found', { dataLength: rawData.length });

    const result = { found: true, raw: rawData };

    // URL
    if (/^https?:\/\//i.test(rawData)) {
      result.type = 'url';
      result.url = rawData;

      // ตรวจว่าเป็น LINE URL
      if (rawData.includes('line.me') || rawData.includes('lin.ee')) {
        result.type = 'line_qr';
        result.lineUrl = rawData;

        // แยก LINE ID
        const lineIdMatch = rawData.match(/line\.me\/ti\/p\/(.+)/);
        if (lineIdMatch) result.lineId = lineIdMatch[1];
      }

      // PromptPay URL
      if (rawData.includes('promptpay') || rawData.startsWith('00020101')) {
        result.type = 'promptpay';
        result.promptpayData = this.parsePromptPay(rawData);
      }

      return result;
    }

    // PromptPay QR (EMVCo format)
    if (rawData.startsWith('00020101') || rawData.startsWith('000201')) {
      result.type = 'promptpay';
      result.promptpayData = this.parsePromptPay(rawData);
      return result;
    }

    // vCard
    if (rawData.startsWith('BEGIN:VCARD')) {
      result.type = 'vcard';
      result.contact = this.parseVCard(rawData);
      return result;
    }

    // WiFi
    if (rawData.startsWith('WIFI:')) {
      result.type = 'wifi';
      result.wifi = this.parseWiFi(rawData);
      return result;
    }

    // Email
    if (rawData.startsWith('mailto:')) {
      result.type = 'email';
      result.email = rawData.replace('mailto:', '');
      return result;
    }

    // Phone
    if (rawData.startsWith('tel:')) {
      result.type = 'phone';
      result.phone = rawData.replace('tel:', '');
      return result;
    }

    // Plain text
    result.type = 'text';
    result.text = rawData;

    return result;
  }

  // Parse PromptPay QR
  parsePromptPay(data) {
    try {
      // EMVCo QR Code parsing (simplified)
      const result = { raw: data };

      // ค้นหาหมายเลขโทรศัพท์หรือ Tax ID
      const phoneMatch = data.match(/(?:0066|66|0)[0-9]{9}/);
      if (phoneMatch) {
        result.phone = phoneMatch[0].replace(/^(?:0066|66)/, '0');
        result.type = 'phone';
      }

      // ค้นหา Tax ID (13 หลัก)
      const taxIdMatch = data.match(/\d{13}/);
      if (taxIdMatch) {
        result.taxId = taxIdMatch[0];
        result.type = 'tax_id';
      }

      // ค้นหาจำนวนเงิน
      const amountTag = data.match(/54(\d{2})(\d+(?:\.\d{2})?)/);
      if (amountTag) {
        result.amount = parseFloat(amountTag[2]);
      }

      return result;
    } catch (error) {
      return { raw: data, error: 'Parse error' };
    }
  }

  // Parse vCard
  parseVCard(data) {
    const contact = {};
    const lines = data.split('\n');

    for (const line of lines) {
      const [key, ...valueParts] = line.split(':');
      const value = valueParts.join(':').trim();

      if (key.startsWith('FN')) contact.name = value;
      else if (key.startsWith('TEL')) contact.phone = value;
      else if (key.startsWith('EMAIL')) contact.email = value;
      else if (key.startsWith('ORG')) contact.organization = value;
      else if (key.startsWith('URL')) contact.url = value;
    }

    return contact;
  }

  // Parse WiFi QR
  parseWiFi(data) {
    const wifi = {};
    const match = data.match(/WIFI:T:([^;]*);S:([^;]*);P:([^;]*)/);

    if (match) {
      wifi.security = match[1];
      wifi.ssid = match[2];
      wifi.password = match[3];
    }

    return wifi;
  }

  // Format ผลลัพธ์สำหรับ LINE
  formatForLINE(qrResult) {
    if (!qrResult.found) {
      return '❌ ไม่พบ QR Code ในรูปภาพ\n\nกรุณาตรวจสอบ:\n• รูปภาพชัดเจน\n• QR Code ไม่เบลอ\n• มีแสงสว่างเพียงพอ';
    }

    const { type } = qrResult;

    switch (type) {
      case 'url':
        return `🔗 พบ URL:\n${qrResult.url}\n\nคลิกเพื่อเปิด`;

      case 'line_qr':
        return `📱 LINE QR Code:\n${qrResult.lineUrl}\n${qrResult.lineId ? `\nLINE ID: ${qrResult.lineId}` : ''}`;

      case 'promptpay':
        const pp = qrResult.promptpayData;
        let ppText = '💳 PromptPay QR Code\n';
        if (pp.phone) ppText += `📞 เบอร์: ${pp.phone}\n`;
        if (pp.taxId) ppText += `🏢 Tax ID: ${pp.taxId}\n`;
        if (pp.amount) ppText += `💰 จำนวน: ${pp.amount.toLocaleString()} บาท\n`;
        return ppText;

      case 'vcard':
        const contact = qrResult.contact;
        return `👤 ข้อมูลติดต่อ:\n` +
          `${contact.name ? `ชื่อ: ${contact.name}\n` : ''}` +
          `${contact.phone ? `โทร: ${contact.phone}\n` : ''}` +
          `${contact.email ? `Email: ${contact.email}\n` : ''}` +
          `${contact.organization ? `บริษัท: ${contact.organization}` : ''}`;

      case 'wifi':
        return `📶 WiFi QR Code:\n` +
          `ชื่อเครือข่าย: ${qrResult.wifi.ssid}\n` +
          `รหัสผ่าน: ${qrResult.wifi.password}\n` +
          `ความปลอดภัย: ${qrResult.wifi.security}`;

      case 'phone':
        return `📞 เบอร์โทรศัพท์:\n${qrResult.phone}`;

      case 'email':
        return `📧 อีเมล:\n${qrResult.email}`;

      default:
        return `📄 ข้อมูล QR Code:\n${qrResult.text || qrResult.raw}`;
    }
  }
}

module.exports = new QRScannerBot();
```

---

## 8. Main Image Handler

```javascript
// src/handlers/imageHandler.js
const line = require('@line/bot-sdk');
const imageDownloader = require('../services/imageDownloader');
const googleVision = require('../services/googleVision');
const foodRecognitionBot = require('../bots/foodRecognitionBot');
const documentScannerBot = require('../bots/documentScannerBot');
const qrScannerBot = require('../bots/qrScannerBot');
const logger = require('../utils/logger');

class ImageHandler {
  constructor() {
    this.lineClient = new line.messagingApi.MessagingApiClient({
      channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    });

    // โหมดการวิเคราะห์ต่างๆ
    this.modes = {
      food: '🍽️ วิเคราะห์อาหาร',
      document: '📄 สแกนเอกสาร',
      qr: '📷 สแกน QR Code',
      face: '👤 ตรวจจับใบหน้า',
      auto: '🤖 อัตโนมัติ',
    };
  }

  async handleImageMessage(event, userId) {
    const messageId = event.message.id;
    const replyToken = event.replyToken;

    // แสดง Loading
    await this.lineClient.showLoadingAnimation({ chatId: userId, loadingSeconds: 15 });

    let imageData;
    try {
      // ดาวน์โหลดรูปภาพ
      imageData = await imageDownloader.downloadFromLINE(
        messageId,
        process.env.LINE_CHANNEL_ACCESS_TOKEN
      );

      // ตรวจสอบ Safe Search ก่อน
      const safeSearch = await googleVision.checkSafeSearch(imageData.buffer);
      if (safeSearch.isUnsafe) {
        await this.lineClient.replyMessage({
          replyToken,
          messages: [{ type: 'text', text: '❌ ไม่สามารถวิเคราะห์รูปภาพนี้ได้' }],
        });
        return;
      }

      // วิเคราะห์แบบ Auto (ตรวจสอบทุกอย่าง)
      const analysis = await this.autoAnalyze(imageData.buffer);

      // สร้าง Response
      const messages = await this.buildResponse(analysis, userId);

      await this.lineClient.replyMessage({ replyToken, messages });
    } catch (error) {
      logger.error('Image handler error', { error: error.message, userId });
      await this.lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: 'ขออภัยครับ ไม่สามารถวิเคราะห์รูปภาพได้' }],
      });
    } finally {
      // ลบไฟล์ชั่วคราว
      if (imageData?.filepath) {
        await imageDownloader.cleanup(imageData.filepath);
      }
    }
  }

  // วิเคราะห์อัตโนมัติ
  async autoAnalyze(imageBuffer) {
    const results = {};

    // ตรวจสอบ QR Code ก่อน (เร็วที่สุด)
    results.qr = await qrScannerBot.scanQR(imageBuffer);

    if (results.qr.found) {
      return { type: 'qr', data: results.qr };
    }

    // วิเคราะห์ Labels พร้อมกัน
    const [labels, text] = await Promise.all([
      googleVision.detectLabels(imageBuffer),
      googleVision.detectText(imageBuffer),
    ]);

    // ตรวจสอบว่าเป็นเอกสารหรือไม่
    const hasText = text.fullText && text.fullText.length > 20;
    const isDocument = labels.some((l) =>
      ['Document', 'Text', 'Paper', 'Receipt', 'Invoice'].includes(l.label)
    );

    if (isDocument && hasText) {
      results.document = await documentScannerBot.scanDocument(imageBuffer);
      return { type: 'document', data: results.document, labels };
    }

    // ตรวจสอบว่าเป็นอาหารหรือไม่
    const isFood = labels.some((l) =>
      ['Food', 'Cuisine', 'Dish', 'Meal'].includes(l.label)
    );

    if (isFood) {
      results.food = await foodRecognitionBot.analyzeFood(imageBuffer, 'auto');
      return { type: 'food', data: results.food, labels };
    }

    // General analysis
    return {
      type: 'general',
      labels,
      text: text.fullText,
      hasText,
    };
  }

  // สร้าง Response สำหรับ LINE
  async buildResponse(analysis, userId) {
    const messages = [];

    switch (analysis.type) {
      case 'qr': {
        const qrText = qrScannerBot.formatForLINE(analysis.data);
        messages.push({ type: 'text', text: qrText });

        // ถ้าเป็น URL เพิ่ม Button
        if (analysis.data.url) {
          messages.push({
            type: 'template',
            altText: 'เปิด URL',
            template: {
              type: 'buttons',
              text: 'ต้องการเปิด URL หรือไม่?',
              actions: [{
                type: 'uri',
                label: 'เปิด URL',
                uri: analysis.data.url,
              }],
            },
          });
        }
        break;
      }

      case 'document': {
        const docText = documentScannerBot.formatForLINE(analysis.data);
        messages.push({ type: 'text', text: docText });
        break;
      }

      case 'food': {
        if (analysis.data.safe && analysis.data.detected.length > 0) {
          const flexMsg = foodRecognitionBot.buildFoodFlexMessage(analysis.data, userId);
          if (flexMsg) {
            messages.push(flexMsg);
          } else {
            messages.push({
              type: 'text',
              text: analysis.data.analysis.message,
            });
          }
        } else {
          messages.push({
            type: 'text',
            text: analysis.data.analysis?.message || 'ไม่สามารถวิเคราะห์รูปอาหารได้',
          });
        }
        break;
      }

      default: {
        // General labels
        if (analysis.labels?.length > 0) {
          const topLabels = analysis.labels.slice(0, 5);
          const labelText = topLabels.map((l) =>
            `• ${l.label} (${(l.confidence * 100).toFixed(0)}%)`
          ).join('\n');

          messages.push({
            type: 'text',
            text: `🔍 ผลการวิเคราะห์รูปภาพ:\n\n${labelText}`,
          });
        } else if (analysis.text) {
          messages.push({
            type: 'text',
            text: `📝 ข้อความในรูปภาพ:\n\n${analysis.text.substring(0, 500)}`,
          });
        } else {
          messages.push({
            type: 'text',
            text: '❓ ไม่สามารถระบุสิ่งที่อยู่ในรูปภาพได้ชัดเจน',
          });
        }

        // Quick Reply สำหรับวิเคราะห์เพิ่มเติม
        if (messages.length > 0) {
          const lastMsg = messages[messages.length - 1];
          if (lastMsg.type === 'text') {
            lastMsg.quickReply = {
              items: [
                {
                  type: 'action',
                  action: { type: 'message', label: '🍽️ วิเคราะห์อาหาร', text: 'วิเคราะห์อาหาร' },
                },
                {
                  type: 'action',
                  action: { type: 'message', label: '📄 สแกนเอกสาร', text: 'สแกนเอกสาร' },
                },
                {
                  type: 'action',
                  action: { type: 'message', label: '📷 สแกน QR', text: 'สแกน QR Code' },
                },
              ],
            };
          }
        }
      }
    }

    return messages.slice(0, 5); // สูงสุด 5 messages
  }
}

module.exports = new ImageHandler();
```

---

## 9. Product Identification Bot

```javascript
// src/bots/productIdentifierBot.js
const googleVision = require('../services/googleVision');
const awsRekognition = require('../services/awsRekognition');

class ProductIdentifierBot {
  constructor() {
    // ฐานข้อมูลสินค้า (ในระบบจริงดึงจาก Database)
    this.productCatalog = [
      {
        id: 'PROD001',
        name: 'เสื้อยืด Classic White',
        category: 'เสื้อผ้า',
        price: 299,
        colors: ['white', 'cream'],
        tags: ['t-shirt', 'clothing', 'white'],
        imageUrl: 'https://example.com/prod001.jpg',
      },
      {
        id: 'PROD002',
        name: 'กระเป๋าหนัง Business',
        category: 'กระเป๋า',
        price: 1590,
        colors: ['brown', 'black'],
        tags: ['bag', 'leather', 'handbag'],
        imageUrl: 'https://example.com/prod002.jpg',
      },
    ];
  }

  // ค้นหาสินค้าจากรูปภาพ
  async identifyProduct(imageBuffer) {
    // ตรวจจับ Labels, Objects และ Logos พร้อมกัน
    const [labels, objects, logos, webDetection] = await Promise.all([
      googleVision.detectLabels(imageBuffer),
      googleVision.detectObjects(imageBuffer),
      googleVision.detectLogos(imageBuffer),
      googleVision.detectWeb(imageBuffer),
    ]);

    // ค้นหาสินค้าใน Catalog
    const matchedProducts = this.matchProducts(labels, objects, logos);

    // ดึง Web results สำหรับสินค้าที่ไม่พบใน catalog
    const webSuggestions = webDetection.bestGuessLabels.slice(0, 3);

    return {
      products: matchedProducts,
      detectedObjects: objects.slice(0, 5),
      logos: logos.slice(0, 3),
      webSuggestions,
      hasProducts: matchedProducts.length > 0,
    };
  }

  matchProducts(labels, objects, logos) {
    const allTerms = [
      ...labels.map((l) => l.label.toLowerCase()),
      ...objects.map((o) => o.name.toLowerCase()),
      ...logos.map((l) => l.description.toLowerCase()),
    ];

    return this.productCatalog.filter((product) =>
      product.tags.some((tag) => allTerms.some((term) => term.includes(tag)))
    );
  }

  buildFlexMessage(result) {
    if (!result.hasProducts) {
      return null;
    }

    const product = result.products[0];
    return {
      type: 'flex',
      altText: `สินค้าที่พบ: ${product.name}`,
      contents: {
        type: 'bubble',
        hero: {
          type: 'image',
          url: product.imageUrl,
          size: 'full',
          aspectRatio: '4:3',
          aspectMode: 'cover',
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: product.name, weight: 'bold', size: 'lg' },
            { type: 'text', text: product.category, color: '#888888' },
            {
              type: 'text',
              text: `฿${product.price.toLocaleString()}`,
              color: '#FF6B35',
              size: 'xl',
              weight: 'bold',
              margin: 'md',
            },
          ],
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'button',
            style: 'primary',
            action: {
              type: 'message',
              label: 'ต้องการสินค้านี้',
              text: `ต้องการสั่งซื้อ ${product.name}`,
            },
            color: '#FF6B35',
          }],
        },
      },
    };
  }
}

module.exports = new ProductIdentifierBot();
```

---

## 10. Main Application

```javascript
// src/index.js
const express = require('express');
const line = require('@line/bot-sdk');
const imageHandler = require('./handlers/imageHandler');
const logger = require('./utils/logger');

const app = express();

app.post('/webhook', line.middleware({
  channelSecret: process.env.LINE_CHANNEL_SECRET,
}), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(handleEvent));
    res.sendStatus(200);
  } catch (error) {
    logger.error('Webhook error', { error: error.message });
    res.sendStatus(500);
  }
});

async function handleEvent(event) {
  const userId = event.source.userId;

  if (event.type === 'message') {
    if (event.message.type === 'image') {
      return imageHandler.handleImageMessage(event, userId);
    }
    if (event.message.type === 'text') {
      // Handle text messages...
    }
  }
}

app.listen(process.env.PORT || 3000, () => {
  logger.info('Server started');
  imageDownloader.initialize().catch(logger.error);
});
```

---

## 11. Error Handling และ Monitoring

```javascript
// src/middleware/imageValidation.js

class ImageValidationMiddleware {
  static async validate(imageBuffer) {
    const errors = [];

    // ตรวจขนาดไฟล์
    if (imageBuffer.length > 10 * 1024 * 1024) {
      errors.push('รูปภาพใหญ่เกินไป (สูงสุด 10MB)');
    }

    // ตรวจขนาดรูป
    const metadata = await require('sharp')(imageBuffer).metadata();
    if (metadata.width < 10 || metadata.height < 10) {
      errors.push('รูปภาพเล็กเกินไป');
    }

    if (metadata.width > 10000 || metadata.height > 10000) {
      errors.push('รูปภาพใหญ่เกินไป');
    }

    return {
      valid: errors.length === 0,
      errors,
      metadata,
    };
  }
}

module.exports = ImageValidationMiddleware;
```

---

## สรุปบทที่ 64

ในบทนี้เราได้เรียนรู้:

1. **Image Download** - การดาวน์โหลดและ Process รูปภาพจาก LINE API
2. **Google Vision API** - Labels, OCR, Face, Logo, Safe Search, Web Detection
3. **AWS Rekognition** - Labels, Text, Faces, Moderation, Celebrities
4. **Food Recognition** - วิเคราะห์อาหารพร้อมข้อมูลโภชนาการ
5. **Document Scanner** - OCR และ Extract ข้อมูลจากเอกสาร
6. **QR/Barcode Scanner** - Decode QR Types ต่างๆ รวม PromptPay
7. **Product Identifier** - จดจำสินค้าจากรูปภาพ
8. **Auto Analysis** - วิเคราะห์อัตโนมัติตามเนื้อหารูป
9. **Safety Check** - ตรวจสอบความเหมาะสมก่อนวิเคราะห์

---

*ต่อไป: Part 65 - Customer Service Bot*
