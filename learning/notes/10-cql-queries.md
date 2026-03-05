# 🔍 CQL — CAP Query Language

> **CQL** (CDS Query Language) là ngôn ngữ query của CAP để truy xuất database từ JavaScript. Nó tương tự SQL nhưng declarative hơn và giúp bạn viết code type-safe, database-agnostic.

---

## 🤔 CQL là gì và khác SQL như thế nào?

```javascript
// SQL thông thường — string dễ bị injection, không type-safe
const sql = `SELECT * FROM Books WHERE stock > ${userInput}`;  // ❌ nguy hiểm!

// CQL — declarative, type-safe, portable
const books = await SELECT.from(Books).where({ stock: { '>': 0 } });  // ✅
// → CAP tự gen SQL phù hợp cho SQLite hoặc HANA
```

**Ưu điểm CQL:**
- **Database-agnostic**: cùng code chạy trên SQLite (dev) và HANA (prod)
- **Type-safe**: VS Code autocomplete, ít typo
- **SQL-injection safe**: params được escape tự động
- **Readable**: gần giống SQL nên dễ đọc

---

## 📦 Import và Setup

```javascript
// srv/catalog-service.js

// Cách 1: Dùng destructuring từ cds (khuyến nghị)
const cds = require('@sap/cds');
const { SELECT, INSERT, UPDATE, DELETE } = cds.ql;

// Cách 2: Dùng global (CAP inject tự động trong handlers)
// SELECT, INSERT, UPDATE, DELETE có sẵn trong module scope
module.exports = (srv) => {
  const { Books, Authors, Orders } = srv.entities;

  srv.on('someAction', async (req) => {
    // SELECT, INSERT, UPDATE, DELETE đều có sẵn ở đây
  });
};
```

---

## 📖 SELECT — Đọc dữ liệu

### SELECT cơ bản

```javascript
// Lấy tất cả records
const books = await SELECT.from(Books);
// → SELECT * FROM Books

// Lấy với điều kiện
const availableBooks = await SELECT.from(Books).where({ stock: { '>': 0 } });
// → SELECT * FROM Books WHERE stock > 0

// Lấy 1 record theo key
const [book] = await SELECT.from(Books).where({ ID: 'uuid-here' });
// Hoặc shorthand:
const [book] = await SELECT.from(Books, 'uuid-here');
```

### SELECT với chỉ định columns

```javascript
// Chỉ lấy 1 số fields
const titles = await SELECT.from(Books).columns('ID', 'title', 'price');
// → SELECT ID, title, price FROM Books

// SELECT.one — chỉ lấy 1 record
const oneBook = await SELECT.one.from(Books).where({ ID: id });
// → SELECT TOP 1 * FROM Books WHERE ID = ?
```

### WHERE — Điều kiện lọc

```javascript
// Bằng / khác
await SELECT.from(Books).where({ status: 'Active' });         // WHERE status = 'Active'
await SELECT.from(Books).where({ status: { '!=': 'Draft' } }); // WHERE status != 'Draft'

// So sánh số
await SELECT.from(Books).where({ stock: { '>': 10 } });       // WHERE stock > 10
await SELECT.from(Books).where({ price: { '<=': 50 } });      // WHERE price <= 50

// Between
await SELECT.from(Books).where({ price: { between: 10, and: 50 } });
// → WHERE price BETWEEN 10 AND 50

// IN list
await SELECT.from(Books).where({ status: { in: ['Active', 'Published'] } });
// → WHERE status IN ('Active', 'Published')

// LIKE (pattern matching)
await SELECT.from(Books).where({ title: { like: 'Harry%' } });
// → WHERE title LIKE 'Harry%'

// AND — Object với nhiều keys
await SELECT.from(Books).where({ stock: { '>': 0 }, status: 'Active' });
// → WHERE stock > 0 AND status = 'Active'

// OR — Dùng mảng hoặc operator
await SELECT.from(Books).where({ status: 'New' }).or({ status: 'Active' });
// → WHERE status = 'New' OR status = 'Active'

// NULL check
await SELECT.from(Books).where({ author: null });               // WHERE author IS NULL
await SELECT.from(Books).where({ author: { '!=': null } });     // WHERE author IS NOT NULL
```

