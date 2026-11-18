# Hướng dẫn Triển khai & Cấu hình: SilentGuard Edge AI Server

Tài liệu này hướng dẫn cách build, deploy và cấu hình máy chủ xử lý AI (Edge Server) của SilentGuard trên môi trường Cloud (Railway) hoặc máy chủ nội bộ.

---

## 1. Kiến Trúc Triển Khai (Deployment Architecture)

Máy chủ AI được đóng gói dưới dạng Docker Container chạy độc lập, giao tiếp với máy chủ Backend thông qua REST API.

```
+─────────────────────────────────+
|         Imou Cloud API          |
+────────────────+────────────────+
                 │
                 │ (1) Lấy URL HLS dynamic (.m3u8)
                 ▼
+─────────────────────────────────+
|     SilentGuard AI Server       |
|    - FastAPI (Cổng 8080)        |         (2) Gửi HTTP POST JSON Alert
|    - YOLOv8 Pose Process        ├─────────────────────────────────────┐
|    - Raw DoH DNS Client         |                                     │
+─────────────────────────────────+                                     ▼
                                                      +─────────────────────────────────+
                                                      |      SilentGuard Backend        |
                                                      |   (NestJS/NodeJS Server)        |
                                                      +─────────────────────────────────+
```

---

## 2. Giải Quyết Lỗi DNS Trên Railway (Bypass System DNS via DoH)

Trong môi trường container của Railway, cổng UDP 53 (DNS mặc định) thường bị chặn hoặc không thể phân giải các máy chủ nước ngoài của Imou (như `openapi.easy4ip.com` hoặc `openapi.imoulife.com`).

### Giải pháp tích hợp:
Để khắc phục triệt để, Edge AI Server tích hợp sẵn bộ phân giải **DoH (DNS-over-HTTPS) Client chạy trực tiếp trên raw TCP/TLS socket**:
* Kết nối trực tiếp đến DNS của Google (`8.8.8.8:443`) hoặc Cloudflare (`1.1.1.1:443`) thông qua giao thức HTTPS (TCP port 443).
* Tự động phân giải các chuỗi CNAME phức tạp của hệ thống CDN của Dahua/Imou.
* Phân giải địa chỉ IP xong sẽ thực hiện chèn trực tiếp IP vào URL kết nối và thiết lập tiêu đề `Host` trong HTTP Header để bypass hoàn toàn lỗi DNS của hệ thống mạng.

---

## 3. Danh Sách Biến Môi Trường (Environment Variables)

Khi deploy máy chủ AI lên Railway hoặc chạy Docker, bạn cần cấu hình các biến môi trường sau:

| Biến Môi Trường | Mô tả | Giá trị Ví dụ |
| :--- | :--- | :--- |
| `IMOU_APP_ID` | AppID đăng ký trên trang phát triển của Imou | `lc7492c68fxxxxxx` |
| `IMOU_APP_SECRET` | AppSecret tương ứng của tài khoản Imou | `9d55ab832xxxxxxxxxxxxxxx` |
| `IMOU_DEVICE_SN` | Số Serial Number của camera Imou cần giám sát | `61C06BDPBVC1D91` |
| `DEVICE_API_KEY` | Key định danh thiết bị của bạn để gửi lên backend | `sg_device_live_key_xyz` |
| `BACKEND_API_URL` | Endpoint API của Backend nhận sự kiện ngã | `https://backend.app/api/events/detect` |
| `IMOU_API_IP` | *(Tùy chọn)* IP tĩnh của API Imou để bypass DNS thủ công | `8.219.71.80` |
| `IMOU_API_HOST` | *(Tùy chọn)* Host API mặc định của Imou | `openapi.easy4ip.com` |

---

## 4. Hướng Dẫn Khởi Chạy Bằng Docker (Local / Self-hosted)

### Cách 1: Sử dụng Dockerfile.ai
Build image Docker chuyên dụng cho AI Server:

```bash
# Build image
docker build -f Dockerfile.ai -t silentguard-ai-server .

# Khởi chạy container
docker run -d \
  -p 8080:8080 \
  -e IMOU_APP_ID="your_app_id" \
  -e IMOU_APP_SECRET="your_app_secret" \
  -e IMOU_DEVICE_SN="your_camera_sn" \
  -e DEVICE_API_KEY="your_device_key" \
  -e BACKEND_API_URL="http://your-backend:3000/api/events/detect" \
  --name silentguard-ai \
  silentguard-ai-server
```

---

## 5. API Endpoints của AI Server

Khi hoạt động, AI Server mở cổng `8080` hỗ trợ các API sau:

### 1. Kiểm tra trạng thái máy chủ (Health Check)
* **Phương thức**: `GET`
* **Đường dẫn**: `/health`
* **Response**: `{"status": "ok"}`

### 2. Yêu cầu phân tích một Video Clip tải lên (Async)
Dùng cho luồng khi người dùng tải clip từ ứng dụng di động lên để phân tích hậu kỳ.
* **Phương thức**: `POST`
* **Đường dẫn**: `/analyze`
* **Request Body (JSON)**:
  ```json
  {
    "video_url": "https://signed-s3-storage-url.mp4",
    "upload_token": "vid_abc123xyz",
    "api_url": "https://backend.app/api/events/detect",
    "room": "bedroom"
  }
  ```
* **Response**:
  ```json
  {
    "status": "queued",
    "message": "AI analysis started in the background",
    "upload_token": "vid_abc123xyz"
  }
  ```

### 3. Kích hoạt giám sát luồng Camera động mới
Kích hoạt nóng một tiến trình YOLOv8 quét camera mới theo số serial.
* **Phương thức**: `POST`
* **Đường dẫn**: `/api/streams/start`
* **Request Body (JSON)**:
  ```json
  {
    "device_sn": "61C06BDPBVC1D91",
    "device_key": "device_key_secret",
    "api_url": "https://backend.app/api/events/detect"
  }
  ```

### 4. Tắt tiến trình quét Camera động
* **Phương thức**: `POST`
* **Đường dẫn**: `/api/streams/stop?device_sn=61C06BDPBVC1D91`

---

## 6. Cơ Chế Reconnect & Tính Tự Trị (Self-Healing Livestream 24/7)

Để đảm bảo hệ thống hoạt động liên tục 24/7 mà không cần sự can thiệp thủ công:
* **Lọc lỗi mất kết nối mạng**: Khi luồng livestream camera bị đứt (do mạng gia đình chập chờn, mất điện camera, v.v.), vòng lặp OpenCV sẽ phát hiện ra `ret=False`. Lúc này, tiến trình con sẽ tự động ngủ 2 giây, yêu cầu Imou Cloud API cấp lại link HLS động mới và kết nối lại từ đầu.
* **Giữ nóng mô hình YOLOv8**: Mô hình pose estimation được nạp sẵn vào bộ nhớ RAM của GPU/CPU một lần duy nhất lúc khởi chạy script, giúp quá trình khôi phục kết nối chỉ mất ít hơn 3 giây thay vì phải khởi động lại toàn bộ tiến trình python.
* **Tự khởi động lại từ FastAPI**: Nếu tiến trình phân tích camera bị crash hoàn toàn, FastAPI Lifespan Manager trong `src/ai_server.py` sẽ phát hiện sự kiện thoát và tự động spawn lại tiến trình con mới sau 10 giây.
