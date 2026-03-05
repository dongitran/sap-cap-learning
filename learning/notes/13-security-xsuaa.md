# 🔐 Security & XSUAA — Bảo Mật CAP Apps

> **XSUAA** (eXtended Services User Account and Authentication) là Auth service của SAP BTP — tương đương Keycloak/Auth0 nhưng native trên SAP ecosystem. Đây là layer bảo mật chính cho mọi CAP app deploy lên BTP.

---

## 🤔 XSUAA hoạt động như thế nào?

```
Luồng xác thực:
────────────────────────────────────────────────────────────────────
1. User login vào XSUAA (IdP)
       ↓
2. XSUAA cấp JWT token (có chứa scopes/roles)
       ↓
3. Client gửi request với token: Authorization: Bearer <JWT>
       ↓
4. CAP service verify token, check scopes
       ↓
5. Nếu đủ quyền → xử lý request | Nếu thiếu → 403 Forbidden
```

**JWT token chứa gì?**

```json
{
  "user_name": "alice",
  "email": "alice@company.com",
  "scope": [
    "my-cap-app.Admin",
    "my-cap-app.Viewer",
    "openid"
  ],
  "exp": 1735689600,
  "tenant": "my-tenant"
}
```

---

## 📦 Khái niệm cốt lõi

| Khái niệm | Mô tả | Ví dụ |
|---|---|---|
| **Scope** | Quyền cụ thể trong app | `my-app.Admin`, `my-app.Viewer` |
| **Role Template** | Container chứa 1 hoặc nhiều scopes | Template "Admin" → scope "my-app.Admin" |
| **Role Collection** | Nhóm các roles, gán cho user | "Bookshop Admin" = Admin + Editor |
| **Identity Provider (IdP)** | Nguồn xác thực | SAP ID, Azure AD, Custom |

```
Hierarchy:
────────────────────────────────────
User
  ↓ assigned
Role Collection (e.g., "Bookshop Admin")
  ↓ contains
Role Templates (e.g., "Admin", "Editor")
  ↓ contain
Scopes (e.g., "my-app.Admin", "my-app.Editor")
  ↓ checked in
CAP Authorization Annotations
```

---

## 🛡️ Định nghĩa Authorization trong CDS

### @requires — Role đơn giản

```cds
// srv/catalog-service.cds
service CatalogService @(requires: 'authenticated-user') {
  // Ai cũng đọc được (nhưng phải logged in)
  @readonly
  entity Books as projection on db.Books;

  // Chỉ user có role 'Viewer' mới đọc được
  @(requires: 'Viewer')
  entity Authors as projection on db.Authors;

  // Admin HOẶC Editor đều được
  @(requires: ['Admin', 'Editor'])
  entity Publishers as projection on db.Publishers;
}
```

### @restrict — Phân quyền chi tiết theo operation

```cds
service OrderService @(requires: 'authenticated-user') {
  // Granular control per operation
  @(restrict: [
    { grant: 'READ' },                                    // Ai cũng READ
    { grant: 'WRITE', to: 'Editor' },                     // Editor: CREATE + UPDATE
    { grant: ['WRITE', 'DELETE'], to: 'Admin' }           // Admin: tất cả
  ])
  entity Orders as projection on db.Orders;

  // Instance-based restriction (chỉ thấy data của mình)
  @(restrict: [
    { grant: 'READ', where: 'createdBy = $user' },        // Chỉ đọc orders của mình
    { grant: 'WRITE', to: 'Admin' }                       // Admin write tất cả
  ])
  entity MyOrders as projection on db.Orders;
}
```

### Kết hợp @requires ở service level + entity level

```cds
// Toàn bộ service chỉ cho authenticated-user
service AdminService @(requires: 'Admin') {
  // Thêm restriction ở entity level
  entity Books   as projection on db.Books;   // Admin: full CRUD
  entity Authors as projection on db.Authors; // Admin: full CRUD

  // Chỉ Admin mới DELETE được Orders
  @(restrict: [
    { grant: ['READ', 'WRITE'], to: 'Admin' },
    { grant: 'DELETE', to: 'SuperAdmin' }
  ])
  entity Orders as projection on db.Orders;
}
```

