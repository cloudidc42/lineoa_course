# Part 56: LINE OA Insight API - การวิเคราะห์ข้อมูลเชิงลึก

## บทนำ

LINE OA Insight API เป็นเครื่องมือที่ช่วยให้ผู้พัฒนาสามารถดึงข้อมูลสถิติและการวิเคราะห์จาก LINE Official Account ได้โดยตรงผ่าน API ข้อมูลเหล่านี้ครอบคลุมทั้งสถิติการส่งข้อความ จำนวนผู้ติดตาม และข้อมูลประชากรศาสตร์ของกลุ่มเป้าหมาย

### ประโยชน์ของ Insight API

- วิเคราะห์ประสิทธิภาพของแคมเปญการตลาด
- เข้าใจพฤติกรรมและลักษณะของผู้ติดตาม
- ปรับปรุงเนื้อหาให้เหมาะสมกับกลุ่มเป้าหมาย
- ติดตามการเติบโตของบัญชี
- สร้าง Dashboard สำหรับทีมการตลาด

---

## 1. การตั้งค่าเบื้องต้น

### 1.1 สิทธิ์การเข้าถึง

Insight API ต้องการ Channel Access Token และ Channel ID ของ LINE Official Account

```javascript
// config/line-insight.js
const lineInsightConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelId: process.env.LINE_CHANNEL_ID,
  baseUrl: 'https://api.line.me/v2/bot',
  insightBaseUrl: 'https://api.line.me/v2/insight',
  dataInsightUrl: 'https://api.line.me/v2/bot/insight'
};

module.exports = lineInsightConfig;
```

### 1.2 ติดตั้ง Dependencies

```bash
npm init -y
npm install axios dotenv moment chart.js sqlite3 express
npm install --save-dev jest nodemon
```

### 1.3 โครงสร้างโปรเจกต์

```
line-insight-dashboard/
├── config/
│   └── line-insight.js
├── services/
│   ├── insight-api.js
│   ├── demographic-service.js
│   └── statistics-service.js
├── database/
│   ├── schema.sql
│   └── db.js
├── routes/
│   └── dashboard.js
├── public/
│   ├── index.html
│   └── charts.js
├── .env
└── server.js
```

---

## 2. LINE OA Insight API Overview

### 2.1 API Endpoints หลัก

```
GET /v2/insight/message/delivery          - สถิติการส่งข้อความ
GET /v2/insight/followers                 - สถิติผู้ติดตาม
GET /v2/insight/demographic               - ข้อมูลประชากรศาสตร์
GET /v2/bot/insight/message/event         - สถิติเหตุการณ์ข้อความ
GET /v2/bot/insight/message/event/aggregate - สรุปสถิติเหตุการณ์
```

### 2.2 HTTP Headers ที่ต้องการ

```javascript
const getHeaders = () => ({
  'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json'
});
```

---

## 3. Message Delivery Statistics (สถิติการส่งข้อความ)

### 3.1 ดึงสถิติการส่งข้อความรายวัน

```javascript
// services/insight-api.js
const axios = require('axios');
const config = require('../config/line-insight');

class InsightApiService {
  constructor() {
    this.client = axios.create({
      baseURL: 'https://api.line.me',
      headers: {
        'Authorization': `Bearer ${config.channelAccessToken}`,
        'Content-Type': 'application/json'
      }
    });
  }

  /**
   * ดึงสถิติการส่งข้อความรายวัน
   * @param {string} date - วันที่ในรูปแบบ YYYYMMDD
   * @returns {Promise<Object>} สถิติการส่งข้อความ
   */
  async getMessageDeliveryStats(date) {
    try {
      const response = await this.client.get('/v2/insight/message/delivery', {
        params: { date }
      });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * ดึงสถิติการส่งข้อความในช่วงเวลา
   * @param {string} startDate - วันที่เริ่มต้น YYYYMMDD
   * @param {string} endDate - วันที่สิ้นสุด YYYYMMDD
   * @returns {Promise<Array>} อาร์เรย์ของสถิติรายวัน
   */
  async getMessageDeliveryRange(startDate, endDate) {
    const dates = this.generateDateRange(startDate, endDate);
    const results = [];

    for (const date of dates) {
      try {
        const stats = await this.getMessageDeliveryStats(date);
        results.push({ date, ...stats });
        // หน่วงเวลาเพื่อป้องกัน rate limit
        await this.delay(200);
      } catch (error) {
        console.error(`ไม่สามารถดึงข้อมูลวันที่ ${date}: ${error.message}`);
        results.push({ date, error: error.message });
      }
    }

    return results;
  }

  /**
   * สร้างรายการวันที่ในช่วงเวลา
   */
  generateDateRange(startDate, endDate) {
    const dates = [];
    const start = new Date(
      startDate.slice(0, 4),
      parseInt(startDate.slice(4, 6)) - 1,
      startDate.slice(6, 8)
    );
    const end = new Date(
      endDate.slice(0, 4),
      parseInt(endDate.slice(4, 6)) - 1,
      endDate.slice(6, 8)
    );

    const current = new Date(start);
    while (current <= end) {
      const year = current.getFullYear();
      const month = String(current.getMonth() + 1).padStart(2, '0');
      const day = String(current.getDate()).padStart(2, '0');
      dates.push(`${year}${month}${day}`);
      current.setDate(current.getDate() + 1);
    }

    return dates;
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  handleError(error) {
    if (error.response) {
      const { status, data } = error.response;
      throw new Error(`LINE API Error ${status}: ${JSON.stringify(data)}`);
    }
    throw error;
  }
}

module.exports = InsightApiService;
```

