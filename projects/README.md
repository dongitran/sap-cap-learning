# 🗂️ SAP CAP — Pet Projects

Mỗi folder là một CAP project độc lập, tăng dần độ phức tạp.

```
projects/
├── 01-bookshop/              # Hello World của CAP — CRUD cơ bản
├── 02-order-management/      # Compositions, Actions, Security
└── 03-fullstack-enterprise/  # Full-stack: HANA + XSUAA + Fiori + Deploy
```

## Cách tạo project mới

```bash
cd projects/01-bookshop
cds init .
npm install
cds watch
```

## Thứ tự nên làm

1. `01-bookshop` → nắm CDS, OData cơ bản
2. `02-order-management` → Compositions, custom handlers, security
3. `03-fullstack-enterprise` → deploy thật lên BTP
