# ตอนที่ 90: High Availability Design สำหรับ LINE Bots

## บทนำ

High Availability (HA) คือการออกแบบระบบให้มี Uptime สูงสุด ลด Downtime และสามารถรับมือกับความผิดพลาดได้โดยอัตโนมัติ สำหรับ LINE Bot ที่ให้บริการ 24/7 นี่คือสิ่งจำเป็นอย่างยิ่ง

## สถาปัตยกรรม HA

```
┌──────────────────────────────────────────────────────────────────────┐
│                    High Availability Architecture                      │
│                                                                        │
│   Internet ──▶  DNS (GeoDNS)                                          │
│                     │                                                  │
│                ┌────┴────┐                                            │
│                │   CDN   │ (Cloudflare/AWS CloudFront)                │
│                └────┬────┘                                            │
│                     │                                                  │
│         ┌───────────┴───────────┐                                     │
│         ▼                       ▼                                     │
│  ┌─────────────┐       ┌─────────────┐                               │
│  │ Load        │       │  Load       │   ← keepalived VIP            │
│  │ Balancer 1  │       │  Balancer 2 │     (Active/Passive)          │
│  │ (MASTER)    │       │ (BACKUP)    │                               │
│  └──────┬──────┘       └─────────────┘                               │
│         │                                                              │
│    ┌────┴────────────────────┐                                        │
│    │                         │                                        │
│    ▼                         ▼                                        │
│  ┌──────────┐           ┌──────────┐                                  │
│  │  App     │           │  App     │  ← Docker Swarm/Kubernetes       │
│  │  Node 1  │           │  Node 2  │    Auto-scaling                  │
│  └──────────┘           └──────────┘                                  │
│         │                    │                                         │
│    ┌────┴────────────────────┘                                        │
│    │                                                                   │
│    ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Database Cluster                              │  │
│  │                                                                   │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐                       │  │
│  │  │ MySQL   │──▶│ MySQL   │──▶│ MySQL   │  ← Group Replication   │  │
│  │  │ Node 1  │   │ Node 2  │   │ Node 3  │    / Patroni (PG)      │  │
│  │  │(PRIMARY)│   │(REPLICA)│   │(REPLICA)│                        │  │
│  │  └─────────┘   └─────────┘   └─────────┘                       │  │
│  │                                                                   │  │
│  │  ┌─────────┐   ┌─────────┐                                       │  │
│  │  │ Redis   │◀─▶│ Redis   │   ← Redis Sentinel / Cluster         │  │
│  │  │ Master  │   │ Replica │                                        │  │
│  │  └─────────┘   └─────────┘                                       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

## HA Components สำคัญ

| Component | HA Solution | RPO | RTO |
|-----------|------------|-----|-----|
| Application | Docker Swarm / K8s | 0 | < 30s |
| MySQL | Group Replication | < 5s | < 30s |
| PostgreSQL | Patroni + etcd | < 5s | < 30s |
| MongoDB | Replica Set | < 1s | < 10s |
| Redis | Sentinel / Cluster | < 1s | < 10s |
| Load Balancer | keepalived VIP | 0 | < 3s |

## 1. MySQL Group Replication

### ติดตั้ง MySQL Group Replication

```yaml
# docker-compose.mysql-gr.yml
version: '3.8'

services:
  mysql-node1:
    image: mysql:8.0
    container_name: mysql-node1
    hostname: mysql-node1
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: linebot_db
    volumes:
      - mysql-node1-data:/var/lib/mysql
      - ./mysql/conf/node1.cnf:/etc/mysql/conf.d/group-replication.cnf
      - ./mysql/init:/docker-entrypoint-initdb.d
    networks:
      - mysql-cluster
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  mysql-node2:
    image: mysql:8.0
    container_name: mysql-node2
    hostname: mysql-node2
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql-node2-data:/var/lib/mysql
      - ./mysql/conf/node2.cnf:/etc/mysql/conf.d/group-replication.cnf
    networks:
      - mysql-cluster
    ports:
      - "3307:3306"
    depends_on:
      mysql-node1:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  mysql-node3:
    image: mysql:8.0
    container_name: mysql-node3
    hostname: mysql-node3
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql-node3-data:/var/lib/mysql
      - ./mysql/conf/node3.cnf:/etc/mysql/conf.d/group-replication.cnf
    networks:
      - mysql-cluster
    ports:
      - "3308:3306"
    depends_on:
      mysql-node1:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MySQL Router สำหรับ Load Balancing
  mysql-router:
    image: mysql/mysql-router:8.0
    container_name: mysql-router
    environment:
      MYSQL_HOST: mysql-node1
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_INNODB_CLUSTER_NAME: linebot_cluster
    networks:
      - mysql-cluster
    ports:
      - "6446:6446"   # Read/Write
      - "6447:6447"   # Read Only
    depends_on:
      - mysql-node1
      - mysql-node2
      - mysql-node3

volumes:
  mysql-node1-data:
  mysql-node2-data:
  mysql-node3-data:

networks:
  mysql-cluster:
    driver: bridge
```

### MySQL Configuration Node 1

```ini
# mysql/conf/node1.cnf
[mysqld]
server-id=1
binlog_format=ROW
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log_slave_updates=ON
log_bin=binlog
transaction_write_set_extraction=XXHASH64

# Group Replication
plugin_load_add='group_replication.so'
group_replication_group_name="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot=OFF
group_replication_local_address="mysql-node1:33061"
group_replication_group_seeds="mysql-node1:33061,mysql-node2:33061,mysql-node3:33061"
group_replication_bootstrap_group=OFF
group_replication_single_primary_mode=ON

# Performance
innodb_buffer_pool_size=512M
innodb_log_file_size=128M
max_connections=500
```

### Setup Script สำหรับ Group Replication

```sql
-- mysql/init/setup-gr.sql

-- สร้าง Replication User
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
GRANT CONNECTION_ADMIN ON *.* TO 'repl'@'%';
GRANT BACKUP_ADMIN ON *.* TO 'repl'@'%';
GRANT GROUP_REPLICATION_STREAM ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- Configure Replication Channel
CHANGE MASTER TO 
    MASTER_USER='repl',
    MASTER_PASSWORD='repl_password'
    FOR CHANNEL 'group_replication_recovery';
```

```bash
# bootstrap-gr.sh - รันบน Node 1 ก่อน
#!/bin/bash
set -e

echo "Bootstrapping MySQL Group Replication..."

