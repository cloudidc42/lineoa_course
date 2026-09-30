# Part 90: High Availability for LINE Bots

## สารบัญ

1. [HA Architecture Overview](#ha-overview)
2. [Database HA: MySQL Galera Cluster](#mysql-galera)
3. [Database HA: PostgreSQL with Patroni](#patroni)
4. [Database HA: MongoDB Replica Set](#mongodb)
5. [Database HA: Redis Sentinel/Cluster](#redis-ha)
6. [Application HA](#app-ha)
7. [Load Balancer HA](#lb-ha)
8. [Health Checks](#health-checks)
9. [Automated Failover](#failover)
10. [Backup and Recovery](#backup)
11. [RTO and RPO Targets](#rto-rpo)
12. [Runbook for Incidents](#runbook)
13. [Complete HA Setup Guide](#complete-ha)

---

## 1. HA Architecture Overview {#ha-overview}

### ภาพรวม High Availability Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                 LINE Bot High Availability Architecture              │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    LINE Platform                             │    │
│  │                  (Webhook Source)                            │    │
│  └────────────────────────┬────────────────────────────────────┘    │
│                           │ HTTPS                                    │
│                           ▼                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Global Load Balancer (Cloudflare/AWS)           │    │
│  │                   - DDoS Protection                          │    │
│  │                   - SSL Termination                          │    │
│  │                   - Health Checks                            │    │
│  └───────────────┬────────────────────┬────────────────────────┘    │
│                  │                    │                               │
│         ┌────────▼──────┐    ┌────────▼──────┐                      │
│         │  Region A     │    │  Region B     │                      │
│         │  (Primary)    │    │  (Standby)    │                      │
│         │               │    │               │                      │
│  ┌──────┴────────────┐  │    │  ┌────────────┴──────┐              │
│  │  HAProxy (Active) │  │    │  │  HAProxy (Active) │              │
│  └──────┬────────────┘  │    │  └────────────┬──────┘              │
│         │               │    │               │                      │
│  ┌──────┴────────────┐  │    │  ┌────────────┴──────┐              │
│  │  App Servers (3x) │  │    │  │  App Servers (2x) │              │
│  │  - Node.js        │  │    │  │  - Node.js         │              │
│  │  - Stateless      │  │    │  │  - Stateless        │              │
│  └──────┬────────────┘  │    │  └────────────┬──────┘              │
│         │               │    │               │                      │
│  ┌──────┴────────────────────────────────────┴──────┐              │
│  │                  Data Layer                       │              │
│  │                                                   │              │
│  │  ┌──────────────────┐    ┌──────────────────┐    │              │
│  │  │ PostgreSQL       │    │  Redis Sentinel  │    │              │
│  │  │ (Patroni)        │    │  (3 nodes)       │    │              │
│  │  │ Primary + 2 Replicas │  │               │    │              │
│  │  └──────────────────┘    └──────────────────┘    │              │
│  └──────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘

RTO (Recovery Time Objective):  < 30 seconds
RPO (Recovery Point Objective): < 5 seconds
Availability Target:            99.99% (52 minutes downtime/year)
```

### Kubernetes HA Setup

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: line-bot-prod
  labels:
    environment: production
    team: platform

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: line-bot
  namespace: line-bot-prod
  labels:
    app: line-bot
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: line-bot
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  template:
    metadata:
      labels:
        app: line-bot
        version: v1.0.0
    spec:
      # Pod anti-affinity: ไม่ให้ pods อยู่บน node เดียวกัน
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - line-bot
              topologyKey: kubernetes.io/hostname
        
        # Prefer different availability zones
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-southeast-1a
                      - ap-southeast-1b
                      - ap-southeast-1c
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
      
      containers:
        - name: line-bot
          image: your-registry/line-bot:v1.0.0
          imagePullPolicy: Always
          
          ports:
            - containerPort: 3000
              name: http
          
          # Resource limits
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          
          # Startup probe สำหรับ slow startup
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 12  # 60 seconds total
          
          # Environment variables
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
          
          envFrom:
            - secretRef:
                name: line-bot-secrets
            - configMapRef:
                name: line-bot-config
          
          # Security context
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          
          # Volume mounts
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}
      
      # Service account
      serviceAccountName: line-bot

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: line-bot-hpa
  namespace: line-bot-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: line-bot
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: line-bot-pdb
  namespace: line-bot-prod
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: line-bot

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: line-bot-svc
  namespace: line-bot-prod
spec:
  selector:
    app: line-bot
  ports:
    - port: 80
      targetPort: 3000
      name: http
  type: ClusterIP
```

---

## 2. Database HA: MySQL Galera Cluster {#mysql-galera}

### Galera Cluster Setup

```
Galera Cluster Architecture:
┌──────────┐     ┌──────────┐     ┌──────────┐
│  MySQL1  │◄───►│  MySQL2  │◄───►│  MySQL3  │
│ (Primary)│     │(Secondary│     │(Secondary│
└────┬─────┘     └─────┬────┘     └──────┬───┘
     │                 │                 │
     └─────────────────┴─────────────────┘
              wsrep (Galera replication)
              
- Synchronous multi-master replication
- Any node can handle writes
- Automatic failover
- Split-brain protection
```

```yaml
# docker-compose-galera.yml
version: '3.9'

x-mysql-defaults: &mysql-defaults
  image: percona/percona-xtradb-cluster:8.0
  environment:
    MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    MYSQL_DATABASE: linebot_db
    MYSQL_USER: linebot
    MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    XTRABACKUP_PASSWORD: ${XTRABACKUP_PASSWORD}
  volumes:
    - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
  networks:
    - galera-net

services:
  mysql1:
    <<: *mysql-defaults
    container_name: mysql1
    hostname: mysql1
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: linebot_db
      MYSQL_USER: linebot
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      XTRABACKUP_PASSWORD: ${XTRABACKUP_PASSWORD}
      CLUSTER_NAME: linebot_cluster
      CLUSTER_JOIN: ""  # Bootstrap node
    volumes:
      - mysql1_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 30s
      timeout: 10s
      retries: 5

  mysql2:
    <<: *mysql-defaults
    container_name: mysql2
    hostname: mysql2
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: linebot_db
      MYSQL_USER: linebot
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      XTRABACKUP_PASSWORD: ${XTRABACKUP_PASSWORD}
      CLUSTER_NAME: linebot_cluster
      CLUSTER_JOIN: "mysql1"
    volumes:
      - mysql2_data:/var/lib/mysql
    depends_on:
      mysql1:
        condition: service_healthy
    ports:
      - "3307:3306"

  mysql3:
    <<: *mysql-defaults
    container_name: mysql3
    hostname: mysql3
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: linebot_db
      MYSQL_USER: linebot
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      XTRABACKUP_PASSWORD: ${XTRABACKUP_PASSWORD}
      CLUSTER_NAME: linebot_cluster
      CLUSTER_JOIN: "mysql1,mysql2"
    volumes:
      - mysql3_data:/var/lib/mysql
    depends_on:
      mysql1:
        condition: service_healthy
    ports:
      - "3308:3306"

  # ProxySQL - connection pooling และ routing
  proxysql:
    image: proxysql/proxysql:2.6.2
    container_name: proxysql
    ports:
      - "6033:6033"   # MySQL proxy port
      - "6032:6032"   # Admin interface
    volumes:
      - ./docker/proxysql/proxysql.cnf:/etc/proxysql.cnf
    depends_on:
      - mysql1
      - mysql2
      - mysql3
    networks:
      - galera-net

networks:
  galera-net:
    driver: bridge

volumes:
  mysql1_data:
  mysql2_data:
  mysql3_data:
```

### MySQL Configuration

```ini
# docker/mysql/my.cnf
[mysqld]
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib64/galera4/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_address=gcomm://mysql1,mysql2,mysql3
wsrep_cluster_name=linebot_cluster
wsrep_sst_method=xtrabackup-v2

# Galera Node Configuration
wsrep_node_name=mysql1
wsrep_node_address=mysql1

# Galera Synchronization Configuration
wsrep_sst_auth=xtrabackup:${XTRABACKUP_PASSWORD}

# InnoDB Configuration
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_buffer_pool_size=256M
innodb_flush_log_at_trx_commit=0
innodb_log_file_size=128M

# Binary Logging
log_bin=binlog
binlog_format=ROW
binlog_expire_logs_seconds=259200  # 3 days

# Performance
max_connections=500
thread_cache_size=50
table_open_cache=4000
table_definition_cache=2000

# Character Set
character_set_server=utf8mb4
collation_server=utf8mb4_unicode_ci

# Timeouts
wait_timeout=28800
interactive_timeout=28800
net_read_timeout=60
net_write_timeout=60
```

---

## 3. Database HA: PostgreSQL with Patroni {#patroni}

### Patroni Cluster Setup

```
Patroni Architecture:
┌─────────────────────────────────────────────────────────────┐
│                     etcd Cluster (3 nodes)                   │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐       │
│  │  etcd1   │◄──────►│  etcd2   │◄──────►│  etcd3   │       │
│  └──────────┘        └──────────┘        └──────────┘       │
└─────────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│   PostgreSQL1  │   │   PostgreSQL2  │   │   PostgreSQL3  │
│   (Primary)    │──►│   (Replica)    │   │   (Replica)    │
│   [Patroni]    │   │   [Patroni]    │   │   [Patroni]    │
└────────────────┘   └────────────────┘   └────────────────┘
         │
         ▼
┌────────────────┐
│   HAProxy      │
│  - Primary:5432│
│  - Replica:5433│
└────────────────┘
```

```yaml
# docker-compose-patroni.yml
version: '3.9'

services:
  # etcd cluster for distributed config
  etcd1:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd1
    hostname: etcd1
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name=etcd1
      - --initial-advertise-peer-urls=http://etcd1:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://etcd1:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-etcd-cluster
    volumes:
      - etcd1_data:/etcd-data
    networks:
      - patroni-net

  etcd2:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd2
    hostname: etcd2
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name=etcd2
      - --initial-advertise-peer-urls=http://etcd2:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://etcd2:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-etcd-cluster
    volumes:
      - etcd2_data:/etcd-data
    networks:
      - patroni-net

  etcd3:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd3
    hostname: etcd3
    command:
      - /usr/local/bin/etcd
      - --data-dir=/etcd-data
      - --name=etcd3
      - --initial-advertise-peer-urls=http://etcd3:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://etcd3:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-cluster=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=linebot-etcd-cluster
    volumes:
      - etcd3_data:/etcd-data
    networks:
      - patroni-net

  # PostgreSQL + Patroni nodes
  postgres1:
    build:
      context: ./docker/patroni
      dockerfile: Dockerfile
    container_name: postgres1
    hostname: postgres1
    environment:
      PATRONI_NAME: postgres1
      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: postgres1:5432
      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_RESTAPI_CONNECT_ADDRESS: postgres1:8008
      PATRONI_ETCD3_HOSTS: "'etcd1:2379','etcd2:2379','etcd3:2379'"
      PATRONI_SCOPE: linebot-cluster
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: ${POSTGRES_PASSWORD}
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
      PATRONI_USERNAME: linebot
      PATRONI_PASSWORD: ${LINEBOT_DB_PASSWORD}
      PATRONI_DATABASE: linebot_db
    volumes:
      - postgres1_data:/var/lib/postgresql/data
      - ./docker/patroni/patroni.yml:/etc/patroni.yml
    ports:
      - "5432:5432"
      - "8008:8008"
    depends_on:
      - etcd1
      - etcd2
      - etcd3
    networks:
      - patroni-net

  postgres2:
    build:
      context: ./docker/patroni
      dockerfile: Dockerfile
    container_name: postgres2
    hostname: postgres2
    environment:
      PATRONI_NAME: postgres2
      PATRONI_POSTGRESQL_LISTEN: 0.0.0.0:5432
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: postgres2:5432
      PATRONI_RESTAPI_LISTEN: 0.0.0.0:8008
      PATRONI_RESTAPI_CONNECT_ADDRESS: postgres2:8008
      PATRONI_ETCD3_HOSTS: "'etcd1:2379','etcd2:2379','etcd3:2379'"
      PATRONI_SCOPE: linebot-cluster
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: ${POSTGRES_PASSWORD}
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
      PATRONI_USERNAME: linebot
      PATRONI_PASSWORD: ${LINEBOT_DB_PASSWORD}
      PATRONI_DATABASE: linebot_db
    volumes:
      - postgres2_data:/var/lib/postgresql/data
      - ./docker/patroni/patroni.yml:/etc/patroni.yml
    ports:
      - "5433:5432"
      - "8009:8008"
    depends_on:
      - etcd1
      - postgres1
    networks:
      - patroni-net

  # HAProxy for routing
  haproxy:
    image: haproxy:2.8-alpine
    container_name: haproxy
    ports:
      - "5000:5000"   # Primary (write)
      - "5001:5001"   # Replicas (read)
      - "7000:7000"   # HAProxy stats
    volumes:
      - ./docker/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    depends_on:
      - postgres1
      - postgres2
    networks:
      - patroni-net

networks:
  patroni-net:
    driver: bridge

volumes:
  etcd1_data:
  etcd2_data:
  etcd3_data:
  postgres1_data:
  postgres2_data:
```

### HAProxy Configuration

```
# docker/haproxy/haproxy.cfg
global
    maxconn 100
    log stdout format raw local0

defaults
    log     global
    mode    tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

# Stats
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 10s
    stats auth admin:admin

# Primary (write) - ใช้ Patroni REST API สำหรับ health check
frontend primary
    bind *:5000
    default_backend primary_backend

backend primary_backend
    option httpchk GET /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres1 postgres1:5432 maxconn 100 check port 8008
    server postgres2 postgres2:5432 maxconn 100 check port 8009

# Replicas (read) - round robin
frontend replicas
    bind *:5001
    default_backend replica_backend

backend replica_backend
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres1 postgres1:5432 maxconn 100 check port 8008
    server postgres2 postgres2:5432 maxconn 100 check port 8009
```

### Patroni Configuration

```yaml
# docker/patroni/patroni.yml
scope: linebot-cluster
namespace: /service/
name: ${PATRONI_NAME}

restapi:
  listen: ${PATRONI_RESTAPI_LISTEN}
  connect_address: ${PATRONI_RESTAPI_CONNECT_ADDRESS}

etcd3:
  hosts: ${PATRONI_ETCD3_HOSTS}

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1 MB
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 256MB
        effective_cache_size: 768MB
        maintenance_work_mem: 64MB
        checkpoint_completion_target: 0.9
        wal_buffers: 16MB
        default_statistics_target: 100
        random_page_cost: 1.1
        effective_io_concurrency: 200
        min_wal_size: 1GB
        max_wal_size: 4GB
        max_worker_processes: 4
        max_parallel_workers_per_gather: 2
        max_parallel_workers: 4
        wal_level: replica
        hot_standby: on
        wal_keep_size: 1GB
        max_wal_senders: 10
        max_replication_slots: 10
        synchronous_commit: on
        
  initdb:
    - encoding: UTF8
    - data-checksums
    
  pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: ${PATRONI_POSTGRESQL_LISTEN}
  connect_address: ${PATRONI_POSTGRESQL_CONNECT_ADDRESS}
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/16/bin
  
  authentication:
    replication:
      username: ${PATRONI_REPLICATION_USERNAME}
      password: ${PATRONI_REPLICATION_PASSWORD}
    superuser:
      username: ${PATRONI_SUPERUSER_USERNAME}
      password: ${PATRONI_SUPERUSER_PASSWORD}

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

---

## 4. Database HA: MongoDB Replica Set {#mongodb}

### MongoDB Replica Set Setup

```yaml
# docker-compose-mongodb.yml
version: '3.9'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    hostname: mongo1
    command: mongod --replSet rs0 --keyFile /etc/mongo-keyfile --bind_ip_all
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: linebot_db
    ports:
      - "27017:27017"
    volumes:
      - mongo1_data:/data/db
      - ./docker/mongodb/keyfile:/etc/mongo-keyfile:ro
      - ./docker/mongodb/init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - mongo-net
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    hostname: mongo2
    command: mongod --replSet rs0 --keyFile /etc/mongo-keyfile --bind_ip_all
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    ports:
      - "27018:27017"
    volumes:
      - mongo2_data:/data/db
      - ./docker/mongodb/keyfile:/etc/mongo-keyfile:ro
    depends_on:
      mongo1:
        condition: service_healthy
    networks:
      - mongo-net

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    hostname: mongo3
    command: mongod --replSet rs0 --keyFile /etc/mongo-keyfile --bind_ip_all
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    ports:
      - "27019:27017"
    volumes:
      - mongo3_data:/data/db
      - ./docker/mongodb/keyfile:/etc/mongo-keyfile:ro
    depends_on:
      mongo1:
        condition: service_healthy
    networks:
      - mongo-net

  # MongoDB initialization
  mongo-setup:
    image: mongo:7.0
    depends_on:
      - mongo1
      - mongo2
      - mongo3
    volumes:
      - ./docker/mongodb/setup.sh:/setup.sh:ro
    entrypoint: ["/bin/bash", "/setup.sh"]
    networks:
      - mongo-net

networks:
  mongo-net:
    driver: bridge

volumes:
  mongo1_data:
  mongo2_data:
  mongo3_data:
```

### MongoDB Replica Set Initialization

```javascript
// docker/mongodb/init.js
// สร้าง Replica Set

const adminDb = db.getSiblingDB('admin')

// Initiate replica set
rs.initiate({
  _id: 'rs0',
  version: 1,
  members: [
    { _id: 0, host: 'mongo1:27017', priority: 2 },
    { _id: 1, host: 'mongo2:27017', priority: 1 },
    { _id: 2, host: 'mongo3:27017', priority: 1 }
  ]
})

// รอให้ replica set ready
sleep(5000)

// สร้าง users
adminDb.createUser({
  user: 'linebot',
  pwd: process.env.MONGO_LINEBOT_PASSWORD,
  roles: [
    { role: 'readWrite', db: 'linebot_db' }
  ]
})

// สร้าง indexes
const linebotDb = db.getSiblingDB('linebot_db')

linebotDb.users.createIndex({ lineUserId: 1 }, { unique: true })
linebotDb.users.createIndex({ updatedAt: -1 })
linebotDb.messages.createIndex({ userId: 1, timestamp: -1 })
linebotDb.messages.createIndex({ timestamp: -1 })
linebotDb.conversations.createIndex({ userId: 1, status: 1 })
linebotDb.conversations.createIndex({ updatedAt: -1 })
```

### MongoDB Read Preference

```typescript
// lib/mongodb.ts
import { MongoClient, ReadPreference, WriteConcern } from 'mongodb'

const MONGODB_URI = process.env.MONGODB_URI!

// Connection string with replica set
// mongodb://mongo1:27017,mongo2:27017,mongo3:27017/linebot_db?replicaSet=rs0

const client = new MongoClient(MONGODB_URI, {
  replicaSet: 'rs0',
  readPreference: ReadPreference.PRIMARY_PREFERRED,
  writeConcern: new WriteConcern('majority', 5000),
  readConcern: { level: 'majority' },
  maxPoolSize: 50,
  minPoolSize: 5,
  maxIdleTimeMS: 60000,
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 10000,
  heartbeatFrequencyMS: 10000,
  retryWrites: true,
  retryReads: true
})

export const db = {
  primary: client.db('linebot_db'),
  
  // Read from nearest secondary for analytics
  analytics: client.db('linebot_db').withReadPreference(
    ReadPreference.SECONDARY_PREFERRED
  )
}

// Monitor replica set events
client.on('serverDescriptionChanged', (event) => {
  console.log('MongoDB topology changed:', event.newDescription.type)
})

client.on('serverHeartbeatFailed', (event) => {
  console.error('MongoDB heartbeat failed:', event.connectionId)
})
```

---

## 5. Database HA: Redis Sentinel/Cluster {#redis-ha}

### Redis Sentinel Setup (สำหรับ < 100GB data)

```yaml
# docker-compose-redis-sentinel.yml
version: '3.9'

services:
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    command: |
      redis-server 
      --requirepass ${REDIS_PASSWORD}
      --masterauth ${REDIS_PASSWORD}
      --appendonly yes
      --appendfsync everysec
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --save 60 10000
    ports:
      - "6379:6379"
    volumes:
      - redis_master_data:/data
    networks:
      - redis-net
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis-replica1:
    image: redis:7-alpine
    container_name: redis-replica1
    command: |
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --masterauth ${REDIS_PASSWORD}
      --replicaof redis-master 6379
      --appendonly yes
      --replica-read-only yes
    ports:
      - "6380:6379"
    volumes:
      - redis_replica1_data:/data
    depends_on:
      redis-master:
        condition: service_healthy
    networks:
      - redis-net

  redis-replica2:
    image: redis:7-alpine
    container_name: redis-replica2
    command: |
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --masterauth ${REDIS_PASSWORD}
      --replicaof redis-master 6379
      --appendonly yes
      --replica-read-only yes
    ports:
      - "6381:6379"
    volumes:
      - redis_replica2_data:/data
    depends_on:
      redis-master:
        condition: service_healthy
    networks:
      - redis-net

  sentinel1:
    image: redis:7-alpine
    container_name: sentinel1
    command: redis-sentinel /etc/sentinel.conf
    volumes:
      - ./docker/redis/sentinel.conf:/etc/sentinel.conf
    ports:
      - "26379:26379"
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2
    networks:
      - redis-net

  sentinel2:
    image: redis:7-alpine
    container_name: sentinel2
    command: redis-sentinel /etc/sentinel.conf
    volumes:
      - ./docker/redis/sentinel.conf:/etc/sentinel.conf
    ports:
      - "26380:26379"
    depends_on:
      - redis-master
    networks:
      - redis-net

  sentinel3:
    image: redis:7-alpine
    container_name: sentinel3
    command: redis-sentinel /etc/sentinel.conf
    volumes:
      - ./docker/redis/sentinel.conf:/etc/sentinel.conf
    ports:
      - "26381:26379"
    depends_on:
      - redis-master
    networks:
      - redis-net

networks:
  redis-net:
    driver: bridge

volumes:
  redis_master_data:
  redis_replica1_data:
  redis_replica2_data:
```

### Sentinel Configuration

```
# docker/redis/sentinel.conf
port 26379
sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster ${REDIS_PASSWORD}
sentinel down-after-milliseconds mymaster 10000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel announce-ip ${SENTINEL_IP}
sentinel announce-port 26379
```

### Redis Sentinel Client (Node.js)

```typescript
// lib/redis.ts
import { Redis } from 'ioredis'

// ใช้ Sentinel สำหรับ HA
export const redis = new Redis({
  sentinels: [
    { host: 'sentinel1', port: 26379 },
    { host: 'sentinel2', port: 26380 },
    { host: 'sentinel3', port: 26381 }
  ],
  name: 'mymaster',
  password: process.env.REDIS_PASSWORD,
  sentinelPassword: process.env.REDIS_PASSWORD,
  enableReadyCheck: true,
  maxRetriesPerRequest: 3,
  retryStrategy: (times) => {
    if (times > 10) return null // Stop retrying after 10 attempts
    return Math.min(times * 100, 3000) // Exponential backoff
  },
  reconnectOnError: (err) => {
    const targetError = 'READONLY'
    if (err.message.includes(targetError)) {
      // Reconnect when connecting to a slave that becomes a master
      return true
    }
    return false
  }
})

redis.on('error', (err) => {
  console.error('Redis connection error:', err.message)
})

redis.on('connect', () => {
  console.log('Redis connected')
})

redis.on('+sentinel', (sinfo) => {
  console.log('Sentinel connected:', sinfo.host, sinfo.port)
})

redis.on('+slave', (sinfo) => {
  console.log('New slave detected:', sinfo.host, sinfo.port)
})

redis.on('+switch-master', (masterName, oldAddr, newAddr) => {
  console.log(`Failover: ${masterName} switched from ${oldAddr} to ${newAddr}`)
})

// Redis Cluster สำหรับ > 100GB data
export const redisCluster = new Redis.Cluster(
  [
    { host: 'redis-node1', port: 7000 },
    { host: 'redis-node2', port: 7001 },
    { host: 'redis-node3', port: 7002 }
  ],
  {
    clusterRetryStrategy: (times) => Math.min(times * 100, 3000),
    enableOfflineQueue: true,
    scaleReads: 'slave',
    redisOptions: {
      password: process.env.REDIS_PASSWORD
    }
  }
)
```

---

## 6. Application HA {#app-ha}

### Graceful Shutdown Handler

```typescript
// src/server.ts
import Fastify from 'fastify'
import { redis } from '@/lib/redis'
import { prisma } from '@/lib/prisma'

const app = Fastify({
  logger: {
    level: process.env.LOG_LEVEL ?? 'info'
  }
})

// Track in-flight requests
let inFlightRequests = 0

app.addHook('onRequest', (request, reply, done) => {
  inFlightRequests++
  done()
})

app.addHook('onResponse', (request, reply, done) => {
  inFlightRequests--
  done()
})

// Health endpoints
app.get('/health/live', async () => {
  return { status: 'alive', timestamp: new Date().toISOString() }
})

app.get('/health/ready', async () => {
  // ตรวจสอบ dependencies
  const checks = await Promise.allSettled([
    prisma.$queryRaw`SELECT 1`,
    redis.ping()
  ])
  
  const allReady = checks.every(c => c.status === 'fulfilled')
  
  if (!allReady) {
    const failures = checks
      .filter(c => c.status === 'rejected')
      .map((c, i) => ({ 
        service: ['database', 'redis'][i], 
        error: (c as PromiseRejectedResult).reason?.message 
      }))
    
    return app.httpErrors.serviceUnavailable(JSON.stringify({ failures }))
  }
  
  return { 
    status: 'ready',
    timestamp: new Date().toISOString(),
    checks: { database: 'ok', redis: 'ok' }
  }
})

// Graceful shutdown
async function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}, starting graceful shutdown...`)
  
  // Stop accepting new connections
  await app.close()
  console.log('Server closed to new connections')
  
  // Wait for in-flight requests (max 30 seconds)
  const maxWait = 30000
  const startTime = Date.now()
  
  while (inFlightRequests > 0 && Date.now() - startTime < maxWait) {
    console.log(`Waiting for ${inFlightRequests} in-flight requests...`)
    await new Promise(resolve => setTimeout(resolve, 1000))
  }
  
  if (inFlightRequests > 0) {
    console.warn(`Forcing shutdown with ${inFlightRequests} requests still in-flight`)
  }
  
  // Close database connections
  await prisma.$disconnect()
  console.log('Database disconnected')
  
  await redis.quit()
  console.log('Redis disconnected')
  
  console.log('Graceful shutdown complete')
  process.exit(0)
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'))
process.on('SIGINT', () => gracefulShutdown('SIGINT'))

// Start server
await app.listen({ port: 3000, host: '0.0.0.0' })
```

### Circuit Breaker Pattern

```typescript
// lib/circuit-breaker.ts
type CircuitState = 'closed' | 'open' | 'half-open'

interface CircuitBreakerOptions {
  failureThreshold: number   // จำนวน failures ก่อน open
  successThreshold: number   // จำนวน successes ก่อน close
  timeout: number            // milliseconds ก่อนลอง half-open
  monitorWindow: number      // window สำหรับนับ failures
}

export class CircuitBreaker<T> {
  private state: CircuitState = 'closed'
  private failures: number = 0
  private successes: number = 0
  private lastFailureTime: number = 0
  private requestCount: number = 0
  private failureTimestamps: number[] = []

  constructor(
    private readonly fn: (...args: unknown[]) => Promise<T>,
    private readonly options: CircuitBreakerOptions = {
      failureThreshold: 5,
      successThreshold: 2,
      timeout: 60000,
      monitorWindow: 60000
    }
  ) {}

  async execute(...args: unknown[]): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.options.timeout) {
        console.log('Circuit breaker: attempting reset (half-open)')
        this.state = 'half-open'
        this.successes = 0
      } else {
        throw new Error('Circuit breaker is OPEN - service unavailable')
      }
    }

    try {
      const result = await this.fn(...args)
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess(): void {
    this.failures = 0
    
    if (this.state === 'half-open') {
      this.successes++
      if (this.successes >= this.options.successThreshold) {
        console.log('Circuit breaker: CLOSED (recovered)')
        this.state = 'closed'
        this.successes = 0
      }
    }
  }

  private onFailure(): void {
    this.lastFailureTime = Date.now()
    
    // Remove old failures outside window
    const windowStart = Date.now() - this.options.monitorWindow
    this.failureTimestamps = this.failureTimestamps.filter(t => t > windowStart)
    this.failureTimestamps.push(Date.now())
    
    if (this.failureTimestamps.length >= this.options.failureThreshold) {
      if (this.state !== 'open') {
        console.error('Circuit breaker: OPENED (too many failures)')
        this.state = 'open'
      }
    }
    
    if (this.state === 'half-open') {
      console.log('Circuit breaker: REOPENED (half-open failed)')
      this.state = 'open'
    }
  }

  getState(): CircuitState {
    return this.state
  }

  getStats() {
    return {
      state: this.state,
      failures: this.failureTimestamps.length,
      successes: this.successes,
      lastFailureTime: this.lastFailureTime
    }
  }
}

// การใช้งาน
const lineApiBreaker = new CircuitBreaker(
  async (token: string, message: unknown) => {
    const response = await fetch('https://api.line.me/v2/bot/message/reply', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}` },
      body: JSON.stringify(message)
    })
    if (!response.ok) throw new Error(`LINE API error: ${response.status}`)
    return response.json()
  },
  {
    failureThreshold: 5,
    successThreshold: 2,
    timeout: 30000,
    monitorWindow: 60000
  }
)
```

---

## 7. Load Balancer HA {#lb-ha}

### HAProxy Configuration สำหรับ LINE Bot

```
# haproxy.cfg
global
    log stdout format raw local0 info
    maxconn 50000
    nbthread 4
    cpu-map auto:1/1-4 0-3

defaults
    log     global
    option  httplog
    option  dontlognull
    option  forwardfor
    option  http-server-close
    retries 3
    timeout connect 10s
    timeout client 60s
    timeout server 60s
    timeout queue 60s
    timeout http-request 10s
    timeout http-keep-alive 10s

# Stats page
frontend stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:${HAPROXY_STATS_PASSWORD}
    stats show-legends
    stats show-node

# LINE Bot webhook endpoint
frontend line_bot_frontend
    bind *:443 ssl crt /etc/ssl/line-bot.pem
    bind *:80
    mode http
    
    # Redirect HTTP to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }
    
    # LINE webhook verification
    acl is_webhook path_beg /webhook
    acl has_line_signature hdr_sub(X-Line-Signature) -m found
    
    # Block requests without LINE signature on webhook endpoint
    http-request deny if is_webhook !has_line_signature
    
    # Rate limiting
    stick-table type ip size 200k expire 30s store http_req_rate(10s),conn_cur
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
    
    default_backend line_bot_backend

backend line_bot_backend
    mode http
    balance leastconn
    option httpchk GET /health/ready
    http-check expect status 200
    
    # Retry on failure
    retries 3
    retry-on all-retryable-errors
    
    # Session persistence สำหรับ WebSocket
    cookie SRVID insert indirect nocache
    
    default-server inter 5s fall 2 rise 2 slowstart 60s
    server app1 app1:3000 check cookie app1
    server app2 app2:3000 check cookie app2  
    server app3 app3:3000 check cookie app3

# LIFF App
frontend liff_frontend
    bind *:443 ssl crt /etc/ssl/liff.pem
    mode http
    
    redirect scheme https code 301 if !{ ssl_fc }
    
    default_backend liff_backend

backend liff_backend
    mode http
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    
    server liff1 liff-app1:3000 check
    server liff2 liff-app2:3000 check
```

---

## 8. Health Checks {#health-checks}

### Comprehensive Health Check System

```typescript
// src/health/health.service.ts
import { redis } from '@/lib/redis'
import { prisma } from '@/lib/prisma'

interface HealthCheck {
  name: string
  status: 'healthy' | 'degraded' | 'unhealthy'
  latency: number
  error?: string
  details?: Record<string, unknown>
}

interface SystemHealth {
  status: 'healthy' | 'degraded' | 'unhealthy'
  timestamp: string
  version: string
  uptime: number
  checks: HealthCheck[]
  metrics: SystemMetrics
}

interface SystemMetrics {
  memory: {
    heapUsed: number
    heapTotal: number
    rss: number
    external: number
  }
  cpu: {
    user: number
    system: number
  }
  eventLoop: {
    lag: number
  }
}

export class HealthService {
  async getFullHealth(): Promise<SystemHealth> {
    const [dbCheck, redisCheck, lineApiCheck, memCheck] = await Promise.all([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkLineAPI(),
      this.checkMemory()
    ])

    const checks = [dbCheck, redisCheck, lineApiCheck, memCheck]
    
    const overallStatus = checks.every(c => c.status === 'healthy') ? 'healthy' :
                         checks.some(c => c.status === 'unhealthy') ? 'unhealthy' : 'degraded'

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION ?? '1.0.0',
      uptime: process.uptime(),
      checks,
      metrics: this.getSystemMetrics()
    }
  }

  private async checkDatabase(): Promise<HealthCheck> {
    const start = Date.now()
    try {
      await prisma.$queryRaw`SELECT 1 as health_check`
      const latency = Date.now() - start
      
      // ตรวจสอบ connection pool
      const metrics = await prisma.$metrics.json()
      
      return {
        name: 'database',
        status: latency < 1000 ? 'healthy' : 'degraded',
        latency,
        details: {
          poolSize: metrics.counters?.find(c => c.key === 'prisma_pool_connections_open')?.value,
          queryCount: metrics.counters?.find(c => c.key === 'prisma_queries_total')?.value
        }
      }
    } catch (error) {
      return {
        name: 'database',
        status: 'unhealthy',
        latency: Date.now() - start,
        error: (error as Error).message
      }
    }
  }

  private async checkRedis(): Promise<HealthCheck> {
    const start = Date.now()
    try {
      const result = await redis.ping()
      const latency = Date.now() - start
      
      if (result !== 'PONG') throw new Error('Unexpected ping response')
      
      // ตรวจสอบ memory usage
      const info = await redis.info('memory')
      const usedMemory = parseInt(info.match(/used_memory:(\d+)/)?.[1] ?? '0')
      const maxMemory = parseInt(info.match(/maxmemory:(\d+)/)?.[1] ?? '0')
      const usagePercent = maxMemory > 0 ? (usedMemory / maxMemory) * 100 : 0
      
      return {
        name: 'redis',
        status: usagePercent < 90 ? 'healthy' : 'degraded',
        latency,
        details: {
          usedMemoryMB: Math.round(usedMemory / 1024 / 1024),
          memoryUsagePercent: Math.round(usagePercent)
        }
      }
    } catch (error) {
      return {
        name: 'redis',
        status: 'unhealthy',
        latency: Date.now() - start,
        error: (error as Error).message
      }
    }
  }

  private async checkLineAPI(): Promise<HealthCheck> {
    const start = Date.now()
    try {
      const response = await fetch('https://api.line.me/v2/bot/info', {
        headers: { 'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}` },
        signal: AbortSignal.timeout(5000)
      })
      
      const latency = Date.now() - start
      
      if (!response.ok) {
        return {
          name: 'line_api',
          status: response.status === 401 ? 'unhealthy' : 'degraded',
          latency,
          error: `HTTP ${response.status}`
        }
      }

      return {
        name: 'line_api',
        status: latency < 2000 ? 'healthy' : 'degraded',
        latency
      }
    } catch (error) {
      return {
        name: 'line_api',
        status: 'degraded', // LINE API down ไม่ทำให้ app หยุดทำงาน
        latency: Date.now() - start,
        error: (error as Error).message
      }
    }
  }

  private async checkMemory(): Promise<HealthCheck> {
    const memUsage = process.memoryUsage()
    const heapPercent = (memUsage.heapUsed / memUsage.heapTotal) * 100

    return {
      name: 'memory',
      status: heapPercent < 80 ? 'healthy' : heapPercent < 90 ? 'degraded' : 'unhealthy',
      latency: 0,
      details: {
        heapUsedMB: Math.round(memUsage.heapUsed / 1024 / 1024),
        heapTotalMB: Math.round(memUsage.heapTotal / 1024 / 1024),
        heapPercent: Math.round(heapPercent),
        rssMB: Math.round(memUsage.rss / 1024 / 1024)
      }
    }
  }

  private getSystemMetrics(): SystemMetrics {
    const memUsage = process.memoryUsage()
    const cpuUsage = process.cpuUsage()

    return {
      memory: {
        heapUsed: memUsage.heapUsed,
        heapTotal: memUsage.heapTotal,
        rss: memUsage.rss,
        external: memUsage.external
      },
      cpu: {
        user: cpuUsage.user,
        system: cpuUsage.system
      },
      eventLoop: {
        lag: 0 // ต้องใช้ library เช่น @newrelic/nr-node-metrics
      }
    }
  }
}
```

---

## 9. Automated Failover {#failover}

### Failover Manager

```typescript
// src/failover/failover.manager.ts
import EventEmitter from 'events'

interface ServiceConfig {
  name: string
  checkFn: () => Promise<boolean>
  checkInterval: number
  failThreshold: number
  recoverThreshold: number
  onFail?: () => Promise<void>
  onRecover?: () => Promise<void>
}

interface ServiceStatus {
  name: string
  isHealthy: boolean
  consecutiveFailures: number
  consecutiveSuccesses: number
  lastCheckAt: Date
  failedAt?: Date
  recoveredAt?: Date
}

export class FailoverManager extends EventEmitter {
  private services: Map<string, ServiceStatus> = new Map()
  private timers: Map<string, ReturnType<typeof setInterval>> = new Map()

  registerService(config: ServiceConfig): void {
    this.services.set(config.name, {
      name: config.name,
      isHealthy: true,
      consecutiveFailures: 0,
      consecutiveSuccesses: 0,
      lastCheckAt: new Date()
    })

    const timer = setInterval(async () => {
      try {
        const isHealthy = await config.checkFn()
        await this.updateStatus(config, isHealthy)
      } catch (error) {
        await this.updateStatus(config, false)
      }
    }, config.checkInterval)

    this.timers.set(config.name, timer)
  }

  private async updateStatus(config: ServiceConfig, isHealthy: boolean): Promise<void> {
    const status = this.services.get(config.name)
    if (!status) return

    status.lastCheckAt = new Date()

    if (!isHealthy) {
      status.consecutiveSuccesses = 0
      status.consecutiveFailures++

      if (status.isHealthy && status.consecutiveFailures >= config.failThreshold) {
        status.isHealthy = false
        status.failedAt = new Date()

        console.error(`[FailoverManager] Service FAILED: ${config.name}`)
        this.emit('service:failed', status)

        if (config.onFail) {
          await config.onFail()
        }
      }
    } else {
      status.consecutiveFailures = 0
      status.consecutiveSuccesses++

      if (!status.isHealthy && status.consecutiveSuccesses >= config.recoverThreshold) {
        status.isHealthy = true
        status.recoveredAt = new Date()

        const downtime = status.failedAt 
          ? Date.now() - status.failedAt.getTime() 
          : 0

        console.log(`[FailoverManager] Service RECOVERED: ${config.name} (downtime: ${downtime}ms)`)
        this.emit('service:recovered', { ...status, downtimeMs: downtime })

        if (config.onRecover) {
          await config.onRecover()
        }
      }
    }

    this.services.set(config.name, status)
  }

  getStatus(): ServiceStatus[] {
    return Array.from(this.services.values())
  }

  stop(): void {
    for (const timer of this.timers.values()) {
      clearInterval(timer)
    }
    this.timers.clear()
  }
}

// การใช้งาน
const failoverManager = new FailoverManager()

failoverManager.registerService({
  name: 'primary-database',
  checkFn: async () => {
    try {
      await prisma.$queryRaw`SELECT 1`
      return true
    } catch {
      return false
    }
  },
  checkInterval: 5000,
  failThreshold: 3,
  recoverThreshold: 2,
  onFail: async () => {
    console.log('Switching to replica database...')
    await switchToReplica()
  },
  onRecover: async () => {
    console.log('Switching back to primary database...')
    await switchToPrimary()
  }
})

failoverManager.on('service:failed', (status) => {
  // ส่ง alert ทาง LINE
  sendLineAlert(`⚠️ Service Failed: ${status.name}\nTime: ${status.failedAt}`)
})

failoverManager.on('service:recovered', ({ name, downtimeMs }) => {
  sendLineAlert(`✅ Service Recovered: ${name}\nDowntime: ${Math.round(downtimeMs / 1000)}s`)
})
```

---

## 10. Backup and Recovery {#backup}

### Backup Script

```bash
#!/bin/bash
# scripts/backup.sh
# Automated backup script สำหรับ LINE Bot

set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/${TIMESTAMP}"
S3_BUCKET="${BACKUP_S3_BUCKET:-linebot-backups}"
RETENTION_DAYS=30

mkdir -p "${BACKUP_DIR}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# === PostgreSQL Backup ===
backup_postgres() {
    log "Starting PostgreSQL backup..."
    
    # Full dump
    PGPASSWORD="${POSTGRES_PASSWORD}" pg_dump \
        -h "${POSTGRES_HOST}" \
        -U "${POSTGRES_USER}" \
        -d "${POSTGRES_DB}" \
        -F c \
        -Z 9 \
        --no-privileges \
        --no-owner \
        -f "${BACKUP_DIR}/postgres_${TIMESTAMP}.dump"
    
    # WAL backup (continuous)
    if command -v wal-g &> /dev/null; then
        WALG_S3_PREFIX="s3://${S3_BUCKET}/wal-g" \
        PGHOST="${POSTGRES_HOST}" \
        PGUSER="${POSTGRES_USER}" \
        PGPASSWORD="${POSTGRES_PASSWORD}" \
        wal-g backup-push "/var/lib/postgresql/data"
    fi
    
    log "PostgreSQL backup completed: ${BACKUP_DIR}/postgres_${TIMESTAMP}.dump"
}

# === Redis Backup ===
backup_redis() {
    log "Starting Redis backup..."
    
    # Trigger BGSAVE
    redis-cli -h "${REDIS_HOST}" -a "${REDIS_PASSWORD}" BGSAVE
    
    # Wait for save to complete
    sleep 5
    
    # Copy RDB file
    redis-cli -h "${REDIS_HOST}" -a "${REDIS_PASSWORD}" CONFIG GET dir | \
        awk 'NR==2 {print $0}' | \
        xargs -I{} rsync -av {}/dump.rdb "${BACKUP_DIR}/redis_${TIMESTAMP}.rdb"
    
    log "Redis backup completed"
}

# === Upload to S3 ===
upload_to_s3() {
    log "Uploading backups to S3..."
    
    aws s3 sync \
        "${BACKUP_DIR}" \
        "s3://${S3_BUCKET}/${TIMESTAMP}/" \
        --storage-class STANDARD_IA \
        --sse AES256
    
    log "Upload completed to s3://${S3_BUCKET}/${TIMESTAMP}/"
}

# === Cleanup Old Backups ===
cleanup_old_backups() {
    log "Cleaning up backups older than ${RETENTION_DAYS} days..."
    
    # Local cleanup
    find /backup -type d -mtime +${RETENTION_DAYS} -exec rm -rf {} + 2>/dev/null || true
    
    # S3 lifecycle policy handles remote cleanup
    log "Local cleanup completed"
}

# === Verify Backup ===
verify_backup() {
    log "Verifying PostgreSQL backup..."
    
    PGPASSWORD="${POSTGRES_PASSWORD}" pg_restore \
        --list \
        "${BACKUP_DIR}/postgres_${TIMESTAMP}.dump" > /dev/null
    
    if [ $? -eq 0 ]; then
        log "Backup verification: PASSED"
    else
        log "Backup verification: FAILED"
        send_alert "Backup verification failed for ${TIMESTAMP}"
        exit 1
    fi
}

# === Send Alert ===
send_alert() {
    local message="$1"
    
    curl -s -X POST \
        "https://api.line.me/v2/bot/message/push" \
        -H "Authorization: Bearer ${LINE_CHANNEL_ACCESS_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "{
            \"to\": \"${LINE_ALERT_USER_ID}\",
            \"messages\": [{\"type\": \"text\", \"text\": \"🔔 Backup Alert\\n${message}\"}]
        }"
}

# === Main ===
main() {
    log "Starting backup process..."
    
    backup_postgres
    backup_redis
    verify_backup
    upload_to_s3
    cleanup_old_backups
    
    log "Backup process completed successfully"
    send_alert "✅ Backup completed successfully\nTimestamp: ${TIMESTAMP}\nSize: $(du -sh ${BACKUP_DIR} | cut -f1)"
}

main "$@"
```

### Recovery Script

```bash
#!/bin/bash
# scripts/restore.sh
# Restore script สำหรับ LINE Bot

set -euo pipefail

BACKUP_TIMESTAMP="${1:-}"
S3_BUCKET="${BACKUP_S3_BUCKET:-linebot-backups}"

if [ -z "${BACKUP_TIMESTAMP}" ]; then
    echo "Usage: $0 <backup_timestamp>"
    echo "Available backups:"
    aws s3 ls "s3://${S3_BUCKET}/" | awk '{print $2}' | sed 's/\///'
    exit 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

RESTORE_DIR="/tmp/restore_${BACKUP_TIMESTAMP}"
mkdir -p "${RESTORE_DIR}"

# Download backup from S3
log "Downloading backup from S3..."
aws s3 sync \
    "s3://${S3_BUCKET}/${BACKUP_TIMESTAMP}/" \
    "${RESTORE_DIR}/"

# Restore PostgreSQL
restore_postgres() {
    log "Restoring PostgreSQL..."
    
    DUMP_FILE="${RESTORE_DIR}/postgres_${BACKUP_TIMESTAMP}.dump"
    
    if [ ! -f "${DUMP_FILE}" ]; then
        log "ERROR: Backup file not found: ${DUMP_FILE}"
        exit 1
    fi
    
    # Drop existing connections
    PGPASSWORD="${POSTGRES_PASSWORD}" psql \
        -h "${POSTGRES_HOST}" \
        -U postgres \
        -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='${POSTGRES_DB}' AND pid <> pg_backend_pid();"
    
    # Drop and recreate database
    PGPASSWORD="${POSTGRES_PASSWORD}" psql \
        -h "${POSTGRES_HOST}" \
        -U postgres \
        -c "DROP DATABASE IF EXISTS ${POSTGRES_DB}_old;"
    
    PGPASSWORD="${POSTGRES_PASSWORD}" psql \
        -h "${POSTGRES_HOST}" \
        -U postgres \
        -c "ALTER DATABASE ${POSTGRES_DB} RENAME TO ${POSTGRES_DB}_old;"
    
    PGPASSWORD="${POSTGRES_PASSWORD}" psql \
        -h "${POSTGRES_HOST}" \
        -U postgres \
        -c "CREATE DATABASE ${POSTGRES_DB};"
    
    # Restore
    PGPASSWORD="${POSTGRES_PASSWORD}" pg_restore \
        -h "${POSTGRES_HOST}" \
        -U "${POSTGRES_USER}" \
        -d "${POSTGRES_DB}" \
        -v \
        "${DUMP_FILE}"
    
    log "PostgreSQL restore completed"
}

restore_postgres

log "Restore completed from backup: ${BACKUP_TIMESTAMP}"
```

---

## 11. RTO and RPO Targets {#rto-rpo}

### SLA Definition

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Service Level Objectives (SLOs)                   │
├─────────────────────┬───────────────────────────────────────────────┤
│  Metric             │ Target                                         │
├─────────────────────┼───────────────────────────────────────────────┤
│  Availability       │ 99.95% (monthly ≈ 22 min downtime)             │
│  RTO                │ < 5 minutes (for most failure scenarios)       │
│  RPO                │ < 1 minute (for database)                      │
│  MTTR               │ < 15 minutes                                   │
│  MTBF               │ > 30 days                                      │
├─────────────────────┼───────────────────────────────────────────────┤
│  P50 Latency        │ < 100ms                                        │
│  P95 Latency        │ < 500ms                                        │
│  P99 Latency        │ < 2000ms                                       │
├─────────────────────┼───────────────────────────────────────────────┤
│  Error Rate         │ < 0.1%                                         │
│  Webhook Success    │ > 99.9%                                        │
└─────────────────────┴───────────────────────────────────────────────┘

ประเภทของ Failures:
1. Single Pod Failure       → RTO: ~30s (K8s auto-restart)
2. Node Failure             → RTO: ~60s (K8s reschedule)
3. Database Failover        → RTO: ~30s (Patroni auto-failover)
4. Redis Failover           → RTO: ~15s (Sentinel auto-failover)
5. Complete Region Failure  → RTO: ~5min (Manual DNS switch)
6. Data Corruption          → RTO: ~30min (Restore from backup)
```

---

## 12. Runbook for Incidents {#runbook}

### Incident Response Runbook

```markdown
# Runbook: LINE Bot Incident Response

## Severity Levels

| Severity | Description | Response Time | Escalation |
|----------|-------------|---------------|------------|
| P1 | Bot ไม่ตอบ (complete outage) | < 5 min | Immediate |
| P2 | Bot ช้ามาก หรือ error rate > 5% | < 15 min | After 15 min |
| P3 | Feature บางอย่าง degraded | < 1 hour | After 1 hour |
| P4 | Minor issues | < 1 day | None |

## P1: Complete Bot Outage

### ขั้นตอนที่ 1: Detect (0-5 min)
1. ตรวจสอบ monitoring alerts
2. ตรวจสอบ Grafana dashboard
3. ลอง webhook ด้วยตนเอง:
   ```
   curl -X POST https://your-domain.com/health/ready
   ```

### ขั้นตอนที่ 2: Triage (5-10 min)
1. ตรวจสอบ pods:
   ```
   kubectl get pods -n line-bot-prod
   kubectl describe pod <failing-pod> -n line-bot-prod
   ```

2. ตรวจสอบ logs:
   ```
   kubectl logs -n line-bot-prod -l app=line-bot --tail=100
   ```

3. ตรวจสอบ database:
   ```
   kubectl exec -it patroni-leader -n databases -- patronictl list
   ```

4. ตรวจสอบ Redis:
   ```
   redis-cli -h redis-master -a $REDIS_PASSWORD INFO replication
   ```

### ขั้นตอนที่ 3: Mitigate (10-30 min)

#### ถ้า Pods ไม่ทำงาน:
```bash
# Restart deployment
kubectl rollout restart deployment/line-bot -n line-bot-prod

# ถ้าต้อง rollback
kubectl rollout undo deployment/line-bot -n line-bot-prod

# ดู rollout status
kubectl rollout status deployment/line-bot -n line-bot-prod
```

#### ถ้า Database ล้มเหลว:
```bash
# ตรวจสอบ Patroni leader
kubectl exec -it patroni-0 -n databases -- patronictl list

# Manual failover ถ้าจำเป็น
kubectl exec -it patroni-0 -n databases -- \
  patronictl failover linebot-cluster --force
```

#### ถ้า Redis Sentinel ล้มเหลว:
```bash
# ตรวจสอบ Sentinel status
redis-cli -h sentinel1 -p 26379 SENTINEL masters

# Force failover
redis-cli -h sentinel1 -p 26379 SENTINEL failover mymaster
```

### ขั้นตอนที่ 4: Communicate
```
เทมเพลต Line Alert:
🚨 [P1] LINE Bot Incident
Time: <timestamp>
Impact: Bot ไม่ตอบสนองทั้งหมด
Status: กำลังแก้ไข
ETA: <estimated time>
Incident ID: INC-<id>
```

### ขั้นตอนที่ 5: Post-Incident
1. RCA (Root Cause Analysis) ภายใน 24 ชั่วโมง
2. บันทึก action items
3. อัปเดต Runbook ถ้าจำเป็น
4. แจ้งลูกค้า (ถ้า SLA breach)
```

---

## 13. Complete HA Setup Guide {#complete-ha}

### Terraform Infrastructure

```hcl
# terraform/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "s3" {
    bucket = "linebot-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-southeast-1"
    encrypt = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "linebot-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway     = true
  single_nat_gateway     = false  # HA: one NAT per AZ
  one_nat_gateway_per_az = true
  
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Environment = "production"
    Project     = "linebot"
    Terraform   = "true"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  
  cluster_name    = "linebot-prod"
  cluster_version = "1.29"
  
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Control plane logging
  cluster_enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
  
  # Managed node groups
  eks_managed_node_groups = {
    # General workloads
    general = {
      name = "general"
      
      instance_types = ["t3.medium", "t3.large"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 3
      max_size     = 10
      desired_size = 3
      
      labels = {
        role = "general"
      }
      
      # Multi-AZ
      subnet_ids = module.vpc.private_subnets
      
      update_config = {
        max_unavailable_percentage = 33
      }
    }
    
    # Database workloads
    database = {
      name = "database"
      
      instance_types = ["r6i.xlarge"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 3
      max_size     = 6
      desired_size = 3
      
      labels = {
        role = "database"
      }
      
      taints = [{
        key    = "role"
        value  = "database"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# RDS Aurora (backup database option)
resource "aws_rds_cluster" "linebot" {
  cluster_identifier      = "linebot-aurora"
  engine                  = "aurora-postgresql"
  engine_version          = "16.1"
  database_name           = "linebot_db"
  master_username         = "postgres"
  master_password         = var.db_master_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.linebot.name
  
  # HA settings
  availability_zones = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  
  backup_retention_period = 7
  preferred_backup_window = "02:00-04:00"
  
  # Enable data API
  enable_http_endpoint = false
  
  # Storage encryption
  storage_encrypted = true
  kms_key_id       = aws_kms_key.rds.arn
  
  # Enable enhanced monitoring
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "linebot-final-${formatdate("YYYY-MM-DD", timestamp())}"
  
  tags = {
    Environment = "production"
    Project     = "linebot"
  }
}

# Aurora instances (Multi-AZ)
resource "aws_rds_cluster_instance" "linebot" {
  count = 3
  
  cluster_identifier = aws_rds_cluster.linebot.id
  identifier         = "linebot-aurora-${count.index}"
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.linebot.engine
  engine_version     = aws_rds_cluster.linebot.engine_version
  
  # Spread across AZs
  availability_zone = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"][count.index]
  
  # Performance insights
  performance_insights_enabled    = true
  performance_insights_kms_key_id = aws_kms_key.rds.arn
  
  # Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}

# ElastiCache Redis (Cluster Mode)
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "linebot-redis"
  description          = "LINE Bot Redis Cluster"
  
  node_type            = "cache.r7g.large"
  num_node_groups      = 3  # 3 shards
  replicas_per_node_group = 2  # 2 replicas per shard
  
  # HA settings
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  subnet_group_name = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]
  
  # Encryption
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  
  # Maintenance window
  maintenance_window = "sun:03:00-sun:05:00"
  snapshot_window    = "01:00-03:00"
  snapshot_retention_limit = 7
  
  # Parameter group
  parameter_group_name = "default.redis7.cluster.on"
  
  tags = {
    Environment = "production"
    Project     = "linebot"
  }
}

# ALB (Application Load Balancer)
resource "aws_lb" "linebot" {
  name               = "linebot-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets
  
  enable_deletion_protection = true
  
  # Access logs
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "linebot-alb"
    enabled = true
  }
  
  tags = {
    Environment = "production"
    Project     = "linebot"
  }
}
```

### Monitoring Setup

```yaml
# k8s/monitoring/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: linebot-alerts
  namespace: monitoring
spec:
  groups:
    - name: linebot.rules
      rules:
        # High error rate
        - alert: LineBotHighErrorRate
          expr: |
            sum(rate(http_requests_total{app="line-bot",status=~"5.."}[5m])) /
            sum(rate(http_requests_total{app="line-bot"}[5m])) > 0.01
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "LINE Bot high error rate"
            description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

        # Pod down
        - alert: LineBotPodDown
          expr: |
            kube_deployment_status_replicas_available{deployment="line-bot",namespace="line-bot-prod"} < 2
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "LINE Bot pods unavailable"
            description: "Only {{ $value }} pods are available (minimum 2 required)"

        # High latency
        - alert: LineBotHighLatency
          expr: |
            histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{app="line-bot"}[5m])) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "LINE Bot high P95 latency"
            description: "P95 latency is {{ $value }}s"

        # Database connection pool exhausted
        - alert: DatabaseConnectionPoolExhausted
          expr: |
            prisma_pool_connections_open / prisma_pool_connections_idle > 0.9
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Database connection pool nearly exhausted"

        # Redis memory high
        - alert: RedisHighMemory
          expr: |
            redis_memory_used_bytes / redis_memory_max_bytes > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Redis memory usage is high"
            description: "Redis is using {{ $value | humanizePercentage }} of max memory"
```

---

### สรุป High Availability Best Practices

```
✅ Infrastructure:
   - Multi-AZ deployment สำหรับทุก component
   - Auto-scaling สำหรับ application tier
   - Managed services (RDS, ElastiCache) สำหรับ database tier
   - Automated failover ทุก layer

✅ Application:
   - Stateless applications (session ใน Redis)
   - Graceful shutdown (drain requests ก่อนหยุด)
   - Circuit breakers สำหรับ external APIs
   - Retry with exponential backoff

✅ Database:
   - Synchronous replication สำหรับ critical data
   - Read replicas สำหรับ read-heavy workloads
   - Regular backup testing
   - Point-in-time recovery capability

✅ Monitoring:
   - Health checks ทุก 5-10 seconds
   - Alerting ทาง LINE และ email
   - Distributed tracing (Jaeger/Tempo)
   - Centralized logging (EFK Stack)

✅ Testing:
   - Chaos engineering (Chaos Monkey)
   - Regular disaster recovery drills
   - Load testing ก่อน major events
   - Failover testing ทุก quarter

💰 Cost Optimization:
   - Spot instances สำหรับ non-critical workloads
   - Reserved instances สำหรับ baseline capacity
   - Right-size instances ตาม metrics
   - Auto-scale down ในช่วง traffic ต่ำ

🔒 Security:
   - Encryption at rest และ in transit
   - Network segmentation (VPC + Security Groups)
   - IAM roles แทน access keys
   - Regular security audits
   - WAF สำหรับ webhook endpoints
```

---

*Part 90 จบแล้ว - ขอบคุณที่ติดตาม LINE OA Course ครบทั้ง 90 Parts!*
