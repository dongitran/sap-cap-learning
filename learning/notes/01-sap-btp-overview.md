# 🏢 SAP BTP — Tổng Quan Nền Tảng

> SAP BTP (Business Technology Platform) là **nền tảng cloud** của SAP để bạn build, deploy và quản lý enterprise apps. Đây là "sân chơi" mà mọi thứ bạn làm với SAP CAP sẽ chạy trên đó.

---

## 🗺️ Bức tranh toàn cảnh

Hãy tưởng tượng SAP BTP như một **tòa nhà văn phòng cao tầng**:

```
┌─────────────────────────────────────────────────────┐
│                   SAP BTP                           │
│                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  Tầng App   │  │  Tầng Data   │  │ Tầng AI   │  │
│  │ CAP, Fiori  │  │ HANA Cloud   │  │ AI Core   │  │
│  └─────────────┘  └──────────────┘  └───────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │           Tầng Runtime                      │    │
│  │    Cloud Foundry    |    Kyma (K8s)         │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌──────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  XSUAA   │  │ Destination │  │  Event Mesh  │   │
│  │  (Auth)  │  │ (Kết nối)   │  │  (Messaging) │   │
│  └──────────┘  └─────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────┘
           │          │          │
    ☁️ AWS   ☁️ Azure   ☁️ GCP   ☁️ Alibaba
```

SAP BTP chạy **multi-cloud** — tức là bạn có thể deploy app trên AWS, Azure, Google Cloud tùy ý, SAP BTP là lớp ở trên cùng.

---

## 🏃 Hai Runtime chính — nơi app của bạn chạy

### Cloud Foundry

Hãy nghĩ **Cloud Foundry** như **Heroku** — bạn chỉ cần `cf push` là xong, platform tự lo hết phần còn lại.

```
Bạn viết code → cf push → Cloud Foundry tự:
  ├─ Build container
  ├─ Scale app
  ├─ Manage networking
  └─ Handle health checks
```

**Dùng khi:** Build CAP app, Fiori apps, REST APIs — đây là **runtime mặc định** khi học SAP CAP.

### Kyma

**Kyma** = Kubernetes được SAP manage sẵn. Bạn có toàn quyền kiểm soát, nhưng cần biết Docker/K8s.

```
Cloud Foundry                    Kyma
─────────────────                ──────────────────
Đơn giản, nhanh                  Phức tạp, linh hoạt
cf push                          kubectl apply -f
SAP lo infrastructure            Bạn tự config nhiều hơn
✅ Học CAP ban đầu               ✅ Production phức tạp
```

> **Kết luận:** Học CAP → dùng **Cloud Foundry** trước. Kyma để sau khi nắm vững.

---

## 🗄️ SAP HANA Cloud — Database chính

**HANA Cloud** là database của SAP, tương đương PostgreSQL nhưng được tối ưu cho enterprise data.

```
Development (local)    →  SQLite  (miễn phí, tự động)
Production (BTP)       →  SAP HANA Cloud  (paid, enterprise)
```

**Điều hay:** CAP tự xử lý sự khác biệt — bạn viết CDS model một lần, chạy được cả 2 database. Không cần đổi code.

```bash
# Local dev — dùng SQLite tự động
cds watch

# Deploy lên BTP — dùng HANA
cds deploy --to hana
```

---

## 🔐 XSUAA — Người bảo vệ cửa

**XSUAA** (eXtended Services User Account and Authentication) = hệ thống **auth** của SAP BTP.

Hãy hình dung như **bảo vệ tòa nhà**:

```
Người dùng muốn vào app
        ↓
   [XSUAA kiểm tra]
   "Bạn là ai?" (Authentication)
   "Bạn được làm gì?" (Authorization)
        ↓
   ✅ Được vào — cấp JWT token
   ❌ Không được — 401/403
```

**Cách hoạt động:**
- Dùng **OAuth 2.0** + **JWT tokens** (tiêu chuẩn industry)
- Bạn define **roles** trong file `xs-security.json`
- Trong CDS, dùng `@requires` và `@restrict` để gắn role vào entity
- Local dev: dùng **mock auth** (không cần XSUAA thật)