# Node 1 - Bootstrap
docker exec mysql-node1 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
    SET GLOBAL group_replication_bootstrap_group=ON;
    START GROUP_REPLICATION;
    SET GLOBAL group_replication_bootstrap_group=OFF;
"

sleep 5

# Node 2 - Join
docker exec mysql-node2 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
    SET GLOBAL group_replication_bootstrap_group=OFF;
    START GROUP_REPLICATION;
"

sleep 5

# Node 3 - Join
docker exec mysql-node3 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
    SET GLOBAL group_replication_bootstrap_group=OFF;
    START GROUP_REPLICATION;
"

# ตรวจสอบสถานะ
docker exec mysql-node1 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
    SELECT * FROM performance_schema.replication_group_members;
"

echo "Group Replication setup complete!"
```

## 2. PostgreSQL กับ Patroni + etcd

```yaml
# docker-compose.postgres-ha.yml
version: '3.8'

services:
  # etcd Cluster (3 nodes)
  etcd1:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd1
    hostname: etcd1
    command:
      - etcd
      - --name=etcd1
      - --data-dir=/etcd-data
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd1:2379
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-advertise-peer-urls=http://etcd1:2380
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-cluster
    volumes:
      - etcd1-data:/etcd-data
    networks:
      - pg-cluster

  etcd2:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd2
    hostname: etcd2
    command:
      - etcd
      - --name=etcd2
      - --data-dir=/etcd-data
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd2:2379
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-advertise-peer-urls=http://etcd2:2380
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-cluster
    volumes:
      - etcd2-data:/etcd-data
    networks:
      - pg-cluster

  etcd3:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd3
    hostname: etcd3
    command:
      - etcd
      - --name=etcd3
      - --data-dir=/etcd-data
      - --listen-client-urls=http://0.0.0.0:2379
      - --advertise-client-urls=http://etcd3:2379
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-advertise-peer-urls=http://etcd3:2380
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-cluster
    volumes:
      - etcd3-data:/etcd-data
    networks:
      - pg-cluster

  # PostgreSQL + Patroni Node 1
  pg-node1:
    image: patroni:latest
    build:
      context: ./patroni
      dockerfile: Dockerfile
    container_name: pg-node1
    hostname: pg-node1
    environment:
      PATRONI_NAME: pg-node1
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: pg-node1:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: pg-node1:8008
      PATRONI_ETCD3_HOSTS: "etcd1:2379,etcd2:2379,etcd3:2379"
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: ${PG_SUPERUSER_PASSWORD}
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: ${PG_REPLICATION_PASSWORD}
      PATRONI_SCOPE: linebot-cluster
    volumes:
      - pg-node1-data:/var/lib/postgresql/data
      - ./patroni/patroni.yml:/etc/patroni.yml
    networks:
      - pg-cluster
    ports:
      - "5432:5432"
      - "8008:8008"
    depends_on:
      - etcd1
      - etcd2
      - etcd3

  pg-node2:
    image: patroni:latest
    build:
      context: ./patroni
      dockerfile: Dockerfile
    container_name: pg-node2
    hostname: pg-node2
    environment:
      PATRONI_NAME: pg-node2
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: pg-node2:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: pg-node2:8008
      PATRONI_ETCD3_HOSTS: "etcd1:2379,etcd2:2379,etcd3:2379"
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: ${PG_SUPERUSER_PASSWORD}
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: ${PG_REPLICATION_PASSWORD}
      PATRONI_SCOPE: linebot-cluster
    volumes:
      - pg-node2-data:/var/lib/postgresql/data
      - ./patroni/patroni.yml:/etc/patroni.yml
    networks:
      - pg-cluster
    ports:
      - "5433:5432"
      - "8009:8008"
    depends_on:
      - pg-node1

  pg-node3:
    image: patroni:latest
    build:
      context: ./patroni
      dockerfile: Dockerfile
    container_name: pg-node3
    hostname: pg-node3
    environment:
      PATRONI_NAME: pg-node3
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: pg-node3:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: pg-node3:8008
      PATRONI_ETCD3_HOSTS: "etcd1:2379,etcd2:2379,etcd3:2379"
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: ${PG_SUPERUSER_PASSWORD}
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: ${PG_REPLICATION_PASSWORD}
      PATRONI_SCOPE: linebot-cluster
    volumes:
      - pg-node3-data:/var/lib/postgresql/data
      - ./patroni/patroni.yml:/etc/patroni.yml
    networks:
      - pg-cluster
    ports:
      - "5434:5432"
      - "8010:8008"
    depends_on:
      - pg-node1

  # HAProxy สำหรับ PostgreSQL Load Balancing
  haproxy-pg:
    image: haproxy:2.8-alpine
    container_name: haproxy-pg
    volumes:
      - ./haproxy/pg-haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - "5000:5000"   # Read/Write (Primary)
      - "5001:5001"   # Read Only (Replicas)
      - "7000:7000"   # Stats
    networks:
      - pg-cluster
    depends_on:
      - pg-node1
      - pg-node2
      - pg-node3

volumes:
  etcd1-data:
  etcd2-data:
  etcd3-data:
  pg-node1-data:
  pg-node2-data:
  pg-node3-data:

networks:
  pg-cluster:
    driver: bridge
```

### HAProxy Config สำหรับ PostgreSQL

```
# haproxy/pg-haproxy.cfg
global
    maxconn 500
    log stdout format raw local0

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

# Stats Page
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s

# Read/Write - Primary only
listen postgres-rw
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 pg-node1:5432 maxconn 100 check port 8008
    server pg-node2 pg-node2:5432 maxconn 100 check port 8008
    server pg-node3 pg-node3:5432 maxconn 100 check port 8008

# Read Only - All nodes (load balanced)
listen postgres-ro
    bind *:5001
    option httpchk
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 pg-node1:5432 maxconn 100 check port 8008
    server pg-node2 pg-node2:5432 maxconn 100 check port 8008
    server pg-node3 pg-node3:5432 maxconn 100 check port 8008
