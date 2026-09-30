# Part 74: A/B Testing Framework สำหรับ LINE Bot

## บทนำ

A/B Testing หรือ Split Testing คือวิธีการทดสอบว่า version ไหนของ message, design, หรือ flow ให้ผลลัพธ์ที่ดีกว่า โดยแสดง version ต่างๆ ให้ผู้ใช้กลุ่มต่างๆ และวัดผล

---

## 1. ทำไมต้องใช้ A/B Testing สำหรับ LINE Bot

### 1.1 สิ่งที่สามารถทดสอบได้

```
┌─────────────────────────────────────────────────────────┐
│              A/B Testing Opportunities                  │
│                                                         │
│  Messages:                                              │
│  ├── Welcome message (tone, length, CTA)               │
│  ├── Reply text (formal vs casual)                      │
│  └── Error messages                                     │
│                                                         │
│  Visual Design:                                         │
│  ├── Flex message layout                                │
│  ├── Color schemes                                      │
│  ├── Button placement                                   │
│  └── Image vs no image                                  │
│                                                         │
│  Flow:                                                  │
│  ├── Onboarding steps                                   │
│  ├── Menu structure                                     │
│  ├── Checkout flow                                      │
│  └── Question order in forms                           │
│                                                         │
│  Timing:                                                │
│  ├── Broadcast time                                     │
│  ├── Follow-up message delay                           │
│  └── Reminder frequency                                │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Metrics ที่วัดได้

- **Click-through rate (CTR)**: กดปุ่ม / เห็นข้อความ
- **Conversion rate**: ซื้อสินค้า, ลงทะเบียน
- **Engagement rate**: ตอบกลับ, postback
- **Retention rate**: กลับมาใช้ bot
- **Unfollow rate**: ยกเลิกติดตาม

---

## 2. Experiment Design

### 2.1 Experiment Model

```javascript
// models/Experiment.js
const { DataTypes } = require('sequelize');
const { sequelize } = require('../database');

const Experiment = sequelize.define('Experiment', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  name: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  description: {
    type: DataTypes.TEXT
  },
  hypothesis: {
    type: DataTypes.TEXT,
    comment: 'เราเชื่อว่า... จะทำให้... เพราะ...'
  },
  status: {
    type: DataTypes.ENUM('draft', 'running', 'paused', 'completed', 'archived'),
    defaultValue: 'draft'
  },
  type: {
    type: DataTypes.ENUM('message', 'flow', 'timing', 'design'),
    allowNull: false
  },
  variants: {
    type: DataTypes.JSONB,
    defaultValue: [],
    comment: '[{ id, name, weight, config }]'
  },
  targetAudience: {
    type: DataTypes.JSONB,
    defaultValue: {},
    comment: '{ segments: [], percentage: 100 }'
  },
  metrics: {
    type: DataTypes.JSONB,
    defaultValue: [],
    comment: '[{ name, type, goal }]'
  },
  startedAt: {
    type: DataTypes.DATE
  },
  endedAt: {
    type: DataTypes.DATE
  },
  minimumSampleSize: {
    type: DataTypes.INTEGER,
    defaultValue: 1000,
    comment: 'จำนวน users ขั้นต่ำต่อ variant'
  },
  confidenceLevel: {
    type: DataTypes.FLOAT,
    defaultValue: 0.95,
    comment: '0.95 = 95% confidence'
  },
  winnerVariantId: {
    type: DataTypes.STRING(100)
  },
  results: {
    type: DataTypes.JSONB,
    defaultValue: {}
  }
}, {
  tableName: 'experiments',
  indexes: [
    { fields: ['status'] },
    { fields: ['type'] }
  ]
});

const ExperimentAssignment = sequelize.define('ExperimentAssignment', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  experimentId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  userId: {
    type: DataTypes.STRING(100),
    allowNull: false
  },
  variantId: {
    type: DataTypes.STRING(100),
    allowNull: false
  },
  assignedAt: {
    type: DataTypes.DATE,
    defaultValue: DataTypes.NOW
  }
}, {
  tableName: 'experiment_assignments',
  indexes: [
    { fields: ['experimentId', 'userId'], unique: true },
    { fields: ['experimentId', 'variantId'] }
  ]
});

const ExperimentEvent = sequelize.define('ExperimentEvent', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  experimentId: {
    type: DataTypes.UUID,
    allowNull: false
  },
  userId: {
    type: DataTypes.STRING(100),
    allowNull: false
  },
  variantId: {
    type: DataTypes.STRING(100),
    allowNull: false
  },
  eventType: {
    type: DataTypes.STRING(100),
    allowNull: false,
    comment: 'impression, click, conversion, etc.'
  },
  metadata: {
    type: DataTypes.JSONB,
    defaultValue: {}
  },
  occurredAt: {
    type: DataTypes.DATE,
    defaultValue: DataTypes.NOW
  }
}, {
  tableName: 'experiment_events',
  indexes: [
    { fields: ['experimentId', 'variantId', 'eventType'] },
    { fields: ['userId', 'experimentId'] },
    { fields: ['occurredAt'] }
  ]
});

module.exports = { Experiment, ExperimentAssignment, ExperimentEvent };
```

---

## 3. User Assignment (Consistent Hashing)

### 3.1 Deterministic Assignment Algorithm

```javascript
// ab-testing/src/UserAssigner.js
const crypto = require('crypto');
const Redis = require('ioredis');
const { ExperimentAssignment } = require('./models/Experiment');
const logger = require('./utils/logger');

class UserAssigner {
  constructor(redis) {
    this.redis = redis;
    this.cacheTTL = 3600 * 24; // 24 hours
  }

