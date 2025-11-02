### 📜 PHẦN 10: TRIỂN KHAI SEARCH API HOÀN CHỈNH & ROADMAP NÂNG CAO

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Xây dựng **Search API hoàn chỉnh trong Spring Boot** với:

   * 🔍 Full-text search
   * 🎨 Highlight
   * ⚙️ Filter (category, price range)
   * 📊 Aggregation (facets)
   * 📑 Pagination + Sorting
2. Hiểu cách triển khai **reindex không downtime** bằng alias.
3. Nắm được **lộ trình nâng cao** để trở thành *Elasticsearch Search Engineer chuyên nghiệp*.

---

## 🧩 1. **Mô hình tổng quan Search API**

```
Frontend (React/Vue)
        ↓
   /api/search?q=macbook&category=laptop&min=1000&max=2500&page=1&sort=price_asc
        ↓
Spring Boot SearchController
        ↓
SearchService (ElasticsearchClient)
        ↓
Elasticsearch Cluster (index: products_v2)
```

---

## 📘 2. **DTO – Request & Response**

### `SearchRequestDTO.java`

```java
package com.example.elasticsearch.dto;

import lombok.Data;

@Data
public class SearchRequestDTO {
    private String q;              // từ khóa
    private String category;       // filter danh mục
    private Double minPrice;       // filter giá tối thiểu
    private Double maxPrice;       // filter giá tối đa
    private String sort;           // ví dụ: price_asc, price_desc
    private int page = 1;
    private int size = 10;
}
```

---

### `SearchResponseDTO.java`

```java
package com.example.elasticsearch.dto;

import lombok.*;
import java.util.*;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class SearchResponseDTO {
    private long total;
    private int page;
    private int size;
    private List<Map<String, Object>> results;
    private Map<String, Long> categoryStats;
}
```

---

## 📗 3. **Service – Elasticsearch Query Builder**

**`SearchService.java`**

```java
package com.example.elasticsearch.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.SortOrder;
import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import co.elastic.clients.elasticsearch.core.SearchResponse;
import co.elastic.clients.json.JsonData;
import com.example.elasticsearch.dto.SearchRequestDTO;
import com.example.elasticsearch.dto.SearchResponseDTO;
import com.example.elasticsearch.model.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.io.IOException;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class SearchService {

    private final ElasticsearchClient client;

    public SearchResponseDTO searchProducts(SearchRequestDTO request) throws IOException {
        int from = (request.getPage() - 1) * request.getSize();

        // === 1️⃣ Xây dựng Bool Query ===
        List<Query> must = new ArrayList<>();
        List<Query> filters = new ArrayList<>();

        if (request.getQ() != null && !request.getQ().isBlank()) {
            must.add(Query.of(q -> q
                .multiMatch(m -> m
                    .fields("name", "description")
                    .query(request.getQ())
                )));
        } else {
            must.add(Query.of(q -> q.matchAll(m -> m)));
        }

        if (request.getCategory() != null) {
            filters.add(Query.of(q -> q.term(t -> t.field("category.keyword").value(request.getCategory()))));
        }
        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            filters.add(Query.of(q -> q.range(r -> r
                .field("price")
                .gte(request.getMinPrice() != null ? JsonData.of(request.getMinPrice()) : null)
                .lte(request.getMaxPrice() != null ? JsonData.of(request.getMaxPrice()) : null)
            )));
        }

        // === 2️⃣ Sort ===
        SortOrder order = SortOrder.Asc;
        String sortField = "_score";
        if (request.getSort() != null) {
            if (request.getSort().equalsIgnoreCase("price_desc")) {
                sortField = "price"; order = SortOrder.Desc;
            } else if (request.getSort().equalsIgnoreCase("price_asc")) {
                sortField = "price"; order = SortOrder.Asc;
            }
        }

        // === 3️⃣ Gửi query ===
        SearchResponse<Product> response = client.search(s -> s
            .index("products") // hoặc alias (vd: products_current)
            .from(from)
            .size(request.getSize())
            .query(q -> q.bool(b -> b.must(must).filter(filters)))
            .highlight(h -> h
                .fields("name", f -> f)
                .fields("description", f -> f)
                .preTags("<mark>")
                .postTags("</mark>")
            )
            .sort(so -> so.field(f -> f.field(sortField).order(order)))
            .aggregations("by_category", a -> a
                .terms(t -> t.field("category.keyword"))
            ),
            Product.class
        );

        // === 4️⃣ Chuyển kết quả ===
        List<Map<String, Object>> results = response.hits().hits().stream()
            .map(hit -> {
                Map<String, Object> map = new HashMap<>();
                Product p = hit.source();
                map.put("id", p.getId());
                map.put("name", p.getName());
                map.put("price", p.getPrice());
                map.put("category", p.getCategory());
                map.put("description", p.getDescription());
                map.put("highlight", hit.highlight());
                return map;
            })
            .collect(Collectors.toList());

        Map<String, Long> categoryStats = new LinkedHashMap<>();
        response.aggregations().get("by_category").sterms().buckets().array()
            .forEach(bucket -> categoryStats.put(bucket.key().stringValue(), bucket.docCount()));

        return SearchResponseDTO.builder()
            .total(response.hits().total().value())
            .page(request.getPage())
            .size(request.getSize())
            .results(results)
            .categoryStats(categoryStats)
            .build();
    }
}
```