```

## 3. MongoDB Replica Set

```yaml
# docker-compose.mongodb-rs.yml
version: '3.8'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    hostname: mongo1
    command: mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb/keyfile
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo1-data:/data/db
      - mongo1-config:/data/configdb
      - ./mongodb/keyfile:/etc/mongodb/keyfile:ro
    networks:
      - mongo-cluster
    ports:
      - "27017:27017"
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    hostname: mongo2
    command: mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb/keyfile
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo2-data:/data/db
      - mongo2-config:/data/configdb
      - ./mongodb/keyfile:/etc/mongodb/keyfile:ro
    networks:
      - mongo-cluster
    ports:
      - "27018:27017"
    depends_on:
      mongo1:
        condition: service_healthy

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    hostname: mongo3
    command: mongod --replSet rs0 --bind_ip_all --keyFile /etc/mongodb/keyfile
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo3-data:/data/db
      - mongo3-config:/data/configdb
      - ./mongodb/keyfile:/etc/mongodb/keyfile:ro
    networks:
      - mongo-cluster
    ports:
      - "27019:27017"
    depends_on:
      mongo1:
        condition: service_healthy

  # MongoDB Init (Replica Set Setup)
  mongo-init:
    image: mongo:7.0
    container_name: mongo-init
    depends_on:
      - mongo1
      - mongo2
      - mongo3
    command: |
      mongosh --host mongo1:27017 -u admin -p ${MONGO_PASSWORD} --eval "
        rs.initiate({
          _id: 'rs0',
          members: [
            { _id: 0, host: 'mongo1:27017', priority: 2 },
            { _id: 1, host: 'mongo2:27017', priority: 1 },
            { _id: 2, host: 'mongo3:27017', priority: 1 }
          ]
        });
      "
    networks:
      - mongo-cluster

volumes:
  mongo1-data:
  mongo2-data:
  mongo3-data:
  mongo1-config:
  mongo2-config:
  mongo3-config:

networks:
  mongo-cluster:
    driver: bridge
```

```bash
# สร้าง MongoDB Keyfile
openssl rand -base64 756 > ./mongodb/keyfile
chmod 400 ./mongodb/keyfile
```

## 4. Redis Sentinel + Cluster

```yaml
# docker-compose.redis-ha.yml
version: '3.8'

# Option A: Redis Sentinel (Simple HA)
services:
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    hostname: redis-master
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --appendfsync everysec
      --save 60 1000
    volumes:
      - redis-master-data:/data
    networks:
      - redis-cluster
    ports:
      - "6379:6379"
    healthcheck:
      test: redis-cli -a ${REDIS_PASSWORD} ping
      interval: 5s
      timeout: 3s
      retries: 5

  redis-replica1:
    image: redis:7-alpine
    container_name: redis-replica1
    hostname: redis-replica1
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --replicaof redis-master 6379
      --masterauth ${REDIS_PASSWORD}
      --replica-read-only yes
      --appendonly yes
    volumes:
      - redis-replica1-data:/data
    networks:
      - redis-cluster
    ports:
      - "6380:6379"
    depends_on:
      redis-master:
        condition: service_healthy

  redis-replica2:
    image: redis:7-alpine
    container_name: redis-replica2
    hostname: redis-replica2
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --replicaof redis-master 6379
      --masterauth ${REDIS_PASSWORD}
      --replica-read-only yes
      --appendonly yes
    volumes:
      - redis-replica2-data:/data
    networks:
      - redis-cluster
    ports:
      - "6381:6379"
    depends_on:
      redis-master:
        condition: service_healthy

  # Sentinel 1
  redis-sentinel1:
    image: redis:7-alpine
    container_name: redis-sentinel1
    hostname: redis-sentinel1
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./redis/sentinel.conf:/etc/redis/sentinel.conf
    networks:
      - redis-cluster
    ports:
      - "26379:26379"
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2

  redis-sentinel2:
    image: redis:7-alpine
    container_name: redis-sentinel2
    hostname: redis-sentinel2
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./redis/sentinel.conf:/etc/redis/sentinel.conf
    networks:
      - redis-cluster
    ports:
      - "26380:26379"
    depends_on:
      - redis-master

  redis-sentinel3:
    image: redis:7-alpine
    container_name: redis-sentinel3
    hostname: redis-sentinel3
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./redis/sentinel.conf:/etc/redis/sentinel.conf
    networks:
      - redis-cluster
    ports:
      - "26381:26379"
    depends_on:
      - redis-master

volumes:
  redis-master-data:
  redis-replica1-data:
  redis-replica2-data:

networks:
  redis-cluster:
    driver: bridge
```

### Redis Sentinel Configuration

```conf
# redis/sentinel.conf
sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster your_redis_password
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 60000
sentinel announce-ip sentinel1
sentinel announce-port 26379
bind 0.0.0.0
port 26379
```

### Node.js Connection กับ Redis Sentinel

```javascript
// src/cache/redis-sentinel.js
const Redis = require('ioredis');

class RedisSentinelClient {
    constructor() {
        this.client = new Redis({
            sentinels: [
                { host: process.env.SENTINEL1_HOST || 'redis-sentinel1', port: 26379 },
                { host: process.env.SENTINEL2_HOST || 'redis-sentinel2', port: 26379 },
                { host: process.env.SENTINEL3_HOST || 'redis-sentinel3', port: 26379 }
            ],
            name: 'mymaster',
            password: process.env.REDIS_PASSWORD,
            sentinelPassword: process.env.REDIS_PASSWORD,
            retryStrategy: (times) => {
                const delay = Math.min(times * 50, 2000);
                return delay;
            },
            reconnectOnError: (err) => {
                const targetError = 'READONLY';
                return err.message.includes(targetError);
            }
        });
        
        this.client.on('connect', () => console.log('Redis Sentinel connected'));
        this.client.on('error', (err) => console.error('Redis Sentinel error:', err));
        this.client.on('+switch-master', (channel, oldHost, oldPort, newHost, newPort) => {
            console.log(`Redis Sentinel: failover from ${oldHost}:${oldPort} to ${newHost}:${newPort}`);
        });
    }

    // Wrapper Methods
    async get(key) { return await this.client.get(key); }
    async set(key, value, ...args) { return await this.client.set(key, value, ...args); }
    async setex(key, seconds, value) { return await this.client.setex(key, seconds, value); }
    async del(key) { return await this.client.del(key); }
    async exists(key) { return await this.client.exists(key); }
    async expire(key, seconds) { return await this.client.expire(key, seconds); }
    async hget(key, field) { return await this.client.hget(key, field); }
    async hset(key, ...args) { return await this.client.hset(key, ...args); }
    async hgetall(key) { return await this.client.hgetall(key); }
    async hincrby(key, field, increment) { return await this.client.hincrby(key, field, increment); }
    async sadd(key, ...members) { return await this.client.sadd(key, ...members); }
    async scard(key) { return await this.client.scard(key); }
    pipeline() { return this.client.pipeline(); }
    async quit() { return await this.client.quit(); }
}

