# 🛠️ CDS Services — Lớp Trung Gian Giữa DB và Client

> **Service** trong CAP = API layer. Bạn định nghĩa service để quyết định **expose entity nào**, **ai được làm gì**, và **endpoint nào được tạo ra**. Mọi OData endpoint đều đến từ service.

---

## 🤔 Service là gì?

```
Flow request trong CAP:
───────────────────────────────────────────────────────────
Client (browser/app/curl)
    ↓  HTTP Request
Service Layer          ← service.cds + service.js (file này)
    ↓  CQL Query        
DB Layer               ← schema.cds (entities, tables)
    ↓  SQL
SQLite / HANA Cloud    ← database thật
```

Service là "contract" giữa client và database:
- Quyết định **endpoint nào** được tạo
- Quyết định **field nào** được expose
- Quyết định **ai có quyền** làm gì
- Nơi để thêm **custom business logic**

---

## 📝 Cú pháp cơ bản

```cds
// srv/catalog-service.cds
using my.bookshop as db from '../db/schema';

service CatalogService {              // tên service → URL: /catalog
  entity Books   as projection on db.Books;
  entity Authors as projection on db.Authors;
}
```

**Điều này tạo ra:**
```
GET    /catalog/Books         → list tất cả books
GET    /catalog/Books(1)      → lấy book ID=1
POST   /catalog/Books         → tạo book mới
PUT    /catalog/Books(1)      → replace book
PATCH  /catalog/Books(1)      → update partial
DELETE /catalog/Books(1)      → xóa book
GET    /catalog/$metadata     → OData metadata
```

---

## 🎯 Service URL Path

Mặc định, URL path = tên service (viết thường):

```cds
service CatalogService { ... }  → /catalog
service AdminService { ... }    → /admin
service OrdersService { ... }   → /orders

// Hoặc chỉ định path thủ công:
service CatalogService @(path: '/api/v1/catalog') { ... }
→ /api/v1/catalog
```

---

## 🔍 Projections — Filter dữ liệu expose

Projection quyết định **fields và rows nào** được expose qua service.

### Expose tất cả (full projection)

```cds
service AdminService {
  entity Books as projection on db.Books;
  // → expose toàn bộ fields của db.Books
}
```

### Chỉ expose một số fields

```cds
service CatalogService {
  // Chỉ expose 4 fields, ẩn các fields nhạy cảm (ví dụ cost, discount)
  entity Books as select from db.Books {
    ID, title, stock, price
  };
}
```

### Lọc theo điều kiện (WHERE clause)

```cds
service CatalogService {
  // Chỉ hiện sách còn hàng
  entity Books as select from db.Books {
    ID, title, stock, price
  } where stock > 0;
}
```

### Thêm field tính toán (virtual columns)

```cds
service CatalogService {
  entity Books as select from db.Books {
    ID,
    title,
    stock,
    price,
    // Field tính toán — không có trong DB
    stock * price as totalValue : Decimal
  };
}
```

### Rename field trong service

```cds
service CatalogService {
  entity Books as select from db.Books {
    ID,
    title    as name,        // đổi tên title → name trong API
    stock    as available
  };
}
```

---

## 🔒 Access Control Annotations

### `@readonly` — Chỉ đọc

```cds
service CatalogService {
  @readonly
  entity Books as projection on db.Books;
  // → Chỉ GET được, không POST/PATCH/DELETE
}
```

### `@insertonly` — Chỉ nhận insert mới

```cds
service LogService {
  @insertonly
  entity AuditLogs as projection on db.AuditLogs;
  // → Chỉ POST được, không GET/PATCH/DELETE
}
```

### `@restrict` — Phân quyền chi tiết theo role

```cds
service OrderService @(requires: 'authenticated-user') {
  //  Ai cũng READ được, chỉ Admin mới WRITE/DELETE
  @(restrict: [
    { grant: 'READ' },
    { grant: ['WRITE', 'DELETE'], to: 'Admin' }
  ])
  entity Orders as projection on db.Orders;

  // Chỉ user có role Viewer mới xem được
  @(requires: 'Viewer')
  entity Books as projection on db.Books;
}
```

### Bảng tóm tắt annotations

| Annotation | Tương đương | HTTP methods cho phép |
|---|---|---|
| *(không có gì)* | full access | GET, POST, PATCH, PUT, DELETE |
| `@readonly` | `@(restrict: [{grant:'READ'}])` | GET chỉ |
| `@insertonly` | `@(restrict: [{grant:'WRITE'}])` | POST chỉ |
| `@(requires: 'Role')` | role check | Tất cả, nhưng phải có Role |
| `@(restrict: [...])` | fine-grained | Tùy cấu hình |

---

## ⚡ Actions và Functions

Khi CRUD thông thường không đủ, dùng Action/Function để tạo custom endpoint.

### Action — thay đổi state (POST)

```cds
service CatalogService {
  entity Books as projection on db.Books;

  // Action: POST /catalog/submitOrder
  // params: book ID, quantity
  // returns: message string
  action submitOrder(
    book     : Books:ID,      // loại kiểu dùng từ entity
    quantity : Integer
  ) returns String;

  // Action trên entity (bound action)
  // POST /catalog/Books(1)/CatalogService.addReview
  action addReview(
    rating  : Integer,
    comment : String
  ) returns Books;
}
```