  // กำหนด variant ให้ user (consistent - user เดิมได้ variant เดิมเสมอ)
  async assignVariant(userId, experiment) {
    const { id: experimentId, variants, targetAudience } = experiment;
    
    // ตรวจสอบว่า user อยู่ใน target audience ไหม
    if (!this.isInTargetAudience(userId, targetAudience)) {
      return null;
    }

    // ลองดู cache ก่อน
    const cached = await this.getCachedAssignment(userId, experimentId);
    if (cached) return cached;

    // ดูใน database
    const existing = await ExperimentAssignment.findOne({
      where: { experimentId, userId }
    });

    if (existing) {
      await this.cacheAssignment(userId, experimentId, existing.variantId);
      return existing.variantId;
    }

    // กำหนด variant ใหม่ด้วย consistent hashing
    const variantId = this.selectVariant(userId, experimentId, variants);
    
    // บันทึก assignment
    await ExperimentAssignment.create({
      experimentId,
      userId,
      variantId
    });

    await this.cacheAssignment(userId, experimentId, variantId);
    
    logger.debug(`Assigned user ${userId} to variant ${variantId} in experiment ${experimentId}`);
    
    return variantId;
  }

  // Consistent hashing: user เดิม → variant เดิมเสมอ
  selectVariant(userId, experimentId, variants) {
    // สร้าง hash จาก userId + experimentId
    const hash = crypto
      .createHash('md5')
      .update(`${userId}-${experimentId}`)
      .digest('hex');
    
    // Convert first 8 chars to number (0-4294967295)
    const hashNum = parseInt(hash.substring(0, 8), 16);
    
    // Normalize to 0-1
    const normalized = hashNum / 0xFFFFFFFF;

    // Select variant based on weights
    let cumulative = 0;
    const totalWeight = variants.reduce((sum, v) => sum + (v.weight || 1), 0);
    
    for (const variant of variants) {
      cumulative += (variant.weight || 1) / totalWeight;
      if (normalized < cumulative) {
        return variant.id;
      }
    }

    // Fallback to last variant
    return variants[variants.length - 1].id;
  }

  isInTargetAudience(userId, targetAudience) {
    // Percentage-based allocation
    if (targetAudience.percentage < 100) {
      const hash = parseInt(
        crypto.createHash('md5').update(userId).digest('hex').substring(0, 4),
        16
      );
      const percentile = (hash / 0xFFFF) * 100;
      if (percentile >= targetAudience.percentage) {
        return false;
      }
    }
    
    return true;
  }

  async getCachedAssignment(userId, experimentId) {
    const key = `exp:assign:${experimentId}:${userId}`;
    return this.redis.get(key);
  }

  async cacheAssignment(userId, experimentId, variantId) {
    const key = `exp:assign:${experimentId}:${userId}`;
    await this.redis.setex(key, this.cacheTTL, variantId);
  }

  // ดึง variant config สำหรับ user
  async getVariantConfig(userId, experimentId, experiments) {
    const experiment = experiments.find(e => e.id === experimentId);
    if (!experiment || experiment.status !== 'running') return null;

    const variantId = await this.assignVariant(userId, experiment);
    if (!variantId) return null;

    const variant = experiment.variants.find(v => v.id === variantId);
    return variant?.config || null;
  }
}

module.exports = UserAssigner;
```

---

## 4. A/B Testing Manager

### 4.1 Experiment Manager

```javascript
// ab-testing/src/ExperimentManager.js
const { Experiment, ExperimentAssignment, ExperimentEvent } = require('./models/Experiment');
const UserAssigner = require('./UserAssigner');
const StatisticsCalculator = require('./StatisticsCalculator');
const Redis = require('ioredis');
const logger = require('./utils/logger');

class ExperimentManager {
  constructor(redis) {
    this.redis = redis;
    this.assigner = new UserAssigner(redis);
    this.stats = new StatisticsCalculator();
    this.activeExperiments = new Map();
    
    // Load active experiments on startup
    this.loadActiveExperiments();
    
    // Refresh every 5 minutes
    setInterval(() => this.loadActiveExperiments(), 5 * 60 * 1000);
  }

  async loadActiveExperiments() {
    try {
      const experiments = await Experiment.findAll({
        where: { status: 'running' }
      });
      
      this.activeExperiments.clear();
      experiments.forEach(exp => {
        this.activeExperiments.set(exp.id, exp.toJSON());
      });
      
      logger.info(`Loaded ${experiments.length} active experiments`);
    } catch (error) {
      logger.error('Failed to load experiments:', error);
    }
  }

  // สร้าง experiment ใหม่
  async createExperiment(params) {
    const {
      name,
      description,
      hypothesis,
      type,
      variants,
      targetAudience = { percentage: 100 },
      metrics = [],
      minimumSampleSize = 1000,
      confidenceLevel = 0.95
    } = params;

    // Validate variants weights
    const totalWeight = variants.reduce((sum, v) => sum + (v.weight || 1), 0);
    const normalizedVariants = variants.map(v => ({
      ...v,
      weight: (v.weight || 1) / totalWeight * 100
    }));

    const experiment = await Experiment.create({
      name,
      description,
      hypothesis,
      type,
      variants: normalizedVariants,
      targetAudience,
      metrics,
      minimumSampleSize,
      confidenceLevel
    });

    logger.info(`Created experiment: ${name} (${experiment.id})`);
    return experiment;
  }

  // เริ่ม experiment
  async startExperiment(experimentId) {
    const experiment = await Experiment.findByPk(experimentId);
    if (!experiment) throw new Error('Experiment not found');
    if (experiment.status !== 'draft') throw new Error('Can only start draft experiments');

    await experiment.update({
      status: 'running',
      startedAt: new Date()
    });

    // Add to active cache
    this.activeExperiments.set(experimentId, experiment.toJSON());
    
    logger.info(`Started experiment: ${experiment.name}`);
    return experiment;
  }

