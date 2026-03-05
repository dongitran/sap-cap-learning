# 🚀 MTA Deployment — Deploy CAP App lên SAP BTP

> **MTA (Multi-Target Application)** là packaging format của SAP — tương đương `docker-compose.yml` nhưng cho SAP BTP Cloud Foundry. Đây là cách chuẩn để deploy CAP apps lên production.

---

## 🤔 MTA là gì?

```
Non-SAP world:          SAP world (BTP Cloud Foundry):
─────────────────       ─────────────────────────────────────────
docker-compose.yml  →   mta.yaml          (mô tả các components)
docker build        →   mbt build         (đóng gói thành .mtar)
docker push         →   cf deploy *.mtar  (deploy lên CF)

Docker images       →   .mtar archive     (single file chứa app code)
Docker containers   →   CF apps + services
```

**Tại sao cần MTA?**
- Deploy **nhiều components cùng lúc** (Node.js service + HANA DB + XSUAA + App Router)
- Tự động **bind services** sau khi tạo
- Handle **dependencies** giữa các components
- Single artifact cho devops CI/CD pipeline

---

## 📋 Cấu trúc mta.yaml

```yaml
# mta.yaml — file mô tả toàn bộ MTA application
_schema-version: '3.1'
ID: my-bookshop          # Unique app ID
version: 1.0.0           # App version

# ═══════════════ MODULES ════════════════
# Modules = code components bạn build và deploy
modules:
  # ── 1. Node.js CAP Service ─────────────
  - name: my-bookshop-srv
    type: nodejs                          # Cloud Foundry buildpack
    path: gen/srv                         # Path sau cds build
    parameters:
      buildpack: nodejs_buildpack
      memory: 256M
    requires:
      - name: my-bookshop-db             # Cần HANA database
      - name: my-bookshop-auth           # Cần XSUAA service
      - name: my-bookshop-logs           # Optional: Cloud Logging
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}         # Expose URL cho App Router

  # ── 2. HANA Database Deployer ───────────
  - name: my-bookshop-db-deployer
    type: hdb
    path: gen/db                          # HANA artifacts
    requires:
      - name: my-bookshop-db
    parameters:
      buildpack: nodejs_buildpack

  # ── 3. App Router (Optional UI gateway) ─
  - name: my-bookshop-app
    type: approuter.nodejs
    path: app/
    parameters:
      memory: 128M
    requires:
      - name: my-bookshop-auth
      - name: srv-api                     # Connect tới backend
        group: destinations
        properties:
          name: srv-api
          url: ~{srv-url}
          forwardAuthToken: true

# ═══════════════ RESOURCES ════════════════
# Resources = BTP managed services cần tạo và bind
resources:
  # ── 1. HANA Cloud HDI Container ─────────
  - name: my-bookshop-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  # ── 2. XSUAA Service ────────────────────
  - name: my-bookshop-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json            # Roles và scopes config

  # ── 3. SAP Cloud Logging (Optional) ─────
  - name: my-bookshop-logs
    type: org.cloudfoundry.managed-service
    parameters:
      service: cloud-logging
      service-plan: standard
```

---

## ⚙️ Setup MTA trong CAP Project

### Bước 1: Thêm MTA config

```bash
# Auto-generate mta.yaml từ CDS project
cds add mta

# Thêm approuter (nếu muốn UI)
cds add approuter

# Thêm HANA support
cds add hana

# Thêm XSUAA
cds add xsuaa
```

### Bước 2: Cài MBT build tool

```bash
# Cài mbt globally
npm install -g mbt

# Verify
mbt --version
```

---

## 🔨 Build MTA Archive

```bash
# Bước 1: Build CAP production artifacts
cds build --production
# → Tạo thư mục gen/ với compiled artifacts

# Bước 2: Build MTA archive từ mta.yaml
mbt build
# → Tạo file: mta_archives/my-bookshop_1.0.0.mtar

# Hoặc build trong 1 lệnh:
mbt build -p cf   # target Cloud Foundry

# Kiểm tra archive được tạo
ls -la mta_archives/
# → my-bookshop_1.0.0.mtar (~20-50MB tùy app size)
```

**Quá trình `mbt build` làm gì?**

```
1. Đọc mta.yaml
2. Chạy npm install cho từng module
3. Compile TypeScript (nếu có)
4. Copy artifacts vào temp folder
5. Đóng gói tất cả vào .mtar (ZIP format)
→ .mtar = single deployable artifact
```

---

## 🚀 Deploy lên Cloud Foundry

### Bước 1: Login vào BTP CF

```bash
# Login
cf login -a https://api.cf.eu10.hana.ondemand.com
# Nhập: email, password, chọn org và space

# Verify đang ở đúng space
cf target
# → org: my-org, space: dev
```

### Bước 2: Deploy MTA archive

