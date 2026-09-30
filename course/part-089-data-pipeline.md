# Part 89: Data Pipeline for LINE OA Analytics

## สารบัญ

1. [Data Pipeline Overview](#overview)
2. [Data Sources](#data-sources)
3. [ETL Pipeline Design](#etl-design)
4. [Apache Airflow for Orchestration](#airflow)
5. [Data Warehouse Design](#warehouse)
6. [dbt for Transformations](#dbt)
7. [Data Quality Checks](#quality)
8. [Business Intelligence with Superset](#superset)
9. [Data Governance](#governance)
10. [Complete Airflow + LINE DAG](#complete-dag)

---

## 1. Data Pipeline Overview {#overview}

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LINE OA Data Pipeline                             │
│                                                                       │
│  Data Sources:                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │
│  │ LINE Insight │  │   Webhook    │  │    LIFF      │  │  CRM   │  │
│  │     API      │  │   Events     │  │   Events     │  │  Data  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └───┬────┘  │
│         │                 │                  │              │        │
│  ┌──────▼─────────────────▼──────────────────▼──────────────▼────┐  │
│  │                    Data Ingestion Layer                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │  │
│  │  │  API Poller  │  │  Kafka/Event │  │   Batch File Import   │  │  │
│  │  │  (Airflow)   │  │  Streaming   │  │   (S3/MinIO)          │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Data Lake (Raw)                               │  │
│  │  MinIO / S3  →  raw/line_insight/, raw/webhooks/, raw/liff/     │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │           ETL / Transformation (dbt + Airflow)                   │  │
│  │  Staging → Intermediate → Marts                                  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              Data Warehouse (PostgreSQL / ClickHouse)            │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌─────────────┐  │  │
│  │  │  User DW  │  │  Message  │  │  Campaign │  │  Financial  │  │  │
│  │  │           │  │  DW       │  │  DW       │  │  DW         │  │  │
│  │  └───────────┘  └───────────┘  └───────────┘  └─────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              Business Intelligence (Apache Superset)             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Sources {#data-sources}

### LINE Insight API Data Source

```python
# dags/extractors/line_insight_extractor.py
import logging
import requests
from datetime import datetime, timedelta
from typing import Optional, List, Dict, Any
import pandas as pd
from io import StringIO
import json

logger = logging.getLogger(__name__)

class LineInsightExtractor:
    """Extract data from LINE Insight API"""
    
    BASE_URL = "https://api.line.me/v2/bot"
    
    def __init__(self, channel_access_token: str):
        self.token = channel_access_token
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {channel_access_token}",
            "Content-Type": "application/json"
        })
    
    def get_followers_demographics(self, date: str) -> Dict[str, Any]:
        """ดึงข้อมูล demographics ของ followers"""
        response = self.session.get(
            f"{self.BASE_URL}/insight/followers",
            params={"date": date}
        )
        response.raise_for_status()
        return response.json()
    
    def get_message_delivery(self, broadcast_id: str) -> Dict[str, Any]:
        """ดึงข้อมูลการส่ง broadcast message"""
        response = self.session.get(
            f"{self.BASE_URL}/insight/message/delivery/broadcast",
            params={"requestId": broadcast_id}
        )
        response.raise_for_status()
        return response.json()
    
    def get_push_message_statistics(self, start_date: str, end_date: str) -> List[Dict]:
        """ดึงสถิติการส่ง push messages"""
        results = []
        current = datetime.strptime(start_date, "%Y%m%d")
        end = datetime.strptime(end_date, "%Y%m%d")
        
        while current <= end:
            date_str = current.strftime("%Y%m%d")
            try:
                response = self.session.get(
                    f"{self.BASE_URL}/insight/message/delivery/push",
                    params={"date": date_str}
                )
                if response.status_code == 200:
                    data = response.json()
                    data["date"] = date_str
                    results.append(data)
                elif response.status_code == 404:
                    logger.info(f"No data for {date_str}")
                else:
                    response.raise_for_status()
            except Exception as e:
                logger.error(f"Error fetching push stats for {date_str}: {e}")
            
            current += timedelta(days=1)
        
        return results
    
    def get_reply_message_statistics(self, start_date: str, end_date: str) -> List[Dict]:
        """ดึงสถิติการ reply messages"""
        results = []
        current = datetime.strptime(start_date, "%Y%m%d")
        end = datetime.strptime(end_date, "%Y%m%d")
        
        while current <= end:
            date_str = current.strftime("%Y%m%d")
            try:
                response = self.session.get(
                    f"{self.BASE_URL}/insight/message/delivery/reply",
                    params={"date": date_str}
                )
                if response.status_code == 200:
                    data = response.json()
                    data["date"] = date_str
                    results.append(data)
            except Exception as e:
                logger.error(f"Error fetching reply stats for {date_str}: {e}")
            
            current += timedelta(days=1)
        
        return results
    
    def get_followers_count_history(self, start_date: str, end_date: str) -> List[Dict]:
        """ดึงประวัติจำนวน followers"""
        results = []
        current = datetime.strptime(start_date, "%Y%m%d")
        end = datetime.strptime(end_date, "%Y%m%d")
        
        while current <= end:
            date_str = current.strftime("%Y%m%d")
            try:
                response = self.session.get(
                    f"{self.BASE_URL}/insight/followers",
                    params={"date": date_str}
                )
                if response.status_code == 200:
                    data = response.json()
                    results.append({
                        "date": date_str,
                        "followers": data.get("followers", 0),
                        "targetedReaches": data.get("targetedReaches", 0),
                        "blocks": data.get("blocks", 0)
                    })
            except Exception as e:
                logger.error(f"Error fetching followers for {date_str}: {e}")
            
            current += timedelta(days=1)
        
        return results
    
    def get_audience_statistics(self) -> Dict[str, Any]:
        """ดึงข้อมูล audience statistics"""
        response = self.session.get(
            f"{self.BASE_URL}/insight/followers",
            params={"date": datetime.now().strftime("%Y%m%d")}
        )
        response.raise_for_status()
        return response.json()
    
    def to_dataframe(self, data: List[Dict]) -> pd.DataFrame:
        """แปลง list ของ dicts เป็น DataFrame"""
        if not data:
            return pd.DataFrame()
        return pd.DataFrame(data)
```

### Webhook Events Collector

```python
# dags/extractors/webhook_collector.py
import json
import logging
from datetime import datetime
from typing import List, Dict, Any
from pathlib import Path
import boto3
from botocore.config import Config

logger = logging.getLogger(__name__)

class WebhookEventCollector:
    """Collect webhook events from S3/MinIO storage"""
    
    def __init__(self, endpoint_url: str, access_key: str, secret_key: str, bucket: str):
        self.s3 = boto3.client(
            's3',
            endpoint_url=endpoint_url,
            aws_access_key_id=access_key,
            aws_secret_access_key=secret_key,
            config=Config(signature_version='s3v4')
        )
        self.bucket = bucket
    
    def get_events_by_date(
        self, 
        date: str,  # YYYY-MM-DD
        event_types: List[str] = None
    ) -> List[Dict[str, Any]]:
        """ดึง webhook events ตาม date"""
        prefix = f"raw/webhooks/{date}/"
        
        objects = self.list_objects(prefix)
        events = []
        
        for obj in objects:
            content = self.get_object_content(obj['Key'])
            parsed = json.loads(content)
            
            # Filter by event type
            if event_types:
                for event in parsed.get('events', []):
                    if event.get('type') in event_types:
                        event['_source_file'] = obj['Key']
                        event['_extracted_at'] = datetime.utcnow().isoformat()
                        events.append(event)
            else:
                for event in parsed.get('events', []):
                    event['_source_file'] = obj['Key']
                    event['_extracted_at'] = datetime.utcnow().isoformat()
                    events.append(event)
        
        return events
    
    def get_message_events(self, date: str) -> List[Dict]:
        """ดึง message events"""
        return self.get_events_by_date(date, ['message'])
    
    def get_follow_unfollow_events(self, date: str) -> List[Dict]:
        """ดึง follow/unfollow events"""
        return self.get_events_by_date(date, ['follow', 'unfollow'])
    
    def get_postback_events(self, date: str) -> List[Dict]:
        """ดึง postback events"""
        return self.get_events_by_date(date, ['postback'])
    
    def list_objects(self, prefix: str) -> List[Dict]:
        """List objects ใน S3/MinIO"""
        objects = []
        paginator = self.s3.get_paginator('list_objects_v2')
        
        for page in paginator.paginate(Bucket=self.bucket, Prefix=prefix):
            objects.extend(page.get('Contents', []))
        
        return objects
    
    def get_object_content(self, key: str) -> str:
        """ดึง content ของ object"""
        response = self.s3.get_object(Bucket=self.bucket, Key=key)
        return response['Body'].read().decode('utf-8')
    
    def save_events(self, events: List[Dict], output_prefix: str) -> str:
        """บันทึก events ไปยัง S3/MinIO"""
        if not events:
            return ''
        
        key = f"{output_prefix}/{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}.json"
        content = json.dumps(events, ensure_ascii=False, default=str)
        
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=content.encode('utf-8'),
            ContentType='application/json'
        )
        
        logger.info(f"Saved {len(events)} events to s3://{self.bucket}/{key}")
        return key
```

### LIFF Events Tracker

```typescript
// apps/liff/src/lib/event-tracker.ts
// ส่วนนี้อยู่ใน LIFF app
interface LiffEvent {
  eventId: string
  eventType: string
  userId: string
  sessionId: string
  timestamp: string
  page: string
  properties: Record<string, unknown>
  liffContext: {
    version: string
    os: string
    language: string
    isInClient: boolean
  }
}

class LiffEventTracker {
  private buffer: LiffEvent[] = []
  private flushTimer: ReturnType<typeof setTimeout> | null = null
  private readonly BUFFER_SIZE = 20
  private readonly FLUSH_INTERVAL = 30000 // 30 seconds

  constructor(
    private readonly userId: string,
    private readonly sessionId: string,
    private readonly liffContext: LiffEvent['liffContext']
  ) {
    this.startAutoFlush()
  }

  track(eventType: string, properties: Record<string, unknown> = {}): void {
    const event: LiffEvent = {
      eventId: this.generateId(),
      eventType,
      userId: this.userId,
      sessionId: this.sessionId,
      timestamp: new Date().toISOString(),
      page: window.location.pathname,
      properties,
      liffContext: this.liffContext
    }

    this.buffer.push(event)

    if (this.buffer.length >= this.BUFFER_SIZE) {
      this.flush()
    }
  }

  // Standard events
  pageView(pageName: string, properties?: Record<string, unknown>): void {
    this.track('page_view', { page_name: pageName, ...properties })
  }

  productView(productId: string, productName: string, price: number): void {
    this.track('product_view', { product_id: productId, product_name: productName, price })
  }

  addToCart(productId: string, quantity: number, price: number): void {
    this.track('add_to_cart', { product_id: productId, quantity, price, total: quantity * price })
  }

  purchase(orderId: string, total: number, items: Array<{id: string; quantity: number; price: number}>): void {
    this.track('purchase', {
      order_id: orderId,
      total,
      item_count: items.length,
      items: items.map(i => ({ id: i.id, quantity: i.quantity, price: i.price }))
    })
  }

  search(query: string, resultsCount: number): void {
    this.track('search', { query, results_count: resultsCount })
  }

  buttonClick(buttonName: string, context?: string): void {
    this.track('button_click', { button_name: buttonName, context })
  }

  formSubmit(formName: string, success: boolean): void {
    this.track('form_submit', { form_name: formName, success })
  }

  error(errorMessage: string, errorType: string): void {
    this.track('error', { error_message: errorMessage, error_type: errorType })
  }

  async flush(): Promise<void> {
    if (this.buffer.length === 0) return

    const events = [...this.buffer]
    this.buffer = []

    try {
      await fetch('/api/analytics/events', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ events }),
        keepalive: true
      })
    } catch (error) {
      // Put back to buffer on failure
      this.buffer = [...events, ...this.buffer]
      console.error('Failed to flush events:', error)
    }
  }

  private startAutoFlush(): void {
    this.flushTimer = setInterval(() => this.flush(), this.FLUSH_INTERVAL)

    // Flush on page unload
    window.addEventListener('pagehide', () => this.flush())
    window.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.flush()
    })
  }

  private generateId(): string {
    return `evt_${Date.now()}_${Math.random().toString(36).slice(2, 9)}`
  }

  destroy(): void {
    if (this.flushTimer) clearInterval(this.flushTimer)
    this.flush()
  }
}
```

---

## 3. ETL Pipeline Design {#etl-design}

### ETL Pipeline Architecture

```
Extract Phase:
  LINE Insight API → JSON files in S3 (raw/)
  Webhook Events  → JSON files in S3 (raw/)
  LIFF Events     → JSON files in S3 (raw/)
  CRM Data        → PostgreSQL snapshot → CSV in S3 (raw/)

Transform Phase (dbt):
  raw/ → staging/ (clean, typed data)
  staging/ → intermediate/ (business logic)
  intermediate/ → marts/ (analytics-ready)

Load Phase:
  marts/ → Data Warehouse (ClickHouse/PostgreSQL)
  Data Warehouse → Apache Superset Dashboards
```

### Extract Functions

```python
# dags/etl/extract.py
import logging
import json
from datetime import datetime, timedelta
from typing import Dict, Any, List
import pandas as pd

logger = logging.getLogger(__name__)

def extract_line_insight(
    token: str,
    date: str,
    s3_client,
    bucket: str
) -> str:
    """Extract LINE Insight data และบันทึกไปยัง S3"""
    from extractors.line_insight_extractor import LineInsightExtractor
    
    extractor = LineInsightExtractor(token)
    
    data = {
        "date": date,
        "followers": extractor.get_followers_count_history(date, date),
        "push_stats": extractor.get_push_message_statistics(date, date),
        "reply_stats": extractor.get_reply_message_statistics(date, date),
        "extracted_at": datetime.utcnow().isoformat()
    }
    
    key = f"raw/line_insight/{date}/data.json"
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=json.dumps(data, ensure_ascii=False).encode('utf-8'),
        ContentType='application/json'
    )
    
    logger.info(f"Extracted LINE Insight data for {date} to {key}")
    return key

def extract_webhook_events(
    date: str,
    s3_client,
    bucket: str
) -> Dict[str, int]:
    """Extract and consolidate webhook events"""
    from extractors.webhook_collector import WebhookEventCollector
    
    collector = WebhookEventCollector(
        endpoint_url=None,  # Use AWS
        access_key=None,    # Use IAM
        secret_key=None,
        bucket=bucket
    )
    
    event_types = ['message', 'follow', 'unfollow', 'postback', 'beacon']
    counts = {}
    
    for event_type in event_types:
        events = collector.get_events_by_date(date, [event_type])
        key = collector.save_events(events, f"raw/webhooks/{date}/{event_type}")
        counts[event_type] = len(events)
        logger.info(f"Extracted {len(events)} {event_type} events for {date}")
    
    return counts

def extract_liff_events(
    date: str,
    db_connection,
    s3_client,
    bucket: str
) -> int:
    """Extract LIFF events from application database"""
    query = """
        SELECT 
            event_id,
            event_type,
            user_id,
            session_id,
            timestamp,
            page,
            properties::text as properties,
            liff_context::text as liff_context
        FROM liff_events
        WHERE DATE(timestamp) = %(date)s
        ORDER BY timestamp
    """
    
    df = pd.read_sql(query, db_connection, params={'date': date})
    
    if df.empty:
        logger.info(f"No LIFF events for {date}")
        return 0
    
    # บันทึกเป็น Parquet สำหรับ efficiency
    import io
    buffer = io.BytesIO()
    df.to_parquet(buffer, index=False, engine='pyarrow')
    
    key = f"raw/liff_events/{date}/events.parquet"
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body=buffer.getvalue(),
        ContentType='application/octet-stream'
    )
    
    logger.info(f"Extracted {len(df)} LIFF events for {date}")
    return len(df)

def extract_crm_data(
    date: str,
    db_connection,
    s3_client,
    bucket: str
) -> Dict[str, int]:
    """Extract CRM data snapshot"""
    tables = {
        'contacts': """
            SELECT * FROM contacts 
            WHERE DATE(updated_at) = %(date)s
        """,
        'deals': """
            SELECT * FROM deals
            WHERE DATE(updated_at) = %(date)s
        """,
        'activities': """
            SELECT * FROM activities
            WHERE DATE(created_at) = %(date)s
        """
    }
    
    counts = {}
    
    for table, query in tables.items():
        df = pd.read_sql(query, db_connection, params={'date': date})
        
        if df.empty:
            counts[table] = 0
            continue
        
        import io
        buffer = io.BytesIO()
        df.to_parquet(buffer, index=False)
        
        key = f"raw/crm/{date}/{table}.parquet"
        s3_client.put_object(
            Bucket=bucket,
            Key=key,
            Body=buffer.getvalue()
        )
        
        counts[table] = len(df)
    
    return counts
```

---

## 4. Apache Airflow for Orchestration {#airflow}

### Installation

```bash
# docker-compose.yml สำหรับ Airflow
version: '3.9'

x-airflow-common: &airflow-common
  image: apache/airflow:2.9.1
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
    AIRFLOW__WEBSERVER__EXPOSE_CONFIG: 'true'
    # LINE credentials
    LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
    LINE_CHANNEL_SECRET: ${LINE_CHANNEL_SECRET}
    # Database
    APP_DATABASE_URL: ${APP_DATABASE_URL}
    # S3/MinIO
    AWS_ACCESS_KEY_ID: ${MINIO_ACCESS_KEY}
    AWS_SECRET_ACCESS_KEY: ${MINIO_SECRET_KEY}
    AWS_ENDPOINT_URL: http://minio:9000
    DATA_BUCKET: line-analytics
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
    - ./requirements.txt:/requirements.txt
  depends_on: &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5

  airflow-worker:
    <<: *airflow-common
    command: celery worker
    environment:
      <<: *airflow-common-env
      DUMB_INIT_SETSID: "0"
    healthcheck:
      test: ["CMD-SHELL", 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"']
      interval: 30s
      timeout: 10s
      retries: 5

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        airflow db init
        airflow users create \
          --username admin \
          --firstname Admin \
          --lastname Admin \
          --role Admin \
          --email admin@example.com \
          --password admin

  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]

volumes:
  postgres-db-volume:
  minio-data:
```

---

## 5. Data Warehouse Design {#warehouse}

### Star Schema Design

```sql
-- dw/schema.sql
-- Dimension Tables

-- dim_date: วันที่
CREATE TABLE dim_date (
    date_key        INTEGER PRIMARY KEY,
    full_date       DATE NOT NULL,
    year            SMALLINT NOT NULL,
    quarter         SMALLINT NOT NULL,
    month           SMALLINT NOT NULL,
    month_name      VARCHAR(20) NOT NULL,
    week            SMALLINT NOT NULL,
    day_of_month    SMALLINT NOT NULL,
    day_of_week     SMALLINT NOT NULL,
    day_name        VARCHAR(20) NOT NULL,
    is_weekend      BOOLEAN NOT NULL,
    is_holiday      BOOLEAN NOT NULL DEFAULT FALSE,
    fiscal_year     SMALLINT,
    fiscal_quarter  SMALLINT
);

-- dim_user: ผู้ใช้ LINE
CREATE TABLE dim_user (
    user_key        BIGSERIAL PRIMARY KEY,
    line_user_id    VARCHAR(50) NOT NULL,
    display_name    VARCHAR(255),
    picture_url     TEXT,
    language        VARCHAR(10),
    country         VARCHAR(5),
    os_type         VARCHAR(20),
    is_follower     BOOLEAN DEFAULT TRUE,
    follow_date     DATE,
    unfollow_date   DATE,
    demographic_age_range   VARCHAR(20),
    demographic_gender      VARCHAR(10),
    valid_from      TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to        TIMESTAMP,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    
    UNIQUE(line_user_id, valid_from)
);

-- dim_message_type: ประเภทข้อความ
CREATE TABLE dim_message_type (
    message_type_key    SMALLSERIAL PRIMARY KEY,
    type_code           VARCHAR(50) NOT NULL UNIQUE,
    type_name           VARCHAR(100) NOT NULL,
    type_category       VARCHAR(50) NOT NULL,
    requires_media      BOOLEAN DEFAULT FALSE
);

-- dim_channel: ช่องทาง (User type ใน LINE)
CREATE TABLE dim_channel (
    channel_key     SMALLSERIAL PRIMARY KEY,
    channel_code    VARCHAR(20) NOT NULL UNIQUE,
    channel_name    VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL
);

-- dim_campaign: แคมเปญ
CREATE TABLE dim_campaign (
    campaign_key        BIGSERIAL PRIMARY KEY,
    campaign_id         VARCHAR(100) NOT NULL,
    campaign_name       VARCHAR(500) NOT NULL,
    campaign_type       VARCHAR(50) NOT NULL,
    target_audience     VARCHAR(500),
    budget              DECIMAL(12, 2),
    start_date          DATE,
    end_date            DATE,
    created_by          VARCHAR(255),
    status              VARCHAR(20) DEFAULT 'active',
    
    UNIQUE(campaign_id)
);

-- Fact Tables

-- fact_messages: ข้อเท็จจริงเกี่ยวกับ messages
CREATE TABLE fact_messages (
    message_key         BIGSERIAL PRIMARY KEY,
    date_key            INTEGER NOT NULL REFERENCES dim_date(date_key),
    user_key            BIGINT REFERENCES dim_user(user_key),
    message_type_key    SMALLINT REFERENCES dim_message_type(message_type_key),
    channel_key         SMALLINT REFERENCES dim_channel(channel_key),
    
    -- Measures
    message_count       INTEGER NOT NULL DEFAULT 1,
    char_count          INTEGER DEFAULT 0,
    has_media           BOOLEAN DEFAULT FALSE,
    
    -- Metadata
    message_id          VARCHAR(100),
    conversation_id     VARCHAR(100),
    reply_token         VARCHAR(200),
    timestamp_utc       TIMESTAMP NOT NULL,
    
    -- Indexes
    CONSTRAINT fk_date FOREIGN KEY (date_key) REFERENCES dim_date(date_key)
);

CREATE INDEX idx_fact_messages_date ON fact_messages(date_key);
CREATE INDEX idx_fact_messages_user ON fact_messages(user_key);
CREATE INDEX idx_fact_messages_timestamp ON fact_messages(timestamp_utc);

-- fact_followers: จำนวน followers รายวัน
CREATE TABLE fact_followers (
    follower_key        BIGSERIAL PRIMARY KEY,
    date_key            INTEGER NOT NULL REFERENCES dim_date(date_key),
    
    -- Measures
    total_followers     INTEGER NOT NULL DEFAULT 0,
    new_followers       INTEGER NOT NULL DEFAULT 0,
    unfollowers         INTEGER NOT NULL DEFAULT 0,
    net_change          INTEGER GENERATED ALWAYS AS (new_followers - unfollowers) STORED,
    blocked             INTEGER NOT NULL DEFAULT 0,
    targeted_reaches    INTEGER NOT NULL DEFAULT 0
);

-- fact_broadcast_performance: ประสิทธิภาพ broadcast messages
CREATE TABLE fact_broadcast_performance (
    broadcast_key       BIGSERIAL PRIMARY KEY,
    date_key            INTEGER NOT NULL REFERENCES dim_date(date_key),
    campaign_key        BIGINT REFERENCES dim_campaign(campaign_key),
    
    -- Measures
    sent_count          INTEGER NOT NULL DEFAULT 0,
    delivered_count     INTEGER NOT NULL DEFAULT 0,
    read_count          INTEGER NOT NULL DEFAULT 0,
    click_count         INTEGER NOT NULL DEFAULT 0,
    conversion_count    INTEGER NOT NULL DEFAULT 0,
    
    -- Rates (calculated)
    delivery_rate       DECIMAL(5, 4) GENERATED ALWAYS AS (
        CASE WHEN sent_count > 0 THEN delivered_count::DECIMAL / sent_count ELSE 0 END
    ) STORED,
    read_rate           DECIMAL(5, 4) GENERATED ALWAYS AS (
        CASE WHEN delivered_count > 0 THEN read_count::DECIMAL / delivered_count ELSE 0 END
    ) STORED,
    click_rate          DECIMAL(5, 4) GENERATED ALWAYS AS (
        CASE WHEN read_count > 0 THEN click_count::DECIMAL / read_count ELSE 0 END
    ) STORED,
    
    -- Financial
    cost                DECIMAL(12, 2) DEFAULT 0,
    revenue             DECIMAL(12, 2) DEFAULT 0,
    
    -- Metadata
    broadcast_id        VARCHAR(100),
    message_type        VARCHAR(50)
);

-- fact_liff_events: LIFF App events
CREATE TABLE fact_liff_events (
    event_key           BIGSERIAL PRIMARY KEY,
    date_key            INTEGER NOT NULL REFERENCES dim_date(date_key),
    user_key            BIGINT REFERENCES dim_user(user_key),
    
    -- Measures
    session_count       INTEGER DEFAULT 1,
    page_views          INTEGER DEFAULT 0,
    add_to_cart_count   INTEGER DEFAULT 0,
    purchase_count      INTEGER DEFAULT 0,
    purchase_value      DECIMAL(12, 2) DEFAULT 0,
    
    -- Metadata
    event_type          VARCHAR(100) NOT NULL,
    page_path           VARCHAR(500),
    session_id          VARCHAR(100),
    os_type             VARCHAR(20),
    is_in_client        BOOLEAN DEFAULT TRUE,
    timestamp_utc       TIMESTAMP NOT NULL
);

-- Populate dim_date
INSERT INTO dim_date
SELECT 
    TO_CHAR(datum, 'YYYYMMDD')::INTEGER AS date_key,
    datum AS full_date,
    EXTRACT(YEAR FROM datum) AS year,
    EXTRACT(QUARTER FROM datum) AS quarter,
    EXTRACT(MONTH FROM datum) AS month,
    TO_CHAR(datum, 'Month') AS month_name,
    EXTRACT(WEEK FROM datum) AS week,
    EXTRACT(DAY FROM datum) AS day_of_month,
    EXTRACT(DOW FROM datum) AS day_of_week,
    TO_CHAR(datum, 'Day') AS day_name,
    CASE WHEN EXTRACT(DOW FROM datum) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend,
    FALSE AS is_holiday
FROM (
    SELECT '2020-01-01'::DATE + SEQUENCE.DAY AS datum
    FROM GENERATE_SERIES(0, 3650) AS SEQUENCE(DAY)
) AS DQ
ORDER BY 1;
```

---

## 6. dbt for Transformations {#dbt}

### dbt Project Structure

```
dbt_project/
├── dbt_project.yml
├── profiles.yml
├── models/
│   ├── staging/
│   │   ├── stg_line_insight.sql
│   │   ├── stg_webhook_events.sql
│   │   ├── stg_liff_events.sql
│   │   └── sources.yml
│   ├── intermediate/
│   │   ├── int_user_activity.sql
│   │   ├── int_message_stats.sql
│   │   └── int_campaign_performance.sql
│   └── marts/
│       ├── core/
│       │   ├── dim_user.sql
│       │   ├── dim_date.sql
│       │   └── fct_messages.sql
│       └── analytics/
│           ├── agg_daily_followers.sql
│           ├── agg_message_performance.sql
│           └── agg_user_engagement.sql
├── tests/
│   ├── generic/
│   └── singular/
├── macros/
│   ├── date_utils.sql
│   └── text_utils.sql
└── seeds/
    ├── message_types.csv
    └── holiday_calendar.csv
```

### dbt Models

```sql
-- models/staging/stg_line_insight.sql
{{ config(
    materialized='incremental',
    unique_key='date_key',
    on_schema_change='sync_all_columns'
) }}

WITH raw_data AS (
    SELECT
        date::VARCHAR AS date_str,
        (followers->0->>'followers')::INTEGER AS total_followers,
        (followers->0->>'targetedReaches')::INTEGER AS targeted_reaches,
        (followers->0->>'blocks')::INTEGER AS blocks,
        extracted_at::TIMESTAMP AS extracted_at
    FROM {{ source('raw', 'line_insight') }}
    {% if is_incremental() %}
    WHERE extracted_at > (SELECT MAX(extracted_at) FROM {{ this }})
    {% endif %}
)

SELECT
    TO_CHAR(TO_DATE(date_str, 'YYYYMMDD'), 'YYYYMMDD')::INTEGER AS date_key,
    TO_DATE(date_str, 'YYYYMMDD') AS full_date,
    total_followers,
    targeted_reaches,
    blocks,
    COALESCE(total_followers, 0) - LAG(COALESCE(total_followers, 0)) 
        OVER (ORDER BY date_str) AS daily_change,
    extracted_at
FROM raw_data
WHERE date_str IS NOT NULL
```

```sql
-- models/staging/stg_webhook_events.sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    partition_by={'field': 'event_date', 'data_type': 'date'}
) }}

WITH raw_events AS (
    SELECT
        (event->>'id')::VARCHAR AS event_id,
        (event->>'type')::VARCHAR AS event_type,
        (event->'source'->>'userId')::VARCHAR AS line_user_id,
        (event->'source'->>'type')::VARCHAR AS source_type,
        TO_TIMESTAMP(
            (event->>'timestamp')::BIGINT / 1000.0
        ) AS event_timestamp,
        event->'message' AS message_data,
        event->'postback' AS postback_data,
        event->>'replyToken' AS reply_token,
        _extracted_at::TIMESTAMP AS extracted_at
    FROM {{ source('raw', 'webhook_events') }},
    LATERAL JSONB_ARRAY_ELEMENTS(events::JSONB) AS event
    {% if is_incremental() %}
    WHERE _extracted_at > (SELECT MAX(extracted_at) FROM {{ this }})
    {% endif %}
)

SELECT
    event_id,
    event_type,
    line_user_id,
    source_type,
    event_timestamp,
    event_timestamp::DATE AS event_date,
    CASE 
        WHEN event_type = 'message' THEN message_data->>'type'
        ELSE NULL
    END AS message_type,
    CASE 
        WHEN event_type = 'message' AND message_data->>'type' = 'text'
        THEN message_data->>'text'
        ELSE NULL
    END AS message_text,
    CASE 
        WHEN event_type = 'postback' 
        THEN postback_data->>'data'
        ELSE NULL
    END AS postback_data,
    reply_token,
    extracted_at
FROM raw_events
WHERE event_id IS NOT NULL
```

```sql
-- models/intermediate/int_user_activity.sql
{{ config(materialized='table') }}

WITH message_events AS (
    SELECT
        line_user_id,
        event_date,
        COUNT(*) AS message_count,
        COUNT(CASE WHEN message_type = 'text' THEN 1 END) AS text_count,
        COUNT(CASE WHEN message_type = 'image' THEN 1 END) AS image_count,
        COUNT(CASE WHEN message_type = 'sticker' THEN 1 END) AS sticker_count,
        MIN(event_timestamp) AS first_message_at,
        MAX(event_timestamp) AS last_message_at
    FROM {{ ref('stg_webhook_events') }}
    WHERE event_type = 'message'
    GROUP BY line_user_id, event_date
),

postback_events AS (
    SELECT
        line_user_id,
        event_date,
        COUNT(*) AS postback_count
    FROM {{ ref('stg_webhook_events') }}
    WHERE event_type = 'postback'
    GROUP BY line_user_id, event_date
),

follow_events AS (
    SELECT
        line_user_id,
        event_date,
        COUNT(CASE WHEN event_type = 'follow' THEN 1 END) AS follow_count,
        COUNT(CASE WHEN event_type = 'unfollow' THEN 1 END) AS unfollow_count
    FROM {{ ref('stg_webhook_events') }}
    WHERE event_type IN ('follow', 'unfollow')
    GROUP BY line_user_id, event_date
)

SELECT
    COALESCE(m.line_user_id, p.line_user_id, f.line_user_id) AS line_user_id,
    COALESCE(m.event_date, p.event_date, f.event_date) AS activity_date,
    COALESCE(m.message_count, 0) AS message_count,
    COALESCE(m.text_count, 0) AS text_count,
    COALESCE(m.image_count, 0) AS image_count,
    COALESCE(m.sticker_count, 0) AS sticker_count,
    COALESCE(p.postback_count, 0) AS postback_count,
    COALESCE(f.follow_count, 0) AS follow_count,
    COALESCE(f.unfollow_count, 0) AS unfollow_count,
    m.first_message_at,
    m.last_message_at,
    COALESCE(m.message_count, 0) + COALESCE(p.postback_count, 0) AS total_interactions
FROM message_events m
FULL OUTER JOIN postback_events p 
    ON m.line_user_id = p.line_user_id AND m.event_date = p.event_date
FULL OUTER JOIN follow_events f 
    ON COALESCE(m.line_user_id, p.line_user_id) = f.line_user_id 
    AND COALESCE(m.event_date, p.event_date) = f.event_date
```

```sql
-- models/marts/analytics/agg_daily_followers.sql
{{ config(
    materialized='incremental',
    unique_key='date_key'
) }}

WITH followers AS (
    SELECT * FROM {{ ref('stg_line_insight') }}
),

daily_activity AS (
    SELECT 
        event_date,
        COUNT(DISTINCT line_user_id) AS active_users,
        SUM(follow_count) AS new_followers,
        SUM(unfollow_count) AS unfollowers
    FROM {{ ref('int_user_activity') }}
    GROUP BY event_date
)

SELECT
    f.date_key,
    f.full_date,
    f.total_followers,
    f.targeted_reaches,
    f.blocks,
    f.daily_change,
    COALESCE(d.active_users, 0) AS active_users,
    COALESCE(d.new_followers, 0) AS new_followers_via_webhook,
    COALESCE(d.unfollowers, 0) AS unfollowers_via_webhook,
    
    -- Engagement rate
    CASE 
        WHEN f.total_followers > 0 
        THEN COALESCE(d.active_users, 0)::DECIMAL / f.total_followers 
        ELSE 0 
    END AS engagement_rate,
    
    NOW() AS dbt_updated_at
FROM followers f
LEFT JOIN daily_activity d ON f.full_date = d.event_date

{% if is_incremental() %}
WHERE f.full_date >= (SELECT MAX(full_date) - INTERVAL '3 days' FROM {{ this }})
{% endif %}
```

---

## 7. Data Quality Checks {#quality}

### dbt Tests

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: raw
    database: analytics
    schema: raw
    tables:
      - name: line_insight
        columns:
          - name: date
            tests:
              - not_null
              - unique
      - name: webhook_events
        columns:
          - name: _extracted_at
            tests:
              - not_null

# models/staging/schema.yml
version: 2

models:
  - name: stg_webhook_events
    description: "Staged webhook events from LINE"
    columns:
      - name: event_id
        tests:
          - not_null
          - unique
      - name: event_type
        tests:
          - not_null
          - accepted_values:
              values: ['message', 'follow', 'unfollow', 'postback', 'beacon', 'join', 'leave']
      - name: event_date
        tests:
          - not_null
      - name: line_user_id
        tests:
          - not_null
  
  - name: stg_line_insight
    description: "LINE Insight daily metrics"
    columns:
      - name: date_key
        tests:
          - not_null
          - unique
      - name: total_followers
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
```

### Custom Data Quality Checks

```python
# dags/quality/data_quality_checks.py
import logging
from typing import List, Dict, Any
from dataclasses import dataclass
from datetime import datetime

logger = logging.getLogger(__name__)

@dataclass
class QualityCheckResult:
    check_name: str
    passed: bool
    severity: str  # error, warning, info
    details: str
    row_count: int = 0
    fail_count: int = 0

class DataQualityChecker:
    def __init__(self, db_connection):
        self.conn = db_connection
    
    def check_completeness(self, table: str, required_columns: List[str]) -> List[QualityCheckResult]:
        """ตรวจสอบความครบถ้วนของข้อมูล"""
        results = []
        
        for col in required_columns:
            query = f"""
                SELECT 
                    COUNT(*) as total,
                    COUNT({col}) as non_null,
                    COUNT(*) - COUNT({col}) as null_count
                FROM {table}
            """
            
            import pandas as pd
            df = pd.read_sql(query, self.conn)
            
            null_rate = df['null_count'][0] / df['total'][0] if df['total'][0] > 0 else 0
            
            results.append(QualityCheckResult(
                check_name=f"completeness_{table}_{col}",
                passed=null_rate < 0.01,  # Allow < 1% nulls
                severity='error' if null_rate > 0.05 else 'warning',
                details=f"Null rate: {null_rate:.2%} ({df['null_count'][0]} of {df['total'][0]})",
                row_count=int(df['total'][0]),
                fail_count=int(df['null_count'][0])
            ))
        
        return results
    
    def check_freshness(self, table: str, timestamp_column: str, max_hours: int = 25) -> QualityCheckResult:
        """ตรวจสอบความสดใหม่ของข้อมูล"""
        query = f"""
            SELECT 
                MAX({timestamp_column}) as latest,
                EXTRACT(EPOCH FROM (NOW() - MAX({timestamp_column}))) / 3600 as hours_since_update
            FROM {table}
        """
        
        import pandas as pd
        df = pd.read_sql(query, self.conn)
        
        hours = df['hours_since_update'][0] or 999
        latest = df['latest'][0]
        
        return QualityCheckResult(
            check_name=f"freshness_{table}",
            passed=hours <= max_hours,
            severity='error' if hours > max_hours * 2 else 'warning',
            details=f"Latest record: {latest}, {hours:.1f} hours ago (max: {max_hours}h)"
        )
    
    def check_row_count_anomaly(
        self, 
        table: str, 
        date_column: str,
        check_date: str,
        expected_min: int,
        expected_max: int
    ) -> QualityCheckResult:
        """ตรวจสอบจำนวน rows ที่ผิดปกติ"""
        query = f"""
            SELECT COUNT(*) as row_count
            FROM {table}
            WHERE DATE({date_column}) = '{check_date}'
        """
        
        import pandas as pd
        df = pd.read_sql(query, self.conn)
        count = int(df['row_count'][0])
        
        return QualityCheckResult(
            check_name=f"row_count_{table}_{check_date}",
            passed=expected_min <= count <= expected_max,
            severity='warning',
            details=f"Row count: {count} (expected {expected_min}-{expected_max})",
            row_count=count
        )
    
    def check_duplicates(self, table: str, key_columns: List[str]) -> QualityCheckResult:
        """ตรวจสอบ duplicate records"""
        key_cols = ', '.join(key_columns)
        query = f"""
            SELECT 
                COUNT(*) as total,
                COUNT(*) - COUNT(DISTINCT ({key_cols})) as duplicate_count
            FROM {table}
        """
        
        import pandas as pd
        df = pd.read_sql(query, self.conn)
        
        dup_count = int(df['duplicate_count'][0])
        total = int(df['total'][0])
        dup_rate = dup_count / total if total > 0 else 0
        
        return QualityCheckResult(
            check_name=f"duplicates_{table}",
            passed=dup_count == 0,
            severity='error' if dup_rate > 0.01 else 'warning',
            details=f"Duplicates: {dup_count} of {total} ({dup_rate:.2%})",
            row_count=total,
            fail_count=dup_count
        )
    
    def run_all_checks(self, date: str) -> Dict[str, List[QualityCheckResult]]:
        """รัน quality checks ทั้งหมด"""
        results = {
            'staging': [],
            'intermediate': [],
            'marts': []
        }
        
        # Staging checks
        results['staging'].extend(
            self.check_completeness(
                'stg_webhook_events',
                ['event_id', 'event_type', 'event_date']
            )
        )
        results['staging'].append(
            self.check_freshness('stg_webhook_events', 'extracted_at')
        )
        results['staging'].append(
            self.check_duplicates('stg_webhook_events', ['event_id'])
        )
        
        # Marts checks
        results['marts'].append(
            self.check_row_count_anomaly(
                'agg_daily_followers',
                'full_date',
                date,
                expected_min=1,
                expected_max=1
            )
        )
        
        # Log results
        all_results = [r for checks in results.values() for r in checks]
        failed = [r for r in all_results if not r.passed]
        
        if failed:
            logger.warning(f"Data quality checks failed: {len(failed)} checks")
            for check in failed:
                logger.warning(f"  [{check.severity}] {check.check_name}: {check.details}")
        else:
            logger.info(f"All {len(all_results)} data quality checks passed")
        
        return results
```

---

## 8. Business Intelligence with Superset {#superset}

### Superset Setup

```yaml
# docker/superset/docker-compose.yml
version: '3.9'

services:
  superset:
    image: apache/superset:3.0.1
    ports:
      - "8088:8088"
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
      - DATABASE_URL=postgresql://superset:superset@postgres:5432/superset
      - REDIS_URL=redis://redis:6379/1
    volumes:
      - superset_home:/app/superset_home
      - ./superset_config.py:/app/pythonpath/superset_config.py
    depends_on:
      - postgres
      - redis

  superset-init:
    image: apache/superset:3.0.1
    command:
      - /bin/bash
      - -c
      - |
        superset db upgrade
        superset fab create-admin \
          --username admin \
          --firstname Admin \
          --lastname Admin \
          --email admin@example.com \
          --password admin
        superset init
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
      - DATABASE_URL=postgresql://superset:superset@postgres:5432/superset
    depends_on:
      - postgres
```

### Dashboard Definitions

```python
# scripts/create_superset_dashboards.py
# สร้าง dashboards ผ่าน Superset API

import requests
import json

SUPERSET_URL = "http://localhost:8088"
AUTH = ("admin", "admin")

def get_access_token():
    response = requests.post(f"{SUPERSET_URL}/api/v1/security/login", json={
        "username": AUTH[0],
        "password": AUTH[1],
        "provider": "db",
        "refresh": True
    })
    return response.json()["access_token"]

def create_dataset(token: str, table_name: str, database_id: int) -> int:
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.post(
        f"{SUPERSET_URL}/api/v1/dataset/",
        headers=headers,
        json={
            "database": database_id,
            "schema": "public",
            "table_name": table_name
        }
    )
    return response.json()["id"]

def create_chart(token: str, params: dict) -> int:
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.post(
        f"{SUPERSET_URL}/api/v1/chart/",
        headers=headers,
        json=params
    )
    return response.json()["id"]

# สร้าง Follower Growth Chart
def create_follower_growth_chart(token: str, dataset_id: int) -> int:
    return create_chart(token, {
        "slice_name": "LINE Follower Growth",
        "viz_type": "echarts_timeseries_line",
        "datasource_id": dataset_id,
        "datasource_type": "table",
        "params": json.dumps({
            "metrics": ["total_followers"],
            "granularity_sqla": "full_date",
            "time_range": "Last 90 days",
            "x_axis_title": "วันที่",
            "y_axis_title": "จำนวน Followers",
            "show_legend": True
        })
    })
```

---

## 9. Data Governance {#governance}

### Data Catalog

```yaml
# data_catalog/line_oa_catalog.yml
name: LINE OA Analytics Data Catalog
version: "1.0"
owner: "data-team@company.com"
last_updated: "2024-01-15"

domains:
  - name: LINE Messaging
    description: "ข้อมูลเกี่ยวกับ messages และ interactions ใน LINE OA"
    datasets:
      - name: fact_messages
        description: "ข้อเท็จจริงเกี่ยวกับ messages ที่รับส่งทุกวัน"
        owner: data-team
        pii_contains: true
        retention_days: 730
        columns:
          - name: line_user_id
            description: "LINE User ID ที่ไม่ระบุตัวตน"
            is_pii: true
            anonymization: hash
          - name: message_count
            description: "จำนวน messages ต่อวัน"
            is_pii: false
          - name: timestamp_utc
            description: "เวลาที่ส่ง message (UTC)"
            is_pii: false
        
      - name: dim_user
        description: "ข้อมูล dimension ของ users (SCD Type 2)"
        owner: data-team
        pii_contains: true
        retention_days: 1825  # 5 years
        columns:
          - name: display_name
            description: "ชื่อที่แสดงใน LINE"
            is_pii: true
            anonymization: truncate
          - name: picture_url
            description: "URL รูป profile"
            is_pii: true
            anonymization: remove

data_lineage:
  - source: LINE Insight API
    target: raw.line_insight
    schedule: daily
    owner: data-engineering
    
  - source: raw.line_insight
    target: stg_line_insight
    transform: dbt
    model: stg_line_insight
    
  - source: stg_line_insight
    target: agg_daily_followers
    transform: dbt
    model: agg_daily_followers
    depends_on: [stg_line_insight, int_user_activity]

access_policies:
  - name: analyst_access
    description: "สิทธิ์สำหรับ data analysts"
    can_access:
      - agg_daily_followers
      - agg_message_performance
      - agg_user_engagement
    cannot_access:
      - raw.*
      - dim_user (display_name, picture_url)
  
  - name: full_access
    description: "สิทธิ์สำหรับ data engineers และ data scientists"
    can_access:
      - "*"
    requires:
      - justification
      - manager_approval
```

---

## 10. Complete Airflow + LINE DAG {#complete-dag}

### Main LINE Analytics DAG

```python
# dags/line_analytics_pipeline.py
"""
LINE OA Analytics Pipeline DAG

Schedule: รันทุกวันเวลา 02:00 AM (หลังจากข้อมูลวันก่อนครบ)
"""

import logging
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.empty import EmptyOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.models import Variable
from airflow.utils.task_group import TaskGroup
from airflow.utils.trigger_rule import TriggerRule

logger = logging.getLogger(__name__)

# Default arguments
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email': ['data-team@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=1)
}

def get_processing_date(**context) -> str:
    """ได้วันที่ที่ต้องการประมวลผล"""
    execution_date = context['execution_date']
    # Process yesterday's data
    processing_date = execution_date - timedelta(days=1)
    return processing_date.strftime('%Y%m%d')

# === EXTRACT TASKS ===

def extract_line_insight(**context):
    """Extract LINE Insight API data"""
    from extractors.line_insight_extractor import LineInsightExtractor
    
    date = get_processing_date(**context)
    token = Variable.get('LINE_CHANNEL_ACCESS_TOKEN', deserialize_json=False)
    bucket = Variable.get('DATA_BUCKET', default_var='line-analytics')
    
    extractor = LineInsightExtractor(token)
    s3 = S3Hook(aws_conn_id='aws_default')
    
    logger.info(f"Extracting LINE Insight data for {date}")
    
    # ดึงข้อมูล
    followers = extractor.get_followers_count_history(date, date)
    push_stats = extractor.get_push_message_statistics(date, date)
    reply_stats = extractor.get_reply_message_statistics(date, date)
    
    # บันทึกไปยัง S3
    import json
    data = {
        'date': date,
        'followers': followers,
        'push_stats': push_stats,
        'reply_stats': reply_stats,
        'extracted_at': datetime.utcnow().isoformat()
    }
    
    key = f"raw/line_insight/{date[:4]}/{date[4:6]}/{date}/data.json"
    s3.load_string(
        string_data=json.dumps(data, ensure_ascii=False),
        key=key,
        bucket_name=bucket,
        replace=True
    )
    
    logger.info(f"Saved LINE Insight data to s3://{bucket}/{key}")
    
    # Push to XCom สำหรับ downstream tasks
    context['ti'].xcom_push(key='insight_s3_key', value=key)
    context['ti'].xcom_push(key='followers_count', value=len(followers))
    
    return {
        'date': date,
        'followers_records': len(followers),
        's3_key': key
    }

def extract_webhook_events(**context):
    """Extract webhook events from S3"""
    date = get_processing_date(**context)
    bucket = Variable.get('DATA_BUCKET', default_var='line-analytics')
    
    s3_hook = S3Hook(aws_conn_id='aws_default')
    prefix = f"raw/webhooks/{date[:4]}/{date[4:6]}/{date}/"
    
    # List all webhook files สำหรับวันนี้
    keys = s3_hook.list_keys(bucket_name=bucket, prefix=prefix)
    
    if not keys:
        logger.warning(f"No webhook files found for {date}")
        context['ti'].xcom_push(key='webhook_file_count', value=0)
        return {'date': date, 'file_count': 0}
    
    logger.info(f"Found {len(keys)} webhook files for {date}")
    context['ti'].xcom_push(key='webhook_file_count', value=len(keys))
    
    return {'date': date, 'file_count': len(keys)}

def extract_liff_events(**context):
    """Extract LIFF events from application database"""
    date = get_processing_date(**context)
    bucket = Variable.get('DATA_BUCKET', default_var='line-analytics')
    
    pg_hook = PostgresHook(postgres_conn_id='app_postgres')
    s3_hook = S3Hook(aws_conn_id='aws_default')
    
    import pandas as pd
    import io
    
    query = """
        SELECT 
            event_id,
            event_type,
            user_id,
            session_id,
            timestamp,
            page,
            properties::text,
            liff_context::text
        FROM liff_events
        WHERE timestamp::date = %(date)s
        ORDER BY timestamp
    """
    
    conn = pg_hook.get_conn()
    df = pd.read_sql(query, conn, params={'date': date.replace('', '-')})
    
    if df.empty:
        logger.info(f"No LIFF events for {date}")
        context['ti'].xcom_push(key='liff_events_count', value=0)
        return {'date': date, 'row_count': 0}
    
    # บันทึกเป็น Parquet
    buffer = io.BytesIO()
    df.to_parquet(buffer, index=False, engine='pyarrow', compression='snappy')
    
    key = f"raw/liff_events/{date[:4]}/{date[4:6]}/{date}/events.parquet"
    s3_hook.load_bytes(
        bytes_data=buffer.getvalue(),
        key=key,
        bucket_name=bucket,
        replace=True
    )
    
    logger.info(f"Extracted {len(df)} LIFF events to s3://{bucket}/{key}")
    context['ti'].xcom_push(key='liff_events_count', value=len(df))
    
    return {'date': date, 'row_count': len(df), 's3_key': key}

# === TRANSFORM TASKS ===

def run_dbt_models(**context):
    """Run dbt transformations"""
    import subprocess
    
    date = get_processing_date(**context)
    
    dbt_commands = [
        f"dbt run --profiles-dir /opt/airflow/dbt --project-dir /opt/airflow/dbt --select staging --vars '{{\"processing_date\": \"{date}\"}}'",
        f"dbt run --profiles-dir /opt/airflow/dbt --project-dir /opt/airflow/dbt --select intermediate --vars '{{\"processing_date\": \"{date}\"}}'",
        f"dbt run --profiles-dir /opt/airflow/dbt --project-dir /opt/airflow/dbt --select marts --vars '{{\"processing_date\": \"{date}\"}}'",
    ]
    
    for cmd in dbt_commands:
        logger.info(f"Running: {cmd}")
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True
        )
        
        if result.returncode != 0:
            logger.error(f"dbt failed:\n{result.stderr}")
            raise Exception(f"dbt command failed: {result.stderr}")
        
        logger.info(f"dbt output:\n{result.stdout}")

def run_dbt_tests(**context):
    """Run dbt data quality tests"""
    import subprocess
    
    date = get_processing_date(**context)
    
    cmd = f"dbt test --profiles-dir /opt/airflow/dbt --project-dir /opt/airflow/dbt --vars '{{\"processing_date\": \"{date}\"}}'"
    
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    
    if result.returncode != 0:
        logger.warning(f"Some dbt tests failed:\n{result.stdout}")
        # ไม่ fail task ถ้า tests ไม่ผ่าน (warning only)
    else:
        logger.info("All dbt tests passed")

# === LOAD TASKS ===

def update_superset_dashboards(**context):
    """Refresh Superset caches"""
    import requests
    
    superset_url = Variable.get('SUPERSET_URL', default_var='http://superset:8088')
    
    # Login
    response = requests.post(f"{superset_url}/api/v1/security/login", json={
        "username": Variable.get('SUPERSET_USER'),
        "password": Variable.get('SUPERSET_PASSWORD'),
        "provider": "db"
    })
    
    if response.status_code != 200:
        logger.warning("Could not refresh Superset caches")
        return
    
    token = response.json()["access_token"]
    headers = {"Authorization": f"Bearer {token}"}
    
    # Warm up all dashboard caches
    dashboards_response = requests.get(
        f"{superset_url}/api/v1/dashboard/",
        headers=headers
    )
    
    for dashboard in dashboards_response.json().get('result', []):
        requests.get(
            f"{superset_url}/api/v1/dashboard/{dashboard['id']}/cache_screenshot/",
            headers=headers
        )
    
    logger.info("Refreshed Superset dashboard caches")

def send_daily_summary(**context):
    """ส่งรายงานสรุปประจำวันผ่าน LINE"""
    date = get_processing_date(**context)
    
    pg_hook = PostgresHook(postgres_conn_id='dw_postgres')
    
    query = """
        SELECT 
            total_followers,
            daily_change,
            active_users,
            engagement_rate
        FROM agg_daily_followers
        WHERE full_date = %(date)s
    """
    
    import pandas as pd
    conn = pg_hook.get_conn()
    df = pd.read_sql(query, conn, params={'date': date})
    
    if df.empty:
        logger.warning(f"No summary data for {date}")
        return
    
    row = df.iloc[0]
    
    # สร้าง LINE message
    import requests
    token = Variable.get('LINE_CHANNEL_ACCESS_TOKEN')
    report_user = Variable.get('LINE_REPORT_USER_ID')
    
    message_text = (
        f"📊 รายงานสรุป LINE OA ประจำวัน {date}\n\n"
        f"👥 Followers: {row['total_followers']:,}\n"
        f"{'↑' if row['daily_change'] >= 0 else '↓'} {abs(row['daily_change']):,} จากวานนี้\n\n"
        f"🔥 Active Users: {row['active_users']:,}\n"
        f"📈 Engagement Rate: {row['engagement_rate']:.1%}"
    )
    
    requests.post(
        "https://api.line.me/v2/bot/message/push",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        },
        json={
            "to": report_user,
            "messages": [{"type": "text", "text": message_text}]
        }
    )
    
    logger.info(f"Sent daily summary to LINE")

# === NOTIFY TASKS ===

def notify_success(**context):
    """แจ้งเตือนเมื่อ pipeline สำเร็จ"""
    date = get_processing_date(**context)
    logger.info(f"LINE Analytics Pipeline completed successfully for {date}")

def notify_failure(**context):
    """แจ้งเตือนเมื่อ pipeline ล้มเหลว"""
    date = get_processing_date(**context)
    dag_id = context['dag'].dag_id
    task_id = context['task'].task_id
    
    logger.error(f"Pipeline failed! DAG: {dag_id}, Task: {task_id}, Date: {date}")
    
    # ส่งแจ้งเตือนผ่าน LINE
    import requests
    try:
        token = Variable.get('LINE_CHANNEL_ACCESS_TOKEN', default_var='')
        alert_user = Variable.get('LINE_ALERT_USER_ID', default_var='')
        
        if token and alert_user:
            requests.post(
                "https://api.line.me/v2/bot/message/push",
                headers={
                    "Authorization": f"Bearer {token}",
                    "Content-Type": "application/json"
                },
                json={
                    "to": alert_user,
                    "messages": [{
                        "type": "text",
                        "text": f"🚨 Pipeline Alert!\nDAG: {dag_id}\nTask: {task_id}\nDate: {date}\n\nกรุณาตรวจสอบ Airflow"
                    }]
                }
            )
    except Exception as e:
        logger.error(f"Could not send LINE alert: {e}")

# === DAG DEFINITION ===

with DAG(
    dag_id='line_analytics_pipeline',
    default_args=default_args,
    description='LINE OA Analytics ETL Pipeline',
    schedule_interval='0 2 * * *',  # 02:00 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['line', 'analytics', 'etl']
) as dag:

    # Start
    start = EmptyOperator(task_id='start')
    
    # Extract Group
    with TaskGroup('extract', tooltip='Extract data from all sources') as extract_group:
        
        extract_insight_task = PythonOperator(
            task_id='extract_line_insight',
            python_callable=extract_line_insight,
            provide_context=True,
            on_failure_callback=notify_failure
        )
        
        extract_webhook_task = PythonOperator(
            task_id='extract_webhook_events',
            python_callable=extract_webhook_events,
            provide_context=True,
            on_failure_callback=notify_failure
        )
        
        extract_liff_task = PythonOperator(
            task_id='extract_liff_events',
            python_callable=extract_liff_events,
            provide_context=True,
            on_failure_callback=notify_failure
        )
        
        # Run in parallel
        [extract_insight_task, extract_webhook_task, extract_liff_task]
    
    # Transform Group
    with TaskGroup('transform', tooltip='Transform data with dbt') as transform_group:
        
        dbt_run_task = PythonOperator(
            task_id='run_dbt_models',
            python_callable=run_dbt_models,
            provide_context=True,
            on_failure_callback=notify_failure
        )
        
        dbt_test_task = PythonOperator(
            task_id='run_dbt_tests',
            python_callable=run_dbt_tests,
            provide_context=True
        )
        
        dbt_run_task >> dbt_test_task
    
    # Load Group
    with TaskGroup('load', tooltip='Load data to BI tools') as load_group:
        
        update_superset_task = PythonOperator(
            task_id='update_superset',
            python_callable=update_superset_dashboards,
            provide_context=True
        )
        
        send_summary_task = PythonOperator(
            task_id='send_daily_summary',
            python_callable=send_daily_summary,
            provide_context=True
        )
        
        [update_superset_task, send_summary_task]
    
    # Success notification
    success_notify = PythonOperator(
        task_id='notify_success',
        python_callable=notify_success,
        provide_context=True,
        trigger_rule=TriggerRule.ALL_SUCCESS
    )
    
    # DAG Flow
    start >> extract_group >> transform_group >> load_group >> success_notify
```

### Airflow Variables Setup

```bash
# ตั้งค่า Airflow Variables ผ่าน CLI
airflow variables set LINE_CHANNEL_ACCESS_TOKEN "your_token_here"
airflow variables set LINE_CHANNEL_SECRET "your_secret_here"
airflow variables set LINE_REPORT_USER_ID "Uxxxxxxxxxxxxx"
airflow variables set LINE_ALERT_USER_ID "Uxxxxxxxxxxxxx"
airflow variables set DATA_BUCKET "line-analytics"
airflow variables set SUPERSET_URL "http://superset:8088"
airflow variables set SUPERSET_USER "admin"
airflow variables set SUPERSET_PASSWORD "admin"

# ตั้งค่า Connections
airflow connections add 'app_postgres' \
  --conn-type 'postgres' \
  --conn-host 'app-db' \
  --conn-login 'app_user' \
  --conn-password 'app_pass' \
  --conn-schema 'app_db' \
  --conn-port 5432

airflow connections add 'dw_postgres' \
  --conn-type 'postgres' \
  --conn-host 'dw-db' \
  --conn-login 'dw_user' \
  --conn-password 'dw_pass' \
  --conn-schema 'analytics' \
  --conn-port 5432
```

---

### สรุป Data Pipeline Best Practices

```
✅ Pipeline Design:
   - Idempotent tasks (รันซ้ำได้โดยไม่เกิดผลเสีย)
   - Backfill capability
   - Data lineage tracking
   - Incremental processing เมื่อเป็นไปได้

✅ Data Quality:
   - Test ทุก layer (staging, intermediate, marts)
   - Alert เมื่อ data ขาดหรือผิดปกติ
   - Monitor data freshness
   - Document data definitions

✅ Performance:
   - Partition data by date
   - Use Parquet format สำหรับ large datasets
   - Cache frequently accessed aggregations
   - Parallel extraction เมื่อเป็นไปได้

✅ Operations:
   - Monitor pipeline health ด้วย Airflow
   - Alert ทาง LINE เมื่อ pipeline ล้มเหลว
   - Regular review ของ data quality metrics
   - Capacity planning

💡 Cost Optimization:
   - ใช้ incremental loads แทน full refresh
   - Compress data ใน S3 (Parquet + Snappy)
   - Optimize dbt queries
   - Archive old data ไปยัง cold storage
```

---

*Part 89 จบแล้ว - ต่อไปใน Part 90: High Availability for LINE Bots*
