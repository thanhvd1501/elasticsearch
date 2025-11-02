### 📙 PHẦN 3: INDEX, MAPPING & ANALYZER

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Hiểu **index** và **mapping** khác nhau thế nào.
2. Biết cách **thiết kế mapping thủ công và tự động** trong Elasticsearch.
3. Hiểu cơ chế **analyzer, tokenizer, filter** và cách chúng ảnh hưởng đến kết quả tìm kiếm.
4. Tự tạo **custom analyzer** (ví dụ: tiếng Việt, tiếng Anh).
5. Thực hành định nghĩa index với analyzer trong Spring Boot và Kibana.

---

## 🧠 1. **Index là gì?**

**Index** trong Elasticsearch tương tự như **database** trong RDBMS.

* Mỗi index chứa nhiều **document** (giống như “bảng”).
* Mỗi document có cấu trúc JSON (giống “dòng dữ liệu”).
* Mỗi index được chia thành nhiều **shard** để lưu phân tán.

Ví dụ:

```
product_index
 ├── document_1.json
 ├── document_2.json
 └── document_3.json
```

---

## 🧩 2. **Mapping là gì?**

**Mapping** là “schema” của document – mô tả kiểu dữ liệu, analyzer, và cách lưu trữ từng field.

Ví dụ một `Product` document:

```json
{
  "name": "MacBook Pro M3",
  "category": "laptop",
  "price": 2499.99,
  "release_date": "2025-10-20"
}
```

Mapping định nghĩa:

```json
{
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "standard" },
      "category": { "type": "keyword" },
      "price": { "type": "float" },
      "release_date": { "type": "date" }
    }
  }
}
```

---

## ⚙️ 3. **Phân biệt `text` vs `keyword`**

| Field Type  | Dùng cho            | Đặc điểm                                                               |
| ----------- | ------------------- | ---------------------------------------------------------------------- |
| **text**    | full-text search    | Được analyzer tách từ (tokenize) → tìm kiếm không chính xác tuyệt đối. |
| **keyword** | exact match, filter | Không tách từ → dùng cho lọc, sort, aggregation.                       |

Ví dụ:

```
"name": "MacBook Pro M3"
→ "macbook", "pro", "m3"
(category: "laptop") → giữ nguyên "laptop"
```

---

## 🧩 4. **Analyzer, Tokenizer & Filter**

**Analyzer** là trái tim của full-text search.
Nó xử lý chuỗi văn bản khi **index** và **search**, gồm 3 bước:

```
Text → Tokenizer → Token Filters → Normalized Tokens
```

Ví dụ:

Input:

```
"Spring Boot for Beginners!"
```

**Standard analyzer**:

```
→ ["spring", "boot", "for", "beginners"]
```

**Keyword analyzer**:

```
→ ["Spring Boot for Beginners!"]
```

---

### 🔍 Các Analyzer thông dụng

| Analyzer       | Mô tả                                                    |
| -------------- | -------------------------------------------------------- |
| **standard**   | Tách theo khoảng trắng + dấu câu (default).              |
| **simple**     | Tách theo ký tự không phải chữ cái.                      |
| **whitespace** | Tách theo dấu cách.                                      |
| **keyword**    | Không tách từ, giữ nguyên chuỗi.                         |
| **stop**       | Giống standard nhưng loại bỏ stopwords (“the”, “is”, …). |
| **ngram**      | Dùng cho autocomplete.                                   |

---

## 🧩 5. **Thực hành: Tạo index với custom analyzer**

Ví dụ: muốn tìm kiếm tiếng Việt (có dấu), ta cần **custom analyzer** dùng plugin `analysis-icu` hoặc `analysis-vi`.

### 🧰 Cách tạo trong Kibana Dev Tools:

```bash
PUT /products_vi
{
  "settings": {
    "analysis": {
      "filter": {
        "vi_stop": {
          "type": "stop",
          "stopwords": "_vietnamese_"
        }
      },
      "analyzer": {
        "vi_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "vi_stop", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "vi_analyzer" },
      "category": { "type": "keyword" },
      "price": { "type": "float" }
    }
  }
}
```

