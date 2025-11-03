# Hướng dẫn cài đặt và triển khai ELK Stack với NestJS

## 1. Cài đặt ELK Stack bằng Docker

### 1.1. Yêu cầu hệ thống

- Docker và Docker Compose đã được cài đặt
- Ít nhất 4GB RAM khả dụng cho Docker
- Docker Desktop (Windows/Mac) hoặc Docker Engine (Linux)

### 1.2. Tạo file docker-compose.yml

Tạo file `docker-compose.yml` trong thư mục gốc của dự án:

```yaml
version: '3.8'

services:
  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - elk
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Logstash
  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - LS_JAVA_OPTS=-Xms256m -Xmx256m
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  # Kibana
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_URL=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch_data:
    driver: local
```

### 1.3. Cấu hình Logstash

Tạo cấu trúc thư mục cho Logstash:

```
logstash/
├── config/
│   └── logstash.yml
└── pipeline/
    └── logstash.conf
```

**File `logstash/config/logstash.yml`:**

```yaml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]
```

**File `logstash/pipeline/logstash.conf`:**

```conf
input {
  tcp {
    port => 5000
    codec => json
  }
  udp {
    port => 5000
    codec => json
  }
}

filter {
  # Thêm timestamp nếu chưa có
  if ![timestamp] {
    mutate {
      add_field => { "timestamp" => "%{@timestamp}" }
    }
  }

  # Parse JSON nếu message là string
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
      target => "parsed"
    }
  }

  # Thêm các field tùy chỉnh
  mutate {
    add_field => {
      "service" => "nestjs-app"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nestjs-logs-%{+YYYY.MM.dd}"
  }

  # Để debug, có thể bật stdout
  stdout {
    codec => rubydebug
  }
}
```

### 1.4. Khởi chạy ELK Stack

```bash
# Khởi động tất cả services
docker-compose up -d

# Kiểm tra trạng thái
docker-compose ps

# Xem logs
docker-compose logs -f
```

### 1.5. Kiểm tra kết nối

- **Elasticsearch**: http://localhost:9200
- **Kibana**: http://localhost:5601
- **Logstash**: TCP/UDP port 5000

Kiểm tra Elasticsearch:
```bash
curl http://localhost:9200
```

Kết quả mong đợi:
```json
{
  "name" : "...",
  "cluster_name" : "docker-cluster",
  "version" : {
    "number" : "8.11.0",
    ...
  }
}
```

---

## 2. Tích hợp Elasticsearch vào NestJS

### 2.1. Cài đặt các package cần thiết

```bash
npm install @nestjs/elasticsearch @elastic/elasticsearch
npm install -D @types/node
```

### 2.2. Cấu hình Elasticsearch Module

**File `.env`:**

```env
# Elasticsearch Configuration
ELASTICSEARCH_NODE=http://localhost:9200
ELASTICSEARCH_INDEX_PREFIX=nestjs
```

**File `src/config/elasticsearch.config.ts`:**

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('elasticsearch', () => ({
  node: process.env.ELASTICSEARCH_NODE || 'http://localhost:9200',
  indexPrefix: process.env.ELASTICSEARCH_INDEX_PREFIX || 'nestjs',
}));
```

### 2.3. Tạo Elasticsearch Module

**File `src/elasticsearch/elasticsearch.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { ElasticsearchModule as NestElasticsearchModule } from '@nestjs/elasticsearch';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ElasticsearchService } from './elasticsearch.service';

@Module({
  imports: [
    NestElasticsearchModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        node: configService.get('elasticsearch.node'),
        maxRetries: 10,
        requestTimeout: 60000,
        pingTimeout: 60000,
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [ElasticsearchService],
  exports: [ElasticsearchService],
})
export class ElasticsearchModule {}
```

## 3. Cấu hình và sử dụng Kibana


## 4. Testing và Debugging

### 4.1. Test kết nối Elasticsearch

```bash
# Kiểm tra health
curl http://localhost:9200/_cluster/health?pretty

# Kiểm tra các indices
curl http://localhost:9200/_cat/indices?v

# Xem dữ liệu trong index
curl http://localhost:9200/nestjs-products/_search?pretty
```

### 4.2. Test Logstash

Gửi test message đến Logstash:

```bash
# Linux/Mac
echo '{"message":"test log","level":"info"}' | nc localhost 5000

# Windows (PowerShell)
'{"message":"test log","level":"info"}' | Out-File -Encoding ASCII temp.json
Get-Content temp.json | & "C:\path\to\nc.exe" localhost 5000
```
