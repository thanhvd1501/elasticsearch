### 📖 PHẦN 9: MONITORING, DEBUG & PERFORMANCE TUNING

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Biết cách **giám sát, kiểm tra sức khỏe của Elasticsearch cluster**.
2. Sử dụng các công cụ **Kibana Dev Tools**, **ElasticHQ**, và **_cat APIs** để theo dõi node, index, shard, heap.
3. Hiểu nguyên nhân khiến Elasticsearch **chậm, tốn RAM, search trễ** và cách tối ưu.
4. Nắm các **chiến lược tuning thực tế** cho production: index, mapping, caching, refresh, và shard allocation.

---

## 🧠 1. **Giám sát sức khỏe cluster**

Elasticsearch là hệ thống **phân tán**, nên giám sát thường xoay quanh 3 trục:

| Thành phần | Mục tiêu theo dõi                 | Công cụ                                 |
| ---------- | --------------------------------- | --------------------------------------- |
| Cluster    | Sức khỏe tổng thể, số node, shard | `_cluster/health`, Kibana               |
| Node       | CPU, heap memory, disk            | `_nodes/stats`, ElasticHQ               |
| Index      | Document count, size, shards      | `_cat/indices`, Kibana Index Management |

---

### 🔹 1.1. Kiểm tra sức khỏe cluster

```bash
GET _cluster/health
```

**Kết quả ví dụ:**

```json
{
  "cluster_name": "docker-cluster",
  "status": "green",
  "number_of_nodes": 1,
  "active_primary_shards": 5,
  "active_shards_percent_as_number": 100.0
}
```

| Status      | Ý nghĩa                                              |
| ----------- | ---------------------------------------------------- |
| 🟢 `green`  | Mọi shard (primary + replica) đều hoạt động.         |
| 🟡 `yellow` | Có shard replica chưa được gán (thường single-node). |
| 🔴 `red`    | Có shard primary bị mất — nguy hiểm!                 |

---

### 🔹 1.2. Kiểm tra danh sách index

```bash
GET _cat/indices?v
```

Kết quả (bảng dễ đọc):

```
health status index       uuid                   pri rep docs.count store.size
green  open   products    aw43wEJBSfSL2E8y9        1   1       1200    5.2mb
```

---

### 🔹 1.3. Kiểm tra node & heap

```bash
GET _nodes/stats/jvm,os,fs
```

> 🔍 Kiểm tra:
>
> * `jvm.mem.heap_used_percent` → Nếu > 75%, cần tăng heap hoặc giảm load.
> * `fs.total.available_in_bytes` → Dung lượng đĩa.
> * `os.cpu.percent` → CPU usage.

---

### 🔹 1.4. Kibana Dev Tools & Stack Monitoring

Kibana có 2 module cực hữu ích:

* **Dev Tools** → chạy query `_cat`, `_search`, `_mapping`.
* **Stack Monitoring** → hiển thị biểu đồ CPU, heap, search latency, indexing rate.

> Mở Kibana → Menu → **Stack Monitoring → Elasticsearch**.

---

## 🧩 2. **Phân tích log & debug query**

### 🔹 2.1. Kiểm tra log trong Docker

```bash
docker logs elasticsearch -f
```

> Log sẽ hiển thị GC (Garbage Collection), slow query, shard errors.

---

### 🔹 2.2. Kiểm tra slow queries

Bật log query chậm bằng cách thêm vào `elasticsearch.yml`:

```yaml
index.search.slowlog.threshold.query.warn: 5s
index.search.slowlog.threshold.query.info: 2s
index.search.slowlog.threshold.query.debug: 500ms
```

Sau đó xem log:

```bash
GET _cat/thread_pool/search?v
```

---

### 🔹 2.3. Debug mapping / analyzer

Trước khi blame performance, luôn check lại:

```bash
GET /products/_mapping
GET /products/_analyze
```

> Nếu bạn thấy field `text` bị analyzer sai (không split token đúng) → kết quả search sai và chậm.

---

## ⚙️ 3. **Chiến lược tối ưu hiệu năng**