  // หยุด experiment
  async stopExperiment(experimentId, winnerId = null) {
    const experiment = await Experiment.findByPk(experimentId);
    if (!experiment) throw new Error('Experiment not found');

    // คำนวณ results ก่อนหยุด
    const results = await this.calculateResults(experimentId);

    await experiment.update({
      status: 'completed',
      endedAt: new Date(),
      winnerVariantId: winnerId,
      results
    });

    this.activeExperiments.delete(experimentId);
    
    logger.info(`Stopped experiment: ${experiment.name}`, {
      winner: winnerId,
      results
    });

    return { experiment, results };
  }

  // ดึง variant สำหรับ user
  async getVariantForUser(userId, experimentType) {
    const relevantExperiments = Array.from(this.activeExperiments.values())
      .filter(exp => exp.type === experimentType);

    const assignments = {};
    
    for (const experiment of relevantExperiments) {
      const variantId = await this.assigner.assignVariant(userId, experiment);
      if (variantId) {
        const variant = experiment.variants.find(v => v.id === variantId);
        assignments[experiment.id] = {
          variantId,
          config: variant?.config
        };
      }
    }

    return assignments;
  }

  // บันทึก event (impression, click, conversion)
  async trackEvent(params) {
    const {
      userId,
      experimentId,
      variantId,
      eventType,
      metadata = {}
    } = params;

    // บันทึกลง database
    await ExperimentEvent.create({
      experimentId,
      userId,
      variantId,
      eventType,
      metadata,
      occurredAt: new Date()
    });

    // Increment counter ใน Redis
    const key = `exp:events:${experimentId}:${variantId}:${eventType}`;
    await this.redis.incr(key);
    await this.redis.expire(key, 7 * 24 * 3600); // 7 days

    // Check if we should analyze
    await this.checkForSignificance(experimentId);
  }

  // ตรวจสอบ statistical significance
  async checkForSignificance(experimentId) {
    const experiment = this.activeExperiments.get(experimentId);
    if (!experiment) return;

    // ดึง current stats
    const stats = await this.getCurrentStats(experimentId, experiment.variants);
    
    // ตรวจสอบ minimum sample size
    const allHaveMinSamples = experiment.variants.every(v => 
      (stats[v.id]?.impressions || 0) >= experiment.minimumSampleSize
    );

    if (!allHaveMinSamples) return;

    // คำนวณ significance
    const results = this.stats.calculateSignificance(
      stats,
      experiment.variants,
      experiment.confidenceLevel
    );

    if (results.hasWinner) {
      logger.info(`Experiment ${experimentId} has significant winner: ${results.winnerId}`);
      
      // Auto-notify (ไม่ auto-stop เพราะต้องการ human decision)
      await this.notifySignificantResult(experiment, results);
    }
  }

  async getCurrentStats(experimentId, variants) {
    const stats = {};
    
    for (const variant of variants) {
      const impressions = parseInt(
        await this.redis.get(`exp:events:${experimentId}:${variant.id}:impression`) || '0'
      );
      const clicks = parseInt(
        await this.redis.get(`exp:events:${experimentId}:${variant.id}:click`) || '0'
      );
      const conversions = parseInt(
        await this.redis.get(`exp:events:${experimentId}:${variant.id}:conversion`) || '0'
      );

      stats[variant.id] = {
        impressions,
        clicks,
        conversions,
        clickRate: impressions > 0 ? clicks / impressions : 0,
        conversionRate: impressions > 0 ? conversions / impressions : 0
      };
    }

    return stats;
  }

  async calculateResults(experimentId) {
    const experiment = await Experiment.findByPk(experimentId);
    const stats = await this.getCurrentStats(experimentId, experiment.variants);
    
    return {
      stats,
      significance: this.stats.calculateSignificance(
        stats,
        experiment.variants,
        experiment.confidenceLevel
      ),
      duration: experiment.startedAt 
        ? Math.round((Date.now() - experiment.startedAt.getTime()) / (1000 * 60 * 60 * 24)) 
        : 0,
      totalParticipants: Object.values(stats).reduce((sum, s) => sum + (s.impressions || 0), 0)
    };
  }

  async notifySignificantResult(experiment, results) {
    // ส่งแจ้งเตือน admin
    logger.info('Significant result found', { experiment: experiment.name, results });
    // TODO: Send notification via LINE or Slack
  }
}

module.exports = ExperimentManager;
```

---

## 5. Statistical Significance Calculator

### 5.1 Chi-Square Test และ Z-Test

```javascript
// ab-testing/src/StatisticsCalculator.js
const jStat = require('jstat');

class StatisticsCalculator {
  // Z-test สำหรับ proportion (click rate, conversion rate)
  calculateZTest(controlStats, treatmentStats, metric = 'conversionRate') {
    const p1 = controlStats[metric];
    const p2 = treatmentStats[metric];
    const n1 = controlStats.impressions;
    const n2 = treatmentStats.impressions;

    if (n1 < 30 || n2 < 30) {
      return { 
        pValue: 1, 
        zScore: 0, 
        significant: false,
        reason: 'Insufficient sample size'
      };
    }

    // Pooled proportion
    const pPooled = (p1 * n1 + p2 * n2) / (n1 + n2);
    
    // Standard error
    const se = Math.sqrt(pPooled * (1 - pPooled) * (1/n1 + 1/n2));
    
    if (se === 0) {
      return { pValue: 1, zScore: 0, significant: false };
    }

    // Z-score
    const zScore = (p2 - p1) / se;
    
    // Two-tailed p-value
    const pValue = 2 * (1 - jStat.normal.cdf(Math.abs(zScore), 0, 1));
    
    // Lift (improvement)
    const lift = p1 > 0 ? ((p2 - p1) / p1) * 100 : 0;
    
    // Confidence interval (95%)
    const margin = 1.96 * se;
    const ciLow = (p2 - p1) - margin;
    const ciHigh = (p2 - p1) + margin;

    return {
      pValue,
      zScore,
      lift,
      confidenceInterval: { low: ciLow, high: ciHigh },
      significant: pValue < 0.05,
      direction: p2 > p1 ? 'positive' : 'negative'
    };
  }

