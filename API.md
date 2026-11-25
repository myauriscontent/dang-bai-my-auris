# Web API Đăng bài My Auris — việc cần làm trên n8n (khoảng 1–2 giờ)

Web tĩnh (GitHub Pages) gọi **một** webhook trên n8n.myauris.vn. Webhook chỉ chuyển tiếp yêu cầu tới các sub-workflow mà connector "Đăng bài My Auris" đang dùng — không đổi gì hệ thống đăng bài.

## Workflow cần dựng (5 node)

`Webhook` → `Code: kiểm tra mã cá nhân` → `Switch theo action` → `Execute Workflow` (6 nhánh) → `Respond to Webhook`

1. **Webhook**: `POST`, path `dang-bai-api`, Respond = *Using 'Respond to Webhook' node*. Options → **Allowed Origins (CORS)** = địa chỉ GitHub Pages của web (vd `https://<tai-khoan>.github.io`).
2. **Code**: so `token` với danh sách mã cá nhân. Sai → Respond `401`. Chỉ nhận 6 action bên dưới, khác → `400`.
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

## Tuyến (ds_tuyen / tao_tuyen)

- Mỗi bài trong `danh_sach_bai` có trường `tuyen` (mã tuyến) — web lấy nhãn tuyến từ đây.
- `ds_tuyen` → `{ "ok": true, "tuyen": [ … ] }`. Mỗi tuyến web đọc được các trường sau (trường nào thiếu thì bỏ qua):
  - `ma` (hoặc `key`/`id`) — mã tuyến, khớp với trường `tuyen` của bài
  - `ten` — tên hiển thị
  - `thu_muc_id` hoặc `link_thu_muc` — thư mục Drive của tuyến (cho nút "mở thư mục Drive")
  - `kenh_video` — kênh chọn sẵn khi lên lịch video, vd `["FB: Bác Sĩ Nha Khoa My Auris", "TikTok: Bác sĩ Nha Khoa My Auris"]`
  - `kenh_anh` — kênh chọn sẵn cho ảnh/album (bỏ trống = 3 page chính)
- `tao_tuyen` `{ ten }` → `{ "ok": true, "thong_bao": "…" }` (web hiện `thong_bao` nếu có).

## Đăng nhập: mã riêng mỗi người

- Web chỉ hỏi **mã cá nhân**, gửi trong trường `token`. Sai / bị khóa → `401` kèm `loi` (web hiện nguyên câu này).
- Nếu `tong_quan` (hoặc `ds_tuyen`) trả thêm `nguoi_dung: "Tên"`, web hiện "Đang dùng: Tên" ở thanh trên cùng.

## Người đang đăng nhập & người quản lý bài (web đã sẵn, chờ API)

1. **Tên người đang đăng nhập** — thêm vào kết quả `tong_quan`:
   `"nguoi_dung": { "ten": "Lan", "email": "lan@myauris.vn" }` (hoặc chỉ chuỗi tên). Lấy theo mã cá nhân gửi lên.
   Web hiện ở góc trên: "Đang đăng nhập: Lan".
2. **Người quản lý từng bài** — thêm vào mỗi bài trong `danh_sach_bai` (cả danh sách lẫn xem theo mã):
   `"nguoi_quan_ly": { "ten": "Lan", "email": "lan@myauris.vn" }`
   - Bài tải lên qua web: người có mã đã gọi `tao_link_tai_len` (lưu lại lúc tạo link, khớp theo tên file).
   - Bài cũ tải thẳng lên Drive: chủ sở hữu file trên Drive (trường `owners` của file).
   Web hiện "Quản lý: Lan" trên thẻ bài, danh sách và lịch tuần.
3. (Tuỳ chọn) Dòng kênh đã lên lịch: `"nguoi_len_lich": "Tên"` — web hiện "Lên lịch: Tên" ở tab Sắp đăng.

## Album: thứ tự ảnh & xem trước bố cục

- Web đánh số tên file theo thứ tự người dùng sắp (`01_…`, `02_…`) khi gọi `tao_link_tai_len`, nên album trên Drive xếp đúng thứ tự khi đăng (hệ thống đang xếp theo tên file).
- (Tuỳ chọn, để mọi máy đều xem trước được bố cục) — `danh_sach_bai` xem theo mã trả thêm cho album:
  `"anh": ["<fileId1>", "<fileId2>", …]` theo đúng thứ tự đăng (hoặc `[{ "id": "…", "w": 1080, "h": 1350 }]` nếu có kích thước).
  Không có trường này thì web chỉ xem trước được trên máy đã tải album lên.

## Chi phí

n8n tự cài trên máy chủ công ty: không giới hạn lượt chạy, không phát sinh phí.
