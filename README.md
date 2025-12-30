# LVTN - Deployment Guide

Hướng dẫn deploy hệ thống LVTN (Luận Văn Tốt Nghiệp) trên server.

> **Liên hệ Admin để lấy credentials:**
> - Database password (MySQL, MongoDB, Redis)
> - Google OAuth credentials
> - MinIO access keys
> - JWT secrets
> - Gemini API key
> - TLS certificates

---

## Mục lục

1. [Cấu trúc thư mục](#cấu-trúc-thư-mục)
2. [Port Mapping](#port-mapping)
3. [Docker Compose Files](#docker-compose-files)
4. [Environment Files](#environment-files)
5. [Deploy Commands](#deploy-commands)
6. [Troubleshooting](#troubleshooting)

---

## Cấu trúc thư mục

```
~/docker-compose/lvtn/
├── server/
│   ├── main/              # GraphQL Gateway (BE_main)
│   │   ├── docker-compose.yml
│   │   ├── .server.env
│   │   └── certs/         # TLS certificates
│   └── admin/             # Admin Server (BE_core)
│       ├── docker-compose.yml
│       ├── .env
│       └── certs/
├── fe/
│   ├── fe-1/              # Frontend chính (FE_main - NextJS)
│   │   └── docker-compose.yml
│   └── fe-2/              # Frontend admin (FE_admin - Vite)
│       └── docker-compose.yml
├── service/
│   ├── academic/          # Academic Service (gRPC :50051)
│   │   ├── docker-compose.yml
│   │   └── academic.env
│   ├── council/           # Council Service (gRPC :50052)
│   │   ├── docker-compose.yml
│   │   └── council.env
│   ├── file/              # File Service (gRPC :50053)
│   │   ├── docker-compose.yml
│   │   └── file.env
│   ├── role/              # Role Service (gRPC :50054)
│   │   ├── docker-compose.yml
│   │   └── role.env
│   ├── thesis/            # Thesis Service (gRPC :50055)
│   │   ├── docker-compose.yml
│   │   └── thesis.env
│   ├── user/              # User Service (gRPC :50056)
│   │   ├── docker-compose.yml
│   │   └── user.env
│   └── plagiarism/        # Plagiarism Service (gRPC :50057)
│       ├── docker-compose.yml
│       ├── .env
│       └── certs/
├── monitoring/            # Grafana + Prometheus + Loki
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   ├── loki-config.yml
│   └── promtail-config.yml
└── logs/                  # Log files
```

---

## Port Mapping

| Service | Port | Container Name | Mô tả |
|---------|------|----------------|-------|
| GraphQL Gateway | 8080 | graphql_gateway | API chính (api-lvtn.thaily.id.vn) |
| Admin Server | 8000 | lvtn-server-2 | API admin + Bull Board |
| Frontend Main | 3000 | fe-server | NextJS (lvtn.thaily.id.vn) |
| Frontend Admin | 5173 | fe-server-2 | Vite |
| Academic | 50051 | academic_service | gRPC Service |
| Council | 50052 | council_service | gRPC Service |
| File | 50053 | file_service | gRPC Service |
| Role | 50054 | role_service | gRPC Service |
| Thesis | 50055 | thesis_service | gRPC Service |
| User | 50056 | user_service | gRPC Service |
| Plagiarism | 50057 | plagiarism-service | gRPC Service |
| Grafana | 20008 | grafana | Monitoring Dashboard |
| Prometheus | 10009 | prometheus | Metrics |
| Loki | 10008 | loki | Log aggregation |

### External Services (không trong lvtn/)

| Service | Port | Mô tả |
|---------|------|-------|
| MySQL | 10001 | Database chính |
| Redis | 10002 | Cache + Queue |
| MinIO | 10005 | Object Storage |
| Ollama | 10006 | Local LLM |
| Elasticsearch | 10010 | Search engine |

---

## Docker Compose Files

### 1. server/main/docker-compose.yml

```yaml
version: '3.8'

services:
  server:
    image: thaily/lvtn-server:latest
    container_name: graphql_gateway
    env_file:
      - .server.env
    ports:
      - "8080:8080"
    volumes:
      - ./certs:/app/certs:ro
      - ../../logs:/app/logs
    networks:
      - lvtn_network
    restart: unless-stopped

networks:
  lvtn_network:
    driver: bridge
```

### 2. server/admin/docker-compose.yml

```yaml
services:
  lvtn-server-2:
    image: thaily/lvtn-server-2:latest
    container_name: lvtn-server-2
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    extra_hosts:
      - "file-service:172.17.0.1"
      - "academic-service:172.17.0.1"
      - "council-service:172.17.0.1"
      - "role-service:172.17.0.1"
      - "thesis-service:172.17.0.1"
      - "user-service:172.17.0.1"
      - "plagiarism-service:172.17.0.1"
    volumes:
      - ./certs:/app/certs:ro
    networks:
      - lvtn-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  lvtn-network:
```

### 3. fe/fe-1/docker-compose.yml (Frontend Main)

```yaml
services:
  fe-server:
    image: thaily/fe-server:latest
    container_name: fe-server
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_GRAPHQL_ENDPOINT=https://api-lvtn.thaily.id.vn/query
      - NEXT_PUBLIC_BACKEND_URL=https://api-lvtn.thaily.id.vn
      - NEXT_PUBLIC_TINYMCE_API_KEY=<TINYMCE_API_KEY>
    networks:
      - lvtn-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  lvtn-network:
```

### 4. fe/fe-2/docker-compose.yml (Frontend Admin)

```yaml
services:
  fe-server-2:
    image: thaily/fe-server-2:latest
    container_name: fe-server-2
    restart: unless-stopped
    ports:
      - "5173:80"
    environment:
      - VITE_API_URL=http://100.89.142.60:3001/api
    networks:
      - lvtn-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  lvtn-network:
```

### 5. service/academic/docker-compose.yml

```yaml
version: '3.8'

services:
  academic:
    image: thaily/lvtn-academic:latest
    container_name: academic_service
    env_file:
      - academic.env
    ports:
      - "50051:50051"
    volumes:
      - ./academic.env:/app/academic.env:ro
      - .:/app/service:ro
    networks:
      - academic_network
    restart: unless-stopped

networks:
  academic_network:
    driver: bridge
```

### 6. service/council/docker-compose.yml

```yaml
version: '3.8'

services:
  council:
    image: thaily/lvtn-council:latest
    container_name: council_service
    env_file:
      - council.env
    ports:
      - "50052:50052"
    volumes:
      - ./council.env:/app/council.env:ro
      - .:/app/service:ro
    restart: unless-stopped
```

### 7. service/file/docker-compose.yml

```yaml
version: '3.8'

services:
  file:
    image: thaily/lvtn-file:latest
    container_name: file_service
    env_file:
      - file.env
    ports:
      - "50053:50053"
    volumes:
      - ./file.env:/app/file.env:ro
      - .:/app/service:ro
    restart: unless-stopped
```

### 8. service/role/docker-compose.yml

```yaml
version: '3.8'

services:
  role:
    image: thaily/lvtn-role:latest
    container_name: role_service
    env_file:
      - role.env
    ports:
      - "50054:50054"
    volumes:
      - ./role.env:/app/role.env:ro
      - .:/app/service:ro
    restart: unless-stopped
```

### 9. service/thesis/docker-compose.yml

```yaml
version: '3.8'

services:
  thesis:
    image: thaily/lvtn-thesis:latest
    container_name: thesis_service
    env_file:
      - thesis.env
    ports:
      - "50055:50055"
    volumes:
      - ./thesis.env:/app/thesis.env:ro
      - .:/app/service:ro
    restart: unless-stopped
```

### 10. service/user/docker-compose.yml

```yaml
version: '3.8'

services:
  user:
    image: thaily/lvtn-user:latest
    container_name: user_service
    env_file:
      - user.env
    ports:
      - "50056:50056"
    volumes:
      - ./user.env:/app/user.env:ro
      - .:/app/service:ro
    restart: unless-stopped
```

### 11. service/plagiarism/docker-compose.yml

```yaml
version: '3.8'

services:
  plagiarism-service:
    image: thaily/plagiarism:latest
    container_name: plagiarism-service
    ports:
      - "50057:50057"
    env_file:
      - .env
    volumes:
      - ./certs:/app/certs:ro
```

### 12. monitoring/docker-compose.yml

```yaml
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "10008:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: always
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    restart: always
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "10009:9090"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
    restart: always
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "20008:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
      - ./dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_USER=<ADMIN_USER>
      - GF_SECURITY_ADMIN_PASSWORD=<ADMIN_PASSWORD>
      - GF_USERS_ALLOW_SIGN_UP=false
    depends_on:
      - loki
      - prometheus
    restart: always
    networks:
      - monitoring

volumes:
  loki_data:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

---

## Environment Files

> **Liên hệ Admin để lấy giá trị thực cho các biến có dấu `<...>`**

### 1. server/main/.server.env

```env
# Server Configuration
DEFAULT_PORT=8080
PORT=8080
GIN_MODE=release

# Service URLs (Docker host IP)
SERVICE_ACADEMIC_URL=172.17.0.1
SERVICE_COUNCIL_URL=172.17.0.1
SERVICE_FILE_URL=172.17.0.1
SERVICE_ROLE_URL=172.17.0.1
SERVICE_THESIS_URL=172.17.0.1
SERVICE_USER_URL=172.17.0.1

# Service Ports
ACADEMIC_NAME=ACADEMIC_SERVICE
ACADEMIC_PORT=50051

COUNCIL_NAME=COUNCIL_SERVICE
COUNCIL_PORT=50052

FILE_NAME=FILE_SERVICE
FILE_PORT=50053

ROLE_NAME=ROLE_SERVICE
ROLE_PORT=50054

THESIS_NAME=THESIS_SERVICE
THESIS_PORT=50055

USER_NAME=USER_SERVICE
USER_PORT=50056

# MinIO Configuration
# 👉 Liên hệ Admin lấy access key
MIN_ENDPOINT=172.17.0.1:10005
MIN_ACCESS_KEY=<MINIO_ACCESS_KEY>
MIN_SECRET_KEY=<MINIO_SECRET_KEY>
MIN_USE_SSL=false
MIN_BUCKET_NAME=lvtn

# Google OAuth Configuration
# 👉 Liên hệ Admin lấy Google OAuth credentials
GOOGLE_CLIENT_ID=<GOOGLE_CLIENT_ID>
GOOGLE_SECRET=<GOOGLE_CLIENT_SECRET>
GOOGLE_REDIRECTURL=https://lvtn.thaily.id.vn/login/callback

# JWT Configuration
# 👉 Liên hệ Admin lấy JWT secrets
JWT_ACCESS_SECRET=<JWT_ACCESS_SECRET>
JWT_REFRESH_SECRET=<JWT_REFRESH_SECRET>
JWT_ACCESS_EXPIRY=15
JWT_REFRESH_EXPIRY=7

# Redis Configuration
# 👉 Liên hệ Admin lấy Redis password
REDIS_ADDRESS=172.17.0.1:10002
REDIS_PASSWORD=<REDIS_PASSWORD>
REDIS_DB=0
REDIS_DIAL_TIMEOUT=5
REDIS_READ_TIMEOUT=3
REDIS_WRITE_TIMEOUT=3
REDIS_POOL_SIZE=10
REDIS_MIN_IDLE_CONNS=2

# MongoDB Configuration
# 👉 Liên hệ Admin lấy MongoDB URI
MONGO_URI=<MONGODB_URI>
MONGO_DATABASE=lvtn
MONGO_USERNAME=
MONGO_PASSWORD=
MONGO_AUTH_SOURCE=admin
MONGO_MAX_POOL_SIZE=100
MONGO_MIN_POOL_SIZE=10
MONGO_MAX_CONN_IDLE_TIME=60
MONGO_CONNECT_TIMEOUT=10
MONGO_SERVER_SELECTION_TIMEOUT=10

# TLS Certificate Path
CERTS_PATH=/app/certs

# CORS Configuration
# 👉 Thêm domain frontend được phép truy cập
CORS_ALLOW_ORIGINS=http://localhost:3000,https://lvtn.thaily.id.vn
```

### 2. server/admin/.env

```env
# Node Environment
NODE_ENV=production

# Service Configuration
SERVICE_NAME=plagiarism-checker-service
LOG_LEVEL=info

# MongoDB Configuration
# 👉 Liên hệ Admin lấy MongoDB URI
MONGODB_URI=<MONGODB_URI>

# Redis Configuration (BullMQ)
# 👉 Liên hệ Admin lấy Redis password
REDIS_HOST=172.17.0.1
REDIS_PORT=10002
REDIS_PASSWORD=<REDIS_PASSWORD>
REDIS_DB=0

# gRPC File Service
GRPC_FILE_SERVICE_URL=172.17.0.1:50053
GRPC_TLS_ENABLED=true
GRPC_CERT_PATH=/app/certs/clients/client.crt
GRPC_KEY_PATH=/app/certs/clients/client.key
GRPC_CA_PATH=/app/certs/ca/ca.crt

# MinIO Configuration
# 👉 Liên hệ Admin lấy MinIO credentials
MINIO_ENDPOINT=172.17.0.1
MINIO_PORT=10005
MINIO_ACCESS_KEY=<MINIO_ACCESS_KEY>
MINIO_SECRET_KEY=<MINIO_SECRET_KEY>
MINIO_USE_SSL=false
MINIO_BUCKET_NAME=lvtn

# Plagiarism API Configuration
PLAGIARISM_API_URL=http://172.17.0.1:8080/check
PLAGIARISM_API_KEY=mock_key
PLAGIARISM_API_TIMEOUT=60000

# Worker Configuration
WORKER_CONCURRENCY=3
WORKER_MAX_RETRIES=3
WORKER_RETRY_DELAY=10000

# Scheduler Configuration
CRON_SCHEDULE=* * * * *

# Bull Board UI
BULL_BOARD_PORT=8000
BULL_BOARD_PATH=/admin/queues

# File Processing
TMP_DIR=./tmp
MAX_FILE_SIZE_MB=50
ALLOWED_FILE_TYPES=.pdf,.doc,.docx,.txt

# JWT Configuration
# 👉 Liên hệ Admin lấy JWT secret
JWT_SECRET=<JWT_SECRET>
JWT_EXPIRES_IN=12h

# Email Configuration (for OTP)
# 👉 Liên hệ Admin lấy SMTP credentials
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=<EMAIL>
SMTP_PASS=<APP_PASSWORD>
EMAIL_FROM=<EMAIL>

# CORS Configuration
FRONTEND_URL=*
```

### 3. service/[academic|council|file|role|thesis|user]/*.env

Template chung cho các gRPC services:

```env
# Database Configuration
# 👉 Liên hệ Admin lấy MySQL credentials
DB_HOST=172.17.0.1
DB_PORT=10001
DB_USER=root
DB_PASSWORD=<MYSQL_PASSWORD>
DB_NAME=lvtn-db-v4
CODE=PRODUCTION

# Service Configuration
SERVICE_NAME=<ServiceName>Service  # Ví dụ: AcademicService
SERVICE_PORT=<PORT>                 # Ví dụ: 50051
SERVICE_CERT_PATH=/app/service
SERVICE_CA_CERT=/app/service
```

| Service | SERVICE_NAME | SERVICE_PORT |
|---------|--------------|--------------|
| academic | AcademicService | 50051 |
| council | CouncilService | 50052 |
| file | FileService | 50053 |
| role | RoleService | 50054 |
| thesis | ThesisService | 50055 |
| user | UserService | 50056 |

### 4. service/plagiarism/.env

```env
# Elasticsearch Configuration
ES_HOST=172.17.0.1
ES_PORT=10010
ES_INDEX=plagiarism_documents
ES_USER=elastic
ES_PASSWORD=changeme
ES_SCHEME=http

# Analyzer Mode: 'external' (Gemini) or 'internal' (Ollama)
ANALYZER_MODE=external

# Gemini Configuration (external mode)
# 👉 Liên hệ Admin lấy Gemini API key
GEMINI_API_KEY=<GEMINI_API_KEY>
GEMINI_MODEL=gemini-2.0-flash
GEMINI_TIMEOUT=60
GEMINI_API_URL=https://generativelanguage.googleapis.com/v1beta/models

# Ollama Configuration (internal mode)
OLLAMA_HOST=http://172.17.0.1:10006
OLLAMA_EMBED_MODEL=nomic-embed-text
OLLAMA_CHAT_MODEL=phi3
OLLAMA_TIMEOUT=300

# gRPC Server
GRPC_HOST=0.0.0.0
GRPC_PORT=50057
GRPC_MAX_WORKERS=3

# TLS Configuration
# 👉 Liên hệ Admin lấy TLS certificates
GRPC_TLS_ENABLED=true
GRPC_CERT_PATH=/app/certs/plagiarism-server.crt
GRPC_KEY_PATH=/app/certs/plagiarism-server.key
GRPC_CA_PATH=/app/certs/ca.crt

# Logging
LOG_LEVEL=INFO

# Plagiarism Detection Thresholds
SIMILARITY_CRITICAL=0.95
SIMILARITY_HIGH=0.85
SIMILARITY_MEDIUM=0.70
SIMILARITY_LOW=0.50

# Text Chunking
CHUNK_SIZE=100
CHUNK_OVERLAP=10
MIN_CHUNK_SIZE=30
MIN_CONTENT_LENGTH=200

# Search Configuration
TOP_K_RESULTS=10
MIN_SCORE_THRESHOLD=0.50
MAX_RESULTS_PER_SOURCE=3

# Embedding
EMBEDDING_DIMS=768
EMBEDDING_BATCH_SIZE=32

# MinIO Storage
# 👉 Liên hệ Admin lấy MinIO credentials
MINIO_ENDPOINT=172.17.0.1
MINIO_PORT=10005
MINIO_ACCESS_KEY=<MINIO_ACCESS_KEY>
MINIO_SECRET_KEY=<MINIO_SECRET_KEY>
MINIO_USE_SSL=false
MINIO_BUCKET_NAME=lvtn
```

---

## TLS Certificates

> 👉 **Liên hệ Admin để lấy TLS certificates**

Cấu trúc thư mục certificates:

```
certs/
├── ca/
│   └── ca.crt              # CA certificate
├── clients/
│   ├── client.crt          # Client certificate
│   └── client.key          # Client private key
├── <service>-server.crt    # Server certificate
└── <service>-server.key    # Server private key
```

---

## Deploy Commands

### 1. Deploy gRPC Services (chạy đầu tiên)

```bash
# Deploy tất cả services cùng lúc
for svc in academic council file role thesis user plagiarism; do
  echo "=== Deploying $svc ==="
  cd ~/docker-compose/lvtn/service/$svc
  sudo docker compose pull
  sudo docker compose up -d
done
```

### 2. Deploy Backend Servers

```bash
# GraphQL Gateway (main)
cd ~/docker-compose/lvtn/server/main
sudo docker compose pull
sudo docker compose down && sudo docker compose up -d

# Admin Server
cd ~/docker-compose/lvtn/server/admin
sudo docker compose pull
sudo docker compose down && sudo docker compose up -d
```

### 3. Deploy Frontend

```bash
# Frontend Main (NextJS)
cd ~/docker-compose/lvtn/fe/fe-1
sudo docker compose pull
sudo docker compose down && sudo docker compose up -d

# Frontend Admin (Vite)
cd ~/docker-compose/lvtn/fe/fe-2
sudo docker compose pull
sudo docker compose down && sudo docker compose up -d
```

### 4. Deploy Monitoring (optional)

```bash
cd ~/docker-compose/lvtn/monitoring
sudo docker compose pull
sudo docker compose up -d
```

### 5. Quick Deploy All (Script)

Tạo file `~/docker-compose/lvtn/deploy-all.sh`:

```bash
#!/bin/bash
set -e

LVTN_DIR=~/docker-compose/lvtn

echo "=========================================="
echo "       LVTN Full System Deployment       "
echo "=========================================="

echo ""
echo "=== [1/4] Deploying gRPC Services ==="
for svc in academic council file role thesis user plagiarism; do
  echo "  → Deploying $svc..."
  cd $LVTN_DIR/service/$svc
  sudo docker compose pull -q
  sudo docker compose up -d
done

echo ""
echo "=== [2/4] Deploying Backend Servers ==="
echo "  → Deploying GraphQL Gateway..."
cd $LVTN_DIR/server/main
sudo docker compose pull -q
sudo docker compose down
sudo docker compose up -d

echo "  → Deploying Admin Server..."
cd $LVTN_DIR/server/admin
sudo docker compose pull -q
sudo docker compose down
sudo docker compose up -d

echo ""
echo "=== [3/4] Deploying Frontend ==="
echo "  → Deploying Frontend Main..."
cd $LVTN_DIR/fe/fe-1
sudo docker compose pull -q
sudo docker compose down
sudo docker compose up -d

echo "  → Deploying Frontend Admin..."
cd $LVTN_DIR/fe/fe-2
sudo docker compose pull -q
sudo docker compose down
sudo docker compose up -d

echo ""
echo "=== [4/4] Deploying Monitoring ==="
cd $LVTN_DIR/monitoring
sudo docker compose pull -q
sudo docker compose up -d

echo ""
echo "=========================================="
echo "          Deployment Complete!           "
echo "=========================================="
echo ""
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | head -20
```

Chạy script:

```bash
chmod +x ~/docker-compose/lvtn/deploy-all.sh
~/docker-compose/lvtn/deploy-all.sh
```

---

## Useful Commands

### Check Status

```bash
# Xem tất cả containers
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Xem logs
sudo docker logs <container_name> --tail 100 -f

# Ví dụ:
sudo docker logs graphql_gateway --tail 50 -f
sudo docker logs fe-server --tail 50 -f
sudo docker logs academic_service --tail 50 -f
```

### Restart Service

```bash
cd ~/docker-compose/lvtn/<path-to-service>
sudo docker compose restart
```

### Update & Restart

```bash
cd ~/docker-compose/lvtn/<path-to-service>
sudo docker compose pull
sudo docker compose down
sudo docker compose up -d
```

### Resource Usage

```bash
sudo docker stats --no-stream
```

### Clean Up

```bash
# Xóa images không dùng
sudo docker image prune -a

# Xóa containers đã dừng
sudo docker container prune

# Xóa tất cả không dùng (cẩn thận!)
sudo docker system prune -a
```

---

## Troubleshooting

### 1. CORS Error

**Lỗi:** `Access-Control-Allow-Origin` blocked

**Fix:**
1. Mở file `~/docker-compose/lvtn/server/main/.server.env`
2. Thêm domain vào `CORS_ALLOW_ORIGINS`:
   ```env
   CORS_ALLOW_ORIGINS=http://localhost:3000,https://lvtn.thaily.id.vn,https://new-domain.com
   ```
3. Restart server:
   ```bash
   cd ~/docker-compose/lvtn/server/main
   sudo docker compose down && sudo docker compose up -d
   ```

### 2. Service Connection Failed

**Lỗi:** gRPC service không connect được

**Check:**
1. Service đang chạy: `sudo docker ps | grep <service_name>`
2. Logs: `sudo docker logs <container_name>`
3. Port đúng trong env file
4. IP `172.17.0.1` (Docker host gateway)

### 3. Image Cũ Sau CI/CD

```bash
# Force pull image mới
sudo docker compose pull --no-cache

# Hoặc xóa image cũ trước
sudo docker rmi <image_name>:latest
sudo docker compose pull
sudo docker compose up -d

# Verify image date
sudo docker images <image_name> --format "{{.ID}} {{.CreatedAt}}"
```

### 4. Container Keeps Restarting

```bash
# Xem logs để tìm lỗi
sudo docker logs <container_name> --tail 100

# Check healthcheck
sudo docker inspect <container_name> | grep -A 10 Health
```

### 5. Certificate Errors

**Lỗi:** TLS handshake failed

**Fix:**
1. Kiểm tra certificates tồn tại trong thư mục `certs/`
2. Kiểm tra permissions: `chmod 644 *.crt && chmod 600 *.key`
3. Verify certificate: `openssl x509 -in cert.crt -text -noout`

---

## CI/CD Flow

```
1. Developer push code → GitHub
                            ↓
2. GitHub Actions trigger → Build Docker image
                            ↓
3. Push image → Docker Hub (thaily/*)
                            ↓
4. SSH to server:
   cd ~/docker-compose/lvtn/<service>
   sudo docker compose pull
   sudo docker compose down && sudo docker compose up -d
```

---

## Monitoring

**Grafana Dashboard:** `http://<server-ip>:20008`

> 👉 Liên hệ Admin lấy credentials

Dashboards:
- Container metrics (CPU, Memory, Network)
- Application logs (via Loki)
- Request metrics (via Prometheus)

---

## Liên hệ

**Admin:** Thaily
- Email: lyvinhthai321@gmail.com
- Server: `100.89.142.60` (Tailscale)

**Cần lấy credentials:**
1. Database passwords (MySQL, MongoDB, Redis)
2. Google OAuth (Client ID, Secret)
3. MinIO (Access Key, Secret Key)
4. JWT Secrets
5. Gemini API Key
6. TLS Certificates
7. SMTP credentials (Gmail App Password)
8. Grafana admin password