🧩 Kiểm tra analyzer:

```bash
GET /products_vi/_analyze
{
  "analyzer": "vi_analyzer",
  "text": "Máy tính xách tay Lenovo"
}
```

Kết quả:

```
["may", "tinh", "xach", "tay", "lenovo"]
```

---

## 🧱 6. **Khai báo Mapping trong Spring Boot**

Bạn có thể định nghĩa mapping ngay trong entity bằng `@Field`.

**`Product.java`**

```java
@Document(indexName = "products_vi")
public class Product {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "vi_analyzer")
    private String name;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Float)
    private Double price;
}
```

> ⚠️ Lưu ý: custom analyzer phải được tạo sẵn trong Elasticsearch (Dev Tools) trước khi Spring Boot index dữ liệu.

---

## 🧩 7. **Dynamic vs Static Mapping**

| Loại mapping        | Đặc điểm                                          | Ưu điểm / Hạn chế                                                    |
| ------------------- | ------------------------------------------------- | -------------------------------------------------------------------- |
| **Dynamic mapping** | ES tự đoán kiểu dữ liệu                           | Nhanh khi prototype, nhưng dễ sai type (ví dụ `float` thành `long`). |
| **Static mapping**  | Bạn định nghĩa thủ công trong Dev Tools hoặc code | Kiểm soát tốt, tránh lỗi.                                            |

> ✅ **Best practice:** Luôn dùng static mapping cho production.

---

## 🧩 8. **Kiểm tra Mapping hiện tại**

```bash
GET /products_vi/_mapping
```

Kết quả (ví dụ):

```json
{
  "products_vi": {
    "mappings": {
      "properties": {
        "name": { "type": "text", "analyzer": "vi_analyzer" },
        "category": { "type": "keyword" },
        "price": { "type": "float" }
      }
    }
  }
}
```

---

## 🧩 9. **Minh hoạ quá trình phân tích dữ liệu**

```
Text: "Laptop Lenovo siêu mỏng"

Analyzer: vi_analyzer
↓
Tokenizer: "laptop", "lenovo", "sieu", "mong"
↓
Inverted Index:
 term      → docIDs
-------------------
 laptop    → [1, 3]
 lenovo    → [1, 2]
 sieu      → [1]
 mong      → [1]
```

---

## ⚠️ 10. **Sai lầm phổ biến**

| Sai lầm                                  | Giải thích                                                   |
| ---------------------------------------- | ------------------------------------------------------------ |
| Quên tạo analyzer trước khi index        | ES báo lỗi “unknown analyzer”.                               |
| Dùng dynamic mapping                     | ES tự suy đoán sai kiểu dữ liệu.                             |
| Nhầm giữa `text` và `keyword`            | Không filter được chính xác hoặc không search được toàn văn. |
| Không normalize dữ liệu Unicode          | Khi có dấu tiếng Việt dễ mismatch.                           |
| Không version index khi thay đổi mapping | Không thể sửa mapping → phải reindex.                        |

---

## ✅ **Checklist Best Practices**

* [ ] Tạo analyzer riêng cho từng ngôn ngữ.
* [ ] Dùng `keyword` cho field lọc, `text` cho field tìm kiếm.
* [ ] Khi đổi mapping, dùng alias (`products_v1` → `products_v2`).
* [ ] Dùng `GET _analyze` để test tokenizer.
* [ ] Luôn kiểm tra mapping trước khi index dữ liệu lớn.

---

## 🧩 **Bài tập thực hành**

1️⃣ Tạo index `products_vi` với analyzer tiếng Việt.
2️⃣ Index 3 document có dấu tiếng Việt.
3️⃣ Dùng `match` query kiểm tra khả năng tìm kiếm không dấu (“may tinh” vẫn match “máy tính”).
4️⃣ Xem kết quả `GET _analyze` để hiểu cách tách token.
