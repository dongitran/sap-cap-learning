# 🔌 CAP Plugins & Extensibility — Mở Rộng CAP App

> CAP được thiết kế **extensible hoàn toàn** qua hệ thống plugins. Từ `cds add hana` đến audit logging, notifications, Fiori elements — tất cả đều là plugins. File này hướng dẫn các plugins phổ biến nhất và cách extend CAP.

---

## 🧩 CAP Plugin là gì?

```
Plugin = npm package cung cấp thêm capabilities cho CAP
─────────────────────────────────────────────────────────
Không dùng plugin:            Dùng plugin:
─────────────────             ─────────────
Tự viết audit logic  →   @cap-js/audit-logging (tự động ghi audit)
Tự write HANA SQL    →   @cap-js/hana (CAP tự xử lý)
Tự viết notification →   @cap-js/notifications (gửi via SAP Alert)

Plugin lifecycle: install → declare trong CDS → configure → done!
```

**2 loại plugins:**
- **Technical plugins**: hana, sqlite, audit-logging, notifications
- **Business plugins**: SAP AFC plugin, custom business domain plugins

---

## 🛠️ cds add — Thêm plugins nhanh

```bash
# Database plugins
cds add hana                # SAP HANA Cloud support
cds add sqlite              # SQLite for local dev (default)
cds add postgres            # PostgreSQL (alternative)

# Security & Auth
cds add xsuaa               # XSUAA authentication service
cds add approuter           # App Router (authentication gateway)

# Deployment
cds add mta                 # MTA deployment config (mta.yaml)
cds add cf-manifest         # Cloud Foundry manifest.yml

# Observability
cds add cloud-logging       # SAP Cloud Logging integration
cds add audit-log           # SAP Audit Log service

# UI
cds add app-frontend        # Application Frontend service integration

# Messaging
cds add messaging           # SAP Event Mesh messaging

# Multitenancy
cds add multitenancy        # MTX multi-tenant extensions

# Data
cds add sample              # Add sample data to project
```

> Mỗi lệnh `cds add` thường làm:
> 1. Cài npm package cần thiết
> 2. Cập nhật `package.json` (config)
> 3. Tạo/cập nhật `mta.yaml`
> 4. Tạo files config nếu cần

---

## 🏗️ App Router — Authentication Gateway

App Router là **reverse proxy** đứng trước CAP service, handle authentication redirect:

```
User Browser → App Router → XSUAA (login) → JWT token
                   ↓
              CAP Backend (token verified)
```

```bash
cds add approuter
```

**Cấu trúc sau khi thêm:**

```
my-app/
├── app/                          ← App Router
│   ├── package.json
│   ├── xs-app.json               ← Routing rules
│   └── resources/                ← Static files (optional)
├── srv/                          ← CAP Service
└── mta.yaml                      ← Cập nhật tự động
```

```json
// app/xs-app.json — routing rules
{
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/catalog/(.*)",
      "destination": "srv-api",          // Forward tới CAP backend
      "authenticationType": "xsuaa"      // Yêu cầu XSUAA auth
    },
    {
      "source": "^/index.html$",
      "localDir": "resources/",
      "authenticationType": "xsuaa"
    }
  ]
}
```

---

## 📋 Audit Logging — Ghi lại ai làm gì

Tự động ghi lại các thao tác nhạy cảm cho compliance:

```bash
cds add audit-log
```

```cds
// Thêm @PersonalData annotation vào entities
entity Customers {
  key ID   : UUID;
  @PersonalData.IsPotentiallyPersonal
  name     : String;
  @PersonalData.IsPotentiallyPersonal
  email    : String;
  @PersonalData.IsSensitive
  creditCard : String;    // ← sẽ được che trong logs
}
```

```javascript
// Audit log tự động khi có @PersonalData annotation
// Kết quả trong SAP Audit Log service:
// {
//   "user": "alice@company.com",
//   "tenant": "my-tenant",
//   "time": "2026-03-05T14:00:00Z",
//   "action": "Read",
//   "attributes": [{ "name": "email", "old": "****", "new": "alice@co.com" }]
// }
```

**Custom audit logging:**

```javascript
// srv/order-service.js
const audit = await cds.connect.to('audit-log');

module.exports = (srv) => {
  srv.after('DELETE', 'Orders', async (order, req) => {
    await audit.log('OrderDeleted', {
      user: req.user.id,
      action: 'Delete',
      object: { type: 'Order', id: order.ID },
      attributes: [{ name: 'status', old: order.status, new: 'DELETED' }]
    });
  });
};
```

---

## 🔔 Notifications — SAP Alert Notification

Gửi notifications qua email, Slack, Microsoft Teams:

```bash
cds add notifications
```

```javascript
// srv/order-service.js
const notifications = await cds.connect.to('notifications');

module.exports = (srv) => {
  srv.after('CREATE', 'Orders', async (order, req) => {
    // Gửi notification khi có đơn hàng mới
    await notifications.notify({
      NotificationTypeKey: 'OrderCreated',
      NotificationTypeVersion: '1',
      Priority: 'NEUTRAL',       // LOW, NEUTRAL, MEDIUM, HIGH
      Properties: [
        { Key: 'orderId', IsSensitive: false, Language: 'en', Value: order.ID },
        { Key: 'amount', IsSensitive: false, Language: 'en', Value: `$${order.totalAmount}` }
      ],
      Recipients: [
        { RecipientId: 'admin@company.com' }
      ]
    });
  });
};
```

