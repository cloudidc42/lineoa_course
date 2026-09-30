# Part 82: Multi-Region Deployment สำหรับ LINE Bots

## บทนำ

การ Deploy LINE Bot ในหลาย Region ช่วยลด latency สำหรับผู้ใช้ทั่วโลก เพิ่มความพร้อมใช้งาน และรองรับ Data Residency Requirements บทนี้จะครอบคลุมการออกแบบและ implement Multi-Region Architecture อย่างสมบูรณ์

---

## 1. LINE API Regions และ Latency

### 1.1 LINE Platform Infrastructure

```
LINE API Endpoints:
┌──────────────────────────────────────────────────────────┐
│  Webhook Endpoint (รับ events จาก LINE)                  │
│  → ไม่มี region selection (LINE เลือกให้)               │
│  → ต้องตอบกลับภายใน 1000ms                              │
│                                                          │
│  Messaging API (ส่ง messages ไปยัง users)                │
│  → api.line.me (Global CDN)                             │
│  → Latency from Thailand: ~50-100ms                     │
│  → Latency from Japan: ~20-50ms                         │
│  → Latency from US: ~150-200ms                          │
│                                                          │
│  LIFF API                                               │
│  → liff.line.me (Global CDN)                           │
└──────────────────────────────────────────────────────────┘
```

### 1.2 Latency Analysis by Region

```
User → LINE Server → Our Server Latency:

ผู้ใช้ในไทย:
├── LINE Server อยู่ที่ Japan: ~50ms
├── Our Server ใน Singapore: +30ms
└── Total round trip: ~80ms ✓

ผู้ใช้ในญี่ปุ่น:
├── LINE Server: ~10ms
├── Our Server ใน Singapore: +80ms  
└── Total round trip: ~90ms ✓

ผู้ใช้ทั่วไป (Optimal):
├── User → LINE: 10-50ms
├── LINE → Webhook Server: 20-100ms
└── Our Processing: 50-200ms
   Total: 80-350ms

Target: < 1000ms (LINE timeout requirement)
Optimal: < 300ms total processing
```

---

## 2. Global Load Balancing Architecture

### 2.1 Multi-Region Architecture Diagram

```
                    ┌─────────────────────────┐
                    │    Cloudflare/Route53    │
                    │    Global Load Balancer  │
                    │  (GeoDNS + Health Check) │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                       │
   ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
   │  Region 1   │        │  Region 2   │        │  Region 3   │
   │ Singapore   │        │   Tokyo     │        │   Mumbai    │
   │ (ap-se-1)   │        │ (ap-ne-1)   │        │ (ap-so-1)   │
   │             │        │             │        │             │
   │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │
   │ │  ALB/   │ │        │ │  ALB/   │ │        │ │  ALB/   │ │
   │ │  NLB    │ │        │ │  NLB    │ │        │ │  NLB    │ │
   │ └────┬────┘ │        │ └────┬────┘ │        │ └────┬────┘ │
   │      │      │        │      │      │        │      │      │
   │ ┌────▼────┐ │        │ ┌────▼────┐ │        │ ┌────▼────┐ │
   │ │  EKS    │ │        │ │  EKS    │ │        │ │  EKS    │ │
   │ │Cluster  │ │        │ │Cluster  │ │        │ │Cluster  │ │
   │ └────┬────┘ │        │ └────┬────┘ │        │ └────┬────┘ │
   │      │      │        │      │      │        │      │      │
   │ ┌────▼────┐ │        │ ┌────▼────┐ │        │ ┌────▼────┐ │
   │ │Aurora   │◄├────────├─┤Aurora   │ │        │ │Aurora   │ │
   │ │Primary  │ │        │ │Replica  │ │        │ │Replica  │ │
   │ └─────────┘ │        │ └─────────┘ │        │ └─────────┘ │
   └─────────────┘        └─────────────┘        └─────────────┘
          │                      │                       │
          └──────────────────────┼──────────────────────┘
                                 │
                          ┌──────▼──────┐
                          │   Kafka     │
                          │  MirrorMaker│
                          │  (Global)   │
                          └─────────────┘
```

### 2.2 Cloudflare Load Balancing Setup

```javascript
// cloudflare-config/load-balancer.js
// Configuration ผ่าน Cloudflare API

const CloudflareAPI = require('cloudflare');

const cf = new CloudflareAPI({
  token: process.env.CLOUDFLARE_API_TOKEN,
});

async function setupLoadBalancer() {
  const zoneId = process.env.CLOUDFLARE_ZONE_ID;
  const accountId = process.env.CLOUDFLARE_ACCOUNT_ID;
  
  // 1. Create Origin Pools สำหรับแต่ละ Region
  const singaporePool = await cf.loadBalancerPools.create({
    account_id: accountId,
    name: 'linebot-singapore',
    origins: [
      {
        name: 'sg-app-1',
        address: 'sg-1.linebot.example.com',
        enabled: true,
        weight: 1,
      },
      {
        name: 'sg-app-2',
        address: 'sg-2.linebot.example.com',
        enabled: true,
        weight: 1,
      },
    ],
    health_checks: {
      path: '/health',
      expected_codes: '200',
      interval: 30,
      timeout: 5,
      retries: 2,
    },
    minimum_origins: 1,
  });
  
  const tokyoPool = await cf.loadBalancerPools.create({
    account_id: accountId,
    name: 'linebot-tokyo',
    origins: [
      {
        name: 'tk-app-1',
        address: 'tk-1.linebot.example.com',
        enabled: true,
        weight: 1,
      },
    ],
    health_checks: {
      path: '/health',
      expected_codes: '200',
      interval: 30,
      timeout: 5,
      retries: 2,
    },
    minimum_origins: 1,
  });
  
  // 2. Create Load Balancer
  const lb = await cf.loadBalancers.create({
    zone_id: zoneId,
    name: 'api.linebot.example.com',
    default_pools: [singaporePool.id, tokyoPool.id],
    fallback_pool: singaporePool.id,
    
    // Proximity Steering - ส่ง traffic ไปยัง Region ที่ใกล้ที่สุด
    steering_policy: 'proximity',
    
    // Session Affinity - user เดิมไปยัง server เดิม
    session_affinity: 'cookie',
    session_affinity_ttl: 1800,
    
    // Adaptive routing
    adaptive_routing: {
      failover_across_pools: true,
    },
    
    rules: [
      // Geo-based routing
      {
        name: 'asia-pacific',
        condition: 'cf.client.geo.continent eq "AS"',
        overrides: {
          default_pools: [singaporePool.id, tokyoPool.id],
          steering_policy: 'geo',
          region_pools: {
            'APAC': [singaporePool.id, tokyoPool.id],
          },
        },
      },
    ],
  });
  
  console.log(`Load Balancer created: ${lb.id}`);
  return lb;
}

module.exports = { setupLoadBalancer };
```

