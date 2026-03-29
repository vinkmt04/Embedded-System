# 🚀 Smart Vending Machine - ESP32 Core Controller
**Hệ thống điều khiển máy bán hàng tự động thông minh đa luồng (Multi-tasking) dựa trên FreeRTOS, tích hợp thanh toán VietQR (SePay API), Firebase Cloud Sync và Captive Portal.**

---

## 🧠 Kiến trúc Hệ thống (System Architecture)
Hệ thống sử dụng vi điều khiển **ESP32 (Dual-core)** đóng vai trò là "Bộ não Giao tiếp" (Communication Hub), giao tiếp với **STM32** (Bộ điều khiển Cơ/Điện) thông qua chuẩn UART bất đồng bộ. 

Mã nguồn không sử dụng vòng lặp `loop()` truyền thống mà được thiết kế hoàn toàn trên kiến trúc **FreeRTOS** với 6 Task chạy song song, được bảo vệ nghiêm ngặt bởi cơ chế **Mutex (Mutual Exclusion)** để chống tranh chấp bộ nhớ (Data Race).

### ⚙️ Finite State Machine (FSM)
Toàn bộ logic hệ thống được điều phối bởi một Máy trạng thái hữu hạn (`currentState`) gồm:
`IDLE` ➔ `WIFI_SETUP` ➔ `DOWNLOADING` ➔ `WAITING_PAY` ➔ `DISPENSING` ➔ `SUCCESS` / `TIMEOUT` / `JAMMED_ERROR` ➔ `RESTOCKING` ➔ `SYSTEM_OFFLINE`.

---

## 🔬 Giải phẫu Chi tiết Luồng Hoạt động (Deep Dive Analysis)

### 1. Luồng Giao tiếp UART Bất đồng bộ (TaskUART)
* **Bản chất kỹ thuật:** UART không có xung nhịp đồng bộ (Clock). Để tránh việc treo máy do chờ dữ liệu (Blocking), hệ thống sử dụng kỹ thuật **Non-blocking Read** (`Serial2.available() > 0`). CPU sẽ ngó qua "hòm thư" UART, nếu có ký tự thì nhặt vào bộ đệm RAM (`rxBuf`), nếu không có thì lập tức nhường quyền cho Task khác (`vTaskDelay`).
* **Thuật toán Parsing (Thái dữ liệu):** Dữ liệu được truyền theo gói tin kết thúc bằng ký tự `\n` (Terminator). Khi nhận đủ chuỗi (VD: `ORDER,1,DH,5000\n`), hệ thống chốt mảng (Null-terminated `\0`), sau đó dùng thuật toán tìm kiếm dấu phẩy `indexOf(',')` để cắt lấy `Slot` và `Amount`.
* **Hardware RNG:** Mã đơn hàng (Order ID) không dùng hàm `random()` giả của Arduino mà gọi trực tiếp API cấp thấp của lõi ESP-IDF: `esp_random()`. Nó sử dụng nhiễu sóng vô tuyến và nhiễu nhiệt (Thermal noise) phần cứng để tạo ra chuỗi số True Random, đảm bảo không bao giờ trùng lặp ID thanh toán dù mạch bị khởi động lại.

### 2. Luồng Xử lý Thanh toán (TaskCheckPayment)
* **API HTTPS GET:** ESP32 mở cổng 443, thực hiện bắt tay bảo mật TLS/SSL với máy chủ SePay. Sử dụng `client.setInsecure()` để bỏ qua bước xác minh Root CA, giúp tiết kiệm hàng Megabyte bộ nhớ Flash nhưng vẫn đảm bảo đường hầm dữ liệu (Tunnel) được mã hóa.
* **Tối ưu hóa RAM (Heap Memory):** Cấu trúc JSON trả về từ ngân hàng rất lớn. Hệ thống gửi Request kèm tham số `?limit=1` để ép Server chỉ trả về đúng 1 giao dịch mới nhất. 
* **JSON Deserialization:** Chuỗi JSON tải về được giải nén vào vùng nhớ quy hoạch sẵn `DynamicJsonDocument doc(2048)`. Việc giới hạn 2KB RAM (Malloc) giúp chống hiện tượng Phân mảnh Heap (Heap Fragmentation) và Tràn bộ nhớ (Overflow) thường gặp trên các vi điều khiển cấp thấp.
* **Xác thực:** So sánh `amount_in >= localAmount` và tìm kiếm chuỗi `indexOf(checkID)` (đã được `toUpperCase()`) để cấp lệnh nhả hàng an toàn, chống gian lận chuyển thiếu tiền.

### 3. Cấu hình Mạng bằng Captive Portal (TaskWiFiConfig)
* **SoftAP & DHCP Server:** Khi người dùng bấm nút cấu hình, ESP32 ngắt kết nối STA, chuyển sang chế độ Access Point, tự cấp phát dải IP `192.168.4.x`.
* **DNS Spoofing:** Đánh chặn mọi truy vấn DNS (Cổng 53) từ điện thoại (đặc biệt là iOS CNA - Captive Network Assistant). Trả về IP của chính ESP32 để ép điện thoại tự động văng giao diện trình duyệt Web lên màn hình.
* **Bảo mật WPA2 & Flash Storage:** Điểm phát sóng được mã hóa WPA2. Trình duyệt Web (lưu trong PROGMEM) sẽ nhận thông tin SSID/Pass từ người dùng, sau đó ghi thẳng vào phân vùng bộ nhớ không bay hơi NVS (Non-Volatile Storage) trước khi khởi động lại.
* **QR Code Offline:** Thuật toán ma trận `ricmoo/QRCode` sinh payload chuẩn `WIFI:S:MayBanHang_PTIT;T:WPA;P:12345678;;` hiển thị trực tiếp lên màn hình TFT ST7789, cho phép kết nối bằng Camera mặc định.