module.exports = new RedisSentinelClient();
```

## 5. Application HA กับ Nginx + keepalived

### Nginx Load Balancer Configuration

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time ut=$upstream_response_time';
    
    access_log /var/log/nginx/access.log main buffer=16k flush=5s;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
    
    # Rate Limiting (LINE Webhook)
    limit_req_zone $binary_remote_addr zone=webhook:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=perip:10m;
    
    # Upstream App Servers
    upstream linebot_backend {
        least_conn;
        
        server app-node1:3000 max_fails=3 fail_timeout=30s weight=1;
        server app-node2:3000 max_fails=3 fail_timeout=30s weight=1;
        
        keepalive 32;
    }
    
    # Health Check Endpoint
    server {
        listen 8080;
        location /health {
            access_log off;
            return 200 "OK\n";
        }
    }
    
    # Main Server
    server {
        listen 80;
        server_name _;
        
        # Redirect ไป HTTPS
        return 301 https://$host$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name _;
        
        # SSL Certificate
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        
        # Security Headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        
        # LINE Webhook
        location /webhook {
            limit_req zone=webhook burst=50 nodelay;
            limit_conn perip 50;
            
            proxy_pass http://linebot_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
            
            # Buffer settings
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }
        
        # API Endpoints
        location /api {
            proxy_pass http://linebot_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            proxy_connect_timeout 10s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }
        
        # Health Check
        location /health {
            access_log off;
            proxy_pass http://linebot_backend/health;
        }
    }
}
```

### keepalived Configuration สำหรับ VIP

```
# keepalived/master.conf (สำหรับ LB1)
vrrp_script chk_nginx {
    script "/usr/bin/killall -0 nginx"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101         # MASTER priority สูงกว่า BACKUP
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass line_bot_ha_secret
    }
    
    virtual_ipaddress {
        192.168.1.100/24  # VIP - ใช้ IP นี้ใน DNS
    }
    
    track_script {
        chk_nginx
    }
    
    notify_master "/etc/keepalived/notify.sh MASTER"
    notify_backup "/etc/keepalived/notify.sh BACKUP"
    notify_fault  "/etc/keepalived/notify.sh FAULT"
}
```

```
# keepalived/backup.conf (สำหรับ LB2)
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100         # BACKUP priority ต่ำกว่า
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass line_bot_ha_secret
    }
    
    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

## 6. Health Checks และ Probes

```javascript
// src/health/health-checker.js
const express = require('express');
const db = require('../database/mysql');
const redis = require('../cache/redis');
const { Client: LineClient } = require('@line/bot-sdk');

class HealthChecker {
    constructor() {
        this.router = express.Router();
        this.setupRoutes();
    }

    setupRoutes() {
        // Simple Health Check (สำหรับ Load Balancer)
        this.router.get('/health', (req, res) => {
            res.status(200).json({ status: 'ok', timestamp: Date.now() });
        });

        // Readiness Check (ตรวจสอบว่าพร้อมรับ Traffic)
        this.router.get('/ready', async (req, res) => {
            const checks = await this.runReadinessChecks();
            const allReady = Object.values(checks).every(c => c.status === 'ok');
            
            res.status(allReady ? 200 : 503).json({
                status: allReady ? 'ready' : 'not_ready',
                checks,
                timestamp: Date.now()
            });
        });

        // Liveness Check (ตรวจสอบว่า Process ยังทำงาน)
        this.router.get('/live', (req, res) => {
            const memUsage = process.memoryUsage();
            const heapUsedMB = memUsage.heapUsed / 1024 / 1024;
            
            // ถ้า Heap ใช้มากกว่า 900MB ให้ถือว่า Unhealthy
            if (heapUsedMB > 900) {
                return res.status(503).json({
                    status: 'unhealthy',
                    reason: 'High memory usage',
                    heapUsedMB
                });
            }
            
            res.status(200).json({
                status: 'alive',
                uptime: process.uptime(),
                memory: {
                    heapUsedMB: Math.round(heapUsedMB),
                    heapTotalMB: Math.round(memUsage.heapTotal / 1024 / 1024)
                }
            });
        });

        // Detailed Status (สำหรับ Monitoring)
        this.router.get('/status', async (req, res) => {
            const status = await this.getDetailedStatus();
            res.json(status);
        });
    }

    // Readiness Checks
    async runReadinessChecks() {
        const checks = {};
        
        // Database Check
        try {
            await db.query('SELECT 1');
            checks.database = { status: 'ok', latency: 0 };
        } catch (e) {
            checks.database = { status: 'error', error: e.message };
        }
        
        // Redis Check
        try {
            const start = Date.now();
            await redis.set('health_check', '1', 'EX', 10);
            const val = await redis.get('health_check');
            checks.redis = { 
                status: val === '1' ? 'ok' : 'error',
                latency: Date.now() - start
            };
        } catch (e) {
            checks.redis = { status: 'error', error: e.message };
        }
        
        // LINE API Check
        try {
            const lineClient = new LineClient({
                channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
            });
            await lineClient.getBotInfo();
            checks.line_api = { status: 'ok' };
        } catch (e) {
            checks.line_api = { status: 'error', error: e.message };
        }
        
        return checks;
    }

    // Detailed Status
    async getDetailedStatus() {
        const checks = await this.runReadinessChecks();
        const memUsage = process.memoryUsage();
        
        // DB Stats
        let dbStats = {};
        try {
            const result = await db.query('SHOW STATUS LIKE "Threads_connected"');
            dbStats.connections = result[0]?.Value || 0;
        } catch (e) {}
        
        return {
            service: 'line-bot',
            version: process.env.npm_package_version || '1.0.0',
            environment: process.env.NODE_ENV || 'production',
            timestamp: new Date().toISOString(),
            uptime: Math.round(process.uptime()),
            components: checks,
            resources: {
                memory: {
                    heapUsedMB: Math.round(memUsage.heapUsed / 1024 / 1024),
                    heapTotalMB: Math.round(memUsage.heapTotal / 1024 / 1024),
                    rssMB: Math.round(memUsage.rss / 1024 / 1024)
                },
                database: dbStats
            }
        };
    }

    getRouter() {
        return this.router;
    }
}

module.exports = new HealthChecker();
```

## 7. Automated Failover Procedures

```javascript
// src/ha/failover-manager.js
const db = require('../database/mysql');
const redis = require('../cache/redis');
const axios = require('axios');