---

## 3. Data Residency Requirements

### 3.1 Thai PDPA Compliance

```
Thai PDPA Data Requirements:
┌──────────────────────────────────────────────────────────┐
│  Personal Data ที่ต้อง comply กับ PDPA                   │
│  ├── ชื่อ-นามสกุล                                       │
│  ├── เบอร์โทรศัพท์                                     │
│  ├── Email address                                       │
│  ├── LINE User ID (ถือว่าเป็น Personal Data)            │
│  ├── LINE Display Name                                  │
│  ├── Profile Picture URL                                │
│  └── ข้อมูลพฤติกรรม (Behavioral Data)                  │
│                                                          │
│  ข้อกำหนด:                                              │
│  ├── ต้องได้รับ consent ก่อน collect                   │
│  ├── บอก purpose of collection                          │
│  ├── ให้ right to access, delete, portability           │
│  ├── Data breach notification ใน 72 ชั่วโมง            │
│  └── Cross-border transfer ต้องมี adequate protection   │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Data Residency Architecture

```javascript
// src/data-residency/DataRouter.js

class DataRouter {
  constructor({ config }) {
    this.regions = {
      TH: {
        dbUrl: process.env.DB_URL_TH,    // Thailand
        region: 'ap-southeast-7',         // AWS Bangkok (ถ้ามี)
        fallback: 'ap-southeast-1',       // Singapore
      },
      JP: {
        dbUrl: process.env.DB_URL_JP,    // Japan
        region: 'ap-northeast-1',
      },
      DEFAULT: {
        dbUrl: process.env.DB_URL_SG,
        region: 'ap-southeast-1',
      },
    };
  }
  
  getRegionForUser(lineUserId, userCountry) {
    // ตรวจสอบ country จาก LINE Profile หรือ IP
    const countryCode = userCountry || this.detectCountry(lineUserId);
    
    switch (countryCode) {
      case 'TH':
        return this.regions.TH;
      case 'JP':
        return this.regions.JP;
      default:
        return this.regions.DEFAULT;
    }
  }
  
  detectCountry(lineUserId) {
    // LINE User ID prefix ไม่ได้ระบุ country
    // ใช้ IP หรือ language preference แทน
    return 'TH'; // Default สำหรับ LINE OA ไทย
  }
  
  async routeDataOperation(userId, operation) {
    const region = this.getRegionForUser(userId);
    const db = this.getDbConnection(region);
    
    return operation(db);
  }
  
  // สำหรับ data ที่ sensitive ต้องอยู่ในไทย
  async storeSensitiveData(userId, data) {
    const thaiDb = this.getDbConnection(this.regions.TH);
    
    // Encrypt ก่อน store
    const encryptedData = await this.encryptionService.encrypt(
      JSON.stringify(data)
    );
    
    await thaiDb.query(
      'INSERT INTO sensitive_data (user_id, data, created_at) VALUES ($1, $2, NOW())',
      [userId, encryptedData]
    );
  }
}

// Consent Management
class ConsentManager {
  constructor({ db }) {
    this.db = db;
  }
  
  async recordConsent(userId, purposes, ipAddress) {
    await this.db.query(`
      INSERT INTO consent_records (
        line_user_id,
        purposes,
        ip_address,
        user_agent,
        consented_at,
        consent_version
      ) VALUES ($1, $2, $3, $4, NOW(), $5)
      ON CONFLICT (line_user_id, consent_version) 
      DO UPDATE SET
        purposes = EXCLUDED.purposes,
        consented_at = NOW(),
        ip_address = EXCLUDED.ip_address
    `, [
      userId,
      JSON.stringify(purposes),
      ipAddress,
      '',
      process.env.CONSENT_VERSION || '1.0',
    ]);
  }
  
  async hasConsent(userId, purpose) {
    const result = await this.db.query(`
      SELECT purposes FROM consent_records 
      WHERE line_user_id = $1 
      AND consent_version = $2
      ORDER BY consented_at DESC 
      LIMIT 1
    `, [userId, process.env.CONSENT_VERSION]);
    
    if (!result.rows.length) return false;
    
    const purposes = result.rows[0].purposes;
    return purposes.includes(purpose) || purposes.includes('all');
  }
  