### 4. Đồng bộ Cloud (TaskWebSync)
* **Giao thức RESTful:** Giao tiếp với Firebase Realtime Database qua nền tảng HTTP PATCH/PUT. Duy trì kết nối bằng cờ Header `Connection: keep-alive` để tránh mất thời gian bắt tay TLS nhiều lần, đạt chuẩn Real-time.
* **Snapshot Mutex:** Khóa `sysMutex` chỉ được giữ trong vài micro-giây để copy (Snapshot) trạng thái máy vào các biến local (`localStock`, `localRev`). Ngay sau đó Mutex được thả ra cho hệ thống hoạt động bình thường, quá trình đẩy dữ liệu lên Firebase diễn ra độc lập bên dưới mà không làm nghẽn cổ chai (Bottleneck) hệ thống.
* **Data Recovery (Phục hồi dữ liệu):** Khi mạch mất điện, RAM bị xóa trắng. Thuật toán sẽ gửi lệnh `GET` lấy dữ liệu cũ từ Firebase đắp lại vào RAM trước khi cho máy hoạt động, chống lỗi ghi đè dữ liệu (Data Override).

### 5. Watchdog & Tự phục hồi mạng (TaskWiFiMonitor)
* **Xử lý Sóng WiFi Ảo (Fake WiFi):** Việc kiểm tra `WiFi.status()` là không đủ vì người dùng có thể rút cáp LAN của Router (vẫn có sóng WiFi nhưng mất Internet).
* **TCP Ping Layer 4:** Task Watchdog đóng vai trò trinh sát, định kỳ mở Socket TCP kết nối đến Google DNS (`8.8.8.8` Cổng `53`). Nếu Timeout, hệ thống ngay lập tức nhận diện trạng thái `SYSTEM_OFFLINE` và khóa máy giao dịch.
* **Driver Reset:** Nếu rớt sóng vật lý, hệ thống không phụ thuộc vào `setAutoReconnect` của Hệ điều hành. Nó gọi `WiFi.disconnect()` và `WiFi.begin()` để "đạp" Driver WiFi reset lại trạng thái, chống kẹt Driver nội bộ của chip.

### 6. Quản lý Năng lượng & Đi ngủ Thực dụng (Power Management)
* **TFT Backlight Hardware Control:** Màn hình được kết nối qua chân `GPIO 22`. Thay vì vẽ màu đen, hệ thống ngắt toàn bộ điện áp đèn nền (Backlight), giảm dòng tiêu thụ từ ~150mA xuống 0mA.
* **DTIM WiFi Modem Sleep:** Bật `WiFi.setSleep(true)`. Tắt lõi Radio của vi điều khiển. Ăng-ten chỉ "hé thức" 2 mili-giây mỗi chu kỳ 300ms (DTIM Interval) để nhận Beacon gói tin. Dòng tiêu thụ của WiFi giảm từ 100mA xuống dưới 5mA.
* **UART Wakeup:** Khi STM32 truyền tín hiệu qua UART, trạng thái `IDLE` bị phá vỡ, ESP32 "bừng sáng" màn hình ngay lập tức mà không làm ảnh hưởng đến bộ đếm `Tick Timer` của FreeRTOS.

---

## 🛠 Hướng dẫn Cài đặt & Build (Dành cho Developer)

### Môi trường
* Khuyến nghị sử dụng **VS Code + PlatformIO Extension**.
* Framework: `Arduino`
* Board: `esp32doit-devkit-v1`
* Phân vùng (Partition Scheme): `Huge APP (3MB No OTA/1MB SPIFFS)` để đủ dung lượng biên dịch các chứng chỉ SSL.

### Thư viện phụ thuộc (Dependencies)
Cấu hình trong `platformio.ini`:
```ini
[env:esp32doit-devkit-v1]
platform = espressif32@^6.8.1
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 115200
board_build.partitions = huge_app.csv

; Vô hiệu hóa tính năng thừa của Firebase để giải phóng RAM
build_flags = 
	-D FIREBASE_DISABLE_GCS
	-D FIREBASE_DISABLE_FCM
	-D FIREBASE_DISABLE_FIRESTORE
	-D FIREBASE_DISABLE_FUNCTIONS

lib_ignore = WiFiProv ; Ép bỏ qua thư viện WiFiProv nội bộ của ESP-IDF để tránh Header Clash

lib_deps = 
	moononournation/GFX Library for Arduino @ 1.4.6
	bblanchon/ArduinoJson @ ^6.21.3
	bodmer/TJpg_Decoder @ ^1.1.0
	mobizt/Firebase Arduino Client Library for ESP8266 and ESP32 @ ^4.4.10
	ricmoo/QRCode @ ^0.0.1
	tzapu/WiFiManager @ ^2.0.16-rc.2
