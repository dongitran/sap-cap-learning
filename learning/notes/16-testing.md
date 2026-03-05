# 🧪 Testing CAP Apps — Unit, Integration & Auth Tests

> CAP có built-in test framework `cds.test`, tương tự **Supertest** cho Express nhưng được tối ưu cho OData và CAP services. Là một developer đến từ NestJS/TypeScript, bạn sẽ thấy pattern này rất quen thuộc.

---

## 🤔 Tại sao cần test trong CAP?

```
CAP auto-generate nhiều thứ → bạn chỉ viết custom logic
→ Cần test:
  1. Custom handlers (before/on/after)
  2. OData endpoints (CRUD operations)
  3. Access control (@requires, @restrict)
  4. Business actions và functions
  5. External service integrations
```

---

## ⚙️ Setup Testing Environment

### Cài test runner và assertion library

```bash
# Cài Jest (recommended) hoặc Mocha
npm install --save-dev jest

# package.json
{
  "scripts": {
    "test": "jest --testEnvironment node"
  },
  "jest": {
    "testEnvironment": "node"
  }
}
```

### Cấu trúc thư mục test

```
my-cap-app/
├── db/
├── srv/
├── test/                               ← thư mục test
│   ├── catalog-service.test.js         ← test CatalogService
│   ├── order-service.test.js           ← test OrderService
│   ├── auth.test.js                    ← test access control
│   └── fixtures/                       ← mock data cho tests
│       └── books.json
└── package.json
```

---

## 🚀 cds.test — Test Server cơ bản

```javascript
// test/catalog-service.test.js
const cds = require('@sap/cds');

// cds.test('.') → start CAP server trước khi test
// Tự động stop server sau khi test xong
const { GET, POST, PUT, DELETE, expect } = cds.test('.');

// Hoặc chỉ test specific service:
// const { GET, POST } = cds.test('./srv/catalog-service.cds');

describe('CatalogService', () => {
  // Test đọc data
  it('GET /catalog/Books → returns list of books', async () => {
    const { data, status } = await GET('/catalog/Books');
    
    expect(status).to.equal(200);
    expect(data.value).to.be.an('array');
    expect(data.value.length).to.be.above(0);
  });

  // Test filter với OData
  it('GET /catalog/Books?$filter → filters correctly', async () => {
    const { data } = await GET('/catalog/Books', {
      params: { $filter: 'stock gt 100', $select: 'ID,title,stock' }
    });
    
    data.value.forEach(book => {
      expect(book.stock).to.be.above(100);
    });
  });

  // Test create
  it('POST /catalog/Books → creates a book', async () => {
    const { status, data } = await POST('/catalog/Books', {
      ID: 999,
      title: 'Test Book',
      stock: 50,
      price: 19.99
    });
    
    expect(status).to.equal(201);
    expect(data.title).to.equal('Test Book');
  });

  // Test update
  it('PATCH /catalog/Books(999) → updates stock', async () => {
    const { status } = await PUT('/catalog/Books(999)', {
      stock: 100
    });
    
    expect(status).to.be.oneOf([200, 204]);
  });

  // Test delete
  it('DELETE /catalog/Books(999) → removes book', async () => {
    const { status } = await DELETE('/catalog/Books(999)');
    expect(status).to.equal(204);
  });
});
```

---

## 📡 Test Actions và Functions

```javascript
describe('Custom Actions', () => {
  // Test bound action
  it('POST submitOrder → places order successfully', async () => {
    const { data, status } = await POST('/catalog/submitOrder', {
      book: 1,       // BookID
      quantity: 3
    });
    
    expect(status).to.equal(200);
    expect(data).to.have.property('message');
  });

  // Test action với validation
  it('POST submitOrder với quantity quá lớn → error 400', async () => {
    try {
      await POST('/catalog/submitOrder', {
        book: 1,
        quantity: 9999    // Stock không đủ
      });
    } catch (error) {
      expect(error.response.status).to.equal(400);
      expect(error.response.data.error.message).to.include('Not enough stock');
    }
  });

  // Test function
  it('GET getTopBooks() → returns top selling books', async () => {
    const { data } = await GET('/catalog/getTopBooks()');
    
    expect(data.value).to.be.an('array');
    expect(data.value).to.have.length.above(0);
    // Verify sorted by sales
    for (let i = 0; i < data.value.length - 1; i++) {
      expect(data.value[i].sales).to.be.gte(data.value[i + 1].sales);
    }
  });
});
```

---

## 🔐 Test Authentication & Authorization

```javascript
describe('Access Control', () => {
  let testSrv;
  
  before(() => {
    // Setup test với mock auth
    testSrv = cds.test('.');
  });

  // Test với user được xác thực
  it('Admin user → có thể DELETE', async () => {
    const { status } = await testSrv
      .DELETE('/admin/Books(1)')
      .set('Authorization', 'Basic alice:alice');  // alice = Admin role
    
    expect(status).to.equal(204);
  });

  // Test với user không có quyền
  it('Viewer user → KHÔNG thể DELETE → 403', async () => {
    let error;
    try {
      await testSrv
        .DELETE('/admin/Books(2)')
        .set('Authorization', 'Basic bob:bob');  // bob = Viewer only
    } catch (e) {
      error = e;
    }
    
    expect(error.response.status).to.equal(403);
  });

  // Test không có authentication
  it('Anonymous → KHÔNG thể access authenticated service → 401', async () => {
    let error;
    try {
      await GET('/admin/Books');  // Admin service yêu cầu auth
    } catch (e) {
      error = e;
    }
    
    expect(error.response.status).to.equal(401);
  });

  // Test role-specific data visibility
  it('Viewer → KHÔNG thấy field nhạy cảm (cost/margin)', async () => {
    const { data } = await testSrv
      .GET('/catalog/Books(1)')
      .set('Authorization', 'Basic bob:bob');
    
    expect(data).to.not.have.property('cost');
    expect(data).to.not.have.property('margin');
  });
});
```

