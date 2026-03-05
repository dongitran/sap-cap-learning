# 🧩 CDS Aspects — Tái Sử Dụng Model

> **Aspect** trong CDS = **Mixin** — một tập hợp các fields và behaviors có thể "trộn vào" (mix in) nhiều entities khác nhau, giúp tránh lặp code và đảm bảo nhất quán.

---

## 🤔 Aspect là gì và tại sao cần?

Hãy xem vấn đề này: bạn có nhiều entities và mỗi entity đều cần các fields tracking cơ bản:

```cds
// ❌ Cách không dùng Aspect — lặp lại ở mọi entity:
entity Books {
  key ID         : UUID;
  title          : String;
  createdAt      : Timestamp;   // lặp
  createdBy      : String;      // lặp
  modifiedAt     : Timestamp;   // lặp
  modifiedBy     : String;      // lặp
}

entity Orders {
  key ID         : UUID;
  customer       : String;
  createdAt      : Timestamp;   // lặp lại
  createdBy      : String;      // lặp lại
  modifiedAt     : Timestamp;   // lặp lại
  modifiedBy     : String;      // lặp lại
}

// 😭 Nếu có 20 entities → copy paste 20 lần → khó maintain
```

```cds
// ✅ Cách dùng Aspect — định nghĩa 1 lần, dùng nhiều nơi:
aspect Tracked {            // định nghĩa aspect
  createdAt  : Timestamp;
  createdBy  : String;
  modifiedAt : Timestamp;
  modifiedBy : String;
}

entity Books : Tracked {    // "mix in" aspect vào entity
  key ID  : UUID;
  title   : String;
  // → tự động có createdAt, createdBy, modifiedAt, modifiedBy
}

entity Orders : Tracked {   // dùng lại cùng aspect
  key ID      : UUID;
  customer    : String;
  // → cũng có đủ 4 tracking fields
}
```

---

## 📚 Cú pháp Aspect

```cds
// Định nghĩa aspect
aspect TenAspect {
  field1 : Type1;
  field2 : Type2;
}

// Áp dụng vào entity (dùng dấu : )
entity MyEntity : TenAspect {
  key ID : UUID;
  // ... các fields riêng
}

// Áp dụng nhiều aspects cùng lúc
entity MyEntity : Aspect1, Aspect2, Aspect3 {
  key ID : UUID;
}
```

**Aspect có thể chứa:**
- Elements (fields)
- Associations / Compositions
- Annotations
- Actions và Functions

---

## 🏗️ Ba Built-in Aspects của SAP (từ `@sap/cds/common`)

SAP cung cấp sẵn 3 aspect cực kỳ hữu ích trong package `@sap/cds/common`. Đây là những thứ bạn sẽ dùng **hàng ngày** khi viết CAP app.

### 1. `cuid` — UUID key tự generate

```cds
using { cuid } from '@sap/cds/common';

entity Books : cuid {    // thay vì tự viết key ID : UUID
  title  : String;
  stock  : Integer;
}

// Tương đương với:
entity Books {
  key ID : UUID;         // @Core.Computed — không nhận từ client
  title  : String;
  stock  : Integer;
}
```

**CAP tự làm gì khi dùng `cuid`:**
```
POST /catalog/Books
Body: { "title": "1984", "stock": 10 }

→ CAP tự generate UUID cho ID
→ Response: { "ID": "a3f4b2-...", "title": "1984", "stock": 10 }
→ Client không cần gửi ID lên
```

> **Khi nào dùng `cuid`:** Hầu hết các entity trong production. Chỉ không dùng khi cần composite key hoặc business key tự định nghĩa.

---

### 2. `managed` — Tự điền thông tin audit

```cds
using { managed } from '@sap/cds/common';

entity Orders : managed {
  key ID    : UUID;
  customer  : String;
  // → tự có: createdAt, createdBy, modifiedAt, modifiedBy
}
```

**Fields được thêm vào:**