class FailoverManager {
    constructor() {
        this.primaryDbHost = process.env.DB_PRIMARY_HOST || 'mysql-router';
        this.primaryDbPort = process.env.DB_PRIMARY_PORT || 6446;
        this.readOnlyDbHost = process.env.DB_READONLY_HOST || 'mysql-router';
        this.readOnlyDbPort = process.env.DB_READONLY_PORT || 6447;
        
        this.checkInterval = 10000; // 10 วินาที
        this.failureThreshold = 3;
        this.failureCount = 0;
        this.isFailoverActive = false;
    }

    // เริ่ม Health Monitoring
    startMonitoring() {
        setInterval(() => this.checkHealth(), this.checkInterval);
        console.log('HA Failover monitoring started');
    }

    // ตรวจสอบ Health
    async checkHealth() {
        try {
            // ตรวจสอบ Database
            const dbHealthy = await this.checkDatabase();
            
            if (!dbHealthy) {
                this.failureCount++;
                console.warn(`Database health check failed (${this.failureCount}/${this.failureThreshold})`);
                
                if (this.failureCount >= this.failureThreshold && !this.isFailoverActive) {
                    await this.triggerDatabaseFailover();
                }
            } else {
                if (this.failureCount > 0) {
                    console.log('Database recovered');
                    this.failureCount = 0;
                    
                    if (this.isFailoverActive) {
                        await this.triggerDatabaseRecovery();
                    }
                }
            }
        } catch (error) {
            console.error('Health check error:', error);
        }
    }

    // ตรวจสอบ Database
    async checkDatabase() {
        try {
            const startTime = Date.now();
            await db.query('SELECT 1 AS health_check', [], { timeout: 3000 });
            const latency = Date.now() - startTime;
            
            // อัพเดท Metric ใน Redis
            await redis.set('db:latency', latency, 'EX', 60);
            
            return latency < 5000;
        } catch {
            return false;
        }
    }

    // เรียก Database Failover
    async triggerDatabaseFailover() {
        this.isFailoverActive = true;
        
        console.error('CRITICAL: Database failover triggered!');
        
        try {
            // แจ้งเตือนทีม
            await this.sendAlert({
                severity: 'critical',
                title: 'Database Failover Triggered',
                message: `Primary database is down. Failover initiated at ${new Date().toISOString()}`,
                runbook: 'https://wiki.company.com/runbook/db-failover'
            });
            
            // บันทึก Event
            await redis.set('ha:failover_time', Date.now(), 'EX', 86400);
            await redis.incr('ha:failover_count');
            
            // ถ้าใช้ MySQL Group Replication / Patroni จะ Auto-failover อยู่แล้ว
            // แต่เราต้องรอให้ Connection Pool reconnect
            await this.reconnectDatabasePool();
            
        } catch (error) {
            console.error('Error during failover:', error);
        }
    }

    // Recovery หลัง Failover
    async triggerDatabaseRecovery() {
        console.log('Database recovered, ending failover state');
        this.isFailoverActive = false;
        
        await this.sendAlert({
            severity: 'info',
            title: 'Database Recovered',
            message: `Database is back online. Recovery completed at ${new Date().toISOString()}`
        });
        
        // คำนวณ Downtime
        const failoverTime = await redis.get('ha:failover_time');
        if (failoverTime) {
            const downtimeMs = Date.now() - parseInt(failoverTime);
            console.log(`Downtime: ${downtimeMs / 1000}s`);
        }
    }

    // Reconnect Database Pool
    async reconnectDatabasePool() {
        console.log('Reconnecting database pool...');
        
        // รอให้ Failover เสร็จ
        await new Promise(resolve => setTimeout(resolve, 5000));
        
        // ลอง Query ใหม่
        for (let i = 0; i < 10; i++) {
            try {
                await db.query('SELECT 1');
                console.log('Database pool reconnected successfully');
                return;
            } catch {
                console.log(`Reconnect attempt ${i + 1}/10 failed, retrying...`);
                await new Promise(resolve => setTimeout(resolve, 3000));
            }
        }
        
        throw new Error('Failed to reconnect database after 10 attempts');
    }

    // ส่งแจ้งเตือน
    async sendAlert(alert) {
        // ส่ง LINE Notify
        if (process.env.LINE_NOTIFY_TOKEN) {
            try {
                await axios.post('https://notify-api.line.me/api/notify', 
                    new URLSearchParams({
                        message: `\n[${alert.severity.toUpperCase()}] ${alert.title}\n${alert.message}${
                            alert.runbook ? `\nRunbook: ${alert.runbook}` : ''
                        }`
                    }),
                    { headers: { Authorization: `Bearer ${process.env.LINE_NOTIFY_TOKEN}` } }
                );
            } catch (e) {
                console.error('LINE Notify error:', e.message);
            }
        }
        
        // ส่ง Webhook (Slack, etc.)
        if (process.env.ALERT_WEBHOOK_URL) {
            try {
                await axios.post(process.env.ALERT_WEBHOOK_URL, {
                    text: `[${alert.severity.toUpperCase()}] ${alert.title}: ${alert.message}`
                });
            } catch (e) {
                console.error('Webhook alert error:', e.message);
            }
        }
        
        console.log(`Alert sent: [${alert.severity}] ${alert.title}`);
    }
}

module.exports = new FailoverManager();
```

## 8. Backup Strategies (RPO/RTO)

```bash
#!/bin/bash
# scripts/backup.sh

set -euo pipefail

BACKUP_DIR="/backup/$(date +%Y/%m/%d)"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# สร้าง Backup Directory
mkdir -p "$BACKUP_DIR"

echo "=== Starting Backup at $(date) ==="

# ============================================
# MySQL Backup
# ============================================
backup_mysql() {
    echo "Backing up MySQL..."
    
    # Full Backup ด้วย mysqldump
    docker exec mysql-node1 mysqldump \
        -u root -p"${MYSQL_ROOT_PASSWORD}" \
        --all-databases \
        --single-transaction \
        --flush-logs \
        --master-data=2 \
        --compress \
        | gzip > "${BACKUP_DIR}/mysql_full_${DATE}.sql.gz"
    
    echo "MySQL backup completed: ${BACKUP_DIR}/mysql_full_${DATE}.sql.gz"
    
    # Binary Log Backup (สำหรับ Point-in-time Recovery)
    docker exec mysql-node1 mysql -u root -p"${MYSQL_ROOT_PASSWORD}" -e "FLUSH LOGS;"
    
    # Copy Binary Logs
    docker cp mysql-node1:/var/lib/mysql/ "/tmp/mysql-binlogs-${DATE}/"
    find "/tmp/mysql-binlogs-${DATE}/" -name "binlog.*" -exec gzip -c {} \; > \
        "${BACKUP_DIR}/mysql_binlogs_${DATE}.tar.gz"
    rm -rf "/tmp/mysql-binlogs-${DATE}/"
}

