### 📔 PHẦN 5: QUERY DSL – MATCH, BOOL, FUZZY, RANGE

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Hiểu và sử dụng được **Elasticsearch Query DSL** (Domain-Specific Language).
2. Biết cách viết các truy vấn phổ biến: `match`, `multi_match`, `bool`, `range`, `fuzzy`, `wildcard`.
3. Áp dụng được cả trong **Kibana Dev Tools** và **Java (Spring Boot)** qua `ElasticsearchClient`.
4. Biết cách debug, tối ưu và kết hợp nhiều điều kiện tìm kiếm (logic AND, OR, NOT).

---

## 🧠 1. **Tổng quan về Query DSL**

Elasticsearch không dùng SQL mà có ngôn ngữ riêng gọi là **Query DSL** (viết bằng JSON).
Cấu trúc cơ bản của một truy vấn:

```json
GET /index_name/_search
{
  "query": {
    "match": { "field_name": "search_text" }
  }
}
```

Mọi truy vấn đều nằm trong khóa `"query"`.
Có 2 nhóm chính:

| Nhóm                   | Mục đích                                                          |
| ---------------------- | ----------------------------------------------------------------- |
| **Full-text queries**  | Tìm kiếm toàn văn (match, multi_match, fuzzy, query_string, etc.) |
| **Term-level queries** | So khớp chính xác (term, range, wildcard, prefix, etc.)           |

---

## 📘 2. **MATCH QUERY (phổ biến nhất)**

Dùng khi bạn cần **full-text search** với analyzer (được tách từ, lowercase, v.v.).

### 🔹 Ví dụ (Kibana Dev Tools)

```bash
GET products/_search
{
  "query": {
    "match": {
      "name": "iphone 16 pro"
    }
  }
}
```

Kết quả trả về document có chứa “iphone”, “16”, hoặc “pro”.
Elasticsearch sẽ tự động **scoring** theo mức độ liên quan (BM25).

### 🔹 Trong Java Client:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .match(m -> m
            .field("name")
            .query("iphone 16 pro")
        )
    ),
    Product.class
);
```

---

## 📙 3. **MULTI_MATCH QUERY**

Tìm kiếm trên **nhiều trường** cùng lúc (ví dụ: `name`, `description`).

```bash
GET products/_search
{
  "query": {
    "multi_match": {
      "query": "macbook pro m3",
      "fields": ["name", "description"]
    }
  }
}
```

Trong Java Client:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .multiMatch(mm -> mm
            .fields("name", "description")
            .query("macbook pro m3")
        )
    ),
    Product.class
);
```

> ✅ Tip: Dùng `multi_match` để tăng độ phủ của tìm kiếm sản phẩm/bài viết.

---

## 📗 4. **BOOL QUERY (kết hợp điều kiện logic)**

Cho phép kết hợp nhiều query con:

* `must` = AND
* `should` = OR
* `must_not` = NOT
* `filter` = điều kiện không ảnh hưởng đến score

### 🔹 Ví dụ:

```bash
GET products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "category": "laptop" } }
      ],
      "filter": [
        { "range": { "price": { "lte": 2000 } } }
      ],
      "must_not": [
        { "match": { "name": "old" } }
      ]
    }
  }
}
```

**Giải thích:**

* Lấy sản phẩm category = "laptop"
* Giá ≤ 2000
* Không chứa từ “old”

---

### 🔹 Java Client tương đương:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .bool(b -> b
            .must(m -> m.match(ma -> ma.field("category").query("laptop")))
            .filter(f -> f.range(r -> r.field("price").lte(JsonData.of(2000))))
            .mustNot(n -> n.match(ma -> ma.field("name").query("old")))
        )
    ),
    Product.class
);
```

---

## 📕 5. **RANGE QUERY (khoảng giá trị số hoặc ngày)**

Dùng để tìm giá, ngày, độ tuổi... trong một khoảng.

```bash
GET products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 1000,
        "lte": 2500
      }
    }
  }
}
```

### Java Client:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .range(r -> r
            .field("price")
            .gte(JsonData.of(1000))
            .lte(JsonData.of(2500))
        )
    ),
    Product.class
);
```