```bash
# Deploy
cf deploy mta_archives/my-bookshop_1.0.0.mtar

# Với verbose output
cf deploy mta_archives/my-bookshop_1.0.0.mtar -e config.mtaext

# Output:
# Deploying multi-target app archive my-bookshop_1.0.0.mtar in org my-org / space dev...
# Creating service "my-bookshop-db" from plan "hdi-shared"...
# Creating service "my-bookshop-auth" from plan "application"...
# Deploying app "my-bookshop-srv"...
# Deploying app "my-bookshop-db-deployer"...
# ✓ Deployment completed
```

### Bước 3: Verify deployment

```bash
# Xem tất cả apps đang chạy
cf apps
# NAME                     STATE   INSTANCES  MEMORY  DISK  URLS
# my-bookshop-srv          started   1/1       256M    1G    my-bookshop-srv.cfapps.eu10.hana.ondemand.com
# my-bookshop-app          started   1/1       128M    1G    my-bookshop-app.cfapps.eu10.hana.ondemand.com

# Xem services đã tạo
cf services
# NAME                  SERVICE             PLAN
# my-bookshop-db        hana                hdi-shared
# my-bookshop-auth      xsuaa               application

# Xem logs của app
cf logs my-bookshop-srv --recent
```

---

## 🔄 Update và Re-deploy

```bash
# Chỉ update code (không recreate services)
mbt build
cf deploy mta_archives/my-bookshop_1.0.0.mtar --strategy blue-green

# Hoặc dùng incremental deploy
cf deploy mta_archives/my-bookshop_1.0.0.mtar --no-restart

# Xem deployment operations history
cf mta-ops
```

---

## 🌍 MTA Extension descriptor — Config per environment

```yaml
# config/dev.mtaext — override config cho dev environment
_schema-version: '3.1'
ID: my-bookshop-dev
extends: my-bookshop   # Extend từ base mta.yaml

modules:
  - name: my-bookshop-srv
    parameters:
      memory: 128M      # Ít memory hơn cho dev

resources:
  - name: my-bookshop-auth
    parameters:
      service-plan: application   # Giữ nguyên hoặc override
```

```bash
# Deploy với extension
cf deploy mta_archives/my-bookshop_1.0.0.mtar -e config/dev.mtaext
```

---

## 🔁 Blue-Green Deployment (Zero downtime)

```bash
# Deploy với blue-green strategy (không có downtime)
cf deploy mta_archives/my-bookshop_1.0.0.mtar --strategy blue-green

# Quá trình:
# 1. Deploy phiên bản mới (blue) song song với old (green)
# 2. Test blue version (temporary URL)
# 3. Route traffic sang blue (switch)
# 4. Xoá green version cũ
```

---

## 📊 Tóm tắt flow deployment

```
DEV (local SQLite)
        │ code xong
        ↓
cds build --production
        │ gen/ folder được tạo
        ↓
mbt build
        │ my-bookshop.mtar được tạo
        ↓
cf login (BTP account)
        │
        ↓
cf deploy *.mtar
        │ CF tự động:
        │  - Tạo HANA HDI container
        │  - Tạo XSUAA service instance
        │  - Upload app code
        │  - Deploy DB schema → HANA
        │  - Start Node.js app
        │  - Bind tất cả services
        ↓
App chạy trên BTP CF
URL: https://my-app.cfapps.eu10.hana.ondemand.com/
```

---

## ✅ Checklist thực hành

- [ ] `cds add mta && cds add hana && cds add xsuaa` → kiểm tra `mta.yaml` sinh ra
- [ ] `cds build --production` → kiểm tra thư mục `gen/`
- [ ] `npm install -g mbt` → `mbt build` → kiểm tra file `.mtar` trong `mta_archives/`
- [ ] `cf login` → `cf target` → verify đúng org/space
- [ ] `cf deploy my-bookshop_1.0.0.mtar` → đợi deployment xong
- [ ] `cf apps` → verify apps đang `started`
- [ ] `cf services` → verify HANA và XSUAA service đã tạo
- [ ] Truy cập URL app trên browser

---

## 🔗 Tài liệu tham khảo

- [CAP — Deploying to CF](https://cap.cloud.sap/docs/guides/deployment/to-cf)
- [MBT Build Tool](https://github.com/SAP/cloud-mta-build-tool)
- [Cloud MTA Build Tool Docs](https://sap.github.io/cloud-mta-build-tool/)
- [CF Deploy Plugin](https://help.sap.com/docs/btp/sap-business-technology-platform/deploy-content-using-generic-application-content-deployment)

---

> **Tóm lại:**  
> - **MTA** = packaging format của SAP, gom app + services vào 1 archive  
> - **mta.yaml** = docker-compose equivalent, khai báo modules và resources  
> - Flow: `cds build --production` → `mbt build` → `cf deploy *.mtar`  
> - CF tự động tạo HANA container, XSUAA service, bind vào app  
> - **Blue-green deployment** = zero downtime update với `--strategy blue-green`  
> - **Extension descriptors** `.mtaext` = override config per environment
