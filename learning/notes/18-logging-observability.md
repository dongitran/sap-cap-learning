# 📊 Logging & Observability — Monitor CAP Apps

> Biết app đang hoạt động như thế nào quan trọng không kém code. CAP có built-in logging và tích hợp với SAP Cloud Logging / SAP Cloud ALM. Dưới demo là những gì bạn cần để monitor app production.

---

## 📝 CAP Built-in Logging — cds.log

### Log levels và cách dùng

```javascript
// srv/catalog-service.js
const cds = require('@sap/cds');

// Tạo logger cho service (gắn context label)
const LOG = cds.log('my.catalog');

module.exports = (srv) => {
  srv.before('READ', 'Books', (req) => {
    // Các log levels từ thấp → cao
    LOG.debug('Request query:', JSON.stringify(req.query));    // dev only
    LOG.verbose('Books are being read');                       // debug level
    LOG.info('User reading books:', req.user.id);              // standard
    LOG.warn('Low stock detected for request');               // warning
    LOG.error('Unexpected error occurred');                   // error
  });

  srv.after('READ', 'Books', (books) => {
    LOG.info(`Returned ${books.length} books`);
  });
};
```

### Config log levels

```json
// package.json — kiểm soát log level per module/environment
{
  "cds": {
    "log": {
      "levels": {
        "my.catalog": "debug",         // dev: show debug logs
        "cds.server": "info",          // framework logs: info+
        "cds.db": "warn",              // DB queries: warn+ only
        "*": "info"                    // Default: info+
      },
      "[development]": {
        "format": "plain"              // Readable format local
      },
      "[production]": {
        "format": "json"               // JSON format cho log aggregation
      }
    }
  }
}
```

### Log format comparison

```
Development (plain):
──────────────────────────────────────────────────────────
[2026-03-05T14:00:00] INFO  | my.catalog | User reading books: alice@company.com
[2026-03-05T14:00:00] WARN  | my.catalog | Low stock detected

Production (JSON — parseable bởi logging tools):
──────────────────────────────────────────────────────────
{"level":"info","msg":"User reading books: alice@company.com","module":"my.catalog","time":"2026-03-05T14:00:00Z","request_id":"abc123","tenant":"my-tenant"}
{"level":"warn","msg":"Low stock detected","module":"my.catalog","time":"2026-03-05T14:00:00Z"}
```

---

## 🏷️ Request Correlation — Trace requests end-to-end

```javascript
module.exports = (srv) => {
  srv.before('*', (req) => {
    // CAP tự attach correlation data vào LOG context
    const correlationId = req.headers['x-correlation-id'] || req.id;

    LOG.info('Request started', {
      correlationId,
      user: req.user.id,
      tenant: req.user.tenant,
      operation: `${req.method} ${req.target?.name}`
    });
  });

  srv.after('*', (results, req) => {
    LOG.info('Request completed', {
      correlationId: req.headers['x-correlation-id'] || req.id,
      resultCount: Array.isArray(results) ? results.length : 1
    });
  });

  srv.on('error', (err, req) => {
    LOG.error('Request failed', {
      correlationId: req.id,
      error: err.message,
      code: err.code,
      stack: err.stack
    });
  });
};
```

---

## ☁️ SAP Cloud Logging — Production Observability (2025+)

> ℹ️ **SAP Application Logging Service bị deprecated từ tháng 7/2025.** Dùng **SAP Cloud Logging** thay thế!

### Thêm Cloud Logging vào project

```bash
# Thêm Cloud Logging support
cds add cloud-logging
```

**Cập nhật mta.yaml:**

```yaml
resources:
  # SAP Cloud Logging Service
  - name: my-bookshop-logs
    type: org.cloudfoundry.managed-service
    parameters:
      service: cloud-logging
      service-plan: standard         # hoặc "dev" cho dev environment

modules:
  - name: my-bookshop-srv
    requires:
      - name: my-bookshop-logs      # Bind Cloud Logging service
```

---

## 🔍 Xem logs trên BTP (Cloud Foundry)

```bash
# Xem recent logs (vài phút gần nhất)
cf logs my-bookshop-srv --recent

# Tail logs real-time (như `tail -f`)
cf logs my-bookshop-srv

# Output:
# 2026-03-05T14:00:01.234+07:00 APP/PROC/WEB/0 OUT {"level":"info","msg":"User reading books","user":"alice"}
# 2026-03-05T14:00:01.435+07:00 APP/PROC/WEB/0 OUT {"level":"info","msg":"Returned 42 books"}
# 2026-03-05T14:00:02.100+07:00 APP/PROC/WEB/0 ERR {"level":"error","msg":"DB connection failed"}
```

---

## 📈 OpenTelemetry — Distributed Tracing (Advanced)

SAP Cloud Logging và SAP Cloud ALM đều dùng **OpenTelemetry** (OTel) — industry standard để thu thập metrics, traces, logs.

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

```javascript
// tracing.js (load trước app)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  serviceName: 'my-bookshop',
  instrumentations: [getNodeAutoInstrumentations()]
  // Exporter được config qua environment variables (Cloud Logging tự inject)
});

sdk.start();
```