  async deleteUserData(userId) {
    // Right to be forgotten
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      await client.query('DELETE FROM users WHERE line_user_id = $1', [userId]);
      await client.query('DELETE FROM messages WHERE user_id IN (SELECT id FROM users WHERE line_user_id = $1)', [userId]);
      await client.query('DELETE FROM consent_records WHERE line_user_id = $1', [userId]);
      
      // ใส่ tombstone record
      await client.query(`
        INSERT INTO deleted_users (line_user_id, deleted_at)
        VALUES ($1, NOW())
      `, [userId]);
      
      await client.query('COMMIT');
      
      return { success: true, message: 'User data deleted successfully' };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

module.exports = { DataRouter, ConsentManager };
```

---

## 4. Database Replication Strategy

### 4.1 PostgreSQL Streaming Replication

```
Database Replication Architecture:
┌──────────────────────────────────────────────────────────┐
│  Singapore (Primary)                                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │  PostgreSQL Primary                              │   │
│  │  ├── WAL (Write-Ahead Log) Stream                │   │
│  │  ├── Synchronous Replicas: 1 (Tokyo)             │   │
│  │  └── Asynchronous Replicas: 1 (Mumbai)           │   │
│  └──────────────────────────────────────────────────┘   │
│                │                                         │
│                │ Synchronous Replication                 │
│                ▼                                         │
│  Tokyo (Sync Replica) ─────────── Replica Lag: ~10ms    │
│                │                                         │
│                │ Async Replication                       │
│                ▼                                         │
│  Mumbai (Async Replica) ────────── Replica Lag: ~50ms   │
└──────────────────────────────────────────────────────────┘
```

### 4.2 PostgreSQL Configuration

```bash
# postgresql.conf - Primary Server
listen_addresses = '*'
max_connections = 500
shared_buffers = 4GB              # 25% of RAM
effective_cache_size = 12GB       # 75% of RAM
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 64MB
default_statistics_target = 100
random_page_cost = 1.1            # SSD
effective_io_concurrency = 200    # SSD
min_wal_size = 1GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4

# Replication settings
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
synchronous_standby_names = 'FIRST 1 (tokyo-replica)'
synchronous_commit = on          # Sync to Tokyo

# pg_hba.conf
host    replication     replicator  10.0.0.0/8    scram-sha-256
```

```bash
# postgresql.conf - Replica Server (Tokyo)
hot_standby = on
hot_standby_feedback = on
max_standby_streaming_delay = 30s
max_standby_archive_delay = 60s

# recovery.conf / postgresql.conf (PG12+)
primary_conninfo = 'host=sg-primary.internal port=5432 user=replicator 
                    password=secret application_name=tokyo-replica'
restore_command = ''
recovery_target_timeline = 'latest'
```

### 4.3 Aurora Global Database

```terraform
# terraform/aurora-global/main.tf

resource "aws_rds_global_cluster" "linebot" {
  global_cluster_identifier = "linebot-global"
  engine                    = "aurora-postgresql"
  engine_version           = "15.4"
  database_name            = "linebot"
  
  # ป้องกันการลบโดยบังเอิญ
  deletion_protection = true
  
  # Storage encryption
  storage_encrypted = true
}

# Primary Cluster - Singapore
module "aurora_primary" {
  source = "./modules/aurora-cluster"
  
  providers = {
    aws = aws.singapore
  }
  
  cluster_id                = "linebot-primary"
  global_cluster_identifier = aws_rds_global_cluster.linebot.id
  instance_class           = "db.r6g.2xlarge"
  instance_count           = 2
  is_primary               = true
  
  master_username = var.db_username
  master_password = var.db_password
  
  # Parameter Group
  parameter_group_family = "aurora-postgresql15"
  parameters = [
    {
      name  = "max_connections"
      value = "1000"
    },
    {
      name  = "shared_preload_libraries"
      value = "pg_stat_statements,auto_explain"
    },
    {
      name  = "log_min_duration_statement"
      value = "1000"  # Log slow queries > 1sec
    },
  ]
  
  backup_retention_period = 14
  preferred_backup_window = "01:00-02:00"
  
  tags = {
    Environment = "production"
    Region      = "primary"
  }
}

# Secondary Cluster - Tokyo
module "aurora_secondary_tokyo" {
  source = "./modules/aurora-cluster"
  
  providers = {
    aws = aws.tokyo
  }
  
  cluster_id                = "linebot-secondary-tokyo"
  global_cluster_identifier = aws_rds_global_cluster.linebot.id
  instance_class           = "db.r6g.xlarge"
  instance_count           = 1
  is_primary               = false
  
  tags = {
    Environment = "production"
    Region      = "dr-tokyo"
  }
}

# Managed Failover
resource "aws_rds_cluster_endpoint" "writer" {
  cluster_identifier          = module.aurora_primary.cluster_id
  cluster_endpoint_identifier = "writer"
  custom_endpoint_type        = "ANY"
  
  tags = {
    Purpose = "write-endpoint"
  }
}

resource "aws_rds_cluster_endpoint" "reader" {
  cluster_identifier          = module.aurora_primary.cluster_id
  cluster_endpoint_identifier = "reader"
  custom_endpoint_type        = "READER"
  
  tags = {
    Purpose = "read-endpoint"
  }
}
```

---

## 5. Active-Active vs Active-Passive

### 5.1 Comparison

```
Active-Active vs Active-Passive:

┌──────────────────┬──────────────────┬──────────────────┐
│  Criterion       │  Active-Active   │  Active-Passive  │
├──────────────────┼──────────────────┼──────────────────┤
│  Traffic         │  Both serve      │  Only Primary    │
│                  │  real traffic    │  serves traffic  │
├──────────────────┼──────────────────┼──────────────────┤
│  Failover time   │  Instant (<1s)   │  Minutes (1-5m)  │
│                  │  (no failover)   │                  │
├──────────────────┼──────────────────┼──────────────────┤
│  Cost            │  2x compute      │  1.5x compute    │
│                  │  costs           │  costs           │
├──────────────────┼──────────────────┼──────────────────┤
│  Complexity      │  High            │  Medium          │
│                  │  (conflict       │                  │
│                  │  resolution)     │                  │
├──────────────────┼──────────────────┼──────────────────┤
│  Data            │  Eventual        │  Synchronous     │
│  Consistency     │  Consistency     │  Replication     │
├──────────────────┼──────────────────┼──────────────────┤
│  Use Case        │  Global users,   │  Disaster        │
│                  │  High SLA (99.99)│  Recovery only   │
├──────────────────┼──────────────────┼──────────────────┤
│  LINE Bot        │  เหมาะสำหรับ    │  เหมาะสำหรับ    │
│  Suitability     │  Global LINE OA  │  Thai-only OA    │
└──────────────────┴──────────────────┴──────────────────┘
```

### 5.2 Active-Active Implementation

```javascript
// src/multi-region/ConflictResolver.js

class ConflictResolver {
  // Last-Write-Wins Strategy
  resolveLastWriteWins(localRecord, remoteRecord) {
    const localTime = new Date(localRecord.updatedAt).getTime();
    const remoteTime = new Date(remoteRecord.updatedAt).getTime();
    
    return localTime >= remoteTime ? localRecord : remoteRecord;
  }
  
  // Merge Strategy สำหรับข้อมูลที่ merge ได้
  resolveByMerge(localRecord, remoteRecord) {
    return {
      ...localRecord,
      ...remoteRecord,
      // Merge arrays
      tags: [...new Set([...localRecord.tags, ...remoteRecord.tags])],
      // Take max value
      messageCount: Math.max(localRecord.messageCount, remoteRecord.messageCount),
      // Use latest timestamp
      updatedAt: new Date(Math.max(
        new Date(localRecord.updatedAt).getTime(),
        new Date(remoteRecord.updatedAt).getTime()
      )).toISOString(),
    };
  }
  
  // Vector Clock สำหรับ causality tracking
  compareVectorClocks(clock1, clock2) {
    const regions = new Set([...Object.keys(clock1), ...Object.keys(clock2)]);
    
    let clock1Dominates = true;
    let clock2Dominates = true;
    
    for (const region of regions) {
      const v1 = clock1[region] || 0;
      const v2 = clock2[region] || 0;
      
      if (v1 < v2) clock1Dominates = false;
      if (v2 < v1) clock2Dominates = false;
    }
    
    if (clock1Dominates && !clock2Dominates) return 'clock1_wins';
    if (clock2Dominates && !clock1Dominates) return 'clock2_wins';
    if (!clock1Dominates && !clock2Dominates) return 'concurrent'; // Conflict!
    return 'equal';
  }
}

// Global State Manager
class MultiRegionStateManager {
  constructor({ regions, currentRegion }) {
    this.regions = regions;
    this.currentRegion = currentRegion;
    this.vectorClock = {};
    regions.forEach(r => this.vectorClock[r] = 0);
  }
  
  tick() {
    this.vectorClock[this.currentRegion]++;
    return { ...this.vectorClock };
  }
  
  async writeWithClock(db, table, data) {
    const clock = this.tick();
    
    await db.query(
      `INSERT INTO ${table} (data, vector_clock, region, created_at)
       VALUES ($1, $2, $3, NOW())
       ON CONFLICT (id) DO UPDATE SET
       data = EXCLUDED.data,
       vector_clock = EXCLUDED.vector_clock,
       updated_at = NOW()`,
      [JSON.stringify(data), JSON.stringify(clock), this.currentRegion]
    );
    
    return clock;
  }
  
  async syncWithRemote(remoteRegion) {
    // ดึง events จาก remote region ที่ยังไม่ได้ sync
    const lastSyncClock = this.vectorClock[remoteRegion];
    
    const remoteEvents = await this.fetchRemoteEvents(remoteRegion, lastSyncClock);
    
    for (const event of remoteEvents) {
      await this.applyRemoteEvent(event);
      this.vectorClock[remoteRegion] = Math.max(
        this.vectorClock[remoteRegion],
        event.clock[remoteRegion]
      );
    }
  }
}
```

---

## 6. AWS Multi-Region Setup

### 6.1 Complete AWS Multi-Region Terraform

```terraform
# terraform/aws-multi-region/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "linebot-terraform-state"
    key            = "multi-region/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

locals {
  regions = {
    primary = "ap-southeast-1"  # Singapore
    dr      = "ap-northeast-1"  # Tokyo
  }
  
  common_tags = {
    Project     = "linebot"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# VPC - Primary Region
module "vpc_primary" {
  source = "terraform-aws-modules/vpc/aws"
  
  providers = {
    aws = aws.primary
  }
  
  name = "linebot-vpc-primary"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway     = true
  single_nat_gateway     = false
  one_nat_gateway_per_az = true
  
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = local.common_tags
}

# VPC Peering สำหรับ cross-region communication
resource "aws_vpc_peering_connection" "primary_to_dr" {
  provider = aws.primary
  
  vpc_id        = module.vpc_primary.vpc_id
  peer_vpc_id   = module.vpc_dr.vpc_id
  peer_region   = local.regions.dr
  auto_accept   = false
  
  tags = merge(local.common_tags, {
    Name = "primary-to-dr"
  })
}

resource "aws_vpc_peering_connection_accepter" "dr_accept" {
  provider = aws.dr
  
  vpc_peering_connection_id = aws_vpc_peering_connection.primary_to_dr.id
  auto_accept               = true
}

# EKS Cluster - Primary
module "eks_primary" {
  source = "terraform-aws-modules/eks/aws"
  
  providers = {
    aws = aws.primary
  }
  
  cluster_name    = "linebot-eks-primary"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc_primary.vpc_id
  subnet_ids = module.vpc_primary.private_subnets
  
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true
  
  # Node Groups
  eks_managed_node_groups = {
    main = {
      min_size       = 3
      max_size       = 20
      desired_size   = 5
      instance_types = ["m5.xlarge"]
      
      labels = {
        role = "main"
      }
      
      taints = []
      
      update_config = {
        max_unavailable_percentage = 25
      }
    }
    
    spot = {
      min_size       = 0
      max_size       = 10
      desired_size   = 2
      instance_types = ["m5.xlarge", "m5.2xlarge", "m4.xlarge"]
      capacity_type  = "SPOT"
      
      labels = {
        role = "spot"
      }
      
      taints = [
        {
          key    = "spot"
          value  = "true"
          effect = "NoSchedule"
        }
      ]
    }
  }
  
  tags = local.common_tags
}

# ElastiCache Redis Cluster
resource "aws_elasticache_replication_group" "redis_primary" {
  provider = aws.primary
  
  replication_group_id = "linebot-redis-primary"
  description          = "LINE Bot Redis Cluster"
  
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.large"
  num_cache_clusters   = 3
  
  automatic_failover_enabled = true
  multi_az_enabled          = true
  
  at_rest_encryption_enabled  = true
  transit_encryption_enabled  = true
  auth_token                  = var.redis_auth_token
  
  subnet_group_name  = aws_elasticache_subnet_group.primary.name
  security_group_ids = [aws_security_group.redis_primary.id]
  
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_primary.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }
  
  tags = local.common_tags
}

# Global Redis Replication (Primary → DR)
resource "aws_elasticache_global_replication_group" "linebot" {
  global_replication_group_id_suffix = "linebot"
  primary_replication_group_id       = aws_elasticache_replication_group.redis_primary.id
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_elasticache_replication_group" "redis_dr" {
  provider = aws.dr
  
  replication_group_id        = "linebot-redis-dr"
  description                 = "LINE Bot Redis DR"
  global_replication_group_id = aws_elasticache_global_replication_group.linebot.global_replication_group_id
  
  node_type          = "cache.r6g.large"
  num_cache_clusters = 2
  
  subnet_group_name  = aws_elasticache_subnet_group.dr.name
  security_group_ids = [aws_security_group.redis_dr.id]
  
  tags = local.common_tags
}

# Route53 Health Checks + Failover
resource "aws_route53_health_check" "primary" {
  fqdn                     = module.alb_primary.dns_name
  port                     = 443
  type                     = "HTTPS"
  resource_path            = "/health"
  failure_threshold        = 3
  request_interval         = 10
  measure_latency          = true
  cloudwatch_alarm_region  = local.regions.primary
  
  tags = merge(local.common_tags, {
    Name = "primary-health"
  })
}

resource "aws_cloudwatch_metric_alarm" "primary_health" {
  provider = aws.primary
  
  alarm_name          = "linebot-primary-health"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1.0
  
  dimensions = {
    HealthCheckId = aws_route53_health_check.primary.id
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## 7. GCP Multi-Region Setup

### 7.1 GCP Global Infrastructure

```terraform
# terraform/gcp-multi-region/main.tf

provider "google" {
  project = var.project_id
}

# Global Load Balancer
resource "google_compute_global_address" "linebot_ip" {
  name = "linebot-global-ip"
}

resource "google_compute_global_forwarding_rule" "https" {
  name                  = "linebot-https"
  ip_address            = google_compute_global_address.linebot_ip.address
  port_range            = "443"
  target                = google_compute_target_https_proxy.linebot.id
  load_balancing_scheme = "EXTERNAL"
}

resource "google_compute_backend_service" "linebot" {
  name                  = "linebot-backend"
  protocol              = "HTTPS"
  port_name             = "https"
  timeout_sec           = 30
  load_balancing_scheme = "EXTERNAL"
  
  health_checks = [google_compute_health_check.linebot.id]
  
  # Singapore backend
  backend {
    group           = google_compute_instance_group_manager.singapore.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
    max_utilization = 0.8
  }
  
  # Tokyo backend
  backend {
    group           = google_compute_instance_group_manager.tokyo.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 0.5  # Less capacity for DR
    max_utilization = 0.8
  }
  
  # Log config
  log_config {
    enable      = true
    sample_rate = 0.1
  }
  
  # Session affinity
  session_affinity = "GENERATED_COOKIE"
  affinity_cookie_ttl_sec = 1800
}

# URL Map with path-based routing
resource "google_compute_url_map" "linebot" {
  name            = "linebot-url-map"
  default_service = google_compute_backend_service.linebot.id
  
  host_rule {
    hosts        = ["api.linebot.example.com"]
    path_matcher = "api-paths"
  }
  
  path_matcher {
    name            = "api-paths"
    default_service = google_compute_backend_service.linebot.id
    
    path_rule {
      paths   = ["/webhook"]
      service = google_compute_backend_service.linebot.id
      route_action {
        timeout {
          seconds = 10  # Webhook timeout
        }
      }
    }
  }
}

# Cloud SQL Regional Instances
resource "google_sql_database_instance" "primary" {
  name             = "linebot-db-primary"
  database_version = "POSTGRES_15"
  region           = "asia-southeast1"  # Singapore
  
  deletion_protection = true
  
  settings {
    tier              = "db-custom-8-32768"  # 8 vCPU, 32GB RAM
    availability_type = "REGIONAL"           # Multi-AZ
    
    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 14
      }
      transaction_log_retention_days = 7
    }
    
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.linebot.id
      require_ssl     = true
    }
    
    database_flags {
      name  = "max_connections"
      value = "1000"
    }
    
    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"
    }
    
    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
    }
  }
}

# Cross-Region Read Replica
resource "google_sql_database_instance" "replica_tokyo" {
  name                 = "linebot-db-replica-tokyo"
  database_version     = "POSTGRES_15"
  region               = "asia-northeast1"  # Tokyo
  master_instance_name = google_sql_database_instance.primary.name
  
  settings {
    tier              = "db-custom-4-16384"
    availability_type = "ZONAL"
    
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.linebot_tokyo.id
      require_ssl     = true
    }
  }
}

# GKE Clusters
resource "google_container_cluster" "primary" {
  name     = "linebot-gke-primary"
  location = "asia-southeast1"
  
  # ใช้ node pools แยก
  remove_default_node_pool = true
  initial_node_count       = 1
  
  network    = google_compute_network.linebot.name
  subnetwork = google_compute_subnetwork.private_singapore.name
  
  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
  
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.32/28"
  }
  
  release_channel {
    channel = "STABLE"
  }
  
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  addons_config {
    horizontal_pod_autoscaling {
      disabled = false
    }
    http_load_balancing {
      disabled = false
    }
  }
}

resource "google_container_node_pool" "main" {
  name       = "main-pool"
  location   = "asia-southeast1"
  cluster    = google_container_cluster.primary.name
  
  autoscaling {
    min_node_count = 3
    max_node_count = 20
  }
  
  node_config {
    machine_type = "e2-standard-4"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

---

## 8. Failover Strategies

### 8.1 Automatic Failover Flow

```
Automatic Failover Process:
┌──────────────────────────────────────────────────────────┐
│  1. Health Check Fails (3 consecutive checks)            │
│     └── Primary region unhealthy detected                │
│                                                          │
│  2. DNS Failover (< 30 seconds)                         │
│     └── Route53/Cloudflare switches to DR region         │
│                                                          │
│  3. Application Layer Failover                           │
│     ├── Circuit breaker opens                            │
│     ├── Traffic routing switches                         │
│     └── Read operations switch to replica                │
│                                                          │
│  4. Database Failover (Aurora Global ~ 1 min)            │
│     ├── Promote secondary to primary                     │
│     ├── Update connection strings                         │
│     └── Invalidate connection pools                      │
│                                                          │
│  5. Notification                                         │
│     ├── PagerDuty alert                                  │
│     ├── Slack notification                               │
│     └── Status page update                              │
│                                                          │
│  6. Post-Failover Verification                          │
│     ├── Smoke tests run                                  │
│     ├── Monitor error rates                              │
│     └── Verify data consistency                         │
└──────────────────────────────────────────────────────────┘
```

### 8.2 Failover Controller

```javascript
// src/failover/FailoverController.js
const { EventEmitter } = require('events');

class FailoverController extends EventEmitter {
  constructor({ healthChecker, dnsManager, dbManager, notifier }) {
    super();
    this.healthChecker = healthChecker;
    this.dnsManager = dnsManager;
    this.dbManager = dbManager;
    this.notifier = notifier;
    
    this.state = 'normal';
    this.failoverInProgress = false;
    this.consecutiveFailures = 0;
    this.failoverThreshold = 3;
  }
  
  async startMonitoring(interval = 10000) {
    setInterval(async () => {
      await this.checkHealth();
    }, interval);
    
    console.log('Failover monitoring started');
  }
  
  async checkHealth() {
    try {
      const health = await this.healthChecker.checkPrimary();
      
      if (health.healthy) {
        this.consecutiveFailures = 0;
        
        if (this.state === 'failover') {
          await this.tryFailback();
        }
      } else {
        this.consecutiveFailures++;
        console.warn(`Health check failed (${this.consecutiveFailures}/${this.failoverThreshold})`);
        
        if (this.consecutiveFailures >= this.failoverThreshold && !this.failoverInProgress) {
          await this.initiateFailover(health.reason);
        }
      }
    } catch (error) {
      console.error('Health check error:', error);
      this.consecutiveFailures++;
    }
  }
  
  async initiateFailover(reason) {
    if (this.failoverInProgress) return;
    
    this.failoverInProgress = true;
    const startTime = Date.now();
    
    console.log(`Initiating failover. Reason: ${reason}`);
    
    await this.notifier.sendCritical({
      title: 'FAILOVER INITIATED',
      message: `Primary region failing. Reason: ${reason}`,
      timestamp: new Date().toISOString(),
    });
    
    try {
      // Step 1: DNS Failover
      await this.dnsManager.switchToDR();
      console.log('DNS switched to DR region');
      
      // Step 2: Database Failover
      await this.dbManager.promoteDRToMaster();
      console.log('Database promoted to master');
      
      // Step 3: Update connection pools
      await this.updateConnectionStrings();
      console.log('Connection strings updated');
      
      // Step 4: Verify DR is working
      await this.verifyDRHealth();
      
      this.state = 'failover';
      const duration = Date.now() - startTime;
      
      await this.notifier.send({
        title: 'FAILOVER COMPLETED',
        message: `Failover completed in ${duration}ms`,
        timestamp: new Date().toISOString(),
      });
      
      this.emit('failover-completed', { duration, reason });
      
    } catch (error) {
      console.error('Failover failed:', error);
      await this.notifier.sendCritical({
        title: 'FAILOVER FAILED',
        message: `Failover failed: ${error.message}`,
        timestamp: new Date().toISOString(),
      });
    } finally {
      this.failoverInProgress = false;
    }
  }
  
  async tryFailback() {
    console.log('Checking if primary is ready for failback...');
    
    // ตรวจสอบว่า primary stable แล้ว
    const primaryHealth = await this.healthChecker.checkPrimary();
    
    if (!primaryHealth.healthy) return;
    
    // Sync data กลับ primary
    await this.dbManager.syncToPrimary();
    
    // Switch DNS back
    await this.dnsManager.switchToPrimary();
    
    this.state = 'normal';
    
    await this.notifier.send({
      title: 'FAILBACK COMPLETED',
      message: 'Successfully failed back to primary region',
      timestamp: new Date().toISOString(),
    });
    
    this.emit('failback-completed');
  }
}

module.exports = FailoverController;
```

---

## 9. CDN Integration

### 9.1 CDN Architecture

```
CDN for LINE Bot Assets:
┌──────────────────────────────────────────────────────────┐
│  LINE Message Types ที่ benefit จาก CDN:                 │
│  ├── Image Messages (product photos, banners)            │
│  ├── Video Messages                                      │
│  ├── Rich Menu Background Images                         │
│  ├── LIFF Static Assets (JS, CSS, Images)               │
│  └── Flex Message Images                                 │
└──────────────────────────────────────────────────────────┘
```

```javascript
// src/cdn/AssetManager.js
const AWS = require('aws-sdk');

class AssetManager {
  constructor({ cloudFrontDomain, s3Bucket }) {
    this.cloudFrontDomain = cloudFrontDomain;
    this.s3 = new AWS.S3({ region: 'ap-southeast-1' });
    this.s3Bucket = s3Bucket;
    this.cloudFront = new AWS.CloudFront();
  }
  
  async uploadAsset(buffer, key, contentType) {
    const uploadParams = {
      Bucket: this.s3Bucket,
      Key: key,
      Body: buffer,
      ContentType: contentType,
      CacheControl: 'public, max-age=31536000, immutable',
      Metadata: {
        uploadedAt: new Date().toISOString(),
      },
    };
    
    await this.s3.upload(uploadParams).promise();
    
    return `https://${this.cloudFrontDomain}/${key}`;
  }
  
  async invalidateCache(paths) {
    const params = {
      DistributionId: process.env.CLOUDFRONT_DISTRIBUTION_ID,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths.map(p => (p.startsWith('/') ? p : `/${p}`)),
        },
      },
    };
    
    return this.cloudFront.createInvalidation(params).promise();
  }
  
  getCDNUrl(key) {
    return `https://${this.cloudFrontDomain}/${key}`;
  }
  
  // Resize image for LINE (max 1024x1024 for flex)
  async processImageForLine(buffer, type = 'flex') {
    const sharp = require('sharp');
    
    const maxDimensions = {
      flex: { width: 1024, height: 1024 },
      'rich-menu': { width: 2500, height: 1686 },
      'image-message': { width: 1024, height: 1024 },
    };
    
    const { width, height } = maxDimensions[type] || maxDimensions.flex;
    
    const processed = await sharp(buffer)
      .resize(width, height, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 85, progressive: true })
      .toBuffer();
    
    return processed;
  }
}

module.exports = AssetManager;
```

---

## 10. Testing Multi-Region

### 10.1 Multi-Region Test Suite

```javascript
// tests/multi-region/integration.test.js
const axios = require('axios');

describe('Multi-Region Integration Tests', () => {
  const regions = [
    { name: 'Singapore', url: 'https://sg.api.linebot.example.com' },
    { name: 'Tokyo', url: 'https://tk.api.linebot.example.com' },
  ];
  
  describe('Health Checks', () => {
    regions.forEach(({ name, url }) => {
      it(`${name} region should be healthy`, async () => {
        const response = await axios.get(`${url}/health`, { timeout: 5000 });
        
        expect(response.status).toBe(200);
        expect(response.data.status).toBe('healthy');
      });
    });
  });
  
  describe('Data Consistency', () => {
    it('should read same data from all regions', async () => {
      const testUserId = `test-user-${Date.now()}`;
      
      // Write to primary
      await axios.post(`${regions[0].url}/test/users`, {
        userId: testUserId,
        name: 'Test User',
      });
      
      // Wait for replication
      await new Promise(resolve => setTimeout(resolve, 1000));
      
      // Read from all regions
      const results = await Promise.all(
        regions.map(({ url }) =>
          axios.get(`${url}/test/users/${testUserId}`)
        )
      );
      
      // All regions should have same data
      const primaryData = results[0].data;
      results.forEach(result => {
        expect(result.data.userId).toBe(primaryData.userId);
        expect(result.data.name).toBe(primaryData.name);
      });
    });
  });
  
  describe('Failover Test', () => {
    it('should serve traffic after primary failover', async () => {
      // Simulate primary failure
      await axios.post(`${regions[0].url}/test/simulate-failure`, {
        duration: 30000, // 30 seconds
      });
      
      // Traffic should automatically go to DR
      const globalUrl = 'https://api.linebot.example.com';
      
      let succeeded = false;
      for (let i = 0; i < 10; i++) {
        try {
          const response = await axios.get(`${globalUrl}/health`, { timeout: 5000 });
          if (response.status === 200) {
            succeeded = true;
            break;
          }
        } catch (e) {
          await new Promise(resolve => setTimeout(resolve, 3000));
        }
      }
      
      expect(succeeded).toBe(true);
    });
  });
  
  describe('Latency Tests', () => {
    it('should respond within SLA from each region', async () => {
      const slaMs = 500;
      
      for (const { name, url } of regions) {
        const start = Date.now();
        await axios.get(`${url}/health`);
        const duration = Date.now() - start;
        
        console.log(`${name}: ${duration}ms`);
        expect(duration).toBeLessThan(slaMs);
      }
    });
  });
});
```

### 10.2 Chaos Engineering

```javascript
// tests/chaos/network-partition.js
const { exec } = require('child_process');
const util = require('util');
const execAsync = util.promisify(exec);

class ChaosExperiment {
  async simulateNetworkPartition(targetRegion, duration) {
    console.log(`Starting network partition for ${targetRegion} (${duration}s)`);
    
    // Block traffic to/from target region using iptables
    await execAsync(`
      kubectl exec -n chaos \
      chaos-mesh-controller-manager-xxx \
      -- chaos-mesh-tool network partition \
      --target ${targetRegion} \
      --duration ${duration}
    `);
    
    return new Promise(resolve => setTimeout(resolve, duration * 1000));
  }
  
  async simulateLatency(targetService, latencyMs, duration) {
    // Add artificial latency using tc
    const experiment = {
      apiVersion: 'chaos-mesh.org/v1alpha1',
      kind: 'NetworkChaos',
      metadata: {
        name: `latency-${Date.now()}`,
        namespace: 'linebot',
      },
      spec: {
        action: 'delay',
        mode: 'one',
        selector: {
          namespaces: ['linebot'],
          labelSelectors: {
            app: targetService,
          },
        },
        delay: {
          latency: `${latencyMs}ms`,
          correlation: '25',
          jitter: '10ms',
        },
        duration: `${duration}s`,
      },
    };
    
    await execAsync(`echo '${JSON.stringify(experiment)}' | kubectl apply -f -`);
  }
}
```

---

## 11. Monitoring Multi-Region

```yaml
# monitoring/grafana/multi-region-dashboard.json
{
  "title": "Multi-Region LINE Bot Dashboard",
  "panels": [
    {
      "title": "Request Rate by Region",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (region)",
          "legendFormat": "{{region}}"
        }
      ]
    },
    {
      "title": "P99 Latency by Region",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (region, le))",
          "legendFormat": "{{region}} p99"
        }
      ]
    },
    {
      "title": "DB Replication Lag",
      "type": "gauge",
      "targets": [
        {
          "expr": "pg_replication_lag_seconds",
          "legendFormat": "{{region}}"
        }
      ],
      "thresholds": [
        { "value": 0, "color": "green" },
        { "value": 5, "color": "yellow" },
        { "value": 30, "color": "red" }
      ]
    },
    {
      "title": "Failover Events",
      "type": "logs",
      "targets": [
        {
          "expr": "{job=\"failover-controller\"}"
        }
      ]
    }
  ]
}
```

---

## 12. Cost Analysis Multi-Region

```
Multi-Region Cost Comparison:

┌──────────────────────┬──────────────┬──────────────┬──────────────┐
│  Setup               │  Cost/Month  │  Availability│  Recovery    │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│  Single Region       │   $5,000     │  99.9%       │  RTO: 4hr    │
│                      │              │              │  RPO: 1hr    │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│  Active-Passive      │   $8,000     │  99.95%      │  RTO: 15min  │
│  (Warm Standby)      │   (+60%)     │              │  RPO: 5min   │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│  Active-Active       │   $15,000    │  99.999%     │  RTO: <1min  │
│  (2 Regions)         │   (+200%)    │              │  RPO: ~0     │
├──────────────────────┼──────────────┼──────────────┼──────────────┤
│  Active-Active       │   $25,000    │  99.9999%    │  RTO: ~0     │
│  (3 Regions)         │   (+400%)    │              │  RPO: ~0     │
└──────────────────────┴──────────────┴──────────────┴──────────────┘

Recommendation for LINE OA Thailand:
→ Active-Passive (Warm Standby) ดีที่สุดสำหรับ cost/benefit
→ Primary: Singapore
→ DR: Tokyo (nearest with LINE infrastructure)
→ Monthly cost: ~$8,000-15,000 depending on scale
```

---

## สรุป

Multi-Region Deployment สำหรับ LINE Bot ต้องพิจารณาหลายปัจจัย:

1. **LINE API Architecture**: Webhook รับจาก LINE ได้ทุก Region
2. **Data Residency**: ข้อมูล personal ต้องอยู่ในประเทศไทยตาม PDPA
3. **Consistency Model**: เลือก Active-Active หรือ Active-Passive ตาม requirement
4. **Cost**: Active-Active แพงกว่า 2-4x แต่ให้ availability สูงกว่า
5. **Testing**: ต้อง test failover สม่ำเสมอ (GameDay exercises)

สำหรับ LINE OA ไทยทั่วไป แนะนำ Active-Passive (Warm Standby) เพราะ:
- Cost-effective
- เพียงพอสำหรับ SLA 99.95%
- ง่ายกว่าในการ manage

---

*จบ Part 82: Multi-Region Deployment สำหรับ LINE Bots*