### 3.2 ตัวอย่าง Response จาก Delivery Stats

```json
{
  "overview": {
    "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
    "timestamp": 1568214000,
    "success": 1234,
    "failure": 0
  },
  "messages": [
    {
      "seq": 1,
      "impression": 800,
      "mediaPlayed": null,
      "mediaPlayed25Percent": null,
      "mediaPlayed50Percent": null,
      "mediaPlayed75Percent": null,
      "mediaPlayed100Percent": null,
      "uniqueMediaPlayed": null,
      "uniqueMediaPlayed25Percent": null,
      "uniqueMediaPlayed50Percent": null,
      "uniqueMediaPlayed75Percent": null,
      "uniqueMediaPlayed100Percent": null
    }
  ],
  "clicks": [
    {
      "seq": 1,
      "url": "https://example.com",
      "click": 150,
      "uniqueClick": 120,
      "uniqueClickOfRequest": null,
      "uniqueClickOfImpression": null
    }
  ]
}
```

---

## 4. Follower Statistics (สถิติผู้ติดตาม)

### 4.1 ดึงข้อมูลผู้ติดตาม

```javascript
// services/follower-service.js
class FollowerService extends InsightApiService {
  /**
   * ดึงสถิติผู้ติดตามรายวัน
   * @param {string} date - YYYYMMDD
   */
  async getFollowerStats(date) {
    try {
      const response = await this.client.get('/v2/insight/followers', {
        params: { date }
      });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * ดึงจำนวนผู้ติดตามปัจจุบัน
   */
  async getCurrentFollowers() {
    try {
      const response = await this.client.get('/v2/bot/info');
      return {
        followersCount: response.data.followersCount,
        chatStatistics: response.data.chatStatistics
      };
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * วิเคราะห์การเติบโตของผู้ติดตาม
   * @param {string} startDate
   * @param {string} endDate
   */
  async analyzeFollowerGrowth(startDate, endDate) {
    const dates = this.generateDateRange(startDate, endDate);
    const followerData = [];

    for (const date of dates) {
      const stats = await this.getFollowerStats(date);
      followerData.push({
        date,
        followers: stats.followers,
        targetedReaches: stats.targetedReaches,
        blocks: stats.blocks
      });
      await this.delay(200);
    }

    // คำนวณ Net Growth
    const growthAnalysis = followerData.map((day, index) => {
      const prevDay = index > 0 ? followerData[index - 1] : null;
      return {
        ...day,
        newFollowers: prevDay ? day.followers - prevDay.followers : 0,
        netGrowth: prevDay ? (day.followers - prevDay.followers) - (day.blocks || 0) : 0
      };
    });

    return {
      data: growthAnalysis,
      summary: this.calculateGrowthSummary(growthAnalysis)
    };
  }

  calculateGrowthSummary(data) {
    const totalNew = data.reduce((sum, d) => sum + (d.newFollowers || 0), 0);
    const totalBlocks = data.reduce((sum, d) => sum + (d.blocks || 0), 0);
    const avgDailyGrowth = totalNew / data.length;

    return {
      totalNewFollowers: totalNew,
      totalBlocks,
      netGrowth: totalNew - totalBlocks,
      averageDailyGrowth: Math.round(avgDailyGrowth * 100) / 100,
      peakGrowthDay: data.reduce((max, d) =>
        (d.newFollowers || 0) > (max.newFollowers || 0) ? d : max, data[0])
    };
  }
}
```

### 4.2 ตัวอย่าง Follower Stats Response

```json
{
  "status": "ready",
  "followers": 7620,
  "targetedReaches": 5489,
  "blocks": 1213
}
```

---

## 5. Demographic Data API (ข้อมูลประชากรศาสตร์)

### 5.1 ดึงข้อมูลประชากรศาสตร์