| Thành phần           | Giải pháp tối ưu                                  | Ghi chú                                |
| -------------------- | ------------------------------------------------- | -------------------------------------- |
| **Index size**       | Gộp shard nhỏ, xóa index cũ                       | Mỗi shard tối ưu ~30–50 GB             |
| **Shard count**      | Không quá 1 shard / 20 GB                         | Tránh 1000 shard cho dữ liệu nhỏ       |
| **Refresh interval** | Tăng từ 1s → 5s hoặc tắt (`-1`) trong batch index | Tối ưu tốc độ ghi                      |
| **Replicas**         | Đặt 0 khi bulk, 1 khi online                      | Tăng tốc indexing                      |
| **Heap size**        | Set = 50% RAM, max 32GB                           | Tránh vượt 32GB vì mất compressed OOPs |
| **Analyzer**         | Dùng custom analyzer                              | Tránh analyzer “whitespace” đơn giản   |
| **Mapping**          | Sử dụng đúng type                                 | Tránh dynamic mapping sai kiểu         |
| **Query**            | Dùng `filter` thay vì `must` khi không cần score  | Filter được cache nhanh hơn            |
| **Cache**            | Dùng query cache & request cache                  | `GET _nodes/stats/indices/query_cache` |

---

## 🧱 4. **Tối ưu khi search nhiều điều kiện**

| Mục tiêu                                                   | Giải pháp                                  |
| ---------------------------------------------------------- | ------------------------------------------ |
| Tìm kiếm nhanh với filter cố định (category, brand, price) | Dùng `bool.filter` thay vì `must`          |
| Query lặp lại nhiều lần                                    | Bật `request_cache`                        |
| Search có nhiều synonym                                    | Tạo analyzer riêng với `synonym filter`    |
| Người dùng nhập sai chính tả                               | Dùng `fuzzy` hoặc `did_you_mean` suggester |

---

## ⚡ 5. **Theo dõi hiệu năng index & search**

### `_cat/indices`

→ số document, size, health per index.

### `_cat/nodes`

→ CPU, heap, load, shards per node.

### `_cat/shards`

→ vị trí từng shard, phát hiện shard imbalance.

### `_cat/pending_tasks`

→ Xem nếu cluster đang pending (chưa apply setting/mapping).

---

## 📊 6. **ElasticHQ – Giao diện quản lý cluster**

Cài ElasticHQ để có UI quản lý (chạy local hoặc Docker):

```bash
docker run -d -p 5000:5000 elastichq/elasticsearch-hq
```

Sau đó vào: [http://localhost:5000](http://localhost:5000)
→ Nhập host Elasticsearch (`http://localhost:9200`)
→ Xem dashboard: nodes, shards, heap, indices, pending tasks, slow queries, v.v.

---

## 🔍 7. **Kiểm thử & Benchmark**

### 🔹 Kiểm tra hiệu suất query

```bash
GET products/_search
{
  "profile": true,
  "query": { "match": { "name": "macbook" } }
}
```

> Elasticsearch sẽ trả chi tiết thời gian thực hiện từng phase:
>
> * **query time**
> * **fetch time**
> * **rewrite time**

→ Dùng để phát hiện “nút cổ chai”.

---

## 🧠 8. **Checklist Debug Production Issue**

| Triệu chứng        | Nguyên nhân thường gặp          | Hướng xử lý                             |
| ------------------ | ------------------------------- | --------------------------------------- |
| Search chậm        | Quá nhiều shard / analyzer nặng | Giảm shard, dùng filter cache           |
| Node CPU cao       | Query phức tạp, wildcard        | Tối ưu query, tránh leading wildcard    |
| Heap full          | GC không giải phóng được        | Tăng heap, tối ưu mapping, giảm replica |
| Index lag khi bulk | Refresh interval quá thấp       | Tắt refresh khi bulk                    |
| Mất dữ liệu index  | Không dùng alias khi reindex    | Sử dụng alias versioning                |
| Mapping conflict   | Nhiều field trùng tên khác type | Dùng template rõ ràng                   |

---

## ✅ **Best Practices tổng kết**

* [ ] Theo dõi cluster health mỗi ngày (`GET _cluster/health`).
* [ ] Dùng Kibana Monitoring hoặc ElasticHQ để xem CPU, heap.
* [ ] Bật slowlog cho index/search để phát hiện query tệ.
* [ ] Giới hạn mỗi index ≤ 50GB và shard ≤ 20GB.
* [ ] Tắt refresh và replica khi bulk index.
* [ ] Sử dụng alias để reindex không downtime.
* [ ] Đặt heap ~½ RAM, tối đa 32GB.

---

## 🧩 **Bài tập thực hành**

1️⃣ Chạy `GET _cat/indices?v` để xem index đang có.
2️⃣ Bật slowlog và thử chạy query “fuzzy” xem log ghi lại.
3️⃣ Dùng `GET _nodes/stats/jvm` để xem heap usage.
4️⃣ Cài ElasticHQ và kiểm tra shard, heap, index size.
5️⃣ Thử điều chỉnh `refresh_interval` từ `1s` → `5s` xem tác động.