---

## 🎨 Fiori Elements — Auto-Generate UI từ CDS

```bash
# Thêm Fiori Elements app vào project
cds add app --name bookshop-ui

# Hoặc dùng Fiori generator trong VS Code / BAS
```

```cds
// srv/catalog-service.cds — thêm UI annotations
@title: 'Books'
annotate CatalogService.Books with @(
  UI: {
    // List view — hiển thị như table
    LineItem: [
      { Value: title,  Label: 'Title' },
      { Value: author_ID, Label: 'Author' },
      { Value: stock,  Label: 'In Stock' },
      { Value: price,  Label: 'Price', Criticality: stockCriticality }
    ],

    // Header của detail view
    HeaderInfo: {
      TypeName: 'Book',
      TypeNamePlural: 'Books',
      Title: { Value: title }
    },

    // Form trong detail view
    FieldGroup#Main: {
      Label: 'Book Details',
      Data: [
        { Value: title },
        { Value: author_ID },
        { Value: price },
        { Value: stock }
      ]
    },

    // Quick filter fields
    SelectionFields: [title, author_ID, price]
  }
);

// Virtual field cho color coding
extend CatalogService.Books with {
  stockCriticality : Integer @Core.Computed;
}
```

```javascript
// srv/catalog-service.js — compute virtual fields
srv.after('READ', 'Books', (books) => {
  books.forEach(book => {
    book.stockCriticality = book.stock < 10 ? 1    // Red
                          : book.stock < 50 ? 2    // Yellow
                          : 3;                      // Green
  });
});
```

---

## 🔧 Viết Custom Plugin (Advanced)

```javascript
// plugins/my-validator/index.js
const cds = require('@sap/cds');

// Plugin = function được gọi khi CAP load
module.exports = (srv, options) => {
  const { maxStock = 10000 } = options || {};

  // Plugin thêm validation tự động cho TẤT CẢ services
  cds.on('served', (services) => {
    services.forEach(service => {
      // Hook vào tất cả CREATE events
      service.before('CREATE', '*', (req) => {
        if (req.data.stock && req.data.stock > maxStock) {
          req.error(400, `Stock cannot exceed ${maxStock}`);
        }
      });
    });
  });
};
```

```json
// package.json — đăng ký plugin
{
  "cds": {
    "plugins": [
      "./plugins/my-validator"       // Load plugin local
    ],
    "my-validator": {
      "maxStock": 5000               // Plugin config
    }
  }
}
```

---

## 📦 Tổng hợp: Ecosystem Plugins phổ biến

| Plugin | Package | Dùng để |
|---|---|---|
| **HANA** | `@cap-js/hana` | Production database |
| **SQLite** | `@cap-js/sqlite` | Local dev database |
| **Audit Log** | `@cap-js/audit-logging` | Compliance logging |
| **Notifications** | `@cap-js/notifications` | Alert Notification service |
| **Messaging** | `@cap-js/messaging` | Event Mesh integration |
| **OpenTelemetry** | `@cap-js/opentelemetry` | Distributed tracing |
| **Multitenancy** | `@cap-js/mtxs` | Multi-tenant apps |
| **Attachments** | `@cap-js/attachments` | File attachments (Object Store) |

---

## ✅ Checklist thực hành

- [ ] `cds add approuter` → kiểm tra `app/xs-app.json` → deploy và test auth flow
- [ ] `cds add audit-log` → thêm `@PersonalData` annotation → verify audit logs
- [ ] Thêm `@UI.LineItem` annotation → chạy Fiori preview trong BAS/VS Code
- [ ] Thêm color coding với `stockCriticality` virtual field
- [ ] Viết custom plugin đơn giản (ví dụ: auto-trim strings)
- [ ] `cds add notifications` → test gửi notification khi tạo order

---

## 🔗 Tài liệu tham khảo

- [CAP Plugins](https://cap.cloud.sap/docs/plugins/)
- [Fiori Elements for CAP](https://cap.cloud.sap/docs/advanced/fiori)
- [CAP Audit Logging](https://cap.cloud.sap/docs/guides/data-privacy/audit-logging)
- [App Router Documentation](https://help.sap.com/docs/btp/sap-business-technology-platform/application-router)
- [Fiori Flexible Programming Model](https://ui5.sap.com/fiori-elements)

---

> **Tóm lại:**  
> - `cds add <plugin>` = cách nhanh nhất để thêm bất kỳ capability  
> - **App Router** = reverse proxy/auth gateway → đứng trước CAP, redirect login  
> - **Audit Log** = dùng `@PersonalData` annotation → CAP auto-log mọi access  
> - **Notifications** = gửi alerts qua email/Teams/Slack từ event handlers  
> - **Fiori Elements** = `@UI.LineItem`, `@UI.HeaderInfo` → auto-gen UI từ annotations  
> - **Custom plugin** = npm package + load qua `cds.plugins` trong `package.json`
