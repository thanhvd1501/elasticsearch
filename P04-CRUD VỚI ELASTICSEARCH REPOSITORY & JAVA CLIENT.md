### 📒 PHẦN 4: CRUD VỚI ELASTICSEARCH REPOSITORY & JAVA CLIENT

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ:

1. Biết cách thao tác dữ liệu trong Elasticsearch qua 2 cách:

   * ✅ `Spring Data Elasticsearch Repository` (cách đơn giản, quen thuộc với developer Spring Boot).
   * ✅ `Elasticsearch Java Client` (cách mạnh mẽ, linh hoạt cho query DSL phức tạp).
2. Hiểu cơ chế CRUD (Create, Read, Update, Delete) trên ES index.
3. Viết được truy vấn nâng cao có filter, highlight, sort, pagination.
4. Biết cách debug truy vấn và xem log gửi xuống cluster.

---

## 🧩 1. **Cách 1 – Dùng `ElasticsearchRepository` (Spring Data style)**

Spring Boot cung cấp `ElasticsearchRepository<T, ID>` tương tự `JpaRepository`, giúp thao tác nhanh mà không cần viết DSL thủ công.

---

### 🧱 1.1. Entity: `Product.java`

```java
package com.example.elasticsearch.model;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Document(indexName = "products")
public class Product {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Float)
    private Double price;

    @Field(type = FieldType.Text)
    private String description;
}
```

---

### 📦 1.2. Repository: `ProductRepository.java`

```java
package com.example.elasticsearch.repository;

import com.example.elasticsearch.model.Product;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // Dựa trên tên method, Spring Data tự build query
    List<Product> findByName(String name);
    List<Product> findByCategory(String category);
    List<Product> findByPriceBetween(Double min, Double max);
}
```

---

### ⚙️ 1.3. Service: `ProductService.java`

```java
package com.example.elasticsearch.service;

import com.example.elasticsearch.model.Product;
import com.example.elasticsearch.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public Product save(Product product) {
        return productRepository.save(product);
    }

    public Iterable<Product> findAll() {
        return productRepository.findAll();
    }

    public List<Product> findByName(String name) {
        return productRepository.findByName(name);
    }

    public List<Product> findByCategory(String category) {
        return productRepository.findByCategory(category);
    }

    public List<Product> findByPriceRange(double min, double max) {
        return productRepository.findByPriceBetween(min, max);
    }

    public void deleteById(String id) {
        productRepository.deleteById(id);
    }
}
```

---

### 🌐 1.4. Controller: `ProductController.java`

```java
package com.example.elasticsearch.controller;

import com.example.elasticsearch.model.Product;
import com.example.elasticsearch.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public Product save(@RequestBody Product product) {
        return productService.save(product);
    }

    @GetMapping
    public Iterable<Product> findAll() {
        return productService.findAll();
    }

    @GetMapping("/name/{name}")
    public List<Product> findByName(@PathVariable String name) {
        return productService.findByName(name);
    }

    @GetMapping("/category/{category}")
    public List<Product> findByCategory(@PathVariable String category) {
        return productService.findByCategory(category);
    }

    @GetMapping("/price")
    public List<Product> findByPriceRange(@RequestParam double min, @RequestParam double max) {
        return productService.findByPriceRange(min, max);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable String id) {
        productService.deleteById(id);
    }
}
```

---

### 🧪 1.5. Test bằng Postman

**Thêm document**

```bash
POST http://localhost:8080/api/products
Content-Type: application/json

{
  "name": "iPhone 16 Pro",
  "category": "smartphone",
  "price": 1399.99,
  "description": "Apple A18 chip, 256GB"
}
```

**Tìm theo khoảng giá**

```bash
GET http://localhost:8080/api/products/price?min=1000&max=2000
```

**Xoá sản phẩm**

```bash
DELETE http://localhost:8080/api/products/1
```

---

### 🔍 1.6. Debug query thực tế

Bật log trong `application.yml`:

```yaml
logging:
  level:
    org.springframework.data.elasticsearch.core: DEBUG
```

Khi gọi API, console sẽ hiển thị JSON query thực sự gửi tới Elasticsearch, ví dụ:

```json
{
  "query": {
    "match": {
      "name": "iphone"
    }
  }
}
```

---

## ⚡ 2. **Cách 2 – Dùng `Elasticsearch Java Client` (DSL linh hoạt)**

Từ Elasticsearch 8.x, client mới là `co.elastic.clients.elasticsearch.ElasticsearchClient`
→ cho phép viết query DSL chi tiết như trong Kibana.

