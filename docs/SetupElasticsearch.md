# Hướng dẫn cài đặt ELK Stack với NestJS

## Giới thiệu

ELK Stack gồm 3 thành phần:
- **Elasticsearch**: Lưu trữ và tìm kiếm dữ liệu
- **Logstash**: Thu thập và xử lý logs
- **Kibana**: Xem và phân tích dữ liệu trực quan

---

## 1. Cài đặt ELK bằng Docker

### 1.1. Tạo file docker-compose.yml

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5000:5000"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch_data:
```

### 1.2. Cấu hình Logstash

Tạo thư mục và file cấu hình:

**File `logstash/config/logstash.yml`:**

```yaml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

#### Option 1: Nhận logs từ ứng dụng (qua TCP)

**File `logstash/pipeline/logstash.conf`:**

```conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  mutate {
    add_field => { "service" => "nestjs-app" }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "nestjs-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

#### Option 2: Lấy dữ liệu từ PostgreSQL

**Cài đặt JDBC driver:**

Tải PostgreSQL JDBC driver và đặt vào `logstash/drivers/`:

```bash
mkdir -p logstash/drivers
cd logstash/drivers
wget https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
```

**File `logstash/pipeline/postgres.conf`:**

```conf
input {
  jdbc {
    jdbc_driver_library => "/usr/share/logstash/drivers/postgresql-42.7.1.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://host.docker.internal:5432/ecommerce_db"
    jdbc_user => "postgres"
    jdbc_password => "your_password"
    
    # Query để lấy dữ liệu products (loại bỏ soft-deleted)
    statement => "
      SELECT 
        id, category_id, name, slug, short_description, description,
        price, compare_at_price, stock_quantity, sku,
        is_active, is_featured, view_count,
        rating_average, review_count,
        created_at, updated_at
      FROM products 
      WHERE deleted_at IS NULL 
        AND updated_at > :sql_last_value 
      ORDER BY updated_at
    "
    
    # Sử dụng updated_at để tracking
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
    
    # Chạy mỗi 30 giây
    schedule => "*/30 * * * * *"
  }
}

filter {
  # Chuyển đổi kiểu dữ liệu
  mutate {
    convert => {
      "id" => "integer"
      "category_id" => "integer"
      "price" => "float"
      "compare_at_price" => "float"
      "stock_quantity" => "integer"
      "view_count" => "integer"
      "rating_average" => "float"
      "review_count" => "integer"
      "is_active" => "boolean"
      "is_featured" => "boolean"
    }
    
    # Xóa các field không cần thiết
    remove_field => ["@version", "@timestamp"]
  }
  
  # Thêm trạng thái stock
  if [stock_quantity] > 0 {
    mutate {
      add_field => { "stock_status" => "in_stock" }
    }
  } else {
    mutate {
      add_field => { "stock_status" => "out_of_stock" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "products"
    document_id => "%{id}"
    
    # Mapping template cho Elasticsearch
    template => "/usr/share/logstash/templates/products-template.json"
    template_name => "products"
    template_overwrite => true
  }
  
  stdout { codec => rubydebug }
}
```

**File `logstash/templates/products-template.json`:**

Tạo file template để định nghĩa mapping cho Elasticsearch:

```json
{
  "index_patterns": ["products"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "id": { "type": "integer" },
      "category_id": { "type": "integer" },
      "name": { 
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "slug": { "type": "keyword" },
      "short_description": { "type": "text" },
      "description": { "type": "text" },
      "price": { "type": "float" },
      "compare_at_price": { "type": "float" },
      "stock_quantity": { "type": "integer" },
      "stock_status": { "type": "keyword" },
      "sku": { "type": "keyword" },
      "is_active": { "type": "boolean" },
      "is_featured": { "type": "boolean" },
      "view_count": { "type": "integer" },
      "rating_average": { "type": "float" },
      "review_count": { "type": "integer" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" }
    }
  }
}
```

**Cập nhật docker-compose.yml để mount driver và template:**

```yaml
logstash:
  image: docker.elastic.co/logstash/logstash:8.11.0
  container_name: logstash
  volumes:
    - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    - ./logstash/pipeline:/usr/share/logstash/pipeline
    - ./logstash/drivers:/usr/share/logstash/drivers
    - ./logstash/templates:/usr/share/logstash/templates
  ports:
    - "5000:5000"
  networks:
    - elk
  depends_on:
    - elasticsearch
```

**Cấu trúc thư mục:**

```
logstash/
├── config/
│   └── logstash.yml
├── pipeline/
│   └── postgres.conf
├── drivers/
│   └── postgresql-42.7.1.jar
└── templates/
    └── products-template.json
```

**Giải thích cấu hình:**

1. **Query SQL:**
   - Lấy tất cả trường từ bảng `products`
   - Loại bỏ các bản ghi đã soft-delete (`deleted_at IS NULL`)
   - Chỉ lấy bản ghi mới hơn lần chạy trước (`updated_at > :sql_last_value`)

2. **Filter xử lý:**
   - Chuyển đổi kiểu dữ liệu phù hợp (integer, float, boolean)
   - Thêm `stock_status` dựa trên `stock_quantity`

3. **Mapping Template:**
   - Định nghĩa kiểu dữ liệu cho mỗi field trong Elasticsearch
   - `name` có cả text (để search) và keyword (để filter/sort)
   - `slug`, `sku` là keyword (exact match)

4. **Schedule:**
   - `*/30 * * * * *` = chạy mỗi 30 giây
   - Có thể điều chỉnh: `*/1 * * * *` = mỗi 1 phút, `*/5 * * * *` = mỗi 5 phút

**Lưu ý quan trọng:**

1. **Soft Delete**: Query đã loại bỏ các bản ghi có `deleted_at` không NULL (soft-deleted records)

2. **Tracking state**: Logstash tự động lưu trạng thái trong file `.logstash_jdbc_last_run`, không lo bị trùng dữ liệu

3. **Auto-sync**: Mỗi 30 giây, Logstash sẽ tự động check và đồng bộ các bản ghi mới/cập nhật từ PostgreSQL

4. **Tính năng nâng cao**:
   - Tự động phân loại trạng thái kho (in_stock/out_of_stock)
   - Mapping tối ưu cho tìm kiếm full-text và filter

5. **Đồng bộ nhiều bảng**: Tạo thêm file `.conf` trong `logstash/pipeline/` cho mỗi bảng (categories, users, orders...)

### 1.3. Khởi động

```bash
# Khởi động
docker-compose up -d

# Kiểm tra
docker-compose ps

# Xem logs
docker-compose logs -f
```

### 1.4. Kiểm tra

- Elasticsearch: http://localhost:9200
- Kibana: http://localhost:5601

```bash
curl http://localhost:9200
```

---

## 2. Tích hợp vào NestJS

### 2.1. Cài đặt package

```bash
npm install @nestjs/elasticsearch
```

### 2.2. Cấu hình Module

**File `.env`:**

```env
ELASTICSEARCH_NODE=http://localhost:9200
```

**File `src/elasticsearch/elasticsearch.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { ElasticsearchModule as NestElasticsearchModule } from '@nestjs/elasticsearch';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    NestElasticsearchModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        node: config.get('ELASTICSEARCH_NODE'),
      }),
      inject: [ConfigService],
    }),
  ],
  exports: [NestElasticsearchModule],
})
export class ElasticsearchModule {}
```

**File `src/app.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { ElasticsearchModule } from './elasticsearch/elasticsearch.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    ElasticsearchModule,
  ],
})
export class AppModule {}
```

### 2.3. Sử dụng trong Service

**Ví dụ: `src/products/products.service.ts`:**

```typescript
import { Injectable } from '@nestjs/common';
import { ElasticsearchService } from '@nestjs/elasticsearch';