# ============================================
# PostgreSQL Backup
# ============================================
backup_postgres() {
    echo "Backing up PostgreSQL..."
    
    # pg_basebackup สำหรับ Physical Backup
    docker exec pg-node1 pg_basebackup \
        -U postgres \
        -D /tmp/pg_backup \
        -Ft -z -Xs -P
    
    docker cp "pg-node1:/tmp/pg_backup/" "${BACKUP_DIR}/pg_${DATE}/"
    docker exec pg-node1 rm -rf /tmp/pg_backup
    
    echo "PostgreSQL backup completed"
    
    # Logical Backup ด้วย pg_dump
    docker exec pg-node1 pg_dumpall \
        -U postgres \
        | gzip > "${BACKUP_DIR}/pg_logical_${DATE}.sql.gz"
}

# ============================================
# Redis Backup
# ============================================
backup_redis() {
    echo "Backing up Redis..."
    
    # BGSAVE
    docker exec redis-master redis-cli -a "${REDIS_PASSWORD}" BGSAVE
    sleep 5
    
    # Copy RDB File
    docker cp redis-master:/data/dump.rdb "${BACKUP_DIR}/redis_${DATE}.rdb"
    gzip "${BACKUP_DIR}/redis_${DATE}.rdb"
    
    echo "Redis backup completed"
}

# ============================================
# Upload ไป S3
# ============================================
upload_to_s3() {
    echo "Uploading backups to S3..."
    
    aws s3 sync "${BACKUP_DIR}" \
        "s3://${S3_BACKUP_BUCKET}/$(date +%Y/%m/%d)/" \
        --storage-class STANDARD_IA \
        --quiet
    
    echo "S3 upload completed"
}

# ============================================
# ลบ Backup เก่า
# ============================================
cleanup_old_backups() {
    echo "Cleaning up old backups..."
    
    find /backup -type f -mtime +${RETENTION_DAYS} -delete
    find /backup -type d -empty -delete
    
    # ลบใน S3 ด้วย (เก็บ 90 วัน)
    aws s3 ls "s3://${S3_BACKUP_BUCKET}/" --recursive | \
        awk '{print $4}' | \
        while read file; do
            file_date=$(echo "$file" | grep -oP '\d{4}/\d{2}/\d{2}')
            if [ -n "$file_date" ]; then
                days_ago=$(( ($(date +%s) - $(date -d "$file_date" +%s)) / 86400 ))
                if [ "$days_ago" -gt 90 ]; then
                    aws s3 rm "s3://${S3_BACKUP_BUCKET}/${file}"
                fi
            fi
        done
    
    echo "Cleanup completed"
}

# ============================================
# ตรวจสอบ Backup
# ============================================
verify_backup() {
    echo "Verifying backups..."
    
    # ตรวจสอบ MySQL
    if gunzip -t "${BACKUP_DIR}/mysql_full_${DATE}.sql.gz" 2>/dev/null; then
        echo "MySQL backup: OK"
    else
        echo "MySQL backup: FAILED"
        send_alert "Backup verification failed: MySQL"
    fi
    
    # ตรวจสอบ Redis
    if [ -f "${BACKUP_DIR}/redis_${DATE}.rdb.gz" ]; then
        echo "Redis backup: OK"
    else
        echo "Redis backup: FAILED"
    fi
}

# รัน
backup_mysql
backup_postgres
backup_redis
verify_backup
upload_to_s3
cleanup_old_backups

echo "=== Backup completed at $(date) ==="
```

## 9. Disaster Recovery Runbook

```markdown
# DR Runbook สำหรับ LINE Bot

## สถานการณ์ที่ 1: Database Primary ล้ม

### ตรวจสอบ:
```bash
# ตรวจสอบสถานะ MySQL Group Replication
docker exec mysql-node1 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
    SELECT * FROM performance_schema.replication_group_members;
"

# ตรวจสอบสถานะ Patroni (PostgreSQL)
patronictl -c /etc/patroni.yml topology

# ตรวจสอบ Redis Sentinel
redis-cli -p 26379 SENTINEL masters
```

### Actions:
1. MySQL Router จะ auto-switch ไป secondary อัตโนมัติ
2. ตรวจสอบ Application logs ว่า Connection ใหม่ทำงาน
3. แจ้ง DBA ตรวจสอบ Node ที่ล้ม

## สถานการณ์ที่ 2: Application Node ล้ม

### ตรวจสอบ:
```bash
# ดู Docker Swarm
docker service ps line-bot

# ดู Container Status
docker ps -a | grep line-bot
```

### Actions:
1. Docker Swarm จะ restart container อัตโนมัติ
2. Nginx จะหยุดส่ง traffic ไป node ที่ล้ม
3. ตรวจสอบ logs: `docker service logs line-bot`

## สถานการณ์ที่ 3: Data Center ล้ม (Full DR)

### Prerequisites:
- Secondary DC พร้อมใน Standby mode
- Backup ล่าสุดใน S3

### Procedure:

**1. ประเมินสถานการณ์ (5 นาที)**
- ตรวจสอบว่า Primary DC กลับมาได้ไหม
- ประเมิน Data Loss (RPO)

**2. ตัดสินใจ Failover (2 นาที)**
- ถ้า ETA > 30 นาที → ทำ DR

**3. Start Secondary DC (15 นาที)**
```bash
cd /opt/linebot-dr
./scripts/start-dr.sh

# ตรวจสอบ Database
./scripts/verify-db.sh

# Point DNS ไป Secondary DC
./scripts/update-dns.sh
```

**4. ตรวจสอบ (5 นาที)**
- ทดสอบ LINE Webhook
- ตรวจสอบ Database data
- Notify ทีม

**Total RTO: ~30 นาที**
```

## 10. Chaos Engineering Basics

```javascript
// src/chaos/chaos-monkey.js
const db = require('../database/mysql');
const redis = require('../cache/redis');