### ORDER BY và LIMIT

```javascript
// Sắp xếp
await SELECT.from(Books).orderBy('price');           // ORDER BY price ASC
await SELECT.from(Books).orderBy('price desc');      // ORDER BY price DESC
await SELECT.from(Books).orderBy('stock desc', 'price asc'); // Multi-sort

// Phân trang
await SELECT.from(Books).limit(10);              // LIMIT 10
await SELECT.from(Books).limit(10, 20);          // LIMIT 10 OFFSET 20
// Trang 3 (10 items/page): .limit(10, 20)
```

### Expand (JOIN) trong CQL

```javascript
// Expand association
const booksWithAuthors = await SELECT.from(Books)
  .columns('*', 'author{ID, name}');
// → SELECT Books.*, author.ID, author.name FROM Books LEFT JOIN Authors author ON ...

// Expand composition (items)
const ordersWithItems = await SELECT.from(Orders)
  .columns('*', 'items{*}');
```

### Template Literal Style (CQL Tag)

```javascript
// CQL cũng hỗ trợ template literal như đây — rất gần với SQL
const { Books } = cds.entities('my.bookshop');

const results = await cds.run(
  SELECT`from ${Books} where stock > ${10} order by price`
);
// Dynamic query với interpolation an toàn (params được escape)
```

---

## ✏️ INSERT — Tạo mới

```javascript
// Insert 1 record
await INSERT.into(Books).entries({
  title: 'New Book',
  stock: 10,
  price: 9.99,
  author_ID: 'uuid-of-author'
});
// → INSERT INTO Books (title, stock, price, author_ID) VALUES (...)

// Insert nhiều records cùng lúc
await INSERT.into(Books).entries([
  { title: 'Book A', stock: 5, price: 12.99 },
  { title: 'Book B', stock: 20, price: 8.99 },
  { title: 'Book C', stock: 15, price: 15.99 }
]);
// → Batch INSERT

// Insert và lấy lại record vừa tạo
const newBook = await INSERT.into(Books)
  .entries({ title: 'Harry Potter', stock: 100, price: 12.99 })
  .returning('*');
```

---

## 🔄 UPDATE — Cập nhật

```javascript
// Update theo key (dùng positional)
await UPDATE(Books, 'uuid-here').with({ stock: 5 });
// → UPDATE Books SET stock = 5 WHERE ID = 'uuid-here'

// Update với WHERE clause
await UPDATE(Books).set({ stock: 0 }).where({ stock: { '<': 0 } });
// → UPDATE Books SET stock = 0 WHERE stock < 0

// Update nhiều fields
await UPDATE(Books, req.data.ID).with({
  title: 'Updated Title',
  price: 15.99,
  modifiedAt: new Date()
});

// Increment/Decrement (atomic update)
await UPDATE(Books, id).with(`stock = stock - ${quantity}`);
// → UPDATE Books SET stock = stock - ? WHERE ID = ?
// Quan trọng: tránh race condition khi nhiều requests cùng update
```

---

## 🗑️ DELETE — Xóa

```javascript
// Xóa theo key
await DELETE.from(Books).where({ ID: 'uuid-here' });
// → DELETE FROM Books WHERE ID = 'uuid-here'

// Xóa theo điều kiện
await DELETE.from(Books).where({ stock: 0 });
// → DELETE FROM Books WHERE stock = 0

// Xóa theo nhiều keys
await DELETE.from(Books).where({ ID: { in: ['id1', 'id2', 'id3'] } });

// Hoặc shorthand
await DELETE.from(Books, 'uuid-here');
```

---

## 🔁 Transactions — Giao dịch

Khi nhiều thao tác cần thực hiện cùng nhau (all or nothing):

