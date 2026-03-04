# 🚀 SAP CAP Learning Roadmap

> **Mục tiêu:** Từ zero đến production-ready CAP developer  
> **Phong cách học:** Hands-on, thực hành từng bước, kết hợp lý thuyết + code

---

## 📌 Tổng Quan — SAP CAP là gì?

SAP CAP = **Framework backend** để build enterprise apps trên **SAP BTP**:

- Viết bằng **Node.js**
- Có **Express.js** dưới hood
- Framework tự generate CRUD/OData — viết ít code hơn
- Kiến trúc **event-driven** (srv.on/before/after)

### So sánh nhanh với các framework khác

| Khía cạnh | Framework thông thường | SAP CAP |
|---|---|---|
| Kiến trúc | Module, DI, Controllers | Model-driven + CDS files |
| Data model | ORM decorators | CDS `.cds` file (declarative) |
| API | REST tự define từng endpoint | OData **auto-generated** từ CDS |
| Handler | Controller methods | `srv.on('READ', 'Entity', ...)` |
| Middleware | Middleware / Guards | `srv.before('*', ...)` |
| Auth | JWT Guards | XSUAA + `@requires`/`@restrict` |
| Database | PostgreSQL, MySQL | SQLite (dev) → SAP HANA Cloud (prod) |
| Deploy | Docker, K8s | Cloud Foundry / Kyma trên SAP BTP |
| CLI scaffold | Framework CLI | `cds init`, `cds add` |
| Dev server | `npm run dev` | `cds watch` (hot reload) |

### Mapping khái niệm

```
Khái niệm chung                 →  SAP CAP
───────────────────────────────────────────────────────
Database Entity/Table           →  entity trong .cds file
Column / Field                  →  element: Type trong CDS
Primary Key (auto)              →  key ID : Integer / UUID
Controller / Router             →  service CatalogService { entity Books }
GET / POST endpoints            →  auto-generated OData endpoints
Business logic service          →  .js handler file
Auth guard / middleware         →  @requires 'RoleName' trong CDS
Before/After hooks              →  srv.before('*', ...) / srv.after(...)
DTO / View model                →  CDS type / aspect
Express bootstrap               →  cds.on('bootstrap', app => {...})
```

---

## 🗺️ Lộ trình 4 Giai đoạn

1. **Giai đoạn 1** — Nền tảng + Setup
2. **Giai đoạn 2** — Hands-on Development
3. **Giai đoạn 3** — Deep Dive + BTP Integration
4. **Giai đoạn 4** — Production + Deployment

---

## 🔷 Giai Đoạn 1: Nền Tảng & Setup

### Mục tiêu
- Hiểu SAP BTP ecosystem
- Setup môi trường dev local
- Chạy được app CAP đầu tiên với `cds watch`

---

### 📚 1.1 — Hiểu SAP BTP

**SAP BTP là gì?**  
SAP BTP (Business Technology Platform) = cloud platform của SAP để build, run, manage enterprise apps.

Các thành phần chính cần biết:

| Thành phần | Tương đương | Mô tả |
|---|---|---|
| **Cloud Foundry** | Heroku / AWS Elastic Beanstalk | Runtime để deploy apps |
| **Kyma** | Kubernetes | Container-based runtime |
| **SAP HANA Cloud** | PostgreSQL on RDS | Database chính của SAP |
| **SAP BAS** | GitHub Codespaces | IDE trực tuyến (VS Code-based) |
| **XSUAA** | Auth0 / Keycloak | Auth & Authorization service |
| **Destination** | API Gateway config | Kết nối external systems |
| **Event Mesh** | Kafka / RabbitMQ | Messaging service |

