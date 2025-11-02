### 📕 PHẦN 6: SEARCH NÂNG CAO – HIGHLIGHT, PAGINATION, SORTING

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Biết cách **hiển thị highlight (bôi vàng từ khóa tìm kiếm)** trong kết quả.
2. Thêm **pagination** (phân trang) và **sorting** (sắp xếp) vào truy vấn tìm kiếm.
3. Kết hợp nhiều điều kiện filter + highlight + sort trong cùng một query.
4. Hiểu cách **tối ưu search API** để đạt tốc độ cao, độ chính xác cao, và kết quả hấp dẫn hơn.

---

## 🧩 1. **Highlight – Làm nổi bật từ khóa trong kết quả**

Khi người dùng tìm kiếm `"macbook"`, ta muốn phần mô tả hoặc tên sản phẩm hiển thị như sau:

> 💡 “Mua ngay **<span style='color:orange'>MacBook</span> Pro M3** chính hãng”

---

### 🔹 1.1. Ví dụ với Kibana Dev Tools

```bash
GET products/_search
{
  "query": {
    "match": { "description": "macbook" }
  },
  "highlight": {
    "fields": {
      "description": {}
    },
    "pre_tags": ["<em style='color:orange'>"],
    "post_tags": ["</em>"]
  }
}
```

**Kết quả trả về (rút gọn):**

```json
{
  "hits": {
    "hits": [
      {
        "_source": {
          "name": "MacBook Pro M3",
          "description": "Laptop Apple MacBook Pro M3 2025"
        },
        "highlight": {
          "description": ["Laptop Apple <em style='color:orange'>MacBook</em> Pro M3 2025"]
        }
      }
    ]
  }
}
```

---

### 🔹 1.2. Highlight trong Java Client

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q.match(m -> m.field("description").query("macbook")))
    .highlight(h -> h
        .fields("description", f -> f)
        .preTags("<em style='color:orange'>")
        .postTags("</em>")
    ),
    Product.class
);

response.hits().hits().forEach(hit -> {
    System.out.println("Name: " + hit.source().getName());
    System.out.println("Highlight: " + hit.highlight().get("description"));
});
```

---

### 💡 **Best practice:**

* Highlight nên dùng trên **text field** (có analyzer).
* Không nên highlight field quá dài (performance cost).
* Chỉ nên hiển thị highlight trên UI, không lưu vào DB.

---

## 📊 2. **Pagination – Phân trang kết quả tìm kiếm**

Elasticsearch hỗ trợ `from` và `size` giống SQL `OFFSET` và `LIMIT`.

```bash
GET products/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,
  "size": 10
}
```

→ Trả về 10 document đầu tiên (page 1).

Trang kế tiếp:

```bash
"from": 10,
"size": 10
```

---

### 🔹 Trong Java Client:

```java
int page = 1;
int size = 10;
int from = (page - 1) * size;

SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .from(from)
    .size(size)
    .query(q -> q.matchAll(m -> m)),
    Product.class
);
```

> ⚠️ **Lưu ý**:
> Pagination lớn (từ trang 1000+) làm giảm hiệu năng.
> Khi cần phân trang sâu → dùng **search_after** hoặc **scroll search** (sẽ học ở Phần 8).

---

## 🔁 3. **Sorting – Sắp xếp kết quả**

Ta có thể sắp xếp theo:

* `price` tăng/giảm
* `release_date`
* `score` (mức độ liên quan – mặc định)

---

### 🔹 Ví dụ (Kibana Dev Tools):

```bash
GET products/_search
{
  "query": {
    "match": { "category": "laptop" }
  },
  "sort": [
    { "price": "asc" },
    { "_score": "desc" }
  ]
}
```

> Kết quả: laptop rẻ nhất hiển thị trước, cùng lúc vẫn sắp xếp theo độ liên quan.

---

### 🔹 Trong Java Client:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q.match(m -> m.field("category").query("laptop")))
    .sort(so -> so.field(f -> f.field("price").order(SortOrder.Asc)))
    .sort(so -> so.field(f -> f.field("_score").order(SortOrder.Desc)))
    .size(10),
    Product.class
);
```