```json
// package.json — load tracing trước app
{
  "scripts": {
    "start": "node --require ./tracing.js node_modules/@sap/cds/bin/cds serve"
  }
}
```

---

## 🎯 Custom Structured Logging Patterns

### Log business events (audit trail)

```javascript
module.exports = (srv) => {
  // Business event logging — dùng structured JSON
  srv.after('CREATE', 'Orders', (order, req) => {
    LOG.info('OrderCreated', {
      event: 'order.created',
      orderId: order.ID,
      customerId: order.customerId,
      totalAmount: order.totalAmount,
      userId: req.user.id,
      tenant: req.user.tenant,
      timestamp: new Date().toISOString()
    });
  });

  // Security event logging
  srv.before('DELETE', 'Books', (req) => {
    if (!req.user.is('Admin')) {
      LOG.warn('UnauthorizedDeleteAttempt', {
        event: 'security.unauthorized_delete',
        userId: req.user.id,
        resource: 'Books',
        resourceId: req._params?.[0]?.ID
      });
    }
  });

  // Performance logging
  srv.before('READ', 'Books', (req) => {
    req._startTime = Date.now();
  });

  srv.after('READ', 'Books', (books, req) => {
    const duration = Date.now() - req._startTime;
    if (duration > 2000) {
      LOG.warn('SlowQuery', {
        event: 'performance.slow_query',
        entitySet: 'Books',
        duration,
        resultCount: books.length
      });
    }
  });
};
```

---

## 🔔 Health Check Endpoint

```javascript
// srv/health-service.js
const HEALTH = cds.log('health');

module.exports = (srv) => {
  // Simple health endpoint
  srv.on('health', async (req) => {
    try {
      const db = await cds.connect.to('db');
      await db.run(SELECT.from('my.app.Books').limit(1));
      
      HEALTH.info('Health check OK');
      return { status: 'OK', db: 'connected', timestamp: new Date() };
    } catch (err) {
      HEALTH.error('Health check FAILED', { error: err.message });
      req.error(503, 'Service Unavailable: ' + err.message);
    }
  });
};
```

```cds
// srv/health-service.cds
service HealthService @(path: '/health') {
  function health() returns { status: String; db: String; timestamp: Timestamp; };
}
```

---

## 💡 SAP Cloud ALM — Centralized Monitoring

```
Architecture (2025+):
─────────────────────────────────────────────────────────
CAP App
  ↓ OpenTelemetry (metrics, traces, logs)
SAP Cloud Logging
  ↓ aggregates, stores, enables search
  ↓ dashboards + alerting
SAP Cloud ALM
  ↓ problem detection
  ↓ cross-system health monitoring
  ↓ root cause analysis
```

**Setup trong mta.yaml:**

```yaml
resources:
  - name: my-cloud-alm
    type: org.cloudfoundry.managed-service
    parameters:
      service: cloud-logging
      service-plan: standard
      config:
        alm-enabled: true
```

---

## 📊 Bảng so sánh logging tools

| Tool | Dùng cho | 2025 Status |
|---|---|---|
| `cds.log` | CAP built-in logging | ✅ Current |
| SAP Cloud Logging | Aggregate, search, dashboard BTP | ✅ Recommended (mới) |
| SAP Application Logging | Old logging service | ❌ Deprecated từ 7/2025 |
| SAP Cloud ALM | Cross-system monitoring | ✅ Enterprise grade |
| `cf logs` | Quick CLI log viewing | ✅ Development |
| OpenTelemetry | Distributed tracing | ✅ Industry standard |

---

## ✅ Checklist thực hành

- [ ] Thêm `const LOG = cds.log('my-service')` vào handlers
- [ ] Log business events: `after CREATE Orders` → log orderId, userId
- [ ] Config log levels trong `package.json` (debug local, info prod)
- [ ] Viết health check endpoint (`GET /health`)
- [ ] Deploy lên CF → `cf logs my-app --recent` để xem production logs
- [ ] Thêm Cloud Logging service vào `mta.yaml`

---

## 🔗 Tài liệu tham khảo

- [CAP — Logging](https://cap.cloud.sap/docs/node.js/cds-log)
- [SAP Cloud Logging](https://help.sap.com/docs/cloud-logging)
- [SAP Cloud ALM](https://help.sap.com/docs/cloud-alm)
- [OpenTelemetry for CAP](https://cap.cloud.sap/docs/guides/observability)

---

> **Tóm lại:**  
> - `cds.log('module-name')` = CAP built-in logger (INFO/WARN/ERROR/DEBUG)  
> - **Production**: format JSON để logging tools parse được  
> - **SAP Application Logging đã deprecated từ 7/2025** → dùng SAP Cloud Logging  
> - `cf logs my-app --recent` = xem logs nhanh sau khi deploy  
> - OpenTelemetry = distributed tracing, tích hợp với SAP Cloud Logging và Cloud ALM  
> - Log cả **business events** (order.created) và **security events** (unauthorized access)
