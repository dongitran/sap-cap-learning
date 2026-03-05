# ☁️ HANA Cloud — Database Thật Trên BTP

> **SAP HANA Cloud** là database production của SAP CAP trên BTP. Khi deploy app lên cloud, bạn sẽ dùng HANA thay vì SQLite. Phần này hướng dẫn setup từ đầu đến cuối.

---

## 🤔 HANA Cloud là gì?

```
SQLite (local dev)          HANA Cloud (BTP production)
──────────────────          ──────────────────────────────────
File nhỏ, local             Managed DB service trên cloud
Chậm với data lớn           Cực nhanh (in-memory columnar DB)
Không cần config            Cần provision và connect
Free, built-in              Có phí (Trial: free 90 ngày)
Single-tenant               Multi-tenant support
No advanced analytics       Advanced analytics, ML built-in
SQLite SQL                  HANA SQL (superset của SQL)

Code CDS: GIỐNG NHAU ← không cần đổi code, chỉ đổi config!
```

**Khi nào dùng HANA Cloud:**
- Deploy lên BTP (staging, production)
- Cần test với data lớn
- Cần HANA-specific features (full-text search mạnh, analytics)
- Multi-tenant apps

---

## 📦 Khái niệm HDI Container

HANA Cloud dùng **HDI (HANA Deployment Infrastructure) Container** — hiểu đơn giản như một "schema riêng" cho mỗi CAP app:

```
HANA Cloud Instance
├── HDI Container (App A)     ← Schema isolated cho App A
│   ├── table: my_bookshop_Books
│   ├── table: my_bookshop_Authors
│   └── ...
├── HDI Container (App B)     ← Schema isolated cho App B
│   └── ...
└── HDI Container (App C)     ← Multi-tenant có nhiều containers
```

**CAP tự tạo và quản lý HDI containers** khi bạn deploy — không cần tự làm thủ công.

---

## 🏗️ Setup BTP Trial Account

### Bước 1: Tạo Trial Account

```
1. Vào: https://cockpit.hanatrial.ondemand.com/
2. Click "Get your free trial account"
3. Điền thông tin, chọn vùng (EU10 hoặc US10)
4. Active email → Login vào BTP Cockpit
```

### Bước 2: Tạo HANA Cloud Instance

Từ BTP Cockpit của Trial account:

```
1. Vào subaccount → "Instances and Subscriptions"
2. Click "Create" → chọn "SAP HANA Cloud" → "SAP HANA Database"
3. Điền:
   - Instance Name: my-hana-db
   - Administrator Password: (tạo password mạnh, nhớ để sau này dùng)
   - Memory: 30 GB (minimum cho trial)
4. Click "Create" → đợi ~10 phút
```

> ⚠️ **Trial HANA tự stop mỗi ngày!** Phải vào BTP Cockpit → HANA Cloud Central → Start lại trước khi dùng.

### Bước 3: Subscribe HANA Cloud Tools

```
BTP Cockpit → Instances and Subscriptions → Create
→ SAP HANA Cloud → SAP HANA Cloud Tools
→ Subscribe
```

```
BTP Cockpit → Security → Users → chọn user của bạn
→ Assign Role Collection:
  - SAP HANA Cloud Administrator
  - SAP HANA Cloud Viewer
```

---

## ⚙️ Thêm HANA vào CAP Project

### Bước 1: Add HANA dependencies

```bash
# Trong thư mục CAP project
cds add hana

# Lệnh này làm gì?
# 1. Thêm @sap/hana-client vào package.json dependencies
# 2. Thêm @cap-js/hana vào package.json dependencies  
# 3. Cập nhật .cdsrc.json hoặc package.json với HANA config
# 4. Tạo .hdbtabledata files (nếu cần)
```

### Bước 2: Kiểm tra config được thêm

```json
// package.json — sau khi cds add hana
{
  "name": "my-cap-app",
  "dependencies": {
    "@sap/cds": "^8",
    "@cap-js/hana": "^2",          ← được thêm tự động
    "@sap/hana-client": "^2"       ← được thêm tự động
  },
  "cds": {
    "requires": {
      "db": {
        "[development]": {
          "kind": "sqlite",
          "credentials": { "url": ":memory:" }
        },
        "[production]": {
          "kind": "hana"              ← prod dùng HANA
        }
      }
    }
  }
}
```

### Bước 3: Deploy schema lên HANA trực tiếp (local dev test)

```bash
# Connect CF (Cloud Foundry)
cf login -a https://api.cf.eu10.hana.ondemand.com
# → Nhập BTP email và password

# Deploy CDS schema → HANA (tạo HDI container và deploy tables)
cds deploy --to hana

# Output:
# Deploying to SAP HANA Database...
# Creating HDI container...
# Deploying schema artifacts...
# ✓ Deployment successful
```

---

## 🔗 Connect App với HANA Cloud

### Cách 1: Deploy lên Cloud Foundry (recommended cho production)

```bash
# Cài thêm tools
npm install -g mbt

# Thêm MTA config
cds add mta
cds add approuter
cds add xsuaa

# Build MTA archive
mbt build

# Deploy lên CF
cf deploy mta_archives/my-cap-app_1.0.0.mtar

# CAP tự:
# - Tạo XSUAA service instance
# - Tạo HDI container trên HANA Cloud
# - Deploy schema artifacts
# - Start app và bind services
```

