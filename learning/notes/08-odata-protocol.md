# 🌐 OData Protocol — Ngôn Ngữ Query của SAP

> **OData** (Open Data Protocol) là protocol API tiêu chuẩn hóa của SAP thay thế REST thông thường. CAP tự generate OData endpoints từ CDS model — bạn cần hiểu OData để query data đúng cách.

---

## 🤔 OData là gì và tại sao cần?

```
REST thông thường:                  OData (cách CAP):
─────────────────                   ─────────────────────────────────────
GET /books                          GET /catalog/Books
GET /books?page=1&limit=10          GET /catalog/Books?$top=10&$skip=0
GET /books?filter[status]=active    GET /catalog/Books?$filter=status eq 'active'
GET /books?sort=-price              GET /catalog/Books?$orderby=price desc
GET /books?include[]=author         GET /catalog/Books?$expand=author

→ Mỗi API tự định nghĩa format  → Chuẩn hóa hoàn toàn
→ Cần viết query logic phức tạp → CAP tự xử lý, không cần viết SQL
```

**OData v4** là version hiện tại (được CAP dùng), v3 đã deprecated.

---

## 📋 Cấu trúc URL OData

```
http://localhost:4004 /catalog       /Books       ?$filter=stock gt 0
─────────────────────  ─────────────  ───────────  ──────────────────────
Base URL               Service path   Entity name  Query options
```

```
Ví dụ đầy đủ:
GET /catalog/Books(1)                    → get entity by key
GET /catalog/Books(1)/author             → navigate to related entity
GET /catalog/Books?$expand=author        → include related data inline
GET /catalog/$metadata                   → service metadata (EDMX format)
```

---

## 🔍 $filter — Lọc dữ liệu

### Toán tử so sánh cơ bản

```bash
# eq (equal) — bằng
/catalog/Books?$filter=stock eq 0

# ne (not equal) — khác
/catalog/Books?$filter=status ne 'Cancelled'

# gt, ge (greater than, greater or equal) — lớn hơn
/catalog/Books?$filter=price gt 100
/catalog/Books?$filter=stock ge 10

# lt, le (less than, less or equal) — nhỏ hơn
/catalog/Books?$filter=price lt 50
/catalog/Books?$filter=stock le 5

# null check
/catalog/Books?$filter=author eq null
/catalog/Books?$filter=author ne null
```

### Toán tử logic

```bash
# and — cả hai điều kiện phải đúng
/catalog/Books?$filter=price lt 20 and stock gt 0

# or — ít nhất một điều kiện đúng
/catalog/Books?$filter=status eq 'New' or status eq 'Active'

# not — phủ định
/catalog/Books?$filter=not (status eq 'Cancelled')
```

### String functions

```bash
# contains — chứa chuỗi con
/catalog/Books?$filter=contains(title, 'Harry')

# startswith — bắt đầu bằng
/catalog/Books?$filter=startswith(title, 'The')

# endswith — kết thúc bằng
/catalog/Books?$filter=endswith(email, '@gmail.com')

# tolower/toupper — chuyển case (dùng khi search không phân biệt hoa thường)
/catalog/Books?$filter=tolower(title) eq 'harry potter'
```

### Date/Number functions

```bash
# year, month — lấy thành phần của date
/catalog/Orders?$filter=year(orderDate) eq 2026
/catalog/Orders?$filter=month(createdAt) eq 3

# round, floor, ceiling — làm tròn số
/catalog/Products?$filter=round(price) eq 10
```

### Lambda operators (filter trong array/collection)

```bash
# any — ít nhất 1 phần tử thỏa mãn
/catalog/Orders?$filter=items/any(i: i/quantity gt 10)
# = Lấy các Orders có ít nhất 1 item quantity > 10

# all — tất cả phần tử thỏa mãn
/catalog/Orders?$filter=items/all(i: i/product ne null)
# = Lấy các Orders mà tất cả items đều có product
```

---

## 📊 $select — Chọn fields cần lấy