class ChaosMonkey {
    constructor() {
        this.enabled = process.env.CHAOS_MONKEY_ENABLED === 'true';
        this.experiments = [];
    }

    // ลงทะเบียน Experiment
    registerExperiment(name, probability, action) {
        this.experiments.push({ name, probability, action });
    }

    // รัน Random Chaos
    async runRandomChaos() {
        if (!this.enabled) return;
        
        for (const experiment of this.experiments) {
            if (Math.random() < experiment.probability) {
                console.log(`[ChaosMonkey] Running: ${experiment.name}`);
                try {
                    await experiment.action();
                } catch (error) {
                    console.error(`[ChaosMonkey] Experiment ${experiment.name} failed:`, error.message);
                }
            }
        }
    }

    // ทดสอบ Database Latency
    addLatencyExperiment(maxLatencyMs = 1000) {
        this.registerExperiment(
            'database_latency',
            0.01, // 1% probability
            async () => {
                const delay = Math.random() * maxLatencyMs;
                await new Promise(resolve => setTimeout(resolve, delay));
                console.log(`[ChaosMonkey] Added ${delay.toFixed(0)}ms database latency`);
            }
        );
    }

    // ทดสอบ Cache Miss
    addCacheMissExperiment() {
        this.registerExperiment(
            'cache_miss',
            0.05, // 5% probability
            async () => {
                await redis.flushdb();
                console.log('[ChaosMonkey] Cache cleared - simulating cache miss');
            }
        );
    }

    // ทดสอบ High Memory Usage
    addMemoryPressureExperiment() {
        this.registerExperiment(
            'memory_pressure',
            0.001, // 0.1% probability
            async () => {
                // สร้าง Memory Pressure
                const leak = [];
                for (let i = 0; i < 1000; i++) {
                    leak.push(Buffer.alloc(1024)); // 1KB ต่อครั้ง
                }
                console.log('[ChaosMonkey] Created memory pressure');
                
                // คืน Memory หลัง 10 วินาที
                setTimeout(() => {
                    leak.length = 0;
                }, 10000);
            }
        );
    }
}

// Middleware สำหรับ Express
const chaosMiddleware = (req, res, next) => {
    if (process.env.CHAOS_MONKEY_ENABLED !== 'true') {
        return next();
    }
    
    // 1% โอกาสที่จะช้า
    if (Math.random() < 0.01) {
        const delay = Math.random() * 2000;
        setTimeout(next, delay);
        return;
    }
    
    next();
};

module.exports = { ChaosMonkey, chaosMiddleware };
```

## 11. Complete HA Setup กับ Docker Compose

```yaml
# docker-compose.ha.yml
version: '3.8'

services:
  # ============================================
  # Load Balancers
  # ============================================
  nginx-lb1:
    image: nginx:1.25-alpine
    container_name: nginx-lb1
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend
      - backend
    depends_on:
      - app-node1
      - app-node2
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  nginx-lb2:
    image: nginx:1.25-alpine
    container_name: nginx-lb2
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    ports:
      - "8080:80"
      - "8443:443"
    networks:
      - frontend
      - backend
    depends_on:
      - app-node1
      - app-node2
    restart: always

  # ============================================
  # Application Nodes
  # ============================================
  app-node1:
    image: line-bot:latest
    build:
      context: .
      dockerfile: Dockerfile.prod
    container_name: app-node1
    hostname: app-node1
    environment:
      - NODE_ENV=production
      - PORT=3000
      - NODE_ID=node1
      - DB_HOST=mysql-router
      - DB_PORT=6446
      - DB_READONLY_HOST=mysql-router
      - DB_READONLY_PORT=6447
      - REDIS_SENTINELS=redis-sentinel1:26379,redis-sentinel2:26379,redis-sentinel3:26379
      - REDIS_NAME=mymaster
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    networks:
      - backend
      - db-network
      - redis-network
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M

  app-node2:
    image: line-bot:latest
    container_name: app-node2
    hostname: app-node2
    environment:
      - NODE_ENV=production
      - PORT=3000
      - NODE_ID=node2
      - DB_HOST=mysql-router
      - DB_PORT=6446
      - DB_READONLY_HOST=mysql-router
      - DB_READONLY_PORT=6447
      - REDIS_SENTINELS=redis-sentinel1:26379,redis-sentinel2:26379,redis-sentinel3:26379
      - REDIS_NAME=mymaster
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    networks:
      - backend
      - db-network
      - redis-network
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  # ============================================
  # Monitoring
  # ============================================
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: always

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_SERVER_ROOT_URL=https://monitoring.yourcompany.com
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3001:3000"
    networks:
      - monitoring
    restart: always

  # Node Exporter สำหรับ System Metrics
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
    networks:
      - monitoring
    restart: always

volumes:
  prometheus-data:
  grafana-data:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  db-network:
    external: true
    name: mysql-cluster
  redis-network:
    external: true
    name: redis-cluster_redis-cluster
  monitoring:
    driver: bridge
```

## 12. Prometheus Monitoring Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'linebot-ha'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  # App Nodes
  - job_name: 'linebot-app'
    static_configs:
      - targets:
          - 'app-node1:3001'
          - 'app-node2:3001'
    metrics_path: '/metrics'
    
  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-exporter:9100'
    
  # MySQL Exporter
  - job_name: 'mysql'
    static_configs:
      - targets:
          - 'mysql-exporter:9104'
    
  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets:
          - 'redis-exporter:9121'
    
  # Nginx Exporter
  - job_name: 'nginx'
    static_configs:
      - targets:
          - 'nginx-lb1:9113'
          - 'nginx-lb2:9113'
```

### Alert Rules

```yaml
# monitoring/rules/ha-alerts.yml
groups:
  - name: ha-critical
    interval: 30s
    rules:
      # Application Down
      - alert: LineBotDown
        expr: up{job="linebot-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "LINE Bot application is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
          
      # High Response Time
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "P95 response time > 2s for 5 minutes"
          
      # Database Connection Issues
      - alert: DatabaseConnectionFailed
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL is down"
          
      # Redis Down
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"
          
      # High Memory Usage
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage (>85%)"
          
      # Disk Space Low
      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space < 10%"
```

## 13. Regular DR Drills

