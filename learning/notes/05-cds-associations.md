# 🔗 CDS Associations & Compositions — Quan Hệ Giữa Entities

> Đây là phần **khó nhất và quan trọng nhất** của CDS. Hiểu rõ Association vs Composition sẽ quyết định data model của bạn có đúng đắn không.

---

## 🤔 Hai loại quan hệ trong CDS

CDS có 2 từ khóa để diễn tả quan hệ:

| | Association | Composition |
|---|---|---|
| **Từ khóa** | `Association to` | `Composition of` |
| **Ý nghĩa** | "tham chiếu tới" | "chứa đựng" |
| **Lifecycle** | Độc lập | Con phụ thuộc cha |
| **Xóa cha** | Con vẫn tồn tại | Con bị xóa theo (cascade) |
| **Ví dụ thực tế** | Book → Author | Order → OrderItems |
| **OData** | Navigation property | Contained navigation |

---

## 🔗 Association — Quan hệ lỏng lẻo

Dùng khi 2 entity có thể **tồn tại độc lập** với nhau.

### Association to-one (nhiều → một)

```cds
entity Books {
  key ID     : UUID;
  title      : String;
  author     : Association to Authors;    // nhiều Books → 1 Author
}

entity Authors {
  key ID     : UUID;
  name       : String;
}
```

**CAP tự làm gì?**
- Tự generate `author_ID` foreign key trong table `Books`
- Tự tạo OData navigation: `GET /catalog/Books(1)?$expand=author`
- Bạn **không** cần khai báo `author_ID` thủ công

```
Table Books (generated):
┌────┬───────────────────┬───────────────────────────┐
│ ID │ title             │ author_ID  ← FK tự gen    │
├────┼───────────────────┼───────────────────────────┤
│ 1  │ Jane Eyre         │ 101                       │
│ 2  │ Wuthering Heights │ 101                       │
│ 3  │ The Alchemist     │ 102                       │
└────┴───────────────────┴───────────────────────────┘
```

### Association to-many (một → nhiều)

```cds
entity Authors {
  key ID    : UUID;
  name      : String;
  books     : Association to many Books on books.author = $self;
  //                                     ↑ điều kiện join
  //                                        $self = bản ghi Authors hiện tại
}
```

> `$self` = tham chiếu tới entity hiện tại (Authors). Đây là **back-link**, dùng để navigate ngược.

### Test OData Navigation

```bash
# Lấy sách kèm thông tin tác giả
GET /catalog/Books?$expand=author

# Lấy tác giả kèm danh sách sách
GET /catalog/Authors?$expand=books

# Filter theo thuộc tính của author
GET /catalog/Books?$filter=author/name eq 'Charlotte Brontë'
```

---

## 🧩 Composition — Quan hệ chứa đựng

Dùng khi entity con **không thể tồn tại** nếu không có entity cha. Con là một phần của cha.

### Ví dụ điển hình — Order và OrderItems

```cds
entity Orders {
  key ID     : UUID;
  customer   : String;
  orderDate  : Date;
  items      : Composition of many OrderItems on items.order = $self;
  //           ↑ OrderItems thuộc về Orders
}

entity OrderItems {
  key order  : Association to Orders;    // FK về cha
  key pos    : Integer;                  // số thứ tự trong order
  product    : String;
  quantity   : Integer;
  price      : Decimal(9,2);
}
```

**Hành vi khác biệt quan trọng:**

```
Khi xóa Order:
  Association → OrderItems VẪN tồn tại (có thể trở thành orphan)
  Composition → OrderItems bị xóa theo tự động (cascade delete) ✅

Khi tạo Order mới (Deep Insert):
  Association → phải tạo riêng từng entity
  Composition → tạo Order + Items trong 1 request ✅
```

### Deep Insert — Tạo cha + con trong 1 request

```javascript
// POST /orders
// Body:
{
  "customer": "John Doe",
  "orderDate": "2026-03-05",
  "items": [                    // ← items tạo cùng lúc với order
    { "pos": 1, "product": "Book A", "quantity": 2, "price": 12.99 },
    { "pos": 2, "product": "Book B", "quantity": 1, "price": 9.99 }
  ]
}
// → CAP tự tạo Order + 2 OrderItems trong 1 transaction
```

---

## 🆚 So sánh trực quan

```
Tình huống: Xóa Order #1

Association (nếu dùng sai):
  Orders          OrderItems
  ──────          ──────────────────────
  [Order #1] ×   [Item pos=1, order=1]  → còn lại! (orphan)
                  [Item pos=2, order=1]  → còn lại! (orphan)

Composition (đúng):
  Orders          OrderItems
  ──────          ──────────────────────
  [Order #1] ×   [Item pos=1, order=1]  → xóa theo ✅
                  [Item pos=2, order=1]  → xóa theo ✅
```

---

## 🎛️ Managed vs Unmanaged Association

