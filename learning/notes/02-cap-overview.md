# 🌱 SAP CAP — Tổng Quan Framework

> SAP CAP (Cloud Application Programming Model) là **framework backend** để build enterprise apps trên SAP BTP. Triết lý cốt lõi: **"Convention over Configuration"** — viết ít code, CAP tự làm phần còn lại.

---

## 🤔 CAP giải quyết vấn đề gì?

Khi build enterprise app, developer thường phải lặp đi lặp lại:

```
❌ Không dùng CAP — tự làm hết:
1. Viết database schema (SQL)
2. Viết ORM models
3. Viết REST/GraphQL endpoints (GET, POST, PUT, DELETE)
4. Viết validation logic
5. Viết auth/authorization
6. Viết query parsing
... và nhiều boilerplate khác

✅ Dùng CAP — chỉ cần:
1. Viết CDS model (define entity)
2. Viết CDS service (expose entity)
→ CAP tự generate: endpoints, CRUD, validation cơ bản, OData
→ Chỉ viết JS khi cần custom logic đặc biệt
```

---

## 🏗️ Kiến trúc 3 lớp của CAP

CAP phân tách app thành 3 lớp rõ ràng:

```
┌─────────────────────────────────────────────────┐
│                  app/  (Lớp UI)                 │
│         SAP Fiori / React / Vue.js              │
│         Tiêu thụ OData API từ srv/              │
└─────────────────────────────────────────────────┘
                        ↑ OData API
┌─────────────────────────────────────────────────┐
│                  srv/  (Lớp Service)            │
│    .cds file: định nghĩa service, expose entity │
│    .js file:  custom handlers (business logic)  │
└─────────────────────────────────────────────────┘
                        ↑ CDS QL queries
┌─────────────────────────────────────────────────┐
│                  db/  (Lớp Data)                │
│    schema.cds: định nghĩa entities (tables)     │
│    data/*.csv: mock data                        │
│    ↓ deploy lên SQLite (dev) / HANA (prod)      │
└─────────────────────────────────────────────────┘
```

**Quy tắc đơn giản:**
- `db/` = định nghĩa **dữ liệu** (entity là gì, có những trường gì)
- `srv/` = định nghĩa **dịch vụ** (expose entity nào, với quyền gì)
- `app/` = **giao diện** người dùng (Fiori hoặc custom UI)

---

## 📦 Cấu trúc Project CAP

```
my-cap-project/
├── db/
│   ├── schema.cds          # Định nghĩa entities (tables)
│   └── data/
│       ├── my.Books.csv    # Mock data cho Books
│       └── my.Authors.csv  # Mock data cho Authors
├── srv/
│   ├── catalog-service.cds # Định nghĩa service (API)
│   └── catalog-service.js  # Custom logic (chỉ khi cần)
├── app/                    # UI (optional)
├── package.json
└── .cdsrc.json             # CAP configuration
```

---

## 🔄 CAP hoạt động như thế nào?

Đây là flow từ khi bạn viết CDS đến khi client gọi API:

```
Bạn viết:                     CAP tự làm:
─────────────────────────────────────────────────────
1. entity Books {           → Tạo bảng Books trong DB
   key ID : Integer;
   title  : String;
}

2. service CatalogService { → Tạo OData endpoint:
   entity Books as          →   GET    /catalog/Books
   projection on db.Books;  →   GET    /catalog/Books(1)
}                           →   POST   /catalog/Books
                            →   PATCH  /catalog/Books(1)
                            →   DELETE /catalog/Books(1)

3. cds watch                → Start server localhost:4004
                            → Hot reload khi sửa file
```

> **Điểm mấu chốt:** Bạn **không** viết route handlers, không viết SQL, không viết CRUD. CAP tự generate tất cả từ CDS model.

---

## ⚡ CDS — Trái tim của CAP

**CDS (Core Data Services)** là ngôn ngữ khai báo đặc biệt của SAP CAP, dùng để định nghĩa data models và services.

Thay vì code imperative (nói "làm thế nào"), CDS là declarative (nói "cái gì"):

