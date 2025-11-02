### 📓 PHẦN 8: BULK INDEX, SCROLL SEARCH & DATA SYNC

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Biết cách **index hàng loạt dữ liệu (Bulk API)** – nhanh hơn 100 lần so với save từng record.
2. Hiểu và dùng **Scroll Search / Search After** cho dữ liệu lớn (pagination hàng triệu bản ghi).
3. Thiết lập chiến lược **đồng bộ dữ liệu giữa Database (MySQL/PostgreSQL) và Elasticsearch**.
4. Nắm các **kỹ thuật tối ưu hiệu năng indexing** trong hệ thống thực tế.

---

## 🧩 1. **Vấn đề: Index từng record quá chậm**

Nếu bạn gọi:

```java
productRepository.save(product);
```

với 100.000 bản ghi → mỗi lần gửi 1 request HTTP → **chậm, không thực tế**.

➡️ Giải pháp: **Bulk API** — cho phép gửi nhiều thao tác (index, update, delete) chỉ trong 1 request.

---

## ⚙️ 2. **Bulk Index trong Kibana Dev Tools**

```bash
POST _bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "iPhone 16 Pro", "category": "smartphone", "price": 1399 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "MacBook Pro M3", "category": "laptop", "price": 2499 }
{ "index": { "_index": "products", "_id": "3" } }
{ "name": "AirPods Pro 2", "category": "accessory", "price": 249 }
```

> ⚠️ Mỗi cặp `{ action }` + `{ document }` trên **phải tách bằng newline**, không có dấu `,`.

---

## 💻 3. **Bulk Index trong Spring Boot (Elasticsearch Java Client)**

**`BulkIndexService.java`**

```java
package com.example.elasticsearch.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch.core.bulk.BulkOperation;
import co.elastic.clients.elasticsearch.core.BulkRequest;
import co.elastic.clients.elasticsearch.core.BulkResponse;
import com.example.elasticsearch.model.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.io.IOException;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class BulkIndexService {

    private final ElasticsearchClient client;

    public void bulkIndex(List<Product> products) throws IOException {

        List<BulkOperation> operations = products.stream()
                .map(product -> BulkOperation.of(op -> op
                        .index(idx -> idx
                                .index("products")
                                .id(product.getId())
                                .document(product)
                        )
                ))
                .collect(Collectors.toList());

        BulkRequest request = BulkRequest.of(b -> b.operations(operations));
        BulkResponse response = client.bulk(request);

        if (response.errors()) {
            response.items().forEach(item -> {
                if (item.error() != null) {
                    System.err.println("❌ Error indexing: " + item.error().reason());
                }
            });
        } else {
            System.out.println("✅ Indexed " + products.size() + " products successfully!");
        }
    }
}
```

---

### 🧠 **Khi nào nên dùng Bulk**

| Trường hợp                     | Giải pháp                     |
| ------------------------------ | ----------------------------- |
| Import dữ liệu lần đầu (100k+) | ✅ Bulk API                    |
| Đồng bộ batch mỗi 5 phút       | ✅ Bulk                        |
| Tìm kiếm real-time (CRUD nhỏ)  | ❌ Save trực tiếp (repository) |

---

## 🧮 4. **Tối ưu hiệu năng khi Bulk Index**

| Cấu hình                      | Ý nghĩa                         | Gợi ý                                      |
| ----------------------------- | ------------------------------- | ------------------------------------------ |
| `refresh_interval`            | ES làm mới index để search được | Đặt `-1` trong quá trình bulk, bật lại sau |
| `number_of_replicas`          | Sao lưu dữ liệu                 | Giảm xuống `0` khi bulk, tăng lại sau      |
| `thread_pool.bulk.queue_size` | Hàng đợi bulk                   | Tăng nếu thấy “rejected execution”         |
| `batch_size`                  | Số bản ghi mỗi batch            | 500–2000 record là tối ưu                  |

Ví dụ tạm tắt refresh khi bulk:

```bash
PUT /products/_settings
{
  "index": {
    "refresh_interval": "-1",
    "number_of_replicas": 0
  }
}
```

Sau khi xong:

```bash
PUT /products/_settings
{
  "index": {
    "refresh_interval": "1s",
    "number_of_replicas": 1
  }
}
```

---

## 🔍 5. **Scroll Search – Duyệt dữ liệu lớn (Deep Pagination)**

Pagination thông thường (`from + size`) chỉ hiệu quả < 10.000 documents.
Để đọc hàng triệu dòng → dùng **Scroll Search** hoặc **Search After**.

---

### 🔹 Scroll Search trong Kibana Dev Tools