| Field | Type | Tự điền khi |
|---|---|---|
| `createdAt` | Timestamp | Entity được tạo (INSERT) |
| `createdBy` | String(255) | Entity được tạo (INSERT) |
| `modifiedAt` | Timestamp | Entity được tạo/sửa (INSERT và UPDATE) |
| `modifiedBy` | String(255) | Entity được tạo/sửa (INSERT và UPDATE) |

**CAP tự điền bằng cách nào?**
```javascript
// CAP đọc user từ request context
// Khi có XSUAA auth: lấy username từ JWT token
// Khi mock auth: lấy từ header hoặc default

// createdBy / modifiedBy = req.user.id (auth service trả về)
// createdAt / modifiedAt = new Date() tại thời điểm request
```

**Test local (không có auth):**
```bash
GET /catalog/Orders/1
→ {
    "ID": "...",
    "customer": "John",
    "createdAt": "2026-03-05T07:14:37Z",
    "createdBy": "anonymous",    ← default khi không có auth
    "modifiedAt": "2026-03-05T07:14:37Z",
    "modifiedBy": "anonymous"
  }
```

---

### 3. `temporal` — Dữ liệu theo thời gian

`temporal` dùng khi bạn cần lưu **lịch sử thay đổi theo thời gian** — ví dụ: bảng giá thay đổi theo ngày, hợp đồng theo giai đoạn hiệu lực.

```cds
using { temporal } from '@sap/cds/common';

entity PriceList : temporal {
  key productID : UUID;
  price         : Decimal(9,2);
  // → tự có: validFrom, validTo
}
```

**Fields được thêm vào:**

| Field | Type | Ý nghĩa |
|---|---|---|
| `validFrom` | Timestamp | Hiệu lực từ ngày |
| `validTo` | Timestamp | Hiệu lực đến ngày |

**CAP tự xử lý "time travel queries":**

```
Dữ liệu trong DB:
productID | price | validFrom           | validTo
─────────────────────────────────────────────────
P001      | 10.00 | 2026-01-01T00:00:00 | 2026-06-30T23:59:59
P001      | 12.00 | 2026-07-01T00:00:00 | 9999-12-31T23:59:59

GET /catalog/PriceList(productID='P001')
→ CAP tự lọc: validFrom <= NOW() AND validTo >= NOW()
→ Trả về giá hiện tại (12.00 nếu sau 1 July 2026)

# Time travel — lấy giá tại 1 ngày cụ thể trong quá khứ:
GET /catalog/PriceList(productID='P001')?sap-valid-at=2026-03-01
→ Trả về 10.00 (giá hồi tháng 3)
```

> `temporal` rất hữu ích cho pricing, contracts, employee assignments — bất kỳ dữ liệu nào thay đổi theo thời gian.

---

## 🔧 Kết hợp cuid + managed (Pattern Phổ Biến Nhất)

Trong thực tế, hầu hết entities dùng cả 2:

```cds
using { cuid, managed } from '@sap/cds/common';

entity Books : cuid, managed {
  title      : String(100) not null;
  stock      : Integer default 0;
  price      : Decimal(9,2);
}

// Kết quả — entity này có đủ:
// key ID         : UUID;          ← từ cuid
// createdAt      : Timestamp;     ← từ managed
// createdBy      : String(255);   ← từ managed
// modifiedAt     : Timestamp;     ← từ managed
// modifiedBy     : String(255);   ← từ managed
// title, stock, price             ← của riêng entity
```

---

## 🎨 Custom Aspect — Tự tạo aspect của bạn

```cds
// Tạo aspect tái sử dụng cho toàn team
aspect Address {
  street     : String(100);
  city       : String(50);
  postalCode : String(10);
  country    : String(2);
}

aspect ContactInfo {
  phone : String(20);
  email : String(200);
}

// Dùng lại trong nhiều entities
entity Customers : cuid, managed, Address, ContactInfo {
  firstName : String(50);
  lastName  : String(50);
}

entity Suppliers : cuid, managed, Address, ContactInfo {
  companyName : String(100);
  taxId       : String(20);
}
```

**Custom aspect với Associations:**

```cds
// Aspect chứa cả association
aspect WithCategory {
  category : Association to Categories;
}

entity Products : cuid, managed, WithCategory {
  name  : String;
  price : Decimal(9,2);
}

entity Articles : cuid, managed, WithCategory {
  title   : String;
  content : LargeString;
}
// → Cả Products và Articles đều có field category_ID (FK)
```