**Việc cần làm:**
- [ ] Đọc: [Discovering SAP BTP](https://learning.sap.com/learning-journeys/discover-sap-business-technology-platform) (FREE)
- [ ] Tạo **BTP Trial Account** miễn phí tại: https://cockpit.hanatrial.ondemand.com/
- [ ] Xem video: [SAP BTP Overview for Developers](https://www.youtube.com/results?search_query=SAP+BTP+overview+developers+2024)

---

### ⚙️ 1.2 — Setup Môi Trường Dev

```bash
# Bạn đã có: Node.js LTS, VS Code, Git, npm

# 1. Cài CAP CLI
npm install -g @sap/cds-dk

# 2. Verify cài đặt thành công
cds version
# Output: @sap/cds: 8.x.x, @sap/cds-dk: 8.x.x

# 3. Cài VS Code Extensions:
# - "SAP CDS Language Support" (by SAP) — syntax highlighting, autocomplete .cds files
# - "SAP Fiori Tools - Extension Pack" (optional, for Fiori UI)

# 4. Cài Cloud Foundry CLI (cần khi deploy)
# brew install cloudfoundry/tap/cf-cli@8
```

**Thử nghiệm ngay:**
```bash
# Tạo project đầu tiên
cds init my-first-cap
cd my-first-cap
npm install

# Xem cấu trúc project
ls -la
# db/       → Data models (entities)
# srv/      → Services (API definitions)
# app/      → UI (Fiori apps) — tương đương frontend
# package.json, .cdsrc.json

# Chạy dev server (hot reload)
cds watch
# → Server chạy tại http://localhost:4004
```

---

### 📖 1.3 — Đọc Docs Cốt Lõi

Bookmark và đọc theo thứ tự:
1. [Getting Started — CAP](https://cap.cloud.sap/docs/get-started/) — Bắt đầu từ đây
2. [Architecture Overview](https://cap.cloud.sap/docs/about/) — Hiểu big picture
3. [CDS — Domain Modeling](https://cap.cloud.sap/docs/cds/cdl) — Ngôn ngữ CDS

---

### 📝 1.4 — Khóa học "Introduction to SAP CAP"

- **Link:** https://learning.sap.com/courses/introduction-to-sap-cloud-application-programming-model
- **Giá:** FREE
- **Nội dung:** CDS basics, project structure, domain modeling, services

**Sau giai đoạn 1, bạn phải có:**
- ✅ BTP Trial Account đã tạo
- ✅ `cds watch` chạy được local
- ✅ Hiểu cấu trúc `db/`, `srv/`, `app/`
- ✅ Biết CDS là gì và viết entity đơn giản

---

## 🔷 Giai Đoạn 2: Hands-on Development

### Mục tiêu
- Build được CAP app hoàn chỉnh (CRUD, custom logic, UI)
- Hiểu sâu CDS + OData
- Connect được SQLite ở local, HANA Cloud ở BTP

---

### 🛠️ 2.1 — Tutorial 1: Bookshop App

**Đây là "Hello World" chính thức của SAP CAP.**

```bash
# Clone sample project của SAP (như create-react-app nhưng dùng để học)
git clone https://github.com/SAP-samples/cloud-cap-samples
cd cloud-cap-samples
npm install
cds watch bookshop
# → Mở http://localhost:4004 → xem auto-generated OData endpoints
```

**Tự build từng bước theo docs:**

**Bước 1: Tạo data model**
```cds
// db/schema.cds
namespace my.bookshop;

entity Books {
  key ID     : Integer;
  title      : String(111);
  author     : Association to Authors;  // Giống @ManyToOne trong TypeORM
  stock      : Integer;
  price      : Decimal(9,2);
}

entity Authors {
  key ID     : Integer;
  name       : String(111);
  books      : Association to many Books on books.author = $self;  // @OneToMany
}
```

**Bước 2: Expose service**
```cds
// srv/catalog-service.cds
using my.bookshop as db from '../db/schema';

service CatalogService @(path: '/catalog') {
  // Projection — giống DTO/View pattern
  @readonly entity Books as projection on db.Books;
  entity Authors as projection on db.Authors;
}
```

> Chỉ với 2 file .cds này, CAP tự tạo:
> - `GET /catalog/Books` → list all books
> - `GET /catalog/Books(1)` → get by ID  
> - `POST /catalog/Books` → create
> - `PATCH /catalog/Books(1)` → update
> - `DELETE /catalog/Books(1)` → delete
> - Và tương tự cho Authors

**Bước 3: Thêm mock data**
```csv
# db/data/my.bookshop-Books.csv
ID,title,stock,price,author_ID
1,Wuthering Heights,100,11.11,101
2,Jane Eyre,500,12.34,102
```

**Bước 4: Custom business logic**
```javascript
// srv/catalog-service.js
module.exports = (srv) => {
  // BEFORE handler — tương đương @BeforeCreate() interceptor trong TypeORM
  srv.before('CREATE', 'Books', async (req) => {
    if (!req.data.title) req.error(400, 'Title is required');
  });

  // ON handler — override default READ behavior
  srv.on('READ', 'Books', async (req, next) => {
    console.log('Books are being read!');
    return next(); // tiếp tục default behavior
  });

  // AFTER handler — transform kết quả sau khi đọc
  srv.after('READ', 'Books', (books) => {
    books.forEach(book => {
      if (book.stock < 10) book.title += ' (Low Stock!)';
    });
  });
};
```

**Thực hành:**
- [ ] Build Bookshop app từ đầu (không clone, tự code)
- [ ] Test OData requests bằng Postman/curl:
  ```bash
  curl http://localhost:4004/catalog/Books
  curl http://localhost:4004/catalog/Books?$filter=stock gt 100
  curl http://localhost:4004/catalog/Books?$select=ID,title&$top=5
  ```
- [ ] Đọc: [SAP Tutorial — Create CAP Business Service](https://developers.sap.com/tutorials/cp-apm-nodejs-create-service.html)

---

### 🔍 2.2 — Hiểu OData

OData là protocol **thay thế REST** trong SAP ecosystem. Nó giống REST nhưng có query language chuẩn hóa.

**OData Query Options — cần biết:**

| OData Query | Tương đương SQL | Ví dụ |
|---|---|---|
| `$filter` | WHERE | `?$filter=price lt 20` |
| `$select` | SELECT columns | `?$select=ID,title` |
| `$expand` | JOIN | `?$expand=author` |
| `$orderby` | ORDER BY | `?$orderby=price desc` |
| `$top` | LIMIT | `?$top=10` |
| `$skip` | OFFSET | `?$skip=20` |
| `$count` | COUNT(*) | `?$count=true` |
| `$search` | Full-text search | `?$search=Wuthering` |

**Thực hành:**
```bash
# Test các OData queries với bookshop app
curl "http://localhost:4004/catalog/Books?\$filter=stock gt 100&\$select=ID,title,stock"
curl "http://localhost:4004/catalog/Books?\$expand=author&\$top=5"
curl "http://localhost:4004/catalog/Books?\$orderby=price desc&\$top=3"
```

---

### 💾 2.3 — Database: SQLite → HANA Cloud

```bash
# Dev mode: SQLite (auto, không cần config)
cds watch  # Dùng in-memory SQLite

# Deploy schema lên SQLite file (persistent)
cds deploy --to sqlite:db.sqlite

# Connect SQLite DB viewer
# Cài VS Code extension: "SQLite Viewer"
```

**Khi deploy lên BTP sẽ dùng HANA Cloud** — syntax CDS giống nhau, CAP tự adapt.

---

### 🎨 2.4 — Fiori UI (tùy chọn)

SAP Fiori Elements là UI framework tự generate từ CDS annotations.

```cds
// srv/catalog-service.cds — thêm annotations
annotate CatalogService.Books with @(
  UI: {
    LineItem: [
      { Value: title, Label: 'Title' },
      { Value: stock, Label: 'Stock' },
      { Value: price, Label: 'Price' }
    ],
    SelectionFields: [title, author_ID]
  }
);
```

```bash
# Thêm Fiori UI vào project
cds add approuter
npm install
cds watch
# → Mở http://localhost:4004/app (Fiori UI tự generate!)
```

**Thực hành:**
- [ ] Follow tutorial: [Build a Business Application Using CAP for Node.js](https://developers.sap.com/group/cap-application-nodejs.html)
- [ ] Deploy app lên BTP Free Tier để xem kết quả thật

**Sau giai đoạn 2, bạn phải có:**
- ✅ Build được CRUD app hoàn chỉnh với CDS
- ✅ Biết cách viết custom handlers (before/on/after)
- ✅ Hiểu OData query options
- ✅ Deploy được lên BTP (basic)

---

## 🔷 Giai Đoạn 3: Deep Dive + BTP Integration

### Mục tiêu
- Master CDS language (associations, compositions, aspects, projections)
- Implement security với XSUAA
- Connect external SAP services
- Testing CAP apps

---

### 🧩 3.1 — CDS Advanced

**Associations vs Compositions — quan trọng nhất:**

```cds
// Associations → foreign key reference (quan hệ tham chiếu)
entity Orders {
  key ID     : UUID;
  customer   : Association to Customers;  // FK: customer_ID
  status     : String;
}

// Compositions → quan hệ owns-children, cascade delete
entity Orders {
  key ID     : UUID;
  items      : Composition of many OrderItems on items.order = $self;
}

entity OrderItems {
  key ID     : UUID;
  order      : Association to Orders;
  product    : String;
  quantity   : Integer;
}
```

**Aspects — tương tự mixins/abstract classes:**

```cds
// Aspect chứa các trường dùng chung (tương tự BaseEntity pattern)
aspect managed {
  createdAt  : Timestamp @cds.on.insert: $now;
  createdBy  : String    @cds.on.insert: $user;
  modifiedAt : Timestamp @cds.on.insert: $now  @cds.on.update: $now;
  modifiedBy : String    @cds.on.insert: $user @cds.on.update: $user;
}

// SAP đã có built-in: cuid, managed, temporal
using { cuid, managed } from '@sap/cds/common';

entity Books : cuid, managed {  // Kế thừa ID (UUID), createdAt, modifiedAt...
  title : String;
}
```

**Projections — giống DTO transforms:**

```cds
service AdminService {
  // Expose tất cả fields (admin)
  entity Books as projection on db.Books;
}

service CatalogService {
  // Chỉ expose một số fields (public)
  entity Books as select from db.Books {
    ID, title, price  // chỉ lấy 3 fields
  } where stock > 0;  // chỉ hiển thị books còn hàng
}
```

**Actions & Functions — custom endpoints:**

```cds
service CatalogService {
  entity Books as projection on db.Books;

  // Action = POST request với body (thay đổi state)
  action  submitOrder(book : Books:ID, quantity : Integer) returns String;

  // Function = GET request (chỉ đọc)
  function getTopBooks(maxCount : Integer) returns array of Books;
}
```

```javascript
// srv/catalog-service.js — implement actions
srv.on('submitOrder', async (req) => {
  const { book, quantity } = req.data;
  const [b] = await SELECT.from(Books).where({ ID: book });
  if (b.stock < quantity) return req.error(409, 'Insufficient stock');
  await UPDATE(Books, book).with({ stock: b.stock - quantity });
  return `Order placed: ${quantity} copies of "${b.title}"`;
});
```

**CQL — CAP Query Language:**

```javascript
// CQL — CAP Query Language, tương tự SQL nhưng declarative
const books = await SELECT.from(Books)
  .where({ stock: { '>': 100 } })
  .orderBy('price desc')
  .limit(10);

// Tương đương SQL:
// SELECT * FROM Books WHERE stock > 100 ORDER BY price DESC LIMIT 10

// INSERT
await INSERT.into(Books).entries({ ID: 1, title: 'New Book', stock: 50 });

// UPDATE
await UPDATE(Books, 1).with({ stock: 45 });

// DELETE
await DELETE.from(Books).where({ ID: 1 });
```

**Thực hành:**
- [ ] Build **Orders management system**: Orders → OrderItems → Products (dùng Compositions)
- [ ] Implement ít nhất 2 Actions và 1 Function
- [ ] Dùng `managed` aspect cho tất cả entities
- [ ] Viết CQL queries phức tạp với JOIN, filter, sort

---

### 🔐 3.2 — Security với XSUAA

XSUAA là Auth service của SAP (tương đương Keycloak/Auth0 nhưng native trên BTP).

**1. Định nghĩa roles trong CDS:**

```cds
// srv/catalog-service.cds
service CatalogService @(
  requires: 'authenticated-user'  // Phải login
) {
  @readonly @(requires: 'Viewer')     // Chỉ user có role Viewer mới READ được
  entity Books as projection on db.Books;

  @(requires: ['Admin', 'Editor'])    // Admin HOẶC Editor
  entity Authors as projection on db.Authors;

  @(restrict: [                        // Fine-grained control
    { grant: 'READ' },                 // Ai cũng READ được
    { grant: ['WRITE', 'DELETE'], to: 'Admin' }  // Chỉ Admin mới WRITE/DELETE
  ])
  entity Orders as projection on db.Orders;
}
```

**2. Cấu hình XSUAA:**

```json
// xs-security.json
{
  "xsappname": "my-cap-app",
  "scopes": [
    { "name": "$XSAPPNAME.Admin", "description": "Admin access" },
    { "name": "$XSAPPNAME.Viewer", "description": "Read-only access" }
  ],
  "role-templates": [
    { "name": "Admin", "scope-references": ["$XSAPPNAME.Admin"] },
    { "name": "Viewer", "scope-references": ["$XSAPPNAME.Viewer"] }
  ]
}
```

**3. Test local với mock auth:**

```json
// .cdsrc.json hoặc package.json
{
  "cds": {
    "[development]": {
      "auth": {
        "kind": "mocked",
        "users": {
          "alice": { "roles": ["Admin"] },
          "bob":   { "roles": ["Viewer"] }
        }
      }
    }
  }
}
```

```bash
# Test với mock user
curl -u alice:alice http://localhost:4004/catalog/Books    # OK (Admin)
curl -u bob:bob http://localhost:4004/catalog/Orders -X DELETE  # ERROR 403
```

**Thực hành:**
- [ ] Thêm authentication vào bookshop (cả Admin và Viewer roles)
- [ ] Test các scenarios: unauthorized, authorized, wrong role
- [ ] Connect XSUAA thật trên BTP Trial

---

### 🔗 3.3 — External Services & Integration

**Consume SAP Business Hub APIs:**

```bash
# Ví dụ: consume API từ S/4HANA Cloud (Business Partner API)
cds import API_BUSINESS_PARTNER.edmx --from api.sap.com
```

```javascript
// srv/external-service.js
const bupa = await cds.connect.to('API_BUSINESS_PARTNER');

srv.on('READ', 'Customers', async (req) => {
  // Delegate request tới external service
  return await bupa.run(req.query);
});
```

**Messaging với SAP Event Mesh (pub/sub):**

```javascript
// Publish event
const messaging = await cds.connect.to('messaging');
await messaging.emit('my.topic', { data: 'payload' });

// Subscribe to event
messaging.on('your.topic', async (msg) => {
  console.log('Received:', msg.data);
});
```

**Thực hành:**
- [ ] Kết nối một external API (dùng mock nếu không có S/4HANA)
- [ ] Implement basic pub/sub messaging

---

### 🧪 3.4 — Testing CAP Apps

CAP có built-in test utilities, tương tự `supertest`.

```javascript
// test/catalog-service.test.js
const cds = require('@sap/cds');
const { GET, POST, expect } = cds.test('.'); // Start CAP test server

describe('Catalog Service', () => {
  it('GET /catalog/Books → returns books', async () => {
    const { data } = await GET('/catalog/Books');
    expect(data.value).to.have.length.above(0);
  });

  it('POST /catalog/Books → creates book', async () => {
    const { status } = await POST('/catalog/Books', {
      ID: 999, title: 'Test Book', stock: 10
    });
    expect(status).to.equal(201);
  });

  it('submitOrder action → reduces stock', async () => {
    const { data } = await POST('/catalog/submitOrder', {
      book: 1, quantity: 5
    });
    expect(data.value).to.include('Order placed');
  });
});
```

```bash
# Chạy tests
npm test
# hoặc
cds test
```

**Thực hành:**
- [ ] Viết unit tests cho tất cả handlers (before/on/after)
- [ ] Viết integration tests cho OData endpoints
- [ ] Test authentication scenarios

---

## 🔷 Giai Đoạn 4: Production + Deploy

### Mục tiêu
- Deploy lên SAP BTP Cloud Foundry
- Connect SAP HANA Cloud
- Multi-target application (MTA)
- Monitoring & Logging

---

### 🚀 4.1 — Deploy lên BTP Cloud Foundry

```bash
# 1. Login vào BTP
cf login -a https://api.cf.us10.hana.ondemand.com
# → Nhập email, password BTP

# 2. Build production
cds build --production

# 3. Deploy lên Cloud Foundry
cf push my-cap-app
```

**MTA (Multi-Target Application) — packaging format của SAP:**

```yaml
# mta.yaml — tương đương docker-compose.yml nhưng cho SAP BTP
_schema-version: '3.1'
ID: my-bookshop
version: 1.0.0

modules:
  - name: my-bookshop-srv         # Node.js backend
    type: nodejs
    path: gen/srv
    requires:
      - name: my-bookshop-db      # Kết nối HANA database
      - name: my-bookshop-auth    # Kết nối XSUAA

resources:
  - name: my-bookshop-db          # HANA Cloud instance
    type: com.sap.xs.hdi-container
    
  - name: my-bookshop-auth        # XSUAA service instance
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      config:
        xsappname: my-bookshop
```

```bash
# Build và deploy toàn bộ MTA
npm install -g mbt
mbt build
cf deploy mta_archives/my-bookshop_1.0.0.mtar
```

---

### 🗄️ 4.2 — SAP HANA Cloud

HANA Cloud = SAP's enterprise database (tương đương PostgreSQL nhưng for SAP).

```bash
# Cấu hình deploy lên HANA trong production
cds deploy --to hana  # CAP tự generate HANA-specific SQL

# Kiểm tra kết nối HANA
cds deploy --to hana --dry  # Xem SQL sẽ được chạy mà không chạy thật
```

**CAP tự xử lý sự khác biệt giữa SQLite (dev) và HANA (prod)** — bạn không cần thay đổi CDS models.

---

### 📊 4.3 — Logging & Observability

```javascript
// CAP có built-in logging
const cds = require('@sap/cds');
const LOG = cds.log('my-service');

module.exports = (srv) => {
  srv.on('READ', 'Books', (req) => {
    LOG.info('Books read by user:', req.user.id);
    LOG.debug('Query:', req.query);
  });
};
```

```bash
# Xem logs trên BTP
cf logs my-cap-app --recent
cf logs my-cap-app  # Tail logs live
```

---

### 🎯 4.4 — CAP Plugins & Extensibility

```bash
# Thêm các plugin phổ biến
cds add hana          # HANA support
cds add approuter     # App Router (authentication gateway)
cds add mta           # MTA deployment config
cds add xsuaa         # XSUAA auth service
cds add audit-log     # SAP Audit Log service
cds add notifications # SAP Alert Notification service
```

---

### 📐 4.5 — SAP Discovery Center Missions

Làm các missions hands-on trên BTP — đây là cách học thực tế nhất:

- [ ] **Mission:** "Develop a Full-Stack CAP Application Following BTP Developer's Guide"
  - https://discovery-center.cloud.sap/missions
  - End-to-end: CDS → deploy → HANA → XSUAA → Fiori

- [ ] **Mission:** "Build Extensions with SAP Build Process Automation"
  - Kết hợp CAP với SAP Build tools

---

---

## 🛠️ Checklist Dự Án Thực Hành

### Mini Project 1 — Bookshop
- [ ] CRUD cho Books, Authors
- [ ] Custom handler: validate stock trước khi order
- [ ] OData query: lọc books theo giá, tác giả
- [ ] Mock data từ CSV

### Mini Project 2 — Order Management System
- [ ] Entities: Customers, Orders, OrderItems, Products
- [ ] Compositions: Order chứa OrderItems  
- [ ] Actions: submitOrder, cancelOrder
- [ ] Functions: getOrderHistory(customerId)
- [ ] Roles: Customer (READ own orders), Admin (ALL)
- [ ] Sử dụng `managed` aspect

### Mini Project 3 — Full-Stack Enterprise App
- [ ] Tất cả features từ project 1 + 2
- [ ] XSUAA authentication thật (BTP)
- [ ] Deploy MTA lên Cloud Foundry
- [ ] HANA Cloud database
- [ ] Fiori UI
- [ ] Tests coverage > 70%

---

## 🔗 Tài Nguyên Thiết Yếu

### Docs Chính Thức (Bookmark hết!)
| Tài liệu | URL |
|---|---|
| CAP Documentation (chính) | https://cap.cloud.sap/docs/ |
| CDS Language Reference | https://cap.cloud.sap/docs/cds/cdl |
| CDS Built-in Types | https://cap.cloud.sap/docs/cds/types |
| CAP Node.js API | https://cap.cloud.sap/docs/node.js/ |
| OData APIs in CAP | https://cap.cloud.sap/docs/advanced/odata |
| Security Guide | https://cap.cloud.sap/docs/node.js/authentication |

### Tutorials (Theo thứ tự nên làm)
| Tên | URL |
|---|---|
| Create a CAP Business Service | https://developers.sap.com/tutorials/cp-apm-nodejs-create-service.html |
| Build Business App (Group) | https://developers.sap.com/group/cap-application-nodejs.html |
| Intro to App Dev with CAP | https://developers.sap.com/tutorials/btp-app-introduction.html |

### Videos
| Kênh | Link |
|---|---|
| SAP Developers (official) | https://www.youtube.com/@SAPDevs |
| DJ Adams — "Back to Basics CAP" | Search "DJ Adams CAP back to basics" on YouTube |
| reCAP Conference Talks | Search "reCAP conference CAP nodejs" on YouTube |

### Code Samples (GitHub)
| Repo | URL |
|---|---|
| cloud-cap-samples (official) | https://github.com/SAP-samples/cloud-cap-samples |
| CAP local dev workshop | https://github.com/SAP-samples/cap-local-development-workshop |
| CAP CodeJam exercises | https://github.com/SAP-samples/cloud-cap-exercises |

### Community
| Platform | URL |
|---|---|
| SAP Community (Q&A) | https://community.sap.com/ |
| SAP Discovery Center | https://discovery-center.cloud.sap/ |
| Reddit r/SAP | https://www.reddit.com/r/SAP/ |

---

## ⚡ Tips khi học SAP CAP

1. **Đừng cố viết REST API** — OData là protocol chính, embrace nó
2. **CDS là ngôn ngữ quan trọng nhất** — Dành nhiều thời gian cho `.cds` files hơn `.js` files
3. **CAP tự generate nhiều thứ** — Đừng overcode; nếu thấy mình viết nhiều JS cho CRUD, có thể bạn đang làm sai
4. **`cds watch` là best friend** — Hot reload, show OData endpoints, mock data
5. **Test với Postman/curl trước khi code UI** — OData endpoints có thể test trực tiếp
6. **SQLite trước, HANA sau** — Develop local với SQLite, chỉ deploy HANA khi cần
7. **Đọc official docs** — SAP CAP docs rất tốt và well-written
8. **Đừng học Fiori sâu ngay** — Tập trung backend (CDS + services) trước

---

## 🎓 Certification Path (Optional nhưng có giá trị)

```
Bước 1 → Discovering SAP BTP          (Badge FREE, ~2h)
            ↓
Bước 2 → C_CPE_2409                    (SAP BTP Extension Developer)
         SAP Certified Development Associate
            ↓
Bước 3 → C_CPE_16                      (Backend Developer SAP CAP, advanced)
         SAP Certified Associate
```

> **Lưu ý:** SAP certs có giá ~$500/lần thi, hiệu lực 1 năm, cần renewal.  
> Nên học xong Giai đoạn 3-4 mới thi C_CPE_2409.

---

*Tài liệu này được tổng hợp từ: SAP Official Documentation, SAP Learning Portal, SAP Community, GitHub Samples.*

*Cập nhật: Tháng 3/2026*
