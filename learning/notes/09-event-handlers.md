# ⚡ Event Handlers — Custom Business Logic

> **Event Handlers** là nơi bạn viết custom business logic trong CAP. Mọi request đều đi qua 3 pha: **before → on → after**. Hiểu cách này sẽ giúp bạn biết đặt code ở đâu.

---

## 🔄 Flow xử lý request trong CAP

```
Client gửi: POST /catalog/Books
                │
                ▼
    ┌─────────────────────────┐
    │   BEFORE phase          │  ← Validate, transform data
    │   srv.before('CREATE')  │     Reject nếu data sai
    └─────────────────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │   ON phase              │  ← Xử lý chính (mặc định: INSERT vào DB)
    │   srv.on('CREATE')      │     Override nếu cần logic đặc biệt
    └─────────────────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │   AFTER phase           │  ← Transform kết quả trả về
    │   srv.after('CREATE')   │     Thêm computed fields, hide sensitive data
    └─────────────────────────┘
                │
                ▼
        Response → Client
```

---

## 🏗️ Cách đăng ký Handler

Handler được đăng ký trong file `.js` cùng tên với file `.cds`:

```javascript
// srv/catalog-service.js
module.exports = (srv) => {
  // Đăng ký handlers ở đây
  srv.before('CREATE', 'Books', (req) => { ... });
  srv.on('READ',   'Books', (req, next) => { ... });
  srv.after('READ', 'Books', (books) => { ... });
};
```

**Cú pháp tổng quát:**

```javascript
srv.before(event, entity, handler)
srv.on(event, entity, handler)
srv.after(event, entity, handler)
```

**`event`** có thể là:
- `'CREATE'` — POST
- `'READ'` — GET
- `'UPDATE'` — PATCH/PUT
- `'DELETE'` — DELETE
- `'*'` — tất cả events

---

## 1️⃣ srv.before — Validation và Pre-processing

Chạy **trước** khi CAP xử lý. Dùng để:
- **Validate** dữ liệu đầu vào
- **Enrich** data (thêm field trước khi INSERT)
- **Authorization** check thêm
- **Reject** request nếu sai

```javascript
module.exports = (srv) => {
  const { Books } = srv.entities;

  // Validate trước khi tạo
  srv.before('CREATE', 'Books', (req) => {
    const { title, price, stock } = req.data;

    if (!title || title.trim() === '') {
      return req.error(400, 'Title is required');
    }
    if (price < 0) {
      return req.error(400, 'Price cannot be negative');
    }
    if (stock < 0) {
      return req.error(400, 'Stock cannot be negative');
    }
  });

  // Validate và clean data trước khi UPDATE
  srv.before('UPDATE', 'Books', (req) => {
    // Normalize dữ liệu
    if (req.data.title) {
      req.data.title = req.data.title.trim();
    }
    // Không cho phép update một số fields
    delete req.data.createdAt;    // readonly - đừng cho update
  });

  // Enrichment — thêm field tự sinh trước khi INSERT
  srv.before('CREATE', 'Orders', async (req) => {
    // Tạo order number tự động
    const count = await SELECT.one.from('Orders').columns('count(*) as n');
    req.data.orderNumber = `ORD-${String(count.n + 1).padStart(6, '0')}`;
  });
};
```

**Quan trọng về `req.error()` vs `req.reject()`:**

```javascript
// req.error() — ghi lỗi nhưng tiếp tục xử lý (collect nhiều lỗi)
srv.before('CREATE', 'Books', (req) => {
  if (!req.data.title) req.error(400, 'Title required');
  if (!req.data.price) req.error(400, 'Price required');
  // Cả 2 lỗi được collect và trả về cùng lúc
});

// req.reject() — dừng ngay lập tức
srv.before('UPDATE', 'Books', (req) => {
  if (!req.user.is('Admin')) req.reject(403, 'Only Admin can update');
  // Dừng tại đây, không xử lý tiếp
});

// Throw error — dừng với exception
srv.before('DELETE', 'Books', (req) => {
  throw new Error('Deletion is not allowed');
});
```

---

## 2️⃣ srv.on — Override hoặc Custom Logic chính

Chạy **trong pha chính**. Nếu không có `on` handler → CAP dùng mặc định (CRUD vào DB).

Dùng khi:
- **Override** hoàn toàn hành vi CRUD
- **Implement** Actions và Functions custom
- **Connect** external services
- **Complex** business logic cần control flow