---

## 🔬 Test Service APIs trực tiếp (không qua HTTP)

```javascript
describe('Service API Tests (programmatic)', () => {
  let srv;
  
  beforeAll(async () => {
    // Start CAP, truy cập service trực tiếp
    await cds.connect();
    srv = await cds.connect.to('CatalogService');
  });

  it('SELECT Books → returns books', async () => {
    const books = await srv.run(
      SELECT.from('CatalogService.Books').limit(5)
    );
    
    expect(books).to.be.an('array');
    expect(books.length).to.be.at.most(5);
  });

  it('INSERT Book → thành công', async () => {
    const result = await srv.create('CatalogService.Books', {
      title: 'Test via API',
      stock: 10,
      price: 9.99
    });
    
    expect(result.ID).to.exist;
    expect(result.title).to.equal('Test via API');
  });
});
```

---

## 🧩 Test với Database (SQLite in-memory)

CAP mặc định dùng **in-memory SQLite** khi test → mỗi test suite bắt đầu với DB sạch.

```javascript
describe('Database Tests', () => {
  it('Bookshop DB → có books từ CSV seed data', async () => {
    const { data } = await GET('/catalog/Books');
    
    // CSV seed data luôn được load trong test mode
    expect(data.value).to.have.length.above(0);
    expect(data.value[0]).to.have.keys(['ID', 'title', 'stock', 'price']);
  });

  it('CRUD flow → create, read, update, delete', async () => {
    // Create
    const { data: created } = await POST('/catalog/Books', {
      title: 'CRUD Test Book', stock: 99, price: 5.00
    });
    const id = created.ID;
    
    // Read
    const { data: read } = await GET(`/catalog/Books(${id})`);
    expect(read.title).to.equal('CRUD Test Book');
    
    // Update
    await PUT(`/catalog/Books(${id})`, { stock: 50 });
    const { data: updated } = await GET(`/catalog/Books(${id})`);
    expect(updated.stock).to.equal(50);
    
    // Delete
    const { status } = await DELETE(`/catalog/Books(${id})`);
    expect(status).to.equal(204);
  });
});
```

---

## 🎭 Mock External Services trong Tests

```javascript
describe('External Service Tests', () => {
  beforeAll(async () => {
    // Mock external Business Partner API
    const bupa = await cds.connect.to('API_BUSINESS_PARTNER');
    
    // Override tất cả requests để return mock data
    bupa.on('*', (req, next) => {
      if (req.target.name.includes('A_BusinessPartner')) {
        return [
          { BusinessPartner: '1001', BusinessPartnerName: 'Test Corp' }
        ];
      }
      return next();
    });
  });

  it('GET Customers → delegates to mock Business Partner API', async () => {
    const { data } = await GET('/catalog/Customers');
    
    expect(data.value).to.have.length.above(0);
    expect(data.value[0].BusinessPartnerName).to.equal('Test Corp');
  });
});
```

---

## 🛠️ Test Configuration trong package.json

```json
{
  "scripts": {
    "test": "jest",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch"
  },
  "jest": {
    "testEnvironment": "node",
    "testMatch": ["**/test/**/*.test.js"],
    "collectCoverageFrom": [
      "srv/**/*.js",
      "!srv/external/**"
    ],
    "coverageThreshold": {
      "global": {
        "lines": 70,
        "functions": 70,
        "branches": 60
      }
    }
  },
  "cds": {
    "[test]": {
      "db": {
        "kind": "sqlite",
        "credentials": { "url": ":memory:" }
      },
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

---

## 📊 Tóm tắt các loại tests

| Test type | Dùng khi | API |
|---|---|---|
| **HTTP tests** | Test OData endpoints | `GET`, `POST`, `PUT`, `DELETE` |
| **Auth tests** | Test roles và permissions | `.set('Authorization', 'Basic ...')` |
| **Service API** | Test business logic | `srv.run(SELECT...)` |
| **Action tests** | Test custom actions | `POST('/srv/action', data)` |
| **Mock ext. service** | Test external integrations | `svc.on('*', handler)` |

---

## ✅ Checklist thực hành

- [ ] Setup Jest trong project, cấu hình `package.json`
- [ ] Viết test Happy path cho GET/POST/PUT/DELETE bookshop
- [ ] Viết test validation: submitOrder với quantity không hợp lệ
- [ ] Viết test auth: `alice` (admin) vs `bob` (viewer) cho operations
- [ ] Chạy `npm run test:coverage` → đảm bảo > 70% coverage
- [ ] Mock external service trong tests (tránh gọi API thật)

---

## 🔗 Tài liệu tham khảo

- [CAP — Testing](https://cap.cloud.sap/docs/node.js/cds-test)
- [CAP — cds.test API](https://cap.cloud.sap/docs/node.js/cds-test#http-level)
- [Jest documentation](https://jestjs.io/docs/getting-started)

---

> **Tóm lại:**  
> - `cds.test('.')` = start CAP server tự động cho tests (như `supertest` cho Express)  
> - `GET/POST/PUT/DELETE` = HTTP test helpers với full OData support  
> - **Mock auth**: basic auth `alice:alice` = Admin, `bob:bob` = Viewer  
> - **In-memory SQLite** mặc định trong tests → DB sạch mỗi test suite  
> - **Mock external services**: `svc.on('*', handler)` override all requests  
> - Coverage target: **70% lines/functions** là minimum production-ready