  // คำนวณ significance สำหรับทุก variants เทียบกับ control
  calculateSignificance(stats, variants, confidenceLevel = 0.95) {
    if (variants.length < 2) {
      return { hasWinner: false, results: [] };
    }

    // ใช้ variant แรกเป็น control
    const controlVariant = variants[0];
    const controlStats = stats[controlVariant.id];

    const results = [];
    let bestVariant = controlVariant;
    let bestConversionRate = controlStats?.conversionRate || 0;

    for (let i = 1; i < variants.length; i++) {
      const variant = variants[i];
      const variantStats = stats[variant.id];

      const test = this.calculateZTest(controlStats, variantStats, 'conversionRate');
      const clickTest = this.calculateZTest(controlStats, variantStats, 'clickRate');

      results.push({
        variantId: variant.id,
        variantName: variant.name,
        ...test,
        clickRateTest: clickTest,
        stats: variantStats
      });

      if (variantStats?.conversionRate > bestConversionRate) {
        bestConversionRate = variantStats.conversionRate;
        bestVariant = variant;
      }
    }

    // ตรวจสอบว่ามี winner ไหม
    const significantResults = results.filter(r => 
      r.significant && r.direction === 'positive' && r.pValue < (1 - confidenceLevel)
    );

    const hasWinner = significantResults.length > 0;
    const winnerId = hasWinner 
      ? significantResults.sort((a, b) => a.pValue - b.pValue)[0].variantId
      : null;

    return {
      hasWinner,
      winnerId,
      results,
      controlVariantId: controlVariant.id,
      minimumDetectableEffect: this.calculateMDE(
        controlStats?.conversionRate || 0,
        controlStats?.impressions || 0
      )
    };
  }

  // คำนวณ Minimum Detectable Effect
  calculateMDE(baseRate, sampleSize, power = 0.8, alpha = 0.05) {
    if (sampleSize < 30 || baseRate <= 0 || baseRate >= 1) return null;

    const zAlpha = jStat.normal.inv(1 - alpha/2, 0, 1);
    const zBeta = jStat.normal.inv(power, 0, 1);
    
    const mde = (zAlpha + zBeta) * Math.sqrt(2 * baseRate * (1 - baseRate) / sampleSize);
    
    return {
      absolute: mde,
      relative: baseRate > 0 ? (mde / baseRate) * 100 : null
    };
  }

  // คำนวณ required sample size
  calculateRequiredSampleSize(baseRate, minimumEffect, power = 0.8, alpha = 0.05) {
    const zAlpha = jStat.normal.inv(1 - alpha/2, 0, 1);
    const zBeta = jStat.normal.inv(power, 0, 1);
    
    const p2 = baseRate * (1 + minimumEffect / 100);
    const pAvg = (baseRate + p2) / 2;
    
    const n = Math.pow(zAlpha * Math.sqrt(2 * pAvg * (1 - pAvg)) + zBeta * Math.sqrt(baseRate * (1 - baseRate) + p2 * (1 - p2)), 2) / Math.pow(p2 - baseRate, 2);
    
    return Math.ceil(n);
  }

  // Bayesian A/B Testing
  calculateBayesianProbability(controlStats, treatmentStats) {
    // Beta distribution parameters
    const alphaControl = (controlStats.conversions || 0) + 1;
    const betaControl = (controlStats.impressions - controlStats.conversions || 0) + 1;
    const alphaT = (treatmentStats.conversions || 0) + 1;
    const betaT = (treatmentStats.impressions - treatmentStats.conversions || 0) + 1;

    // Monte Carlo simulation
    const SIMULATIONS = 10000;
    let treatmentWins = 0;

    for (let i = 0; i < SIMULATIONS; i++) {
      const controlSample = this.betaSample(alphaControl, betaControl);
      const treatmentSample = this.betaSample(alphaT, betaT);
      
      if (treatmentSample > controlSample) {
        treatmentWins++;
      }
    }

    return treatmentWins / SIMULATIONS;
  }

  // Beta distribution sampler (Box-Muller method approximation)
  betaSample(alpha, beta) {
    // Using Johnk's method
    let x, y;
    do {
      x = Math.pow(Math.random(), 1 / alpha);
      y = Math.pow(Math.random(), 1 / beta);
    } while (x + y > 1);
    
    return x / (x + y);
  }
}

module.exports = StatisticsCalculator;
```

---

## 6. A/B Testing ใน LINE Bot

### 6.1 Message A/B Test

```javascript
// ab-testing/src/LineABTest.js
const ExperimentManager = require('./ExperimentManager');
const Redis = require('ioredis');
const logger = require('./utils/logger');

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
});

const experimentManager = new ExperimentManager(redis);

// ตัวอย่าง: ทดสอบ welcome message
async function getWelcomeMessage(userId) {
  // ดึง variant assignments สำหรับ message experiments
  const assignments = await experimentManager.getVariantForUser(userId, 'message');
  
  // หา welcome message experiment
  const welcomeExp = Object.entries(assignments).find(([expId]) => {
    const exp = experimentManager.activeExperiments.get(expId);
    return exp?.name.includes('welcome');
  });

  if (!welcomeExp) {
    // Default message
    return getDefaultWelcomeMessage();
  }

  const [experimentId, { variantId, config }] = welcomeExp;

  // Track impression
  await experimentManager.trackEvent({
    userId,
    experimentId,
    variantId,
    eventType: 'impression'
  });

  return config?.message || getDefaultWelcomeMessage();
}

