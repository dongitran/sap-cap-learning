# 🧱 CDS Entities — Data Modeling Cơ Bản

> **Entity** trong CDS = **Table** trong database. Đây là nền tảng của mọi CAP app — bạn phải nắm chắc phần này trước khi học bất cứ thứ gì khác.

---

## 🤔 Entity là gì?

Trong CAP, thay vì viết SQL `CREATE TABLE`, bạn viết entity trong file `.cds`:

```
SQL (cách cũ):                      CDS (cách CAP):
──────────────────────────          ──────────────────────────
CREATE TABLE Books (                entity Books {
  ID INTEGER PRIMARY KEY,             key ID    : Integer;
  title VARCHAR(100),                 title     : String(100);
  stock INTEGER                       stock     : Integer;
);                                  }

→ Bạn tự viết SQL                  → CAP tự generate SQL + OData
```

**Một entity được compile thành:**
- ✅ Một **table** trong database (SQLite / HANA)
- ✅ Các **OData endpoints** CRUD tự động (khi expose qua service)
- ✅ **JavaScript types** để dùng trong handler

---

## 📝 Cú pháp cơ bản

```cds
namespace my.company;     // tổ chức entity vào namespace

entity Books {            // tên entity = tên table
  key ID    : Integer;    // PRIMARY KEY
  title     : String(100);
  stock     : Integer;
  price     : Decimal(9,2);
  active    : Boolean default true;
  createdAt : Timestamp;
}
```

**Quy tắc đặt tên:**
- Entity: **PascalCase** (Books, OrderItems, CustomerProfile)
- Element (field): **camelCase** (firstName, orderDate, totalAmount)
- Namespace: **lowercase với dấu chấm** (my.company, sap.common)

---

## 🔑 Key Elements

`key` = primary key của entity. Một entity phải có **ít nhất 1** key element.

```cds
entity Books {
  key ID : Integer;           // simple integer key (tự increment bằng CSV mock)
  title  : String;
}

// ✅ Composite key (nhiều field làm key cùng nhau)
entity OrderItems {
  key order  : Association to Orders;   // FK + key
  key pos    : Integer;
  quantity   : Integer;
}

// ✅ UUID key (phổ biến nhất trong production)
entity Products {
  key ID    : UUID;       // gen tự động nếu dùng aspect cuid
  name      : String;
}
```

> **Khuyến nghị:** Dùng `UUID` key thay `Integer` cho production. Integer thường chỉ dùng trong tutorial/mock data.

---

## 📦 Các Built-in Types

CDS có đầy đủ các kiểu dữ liệu, tương tự TypeScript/SQL:

### Kiểu cơ bản

| CDS Type | Tương đương | Ghi chú |
|---|---|---|
| `String` | VARCHAR | `String(100)` = độ dài tối đa 100 |
| `String` (không limit) | NVARCHAR(5000) | Dùng cho text ngắn |
| `LargeString` | NCLOB | Text dài, không dùng trong filter |
| `Integer` | INTEGER | 32-bit |
| `Int64` | BIGINT | 64-bit, dùng khi cần số lớn |
| `Decimal(p,s)` | DECIMAL | `p` = tổng chữ số, `s` = chữ số thập phân |
| `Double` | DOUBLE | Số thực, ít dùng hơn Decimal |
| `Boolean` | BOOLEAN | true/false |
| `UUID` | VARCHAR(36) | `'xxxxxxxx-xxxx-...'` format |
| `Date` | DATE | `'2024-01-15'` |
| `Time` | TIME | `'14:30:00'` |
| `DateTime` | TIMESTAMP | `'2024-01-15T14:30:00'` (giây) |
| `Timestamp` | TIMESTAMP | `'2024-01-15T14:30:00.000Z'` (millisecond) |
| `Binary` | VARBINARY | Blob nhỏ |
| `LargeBinary` | BLOB | File, image |

### Ví dụ thực tế

```cds
entity Customers {
  key ID          : UUID;
  firstName       : String(50);
  lastName        : String(50);
  email           : String(100);
  phone           : String(20);
  dateOfBirth     : Date;
  registeredAt    : Timestamp;
  creditLimit     : Decimal(15,2);
  isActive        : Boolean default true;
  profilePicture  : LargeBinary;           // file ảnh
  notes           : LargeString;           // text dài
}
```

### Enum Types

Dùng khi field chỉ nhận một trong số giá trị xác định:

```cds
type OrderStatus : String enum {
  Pending   = 'P';
  Confirmed = 'C';
  Shipped   = 'S';
  Cancelled = 'X';
}

entity Orders {
  key ID  : UUID;
  status  : OrderStatus default 'P';    // chỉ nhận P, C, S, X
}
```

### Custom Types (tái sử dụng)

```cds
// Định nghĩa type tái dụng
type Name        : String(100);
type Email       : String(200);
type Money       : Decimal(15,2);
type PhoneNumber : String(20);

entity Employees {
  key ID    : UUID;
  firstName : Name;      // dùng lại type
  lastName  : Name;
  email     : Email;
  salary    : Money;
}

entity Contacts {
  key ID    : UUID;
  name      : Name;      // dùng lại cùng type
  email     : Email;
  phone     : PhoneNumber;
}
```

---

## 🏷️ Namespaces — Tổ chức entities