```javascript
srv.on('transferStock', async (req) => {
  const { fromBook, toBook, quantity } = req.data;

  // Cách 1: Trong handler, CAP tự wrap trong transaction
  // Nếu bất kỳ lệnh nào throw error → auto rollback
  const [from] = await SELECT.from(Books, fromBook);
  if (from.stock < quantity) return req.error(409, 'Not enough stock');

  await UPDATE(Books, fromBook).with({ stock: from.stock - quantity });
  await UPDATE(Books, toBook).with(`stock = stock + ${quantity}`);
  
  // Nếu UPDATE thứ 2 fail → cả 2 UPDATEs đều rollback
  return 'Transfer complete';
});

// Cách 2: Explicit transaction khi cần
const tx = await cds.tx(req);
try {
  await tx.run(UPDATE(Books, fromBook).with({ stock: '...' }));
  await tx.run(UPDATE(Books, toBook).with({ stock: '...' }));
  await tx.commit();
} catch (e) {
  await tx.rollback();
  throw e;
}
```

---

## 💡 Patterns hay dùng

### Pattern: Check exists trước khi action

```javascript
srv.on('deleteBook', async (req) => {
  const { bookId } = req.data;
  
  // Check book exists
  const [book] = await SELECT.from(Books, bookId);
  if (!book) return req.error(404, `Book ${bookId} not found`);
  
  // Check không có orders active
  const activeOrders = await SELECT.from(OrderItems)
    .where({ book_ID: bookId })
    .columns('count(*) as n');
  if (activeOrders[0].n > 0) {
    return req.error(409, 'Cannot delete book with existing orders');
  }
  
  await DELETE.from(Books, bookId);
  return 'Deleted';
});
```

### Pattern: Upsert (insert or update)

```javascript
srv.on('upsertBook', async (req) => {
  const book = req.data;
  
  const [existing] = await SELECT.from(Books, book.ID);
  if (existing) {
    await UPDATE(Books, book.ID).with(book);
  } else {
    await INSERT.into(Books).entries(book);
  }
});
```

### Pattern: Aggregation query

```javascript
// Đếm, tính tổng
const stats = await SELECT.from(Books).columns([
  'count(*) as totalBooks',
  'sum(stock) as totalStock',
  'avg(price) as avgPrice',
  'max(price) as maxPrice'
]);
// stats[0] = { totalBooks: 150, totalStock: 3420, avgPrice: 12.5, maxPrice: 49.99 }
```

---

## ⚠️ Lưu ý

| Vấn đề | Giải thích | Fix |
|---|---|---|
| `await SELECT` trả về array | Ngay cả khi 1 record | Dùng `const [item] = await SELECT...` |
| `SELECT.one` vẫn trả `undefined` | Không tìm thấy | Check `if (!item) req.error(404, ...)` |
| Race condition khi update stock | 2 requests cùng đọc stock cũ | Dùng atomic `SET stock = stock - ?` |
| Circular reference khi expand | Model A expand B expand A | Giới hạn depth `columns('*', 'b{*}')` |

---

## ✅ Checklist thực hành

- [ ] `SELECT.from(Books)` — lấy tất cả, verify data trong browser
- [ ] `SELECT.from(Books).where({ stock: { '>': 0 } })` — filter
- [ ] `SELECT.from(Books).orderBy('price desc').limit(5)` — sort + phân trang
- [ ] `INSERT.into(Books).entries({...})` — tạo mới
- [ ] `UPDATE(Books, id).with({...})` — cập nhật
- [ ] `DELETE.from(Books).where({...})` — xóa
- [ ] Viết action dùng SELECT + UPDATE cùng nhau trong 1 handler

---

## 🔗 Tài liệu tham khảo

- [CQL — CAP Query Language](https://cap.cloud.sap/docs/cds/cql)
- [Node.js — cds.ql Reference](https://cap.cloud.sap/docs/node.js/cds-ql)
- [Querying Databases](https://cap.cloud.sap/docs/guides/databases)

---

> **Tóm lại:**  
> - **SELECT** → đọc, trả về array — luôn dùng `const [item] = await SELECT.from(..., key)`
> - **INSERT.into()** → tạo mới, batch insert được  
> - **UPDATE()** → update theo key hoặc WHERE  
> - **DELETE.from()** → xóa theo key hoặc WHERE  
> - Trong handlers, tất cả ops tự wrap trong **transaction** — throw error = rollback tự động