function getDefaultWelcomeMessage() {
  return {
    type: 'text',
    text: 'ยินดีต้อนรับ! พิมพ์ "เมนู" เพื่อเริ่มต้นครับ'
  };
}

// Middleware สำหรับ tracking postback
async function trackPostbackClick(event, experimentId) {
  const userId = event.source.userId;
  
  // ดึง variant ของ user
  const assignment = await experimentManager.assigner.getCachedAssignment(userId, experimentId);
  if (!assignment) return;

  await experimentManager.trackEvent({
    userId,
    experimentId,
    variantId: assignment,
    eventType: 'click',
    metadata: {
      action: event.postback.data,
      timestamp: event.timestamp
    }
  });
}

// Track conversion
async function trackConversion(userId, experimentId, metadata = {}) {
  const assignment = await experimentManager.assigner.getCachedAssignment(userId, experimentId);
  if (!assignment) return;

  await experimentManager.trackEvent({
    userId,
    experimentId,
    variantId: assignment,
    eventType: 'conversion',
    metadata
  });
}

module.exports = {
  getWelcomeMessage,
  trackPostbackClick,
  trackConversion,
  experimentManager
};
```

### 6.2 Flex Message A/B Test

```javascript
// examples/flexMessageABTest.js
const { experimentManager } = require('../ab-testing/src/LineABTest');

// สร้าง experiment สำหรับ Flex Message
async function createFlexABTest() {
  const experiment = await experimentManager.createExperiment({
    name: 'product-card-cta-button-test',
    description: 'ทดสอบ CTA button text สำหรับ product card',
    hypothesis: 'การใช้ "สั่งซื้อเลย" แทน "ดูรายละเอียด" จะเพิ่ม conversion rate เพราะตรงกับ intent ของ user มากกว่า',
    type: 'message',
    variants: [
      {
        id: 'control',
        name: 'Control (ดูรายละเอียด)',
        weight: 50,
        config: {
          buttonLabel: 'ดูรายละเอียด',
          buttonStyle: 'secondary',
          buttonColor: '#00B900'
        }
      },
      {
        id: 'treatment-a',
        name: 'Treatment A (สั่งซื้อเลย)',
        weight: 25,
        config: {
          buttonLabel: '🛒 สั่งซื้อเลย',
          buttonStyle: 'primary',
          buttonColor: '#FF4500'
        }
      },
      {
        id: 'treatment-b',
        name: 'Treatment B (เพิ่มลงตะกร้า)',
        weight: 25,
        config: {
          buttonLabel: '+ เพิ่มลงตะกร้า',
          buttonStyle: 'primary',
          buttonColor: '#FF8C00'
        }
      }
    ],
    targetAudience: {
      percentage: 80  // ทดสอบกับ 80% ของ users
    },
    metrics: [
      { name: 'button_click', type: 'click', goal: 'increase' },
      { name: 'purchase', type: 'conversion', goal: 'increase' }
    ],
    minimumSampleSize: 500
  });

  return experiment;
}

// Build Flex message ตาม variant
async function buildProductCard(userId, product, experimentId) {
  const assignments = await experimentManager.getVariantForUser(userId, 'message');
  const assignment = assignments[experimentId];

  // Default config ถ้าไม่มี assignment
  const config = assignment?.config || {
    buttonLabel: 'ดูรายละเอียด',
    buttonStyle: 'secondary',
    buttonColor: '#00B900'
  };

  // Track impression
  if (assignment) {
    await experimentManager.trackEvent({
      userId,
      experimentId,
      variantId: assignment.variantId,
      eventType: 'impression',
      metadata: { productId: product.id }
    });
  }

  return {
    type: 'flex',
    altText: product.name,
    contents: {
      type: 'bubble',
      hero: {
        type: 'image',
        url: product.imageUrl,
        size: 'full',
        aspectRatio: '20:13',
        aspectMode: 'cover'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: product.name,
            weight: 'bold',
            size: 'xl',
            wrap: true
          },
          {
            type: 'text',
            text: `฿${product.price.toLocaleString()}`,
            size: 'lg',
            color: '#00B900',
            weight: 'bold'
          },
          {
            type: 'text',
            text: product.description,
            size: 'sm',
            color: '#666666',
            wrap: true,
            maxLines: 3
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
            // Include experiment info ใน postback data
            data: `action=product_cta&productId=${product.id}&expId=${experimentId}&variantId=${assignment?.variantId || 'none'}`,
            label: config.buttonLabel
          },
          style: config.buttonStyle,
          color: config.buttonColor
        }]
      }
    }
  };
}

// Handle postback จาก product card
async function handleProductCTA(event) {
  const params = new URLSearchParams(event.postback.data);
  const productId = params.get('productId');
  const expId = params.get('expId');
  const variantId = params.get('variantId');
  const userId = event.source.userId;

  // Track click
  if (expId && variantId && variantId !== 'none') {
    await experimentManager.trackEvent({
      userId,
      experimentId: expId,
      variantId,
      eventType: 'click',
      metadata: { productId }
    });
  }

  // Continue with normal flow...
}

module.exports = {
  createFlexABTest,
  buildProductCard,
  handleProductCTA
};
```

---

## 7. Feature Flags System

### 7.1 Feature Flag Manager

```javascript
// feature-flags/src/FeatureFlagManager.js
const Redis = require('ioredis');
const crypto = require('crypto');
const logger = require('./utils/logger');

class FeatureFlagManager {
  constructor(redis) {
    this.redis = redis;
    this.flags = new Map();
    this.cacheTTL = 60; // 1 minute
    
    this.loadFlags();
    setInterval(() => this.loadFlags(), 60 * 1000);
  }

