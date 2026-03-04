# ⚙️ Setup Môi Trường Dev — SAP CAP

> Hướng dẫn cài đặt môi trường để bắt đầu code SAP CAP trên local machine. Sau khi xong sẽ chạy được `cds watch` và thấy app chạy tại `localhost:4004`.

---

## 📋 Kiểm tra Prerequisites

Trước khi bắt đầu, kiểm tra các tools đã có:

```bash
# Kiểm tra Node.js — cần >= 18 (khuyến nghị LTS 20+)
node -v
# → v20.x.x  ✅ hoặc v18.x.x  ✅

# Kiểm tra npm
npm -v
# → 10.x.x  ✅

# Kiểm tra Git
git --version
# → git version 2.x.x  ✅
```

> **Node.js 20 LTS** là phiên bản được khuyến nghị cho CAP Node.js v9 (2025).  
> Tải tại: https://nodejs.org/

---

## 🛠️ Cài đặt CAP CLI

**`@sap/cds-dk`** là CLI chính để làm mọi thứ với CAP:

```bash
# Cài global
npm install -g @sap/cds-dk

# Verify cài thành công
cds version
# Output mong đợi:
# @sap/cds: 8.x.x
# @sap/cds-dk: 8.x.x
# Node.js: v20.x.x

# Xem tất cả commands có sẵn
cds help
```

> **Lưu ý:** Không cần install `@sap/cds` (runtime) riêng — nó sẽ được tự động thêm vào khi bạn `cds init` project.

---

## 💻 VS Code Extensions

Mở VS Code → Extensions (`Cmd+Shift+X`) → Cài các extensions sau:

### Bắt buộc

| Extension | ID | Tác dụng |
|---|---|---|
| **SAP CDS Language Support** | `SAPSE.vscode-cds` | Syntax highlighting, autocomplete, validation cho `.cds` files |

### Khuyến nghị

| Extension | Tác dụng |
|---|---|
| **REST Client** | Test OData API trực tiếp trong VS Code (thay Postman) |
| **SQLite Viewer** | Xem database SQLite khi dev local |
| **Rainbow CSV** | Highlight CSV files (mock data trong `db/data/`) |
| **Prettier** | Format code tự động |
| **ESLint** | Lint JavaScript |

```bash
# Cài nhanh qua terminal (optional)
code --install-extension SAPSE.vscode-cds
```

---

## 🚀 Tạo Project CAP Đầu Tiên

```bash
# 1. Tạo folder và init project
mkdir my-first-cap
cd my-first-cap
cds init .      # init trong folder hiện tại

# Hoặc tạo luôn với 1 lệnh:
cds init my-first-cap
cd my-first-cap
```

### Cấu trúc sau khi `cds init`:

```
my-first-cap/
├── db/                   # Data layer — để đây
├── srv/                  # Service layer — để đây
├── app/                  # UI layer — để đây
├── package.json          # Dependencies
├── .cdsrc.json           # CAP config (thường để mặc định)
└── README.md
```

```bash
# 2. Cài dependencies
npm install

# 3. Chạy dev server
cds watch
```

### Output khi `cds watch` chạy thành công:

```
cds serve all --with-mocks --in-memory?
...

[cds] - model loaded from 0 model files

[cds] - connect to db > sqlite { database: ':memory:' }
[cds] - serving ... at http://localhost:4004

[cds] - launched in: 347ms
[cds] - server listening on { url: 'http://localhost:4004' }
```

> Mở browser vào `http://localhost:4004` sẽ thấy giao diện CAP Launchpad.

---

## 🔄 cds watch — Dev Server

`cds watch` là lệnh bạn dùng hàng ngày khi dev:

```
cds watch làm những gì?
─────────────────────────────────────────────────────
1. Đọc tất cả file .cds trong db/ và srv/
2. Compile → tạo database schema (SQLite in-memory)
3. Generate OData endpoints tự động
4. Start Express server tại localhost:4004
5. Theo dõi thay đổi → tự restart khi bạn sửa file
```

**So sánh với `npm run dev`:**