---

## 📓 6. **FUZZY QUERY (tìm kiếm sai chính tả)**

Cho phép “xấp xỉ” ký tự (edit distance ≤ 2).

```bash
GET products/_search
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "iphnoe",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

→ Tự động khớp với “iphone”.

**Java Client:**

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .fuzzy(f -> f
            .field("name")
            .value("iphnoe")
            .fuzziness("AUTO")
        )
    ),
    Product.class
);
```

> ⚙️ Mẹo: `fuzziness: "AUTO"` giúp ES tự chọn khoảng cách phù hợp (0–2).

---

## 📒 7. **WILDCARD QUERY (so khớp mẫu ký tự)**

Dùng `*` (nhiều ký tự) hoặc `?` (một ký tự).
Không được phân tích (no analyzer).

```bash
GET products/_search
{
  "query": {
    "wildcard": {
      "name": {
        "value": "mac*"
      }
    }
  }
}
```

=> Khớp “macbook”, “mac mini”, “mac pro”...

> ⚠️ Chậm với dữ liệu lớn – tránh dùng wildcard ở đầu chuỗi: `*book`.

---

## 📘 8. **Kết hợp Match + Filter (Search nâng cao)**

```bash
GET products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "macbook" } }
      ],
      "filter": [
        { "term": { "category": "laptop" } },
        { "range": { "price": { "lte": 2000 } } }
      ]
    }
  }
}
```

Trong Java Client:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q.bool(b -> b
        .must(m -> m.match(ma -> ma.field("name").query("macbook")))
        .filter(
            f -> f.term(t -> t.field("category").value("laptop")),
            f -> f.range(r -> r.field("price").lte(JsonData.of(2000)))
        )
    )),
    Product.class
);
```

---

## 📊 9. **Sơ đồ logic các loại query**

```
Full-text queries      →  match / multi_match / query_string
Term-level queries     →  term / range / wildcard / prefix
Compound queries       →  bool / dis_max / function_score
Joining queries        →  nested / has_child / has_parent
```

---

## ⚠️ 10. **Sai lầm phổ biến**

| Sai lầm                                      | Giải thích                                                     |
| -------------------------------------------- | -------------------------------------------------------------- |
| Dùng `term` thay vì `match` cho text field   | `term` không qua analyzer, nên không khớp dữ liệu “tokenized”. |
| Lạm dụng `wildcard`                          | Rất chậm vì quét toàn bộ inverted index.                       |
| Không dùng `filter` cho điều kiện cố định    | Filter nhanh hơn vì không tính score.                          |
| Dùng `should` mà quên `minimum_should_match` | Kết quả rỗng hoặc quá nhiều.                                   |
| Không kiểm tra `mapping` trước khi query     | Dẫn tới mismatch type.                                         |

---

## ✅ **Best Practices**

* [ ] Với text → dùng `match` hoặc `multi_match`.
* [ ] Với keyword → dùng `term`.
* [ ] Khi lọc theo giá hoặc ngày → dùng `range`.
* [ ] Khi cần kết hợp nhiều điều kiện → dùng `bool` + `filter`.
* [ ] Khi có lỗi chính tả → dùng `fuzzy`.
* [ ] Test query trước trong Kibana Dev Tools, sau đó chuyển sang Java Client.

---

## 🧩 **Bài tập thực hành**

1️⃣ Viết truy vấn `bool` tìm tất cả laptop giá dưới 2000, không chứa từ “old”.
2️⃣ Viết `multi_match` tìm sản phẩm theo `name` và `description`.
3️⃣ Viết `fuzzy` để “iphone” vẫn tìm được khi người dùng nhập sai “iphnoe”.
4️⃣ So sánh thời gian query giữa `match` và `term` với cùng dữ liệu.