  async loadFlags() {
    try {
      const keys = await this.redis.keys('feature:flag:*');
      
      for (const key of keys) {
        const flag = JSON.parse(await this.redis.get(key));
        const flagName = key.replace('feature:flag:', '');
        this.flags.set(flagName, flag);
      }
      
      logger.debug(`Loaded ${this.flags.size} feature flags`);
    } catch (error) {
      logger.error('Failed to load feature flags:', error);
    }
  }

  // สร้าง feature flag
  async createFlag(name, config) {
    const flag = {
      name,
      description: config.description,
      enabled: config.enabled !== false,
      rolloutPercentage: config.rolloutPercentage || 100,
      allowList: config.allowList || [],    // Users always enabled
      denyList: config.denyList || [],      // Users always disabled
      segments: config.segments || [],      // Enabled for these segments
      conditions: config.conditions || [],  // Custom conditions
      variants: config.variants || null,    // For gradual rollout
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };

    await this.redis.set(`feature:flag:${name}`, JSON.stringify(flag));
    this.flags.set(name, flag);
    
    logger.info(`Created feature flag: ${name}`);
    return flag;
  }

  // ตรวจสอบว่า user ได้รับ feature นี้ไหม
  isEnabled(flagName, userId, userSegments = []) {
    const flag = this.flags.get(flagName);
    
    if (!flag) {
      logger.warn(`Feature flag not found: ${flagName}`);
      return false;
    }

    if (!flag.enabled) return false;

    // Check deny list (สูงสุด)
    if (flag.denyList.includes(userId)) return false;

    // Check allow list
    if (flag.allowList.includes(userId)) return true;

    // Check segment
    if (flag.segments.length > 0) {
      const hasSegment = flag.segments.some(s => userSegments.includes(s));
      if (!hasSegment) return false;
    }

    // Percentage-based rollout
    if (flag.rolloutPercentage < 100) {
      const hash = parseInt(
        crypto.createHash('md5')
          .update(`${userId}-${flagName}`)
          .digest('hex')
          .substring(0, 4),
        16
      );
      const percentile = (hash / 0xFFFF) * 100;
      if (percentile >= flag.rolloutPercentage) return false;
    }

    return true;
  }

  // Gradual rollout
  async gradualRollout(flagName, targetPercentage, stepSize = 10, intervalHours = 24) {
    const flag = this.flags.get(flagName);
    if (!flag) throw new Error(`Flag not found: ${flagName}`);

    const currentPercentage = flag.rolloutPercentage;
    
    if (currentPercentage >= targetPercentage) {
      return { message: 'Already at target percentage' };
    }

    const steps = Math.ceil((targetPercentage - currentPercentage) / stepSize);
    
    logger.info(`Starting gradual rollout for ${flagName}`, {
      from: currentPercentage,
      to: targetPercentage,
      steps,
      intervalHours
    });

    // Schedule steps
    for (let i = 0; i < steps; i++) {
      const newPercentage = Math.min(currentPercentage + (stepSize * (i + 1)), targetPercentage);
      const delay = i * intervalHours * 3600 * 1000;
      
      setTimeout(async () => {
        await this.updateFlag(flagName, { rolloutPercentage: newPercentage });
        logger.info(`Rollout step: ${flagName} now at ${newPercentage}%`);
      }, delay);
    }

    return { 
      steps, 
      scheduledAt: new Date().toISOString(),
      completionEstimate: new Date(Date.now() + steps * intervalHours * 3600 * 1000).toISOString()
    };
  }

  // Update flag
  async updateFlag(flagName, updates) {
    const flag = this.flags.get(flagName);
    if (!flag) throw new Error(`Flag not found: ${flagName}`);

    const updated = { 
      ...flag, 
      ...updates, 
      updatedAt: new Date().toISOString() 
    };
    
    await this.redis.set(`feature:flag:${flagName}`, JSON.stringify(updated));
    this.flags.set(flagName, updated);
    
    return updated;
  }

  // Rollback (disable flag)
  async rollback(flagName, reason) {
    logger.warn(`Rolling back feature flag: ${flagName}`, { reason });
    return this.updateFlag(flagName, { enabled: false, rollbackReason: reason });
  }

  // Get all flags
  getAllFlags() {
    return Array.from(this.flags.entries()).map(([name, flag]) => ({
      name,
      ...flag,
      rolloutPercentage: flag.rolloutPercentage
    }));
  }
}

module.exports = FeatureFlagManager;
```

---

## 8. Rollback Strategy

### 8.1 Automated Rollback

```javascript
// ab-testing/src/AutoRollback.js
const ExperimentManager = require('./ExperimentManager');
const FeatureFlagManager = require('../feature-flags/src/FeatureFlagManager');
const logger = require('./utils/logger');

class AutoRollback {
  constructor(experimentManager, featureFlagManager) {
    this.experimentManager = experimentManager;
    this.featureFlagManager = featureFlagManager;
    this.thresholds = {
      errorRateIncrease: 5,    // เพิ่มขึ้น 5% → rollback
      unfollowRateIncrease: 2, // เพิ่มขึ้น 2% → rollback
      responseTimeIncrease: 500 // เพิ่มขึ้น 500ms → rollback
    };
    
    this.monitoringInterval = null;
  }

  startMonitoring(intervalMs = 5 * 60 * 1000) {
    this.monitoringInterval = setInterval(
      () => this.checkAllExperiments(),
      intervalMs
    );
    logger.info('Auto-rollback monitoring started');
  }

  async checkAllExperiments() {
    const activeExperiments = Array.from(
      this.experimentManager.activeExperiments.values()
    );

    for (const experiment of activeExperiments) {
      await this.checkExperiment(experiment);
    }
  }

