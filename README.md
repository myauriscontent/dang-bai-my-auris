# Đăng bài My Auris

Web nội bộ để team tải bài, xem caption AI viết sẵn, chọn kênh – ngày – giờ và lên lịch đăng (lẻ hoặc hàng loạt) lên các page Facebook, Instagram, TikTok của My Auris.

## Cách dùng

1. Bấm **Tải bài lên** → mở thư mục Drive của tuyến (Bác sĩ / Nha khoa My Auris / My Auris Việt Kiều) và bỏ video/ảnh vào.
2. Sau 2–5 phút bài hiện ở **Kho bài**, có caption AI.
3. Tick các bài → **Lên lịch**: cùng ngày giờ, mỗi ngày 1 bài, hoặc tự chọn từng bài. Kênh được chọn sẵn theo tuyến, bấm để đổi.
4. **Sắp đăng**: dời giờ, tạm hoãn, hủy. **Cần xử lý**: duyệt lại bài lỗi / vừa sửa. **Đã đăng**: link bài.

Vào web bằng **mã cá nhân** do anh Duy cấp (nhập một lần trên mỗi máy).

## Kỹ thuật

- Web tĩnh: `index.html` + `config.js`, chạy trên GitHub Pages.
- Tải file trực tiếp lên Drive, sửa prompt caption theo tuyến.
- Dữ liệu và thao tác đi qua webhook n8n (`config.js → API_URL`). Mô tả API: [API.md](API.md).
- Repo không chứa mật khẩu hay token nào.