```bash
POST /products/_search?scroll=1m
{
  "size": 1000,
  "query": { "match_all": {} }
}
```

Phản hồi:

```json
{
  "_scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAA...",
  "hits": { "hits": [ ... 1000 docs ... ] }
}
```

Tiếp tục gọi:

```bash
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAA..."
}
```

> Lặp lại cho đến khi không còn `hits` nào nữa.
> Cuối cùng gọi `_search/scroll` DELETE để dọn cache.

---

### 🔹 Scroll Search bằng Java Client

```java
String scrollId = null;
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .size(1000)
    .scroll(Time.of(t -> t.time("1m")))
    .query(q -> q.matchAll(m -> m)),
    Product.class
);

scrollId = response.scrollId();

while (response.hits().hits().size() > 0) {
    for (Hit<Product> hit : response.hits().hits()) {
        System.out.println(hit.source().getName());
    }

    SearchResponse<Product> next = client.scroll(sc -> sc
        .scrollId(scrollId)
        .scroll(Time.of(t -> t.time("1m"))),
        Product.class
    );

    scrollId = next.scrollId();
    response = next;
}
```

---

## ⚙️ 6. **Search After – Pagination ổn định (API kiểu infinite scroll)**

Phù hợp cho các API “load more” của frontend.

```bash
GET products/_search
{
  "size": 10,
  "sort": [{ "price": "asc" }, { "_id": "desc" }],
  "search_after": [2499.99, "product_100"]
}
```

> `search_after` nhận giá trị từ document cuối của trang trước → pagination ổn định hơn `from`.

---

## 🔁 7. **Data Sync giữa Database và Elasticsearch**

Trong thực tế, dữ liệu **gốc** lưu ở MySQL/PostgreSQL, còn ES chỉ để search.
→ Cần **đồng bộ hóa (synchronize)** khi DB thay đổi.

---

### 🔹 3 mô hình đồng bộ phổ biến

| Mô hình                           | Cách hoạt động                                                           | Gợi ý dùng                        |
| --------------------------------- | ------------------------------------------------------------------------ | --------------------------------- |
| **(1) Application-level Sync**    | Khi CRUD DB → đồng thời update ES                                        | Hệ thống nhỏ, real-time           |
| **(2) Batch Sync**                | Cron job định kỳ (mỗi X phút) đọc DB → Bulk lên ES                       | Dữ liệu lớn, delay chấp nhận được |
| **(3) Change Data Capture (CDC)** | Dùng Debezium, Kafka Connect, hoặc Logstash đọc binlog DB → sync tự động | Hệ thống lớn, scale cao           |

---

### 🔹 Ví dụ Batch Sync cơ bản (Spring Boot Scheduler)

```java
@Service
@RequiredArgsConstructor
public class DataSyncService {

    private final ProductRepository productRepository;
    private final BulkIndexService bulkIndexService;
    private final ProductJdbcRepository productJdbcRepository;

    @Scheduled(fixedDelay = 300000) // 5 phút
    public void syncDatabaseToElasticsearch() throws IOException {
        List<Product> products = productJdbcRepository.findUpdatedSinceLastSync();
        if (!products.isEmpty()) {
            bulkIndexService.bulkIndex(products);
            System.out.println("✅ Synced " + products.size() + " products to Elasticsearch.");
        }
    }
}
```

> ⚙️ Nếu hệ thống lớn → chuyển sang Debezium + Kafka → ES Sink Connector.

---

## 🧠 8. **Best Practices khi xử lý dữ liệu lớn**

| Vấn đề                           | Giải pháp                                            |
| -------------------------------- | ---------------------------------------------------- |
| Import 1 triệu bản ghi           | Dùng Bulk API + batch 1000 records/lần               |
| Cần phân trang sâu               | Dùng Scroll hoặc Search After                        |
| Dữ liệu DB thay đổi thường xuyên | Dùng CDC (Debezium, Kafka)                           |
| Cluster chậm khi bulk            | Giảm replica, tắt refresh, tăng batch size           |
| Cần rollback dữ liệu sai         | Dùng index alias (vd: `products_v1` → `products_v2`) |

---

## 🧩 **Bài tập thực hành**

1️⃣ Tạo danh sách 10.000 sản phẩm (mock data bằng Faker).
2️⃣ Dùng `BulkIndexService.bulkIndex()` để import nhanh vào Elasticsearch.
3️⃣ Viết API `/api/products/scroll` để duyệt toàn bộ dữ liệu bằng Scroll Search.
4️⃣ Viết batch job (Scheduler) để đồng bộ sản phẩm mới từ DB sang Elasticsearch mỗi 10 phút.