  async checkExperiment(experiment) {
    const stats = await this.experimentManager.getCurrentStats(
      experiment.id,
      experiment.variants
    );

    const controlVariant = experiment.variants[0];
    const controlStats = stats[controlVariant.id];

    for (const variant of experiment.variants.slice(1)) {
      const variantStats = stats[variant.id];
      
      const issues = this.detectIssues(controlStats, variantStats);
      
      if (issues.length > 0) {
        logger.warn(`Potential issues in experiment ${experiment.name}`, {
          variant: variant.name,
          issues
        });

        // Auto-rollback ถ้ามีปัญหาร้ายแรง
        const criticalIssues = issues.filter(i => i.severity === 'critical');
        if (criticalIssues.length > 0) {
          await this.rollbackVariant(experiment, variant, criticalIssues);
        }
      }
    }
  }

  detectIssues(controlStats, variantStats) {
    const issues = [];
    
    // Check error rate
    const controlErrorRate = controlStats.errors / Math.max(controlStats.impressions, 1);
    const variantErrorRate = variantStats.errors / Math.max(variantStats.impressions, 1);
    
    if (variantErrorRate - controlErrorRate > this.thresholds.errorRateIncrease / 100) {
      issues.push({
        type: 'error_rate',
        severity: 'critical',
        increase: ((variantErrorRate - controlErrorRate) * 100).toFixed(2)
      });
    }

    // Check unfollow rate
    const controlUnfollowRate = controlStats.unfollows / Math.max(controlStats.impressions, 1);
    const variantUnfollowRate = variantStats.unfollows / Math.max(variantStats.impressions, 1);
    
    if (variantUnfollowRate - controlUnfollowRate > this.thresholds.unfollowRateIncrease / 100) {
      issues.push({
        type: 'unfollow_rate',
        severity: 'critical',
        increase: ((variantUnfollowRate - controlUnfollowRate) * 100).toFixed(2)
      });
    }

    return issues;
  }

  async rollbackVariant(experiment, variant, issues) {
    logger.error(`AUTO-ROLLBACK: Experiment ${experiment.name}, variant ${variant.name}`, {
      issues
    });

    // Pause experiment
    await this.experimentManager.stopExperiment(experiment.id);

    // Disable feature flag if exists
    const flagName = `exp-${experiment.id}`;
    try {
      await this.featureFlagManager.rollback(
        flagName,
        `Auto-rollback: ${issues.map(i => i.type).join(', ')}`
      );
    } catch (e) {
      // Flag might not exist
    }

    // Send alert
    await this.sendRollbackAlert(experiment, variant, issues);
  }

  async sendRollbackAlert(experiment, variant, issues) {
    const alertWebhook = process.env.ADMIN_ALERT_WEBHOOK;
    if (!alertWebhook) return;

    const message = {
      text: `🚨 *AUTO-ROLLBACK TRIGGERED*\n\nExperiment: ${experiment.name}\nVariant: ${variant.name}\n\nIssues:\n${issues.map(i => `- ${i.type}: +${i.increase}%`).join('\n')}`
    };

    try {
      const axios = require('axios');
      await axios.post(alertWebhook, message);
    } catch (e) {
      logger.error('Failed to send rollback alert:', e);
    }
  }

  stopMonitoring() {
    if (this.monitoringInterval) {
      clearInterval(this.monitoringInterval);
    }
  }
}

module.exports = AutoRollback;
```

---

## 9. Results Dashboard

### 9.1 Dashboard API

```javascript
// dashboard/src/experimentDashboard.js
const express = require('express');
const { Experiment, ExperimentAssignment, ExperimentEvent } = require('../models');
const ExperimentManager = require('../ab-testing/src/ExperimentManager');
const StatisticsCalculator = require('../ab-testing/src/StatisticsCalculator');
const Redis = require('ioredis');

const router = express.Router();
const redis = new Redis({ host: process.env.REDIS_HOST });
const experimentManager = new ExperimentManager(redis);
const stats = new StatisticsCalculator();

// List experiments
router.get('/experiments', async (req, res) => {
  const { status, page = 1, limit = 20 } = req.query;
  
  const where = {};
  if (status) where.status = status;

  const { count, rows } = await Experiment.findAndCountAll({
    where,
    limit: parseInt(limit),
    offset: (parseInt(page) - 1) * parseInt(limit),
    order: [['createdAt', 'DESC']]
  });

  res.json({
    experiments: rows,
    total: count,
    page: parseInt(page),
    pages: Math.ceil(count / parseInt(limit))
  });
});

// Get experiment results
router.get('/experiments/:id/results', async (req, res) => {
  const experiment = await Experiment.findByPk(req.params.id);
  if (!experiment) return res.status(404).json({ error: 'Not found' });

  const currentStats = await experimentManager.getCurrentStats(
    experiment.id,
    experiment.variants
  );

  const significance = stats.calculateSignificance(
    currentStats,
    experiment.variants,
    experiment.confidenceLevel
  );

  // Time series data
  const timeSeriesData = await getTimeSeriesData(experiment.id, experiment.variants);

  res.json({
    experiment: experiment.toJSON(),
    stats: currentStats,
    significance,
    timeSeriesData,
    summary: buildSummary(experiment, currentStats, significance)
  });
});