---

## 🔍 4. **Kết hợp Highlight + Filter + Sort + Pagination**

### Ví dụ truy vấn hoàn chỉnh:

```bash
GET products/_search
{
  "query": {
    "bool": {
      "must": [
        { "multi_match": { "query": "macbook pro", "fields": ["name", "description"] } }
      ],
      "filter": [
        { "term": { "category": "laptop" } },
        { "range": { "price": { "lte": 2500 } } }
      ]
    }
  },
  "sort": [
    { "price": "asc" }
  ],
  "from": 0,
  "size": 5,
  "highlight": {
    "fields": {
      "name": {},
      "description": {}
    },
    "pre_tags": ["<b style='color:#ff6600'>"],
    "post_tags": ["</b>"]
  }
}
```

---

### Java Client tương đương:

```java
SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .from(0)
    .size(5)
    .query(q -> q.bool(b -> b
        .must(m -> m.multiMatch(mm -> mm
            .query("macbook pro")
            .fields("name", "description")
        ))
        .filter(
            f -> f.term(t -> t.field("category").value("laptop")),
            f -> f.range(r -> r.field("price").lte(JsonData.of(2500)))
        )
    ))
    .sort(so -> so.field(f -> f.field("price").order(SortOrder.Asc)))
    .highlight(h -> h
        .fields("name", f -> f)
        .fields("description", f -> f)
        .preTags("<b style='color:#ff6600'>")
        .postTags("</b>")
    ),
    Product.class
);

for (Hit<Product> hit : response.hits().hits()) {
    System.out.println(hit.source().getName());
    System.out.println("Highlight: " + hit.highlight());
}
```

---

## 🧭 5. **Tối ưu search UI (Spring Boot + Frontend)**

Khi frontend gọi API, backend có thể trả về JSON như sau:

```json
{
  "total": 128,
  "page": 1,
  "size": 10,
  "results": [
    {
      "id": "123",
      "name": "MacBook Pro M3",
      "price": 2499,
      "highlight": "Apple <b style='color:#ff6600'>MacBook</b> Pro M3"
    }
  ]
}
```

Khi render ở frontend (React/Vue/Angular), phần highlight sẽ tự hiển thị như text HTML.

---

## ⚠️ 6. **Sai lầm phổ biến**

| Sai lầm                   | Nguyên nhân                                    | Giải pháp                               |
| ------------------------- | ---------------------------------------------- | --------------------------------------- |
| Highlight không hoạt động | Field không phải `text` hoặc không có analyzer | Dùng `text` + analyzer hợp lý           |
| Kết quả trùng lặp         | Không filter đúng                              | Dùng `term` hoặc `bool.filter`          |
| Sort sai                  | Field không có type number/date                | Kiểm tra mapping trước                  |
| Phân trang chậm           | Dữ liệu lớn                                    | Dùng `search_after` thay vì `from/size` |
| Không hiển thị highlight  | Không parse `hit.highlight()` trong code       | Lấy thủ công trong response             |

---

## ✅ **Best Practices Checklist**

* [ ] Luôn giới hạn `size` (10–50) để tránh tải nặng.
* [ ] Sử dụng `filter` cho điều kiện cố định (category, brand).
* [ ] Dùng `highlight` để tăng trải nghiệm người dùng (UX).
* [ ] Sort theo `_score` + field phụ (như price/date).
* [ ] Cache truy vấn phổ biến (Redis hoặc ES query cache).

---

## 🧩 **Bài tập thực hành**

1️⃣ Viết API `/search` có các tham số:
`q` (từ khóa), `category`, `minPrice`, `maxPrice`, `page`, `size`, `sort`.

2️⃣ Query ES trả về danh sách sản phẩm có highlight từ khóa trong `name` và `description`.

3️⃣ Thử sort theo `price asc` hoặc `score desc`.

4️⃣ Dùng Kibana Dev Tools để test cùng query → so sánh JSON trả về.
