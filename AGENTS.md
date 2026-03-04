# AGENTS.md

Hướng dẫn cho AI coding agents làm việc trong repository này.

## Tổng quan

Repository học SAP CAP (Cloud Application Programming Model). Nội dung gồm:
- Tài liệu học (Markdown) trong `learning/`
- Pet projects CAP (Node.js) trong `projects/`

## Cấu trúc quan trọng

```
learning/roadmap.md     # Lộ trình học tổng thể
learning/notes/         # Ghi chú từng chủ đề, đặt tên: 01-cds-basics.md
learning/cheatsheets/   # Quick reference, ngắn gọn
learning/exercises/     # Mini CDS exercises, mỗi bài một subfolder
projects/               # CAP projects, mỗi project là một cds app riêng
```

## Ngôn ngữ

- Tất cả tài liệu Markdown viết bằng **tiếng Việt**
- Code examples viết bằng **tiếng Anh** (comments có thể tiếng Việt)

## Conventions khi tạo tài liệu Markdown

- Dùng emoji ở heading chính để dễ scan
- Không ghi thời gian học (ví dụ: "tuần 1", "2 giờ") — bỏ hết
- Không mention framework cụ thể khác (NestJS, Spring...) trong so sánh
- Mỗi file notes có cấu trúc: lý thuyết → code example → checklist thực hành → links

## Conventions khi tạo CAP projects

```bash
# Tạo project mới
cd projects/<project-name>
cds init .
npm install

# Dev server
cds watch  # chạy tại localhost:4004

# Deploy schema
cds deploy --to sqlite:db.sqlite
```

## File types trong project CAP

```
db/schema.cds       # Data models (entities, aspects)
srv/*.cds           # Service definitions
srv/*.js            # Custom handlers (chỉ viết khi cần override)
db/data/*.csv       # Mock data
.cdsrc.json         # CAP config
mta.yaml            # Deployment config (BTP)
xs-security.json    # XSUAA config
```

## Không được sửa

- `learning/roadmap.md` — chỉ sửa khi được yêu cầu rõ ràng
- Xóa hoặc rename folder cấu trúc chính (`learning/`, `projects/`)
