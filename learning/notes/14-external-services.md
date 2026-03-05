# 🔗 External Services — Kết Nối SAP và Third-party APIs

> CAP không chỉ là backend framework — nó còn là **integration hub** để kết nối app của bạn với S/4HANA, SAP Business Hub, và bất kỳ external API nào. Tất cả thông qua `cds.connect.to()`.

---

## 🤔 Tại sao cần External Services?

```
Thực tế enterprise apps:
─────────────────────────────────────────────────────────────────
                    CAP App (bạn build)
                         │
               ┌─────────┼──────────┐
               │         │          │
          S/4HANA    Business    Custom
          (Business   Partner    REST API
          Partner API) API       (weather,
                                  payment, etc.)

→ CAP xử lý kết nối, retry, error handling
→ Bạn chỉ cần viết business logic
```

---

## 📋 Các loại External Services

| Loại | Ví dụ | Cách kết nối |
|---|---|---|
| **SAP OData v4** | S/4HANA Cloud APIs | `cds import *.edmx` |
| **SAP OData v2** | SAP ECC APIs | `cds import *.edmx --as odata-v2` |
| **REST API** | Third-party APIs | Tự viết adapter |
| **SAP Event Mesh** | Messaging | `cds connect.to('messaging')` |
| **CAP to CAP** | Microservices | CDS service definition |

---

## 🗂️ Bước 1: Tìm API trên SAP Business Accelerator Hub

```
1. Vào: https://api.sap.com (SAP Business Accelerator Hub)
2. Tìm kiếm API (ví dụ: "Business Partner", "Sales Order")
3. Chọn API → "API Specification" → Download EDMX file
4. Tải về: API_BUSINESS_PARTNER.edmx
```

---

## 📥 Bước 2: Import API vào CAP Project

```bash
# Import EDMX → CAP generate CDS model
cds import srv/external/API_BUSINESS_PARTNER.edmx

# Output:
# Creating srv/external/API_BUSINESS_PARTNER.csn
# Updating package.json

# Hoặc import trực tiếp từ URL (nếu có):
cds import API_BUSINESS_PARTNER.edmx --as odata-v4
```

**Cấu trúc sau khi import:**

```
my-cap-app/
├── db/
├── srv/
│   ├── external/
│   │   ├── API_BUSINESS_PARTNER.csn     ← CDS model của external service
│   │   └── API_BUSINESS_PARTNER.edmx    ← Original EDMX (để reference)
│   └── catalog-service.cds
└── package.json       ← Đã được cập nhật tự động
```

**package.json sau import:**

```json
{
  "cds": {
    "requires": {
      "API_BUSINESS_PARTNER": {
        "kind": "odata-v4",
        "model": "srv/external/API_BUSINESS_PARTNER"
      }
    }
  }
}
```

---

## 🔌 Bước 3: Kết nối và Dùng External Service

### Cách 1: Delegate request (pass-through)

```javascript
// srv/catalog-service.js
module.exports = async (srv) => {
  // Connect tới external service (lazy — chỉ connect khi dùng)
  const bupa = await cds.connect.to('API_BUSINESS_PARTNER');
  const { A_BusinessPartner } = bupa.entities;

  // Khi client đọc Customers, delegate sang Business Partner API
  srv.on('READ', 'Customers', async (req) => {
    // Pass request trực tiếp tới external service
    return await bupa.run(req.query);
  });
};
```

### Cách 2: Mash-up — Kết hợp dữ liệu từ nhiều nguồn

```javascript
module.exports = async (srv) => {
  const bupa = await cds.connect.to('API_BUSINESS_PARTNER');
  const { A_BusinessPartner } = bupa.entities;

  srv.on('READ', 'EnrichedOrders', async (req) => {
    // 1. Lấy orders từ DB local
    const { Orders } = cds.entities('my.shop');
    const orders = await SELECT.from(Orders).where(req._params);

    // 2. Lấy thêm customer info từ S/4HANA cho mỗi order
    for (const order of orders) {
      const [customer] = await bupa.run(
        SELECT.from(A_BusinessPartner)
          .where({ BusinessPartner: order.customerId })
          .columns('BusinessPartnerName', 'Country', 'City')
      );
      order.customerName = customer?.BusinessPartnerName;
      order.customerCity = customer?.City;
    }

    return orders;
  });
};
```

### Cách 3: Write về external system

```javascript
module.exports = async (srv) => {
  const bupa = await cds.connect.to('API_BUSINESS_PARTNER');

  srv.on('CREATE', 'BusinessPartners', async (req) => {
    // Tạo record trực tiếp trong S/4HANA
    const result = await bupa.run(
      INSERT.into(bupa.entities.A_BusinessPartner)
        .entries(req.data)
    );
    return result;
  });
};
```

---

## 🎭 Mock External Service (Local Dev)

Khi dev local, không có S/4HANA thật → dùng mock:

### Tự viết mock data

```json
// srv/external/data/API_BUSINESS_PARTNER-A_BusinessPartner.json
[
  {
    "BusinessPartner": "1001",
    "BusinessPartnerName": "Acme Corp",
    "Country": "VN",
    "City": "Hanoi"
  },
  {
    "BusinessPartner": "1002",
    "BusinessPartnerName": "Tech Solutions",
    "Country": "US",
    "City": "New York"
  }
]
```