```
npm run dev (thông thường)    cds watch
─────────────────────          ──────────────────────
Chỉ restart Node process      Còn reload CDS model
Không gen DB schema           Tự động gen/update schema
Không gen endpoints           Tự động gen OData endpoints
```

**Một số lệnh cds hay dùng:**

```bash
cds watch              # Dev server với hot reload
cds serve              # Chạy production (không watch)
cds compile db/        # Compile CDS sang CSN/SQL (debug)
cds deploy             # Deploy schema lên DB
cds add hana           # Thêm HANA support
cds add xsuaa          # Thêm XSUAA auth
cds add mta            # Thêm MTA deployment config
cds env                # Xem config hiện tại
cds repl               # REPL để test CDS queries
cds help               # Xem tất cả commands
```

---

## 🧪 Thêm Entity và Test Ngay

Sau khi server đang chạy, thêm entity đầu tiên để kiểm tra setup:

**Bước 1:** Tạo file `db/schema.cds`:

```cds
namespace my.bookshop;

entity Books {
  key ID     : Integer;
  title      : String(100);
  stock      : Integer;
}
```

**Bước 2:** Tạo file `srv/catalog-service.cds`:

```cds
using my.bookshop as db from '../db/schema';

service CatalogService {
  entity Books as projection on db.Books;
}
```

**Bước 3:** Quan sát terminal — `cds watch` tự reload:

```
[cds] - model loaded from 2 model files

[cds] - connect to db > sqlite { database: ':memory:' }
[cds] - serving CatalogService { at: '/catalog', impl: 'srv/...' }

✔ Endpoints:
   GET /catalog/Books
   GET /catalog/Books(ID)
   ...
```

**Bước 4:** Test ngay trên browser:

```
http://localhost:4004/catalog/Books
→ Trả về JSON OData (danh sách Books trống)

http://localhost:4004/catalog/$metadata
→ Trả về OData metadata XML
```

---

## 🗃️ Thêm Mock Data

Tạo file `db/data/my.bookshop-Books.csv`:

```
# Tên file format: <namespace>-<EntityName>.csv
ID,title,stock
1,The Great Gatsby,10
2,To Kill a Mockingbird,5
3,1984,20
```

`cds watch` tự load CSV khi restart → gọi lại API sẽ có data.

---

## 🔍 Kiểm tra Cài đặt

Checklist để xác nhận setup đúng:

```bash
# Tất cả lệnh này phải chạy không lỗi
node -v           # >= 18.x
npm -v            # >= 9.x
cds version       # hiển thị @sap/cds version
cds watch         # server chạy tại localhost:4004
```

Trên browser:
- [ ] `localhost:4004` → thấy CAP Launchpad
- [ ] `localhost:4004/catalog/Books` → trả JSON
- [ ] Sửa file `.cds` → terminal tự reload

---

## 🐛 Lỗi thường gặp

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `cds: command not found` | Chưa cài cds-dk | `npm install -g @sap/cds-dk` |
| `Error: Cannot find module '@sap/cds'` | Chưa npm install | `npm install` trong project folder |
| Port 4004 đang dùng | App khác chiếm port | `cds watch --port 4005` |
| Model load error | Lỗi syntax trong `.cds` file | Xem highlight lỗi trong VS Code |

---

## 📋 Checklist — Việc cần làm

- [ ] Cài Node.js LTS 20
- [ ] `npm install -g @sap/cds-dk` → verify `cds version`
- [ ] Cài VS Code extension **SAP CDS Language Support**
- [ ] `cds init my-first-cap` → `cds watch` → thấy localhost:4004
- [ ] Thêm entity Book, service, CSV data → test API trên browser

---

## 🔗 Tài liệu tham khảo

- [CAP Getting Started Guide](https://cap.cloud.sap/docs/get-started/)
- [SAP CDS Language Support (VS Code)](https://marketplace.visualstudio.com/items?itemName=SAPSE.vscode-cds)
- [CAP Samples — Bookshop](https://github.com/SAP-samples/cloud-cap-samples)

---

> **Tóm lại:** Chỉ cần 3 lệnh để bắt đầu: `npm install -g @sap/cds-dk` → `cds init myapp` → `cds watch`. SAP CAP cực kỳ developer-friendly ở môi trường local.