```javascript
// services/demographic-service.js
class DemographicService extends InsightApiService {
  /**
   * ดึงข้อมูลประชากรศาสตร์ของผู้ติดตาม
   * @param {string} date - YYYYMMDD
   */
  async getDemographics(date) {
    try {
      const response = await this.client.get('/v2/insight/demographic', {
        params: { date }
      });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * แยกข้อมูลตามกลุ่มอายุ
   */
  parseAgeGroups(demographicData) {
    const ageAttributes = demographicData.genders || [];
    return ageAttributes
      .filter(attr => attr.type === 'age')
      .map(attr => ({
        ageGroup: attr.ageGroup,
        count: attr.count,
        percentage: attr.percentage
      }))
      .sort((a, b) => a.ageGroup.localeCompare(b.ageGroup));
  }

  /**
   * แยกข้อมูลตามเพศ
   */
  parseGenderData(demographicData) {
    return (demographicData.genders || []).map(gender => ({
      gender: gender.gender === 'male' ? 'ชาย' : 'หญิง',
      percentage: gender.percentage
    }));
  }

  /**
   * แยกข้อมูลตามระบบปฏิบัติการ
   */
  parseOSData(demographicData) {
    return (demographicData.os || []).map(os => ({
      os: os.os,
      percentage: os.percentage
    }));
  }

  /**
   * แยกข้อมูลตามภูมิภาค
   */
  parseRegionData(demographicData) {
    return (demographicData.regions || [])
      .map(region => ({
        region: this.translateRegion(region.region),
        percentage: region.percentage
      }))
      .sort((a, b) => b.percentage - a.percentage);
  }

  /**
   * แปลชื่อภูมิภาคเป็นภาษาไทย
   */
  translateRegion(region) {
    const translations = {
      'TH-10': 'กรุงเทพมหานคร',
      'TH-11': 'สมุทรปราการ',
      'TH-12': 'นนทบุรี',
      'TH-13': 'ปทุมธานี',
      'TH-14': 'พระนครศรีอยุธยา',
      'TH-15': 'อ่างทอง',
      'TH-16': 'ลพบุรี',
      'TH-17': 'สิงห์บุรี',
      'TH-18': 'ชัยนาท',
      'TH-19': 'สระบุรี',
      'TH-20': 'ชลบุรี',
      'TH-21': 'ระยอง',
      'TH-22': 'จันทบุรี',
      'TH-23': 'ตราด',
      'TH-24': 'ฉะเชิงเทรา',
      'TH-25': 'ปราจีนบุรี',
      'TH-26': 'นครนายก',
      'TH-27': 'สระแก้ว',
      'TH-30': 'นครราชสีมา',
      'TH-31': 'บุรีรัมย์',
      'TH-32': 'สุรินทร์',
      'TH-33': 'ศรีสะเกษ',
      'TH-34': 'อุบลราชธานี',
      'TH-35': 'ยโสธร',
      'TH-36': 'ชัยภูมิ',
      'TH-37': 'อำนาจเจริญ',
      'TH-40': 'ขอนแก่น',
      'TH-41': 'อุดรธานี',
      'TH-42': 'เลย',
      'TH-43': 'หนองคาย',
      'TH-44': 'มหาสารคาม',
      'TH-45': 'ร้อยเอ็ด',
      'TH-46': 'กาฬสินธุ์',
      'TH-47': 'สกลนคร',
      'TH-48': 'นครพนม',
      'TH-49': 'มุกดาหาร',
      'TH-50': 'เชียงใหม่',
      'TH-51': 'ลำพูน',
      'TH-52': 'ลำปาง',
      'TH-53': 'อุตรดิตถ์',
      'TH-54': 'แพร่',
      'TH-55': 'น่าน',
      'TH-56': 'พะเยา',
      'TH-57': 'เชียงราย',
      'TH-58': 'แม่ฮ่องสอน',
      'TH-60': 'นครสวรรค์',
      'TH-61': 'อุทัยธานี',
      'TH-62': 'กำแพงเพชร',
      'TH-63': 'ตาก',
      'TH-64': 'สุโขทัย',
      'TH-65': 'พิษณุโลก',
      'TH-66': 'พิจิตร',
      'TH-67': 'เพชรบูรณ์',
      'TH-70': 'ราชบุรี',
      'TH-71': 'กาญจนบุรี',
      'TH-72': 'สุพรรณบุรี',
      'TH-73': 'นครปฐม',
      'TH-74': 'สมุทรสาคร',
      'TH-75': 'สมุทรสงคราม',
      'TH-76': 'เพชรบุรี',
      'TH-77': 'ประจวบคีรีขันธ์',
      'TH-80': 'นครศรีธรรมราช',
      'TH-81': 'กระบี่',
      'TH-82': 'พังงา',
      'TH-83': 'ภูเก็ต',
      'TH-84': 'สุราษฎร์ธานี',
      'TH-85': 'ระนอง',
      'TH-86': 'ชุมพร',
      'TH-90': 'สงขลา',
      'TH-91': 'สตูล',
      'TH-92': 'ตรัง',
      'TH-93': 'พัทลุง',
      'TH-94': 'ปัตตานี',
      'TH-95': 'ยะลา',
      'TH-96': 'นราธิวาส'
    };
    return translations[region] || region;
  }

  /**
   * แยกข้อมูลตามระยะเวลาการ subscribe
   */
  parseSubscriptionPeriod(demographicData) {
    return (demographicData.subscriptionPeriods || []).map(period => ({
      period: this.translateSubscriptionPeriod(period.subscriptionPeriod),
      percentage: period.percentage
    }));
  }

  translateSubscriptionPeriod(period) {
    const translations = {
      'DAY_7': 'ภายใน 7 วัน',
      'DAY_30': '7-30 วัน',
      'DAY_90': '30-90 วัน',
      'DAY_180': '90-180 วัน',
      'DAY_365': '180-365 วัน',
      'DAY_INFINITE': 'มากกว่า 365 วัน'
    };
    return translations[period] || period;
  }
}

module.exports = DemographicService;
```

### 5.2 ตัวอย่าง Demographics Response

```json
{
  "status": "ready",
  "genders": [
    { "gender": "male", "percentage": 35.2 },
    { "gender": "female", "percentage": 64.8 }
  ],
  "ages": [
    { "age": "AGE_UNDER18", "percentage": 5.1 },
    { "age": "AGE_18TO24", "percentage": 18.3 },
    { "age": "AGE_25TO34", "percentage": 31.7 },
    { "age": "AGE_35TO44", "percentage": 24.9 },
    { "age": "AGE_45TO54", "percentage": 13.2 },
    { "age": "AGE_OVER55", "percentage": 6.8 }
  ],
  "os": [
    { "os": "ios", "percentage": 42.5 },
    { "os": "android", "percentage": 57.5 }
  ],
  "regions": [
    { "region": "TH-10", "percentage": 28.3 },
    { "region": "TH-20", "percentage": 8.7 },
    { "region": "TH-50", "percentage": 7.2 }
  ],
  "subscriptionPeriods": [
    { "subscriptionPeriod": "DAY_7", "percentage": 3.2 },
    { "subscriptionPeriod": "DAY_30", "percentage": 8.5 },
    { "subscriptionPeriod": "DAY_90", "percentage": 15.7 },
    { "subscriptionPeriod": "DAY_180", "percentage": 22.1 },
    { "subscriptionPeriod": "DAY_365", "percentage": 28.3 },
    { "subscriptionPeriod": "DAY_INFINITE", "percentage": 22.2 }
  ]
}
```

