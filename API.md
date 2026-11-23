# Web API Đăng bài My Auris — việc cần làm trên n8n (khoảng 1–2 giờ)

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

### 3 action mới (tải file trực tiếp + prompt theo tuyến)

| action | input | trả về |
|---|---|---|
| `tao_link_tai_len` | `tuyen` (bacsi / nhakhoa / vietkieu), `ten_file`, `mime`, `kich_thuoc`, `album` (tên album, rỗng nếu bài lẻ) | `{ "ok": true, "upload_url": "…" }` |
| `xem_prompt` | `{}` | `{ "ok": true, "prompts": { "bacsi": "…", "nhakhoa": "…", "vietkieu": "…" } }` (mỗi giá trị có thể là chuỗi hoặc `{ noi_dung, cap_nhat, nguoi }`) |
| `sua_prompt` | `tuyen`, `prompt` | `{ "ok": true }` |

**`tao_link_tai_len`** — web tải file thẳng lên Google Drive (video lớn không đi qua n8n):
1. Chọn thư mục theo `tuyen`: bacsi → `FP: Bác Sĩ My Auris`, nhakhoa → `FP: Nha Khoa My Auris`, vietkieu → `FP: My Auris`. Nếu `album` có giá trị: tạo (hoặc dùng lại) thư mục con tên `album` trong thư mục tuyến.
2. Node HTTP Request (credential Google Drive OAuth2): `POST https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable&supportsAllDrives=true`, body `{ "name": ten_file, "parents": [folderId] }`, header `X-Upload-Content-Type: mime`, `X-Upload-Content-Length: kich_thuoc`, **`Origin: https://myauriscontent.github.io`** (bắt buộc để trình duyệt được phép tải lên).
3. Trả header `Location` của response làm `upload_url`. Web tự `PUT` file vào đó; xong thì luồng quét Drive 2 phút/lần đang có tự nhận file như bình thường.

**`xem_prompt` / `sua_prompt`** — luồng AI viết caption chọn prompt theo thư mục tuyến của file (FP: Bác Sĩ / FP: Nha Khoa / FP: My Auris). Lưu 3 prompt ở chỗ cấu hình đang lưu prompt hiện tại; `sua_prompt` ghi tab Log (người sửa = `nguoi`, giá trị cũ, giá trị mới).

## Mã chung

Một mã cho cả team, không tạo tài khoản. Đổi mã khi có người nghỉ việc: sửa ở node Code, báo lại team. Web chỉ cần nhập một lần trên mỗi máy.

## Chi phí

n8n tự cài trên máy chủ công ty: không giới hạn lượt chạy, không phát sinh phí.