Namespace giúp tránh xung đột tên khi project lớn:

```cds
// Khai báo namespace — áp dụng cho tất cả definitions trong file
namespace my.company.hr;

entity Employees { ... }     // tên đầy đủ: my.company.hr.Employees
entity Departments { ... }   // tên đầy đủ: my.company.hr.Departments
```

```cds
namespace my.company.sales;

entity Orders { ... }        // my.company.sales.Orders
entity Customers { ... }     // my.company.sales.Customers
```

**Import từ file khác:**

```cds
// srv/catalog-service.cds
using my.company.hr   as hr   from '../db/hr-schema';
using my.company.sales as sales from '../db/sales-schema';

service HRService {
  entity Employees as projection on hr.Employees;
}

service SalesService {
  entity Orders as projection on sales.Orders;
}
```

---

## 🔧 Element Modifiers

### `default` — Giá trị mặc định

```cds
entity Orders {
  key ID    : UUID;
  status    : String(1) default 'P';      // mặc định 'P' (Pending)
  priority  : Integer default 5;
  active    : Boolean default true;
  createdAt : Timestamp default $now;     // $now = thời điểm tạo
}
```

### `not null` — Bắt buộc nhập

```cds
entity Employees {
  key ID    : UUID;
  firstName : String(50) not null;    // bắt buộc
  lastName  : String(50) not null;
  middleName: String(50);             // optional (default)
}
```

### Annotations — Metadata bổ sung

Annotations bắt đầu bằng `@`, không ảnh hưởng DB schema nhưng dùng cho UI, validation, OData:

```cds
entity Books {
  @mandatory        // UI validation: field bắt buộc
  key ID    : UUID;
  
  @title: 'Book Title'     // label hiển thị trên UI
  @mandatory
  title     : String(100);
  
  @readonly         // không cho phép cập nhật qua API
  createdAt : Timestamp;
  
  @assert.range: [0, 9999]   // validate range
  stock     : Integer;
}
```

---

## 📂 Tổ chức file CDS

```
db/
├── schema.cds          # File chính → import từ các file con
├── books.cds           # Entity Books, Authors
├── orders.cds          # Entity Orders, OrderItems
└── customers.cds       # Entity Customers
```

```cds
// db/schema.cds — file tổng hợp
using from './books';
using from './orders';
using from './customers';
```

```cds
// db/books.cds
namespace my.bookshop;

entity Books {
  key ID    : UUID;
  title     : String(100) not null;
  stock     : Integer default 0;
  price     : Decimal(9,2);
}

entity Authors {
  key ID    : UUID;
  name      : String(100) not null;
}
```

---

## 🗃️ Mock Data — CSV

Mỗi entity có thể có CSV file để generate data khi `cds watch`:

```
db/data/
├── my.bookshop-Books.csv      # format: <namespace>-<EntityName>.csv
└── my.bookshop-Authors.csv
```

```csv
# my.bookshop-Books.csv
ID,title,stock,price
"7e2f2640-6866-4dcf-8f4d-3027aa831cad","The Great Gatsby",10,12.99
"d85e2d41-e2a9-4dff-ac47-fc7b07b9c9c1","1984",5,9.99
"3451a9e4-bfef-442e-8e98-ab70dbf73f95","To Kill a Mockingbird",20,11.50
```

> **Tên file quan trọng:** Phải đúng format `<namespace>-<EntityName>.csv`. Ví dụ namespace `my.bookshop` + entity `Books` → `my.bookshop-Books.csv`.

---

## ⚠️ Một số lỗi hay gặp

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| Entity không generate table | Không có key element | Thêm `key ID : ...` |
| CSV data không load | Tên file sai | Kiểm tra namespace-EntityName.csv |
| OData không trả field | Field là `LargeString` hoặc `LargeBinary` | Dùng `$select` để explicitly lấy |
| Circular dependency | 2 file import nhau | Dùng 1 file schema.cds tổng hợp |
| Type mismatch | Giá trị CSV sai kiểu | Kiểm tra type trong entity |

---

## ✅ Checklist thực hành

- [ ] Tạo entity `Books` với các types khác nhau (String, Integer, Decimal, Boolean, UUID, Timestamp)
- [ ] Tạo `enum type` cho status field
- [ ] Tạo `custom type` tái sử dụng (ví dụ: `Name`, `Money`)
- [ ] Dùng `default` và `not null` trên các fields
- [ ] Tổ chức entities vào namespace `my.xxx`
- [ ] Tạo CSV mock data và verify data load trong `cds watch`
- [ ] Chạy `cds compile db/ --to sql` để xem SQL được generate

---

## 🔗 Tài liệu tham khảo

- [CDS Types Reference](https://cap.cloud.sap/docs/cds/types)
- [CDS Definition Language (CDL)](https://cap.cloud.sap/docs/cds/cdl)
- [Domain Modeling Guide](https://cap.cloud.sap/docs/guides/domain-modeling)
- [CDS Common Types (`@sap/cds/common`)](https://cap.cloud.sap/docs/cds/common)

---

> **Tóm lại:** Entity = table, element = column, key = primary key. Dùng UUID cho production, Integer cho tutorial. Namespace giúp tổ chức khi project lớn. Annotations là metadata bổ sung, không ảnh hưởng DB.