---

## 📙 4. **Controller – REST Endpoint**

```java
package com.example.elasticsearch.controller;

import com.example.elasticsearch.dto.SearchRequestDTO;
import com.example.elasticsearch.dto.SearchResponseDTO;
import com.example.elasticsearch.service.SearchService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/search")
@RequiredArgsConstructor
public class SearchController {

    private final SearchService searchService;

    @GetMapping
    public SearchResponseDTO search(SearchRequestDTO request) throws Exception {
        return searchService.searchProducts(request);
    }
}
```

---

### ✅ Demo Request (Postman)

```
GET http://localhost:8080/api/search?q=macbook&category=laptop&minPrice=1000&maxPrice=3000&sort=price_asc&page=1&size=5
```

### ✅ Response

```json
{
  "total": 14,
  "page": 1,
  "size": 5,
  "results": [
    {
      "id": "1",
      "name": "MacBook Air M3",
      "price": 1499.0,
      "highlight": {
        "name": ["<mark>MacBook</mark> Air M3"]
      }
    }
  ],
  "categoryStats": {
    "laptop": 10,
    "smartphone": 4
  }
}
```

---

## 🧱 5. **Triển khai Alias & Reindex không downtime**

Khi bạn muốn thay đổi mapping, ES **không cho sửa trực tiếp** — cần tạo index mới và chuyển alias.

### Bước 1️⃣ – Tạo index mới

```bash
PUT /products_v2
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "standard" },
      "price": { "type": "float" },
      "category": { "type": "keyword" }
    }
  }
}
```

### Bước 2️⃣ – Reindex dữ liệu cũ

```bash
POST _reindex
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" }
}
```

### Bước 3️⃣ – Gán alias

```bash
POST _aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" }},
    { "add":    { "index": "products_v2", "alias": "products" }}
  ]
}
```

➡️ Từ nay, app luôn truy cập `products` → không downtime.
➡️ Dễ rollback chỉ bằng đổi alias ngược lại.

---

## ⚙️ 6. **Triển khai thực tế (Production Tips)**

| Thành phần            | Gợi ý                                                          |
| --------------------- | -------------------------------------------------------------- |
| **Index alias**       | Luôn truy cập qua alias (`products_current`) để dễ versioning. |
| **Refresh interval**  | 1–5s tùy hệ thống (đừng để 1s nếu ghi nhiều).                  |
| **Backup & Snapshot** | Dùng `snapshot API` → lưu vào S3 / NFS định kỳ.                |
| **Cluster scale**     | 3 node (1 master, 2 data) là tối thiểu cho high availability.  |
| **Security**          | Bật TLS & user/password trong `xpack.security.enabled: true`.  |
| **Logging**           | Bật slowlog (query + fetch > 2s) để phát hiện bottleneck.      |

---

## 📈 7. **Roadmap nâng cao sau khi thành thạo**

### 🔹 Elastic Stack Integration

| Thành phần                       | Mục đích                                   |
| -------------------------------- | ------------------------------------------ |
| **Logstash**                     | Xử lý & đẩy dữ liệu từ nhiều nguồn vào ES. |
| **Beats (Filebeat, Metricbeat)** | Thu thập log, metrics hệ thống.            |
| **Kibana**                       | Phân tích & visualization dữ liệu.         |
| **Elastic APM**                  | Theo dõi hiệu năng ứng dụng.               |

