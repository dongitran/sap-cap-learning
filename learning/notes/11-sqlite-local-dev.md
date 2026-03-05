# 🗃️ SQLite — Local Development Database

> **SQLite** là database mặc định cho local dev trong CAP. Không cần install, không cần config — chỉ `cds watch` là có DB. Nhưng bạn cần hiểu rõ để dùng đúng cách.

---

## 🤔 Tại sao dùng SQLite cho local dev?

```
Dev workflow với CAP:
────────────────────────────────────────────────────────────────
Local (bạn đang đây) → BTP Staging → BTP Production
        SQLite              HANA Cloud      HANA Cloud
        ↑ nhẹ, nhanh        ↑ giống prod    ↑ thật

CAP tự động adapt code CDS → hoạt động đúng trên cả 2 loại DB!
Không cần sửa code khi switch SQLite → HANA.
```

**SQLite trong CAP có 2 chế độ:**

| Chế độ | Cách dùng | Dữ liệu mất khi restart? |
|---|---|---|
| **In-memory** | `cds watch` mặc định | ✅ Mất (mỗi lần restart load lại CSV) |
| **File-based** | `cds deploy --to sqlite` | ❌ Không mất, lưu vào file `.sqlite` |

---

## 🚀 In-Memory SQLite (Mặc định)

```bash
# Chỉ cần chạy
cds watch
```

**Điều này tự động:**
1. Compile các file `.cds` trong `db/`
2. Tạo SQLite database **trong RAM** (`:memory:`)
3. Deploy schema (CREATE TABLE)
4. Load CSV files từ `db/data/` làm seed data
5. Start HTTP server tại `localhost:4004`

```
[Output trong terminal khi cds watch:]
[cds] - connect to db > sqlite { database: ':memory:' }
                                              ↑ in-memory
[cds] - model loaded from 2 model files
[cds] - serving CatalogService { at: '/catalog' }
```

**Ưu điểm:** Nhanh, sạch — mỗi restart là fresh database có CSV data.  
**Nhược điểm:** Tạo data qua API sẽ mất khi restart.

---

## 💾 File-Based SQLite (Persistent)

Dùng khi muốn **giữ data** giữa các lần restart.

### Bước 1: Deploy schema vào file SQLite

```bash
# Tạo file db.sqlite với schema
cds deploy --to sqlite:db.sqlite
# Hoặc ngắn hơn:
cds deploy --to sqlite

# Output:
# Deployed to ./db.sqlite
```

### Bước 2: Config `package.json` để dùng file SQLite

```json
{
  "cds": {
    "requires": {
      "db": {
        "kind": "sqlite",
        "credentials": {
          "url": "db.sqlite"
        }
      }
    }
  }
}
```

### Bước 3: Chạy server — giờ dùng file DB

```bash
cds watch
# [cds] - connect to db > sqlite { database: 'db.sqlite' }
#                                              ↑ file-based!
```

Giờ khi bạn POST data qua API, data sẽ được lưu vào `db.sqlite` và **không mất** khi restart.

---

## 🔄 In-memory vs File-based — Khi nào dùng?

| Tình huống | Dùng chế độ nào |
|---|---|
| Dev thông thường, test nhanh | In-memory (mặc định) |
| Cần giữ data test giữa các session | File-based |
| Demo cho khách hàng | File-based |
| Testing (unit tests, CI) | In-memory (clean state) |
| Trước khi chuyển sang HANA | Thử file-based để detect issues |

---

## 🌱 CSV Seed Data

CSV files được load khi database khởi tạo (in-memory mode luôn load, file-based load lần đầu hoặc khi `cds deploy`):

```
db/
└── data/
    ├── my.bookshop-Books.csv       ← namespace-EntityName.csv
    ├── my.bookshop-Authors.csv
    └── my.bookshop-Orders.csv
```

```csv
# my.bookshop-Books.csv
ID,title,stock,price,author_ID
"7e2f2640-6866-4dcf-8f4d-3027aa831cad","The Great Gatsby",10,12.99,"101-auth-uuid"
"d85e2d41-e2a9-4dff-ac47-fc7b07b9c9c1","1984",5,9.99,"102-auth-uuid"

# my.bookshop-Authors.csv
ID,name
"101-auth-uuid","F. Scott Fitzgerald"
"102-auth-uuid","George Orwell"
```

> **Thứ tự load CSV:** CAP load CSV theo thứ tự dependency — Authors trước, Books sau (vì Books có FK `author_ID`).  
> **Naming convention bắt buộc:** `<namespace>-<EntityName>.csv` — phải chính xác.

---

## 👁️ Xem Database với SQLite Viewer

### VS Code Extension: SQLite Viewer