---

### 🧰 2.1. Thêm dependency

```xml
<dependency>
    <groupId>co.elastic.clients</groupId>
    <artifactId>elasticsearch-java</artifactId>
    <version>8.12.0</version>
</dependency>
```

---

### 🔧 2.2. Tạo cấu hình client

**`ElasticSearchConfig.java`**

```java
package com.example.elasticsearch.config;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.ElasticsearchException;
import co.elastic.clients.json.jackson.JacksonJsonpMapper;
import co.elastic.clients.transport.rest_client.RestClientTransport;
import lombok.extern.slf4j.Slf4j;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@Slf4j
public class ElasticSearchConfig {

    @Bean
    public ElasticsearchClient elasticsearchClient() {
        RestClient restClient = RestClient.builder(
                new HttpHost("localhost", 9200)
        ).build();

        RestClientTransport transport = new RestClientTransport(
                restClient,
                new JacksonJsonpMapper()
        );

        return new ElasticsearchClient(transport);
    }
}
```

---

### 💾 2.3. CRUD cơ bản với Java Client

**`ProductClientService.java`**

```java
package com.example.elasticsearch.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch.core.*;
import com.example.elasticsearch.model.Product;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductClientService {

    private final ElasticsearchClient client;

    public void indexProduct(Product product) throws IOException {
        client.index(i -> i
                .index("products")
                .id(product.getId())
                .document(product)
        );
    }

    public Product getProduct(String id) throws IOException {
        GetResponse<Product> response = client.get(g -> g
                .index("products")
                .id(id), Product.class
        );
        return response.source();
    }

    public List<Product> searchByName(String name) throws IOException {
        SearchResponse<Product> response = client.search(s -> s
                .index("products")
                .query(q -> q
                        .match(m -> m
                                .field("name")
                                .query(name)
                        )
                ), Product.class
        );
        return response.hits().hits().stream()
                .map(hit -> hit.source())
                .toList();
    }

    public void deleteById(String id) throws IOException {
        client.delete(d -> d
                .index("products")
                .id(id)
        );
    }
}
```

---

### 🧠 2.4. So sánh hai cách

| Tiêu chí                                | `ElasticsearchRepository` | `ElasticsearchClient`  |
| --------------------------------------- | ------------------------- | ---------------------- |
| Dễ dùng                                 | ✅ Dễ, giống JPA           | ❌ Hơi verbose          |
| Kiểm soát query                         | ❌ Hạn chế                 | ✅ Toàn quyền DSL       |
| Hiệu năng                               | Trung bình                | Tốt (ít overhead hơn)  |
| Query phức tạp (bool, fuzzy, highlight) | Khó viết                  | Dễ, chi tiết           |
| Reindex/Bulk                            | Hạn chế                   | ✅ Hỗ trợ đầy đủ        |
| Thích hợp cho                           | CRUD, prototype           | Production, logic nặng |

---

### ⚠️ 2.5. Lỗi thường gặp

| Lỗi                                                      | Nguyên nhân                  | Giải pháp                                               |
| -------------------------------------------------------- | ---------------------------- | ------------------------------------------------------- |
| `ElasticsearchStatusException[mapper_parsing_exception]` | Mapping không khớp           | Xoá index, tạo lại.                                     |
| `no such index`                                          | Index chưa được tạo          | Tạo trước bằng Kibana hoặc `client.indices().create()`. |
| `Connection refused`                                     | Elasticsearch chưa chạy      | Kiểm tra Docker container.                              |
| `Serialization error`                                    | Object chưa có getter/setter | Dùng Lombok `@Data` hoặc `@Getter/@Setter`.             |

---

### ✅ 2.6. Best Practices

* Dùng `Repository` cho CRUD nhanh.
* Dùng `ElasticsearchClient` cho **truy vấn phức tạp** (bool, fuzzy, aggregation, highlight).
* Luôn log query JSON để dễ debug.
* Gắn annotation `@Document(indexName = "xxx")` để tránh sai index.
* Khi thay đổi mapping → reindex hoặc dùng alias để tránh downtime.

---

### 🧩 **Bài tập thực hành**

1️⃣ Thêm 3 sản phẩm bằng API `/api/products`.
2️⃣ Gọi service `ProductClientService.searchByName("iphone")` để test Java Client.
3️⃣ So sánh JSON query giữa Repository và Java Client.
4️⃣ Bật Kibana Dev Tools và chạy `GET products/_search` xem dữ liệu.