async function getTimeSeriesData(experimentId, variants) {
  const { Op } = require('sequelize');
  
  // ดึงข้อมูลรายวัน
  const events = await ExperimentEvent.findAll({
    where: { experimentId },
    attributes: [
      [ExperimentEvent.sequelize.fn('DATE', ExperimentEvent.sequelize.col('occurredAt')), 'date'],
      'variantId',
      'eventType',
      [ExperimentEvent.sequelize.fn('COUNT', '*'), 'count']
    ],
    group: ['date', 'variantId', 'eventType'],
    order: [['date', 'ASC']],
    raw: true
  });

  // จัดรูปแบบสำหรับ chart
  const organized = {};
  events.forEach(event => {
    if (!organized[event.date]) organized[event.date] = {};
    if (!organized[event.date][event.variantId]) {
      organized[event.date][event.variantId] = {};
    }
    organized[event.date][event.variantId][event.eventType] = parseInt(event.count);
  });

  return Object.entries(organized).map(([date, variantData]) => ({
    date,
    ...Object.entries(variantData).reduce((acc, [variantId, metrics]) => {
      acc[variantId] = metrics;
      return acc;
    }, {})
  }));
}

function buildSummary(experiment, currentStats, significance) {
  const totalParticipants = Object.values(currentStats)
    .reduce((sum, s) => sum + (s.impressions || 0), 0);

  const bestVariant = experiment.variants.reduce((best, v) => {
    const rate = currentStats[v.id]?.conversionRate || 0;
    return rate > (currentStats[best?.id]?.conversionRate || 0) ? v : best;
  }, experiment.variants[0]);

  return {
    status: experiment.status,
    totalParticipants,
    daysRunning: experiment.startedAt 
      ? Math.round((Date.now() - new Date(experiment.startedAt).getTime()) / (1000 * 60 * 60 * 24))
      : 0,
    hasSignificantResult: significance.hasWinner,
    currentWinner: significance.winnerId,
    bestPerformingVariant: bestVariant?.id,
    recommendation: buildRecommendation(experiment, significance)
  };
}

function buildRecommendation(experiment, significance) {
  if (significance.hasWinner) {
    return {
      action: 'conclude',
      message: `มีผลลัพธ์ที่มีนัยสำคัญ แนะนำให้ใช้ variant "${significance.winnerId}" ในการ deploy`
    };
  }

  const minSampleMet = experiment.variants.every(() => true); // simplified
  if (!minSampleMet) {
    return {
      action: 'wait',
      message: 'รอให้ได้ sample size ที่เพียงพอก่อน'
    };
  }

  return {
    action: 'extend',
    message: 'ยังไม่มีผลลัพธ์ที่มีนัยสำคัญ อาจต้องทดสอบต่อหรือปรับ effect size'
  };
}

// Create experiment
router.post('/experiments', async (req, res) => {
  try {
    const experiment = await experimentManager.createExperiment(req.body);
    res.status(201).json(experiment);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Start experiment
router.post('/experiments/:id/start', async (req, res) => {
  try {
    const experiment = await experimentManager.startExperiment(req.params.id);
    res.json(experiment);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Stop experiment
router.post('/experiments/:id/stop', async (req, res) => {
  try {
    const { winnerId } = req.body;
    const result = await experimentManager.stopExperiment(req.params.id, winnerId);
    res.json(result);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

---

## 10. Complete Docker Setup สำหรับ A/B Testing

```yaml
# docker-compose.abtesting.yml
version: '3.9'

services:
  ab-testing-service:
    build:
      context: ./ab-testing
    environment:
      PORT: 3008
      REDIS_HOST: redis
      DATABASE_URL: ${DATABASE_URL}
      NODE_ENV: production
    ports:
      - "3008:3008"
    depends_on:
      - redis
      - postgres
    networks:
      - bot-network

  feature-flags-service:
    build:
      context: ./feature-flags
    environment:
      PORT: 3009
      REDIS_HOST: redis
      NODE_ENV: production
    ports:
      - "3009:3009"
    networks:
      - bot-network

  experiment-worker:
    build:
      context: ./ab-testing
      target: worker
    environment:
      REDIS_HOST: redis
      DATABASE_URL: ${DATABASE_URL}
      WORKER_TYPE: experiment_analyzer
      CHECK_INTERVAL: 300000  # 5 minutes
    networks:
      - bot-network
```

---

## 11. สรุปและ Best Practices

### 11.1 A/B Testing Checklist

```
Before Running Experiment:
  ✅ กำหนด hypothesis ที่ชัดเจน
  ✅ คำนวณ required sample size ก่อน
  ✅ กำหนด success metrics ล่วงหน้า
  ✅ ตั้ง minimum confidence level (95%)
  ✅ ตั้ง rollback criteria ชัดเจน

During Experiment:
  ✅ Monitor daily ไม่ stop early (p-hacking)
  ✅ ตรวจสอบ data quality
  ✅ Check for novelty effect (ผู้ใช้ใหม่ vs เก่า)
  ✅ Track secondary metrics ด้วย

After Experiment:
  ✅ Wait for statistical significance
  ✅ Document results (ทั้ง positive และ negative)
  ✅ Gradual rollout winner
  ✅ Monitor after full rollout
```

### 11.2 Common Pitfalls

```
❌ หยุด experiment เพราะเห็นผลที่ดีเร็วเกินไป (Peeking)
   → รอให้ครบ minimum sample size เสมอ

❌ ทดสอบหลาย variants พร้อมกันโดยไม่ปรับ p-value (Multiple Testing)
   → ใช้ Bonferroni correction หรือ Bayesian approach

❌ ใช้ sample size เล็กเกินไป
   → คำนวณ required sample size ก่อนเริ่มทุกครั้ง

❌ ทดสอบในช่วงเวลาพิเศษ (วันหยุด, โปรโมชั่น)
   → รอให้ traffic เป็นปกติก่อน

❌ ไม่ isolate experiment
   → ผู้ใช้ควรอยู่ใน experiment เดียวเท่านั้นต่อ metric
```

บทถัดไปจะพูดถึง Performance Optimization เพื่อให้ LINE Bot ตอบสนองได้รวดเร็วและรองรับ load สูงได้