---

## 📦 `@sap/cds/common` — Thư viện Types

Ngoài 3 aspects trên, `@sap/cds/common` còn cung cấp các reuse types cho internationalization:

```cds
using { Country, Currency, Language } from '@sap/cds/common';

entity Products : cuid, managed {
  name       : String;
  price      : Decimal(9,2);
  currency   : Currency;      // Association to sap.common.Currencies
  origin     : Country;       // Association to sap.common.Countries
}
```

| Type | Là gì | Ví dụ giá trị |
|---|---|---|
| `Country` | Association to Countries | 'US', 'VN', 'DE' |
| `Currency` | Association to Currencies | 'USD', 'VND', 'EUR' |
| `Language` | Association to Languages | 'en', 'vi', 'de' |
| `Timezone` | String(100) | 'Asia/Ho_Chi_Minh' |

```bash
# Để load data cho các code lists (Countries, Currencies, Languages):
cds add data      # thêm initial data
# → Tự động có CSV data cho Countries, Currencies, Languages
```

---

## 🆚 So sánh: Aspect vs Inheritance

CDS không có traditional inheritance như OOP. Aspect là cơ chế duy nhất để tái sử dụng:

| | Aspect (CDS) | Class Inheritance (OOP) |
|---|---|---|
| Dùng nhiều cùng lúc | ✅ `:Aspect1, Aspect2` | ❌ (single inheritance) |
| Overridable | ✅ Có thể override element | ✅ |
| Generate thêm table | ❌ Không | Tùy DB |
| Kết hợp với annotations | ✅ | Phức tạp hơn |

---

## ⚠️ Lưu ý quan trọng

1. **`managed` fields là readonly** — Client không được gửi lên, CAP tự điền:
   ```javascript
   // BAD: gửi lên sẽ bị ignore hoặc error
   POST /orders { "createdBy": "hacker", ... }

   // GOOD: chỉ gửi business data
   POST /orders { "customer": "John", ... }
   ```

2. **`cuid` ID là readonly** — không thể set từ client:
   ```javascript
   // BAD: ID sẽ bị ignore
   POST /books { "ID": "my-custom-id", "title": "1984" }

   // GOOD: CAP tự gen UUID
   POST /books { "title": "1984" }
   ```

3. **Aspect không tạo bảng riêng** — Tất cả fields của aspect được "flatten" vào bảng của entity.

---

## ✅ Checklist thực hành

- [ ] Dùng `cuid` cho tất cả entities mới (không write `key ID : UUID` thủ công nữa)
- [ ] Dùng `managed` cho entities cần audit trail (tạo bởi ai, lúc nào)
- [ ] Tạo 1 custom aspect `Address` với các fields địa chỉ, dùng cho 2 entities
- [ ] Thêm `Currency` từ `@sap/cds/common` vào entity Products
- [ ] Test: POST 1 entity → verify `createdAt`, `createdBy`, `ID` được tự điền
- [ ] Chạy `cds compile db/ --to sql` → kiểm tra tất cả fields có trong output

---

## 🔗 Tài liệu tham khảo

- [CDS Aspects](https://cap.cloud.sap/docs/cds/cdl#aspects)
- [@sap/cds/common Reference](https://cap.cloud.sap/docs/cds/common)
- [Domain Modeling — Reuse](https://cap.cloud.sap/docs/guides/domain-modeling#use-managed-aspects)
- [Temporal Data](https://cap.cloud.sap/docs/guides/temporal-data)

---

> **Tóm lại:**  
> - **Aspect** = mixin, tái sử dụng fields/behaviors giữa nhiều entities  
> - **`cuid`** = tự gen UUID key, không cần client gửi ID  
> - **`managed`** = tự điền createdAt/By, modifiedAt/By — must-have cho audit  
> - **`temporal`** = dữ liệu có thời hạn hiệu lực, hỗ trợ time-travel queries  
> - Pattern phổ biến nhất: `entity Foo : cuid, managed { ... }`