---

## 📄 xs-security.json — Khai báo Scopes và Roles

```json
// xs-security.json (root của project)
{
  "xsappname": "my-cap-app",
  "tenant-mode": "dedicated",
  "scopes": [
    {
      "name": "$XSAPPNAME.Admin",
      "description": "Full admin access to all resources"
    },
    {
      "name": "$XSAPPNAME.Editor",
      "description": "Can create and update but not delete"
    },
    {
      "name": "$XSAPPNAME.Viewer",
      "description": "Read-only access"
    }
  ],
  "role-templates": [
    {
      "name": "Admin",
      "description": "Administrator role",
      "scope-references": ["$XSAPPNAME.Admin"]
    },
    {
      "name": "Editor",
      "description": "Editor role",
      "scope-references": ["$XSAPPNAME.Editor"]
    },
    {
      "name": "Viewer",
      "description": "Viewer role",
      "scope-references": ["$XSAPPNAME.Viewer"]
    }
  ]
}
```

> **Tự generate xs-security.json từ CDS annotations:**
> ```bash
> cds compile srv/ --to xsuaa > xs-security.json
> ```
> CAP tự đọc `@requires` và `@restrict` → generate file JSON phù hợp.

---

## 🧪 Mock Authentication — Test Local không cần BTP

Khi dev local, dùng mock users thay vì XSUAA thật:

```json
// package.json — thêm vào section "cds"
{
  "cds": {
    "[development]": {
      "auth": {
        "kind": "mocked",
        "users": {
          "alice": {
            "roles": ["Admin"],
            "ID": "alice@company.com"
          },
          "bob": {
            "roles": ["Viewer"],
            "ID": "bob@company.com"
          },
          "carol": {
            "roles": ["Editor"],
            "ID": "carol@company.com"
          },
          "anonymous": {}
        }
      }
    }
  }
}
```

```bash
# Test với mock user (HTTP Basic Auth)
# alice là Admin → có thể DELETE
curl -u alice:alice -X DELETE http://localhost:4004/catalog/Books(1)
# → 204 No Content (success)

# bob là Viewer → không thể DELETE
curl -u bob:bob -X DELETE http://localhost:4004/catalog/Books(1)
# → 403 Forbidden

# Không có user → 401 Unauthorized
curl http://localhost:4004/catalog/Books
# → 401 Unauthorized (nếu service @requires: 'authenticated-user')

# Test với admin service
curl -u alice:alice http://localhost:4004/admin/Books
# → 200 OK

curl -u bob:bob http://localhost:4004/admin/Books
# → 403 Forbidden (bob không có role Admin)
```

---

## 🔍 Dùng req.user trong Handlers

```javascript
// srv/catalog-service.js
module.exports = (srv) => {
  srv.before('CREATE', 'Orders', (req) => {
    // Kiểm tra xem user có đăng nhập không
    if (req.user.is('anonymous')) {
      return req.reject(401, 'Please login to create orders');
    }

    // Lấy thông tin user
    const userId = req.user.id;     // 'alice' hoặc email/UUID
    const locale = req.user.locale; // 'vi-VN', 'en-US', etc.
    const tenant = req.user.tenant; // tenant ID (multi-tenant)

    // Kiểm tra role
    if (!req.user.is('Admin') && !req.user.is('Editor')) {
      return req.reject(403, 'Only Admin or Editor can create orders');
    }

    // Custom logic dựa trên role
    if (req.user.is('Admin')) {
      req.data.priority = 'HIGH';  // Admin orders có priority cao hơn
    }

    // Gắn user ID vào data trước khi INSERT
    req.data.createdBy = userId;
    req.data.createdAt = new Date();
  });

  // Chỉ admin mới xem được cost (field nhạy cảm)
  srv.after('READ', 'Books', (books, req) => {
    if (!req.user.is('Admin')) {
      books.forEach(book => {
        delete book.cost;        // ẩn cost với non-admin
        delete book.margin;      // ẩn margin với non-admin
      });
    }
  });
};
```