@Injectable()
export class ProductsService {
  constructor(private readonly esService: ElasticsearchService) {}

  // Tìm kiếm sản phẩm theo tên/mô tả
  async searchProducts(keyword: string, page = 1, limit = 20) {
    const { hits } = await this.esService.search({
      index: 'products',
      from: (page - 1) * limit,
      size: limit,
      body: {
        query: {
          bool: {
            must: [
              {
                multi_match: {
                  query: keyword,
                  fields: ['name^2', 'description', 'short_description'],
                  fuzziness: 'AUTO'
                }
              },
              { term: { is_active: true } }
            ]
          }
        },
        sort: [{ rating_average: 'desc' }, { view_count: 'desc' }]
      }
    });
    
    return {
      data: hits.hits.map(hit => hit._source),
      total: hits.total.value,
      page,
      limit
    };
  }

  // Lấy sản phẩm theo category
  async getProductsByCategory(categoryId: number, page = 1, limit = 20) {
    const { hits } = await this.esService.search({
      index: 'products',
      from: (page - 1) * limit,
      size: limit,
      body: {
        query: {
          bool: {
            must: [
              { term: { category_id: categoryId } },
              { term: { is_active: true } }
            ]
          }
        },
        sort: [{ created_at: 'desc' }]
      }
    });
    
    return hits.hits.map(hit => hit._source);
  }

  // Tìm sản phẩm theo khoảng giá
  async getProductsByPriceRange(minPrice: number, maxPrice: number) {
    const { hits } = await this.esService.search({
      index: 'products',
      body: {
        query: {
          bool: {
            must: [
              { range: { price: { gte: minPrice, lte: maxPrice } } },
              { term: { is_active: true } }
            ]
          }
        }
      }
    });
    
    return hits.hits.map(hit => hit._source);
  }

  // Lấy sản phẩm nổi bật
  async getFeaturedProducts(limit = 10) {
    const { hits } = await this.esService.search({
      index: 'products',
      size: limit,
      body: {
        query: {
          bool: {
            must: [
              { term: { is_featured: true } },
              { term: { is_active: true } }
            ]
          }
        },
        sort: [{ view_count: 'desc' }]
      }
    });
    
    return hits.hits.map(hit => hit._source);
  }
}
```

### 2.4. Tích hợp Logger với Logstash

**Cài đặt:**

```bash
npm install winston winston-logstash
```

**File `src/logger/winston.config.ts`:**

```typescript
import * as winston from 'winston';
import LogstashTransport from 'winston-logstash/lib/winston-logstash-latest';

