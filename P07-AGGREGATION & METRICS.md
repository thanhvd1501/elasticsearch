### 📚 PHẦN 7: AGGREGATION & METRICS

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Hiểu **Aggregation** là gì và vì sao nó là trái tim của *analytics trong Elasticsearch*.
2. Phân biệt rõ các loại aggregation:

   * 🪣 **Bucket Aggregation** – nhóm dữ liệu (như SQL `GROUP BY`)
   * 📏 **Metric Aggregation** – tính toán thống kê (min, max, avg, sum, count)
   * 🔗 **Pipeline Aggregation** – xử lý kết quả của các aggregation khác
3. Viết được truy vấn aggregation trong Kibana và Java Client.
4. Xây dựng API thống kê dữ liệu sản phẩm thực tế (VD: đếm sản phẩm theo danh mục, tính giá trung bình, v.v.).

---

## 🧠 1. **Aggregation là gì?**

> Elasticsearch Aggregation = “**GROUP BY + HAVING + SUM/AVG/MIN/MAX**” của SQL, nhưng nhanh hơn rất nhiều.

Elasticsearch **không chỉ là công cụ tìm kiếm** – nó còn là engine phân tích dữ liệu real-time.
Khi bạn tìm kiếm sản phẩm, hệ thống có thể đồng thời trả về:

* Số lượng sản phẩm theo danh mục.
* Giá trung bình của sản phẩm.
* Biểu đồ phân bố giá.

---

### 🔹 Ví dụ trực quan

| Sản phẩm    | Danh mục   | Giá  |
| ----------- | ---------- | ---- |
| MacBook Pro | Laptop     | 2500 |
| MacBook Air | Laptop     | 1600 |
| iPhone 16   | Smartphone | 1400 |
| iPhone 15   | Smartphone | 999  |

**Aggregation mục tiêu:**

* Laptop: trung bình 2050
* Smartphone: trung bình 1199.5

---

## 📘 2. **Bucket Aggregation – Nhóm dữ liệu**

### 🔹 Ví dụ trong Kibana Dev Tools

```bash
GET products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword" }
    }
  }
}
```

**Kết quả:**

```json
{
  "aggregations": {
    "by_category": {
      "buckets": [
        { "key": "laptop", "doc_count": 10 },
        { "key": "smartphone", "doc_count": 8 }
      ]
    }
  }
}
```

📊 Tương đương SQL:

```sql
SELECT category, COUNT(*) FROM products GROUP BY category;
```

---

### 🔹 Trong Java Client

```java
SearchResponse<Void> response = client.search(s -> s
    .index("products")
    .size(0) // chỉ trả về aggregation, không cần document
    .aggregations("by_category", a -> a
        .terms(t -> t.field("category.keyword"))
    ),
    Void.class
);

response.aggregations().get("by_category")
    .sterms().buckets().array()
    .forEach(bucket ->
        System.out.println(bucket.key().stringValue() + ": " + bucket.docCount())
    );
```

---

## 📗 3. **Metric Aggregation – Tính toán thống kê**

### 🔹 Ví dụ: Giá trung bình, cao nhất, thấp nhất

```bash
GET products/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "max_price": { "max": { "field": "price" } },
    "min_price": { "min": { "field": "price" } }
  }
}
```

**Kết quả:**

```json
{
  "aggregations": {
    "avg_price": { "value": 1549.3 },
    "max_price": { "value": 2999 },
    "min_price": { "value": 499 }
  }
}
```

---

### 🔹 Trong Java Client

```java
SearchResponse<Void> response = client.search(s -> s
    .index("products")
    .size(0)
    .aggregations("avg_price", a -> a.avg(avg -> avg.field("price")))
    .aggregations("max_price", a -> a.max(max -> max.field("price")))
    .aggregations("min_price", a -> a.min(min -> min.field("price"))),
    Void.class
);

System.out.println("Avg: " + response.aggregations().get("avg_price").avg().value());
System.out.println("Max: " + response.aggregations().get("max_price").max().value());
System.out.println("Min: " + response.aggregations().get("min_price").min().value());
```

---

## 📙 4. **Nested Aggregation – Nhóm + Tính toán**

Ví dụ: Đếm số sản phẩm từng danh mục và tính giá trung bình của mỗi nhóm.

