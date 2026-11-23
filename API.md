# Web API Đăng bài My Auris — việc cần làm trên n8n (khoảng 30 phút)

Web tĩnh (GitHub Pages) gọi **một** webhook trên n8n.myauris.vn. Webhook chỉ chuyển tiếp yêu cầu tới các sub-workflow mà connector "Đăng bài My Auris" đang dùng — không đổi gì hệ thống đăng bài.

## Workflow cần dựng (5 node)

`Webhook` → `Code: kiểm tra mã` → `Switch theo action` → `Execute Workflow` (6 nhánh) → `Respond to Webhook`

1. **Webhook**: `POST`, path `dang-bai-api`, Respond = *Using 'Respond to Webhook' node*. Options → **Allowed Origins (CORS)** = địa chỉ GitHub Pages của web (vd `https://<tai-khoan>.github.io`).
2. **Code**: so `token` với **một mã chung** của team (đặt trong biến môi trường hoặc ngay trong node). Sai → Respond `401`. Chỉ nhận 6 action bên dưới, khác → `400`.
3. **Switch** theo `action`, mỗi nhánh gọi đúng sub-workflow của công cụ cùng tên, truyền `input` y nguyên. Ghi người thao tác vào tab Log là `Web – {nguoi}`.
4. **Respond to Webhook**: trả nguyên JSON mà công cụ connector trả (`ok`, `loi`, `bai`, `mode`, `so_luong`, `kenh`, `da_dang_gan_day`, `mode_note`…). Web đã đọc đúng định dạng này.

## Request web gửi

```json
{ "action": "len_lich", "input": { "ma": "5MTGH", "lich": "My Auris | 24/11/2026 | 19:00" }, "token": "<mã chung>", "nguoi": "Lan" }
```

| action | input (giống tham số công cụ connector cùng tên) |
|---|---|
| `tong_quan` | `{}` |
| `danh_sach_bai` | `loc`, `ma`, `so_luong`, `tuyen` |
| `len_lich` | `ma`, `lich`, `caption`, `hashtags`, `tieu_de`, `van_dang` |
| `sua_bai` | `ma`, `tieu_de`, `caption`, `hashtags`, `ngay`, `gio` |
| `doi_trang_thai` | `ma`, `trang_thai` (skip / regenerate / draft), `ly_do` |
| `duyet_dang` | `ma`, `dang_ngay`, `van_dang` |

## Mã chung

Một mã cho cả team, không tạo tài khoản. Đổi mã khi có người nghỉ việc: sửa ở node Code, báo lại team. Web chỉ cần nhập một lần trên mỗi máy.

## Chi phí

n8n tự cài trên máy chủ công ty: không giới hạn lượt chạy, không phát sinh phí.