```cds
// Bạn khai báo: "Tôi muốn có entity Books với các trường này"
entity Books {
  key ID     : Integer;
  title      : String(100);
  stock      : Integer;
  price      : Decimal(9,2);
}

// CAP sẽ tự tạo ra SQL table, OData endpoints, validation...
```

CDS có 4 thành phần:

| Tên | Viết tắt | Là gì |
|-----|---------|-------|
| **CDS Definition Language** | CDL | Ngôn ngữ viết file `.cds` |
| **CDS Schema Notation** | CSN | Output JSON sau khi compile `.cds` |
| **CDS Query Language** | CQL | Ngôn ngữ query dữ liệu trong JS handler |
| **CDS Query Notation** | CQN | Object representation của CQL query |

> Giai đoạn đầu: chỉ cần biết **CDL** (viết `.cds` file) và **CQL** (query trong handler).

---

## 📡 OData — Protocol API của CAP

Thay vì REST thuần, CAP expose API theo protocol **OData v4**.

OData giống REST nhưng có một query language chuẩn hóa built-in:

```
REST thông thường:             OData (CAP):
────────────────────           ─────────────────────────────────
GET /books                     GET /catalog/Books
GET /books?author=1&limit=10   GET /catalog/Books?$filter=author_ID eq 1&$top=10
GET /books/1                   GET /catalog/Books(1)
GET /books/1/author            GET /catalog/Books(1)?$expand=author
```

OData cho phép client **tự tạo query**, không cần backend viết thêm endpoint tùy chỉnh. Đây là lý do enterprise apps ưa dùng OData.

---

## 🎯 Event Handlers — Viết custom logic

Khi cần logic đặc biệt ngoài CRUD mặc định, bạn viết **JS handler**:

```
Lifecycle của 1 request trong CAP:
─────────────────────────────────────────────────────
Request đến
    ↓
[BEFORE]  → Validate, transform input trước khi xử lý
    ↓
[ON]      → Xử lý chính (mặc định: CAP tự xử lý CRUD)
    ↓
[AFTER]   → Transform output, add computed fields
    ↓
Response trả về
```

```javascript
// srv/catalog-service.js
module.exports = (srv) => {

  // BEFORE: chạy trước khi tạo entity
  srv.before('CREATE', 'Books', (req) => {
    if (req.data.stock < 0) {
      req.error(400, 'Stock không được âm');
    }
  });

  // AFTER: chạy sau khi đọc xong
  srv.after('READ', 'Books', (books) => {
    books.forEach(b => {
      b.isLowStock = b.stock < 10; // thêm field computed
    });
  });

};
```

---

## 🆚 CAP vs Framework thuần Node.js

| | Framework thường | SAP CAP |
|---|---|---|
| Viết CRUD | Tự viết từng endpoint | Tự động generate |
| Data model | Code + SQL riêng biệt | CDS: 1 file = cả 2 |
| API Protocol | REST tự define | OData chuẩn hóa |
| Auth | Tự implement | `@requires` trong CDS |
| DB migration | Tự viết | `cds deploy` tự lo |
| Mock data | Tự setup | CSV files trong `db/data/` |
| Dev server | `npm run dev` | `cds watch` (hot reload) |

---

## 📋 Checklist — Sau khi đọc note này

- [ ] Hiểu được 3 lớp: `db/`, `srv/`, `app/`
- [ ] Biết CDS là ngôn ngữ khai báo dùng để define entities và services
- [ ] Hiểu CAP tự generate OData endpoints từ CDS model
- [ ] Biết lifecycle của request: BEFORE → ON → AFTER
- [ ] Đọc: [CAP Architecture Overview](https://cap.cloud.sap/docs/about/)

---

## 🔗 Tài liệu tham khảo

- [CAP Official Documentation](https://cap.cloud.sap/docs/)
- [CAP Getting Started](https://cap.cloud.sap/docs/get-started/)
- [Bookshop Sample — code mẫu chuẩn nhất](https://github.com/SAP-samples/cloud-cap-samples)

---

> **Tóm lại:** CAP = viết CDS (khai báo data + service) → framework tự generate API. Bạn chỉ viết JS khi cần business logic đặc biệt. Học CDS syntax là bước quan trọng nhất.