```bash
# Chỉ lấy ID và title (giảm payload)
/catalog/Books?$select=ID,title

# Nhiều fields
/catalog/Books?$select=ID,title,stock,price

# Kết hợp với $expand — lấy cả fields của entity mở rộng
/catalog/Books?$select=ID,title&$expand=author($select=ID,name)
```

> **Tại sao dùng `$select`?**  
> `LargeString` và `LargeBinary` fields **không được trả về** trong list queries mặc định. Phải dùng `$select` để lấy chúng.

---

## 🔗 $expand — Load dữ liệu liên quan (JOIN)

```bash
# Expand association (JOIN 1 bảng)
/catalog/Books?$expand=author
# Kết quả: mỗi book có object author nhúng bên trong

# Expand composition (items thuộc về order)
/orders/Orders(1)?$expand=items
# Kết quả: order kèm array items

# Expand nhiều navigation cùng lúc
/catalog/Books?$expand=author,publisher
# Kết quả: mỗi book có cả author lẫn publisher

# Nested expand (expand trong expand)
/catalog/Books?$expand=author($expand=nationality)
# Kết quả: mỗi book có author, author có nationality

# Expand kèm filter và select
/catalog/Authors?$expand=books($filter=stock gt 0;$select=title,price)
# Kết quả: authors kèm chỉ những books còn hàng, chỉ lấy title và price
```

---

## 📐 $orderby — Sắp xếp kết quả

```bash
# Tăng dần (mặc định = asc)
/catalog/Books?$orderby=price
/catalog/Books?$orderby=price asc

# Giảm dần
/catalog/Books?$orderby=price desc

# Nhiều tiêu chí sort
/catalog/Books?$orderby=stock desc,price asc
# = Sort theo stock giảm dần, nếu bằng nhau thì sort theo price tăng dần

# Sort theo field của entity liên quan
/catalog/Books?$orderby=author/name asc
```

---

## 📄 $top và $skip — Phân trang

```bash
# Lấy 10 records đầu tiên
/catalog/Books?$top=10

# Skip 20 records đầu, lấy 10 tiếp theo (page 3)
/catalog/Books?$skip=20&$top=10

# Công thức phân trang:
# page N, size S → $skip=(N-1)*S&$top=S
```

---

## 🔢 $count — Đếm số lượng

```bash
# Trả về metadata gồm `@odata.count`
/catalog/Books?$count=true
# Response: { "@odata.count": 150, "value": [...] }

# Chỉ lấy count, không lấy data
/catalog/Books/$count
# Response: 150
```

> Hữu ích cho pagination: lấy total pages = `ceil(@odata.count / pageSize)`

---

## 🔎 $search — Full-text search

```bash
# Tìm kiếm toàn văn bản
/catalog/Books?$search=Harry Potter

# Kết hợp với $filter
/catalog/Books?$search=magic&$filter=stock gt 0
```

> `$search` tìm trong tất cả string fields. Hành vi cụ thể tùy database (SQLite vs HANA).

---

## 🔄 Kết hợp nhiều query options

```bash
# Ví dụ thực tế — books trang 2, sắp xếp theo giá, chỉ lấy 3 fields, kèm tác giả
/catalog/Books
  ?$filter=stock gt 0
  &$select=ID,title,price
  &$expand=author($select=name)
  &$orderby=price desc
  &$top=20
  &$skip=20
  &$count=true
```

---

## 📦 Định dạng Response OData

```json
// Response JSON tiêu chuẩn OData v4:
{
  "@odata.context": "$metadata#Books",
  "@odata.count": 150,
  "value": [
    {
      "ID": "a3f4b2-...",
      "title": "Harry Potter",
      "price": 12.99,
      "stock": 10,
      "author": {              
        "ID": "...",
        "name": "J.K. Rowling"
      }
    }
  ]
}
```

```json
// Single entity (Books/1):
{
  "@odata.context": "$metadata#Books/$entity",
  "ID": "a3f4b2-...",
  "title": "Harry Potter",
  "price": 12.99
}
```

---

## 📮 OData Mutations (Write operations)