### Cách 2: Hybrid testing (local app + HANA cloud DB)

```bash
# Bind local project với services trên CF
cds bind --to cf:my-hana-hdi-container
cds bind --to cf:my-xsuaa

# Chạy local nhưng connect cloud DB
cds watch --profile hybrid

# Hoặc set env vars thủ công
export VCAP_SERVICES='{"hana":[{"credentials":{...}}]}'
```

---

## 📊 HANA vs SQLite — Khác biệt về query

Phần lớn code CDS giống nhau, nhưng có một số điểm cần lưu ý:

### String case sensitivity

```javascript
// SQLite: case-insensitive (mặc định)
SELECT.from(Books).where({ title: 'harry potter' })
// → Sẽ tìm thấy 'Harry Potter' trên SQLite, NHƯNG...

// HANA: case-sensitive!
SELECT.from(Books).where({ title: 'harry potter' })
// → Sẽ KHÔNG tìm thấy 'Harry Potter' trên HANA
// → Phải dùng: tolower(title) = 'harry potter'
```

### Full-text search

```bash
# SQLite: basic LIKE search
/catalog/Books?$search=harry
# → WHERE title LIKE '%harry%' (chậm với data lớn)

# HANA: native full-text search (context-aware, stemming, etc.)
/catalog/Books?$search=harry
# → HANA uses optimized text search index (cực nhanh)
```

### Large data performance

```
SQLite: 1 triệu records → query chậm (~seconds)
HANA:   1 triệu records → query nhanh (~milliseconds)
        (do in-memory columnar architecture)
```

---

## 🔍 HANA Cloud Central — Quản lý Instance

```
BTP Cockpit → Instances → SAP HANA Cloud → "Manage SAP HANA Cloud"
→ Opens HANA Cloud Central (web UI)
```

Trong HANA Cloud Central bạn có thể:
- **Start/Stop** instance (trial auto-stops hàng ngày)
- **Database Explorer** — SQL console, browse tables
- **Monitor** memory, CPU usage
- **Create** additional schemas

### Database Explorer — Xem data thật

```
HANA Cloud Central → Database Explorer
→ Connect with admin credentials
→ Browse tables
→ Run SQL queries
```

---

## 🧪 Test HANA-specific features

### @sap/hana-client trong handlers

```javascript
// Truy cập HANA-specific SQL khi cần
const db = await cds.connect.to('db');

// Chạy raw HANA SQL (dùng sparingly)
const result = await db.run(
  `SELECT * FROM "MY_BOOKSHOP_BOOKS" WHERE "STOCK" > 0`
);
```

### HANA Spatial, Series, JSON collections

```cds
// HANA-specific types (chỉ hoạt động trên HANA, không phải SQLite)
entity Locations {
  key ID       : UUID;
  name         : String;
  // geoLocation : hana.ST_POINT;   // Spatial (HANA only)
}
```

---

## 🚨 Lưu ý trial account

| Vấn đề | Chi tiết | Cách xử lý |
|---|---|---|
| Trial hết hạn sau 90 ngày | Cần gia hạn hoặc tạo mới | Đăng nhập ít nhất 1 lần/30 ngày |
| HANA tự stop hàng ngày | Phải restart thủ công | Bookmark HANA Cloud Central |
| CF không available cho trial mới | SAP tạm hạn chế CF cho trial mới | Dùng Business Application Studio, hoặc test local |
| HDI container bị delete | Khi trial expired | Redeploy sau khi tạo trial mới |

---

## ✅ Checklist thực hành

- [ ] Tạo BTP Trial Account tại `cockpit.hanatrial.ondemand.com`
- [ ] Tạo HANA Cloud instance (10 phút, chọn memory 30GB)
- [ ] Subscribe SAP HANA Cloud Tools, assign roles
- [ ] `cds add hana` trong project → verify `package.json` changes
- [ ] `cf login` → `cds deploy --to hana` → verify tables tạo trong HANA
- [ ] Mở HANA Database Explorer → xem tables, chạy SQL query kiểm tra
- [ ] Test `$search` trên HANA Cloud (so sánh với SQLite)
- [ ] Start/stop HANA instance qua HANA Cloud Central

---

## 🔗 Tài liệu tham khảo

- [CAP — HANA Databases](https://cap.cloud.sap/docs/guides/databases-hana)
- [BTP Trial Account](https://cockpit.hanatrial.ondemand.com/)
- [HANA Cloud Getting Started](https://help.sap.com/docs/hana-cloud/sap-hana-cloud-getting-started-guide/getting-started-with-sap-hana-cloud)
- [cds add hana](https://cap.cloud.sap/docs/tools/cds-cli#cds-add)

---

> **Tóm lại:**  
> - HANA Cloud = production DB, SQLite = local dev — code CDS **không đổi**  
> - Setup: Tạo BTP trial → provision HANA instance → `cds add hana` → `cds deploy --to hana`  
> - HDI containers = isolated schema per app, CAP tự quản lý  
> - ⚠️ Trial HANA **tự stop hàng ngày** — phải restart thủ công từ HANA Cloud Central  
> - Khác biệt quan trọng: HANA **case-sensitive** string, full-text search tốt hơn nhiều