```json
// package.json — config sandbox mode cho development
{
  "cds": {
    "requires": {
      "API_BUSINESS_PARTNER": {
        "[development]": {
          "kind": "odata-v4",
          "credentials": {
            "url": "https://sandbox.api.sap.com/s4hanacloud",
            "headers": {
              "APIKey": "your-api-key-from-api.sap.com"
            }
          }
        }
      }
    }
  }
}
```

### CDS test fixture (mock responses)

```javascript
// test/external-service.test.js
const cds = require('@sap/cds');

// Mock external service
cds.test('.').in(__dirname, '..');
const bupa = cds.connect.to('API_BUSINESS_PARTNER');

// Intercept requests và return mock data
bupa.on('*', () => ([
  { BusinessPartner: '1001', BusinessPartnerName: 'Test Corp' }
]));
```

---

## 🔑 Credentials & Destination Service

Trong production, credentials được quản lý qua **SAP Destination Service**:

```bash
# Tạo Destination trên BTP Cockpit
# Connectivity → Destinations → New Destination
# Name: API_BUSINESS_PARTNER
# URL: https://my-s4hana.s4hana.cloud.sap/sap/opu/odata/sap/
# Authentication: OAuth2SAMLBearerAssertion (hoặc BasicAuthentication)
```

```json
// package.json production config
{
  "cds": {
    "requires": {
      "API_BUSINESS_PARTNER": {
        "[production]": {
          "kind": "odata-v4",
          "credentials": {
            "destination": "API_BUSINESS_PARTNER",  // tên destination trên BTP
            "path": "/sap/opu/odata/sap/API_BUSINESS_PARTNER"
          }
        }
      }
    }
  }
}
```

---

## 🌐 Kết nối REST API thông thường (Non-OData)

CAP không chỉ support OData — bạn có thể kết nối bất kỳ REST API:

```javascript
// srv/weather-service.js
const cds = require('@sap/cds');

module.exports = async (srv) => {
  srv.on('getWeather', async (req) => {
    const { city } = req.data;

    // Dùng fetch hoặc axios để gọi external REST API
    const response = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${process.env.WEATHER_API_KEY}`
    );
    const data = await response.json();

    return {
      city: data.name,
      temperature: data.main.temp,
      description: data.weather[0].description
    };
  });
};
```

---

## ⚡ Xử lý lỗi từ External Services

```javascript
module.exports = async (srv) => {
  const bupa = await cds.connect.to('API_BUSINESS_PARTNER');

  srv.on('READ', 'Customers', async (req) => {
    try {
      return await bupa.run(req.query);
    } catch (error) {
      // External service down hoặc timeout
      if (error.code === 'ETIMEDOUT' || error.code === 'ECONNREFUSED') {
        // Fallback: trả local data nếu có
        const { LocalCustomers } = cds.entities;
        return SELECT.from(LocalCustomers);
      }

      // Log lỗi và trả về thông báo user-friendly
      console.error('Business Partner API error:', error.message);
      req.error(503, 'Customer data temporarily unavailable. Please try again later.');
    }
  });
};
```

---

## 📊 Sơ đồ tổng hợp

```
                          ┌─────────────────────┐
                          │   package.json       │
                          │   cds.requires {     │
                          │     "BUPA": {        │
                          │       kind:"odata-v4"│
                          │       model: "..."   │
                          │     }                │
                          │   }                  │
                          └──────────┬──────────┘
                                     │
          ┌──────────────────────────▼────────────────────────┐
          │              CAP App                              │
          │  const bupa = await cds.connect.to('BUPA')       │
          │  srv.on('READ', 'Customers', req =>               │
          │    bupa.run(req.query)   ← delegate              │
          │  )                                                │
          └──────────────────────────┬────────────────────────┘
                                     │ HTTP OData request
                          ┌──────────▼──────────┐
                          │   Destination Service│ (BTP)
                          │   + credentials     │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │   S/4HANA Cloud     │
                          │   API endpoint      │
                          └─────────────────────┘
```

---

## ✅ Checklist thực hành

- [ ] Vào `api.sap.com` → tìm Business Partner API → download EDMX
- [ ] `cds import API_BUSINESS_PARTNER.edmx` → kiểm tra `srv/external/` và `package.json`
- [ ] Viết handler delegate `bupa.run(req.query)` cho entity Customers
- [ ] Thử mock data: tạo JSON file trong `srv/external/data/`
- [ ] Thử kết nối API sandbox của SAP API Hub (cần API key)
- [ ] Viết handler mash-up: combine local DB + external data

---

## 🔗 Tài liệu tham khảo

- [CAP — Consuming Services](https://cap.cloud.sap/docs/guides/using-services)
- [SAP Business Accelerator Hub](https://api.sap.com)
- [CAP — Destinations](https://cap.cloud.sap/docs/guides/using-services#destinations)
- [cds connect.to](https://cap.cloud.sap/docs/node.js/cds-connect)

---

> **Tóm lại:**  
> 1. **Download EDMX** từ api.sap.com  
> 2. **`cds import`** → CAP gen CDS model + update `package.json`  
> 3. **`cds.connect.to('ServiceName')`** → lấy service instance  
> 4. **`bupa.run(req.query)`** = delegate request sang external service  
> 5. **Mash-up** = combine local DB + external data  
> 6. **Local mock** = JSON files hoặc API sandbox key  
> 7. **Production** = BTP Destination Service quản lý credentials