---

## ☁️ Connect XSUAA thật trên BTP

### Bước 1: Thêm XSUAA vào project

```bash
cds add xsuaa
```

Lệnh này:
- Thêm `@sap/xssec` và `passport` vào dependencies
- Cập nhật config trong `package.json`
- Tạo hoặc cập nhật `xs-security.json`

### Bước 2: Tạo XSUAA service instance trên BTP Trial

```bash
# Login vào CF
cf login -a https://api.cf.eu10.hana.ondemand.com

# Tạo XSUAA service instance
cf create-service xsuaa application my-xsuaa -c xs-security.json

# Bind vào app (hoặc tự động qua mta.yaml)
cf bind-service my-cap-app my-xsuaa
```

### Bước 3: Hybrid testing (local app + cloud XSUAA)

```bash
# Bind local project với XSUAA instance trên CF
cds bind --to cf:my-xsuaa

# Chạy với profile hybrid (dùng XSUAA thật)
cds watch --profile hybrid
```

### Bước 4: config trong mta.yaml (deployment)

```yaml
# mta.yaml
resources:
  - name: my-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

modules:
  - name: my-cap-app-srv
    type: nodejs
    requires:
      - name: my-xsuaa         # bind XSUAA service
      - name: my-hana-db
```

---

## 📊 Bảng test scenarios

| Test case | User | Operation | Expected |
|---|---|---|---|
| Anonymous read | *(no user)* | GET /catalog/Books | 401 nếu @requires auth |
| Viewer read | bob (Viewer) | GET /catalog/Books | 200 OK |
| Viewer delete | bob (Viewer) | DELETE /catalog/Books(1) | 403 Forbidden |
| Admin delete | alice (Admin) | DELETE /catalog/Books(1) | 204 OK |
| Wrong service | bob (Viewer) | GET /admin/Books | 403 Forbidden |
| Admin all | alice (Admin) | any operation | 2xx OK |

---

## ✅ Checklist thực hành

- [ ] Thêm `@(requires: 'authenticated-user')` vào CatalogService
- [ ] Thêm `@readonly @(requires: 'Viewer')` cho entity Books
- [ ] Thêm `@(requires: 'Admin')` cho AdminService
- [ ] Config mock users (alice=Admin, bob=Viewer) trong `package.json`
- [ ] Test 5 scenarios: anonymous, viewer-read, viewer-delete, admin-delete, wrong-service
- [ ] `cds compile srv/ --to xsuaa > xs-security.json` → kiểm tra file sinh ra
- [ ] Dùng `req.user.is('Admin')` trong after handler để ẩn sensitive fields

---

## 🔗 Tài liệu tham khảo

- [CAP — Authorization](https://cap.cloud.sap/docs/guides/authorization)
- [CAP — Authentication](https://cap.cloud.sap/docs/node.js/authentication)
- [XSUAA on BTP](https://help.sap.com/docs/btp/sap-business-technology-platform/xsuaa-service)
- [Mock Users for Testing](https://cap.cloud.sap/docs/guides/authorization#mock-users)

---

> **Tóm lại:**  
> - XSUAA = Auth0/Keycloak của SAP, cấp JWT token với scopes  
> - `@requires: 'Role'` = role check đơn giản; `@restrict: [...]` = fine-grained  
> - **Local dev**: dùng mock users trong `package.json` (không cần BTP)  
> - **Production**: `cds add xsuaa` → tạo XSUAA instance → bind vào app  
> - Trong handler: `req.user.id`, `req.user.is('Admin')`, `req.user.tenant`  
> - Generate `xs-security.json` tự động: `cds compile srv/ --to xsuaa > xs-security.json`
