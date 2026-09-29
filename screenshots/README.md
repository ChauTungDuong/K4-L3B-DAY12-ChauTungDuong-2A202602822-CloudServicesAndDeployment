# Ảnh minh chứng deployment Railway

Thư mục này phải có hai ảnh PNG thật, chụp từ deployment của học viên:

- `dashboard.png`: Railway dashboard của service. Ảnh phải thấy service đã
  deploy thành công và domain public. Không để lộ giá trị của API key hoặc URL
  Redis nếu dashboard đang hiển thị chúng.
- `health.png`: terminal hoặc trình duyệt gọi
  `https://k4-l3b-day12-chau-tung-duong-2a202602822-cloud-s-production.up.railway.app/health`.
  Ảnh phải thấy HTTP `200` và JSON có `"status":"ok"`.

Nên chụp nguyên cửa sổ để thấy URL, thời điểm và kết quả cùng lúc. Trước khi
commit, mở thử từng ảnh để chắc chắn ảnh không rỗng, đọc được và không chứa
secret.
