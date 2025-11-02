### 📗 PHẦN 2: CÀI ĐẶT & KẾT NỐI ELASTICSEARCH TRONG SPRING BOOT

---

#### 🎯 **Mục tiêu học tập**

Sau phần này, bạn sẽ biết cách:

1. Cài đặt Elasticsearch 8.x (local hoặc Docker).
2. Cấu hình `spring-data-elasticsearch` trong dự án Spring Boot 3.x.
3. Tạo entity – repository – service cơ bản.
4. Kiểm tra kết nối giữa ứng dụng và cluster Elasticsearch.
5. Debug và xử lý lỗi kết nối thường gặp.

---

## 🧱 1. **Cài đặt Elasticsearch + Kibana bằng Docker Compose**

Tạo file `docker-compose.yml` tại thư mục gốc dự án:

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
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"

volumes:
  es_data:
```

Chạy lệnh:

```bash
docker-compose up -d
```

🧩 Kiểm tra:

* Truy cập [http://localhost:9200](http://localhost:9200) → thấy JSON version 8.x.
* Truy cập [http://localhost:5601](http://localhost:5601) → Kibana UI.

---

## ⚙️ 2. **Tạo dự án Spring Boot kết nối Elasticsearch**

Bạn có thể tạo bằng **Spring Initializr** ([https://start.spring.io](https://start.spring.io)):

**Dependencies:**

* Spring Web
* Spring Data Elasticsearch
* Lombok
* Spring Boot DevTools
* (Tuỳ chọn) PostgreSQL/MySQL driver nếu có database chính

---

### 📁 Cấu trúc dự án ví dụ

```
springboot-elasticsearch-demo/
├── src/main/java/com/example/elasticsearch/
│   ├── controller/
│   │   └── ProductController.java
│   ├── service/
│   │   └── ProductService.java
│   ├── repository/
│   │   └── ProductRepository.java
│   ├── model/
│   │   └── Product.java
│   └── ElasticsearchApplication.java
└── src/main/resources/
    └── application.yml
```

---

## 🧩 3. **Cấu hình `application.yml`**

```yaml
spring:
  application:
    name: elasticsearch-demo

  data:
    elasticsearch:
      cluster-name: docker-cluster
      repositories:
        enabled: true
      client:
        rest:
          uris: http://localhost:9200

logging:
  level:
    org.springframework.data.elasticsearch: DEBUG
```

> ⚠️ Lưu ý:
>
> * Nếu bạn dùng **Elastic Cloud** hoặc có security bật, cần thêm username/password + SSL:
>
>   ```yaml
>   spring.data.elasticsearch.client.rest.username: elastic
>   spring.data.elasticsearch.client.rest.password: changeme
>   spring.data.elasticsearch.client.rest.use-ssl: true
>   ```

---

## 🧩 4. **Tạo Entity – Mapping Index**

**`Product.java`**

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

🔹 **Giải thích:**

| Annotation                   | Mô tả                                                                         |
| ---------------------------- | ----------------------------------------------------------------------------- |
| `@Document(indexName="...")` | Chỉ định index trong ES.                                                      |
| `@Id`                        | Document ID.                                                                  |
| `@Field`                     | Định nghĩa mapping, analyzer, type (`Text`, `Keyword`, `Date`, `Float`, ...). |

---

## 🧩 5. **Tạo Repository**

**`ProductRepository.java`**

```java
package com.example.elasticsearch.repository;

import com.example.elasticsearch.model.Product;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface ProductRepository extends ElasticsearchRepository<Product, String> {
    List<Product> findByName(String name);
    List<Product> findByCategory(String category);
}
```

> `ElasticsearchRepository` hoạt động tương tự `JpaRepository`, hỗ trợ CRUD cơ bản và query tự động dựa vào tên method.

---

## 🧩 6. **Service Layer**

**`ProductService.java`**

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

    public void deleteById(String id) {
        productRepository.deleteById(id);
    }
}
```

---

## 🧩 7. **Controller (REST API)**

**`ProductController.java`**

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

    @DeleteMapping("/{id}")
    public void delete(@PathVariable String id) {
        productService.deleteById(id);
    }
}
```

---

## 🧪 8. **Kiểm tra bằng Postman hoặc curl**

**Thêm document:**

```bash
POST http://localhost:8080/api/products
Content-Type: application/json

{
  "name": "MacBook Pro M3",
  "category": "laptop",
  "price": 2499.99,
  "description": "Apple M3 chip, 16GB RAM, 1TB SSD"
}
```

**Lấy tất cả sản phẩm:**

```bash
GET http://localhost:8080/api/products
```

**Tìm theo tên:**

```bash
GET http://localhost:8080/api/products/name/MacBook
```

---

## 🔍 9. **Kiểm tra trong Kibana Dev Tools**

```bash
GET products/_search
{
  "query": {
    "match_all": {}
  }
}
```

✅ Kết quả trả về document vừa index.

---

## ⚠️ 10. **Lỗi kết nối thường gặp & cách khắc phục**

| Lỗi                                                | Nguyên nhân                  | Cách xử lý                                                |
| -------------------------------------------------- | ---------------------------- | --------------------------------------------------------- |
| `Connection refused: localhost/127.0.0.1:9200`     | Elasticsearch chưa khởi động | Chạy `docker ps`, đảm bảo container đang chạy.            |
| `NoNodeAvailableException`                         | Cấu hình sai URI             | Kiểm tra `spring.data.elasticsearch.client.rest.uris`.    |
| `ElasticsearchStatusException[security_exception]` | ES có bật bảo mật            | Tắt `xpack.security.enabled` hoặc thêm username/password. |
| `MappingException`                                 | Field type không khớp        | Xoá index, tạo lại với mapping đúng.                      |
| `UnknownHostException`                             | Sai host/port                | Sửa cấu hình `application.yml`.                           |

---

## ✅ **Checklist Best Practices**

* [ ] Dùng `spring-data-elasticsearch` để CRUD nhanh.
* [ ] Kiểm tra version Spring Boot ↔ Elasticsearch tương thích.
* [ ] Dùng Kibana Dev Tools để test nhanh các query.
* [ ] Bật log `org.springframework.data.elasticsearch: DEBUG` để debug query.
* [ ] Giữ Docker cluster “single-node” khi dev, cluster 3 node khi production.

---

## 🧩 **Bài tập thực hành**

1️⃣ Dùng Postman tạo 5 sản phẩm khác nhau.
2️⃣ Dùng Kibana kiểm tra dữ liệu index.
3️⃣ Thử xoá index và index lại để hiểu vòng đời document.