```javascript
module.exports = (srv) => {
  const { Books } = srv.entities;

  // Override READ — thêm logic custom rồi gọi next()
  srv.on('READ', 'Books', async (req, next) => {
    console.log(`Books được đọc bởi: ${req.user.id}`);
    
    // next() = tiếp tục với default behavior (SELECT từ DB)
    const result = await next();
    return result;
  });

  // Override CREATE hoàn toàn
  srv.on('CREATE', 'Books', async (req) => {
    const book = req.data;
    
    // Logic tùy chỉnh
    book.slug = book.title.toLowerCase().replace(/ /g, '-');
    
    // Tự INSERT và trả về kết quả
    return INSERT.into(Books).entries(book);
  });

  // Implement Action
  srv.on('submitOrder', async (req) => {
    const { book: bookId, quantity } = req.data;
    
    const [book] = await SELECT.from(Books).where({ ID: bookId });
    if (!book) return req.error(404, `Book ${bookId} not found`);
    if (book.stock < quantity) {
      return req.error(409, `Not enough stock. Available: ${book.stock}`);
    }
    
    await UPDATE(Books).set({ stock: book.stock - quantity })
                       .where({ ID: bookId });
    
    return `Success: ordered ${quantity} copies of "${book.title}"`;
  });

  // Implement Function
  srv.on('getTopBooks', async (req) => {
    const { maxCount = 10 } = req.data;
    return SELECT.from(Books)
      .orderBy('stock desc')
      .limit(maxCount);
  });
};
```

**Sự khác biệt quan trọng khi dùng `next()`:**

```javascript
// Không gọi next() → override hoàn toàn, CAP không SELECT DB
srv.on('READ', 'Books', async (req) => {
  return [{ ID: 1, title: 'Hardcoded' }]; // Không query DB
});

// Gọi next() → CAP vẫn SELECT DB, bạn có thể modify sau
srv.on('READ', 'Books', async (req, next) => {
  const result = await next();   // CAP query DB trước
  result.forEach(b => b.discounted = b.price * 0.9);  // bạn modify
  return result;
});
```

---

## 3️⃣ srv.after — Transform kết quả

Chạy **sau** ON phase. Nhận `results` là data đã lấy từ DB.

Dùng khi:
- **Compute** derived fields từ results
- **Filter** hoặc mask sensitive data
- **Format** dữ liệu trước khi trả về client
- **Send notifications** sau khi operation thành công

```javascript
module.exports = (srv) => {
  const { Books } = srv.entities;

  // Thêm field tính toán sau khi đọc
  srv.after('READ', 'Books', (books) => {
    books.forEach(book => {
      // Thêm field status dựa trên stock
      if (book.stock === 0)        book.stockStatus = 'Out of Stock';
      else if (book.stock < 10)    book.stockStatus = 'Low Stock';
      else                         book.stockStatus = 'Available';
      
      // Mask sensitive data
      if (book.cost) delete book.cost;
    });
  });

  // After CREATE — gửi thông báo
  srv.after('CREATE', 'Orders', async (order, req) => {
    console.log(`New order created: ${order.ID} by ${req.user.id}`);
    // Gửi email, push notification, v.v.
    // await sendNotification(order);
  });

  // After DELETE — cleanup related data
  srv.after('DELETE', 'Books', async (_, req) => {
    const bookId = req.data.ID;
    console.log(`Book ${bookId} deleted, cleaning up related data...`);
    // Cleanup logic
  });
};
```

---

## 📦 Request Object (req)

`req` là object chính trong mọi handler:

```javascript
srv.on('CREATE', 'Books', async (req) => {
  // req.data — data từ request body
  console.log(req.data);
  // { title: 'Harry Potter', price: 12.99, stock: 10 }

  // req.user — user đang đăng nhập
  console.log(req.user.id);       // 'alice'
  console.log(req.user.locale);   // 'vi-VN'
  console.log(req.user.tenant);   // tenant ID (multi-tenant apps)
  console.log(req.user.is('Admin'));  // true/false — kiểm tra role

  // req.params — parameters từ URL (entity key)
  // Ví dụ: GET /catalog/Books(1)
  console.log(req.params);  // [ { ID: '1' } ]

  // req.query — toàn bộ parsed query (READ requests)
  console.log(req.query);

  // req.headers — HTTP headers
  console.log(req.headers['content-language']);

  // req.method — HTTP method
  console.log(req.method);  // 'POST', 'GET', etc.

  // req.error() — thêm lỗi (không dừng)
  req.error(400, 'Validation error message');

  // req.reject() — reject ngay lập tức
  req.reject(403, 'Access denied');

  // req.notify() — send notification message (không fail request)
  req.notify(200, 'Note: Value was auto-corrected');
});
```

---

## 🗃️ CQL trong Handlers — Query Database

Trong handlers, bạn dùng CQL (CAP Query Language) để query database:

```javascript
module.exports = (srv) => {
  const { Books, Authors } = srv.entities;

  srv.on('someAction', async (req) => {
    // SELECT
    const books = await SELECT.from(Books)
      .where({ stock: { '>': 0 } })
      .orderBy('price')
      .limit(10);

    const [book] = await SELECT.from(Books, { ID: req.data.bookId });
    // Hoặc:
    const [book] = await SELECT.from(Books).where({ ID: req.data.bookId });

    // SELECT with JOIN (expand association)
    const booksWithAuthors = await SELECT.from(Books)
      .columns('*', 'author{*}')   // expand author
      .where({ stock: { '>': 0 } });

    // INSERT
    await INSERT.into(Books).entries({
      title: 'New Book',
      stock: 10,
      price: 9.99
    });

    // INSERT nhiều records
    await INSERT.into(Books).entries([
      { title: 'Book 1', stock: 5 },
      { title: 'Book 2', stock: 10 }
    ]);

    // UPDATE
    await UPDATE(Books, { ID: req.data.bookId }).with({ stock: 0 });
    // Hoặc:
    await UPDATE(Books).set({ stock: 0 }).where({ ID: req.data.bookId });

    // DELETE
    await DELETE.from(Books).where({ ID: req.data.bookId });
  });
};
```

---

## 🔐 Authorization trong Handlers

```javascript
module.exports = (srv) => {
  srv.before('DELETE', 'Books', (req) => {
    // Kiểm tra role
    if (!req.user.is('Admin')) {
      return req.reject(403, 'Only admins can delete books');
    }
  });

  srv.on('submitOrder', async (req) => {
    // Chỉ user đã đăng nhập mới order được
    if (req.user.is('anonymous')) {
      return req.reject(401, 'Please login to order');
    }
    
    // Logic order...
  });

  // Handler cho authenticated user
  srv.before('CREATE', 'Orders', (req) => {
    // Gán user ID vào order
    req.data.createdBy = req.user.id;
  });
};
```

---

## 💡 Tips và Patterns hay dùng

### Pattern: Validate và enrich cùng lúc

```javascript
srv.before('CREATE', 'Orders', async (req) => {
  // 1. Validate
  if (!req.data.items || req.data.items.length === 0) {
    return req.error(400, 'Order must have at least one item');
  }
  
  // 2. Enrich
  req.data.status = 'Pending';
  req.data.orderDate = new Date();
  req.data.customerId = req.user.id;
});
```

### Pattern: Computed fields trong after

```javascript
srv.after('READ', 'Orders', (orders) => {
  orders.forEach(order => {
    // Tính tổng từ items (nếu đã expand)
    if (order.items) {
      order.totalAmount = order.items.reduce(
        (sum, item) => sum + (item.quantity * item.price), 0
      );
    }
    // Thêm display status
    order.statusLabel = {
      'P': 'Pending', 'C': 'Confirmed', 
      'S': 'Shipped', 'X': 'Cancelled'
    }[order.status] || 'Unknown';
  });
});
```

### Pattern: Đăng ký handlers với class

```javascript
// srv/catalog-service.js (cách class-based dùng async init)
class CatalogService extends cds.ApplicationService {
  async init() {
    const { Books } = this.entities;
    
    this.before('CREATE', Books, (req) => {
      if (!req.data.title) req.error(400, 'Title required');
    });
    
    this.on('submitOrder', async (req) => {
      // action logic
    });
    
    await super.init(); // QUAN TRỌNG: luôn gọi super.init()!
  }
}

module.exports = CatalogService;
```

---

## ✅ Checklist thực hành

- [ ] Viết `before('CREATE')` để validate: title bắt buộc, price >= 0
- [ ] Viết `before('UPDATE')` để prevent update trên `@readonly` fields tự chọn
- [ ] Viết `on('submitOrder')` action: check stock, giảm stock, return message
- [ ] Viết `after('READ', 'Books')` để thêm `stockStatus` field
- [ ] Test `req.user.is('Admin')` với mock users trong `.cdsrc.json`
- [ ] Test `req.data`, `req.params`, `req.user` trong handlers bằng `console.log`
- [ ] Viết 1 async handler dùng `await SELECT.from(...)` để query thêm data

---

## 🔗 Tài liệu tham khảo

- [CAP — Implementing Services](https://cap.cloud.sap/docs/guides/providing-services#event-handler)
- [CAP — Request Object](https://cap.cloud.sap/docs/node.js/events#request)
- [CQL — Query Language](https://cap.cloud.sap/docs/cds/cql)
- [CAP Node.js — Handlers](https://cap.cloud.sap/docs/node.js/core-services#event-handlers)

---

> **Tóm lại:**  
> - **before** = validate/enrich trước khi DB (stop nếu dùng `req.reject()`)  
> - **on** = custom logic chính, có thể override hoàn toàn (gọi `next()` để tiếp tục default)  
> - **after** = transform kết quả sau khi DB → computed fields, mask data  
> - `req.data` = body data, `req.user` = authenticated user, `req.error()` = lỗi  
> - CQL (`SELECT.from`, `INSERT.into`, `UPDATE`, `DELETE.from`) để query DB trong handlers