```bash
GET products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

**Kết quả:**

```json
{
  "aggregations": {
    "by_category": {
      "buckets": [
        { "key": "laptop", "doc_count": 10, "avg_price": { "value": 1850.5 } },
        { "key": "smartphone", "doc_count": 8, "avg_price": { "value": 1220.8 } }
      ]
    }
  }
}
```

> ✅ Đây chính là logic tạo **bộ lọc phân loại (faceted filter)** trong các trang e-commerce.

---

### 🔹 Java Client tương đương

```java
SearchResponse<Void> response = client.search(s -> s
    .index("products")
    .size(0)
    .aggregations("by_category", a -> a
        .terms(t -> t.field("category.keyword"))
        .aggregations("avg_price", sub -> sub.avg(v -> v.field("price")))
    ),
    Void.class
);

response.aggregations().get("by_category")
    .sterms().buckets().array()
    .forEach(bucket -> {
        String category = bucket.key().stringValue();
        double avgPrice = bucket.aggregations().get("avg_price").avg().value();
        System.out.println(category + " → trung bình $" + avgPrice);
    });
```

---

## 📓 5. **Range Aggregation – Phân loại theo khoảng giá**

Ví dụ: nhóm sản phẩm theo các khoảng giá:

```bash
GET products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 500 },
          { "from": 500, "to": 1500 },
          { "from": 1500, "to": 2500 },
          { "from": 2500 }
        ]
      }
    }
  }
}
```

**Kết quả:**

```json
{
  "aggregations": {
    "price_ranges": {
      "buckets": [
        { "key": "*-500", "doc_count": 3 },
        { "key": "500-1500", "doc_count": 8 },
        { "key": "1500-2500", "doc_count": 5 },
        { "key": "2500-*", "doc_count": 2 }
      ]
    }
  }
}
```

📊 Dùng để tạo thanh “lọc theo giá” trong e-commerce.

---

## 📏 6. **Pipeline Aggregation – Xử lý kết quả trung gian**

Ví dụ: Tính **trung bình cộng của trung bình giá từng danh mục**:

```bash
GET products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    },
    "avg_of_avgs": {
      "avg_bucket": { "buckets_path": "by_category>avg_price" }
    }
  }
}
```

Kết quả:

```json
{
  "aggregations": {
    "avg_of_avgs": { "value": 1535.6 }
  }
}
```

---

## 🧩 7. **Kết hợp Aggregation + Filter**

Tính giá trung bình **theo danh mục**, chỉ tính cho sản phẩm có giá < 2000.

```bash
GET products/_search
{
  "size": 0,
  "query": {
    "range": { "price": { "lte": 2000 } }
  },
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

---

## ⚠️ 8. **Sai lầm phổ biến**

| Sai lầm                            | Nguyên nhân                    | Giải pháp                             |
| ---------------------------------- | ------------------------------ | ------------------------------------- |
| Không đặt `"size": 0`              | ES trả cả document → chậm      | Thêm `"size": 0` nếu chỉ cần thống kê |
| Dùng field `text` cho terms agg    | `text` bị analyzer → phân mảnh | Dùng `.keyword` field                 |
| Không dùng nested agg khi cần nhóm | Bỏ sót dữ liệu nhóm con        | Dùng `"aggs"` lồng nhau               |
| Quên `field` có kiểu numeric       | Range/avg agg lỗi              | Kiểm tra mapping trước                |
| Tính trung bình sai                | Không lọc dữ liệu trước        | Kết hợp `query` với agg               |

---

## ✅ **Best Practices**

* [ ] Dùng `.keyword` field cho grouping (`terms`).
* [ ] Luôn giới hạn số bucket (`size: 10` mặc định).
* [ ] Dùng nested aggregation để nhóm + thống kê.
* [ ] Kết hợp `query` để lọc trước khi tính toán.
* [ ] Trả về JSON gọn cho frontend: category → count → avg price.

---

## 🧩 **Bài tập thực hành**

1️⃣ Viết query tính **số lượng sản phẩm mỗi danh mục** và **giá trung bình**.
2️⃣ Viết query tạo **phân loại theo khoảng giá (range aggregation)**.
3️⃣ Viết query chỉ tính **sản phẩm giá dưới 2000**.
4️⃣ Chuyển query đó thành Java code với `ElasticsearchClient`.
5️⃣ Tạo endpoint `/api/products/analytics` trả về JSON thống kê.