### Managed Association (khuyến nghị — dùng đa số)

CAP **tự generate** foreign key. Bạn chỉ cần khai báo:

```cds
entity Books {
  key ID    : UUID;
  author    : Association to Authors;    // managed
  // → CAP tự tạo field "author_ID" trong DB
}
```

### Unmanaged Association (dùng khi cần logic join phức tạp)

Bạn **tự khai báo** điều kiện join trong `on` clause:

```cds
entity Books {
  key ID          : UUID;
  author_ID       : UUID;                         // bạn tự khai báo FK
  author          : Association to Authors
    on author.ID = author_ID;                     // bạn tự viết điều kiện join
}
```

**Khi nào dùng unmanaged?**
- Join trên nhiều columns
- Join với điều kiện phức tạp hơn bằng FK đơn giản
- Entity đã có sẵn FK field (không muốn duplicate)

```cds
// Ví dụ: join trên 2 columns
entity PriceHistory {
  key product_ID    : UUID;
  key currency_code : String(3);
  amount            : Decimal;

  product  : Association to Products
    on  product.ID = product_ID;

  currency : Association to Currencies
    on  currency.code = currency_code;
}
```

---

## 🔄 Cách viết Back-link (two-way navigation)

Để navigate được từ **cả hai phía**, khai báo 2 association:

```cds
namespace my.bookshop;

entity Books {
  key ID     : UUID;
  title      : String;
  author     : Association to Authors;          // Books → Authors (to-one)
}

entity Authors {
  key ID     : UUID;
  name       : String;
  books      : Association to many Books        // Authors → Books (to-many)
               on books.author = $self;         // ← back-link
}
```

> **Lưu ý:** Back-link (`books`) không tạo thêm column trong DB. Nó chỉ là metadata để CAP biết cách tạo JOIN query khi cần.

---

## 🧪 OData với Associations & Compositions

### Navigation thông qua Association

```
# Expand — JOIN inline
GET /catalog/Books?$expand=author
→ Trả về Books, mỗi book có object author nhúng bên trong

GET /catalog/Authors?$expand=books
→ Trả về Authors, mỗi author có array books

# Navigate path
GET /catalog/Books(1)/author
→ Trả về author của Book(1)

GET /catalog/Authors(1)/books
→ Trả về danh sách books của Author(1)
```

### Deep Navigation với Composition

```
# OrderItems là contained entity — chỉ access qua cha
GET /orders/Orders(1)/items
→ Trả về items của Order(1)

GET /orders/Orders(1)/items(pos=1)
→ Trả về item pos=1 của Order(1)

# PATCH nested — cập nhật item trong order
PATCH /orders/Orders(1)/items(pos=1)
Body: { "quantity": 5 }
```

---

## 📐 Best Practices

| Tình huống | Dùng loại nào |
|---|---|
| Sách ↔ Tác giả (2 entity độc lập) | `Association` |
| Đơn hàng → Chi tiết đơn hàng | `Composition` |
| Sản phẩm ↔ Danh mục | `Association` |
| Hóa đơn → Dòng hóa đơn | `Composition` |
| Blog Post → Comments | `Composition` |
| Employee → Department | `Association` |

**Nguyên tắc đơn giản:**
> Nếu xóa cha mà con **không còn ý nghĩa** → dùng **Composition**  
> Nếu xóa cha mà con **vẫn có ý nghĩa riêng** → dùng **Association**

---

## ✅ Checklist thực hành

- [ ] Tạo model `Books` ↔ `Authors` với `Association to` và back-link
- [ ] Tạo model `Orders` → `OrderItems` với `Composition of many`
- [ ] Test `$expand=author` trên `/catalog/Books`
- [ ] Test deep insert: POST order kèm items trong 1 request
- [ ] Xóa 1 Order → verify OrderItems bị xóa theo
- [ ] Thử dùng `$filter=author/name eq '...'` để filter qua association
- [ ] Chạy `cds compile db/ --to sql` → kiểm tra FK được generate

---

## 🔗 Tài liệu tham khảo

- [CDS — Associations](https://cap.cloud.sap/docs/cds/cdl#associations)
- [CDS — Compositions](https://cap.cloud.sap/docs/cds/cdl#compositions)
- [Managed Associations](https://cap.cloud.sap/docs/cds/cdl#managed-associations)
- [Deep Insert/Update](https://cap.cloud.sap/docs/guides/providing-services#deep-insert-update)

---

> **Tóm lại:**  
> - **Association** = quan hệ lỏng, 2 entity sống độc lập, dùng FK (`author_ID` tự gen)  
> - **Composition** = quan hệ chặt, con sống trong cha, xóa cha thì con xóa theo, hỗ trợ deep insert  
> - **Managed** = CAP tự gen FK — dùng hầu hết trường hợp  
> - **Unmanaged** = bạn tự viết `on` clause — dùng khi cần join phức tạp