---

### 🔹 Nâng cao kỹ thuật Search

| Chủ đề                              | Mô tả                                                 |
| ----------------------------------- | ----------------------------------------------------- |
| **Search Suggesters**               | Gợi ý từ khóa (“Bạn có ý định tìm ‘macbook’ không?”). |
| **Synonyms & Language analyzers**   | Mở rộng tìm kiếm tiếng Việt, tiếng Anh, đa ngôn ngữ.  |
| **Vector Search / Semantic Search** | Dùng embedding + cosine similarity (Elastic 8.10+).   |
| **Rank Feature & Function Score**   | Xếp hạng kết quả theo độ ưu tiên, độ nổi bật.         |
| **Relevance Tuning**                | Điều chỉnh điểm BM25, boosting field, decay function. |

---

### 🔹 Mở rộng quy mô hệ thống

| Mục tiêu           | Công nghệ                                        |
| ------------------ | ------------------------------------------------ |
| Real-time Sync     | Kafka + Debezium + Logstash                      |
| Distributed Search | Multi-node, cross-cluster search                 |
| Monitoring & Alert | ElasticHQ, Grafana, Prometheus                   |
| Machine Learning   | Elastic ML Jobs (anomaly detection, forecasting) |

---

## 🧠 **Checklist tổng kết toàn bộ khóa học**

| Học phần                 | Bạn đã nắm được                         |
| ------------------------ | --------------------------------------- |
| 📘 Kiến trúc & cơ chế ES | ✅ Cluster, Node, Index, Shard           |
| 📗 Cấu hình Spring Boot  | ✅ Kết nối, Repository, Client           |
| 📙 Mapping & Analyzer    | ✅ Phân tích text, custom analyzer       |
| 📒 CRUD & Java Client    | ✅ Ghi, đọc, xóa, cập nhật               |
| 📔 Query DSL             | ✅ match, bool, fuzzy, range             |
| 📕 Search nâng cao       | ✅ Highlight, Pagination, Sorting        |
| 📚 Aggregation & Metrics | ✅ Grouping, Thống kê, Range             |
| 📓 Bulk & Scroll         | ✅ Import nhanh, deep pagination         |
| 📖 Monitoring & Tuning   | ✅ Debug, optimize, check cluster health |
| 📜 Search API hoàn chỉnh | ✅ Full-featured search REST API         |

---

## 💼 **Bộ công cụ nên thành thạo tiếp theo**

| Công cụ                      | Vai trò                       |
| ---------------------------- | ----------------------------- |
| 🧩 **Kibana**                | Dev Tools, Monitor, Dashboard |
| 🧠 **ElasticHQ**             | UI quản lý cluster            |
| ⚙️ **Logstash**              | Đẩy & transform dữ liệu       |
| 🧾 **Filebeat / Metricbeat** | Thu thập log, metrics         |
| 📈 **Elastic APM**           | Theo dõi hiệu năng ứng dụng   |
| 🛡️ **Elastic Security**     | Phát hiện, cảnh báo an ninh   |

---

## 🚀 **Kết luận**

Bạn đã hoàn thành toàn bộ lộ trình **“Elasticsearch trong Spring Boot từ cơ bản đến chuyên sâu”**.
Giờ đây, bạn có thể:

* Tự xây dựng **search engine hoàn chỉnh** như Shopee, Tiki, Medium, hay bất kỳ hệ thống tìm kiếm nào.
* Tối ưu hóa tốc độ, độ chính xác và khả năng mở rộng.
* Triển khai **search-as-a-service** tích hợp vào microservice thực tế.

---

### 🧩 **Bài tập cuối cùng:**

1️⃣ Hoàn thiện API `/api/search` với:

* Full-text search (`multi_match`)
* Filter (category, price range)
* Sort (price/date/relevance)
* Highlight
* Aggregation (category count)

2️⃣ Tạo alias `products_current` để trỏ tới index mới.
3️⃣ Triển khai trên Docker Compose gồm:

* Elasticsearch
* Kibana
* Spring Boot
  4️⃣ Dùng Postman test với 3 query khác nhau → đo thời gian phản hồi.