---

## 6. Message Statistics Per Broadcast (สถิติข้อความต่อการ Broadcast)

### 6.1 ดึงสถิติเหตุการณ์ข้อความ

```javascript
// services/broadcast-stats-service.js
class BroadcastStatsService extends InsightApiService {
  /**
   * ดึงสถิติเหตุการณ์ของข้อความที่ส่ง
   * @param {string} requestId - Request ID ของการ Broadcast
   */
  async getMessageEventStats(requestId) {
    try {
      const response = await this.client.get('/v2/bot/insight/message/event', {
        params: { requestId }
      });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * ดึงสรุปสถิติเหตุการณ์
   * @param {string} requestId
   */
  async getMessageEventAggregate(requestId) {
    try {
      const response = await this.client.get('/v2/bot/insight/message/event/aggregate', {
        params: { requestId }
      });
      return response.data;
    } catch (error) {
      this.handleError(error);
    }
  }

  /**
   * คำนวณ Click-Through Rate (CTR)
   */
  calculateCTR(stats) {
    if (!stats.overview || stats.overview.success === 0) return 0;
    const totalClicks = stats.clicks
      ? stats.clicks.reduce((sum, click) => sum + (click.click || 0), 0)
      : 0;
    return Math.round((totalClicks / stats.overview.success) * 100 * 100) / 100;
  }

  /**
   * คำนวณ Open Rate
   */
  calculateOpenRate(stats) {
    if (!stats.overview || stats.overview.success === 0) return 0;
    const totalImpressions = stats.messages
      ? stats.messages.reduce((sum, msg) => sum + (msg.impression || 0), 0)
      : 0;
    return Math.round((totalImpressions / stats.overview.success) * 100 * 100) / 100;
  }

  /**
   * วิเคราะห์ประสิทธิภาพ Broadcast
   */
  async analyzeBroadcastPerformance(requestId) {
    const [eventStats, aggregateStats] = await Promise.all([
      this.getMessageEventStats(requestId),
      this.getMessageEventAggregate(requestId)
    ]);

    return {
      requestId,
      overview: eventStats.overview,
      ctr: this.calculateCTR(eventStats),
      openRate: this.calculateOpenRate(eventStats),
      topLinks: this.getTopLinks(eventStats.clicks),
      messagePerformance: eventStats.messages,
      aggregate: aggregateStats
    };
  }

  getTopLinks(clicks) {
    if (!clicks) return [];
    return clicks
      .sort((a, b) => (b.click || 0) - (a.click || 0))
      .slice(0, 5)
      .map(click => ({
        url: click.url,
        clicks: click.click,
        uniqueClicks: click.uniqueClick
      }));
  }
}

module.exports = BroadcastStatsService;
```

---

## 7. Fetching Statistics by Date Range (ดึงสถิติตามช่วงเวลา)

### 7.1 Data Fetcher หลัก

```javascript
// services/data-fetcher.js
const InsightApiService = require('./insight-api');
const FollowerService = require('./follower-service');
const DemographicService = require('./demographic-service');
const BroadcastStatsService = require('./broadcast-stats-service');
const db = require('../database/db');

class DataFetcher {
  constructor() {
    this.insightApi = new InsightApiService();
    this.followerService = new FollowerService();
    this.demographicService = new DemographicService();
    this.broadcastService = new BroadcastStatsService();
  }

  /**
   * ดึงข้อมูลทั้งหมดสำหรับช่วงเวลา
   * @param {string} startDate - YYYYMMDD
   * @param {string} endDate - YYYYMMDD
   */
  async fetchAllStats(startDate, endDate) {
    console.log(`เริ่มดึงข้อมูลตั้งแต่ ${startDate} ถึง ${endDate}`);

    const [deliveryStats, followerStats] = await Promise.all([
      this.insightApi.getMessageDeliveryRange(startDate, endDate),
      this.followerService.analyzeFollowerGrowth(startDate, endDate)
    ]);

    // ดึงข้อมูล Demographics สำหรับวันล่าสุด
    const demographicStats = await this.demographicService.getDemographics(endDate);

    return {
      period: { startDate, endDate },
      delivery: deliveryStats,
      followers: followerStats,
      demographics: demographicStats,
      fetchedAt: new Date().toISOString()
    };
  }

  /**
   * บันทึกข้อมูลลงฐานข้อมูล
   */
  async saveToDatabase(stats) {
    const stmt = db.prepare(`
      INSERT OR REPLACE INTO daily_stats
      (date, success_count, follower_count, new_followers, blocks)
      VALUES (?, ?, ?, ?, ?)
    `);

    for (const delivery of stats.delivery) {
      const followerData = stats.followers.data.find(f => f.date === delivery.date);
      stmt.run(
        delivery.date,
        delivery.overview?.success || 0,
        followerData?.followers || 0,
        followerData?.newFollowers || 0,
        followerData?.blocks || 0
      );
    }

    console.log('บันทึกข้อมูลสำเร็จ');
  }

  /**
   * ดึงข้อมูลรายวันแบบ Auto
   */
  async fetchDailyStats() {
    const today = new Date();
    const yesterday = new Date(today);
    yesterday.setDate(yesterday.getDate() - 1);

    const formatDate = (date) => {
      return date.toISOString().slice(0, 10).replace(/-/g, '');
    };

    const dateStr = formatDate(yesterday);
    console.log(`ดึงข้อมูลวันที่ ${dateStr}`);

    const [delivery, follower, demographic] = await Promise.all([
      this.insightApi.getMessageDeliveryStats(dateStr),
      this.followerService.getFollowerStats(dateStr),
      this.demographicService.getDemographics(dateStr)
    ]);

    return {
      date: dateStr,
      delivery,
      follower,
      demographic
    };
  }
}

module.exports = DataFetcher;
```