### Function — chỉ đọc (GET)

```cds
service CatalogService {
  entity Books as projection on db.Books;

  // Function: GET /catalog/getTopBooks(maxCount=5)
  function getTopBooks(maxCount: Integer) returns array of Books;

  // Function không params
  function getTodaysBestsellers() returns array of Books;
}
```

### Implement trong .js handler

```javascript
// srv/catalog-service.js
module.exports = (srv) => {
  const { Books } = srv.entities;

  // Implement Action
  srv.on('submitOrder', async (req) => {
    const { book, quantity } = req.data;
    
    // Đọc stock hiện tại
    const [b] = await SELECT.from(Books).where({ ID: book });
    if (!b) return req.error(404, `Book ${book} not found`);
    if (b.stock < quantity) return req.error(409, 'Insufficient stock');
    
    // Giảm stock
    await UPDATE(Books, book).with({ stock: b.stock - quantity });
    return `Order placed: ${quantity} copies of "${b.title}"`;
  });

  // Implement Function
  srv.on('getTopBooks', async (req) => {
    const { maxCount } = req.data;
    return SELECT.from(Books)
      .orderBy('stock desc')
      .limit(maxCount || 10);
  });
};
```

---

## 🏗️ Tách thành nhiều Services

Một app thường có nhiều services phục vụ các mục đích khác nhau:

```cds
// srv/catalog-service.cds — public API
service CatalogService {
  @readonly entity Books as projection on db.Books;
  entity Orders as projection on db.Orders;
}

// srv/admin-service.cds — dành cho admin
service AdminService @(requires: 'Admin') {
  entity Books   as projection on db.Books;     // full CRUD
  entity Authors as projection on db.Authors;
  entity Orders  as projection on db.Orders;
}

// srv/stats-service.cds — reporting
service StatsService @(requires: 'authenticated-user') {
  @readonly entity OrderSummary as select from db.Orders {
    count(*) as totalOrders : Integer,
    sum(amount) as totalRevenue : Decimal
  };
}
```

---

## 📌 Service định nghĩa 2 files

```
srv/
├── catalog-service.cds     ← khai báo (contract)
└── catalog-service.js      ← implement logic (optional, nếu cần custom)
```

```javascript
// srv/catalog-service.js — chỉ cần nếu có custom logic
module.exports = (srv) => {
  // Không có code ở đây = CAP tự handle CRUD hoàn toàn!
  
  // Có code ở đây = override hành vi mặc định
  srv.before('CREATE', 'Books', (req) => {
    if (!req.data.title) req.error(400, 'Title is required');
  });
};
```

> ✅ **Không cần file .js** nếu chỉ cần CRUD cơ bản — CAP tự gen toàn bộ!

---

## 🧪 Test Service sau khi định nghĩa

```bash
# Start server
cds watch

# Test REST endpoints (mở browser hoặc dùng curl)
curl http://localhost:4004/catalog/Books
curl http://localhost:4004/catalog/Books/1
curl http://localhost:4004/catalog/Books?$filter=stock gt 0

# Xem service metadata
curl http://localhost:4004/catalog/$metadata

# Test action
curl -X POST http://localhost:4004/catalog/submitOrder \
  -H "Content-Type: application/json" \
  -d '{"book": "uuid-here", "quantity": 2}'
```

---

## ⚠️ Lỗi hay gặp

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| 405 Method Not Allowed | `@readonly` nhưng gửi POST | Remove `@readonly` hoặc dùng service khác |
| 403 Forbidden | Thiếu role | Confirm mock user có role phù hợp |
| Entity not found | Projection sai tên | Kiểm tra `using ... from ...` import |
| Action not recognized | Tên action sai | Kiểm tra tên trong .cds khớp với `.on('name', ...)` |

---

## ✅ Checklist thực hành

- [ ] Tạo `CatalogService` với `@readonly` entity Books
- [ ] Tạo `AdminService` với full CRUD cho Books
- [ ] Dùng `as select from ... { fields } where ...` để filter projection
- [ ] Định nghĩa 1 Action (`submitOrder`) và 1 Function (`getTopBooks`)
- [ ] Implement handlers trong file `.js`
- [ ] Test tất cả endpoints bằng curl hoặc REST Client trong VS Code

---

## 🔗 Tài liệu tham khảo

- [CAP — Providing Services](https://cap.cloud.sap/docs/guides/providing-services)
- [CDS — Service Definitions](https://cap.cloud.sap/docs/cds/cdl#services)
- [CDS — Actions & Functions](https://cap.cloud.sap/docs/cds/cdl#actions)
- [Authorization & Access Control](https://cap.cloud.sap/docs/guides/authorization)

---

> **Tóm lại:**  
> - **Service** = API contract, quyết định expose gì cho client  
> - **Projection** = filter fields/rows từ entity gốc  
> - `@readonly` / `@insertonly` / `@restrict` = phân quyền đơn giản  
> - **Action** (POST) thay đổi state, **Function** (GET) chỉ đọc  
> - File `.js` chỉ cần khi muốn custom logic, còn không CAP tự gen CRUD
