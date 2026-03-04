# sap-cap-learning

Tài liệu học và thực hành SAP Cloud Application Programming Model (CAP).

## Cấu trúc

```
├── learning/
│   ├── roadmap.md        # Lộ trình học tổng thể
│   ├── notes/            # Ghi chú chi tiết từng chủ đề
│   ├── cheatsheets/      # Quick reference / tra cứu nhanh
│   └── exercises/        # Bài tập thực hành nhỏ
└── projects/             # Pet projects thực hành CAP
```

## Stack

- **Runtime:** Node.js
- **Framework:** SAP CAP (`@sap/cds`)
- **Database:** SQLite (dev) → SAP HANA Cloud (prod)
- **Platform:** SAP BTP (Cloud Foundry / Kyma)
- **Auth:** XSUAA
- **API:** OData v4 (auto-generated từ CDS)

## Bắt đầu

```bash
npm install -g @sap/cds-dk
cds version
```

Đọc [`learning/roadmap.md`](./learning/roadmap.md) để bắt đầu.