---

## 8. Database Schema (โครงสร้างฐานข้อมูล)

### 8.1 SQLite Schema

```sql
-- database/schema.sql
CREATE TABLE IF NOT EXISTS daily_stats (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT NOT NULL UNIQUE,
  success_count INTEGER DEFAULT 0,
  failure_count INTEGER DEFAULT 0,
  follower_count INTEGER DEFAULT 0,
  new_followers INTEGER DEFAULT 0,
  blocks INTEGER DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS demographics (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT NOT NULL,
  category TEXT NOT NULL,
  label TEXT NOT NULL,
  percentage REAL DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS broadcast_stats (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  request_id TEXT NOT NULL UNIQUE,
  date TEXT NOT NULL,
  success_count INTEGER DEFAULT 0,
  failure_count INTEGER DEFAULT 0,
  impression_count INTEGER DEFAULT 0,
  click_count INTEGER DEFAULT 0,
  ctr REAL DEFAULT 0,
  open_rate REAL DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS link_clicks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  broadcast_id INTEGER REFERENCES broadcast_stats(id),
  url TEXT NOT NULL,
  click_count INTEGER DEFAULT 0,
  unique_click_count INTEGER DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_daily_stats_date ON daily_stats(date);
CREATE INDEX IF NOT EXISTS idx_demographics_date ON demographics(date);
CREATE INDEX IF NOT EXISTS idx_broadcast_stats_date ON broadcast_stats(date);
```

### 8.2 Database Connection

```javascript
// database/db.js
const Database = require('better-sqlite3');
const path = require('path');
const fs = require('fs');

const DB_PATH = path.join(__dirname, 'insights.db');
const SCHEMA_PATH = path.join(__dirname, 'schema.sql');

let db;

function getDb() {
  if (!db) {
    db = new Database(DB_PATH);
    db.pragma('journal_mode = WAL');
    db.pragma('foreign_keys = ON');

    // สร้างตารางถ้ายังไม่มี
    const schema = fs.readFileSync(SCHEMA_PATH, 'utf8');
    db.exec(schema);

    console.log('เชื่อมต่อฐานข้อมูลสำเร็จ');
  }
  return db;
}

module.exports = getDb();
```

---

## 9. Building Analytics Dashboard (สร้าง Dashboard)

### 9.1 Express Server

```javascript
// server.js
require('dotenv').config();
const express = require('express');
const path = require('path');
const dashboardRoutes = require('./routes/dashboard');
const DataFetcher = require('./services/data-fetcher');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Routes
app.use('/api', dashboardRoutes);

// หน้า Dashboard
app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Dashboard รันอยู่ที่ http://localhost:${PORT}`);
});

module.exports = app;
```

### 9.2 Dashboard Routes

```javascript
// routes/dashboard.js
const express = require('express');
const router = express.Router();
const DataFetcher = require('../services/data-fetcher');
const db = require('../database/db');

const fetcher = new DataFetcher();