```json
// xs-security.json — định nghĩa ai được làm gì
{
  "xsappname": "my-app",
  "scopes": [
    { "name": "$XSAPPNAME.Admin" },
    { "name": "$XSAPPNAME.Viewer" }
  ]
}
```

---

## 🔌 Destination Service — Sổ địa chỉ

**Destination Service** = nơi lưu thông tin kết nối tới **hệ thống bên ngoài**.

```
❌ Cách sai (hardcode credentials trong code):
const url = "https://s4hana.company.com";
const username = "admin";
const password = "secret123";  // 💀 nguy hiểm!

✅ Cách đúng (dùng Destination):
const dest = await cds.connect.to('S4HANA_SYSTEM');
// → BTP tự lấy URL, credentials từ Destination config
// → Không có secret nào trong code
```

**Dùng khi:** App CAP cần gọi sang SAP S/4HANA, Salesforce, hay bất kỳ external API nào.

---

## 📨 Event Mesh — Bưu điện trung tâm

**Event Mesh** = hệ thống **messaging async** của SAP BTP, tương tự Kafka hoặc RabbitMQ.

```
Cách giao tiếp trực tiếp (sync):
App A ──────────────────→ App B
      "Tôi cần dữ liệu"
      ← "Đây!"

Cách dùng Event Mesh (async):
App A → [Event Mesh] → App B
  "Có đơn hàng mới!"   "Tôi đã nhận, sẽ xử lý sau"
  
→ App A không cần chờ App B xong mới tiếp tục
→ Nếu App B chết → message vẫn còn đó, xử lý sau
```

**Dùng khi:** Cần notify hệ thống khác về sự kiện (Order Created, Payment Done...) mà không muốn ràng buộc chặt chẽ giữa các service.

---

## 🛠️ SAP Business Application Studio (BAS)

**BAS** = VS Code chạy trên cloud của SAP.

```
Ưu điểm:
✅ Không cần cài gì trên máy local
✅ Đã có sẵn CAP tools, CDS extension
✅ Tích hợp trực tiếp với BTP
✅ Miễn phí với BTP Trial

Nhược điểm:
❌ Cần internet
❌ Chậm hơn VS Code local đôi khi
```

> **Lời khuyên:** Dùng **VS Code local** + cài `SAP CDS Language Support` extension. Nhanh hơn và quen thuộc hơn.

---

## 🌐 BTP Cockpit — Bảng điều khiển

**BTP Cockpit** (https://cockpit.hanatrial.ondemand.com) = giao diện web để quản lý toàn bộ BTP:

```
BTP Cockpit
├── Global Account (tài khoản tổng)
│   └── Subaccount (môi trường con, vd: dev, prod)
│       ├── Cloud Foundry Space (nơi deploy app)
│       ├── Service Instances (XSUAA, HANA, Event Mesh...)
│       └── Destinations (kết nối external)
```

---

## 📋 Checklist — Việc cần làm ngay

- [ ] Tạo **BTP Trial Account** miễn phí: https://cockpit.hanatrial.ondemand.com/
  - Chọn region gần nhất (Singapore nếu ở VN)
  - Email thật để verify
- [ ] Vào Cockpit → khám phá giao diện
- [ ] Đọc: [Discovering SAP BTP](https://learning.sap.com/learning-journeys/discover-sap-business-technology-platform) (FREE, ~1-2h)

---

## 🔗 Tài liệu tham khảo

- [SAP BTP Official Docs](https://help.sap.com/docs/btp)
- [BTP Trial Account](https://cockpit.hanatrial.ondemand.com/)
- [SAP BTP Overview (YouTube)](https://www.youtube.com/@SAPDevs)
- [What is SAP BTP? (SAP Press)](https://www.sap-press.com/sap-business-technology-platform_6193/)

---

> **Tóm lại:** SAP BTP là "đám mây" của SAP. Với tư cách developer học CAP, bạn chủ yếu cần biết: **Cloud Foundry** (để deploy), **HANA Cloud** (database), **XSUAA** (auth). Event Mesh và Destination học sau khi nắm vững cơ bản.