export const logger = winston.createLogger({
  transports: [
    new winston.transports.Console(),
    new LogstashTransport({
      port: 5000,
      host: 'localhost',
    }),
  ],
});
```

**Sử dụng trong code:**

```typescript
import { logger } from './logger/winston.config';

logger.info('User logged in', { userId: 123 });
logger.error('Payment failed', { orderId: 456, error: 'timeout' });
```

---

## 3. Xem dữ liệu trên Kibana

### 3.1. Truy cập Kibana

Mở trình duyệt: http://localhost:5601

### 3.2. Tạo Index Pattern

**Cho logs từ ứng dụng:**

1. Vào **Management** → **Stack Management** → **Index Patterns**
2. Click **Create index pattern**
3. Nhập: `nestjs-logs-*`
4. Chọn Time field: `@timestamp`
5. Click **Create**

**Cho dữ liệu từ PostgreSQL:**

1. Tạo index pattern mới
2. Nhập: `products` (hoặc `users`, tùy table bạn đồng bộ)
3. Chọn Time field: `updated_at` hoặc skip nếu không có
4. Click **Create**

### 3.3. Xem dữ liệu

**Xem Logs:**

1. Vào menu **Discover**
2. Chọn index pattern `nestjs-logs-*`
3. Bạn sẽ thấy tất cả logs từ ứng dụng

**Xem dữ liệu PostgreSQL:**

1. Vào menu **Discover**
2. Chọn index pattern `products`
3. Bạn sẽ thấy tất cả dữ liệu từ bảng PostgreSQL được đồng bộ

### 3.4. Lọc dữ liệu

Sử dụng KQL (Kibana Query Language):

**Lọc logs:**
```
level: "error"
service: "nestjs-app"
userId: 123
```

**Lọc dữ liệu Products từ PostgreSQL:**
```
# Tìm sản phẩm theo tên
name: *laptop*

# Lọc theo category
category_id: 5

# Lọc theo giá
price >= 100 AND price <= 500

# Chỉ sản phẩm đang active
is_active: true

# Sản phẩm nổi bật
is_featured: true

# Sản phẩm còn hàng
stock_status: "in_stock"

# Rating cao
rating_average >= 4.0
```

---

## 4. Testing

### 4.1. Test Elasticsearch

```bash
# Kiểm tra health
curl http://localhost:9200/_cluster/health

# Xem indices
curl http://localhost:9200/_cat/indices

# Tìm kiếm
curl http://localhost:9200/products/_search?q=name:laptop
```

### 4.2. Test Logstash

**Test nhận logs qua TCP:**

```bash
# Gửi test log (Linux/Mac)
echo '{"message":"test","level":"info"}' | nc localhost 5000
```

**Test đồng bộ từ PostgreSQL:**

```bash
# Kiểm tra Logstash logs để xem có lỗi không
docker logs logstash -f

# Kiểm tra dữ liệu đã được index chưa
curl http://localhost:9200/products/_search?pretty

# Xem số lượng documents
curl http://localhost:9200/products/_count
```

### 4.3. Test trong NestJS

**File `src/health/health.controller.ts`:**

```typescript
import { Controller, Get } from '@nestjs/common';
import { ElasticsearchService } from '@nestjs/elasticsearch';

@Controller('health')
export class HealthController {
  constructor(private readonly es: ElasticsearchService) {}

  @Get('elasticsearch')
  async check() {
    try {
      const health = await this.es.cluster.health();
      return { status: 'ok', health };
    } catch (error) {
      return { status: 'error', message: error.message };
    }
  }
}
```

---

## 5. Workflow tổng quan

### Luồng dữ liệu từ PostgreSQL → Elasticsearch → Kibana

```
PostgreSQL (products table)
         ↓
    Logstash (đồng bộ mỗi 30s)
         ↓
    Elasticsearch (index: products)
         ↓
    ┌──────────────┴──────────────┐
    ↓                             ↓
  Kibana                    NestJS API
  (xem trực quan)          (search, filter)
```

### Các bước triển khai nhanh

1. **Setup ELK Stack**
   ```bash
   docker-compose up -d
   ```

2. **Tạo cấu trúc thư mục Logstash**
   ```bash
   mkdir -p logstash/{config,pipeline,drivers,templates}
   ```

3. **Download JDBC driver**
   ```bash
   cd logstash/drivers
   wget https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
   ```

4. **Tạo các file cấu hình** (như đã hướng dẫn ở trên)

5. **Khởi động lại Logstash**
   ```bash
   docker-compose restart logstash
   ```

6. **Kiểm tra dữ liệu**
   ```bash
   curl http://localhost:9200/products/_search?pretty
   ```

7. **Tạo Index Pattern trên Kibana** (http://localhost:5601)

8. **Tích hợp vào NestJS** và sử dụng!