// GET /api/stats/overview
router.get('/stats/overview', async (req, res) => {
  try {
    const { startDate, endDate } = req.query;

    if (!startDate || !endDate) {
      return res.status(400).json({
        error: 'กรุณาระบุ startDate และ endDate ในรูปแบบ YYYYMMDD'
      });
    }

    const stats = await fetcher.fetchAllStats(startDate, endDate);
    res.json(stats);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /api/stats/followers
router.get('/stats/followers', async (req, res) => {
  try {
    const { startDate, endDate } = req.query;
    const stats = await fetcher.followerService.analyzeFollowerGrowth(startDate, endDate);
    res.json(stats);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /api/stats/demographics
router.get('/stats/demographics', async (req, res) => {
  try {
    const { date } = req.query;
    const demographics = await fetcher.demographicService.getDemographics(
      date || getTodayStr()
    );

    // แยกข้อมูลแต่ละประเภท
    res.json({
      genders: fetcher.demographicService.parseGenderData(demographics),
      ages: demographics.ages,
      os: fetcher.demographicService.parseOSData(demographics),
      regions: fetcher.demographicService.parseRegionData(demographics),
      subscriptionPeriods: fetcher.demographicService.parseSubscriptionPeriod(demographics)
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /api/stats/broadcast/:requestId
router.get('/stats/broadcast/:requestId', async (req, res) => {
  try {
    const { requestId } = req.params;
    const stats = await fetcher.broadcastService.analyzeBroadcastPerformance(requestId);
    res.json(stats);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /api/stats/history
router.get('/stats/history', (req, res) => {
  try {
    const { days = 30 } = req.query;
    const history = db.prepare(`
      SELECT * FROM daily_stats
      ORDER BY date DESC
      LIMIT ?
    `).all(parseInt(days));

    res.json(history);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

function getTodayStr() {
  return new Date().toISOString().slice(0, 10).replace(/-/g, '');
}

module.exports = router;
```

### 9.3 Frontend Dashboard HTML

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LINE OA Insight Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'Segoe UI', sans-serif;
      background: #f0f2f5;
      color: #333;
    }
    .header {
      background: #06C755;
      color: white;
      padding: 16px 24px;
      display: flex;
      align-items: center;
      gap: 12px;
    }
    .header h1 { font-size: 1.5rem; }
    .container { max-width: 1400px; margin: 0 auto; padding: 24px; }
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
      gap: 16px;
      margin-bottom: 24px;
    }
    .stat-card {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
    }
    .stat-card .label { color: #666; font-size: 0.875rem; margin-bottom: 8px; }
    .stat-card .value { font-size: 2rem; font-weight: 700; color: #06C755; }
    .stat-card .change { font-size: 0.875rem; margin-top: 4px; }
    .change.positive { color: #16a34a; }
    .change.negative { color: #dc2626; }
    .charts-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(500px, 1fr));
      gap: 24px;
    }
    .chart-card {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
    }
    .chart-card h3 { margin-bottom: 16px; color: #444; }
    .date-picker {
      display: flex;
      gap: 12px;
      align-items: center;
      margin-bottom: 24px;
      background: white;
      padding: 16px;
      border-radius: 12px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
    }
    .date-picker input {
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 8px;
      font-size: 0.9rem;
    }
    .btn {
      padding: 8px 20px;
      background: #06C755;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      font-size: 0.9rem;
    }
    .btn:hover { background: #05a847; }
  </style>
</head>
<body>
  <div class="header">
    <svg width="32" height="32" viewBox="0 0 32 32" fill="white">
      <path d="M16 3C9.373 3 4 8.373 4 15c0 6.627 5.373 12 12 12s12-5.373 12-12C28 8.373 22.627 3 16 3z"/>
    </svg>
    <h1>LINE OA Insight Dashboard</h1>
  </div>

  <div class="container">
    <div class="date-picker">
      <label>ช่วงเวลา:</label>
      <input type="date" id="startDate">
      <span>ถึง</span>
      <input type="date" id="endDate">
      <button class="btn" onclick="loadData()">โหลดข้อมูล</button>
    </div>

    <div class="stats-grid">
      <div class="stat-card">
        <div class="label">ผู้ติดตามทั้งหมด</div>
        <div class="value" id="totalFollowers">-</div>
        <div class="change" id="followerChange">-</div>
      </div>
      <div class="stat-card">
        <div class="label">ข้อความส่งสำเร็จ</div>
        <div class="value" id="totalDelivery">-</div>
        <div class="change">ในช่วงเวลาที่เลือก</div>
      </div>
      <div class="stat-card">
        <div class="label">อัตราการเปิดอ่าน</div>
        <div class="value" id="openRate">-</div>
        <div class="change">เฉลี่ย</div>
      </div>
      <div class="stat-card">
        <div class="label">ผู้ติดตามใหม่</div>
        <div class="value" id="newFollowers">-</div>
        <div class="change" id="avgDailyGrowth">-</div>
      </div>
    </div>

    <div class="charts-grid">
      <div class="chart-card">
        <h3>การเติบโตของผู้ติดตาม</h3>
        <canvas id="followerChart"></canvas>
      </div>
      <div class="chart-card">
        <h3>สถิติการส่งข้อความ</h3>
        <canvas id="deliveryChart"></canvas>
      </div>
      <div class="chart-card">
        <h3>เพศของผู้ติดตาม</h3>
        <canvas id="genderChart"></canvas>
      </div>
      <div class="chart-card">
        <h3>กลุ่มอายุ</h3>
        <canvas id="ageChart"></canvas>
      </div>
      <div class="chart-card">
        <h3>ระบบปฏิบัติการ</h3>
        <canvas id="osChart"></canvas>
      </div>
      <div class="chart-card">
        <h3>ภูมิภาค Top 10</h3>
        <canvas id="regionChart"></canvas>
      </div>
    </div>
  </div>

  <script src="charts.js"></script>
</body>
</html>
```

### 9.4 Charts JavaScript

```javascript
// public/charts.js
let charts = {};

// ตั้งค่าวันที่เริ่มต้น
window.addEventListener('DOMContentLoaded', () => {
  const today = new Date();
  const thirtyDaysAgo = new Date(today);
  thirtyDaysAgo.setDate(today.getDate() - 30);

  document.getElementById('endDate').value = today.toISOString().slice(0, 10);
  document.getElementById('startDate').value = thirtyDaysAgo.toISOString().slice(0, 10);

  loadData();
});

async function loadData() {
  const startDate = document.getElementById('startDate').value.replace(/-/g, '');
  const endDate = document.getElementById('endDate').value.replace(/-/g, '');

  try {
    const [overview, demographics] = await Promise.all([
      fetch(`/api/stats/overview?startDate=${startDate}&endDate=${endDate}`).then(r => r.json()),
      fetch(`/api/stats/demographics?date=${endDate}`).then(r => r.json())
    ]);

    updateStats(overview);
    renderFollowerChart(overview.followers.data);
    renderDeliveryChart(overview.delivery);
    renderGenderChart(demographics.genders);
    renderAgeChart(demographics.ages);
    renderOsChart(demographics.os);
    renderRegionChart(demographics.regions);
  } catch (error) {
    console.error('เกิดข้อผิดพลาด:', error);
  }
}

function updateStats(overview) {
  const summary = overview.followers.summary;
  document.getElementById('totalFollowers').textContent =
    overview.followers.data[overview.followers.data.length - 1]?.followers?.toLocaleString() || '-';
  document.getElementById('newFollowers').textContent =
    summary.totalNewFollowers?.toLocaleString() || '-';
  document.getElementById('avgDailyGrowth').textContent =
    `เฉลี่ย ${summary.averageDailyGrowth}/วัน`;

  const totalDelivery = overview.delivery.reduce((sum, d) =>
    sum + (d.overview?.success || 0), 0);
  document.getElementById('totalDelivery').textContent = totalDelivery.toLocaleString();
}

function renderFollowerChart(data) {
  const ctx = document.getElementById('followerChart').getContext('2d');
  if (charts.follower) charts.follower.destroy();

  charts.follower = new Chart(ctx, {
    type: 'line',
    data: {
      labels: data.map(d => formatDate(d.date)),
      datasets: [{
        label: 'จำนวนผู้ติดตาม',
        data: data.map(d => d.followers),
        borderColor: '#06C755',
        backgroundColor: 'rgba(6, 199, 85, 0.1)',
        fill: true,
        tension: 0.4
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { position: 'top' }
      },
      scales: {
        y: { beginAtZero: false }
      }
    }
  });
}

function renderDeliveryChart(data) {
  const ctx = document.getElementById('deliveryChart').getContext('2d');
  if (charts.delivery) charts.delivery.destroy();

  charts.delivery = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: data.map(d => formatDate(d.date)),
      datasets: [{
        label: 'ส่งสำเร็จ',
        data: data.map(d => d.overview?.success || 0),
        backgroundColor: '#06C755'
      }, {
        label: 'ส่งไม่สำเร็จ',
        data: data.map(d => d.overview?.failure || 0),
        backgroundColor: '#ef4444'
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { position: 'top' }
      }
    }
  });
}

function renderGenderChart(genders) {
  const ctx = document.getElementById('genderChart').getContext('2d');
  if (charts.gender) charts.gender.destroy();

  charts.gender = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels: genders.map(g => g.gender),
      datasets: [{
        data: genders.map(g => g.percentage),
        backgroundColor: ['#3b82f6', '#ec4899']
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { position: 'bottom' }
      }
    }
  });
}

function renderAgeChart(ages) {
  const ctx = document.getElementById('ageChart').getContext('2d');
  if (charts.age) charts.age.destroy();

  const labels = {
    'AGE_UNDER18': 'ต่ำกว่า 18',
    'AGE_18TO24': '18-24 ปี',
    'AGE_25TO34': '25-34 ปี',
    'AGE_35TO44': '35-44 ปี',
    'AGE_45TO54': '45-54 ปี',
    'AGE_OVER55': '55 ปีขึ้นไป'
  };

  charts.age = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: ages.map(a => labels[a.age] || a.age),
      datasets: [{
        label: 'เปอร์เซ็นต์',
        data: ages.map(a => a.percentage),
        backgroundColor: [
          '#06C755', '#3b82f6', '#f59e0b',
          '#ef4444', '#8b5cf6', '#ec4899'
        ]
      }]
    },
    options: {
      indexAxis: 'y',
      responsive: true,
      plugins: {
        legend: { display: false }
      }
    }
  });
}

function renderOsChart(osData) {
  const ctx = document.getElementById('osChart').getContext('2d');
  if (charts.os) charts.os.destroy();

  charts.os = new Chart(ctx, {
    type: 'pie',
    data: {
      labels: osData.map(o => o.os.toUpperCase()),
      datasets: [{
        data: osData.map(o => o.percentage),
        backgroundColor: ['#06C755', '#3b82f6', '#f59e0b']
      }]
    },
    options: {
      responsive: true,
      plugins: {
        legend: { position: 'bottom' }
      }
    }
  });
}

function renderRegionChart(regions) {
  const ctx = document.getElementById('regionChart').getContext('2d');
  if (charts.region) charts.region.destroy();

  const top10 = regions.slice(0, 10);

  charts.region = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: top10.map(r => r.region),
      datasets: [{
        label: 'เปอร์เซ็นต์',
        data: top10.map(r => r.percentage),
        backgroundColor: '#06C755'
      }]
    },
    options: {
      indexAxis: 'y',
      responsive: true,
      plugins: {
        legend: { display: false }
      }
    }
  });
}

function formatDate(dateStr) {
  const year = dateStr.slice(0, 4);
  const month = dateStr.slice(4, 6);
  const day = dateStr.slice(6, 8);
  return `${day}/${month}`;
}
```

---

## 10. Automated Data Collection (ระบบดึงข้อมูลอัตโนมัติ)

### 10.1 Cron Job สำหรับดึงข้อมูลรายวัน

```javascript
// jobs/daily-fetcher.js
const cron = require('node-cron');
const DataFetcher = require('../services/data-fetcher');

const fetcher = new DataFetcher();

// รันทุกวันเวลา 02:00 น.
cron.schedule('0 2 * * *', async () => {
  console.log(`[${new Date().toISOString()}] เริ่มดึงข้อมูลรายวัน`);

  try {
    const stats = await fetcher.fetchDailyStats();
    await fetcher.saveToDatabase({ delivery: [stats.delivery], followers: { data: [stats.follower] } });
    console.log('ดึงและบันทึกข้อมูลสำเร็จ');
  } catch (error) {
    console.error('เกิดข้อผิดพลาดในการดึงข้อมูล:', error);
  }
}, {
  timezone: 'Asia/Bangkok'
});

console.log('เริ่มระบบดึงข้อมูลรายวันแล้ว');
```

### 10.2 ตัวอย่างการรันด้วย Package.json

```json
{
  "name": "line-insight-dashboard",
  "version": "1.0.0",
  "description": "LINE OA Insight Dashboard",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "fetch": "node jobs/daily-fetcher.js",
    "test": "jest --coverage"
  },
  "dependencies": {
    "axios": "^1.6.0",
    "better-sqlite3": "^9.0.0",
    "chart.js": "^4.4.0",
    "dotenv": "^16.3.0",
    "express": "^4.18.0",
    "node-cron": "^3.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "nodemon": "^3.0.0"
  }
}
```

---

## 11. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINE OA Insight Dashboard                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌─────────────────┐     ┌─────────────┐  │
│  │  LINE API    │────▶│  Data Fetcher   │────▶│  SQLite DB  │  │
│  │  (Insight)   │     │  (Scheduled)    │     │             │  │
│  └──────────────┘     └─────────────────┘     └─────────────┘  │
│         │                      │                      │         │
│         │             ┌────────┴───────┐              │         │
│         │             │  Services      │              │         │
│         │             ├────────────────┤              │         │
│         │             │ - InsightAPI   │              │         │
│         │             │ - Follower     │              │         │
│         │             │ - Demographic  │              │         │
│         │             │ - Broadcast    │              │         │
│         │             └────────────────┘              │         │
│         │                                             │         │
│  ┌──────▼──────────────────────────────────────────▼──────┐   │
│  │                  Express REST API                        │   │
│  │  GET /api/stats/overview                                 │   │
│  │  GET /api/stats/followers                                │   │
│  │  GET /api/stats/demographics                             │   │
│  │  GET /api/stats/broadcast/:id                            │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │               Frontend Dashboard (Chart.js)               │   │
│  │  - Follower Growth Chart                                  │   │
│  │  - Message Delivery Chart                                 │   │
│  │  - Demographics Charts (Gender/Age/OS/Region)             │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. Best Practices สำหรับ Production

### 12.1 Rate Limiting

```javascript
// utils/rate-limiter.js
class RateLimiter {
  constructor(maxRequests = 10, windowMs = 1000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = [];
  }

  async throttle() {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = this.requests[0];
      const waitTime = this.windowMs - (now - oldestRequest);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }

    this.requests.push(Date.now());
  }
}

module.exports = RateLimiter;
```

### 12.2 Caching

```javascript
// utils/cache.js
class SimpleCache {
  constructor(ttl = 300000) { // 5 minutes default
    this.cache = new Map();
    this.ttl = ttl;
  }

  set(key, value) {
    this.cache.set(key, {
      value,
      expiry: Date.now() + this.ttl
    });
  }

  get(key) {
    const item = this.cache.get(key);
    if (!item) return null;
    if (Date.now() > item.expiry) {
      this.cache.delete(key);
      return null;
    }
    return item.value;
  }

  has(key) {
    return this.get(key) !== null;
  }
}

module.exports = SimpleCache;
```

---

## 13. Testing

### 13.1 Unit Tests

```javascript
// tests/insight-api.test.js
const InsightApiService = require('../services/insight-api');
const axios = require('axios');

jest.mock('axios');

describe('InsightApiService', () => {
  let service;

  beforeEach(() => {
    service = new InsightApiService();
  });

  test('getMessageDeliveryStats ควรดึงข้อมูลสำเร็จ', async () => {
    const mockData = {
      overview: { requestId: 'test-123', success: 1000, failure: 0 }
    };

    axios.create.mockReturnValue({
      get: jest.fn().mockResolvedValue({ data: mockData })
    });

    const result = await service.getMessageDeliveryStats('20240101');
    expect(result).toEqual(mockData);
  });

  test('generateDateRange ควรสร้างรายการวันที่ถูกต้อง', () => {
    const dates = service.generateDateRange('20240101', '20240105');
    expect(dates).toEqual(['20240101', '20240102', '20240103', '20240104', '20240105']);
  });

  test('handleError ควรโยน Error ที่มีรายละเอียด', () => {
    const mockError = {
      response: {
        status: 401,
        data: { message: 'Unauthorized' }
      }
    };

    expect(() => service.handleError(mockError)).toThrow('LINE API Error 401');
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **LINE OA Insight API** - การดึงข้อมูลสถิติต่างๆ ผ่าน API
2. **Message Delivery Stats** - สถิติการส่งข้อความ
3. **Follower Statistics** - การติดตามจำนวนผู้ติดตาม
4. **Demographic Data** - ข้อมูลประชากรศาสตร์ (อายุ, เพศ, OS, ภูมิภาค)
5. **Broadcast Statistics** - วิเคราะห์ประสิทธิภาพการ Broadcast
6. **Dashboard Development** - สร้าง Dashboard ด้วย Chart.js
7. **Database Storage** - เก็บข้อมูลใน SQLite
8. **Automated Collection** - ดึงข้อมูลอัตโนมัติด้วย Cron Job

ขั้นตอนต่อไป: ใน Part 57 เราจะเรียนรู้เรื่อง Webhook Reliability และการจัดการ Queue
