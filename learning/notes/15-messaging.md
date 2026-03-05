# 📨 Messaging & Event Mesh — Kiến Trúc Hướng Sự Kiện

> **Event-Driven Architecture (EDA)** là pattern phổ biến trong enterprise. SAP **Event Mesh** là messaging service của BTP, đóng vai trò như Kafka/RabbitMQ nhưng native trên SAP ecosystem. CAP hỗ trợ gửi/nhận events qua API đơn giản.

---

## 🤔 Tại sao cần Messaging?

```
Vấn đề với synchronous:
──────────────────────
Client → CAP App → Gửi email → Cập nhật SAP → Gửi notification
                       ↑ Nếu 1 bước fail → toàn bộ request fail
                       ↑ Client phải chờ toàn bộ hoàn thành

Giải pháp với messaging (async):
──────────────────────────────────
Client → CAP App → [Return ngay 202 Accepted]
                       ↓ emit event
                  Event Mesh (queue/topic)
                       ↓ deliver
              Email Service    SAP System    Notification Service
              (xử lý riêng)   (xử lý riêng) (xử lý riêng)
```

**Lợi ích:**
- **Decoupled**: các services không phụ thuộc nhau trực tiếp
- **Resilient**: nếu 1 consumer fail, messages được giữ trong queue
- **Scalable**: consumer có thể scale độc lập
- **Async**: client không cần chờ toàn bộ workflow

---

## 📦 Topics vs Queues

```
Topic (pub/sub):                    Queue (point-to-point):
──────────────────                  ────────────────────────
Publisher                           Sender
    ↓ publish to topic                  ↓ send to queue
  "order.created"                    "email-processing-queue"
    ↓                                   ↓
  All subscribers                    One consumer
  nhận cùng 1 message               (first come, first served)
   - Email Service                    - Email Service
   - Analytics Service                  nhận và process
   - Audit Logger                    → reliable delivery

→ 1-to-many (fan-out)               → 1-to-1, guaranteed delivery
```

---

## ⚙️ Setup SAP Event Mesh

### Bước 1: Thêm messaging vào CAP project

```bash
# Thêm SAP Event Mesh support
cds add messaging

# Hoặc thêm thủ công vào package.json
```

**package.json sau khi thêm:**

```json
{
  "dependencies": {
    "@cap-js/messaging": "^1"    ← được thêm tự động
  },
  "cds": {
    "requires": {
      "messaging": {
        "kind": "enterprise-messaging",  // SAP Event Mesh
        "[development]": {
          "kind": "file-based-messaging"  // Mock local - dùng file
        }
      }
    }
  }
}
```

### Bước 2: Tạo Event Mesh service instance trên BTP

```bash
# Login CF
cf login

# Tạo Event Mesh instance (plan dev = trial)
cf create-service enterprise-messaging dev my-event-mesh

# Tạo service key để lấy credentials
cf create-service-key my-event-mesh my-event-mesh-key
cf service-key my-event-mesh my-event-mesh-key
```

---

## 📤 Emit — Gửi Events

### Trong CAP handler, sau khi có sự kiện

```javascript
// srv/order-service.js
module.exports = async (srv) => {
  const messaging = await cds.connect.to('messaging');

  srv.after('CREATE', 'Orders', async (order, req) => {
    // Gửi event sau khi order được tạo thành công
    await messaging.emit('my.shop.order.created', {
      orderId: order.ID,
      customerId: order.customerId,
      totalAmount: order.totalAmount,
      items: order.items,
      createdAt: new Date().toISOString()
    });

    console.log(`Event emitted: order.created for ${order.ID}`);
  });

  srv.after('UPDATE', 'Orders', async (order, req) => {
    // Gửi event khi order status thay đổi
    if (order.status === 'Shipped') {
      await messaging.emit('my.shop.order.shipped', {
        orderId: order.ID,
        shippingDate: new Date().toISOString(),
        trackingNumber: order.trackingNumber
      });
    }
  });
};
```

### Emit class-style (trong ApplicationService)

```javascript
class OrderService extends cds.ApplicationService {
  async init() {
    // Đăng ký event definition trong CDS
    await super.init();

    this.after('CREATE', 'Orders', async (order) => {
      // emit event tới messaging service
      await this.emit('OrderCreated', {
        order: order,
        timestamp: Date.now()
      });
    });
  }
}
module.exports = OrderService;
```

---

## 📥 Subscribe — Nhận và xử lý Events

### Nhận events trong cùng app

```javascript
// srv/notification-handler.js
const cds = require('@sap/cds');

module.exports = async () => {
  const messaging = await cds.connect.to('messaging');

  // Subscribe tới topic
  messaging.on('my.shop.order.created', async (msg) => {
    const { orderId, customerId, totalAmount } = msg.data;

    console.log(`New order received: ${orderId}`);

    // Xử lý: gửi email confirmation
    await sendConfirmationEmail(customerId, orderId, totalAmount);

    // Cập nhật analytics
    await updateAnalytics({ orderId, totalAmount });
  });

  messaging.on('my.shop.order.shipped', async (msg) => {
    const { orderId, trackingNumber } = msg.data;

    // Gửi shipping notification cho customer
    await sendShippingNotification(msg.data);
    console.log(`Shipping notification sent for ${orderId}`);
  });
};
```

### Nhiều consumers độc lập