```bash
# CREATE — POST với body JSON
curl -X POST /catalog/Books \
  -H "Content-Type: application/json" \
  -d '{"title": "New Book", "stock": 10, "price": 9.99}'

# UPDATE partial — PATCH (chỉ update fields được gửi)
curl -X PATCH /catalog/Books(1) \
  -H "Content-Type: application/json" \
  -d '{"stock": 5}'

# UPDATE full — PUT (replace toàn bộ entity)
curl -X PUT /catalog/Books(1) \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated", "stock": 5, "price": 9.99}'

# DELETE
curl -X DELETE /catalog/Books(1)
```

---

## 🗺️ OData Metadata ($metadata)

```bash
GET /catalog/$metadata
```

Trả về EDMX (XML) mô tả toàn bộ service:
- Tất cả entities và fields
- Tất cả relationships (navigation properties)
- Tất cả actions và functions

> Fiori Elements và nhiều tools SAP đọc `$metadata` để tự generate UI.

---

## 🛠️ Test OData trong VS Code

Cài extension **REST Client**, tạo file `.http`:

```http
### GET all Books
GET http://localhost:4004/catalog/Books
Accept: application/json

### GET with filters
GET http://localhost:4004/catalog/Books?$filter=stock gt 5&$select=ID,title,stock&$orderby=price desc&$top=5
Accept: application/json

### GET with expand
GET http://localhost:4004/catalog/Books?$expand=author
Accept: application/json

### CREATE Book
POST http://localhost:4004/catalog/Books
Content-Type: application/json

{
  "title": "Test Book",
  "stock": 10,
  "price": 9.99
}

### DELETE Book
DELETE http://localhost:4004/catalog/Books(1)
```

---

## 📋 Cheat Sheet nhanh

| Query option | Tương đương SQL | Ví dụ |
|---|---|---|
| `$filter=x eq 'v'` | `WHERE x = 'v'` | `?$filter=status eq 'P'` |
| `$filter=x gt 0` | `WHERE x > 0` | `?$filter=stock gt 0` |
| `$select=a,b` | `SELECT a, b` | `?$select=ID,title` |
| `$expand=rel` | `JOIN` | `?$expand=author` |
| `$orderby=x desc` | `ORDER BY x DESC` | `?$orderby=price desc` |
| `$top=N` | `LIMIT N` | `?$top=10` |
| `$skip=N` | `OFFSET N` | `?$skip=20` |
| `$count=true` | `COUNT(*)` | `?$count=true` |
| `$search=term` | Full-text search | `?$search=Harry` |
| `contains(x,'v')` | `LIKE '%v%'` | `$filter=contains(title,'Harry')` |

---

## ✅ Checklist thực hành

- [ ] Test `$filter` với các toán tử: eq, ne, gt, lt, contains, startswith
- [ ] Test `$select` để lấy subset của fields
- [ ] Test `$expand` để load dữ liệu liên quan (author, items)
- [ ] Test phân trang với `$top` và `$skip` + `$count=true`
- [ ] Test `$orderby` với nhiều tiêu chí
- [ ] Kết hợp nhiều query options trong 1 request
- [ ] Dùng file `.http` trong VS Code REST Client để test nhanh

---

## 🔗 Tài liệu tham khảo

- [OData.org — OData v4 Spec](https://www.odata.org/documentation/)
- [CAP — OData Support](https://cap.cloud.sap/docs/guides/providing-services#odata-requests)
- [OData v4 Tutorial (odata.org)](https://www.odata.org/getting-started/basic-tutorial/)
- [Microsoft OData Query Docs](https://docs.microsoft.com/en-us/odata/client/query-options)

---

> **Tóm lại:**  
> OData = REST nhưng chuẩn hóa. Các query options quan trọng nhất:  
> `$filter` (WHERE), `$select` (SELECT columns), `$expand` (JOIN), `$orderby` (ORDER BY), `$top`+`$skip` (LIMIT+OFFSET), `$count` (COUNT).  
> CAP tự gen tất cả, bạn chỉ cần biết cú pháp để query.