```bash
#!/bin/bash
# scripts/dr-drill.sh
# รัน DR Drill ทุกเดือน

set -euo pipefail

DRILL_DATE=$(date +%Y%m%d)
DRILL_LOG="/var/log/dr-drills/drill-${DRILL_DATE}.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$DRILL_LOG"
}

notify() {
    local message="$1"
    local color="${2:-good}"
    
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" -H 'Content-type: application/json' -d "{
            \"attachments\": [{
                \"color\": \"$color\",
                \"text\": \"$message\"
            }]
        }"
    fi
    
    log "$message"
}

# ============================================
# DR Drill Steps
# ============================================

log "=== Starting DR Drill ${DRILL_DATE} ==="
notify "🔄 DR Drill started on ${DRILL_DATE}" "warning"

START_TIME=$(date +%s)

# Step 1: ตรวจสอบ Backup
log "Step 1: Verifying latest backup..."
LATEST_BACKUP=$(aws s3 ls "s3://${S3_BACKUP_BUCKET}/" --recursive | sort | tail -1 | awk '{print $4}')

if [ -z "$LATEST_BACKUP" ]; then
    notify "❌ DR Drill FAILED: No backup found!" "danger"
    exit 1
fi

BACKUP_AGE_HOURS=$(( ($(date +%s) - $(aws s3 ls "s3://${S3_BACKUP_BUCKET}/${LATEST_BACKUP}" | awk '{print $1" "$2}' | xargs -I{} date -d {} +%s)) / 3600 ))

if [ "$BACKUP_AGE_HOURS" -gt 25 ]; then
    notify "⚠️ DR Drill WARNING: Latest backup is ${BACKUP_AGE_HOURS}h old" "warning"
fi

log "Latest backup: ${LATEST_BACKUP} (${BACKUP_AGE_HOURS}h old)"

# Step 2: ทดสอบ Database Failover
log "Step 2: Testing database failover..."

# หา Primary ปัจจุบัน
CURRENT_PRIMARY=$(docker exec mysql-node1 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e \
    "SELECT MEMBER_HOST FROM performance_schema.replication_group_members WHERE MEMBER_ROLE='PRIMARY'" \
    --silent --skip-column-names 2>/dev/null)

log "Current primary: ${CURRENT_PRIMARY}"

# ทำ Failover โดย Kill Primary
FAILOVER_START=$(date +%s)
log "Killing primary node..."

if [ "$CURRENT_PRIMARY" = "mysql-node1" ]; then
    docker stop mysql-node1
    
    # รอ Failover
    sleep 15
    
    # ตรวจสอบ Primary ใหม่
    NEW_PRIMARY=$(docker exec mysql-node2 mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e \
        "SELECT MEMBER_HOST FROM performance_schema.replication_group_members WHERE MEMBER_ROLE='PRIMARY'" \
        --silent --skip-column-names 2>/dev/null)
    
    FAILOVER_END=$(date +%s)
    FAILOVER_TIME=$((FAILOVER_END - FAILOVER_START))
    
    log "Failover completed in ${FAILOVER_TIME}s"
    log "New primary: ${NEW_PRIMARY}"
    
    if [ -z "$NEW_PRIMARY" ]; then
        notify "❌ Database failover FAILED!" "danger"
    elif [ "$FAILOVER_TIME" -gt 30 ]; then
        notify "⚠️ Database failover took ${FAILOVER_TIME}s (target: <30s)" "warning"
    else
        notify "✅ Database failover completed in ${FAILOVER_TIME}s" "good"
    fi
    
    # Restart mysql-node1
    docker start mysql-node1
    sleep 30
fi

# Step 3: ทดสอบ Application Recovery
log "Step 3: Testing application recovery..."

# ตรวจสอบ Health ก่อน
HEALTH_BEFORE=$(curl -s http://localhost/health | jq -r .status 2>/dev/null || echo "unknown")
log "Health before: $HEALTH_BEFORE"

# Kill App Node 1
docker stop app-node1
sleep 10

# ตรวจสอบ Health ระหว่าง Failover
HEALTH_DURING=$(curl -s http://localhost/health | jq -r .status 2>/dev/null || echo "failed")
log "Health during failover: $HEALTH_DURING"

# Restart App Node 1
docker start app-node1
sleep 20

# ตรวจสอบ Health หลัง Recovery
HEALTH_AFTER=$(curl -s http://localhost/health | jq -r .status 2>/dev/null || echo "unknown")
log "Health after recovery: $HEALTH_AFTER"

if [ "$HEALTH_DURING" = "ok" ] && [ "$HEALTH_AFTER" = "ok" ]; then
    notify "✅ Application failover successful" "good"
else
    notify "❌ Application failover failed (during: $HEALTH_DURING, after: $HEALTH_AFTER)" "danger"
fi

# สรุปผล
END_TIME=$(date +%s)
TOTAL_TIME=$((END_TIME - START_TIME))

SUMMARY="=== DR Drill Summary ===
Date: ${DRILL_DATE}
Total Duration: ${TOTAL_TIME}s
Failover Time: ${FAILOVER_TIME}s
Backup Age: ${BACKUP_AGE_HOURS}h
Application During Failover: ${HEALTH_DURING}
Application After Recovery: ${HEALTH_AFTER}
Log File: ${DRILL_LOG}"

notify "$SUMMARY" "good"
log "DR Drill completed in ${TOTAL_TIME}s"
```

## สรุป

ในบทนี้เราสร้างระบบ High Availability ครบถ้วนสำหรับ LINE Bot:

1. **HA Architecture** - สถาปัตยกรรมที่รองรับ Failure ทุกระดับ
2. **MySQL Group Replication** - Database Cluster 3 nodes พร้อม Auto-failover
3. **PostgreSQL + Patroni** - HA PostgreSQL ด้วย etcd Cluster
4. **MongoDB Replica Set** - HA MongoDB สำหรับ Document Storage
5. **Redis Sentinel** - HA Redis Cache พร้อม Auto-failover
6. **Nginx + keepalived** - HA Load Balancer ด้วย Virtual IP
7. **Health Checks** - Readiness, Liveness, และ Detailed Status
8. **Failover Manager** - Automated Failover พร้อมแจ้งเตือน
9. **Backup Strategy** - RPO/RTO ที่ชัดเจน พร้อม S3 Backup
10. **DR Runbook** - ขั้นตอน Disaster Recovery ที่ชัดเจน
11. **Chaos Engineering** - ทดสอบความแข็งแกร่งของระบบ
12. **Complete Docker Setup** - HA Stack ทั้งหมดใน Docker Compose
13. **Prometheus/Grafana** - Monitoring และ Alerting
14. **DR Drills** - การฝึกซ้อม DR ประจำเดือน
