### 📘 PHẦN 1: TỔNG QUAN ELASTICSEARCH & KIẾN TRÚC HỆ THỐNG

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

* Hiểu **Elasticsearch là gì**, dùng để làm gì trong hệ sinh thái Spring Boot.
* Nắm được **kiến trúc nội tại** của Elasticsearch: cluster, node, index, shard, replica.
* Biết **vòng đời dữ liệu** (từ index → search → update → delete).
* Hiểu cơ chế **phân tán, full-text search, scoring và inverted index**.
* Làm quen với **Elasticsearch REST API** và công cụ quản trị **Kibana**.

---

#### 🧠 **1. Giới thiệu về Elasticsearch**

Elasticsearch (ES) là **search & analytics engine phân tán**, mã nguồn mở, xây dựng trên nền **Apache Lucene**.
Nó không phải cơ sở dữ liệu quan hệ mà là **document store** tối ưu cho **tìm kiếm toàn văn, phân tích dữ liệu, log, và metrics**.

Ứng dụng thực tế:

* 🔍 Tìm kiếm sản phẩm, bài viết, hồ sơ người dùng.
* 📊 Phân tích log hệ thống (Elastic Stack: Elasticsearch + Logstash + Kibana).
* 🧾 Recommendation, Autocomplete, Suggestion.
* ⚙️ Phân tích dữ liệu lớn, dashboard thời gian thực.

---

#### ⚙️ **2. Kiến trúc tổng thể của Elasticsearch**

```
+---------------------------------------------------+
|                 Elasticsearch Cluster             |
|---------------------------------------------------|
|  Node 1 (Master + Data)   Node 2 (Data)           |
|  Node 3 (Ingest + Coordinator)                    |
+---------------------------------------------------+
       ↓
    Index: product_index
       ↓
   +-----------------------------+
   | Primary Shards   | Replicas |
   | Shard 0, 1, 2    | 0’, 1’, 2’|
   +-----------------------------+
       ↓
     Documents (JSON)
```

**Giải thích các thành phần:**

| Thành phần   | Mô tả                                                                  |
| ------------ | ---------------------------------------------------------------------- |
| **Cluster**  | Nhóm các node làm việc cùng nhau, chia sẻ dữ liệu và tải.              |
| **Node**     | Một instance Elasticsearch chạy độc lập (trên 1 máy hoặc container).   |
| **Index**    | Tương đương “database” – nơi chứa các document cùng cấu trúc.          |
| **Document** | Một bản ghi (JSON) đại diện cho 1 sản phẩm, user, blog, v.v.           |
| **Shard**    | Elasticsearch chia index thành nhiều phần nhỏ để phân tán và tăng tốc. |
| **Replica**  | Bản sao của shard để dự phòng và tăng tốc độ đọc.                      |
| **Analyzer** | Xử lý text (tách từ, bỏ stopwords, lowercase...) khi index/search.     |

---

#### 🔍 **3. Cơ chế hoạt động của Elasticsearch**

##### (1) **Inverted Index (chìa khóa của tìm kiếm toàn văn)**

Thay vì lưu từng document nguyên bản, ES lưu **bảng ánh xạ từ → danh sách document chứa từ đó**.

```
Document 1: "Spring Boot Elasticsearch tutorial"
Document 2: "Learn Elasticsearch with Spring Data"
Document 3: "Spring Boot microservices"

=> Inverted Index:

Term             | Document IDs
-----------------|--------------
spring           | 1, 2, 3
boot             | 1, 3
elasticsearch    | 1, 2
tutorial         | 1
microservices    | 3
```

✅ Kết quả: tìm kiếm “spring boot” chỉ cần tra ngược bảng này → rất nhanh.

---

##### (2) **Scoring và Relevance**

Khi bạn tìm `"spring boot"`, ES không chỉ trả về document có chứa từ, mà còn **tính điểm (score)** theo mức độ liên quan:

* Từ khóa xuất hiện nhiều → điểm cao hơn.
* Từ khóa xuất hiện trong tiêu đề → điểm cao hơn.
* Độ phổ biến của từ → ảnh hưởng đến điểm.

Công thức tính điểm dựa trên mô hình **BM25** (phiên bản cải tiến của TF-IDF).

---

##### (3) **Flow dữ liệu**

```
📥 Indexing:
Client → Coordinator Node → Primary Shard → Replicas
📤 Searching:
Client → Coordinator Node → Query Shards → Merge Results → Return Top N
```

---

#### 🧩 **4. Các thao tác cơ bản với REST API**

Giả sử bạn có index `products`, ta thử thao tác cơ bản (dùng cURL hoặc Kibana Dev Tools):

**Tạo index:**

```bash
PUT /products
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "float" }
    }
  }
}
```

**Thêm document:**

```bash
POST /products/_doc/1
{
  "name": "Laptop Lenovo ThinkPad",
  "category": "electronics",
  "price": 1200
}
```

**Tìm kiếm đơn giản:**

```bash
GET /products/_search
{
  "query": {
    "match": { "name": "laptop" }
  }
}
```

**Xoá index:**

```bash
DELETE /products
```

---

#### 🧱 **5. Sơ đồ liên kết giữa Elasticsearch và Spring Boot**

```
+----------------------------+
| Spring Boot Application    |
|----------------------------|
| Controller (REST API)      |
| Service (Logic)            |
| Repository (Elasticsearch) |
|----------------------------|
| Spring Data Elasticsearch  |
|----------------------------|
| Elasticsearch Cluster (8.x)|
+----------------------------+
```

---

#### 🧩 **6. Bài tập nhỏ**

1️⃣ Cài **Elasticsearch 8.x** bằng Docker Compose:

```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - 9200:9200
```

Sau đó truy cập:
👉 [http://localhost:9200](http://localhost:9200)
Sẽ thấy JSON trả về:

```json
{
  "name" : "elasticsearch",
  "cluster_name" : "docker-cluster",
  "version" : { "number" : "8.12.0" }
}
```

2️⃣ Mở Kibana Dev Tools (nếu cài Kibana) và chạy thử:

```bash
GET _cat/indices?v
```

để xem danh sách index.

---

#### ⚠️ **7. Sai lầm thường gặp**

| Sai lầm                                                          | Giải thích                                                  |
| ---------------------------------------------------------------- | ----------------------------------------------------------- |
| Không phân biệt `text` vs `keyword`                              | `text` dùng để full-text search; `keyword` cho exact match. |
| Không thiết lập `analyzer` phù hợp                               | Gây sai lệch kết quả tìm kiếm.                              |
| Quên set `number_of_shards` và `replicas` đúng cách              | Ảnh hưởng hiệu năng và độ tin cậy.                          |
| Index lại (reindex) mà không dùng alias                          | Gây downtime.                                               |
| Nhầm lẫn giữa `ElasticsearchRepository` và `RestHighLevelClient` | Hai cách tiếp cận khác nhau trong Spring Boot.              |

---

#### ✅ **8. Best Practices & Checklist**

* [ ] Dùng **Docker Compose** để setup ES + Kibana cục bộ.
* [ ] Hiểu rõ **index vs document**.
* [ ] Làm quen với **Dev Tools** để chạy truy vấn nhanh.
* [ ] Thử tạo index, thêm document, xóa, tìm kiếm.
* [ ] Ghi nhớ **mapping, analyzer, shard, replica** là 4 nền tảng quan trọng nhất.