```javascript
// srv/email-service.js — chỉ xử lý emails
const cds = require('@sap/cds');
module.exports = async () => {
  const messaging = await cds.connect.to('messaging');
  messaging.on('my.shop.order.created', async (msg) => {
    await sendEmail(msg.data);
  });
};

// srv/analytics-service.js — chỉ xử lý analytics
module.exports = async () => {
  const messaging = await cds.connect.to('messaging');
  messaging.on('my.shop.order.created', async (msg) => {
    await recordInAnalytics(msg.data);
  });
};
// Cả 2 đều nhận cùng 1 event "order.created" — fan-out!
```

---

## 🧪 Local Dev với File-based Messaging (Mock)

Khi dev local, không cần Event Mesh thật — dùng file-based mock:

```json
// package.json
{
  "cds": {
    "requires": {
      "messaging": {
        "[development]": {
          "kind": "file-based-messaging",
          "file": "/tmp/messages.json"    ← messages lưu vào file này
        }
      }
    }
  }
}
```

```bash
# Chạy local → messages sẽ được lưu vào file JSON
cds watch

# Xem messages đang pending
cat /tmp/messages.json
```

**Điểm hay:** Code emit/subscribe không đổi, chỉ cần đổi config `kind`.

---

## 📋 CDS Event Definition

Có thể khai báo events trong CDS để có type safety:

```cds
// srv/order-service.cds
service OrderService {
  entity Orders as projection on db.Orders;
  
  // Khai báo event types
  event OrderCreated {
    orderId    : UUID;
    customerId : UUID;
    amount     : Decimal;
  }
  
  event OrderShipped {
    orderId        : UUID;
    trackingNumber : String;
    shippedAt      : Timestamp;
  }
}
```

```javascript
// Emit typed event
srv.after('CREATE', 'Orders', async (order) => {
  await srv.emit('OrderCreated', {
    orderId: order.ID,
    customerId: order.customerId,
    amount: order.totalAmount
  });
});
```

---

## 🔄 Event Mesh Config trong MTA (Production)

```yaml
# mta.yaml
resources:
  - name: my-event-mesh
    type: org.cloudfoundry.managed-service
    parameters:
      service: enterprise-messaging
      service-plan: dev         # plan cho development/trial
      path: ./em-config.json    # Event Mesh configuraton

modules:
  - name: my-cap-app-srv
    type: nodejs
    requires:
      - name: my-event-mesh    # bind Event Mesh service
      - name: my-xsuaa
      - name: my-hana-db
```

```json
// em-config.json — Event Mesh namespace config
{
  "emname": "my-shop-messages",
  "namespace": "my/shop/1",
  "version": "1.1.0",
  "authority-type": "external"
}
```

---

## ⚡ Advanced: Event-Mesh Integration với SAP S/4HANA

```javascript
// Nhận business events từ S/4HANA Cloud
// (ví dụ: khi Sales Order được tạo trong S/4HANA)
module.exports = async () => {
  const messaging = await cds.connect.to('messaging');

  // S/4HANA emit event khi có Sales Order mới
  messaging.on('sap.s4.beh.salesorder.v1.SalesOrder.Created.v1', async (msg) => {
    const { SalesOrder, SoldToParty, RequestedDeliveryDate } = msg.data;
    
    console.log(`New S/4HANA Sales Order: ${SalesOrder}`);
    
    // Sync vào local DB
    await INSERT.into('LocalOrders').entries({
      externalId: SalesOrder,
      customerId: SoldToParty,
      deliveryDate: RequestedDeliveryDate
    });
  });
};
```

---

## 📊 Tóm tắt patterns

| Pattern | Dùng khi | Code |
|---|---|---|
| **Fire and Forget** | Không cần biết result | `messaging.emit(topic, data)` |
| **Request/Reply** | Cần response từ consumer | Dùng correlation ID |
| **Fan-out** | Nhiều consumer cùng nhận | Topic (1-to-many) |
| **Work Queue** | Chỉ 1 consumer xử lý | Queue (1-to-1) |
| **S/4HANA events** | Nhận events từ S/4HANA | Subscribe to pre-defined topics |

---

## ✅ Checklist thực hành

- [ ] `cds add messaging` → kiểm tra `package.json` changes
- [ ] Config `file-based-messaging` cho local dev
- [ ] Viết handler `after('CREATE', 'Orders')` → emit `order.created` event
- [ ] Viết subscriber nhận event và log ra console
- [ ] Kiểm tra file `/tmp/messages.json` khi tạo order
- [ ] Tạo 2 subscribers riêng biệt (email, analytics) cùng nhận 1 event
- [ ] Test: tạo order → verify cả 2 subscribers đều nhận event

---

## 🔗 Tài liệu tham khảo

- [CAP — Messaging](https://cap.cloud.sap/docs/guides/messaging/)
- [CAP — Events](https://cap.cloud.sap/docs/cds/events)
- [SAP Event Mesh documentation](https://help.sap.com/docs/SAP_EM)
- [Event-Driven Integration Patterns](https://cap.cloud.sap/docs/guides/messaging/event-mesh)

---

> **Tóm lại:**  
> - Event Mesh = SAP Kafka — pub/sub messaging service trên BTP  
> - **Topics** = 1-to-many; **Queues** = 1-to-1 reliable delivery  
> - `messaging.emit(topic, data)` = gửi event (fire & forget)  
> - `messaging.on(topic, handler)` = nhận và xử lý event  
> - **Local dev**: `kind: "file-based-messaging"` — không cần Event Mesh thật  
> - **Production**: tạo `enterprise-messaging` service instance, bind qua MTA