1. Cài extension: **SQLite Viewer** (tác giả: Florian Klampfer)
2. Mở file `db.sqlite` trong VS Code
3. Browse tables, xem data thực tế trong DB

```bash
# Tạo file DB trước để xem
cds deploy --to sqlite:db.sqlite
```

### CLI: sqlite3 command

```bash
# Mở SQLite shell
sqlite3 db.sqlite

# Các lệnh trong sqlite3 shell
.tables                    # list all tables
.schema Books              # xem schema của table Books
SELECT * FROM my_bookshop_Books LIMIT 5;   # query data
.quit                      # exit
```

> **Lưu ý:** Tên table trong SQLite = namespace dùng dấu `_` thay dấu `.`  
> Ví dụ: namespace `my.bookshop` + entity `Books` → table `my_bookshop_Books`

### Xem generated SQL

```bash
# Xem CAP generate SQL gì từ CDS model
cds compile db/ --to sql

# Output dạng:
CREATE TABLE my_bookshop_Books (
  ID NVARCHAR(36) NOT NULL,
  title NVARCHAR(111),
  stock INTEGER,
  price DECIMAL(9, 2),
  author_ID NVARCHAR(36),
  PRIMARY KEY (ID)
);

CREATE INDEX my_bookshop_Books_author_ID 
  ON my_bookshop_Books (author_ID);
```

---

## 🔄 Reset Database

```bash
# Reset in-memory: chỉ cần restart cds watch
Ctrl+C → cds watch   # fresh DB với CSV data

# Reset file-based: xóa file và deploy lại
rm db.sqlite
cds deploy --to sqlite:db.sqlite
cds watch
```

---

## ⚙️ Cấu hình nâng cao

### Config theo environment

```json
// package.json — dev dùng in-memory, prod dùng HANA
{
  "cds": {
    "requires": {
      "db": {
        "[development]": {
          "kind": "sqlite",
          "credentials": { "url": ":memory:" }
        },
        "[production]": {
          "kind": "hana"
        }
      }
    }
  }
}
```

### `.cdsrc.json` — Config file riêng

```json
// .cdsrc.json
{
  "requires": {
    "db": {
      "kind": "sqlite",
      "credentials": {
        "url": "db.sqlite"
      }
    }
  }
}
```

---

## 📊 SQLite vs HANA Cloud — Khác biệt cần biết

| Feature | SQLite | HANA Cloud |
|---|---|---|
| Full-text search `$search` | Basic | Advanced |
| Large data sets | Chậm | Rất nhanh |
| Multi-tenant | ❌ | ✅ |
| Column store | ❌ | ✅ |
| UUID generation | Manual trong CSV | `cuid` auto |
| JSON columns | Limited | Native support |
| Case sensitivity | Insensitive | Sensitive |

> **Quan trọng:** Nếu code của bạn có case-sensitive string comparison, sẽ hoạt động khác nhau trên SQLite vs HANA. Test kỹ trước khi deploy.

---

## 🐛 Lỗi thường gặp

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `SQLITE_CANTOPEN` | File db.sqlite không tồn tại | `cds deploy --to sqlite:db.sqlite` |
| CSV không load | Tên file sai format | Kiểm tra `namespace-EntityName.csv` |
| Dữ liệu mất sau restart | Dùng in-memory | Switch sang file-based mode |
| Table not found | Schema chưa deploy | `cds deploy --to sqlite` |
| FK violation khi load CSV | CSV load không đúng thứ tự | Kiểm tra thứ tự, đảm bảo parent CSV load trước |

---

## ✅ Checklist thực hành

- [ ] Chạy `cds watch` → verify SQLite in-memory hoạt động
- [ ] Tạo CSV data cho entities, verify load đúng qua GET API
- [ ] `cds deploy --to sqlite:db.sqlite` → mở file với SQLite Viewer
- [ ] `cds compile db/ --to sql` → đọc SQL được generate
- [ ] Config `package.json` để switch dev/prod database
- [ ] POST data qua API trong file-based mode → restart → verify data vẫn còn

---

## 🔗 Tài liệu tham khảo

- [CAP — SQLite](https://cap.cloud.sap/docs/guides/databases-sqlite)
- [cds deploy](https://cap.cloud.sap/docs/tools/cds-cli#cds-deploy)
- [SQLite Viewer (VS Code)](https://marketplace.visualstudio.com/items?itemName=qwtel.sqlite-viewer)

---

> **Tóm lại:**  
> - **In-memory** (mặc định): nhanh, data reset mỗi restart, load CSV tự động  
> - **File-based**: `cds deploy --to sqlite:db.sqlite` + config `package.json` → data persistent  
> - Xem data với **SQLite Viewer** (VS Code extension) hoặc `cds compile db/ --to sql`  
> - CDS code **không đổi** khi switch SQLite → HANA — database-agnostic!
