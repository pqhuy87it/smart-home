Được, tôi viết thành guide đầy đủ từ mua đồ đến pair thiết bị đầu tiên.

---

# Phần A — Danh sách phần cứng

## A1. Máy chủ (host) — chạy 24/7

Đây là thứ quyết định trải nghiệm nhiều nhất. Ba lựa chọn thực tế:

| Lựa chọn | Ưu | Nhược |
|---|---|---|
| **Raspberry Pi 5 (4GB)** + SSD USB | Rẻ, ít điện (~5W), cộng đồng lớn | Cần SSD, đừng chạy HA trên thẻ SD (chết thẻ sau ~1 năm) |
| **Mini PC N100** (Beelink/GMKtec) | Mạnh hơn Pi nhiều, có NVMe sẵn, giá không chênh nhiều | ~10–15W |
| **Home Assistant Green** | Cắm là chạy, không phải setup gì | Đắt hơn, phần cứng yếu, không linh hoạt |

**Cảnh báo quan trọng cho anh** (vì anh làm trên macOS): **đừng định chạy Zigbee2MQTT trong Docker trên Mac.** Docker Desktop trên macOS chạy container trong một LinuxKit VM và **không hỗ trợ USB passthrough** — container sẽ không bao giờ thấy được dongle. Nếu bắt buộc phải dùng Mac thì có 2 đường:
- Cài Z2M **native** bằng Node.js + pnpm trực tiếp trên macOS (chạy được, nhưng Mac của anh phải bật 24/7).
- Hoặc dùng **coordinator qua Ethernet** (SLZB-06) — lúc đó không còn USB nữa, Z2M kết nối qua TCP socket, chạy trong Docker trên Mac thoải mái.

Khuyến nghị của tôi: mua một con mini PC N100 hoặc Pi 5 riêng. Smart home cần uptime, không nên dính vào máy làm việc.

## A2. Coordinator dongle — bắt buộc

Chọn 1 trong:

- **Sonoff ZBDongle-P** (CC2652P, firmware zStack) — an toàn nhất cho người mới, nhiều tài liệu nhất.
- **Sonoff ZBDongle-E** (EFR32MG21, firmware EmberZNet, driver `ember`) — vẫn là lựa chọn đáng giá nhất về tiền/hiệu năng, đã chạy ổn định trên hàng nghìn hệ thống suốt hơn hai năm.
- **SMLIGHT SLZB-06** — bản Ethernet/PoE, chọn nếu server nằm ở góc nhà tệ (tủ rack, gầm cầu thang) hoặc anh muốn chạy trên Mac.

## A3. Phụ kiện không được bỏ qua

- **Cáp nối dài USB 2.0, dài 1–2m.** Bắt buộc, không phải tuỳ chọn. Cắm dongle thẳng vào máy = nhiễu, rớt thiết bị ngẫu nhiên, và anh sẽ debug nhầm hướng cả tuần.
- **SSD / NVMe** cho host. Đừng dùng thẻ nhớ.

## A4. Thiết bị Zigbee đầu tiên nên mua gì

Nguyên tắc: **mua router (thiết bị cắm điện) trước, sensor pin sau.** Router là xương sống của mesh.

Bộ khởi đầu hợp lý:
1. **2–3 ổ cắm thông minh Zigbee** hoặc **công tắc âm tường Zigbee** — đây là router, đặt rải ra các phòng.
2. **1 cảm biến cửa** (contact sensor) — để test pairing, rẻ nhất.
3. **1 cảm biến chuyển động** — Aqara P1 hoặc loại mmWave nếu anh muốn chơi sâu.
4. **1 cảm biến nhiệt độ/độ ẩm**.

Ở thị trường VN thì đồ **Tuya/Zemismart/Moes** bán đầy trên Shopee/Lazada và rất rẻ, Z2M hỗ trợ hầu hết. **Aqara** đắt hơn nhưng chất lượng build và độ ổn định tốt hơn rõ rệt. Trước khi bấm mua bất cứ món nào, tra model trên `zigbee2mqtt.io/supported-devices` — mất 30 giây và tránh được món đồ vô dụng.

Một lưu ý: nhiều thiết bị Tuya trông giống hệt nhau nhưng khác `manufacturerName` bên trong. Nếu mua nhầm loại chưa được hỗ trợ, anh vẫn cứu được bằng cách viết **external converter** (một file JS) — tôi có thể hướng dẫn riêng phần đó.

---

# Phần B — Chọn đường cài đặt

**Đường 1 — Home Assistant OS + add-on.** Flash HAOS lên máy, vào Add-on Store cài Mosquitto broker và Zigbee2MQTT. Nhanh nhất, ít lỗi nhất. Nếu anh chỉ muốn nó chạy thì đi đường này.

**Đường 2 — Docker Compose.** Kiểm soát hoàn toàn, dễ version control, dễ backup, đúng gu dev. Tôi viết chi tiết đường này bên dưới.

---

# Phần C — Cài bằng Docker Compose

## C1. Chuẩn bị host

```bash
# Cài Docker (Debian/Ubuntu/Raspberry Pi OS)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Cấp quyền truy cập serial port
sudo usermod -aG dialout $USER

# Tạo cấu trúc thư mục
mkdir -p ~/smarthome/{mosquitto/{config,data,log},zigbee2mqtt/data,homeassistant/config}
cd ~/smarthome
```

## C2. Tìm đúng đường dẫn dongle

Cắm dongle qua cáp nối dài, rồi:

```bash
ls -l /dev/serial/by-id/
```

Kết quả kiểu:
```
usb-ITead_Sonoff_Zigbee-3.0_USB_Dongle_Plus_a1b2c3d4-if00-port0 -> ../../ttyUSB0
```

**Luôn dùng đường `/dev/serial/by-id/...`**, không bao giờ dùng `/dev/ttyUSB0` — số thứ tự đó đổi sau mỗi lần reboot và Z2M sẽ chết.

## C3. Chọn Zigbee channel

Zigbee và WiFi 2.4GHz đè lên nhau. Quét xem WiFi nhà anh đang ở channel nào:

```bash
sudo iw dev wlan0 scan | grep -E "SSID|DS Parameter"
```

Rồi chọn Zigbee channel nằm vào khe trống:

| WiFi đang dùng | Chọn Zigbee channel |
|---|---|
| 1 | 20 hoặc 25 |
| 6 | 15 hoặc 25 |
| 11 | 15 hoặc 20 |
| Nhiều WiFi lộn xộn (chung cư) | **25** |

Ở chung cư Hà Nội thì band 2.4GHz thường rất chật, tôi khuyên chọn **channel 25**. Tránh channel 26 vì một số thiết bị cũ (Xiaomi đời đầu) không join được.

Chốt channel **ngay từ đầu** — đổi sau sẽ phải pair lại toàn bộ thiết bị.

## C4. `mosquitto/config/mosquitto.conf`

```conf
listener 1883
protocol mqtt

persistence true
persistence_location /mosquitto/data/
log_dest stdout

# Không cho phép anonymous — quan trọng, đừng bỏ qua
allow_anonymous false
password_file /mosquitto/config/passwd
```

Tạo user cho Z2M và cho Home Assistant:

```bash
# Tạo file passwd với user đầu tiên (-c chỉ dùng lần đầu)
docker run --rm -it -v ~/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 mosquitto_passwd -c /mosquitto/config/passwd z2m

# Thêm user thứ hai (bỏ -c)
docker run --rm -it -v ~/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 mosquitto_passwd /mosquitto/config/passwd hass
```

## C5. `docker-compose.yml`

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    networks:
      - smarthome

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8080:8080"          # Web UI của Z2M
    volumes:
      - ./zigbee2mqtt/data:/app/data
      - /run/udev:/run/udev:ro
    devices:
      # Thay bằng đường by-id của anh ở bước C2.
      # Vế phải luôn để /dev/ttyACM0 cho khớp configuration.yaml
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee-3.0_USB_Dongle_Plus_a1b2c3d4-if00-port0:/dev/ttyACM0
    environment:
      - TZ=Asia/Ho_Chi_Minh
    networks:
      - smarthome

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    # network_mode host là bắt buộc để HA làm được mDNS discovery
    # (tìm Chromecast, HomeKit, ESPHome... trong LAN)
    network_mode: host
    privileged: true
    volumes:
      - ./homeassistant/config:/config
      - /run/dbus:/run/dbus:ro
    environment:
      - TZ=Asia/Ho_Chi_Minh

networks:
  smarthome:
    driver: bridge
```

Lưu ý một chi tiết dễ vấp: vì `homeassistant` dùng `network_mode: host` nên nó **không nằm chung bridge network** với hai container kia. Khi cấu hình MQTT integration trong HA, broker address phải là **IP LAN của máy host** (ví dụ `192.168.1.50`), không phải `mosquitto`.

## C6. `zigbee2mqtt/data/configuration.yaml`

Bản Z2M mới có onboarding wizard chạy lần đầu, nhưng tạo sẵn file này thì gọn hơn:

```yaml
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883    # dùng service name, cùng bridge network
  user: z2m
  password: 'mat_khau_da_tao_o_C4'

serial:
  port: /dev/ttyACM0
  # zstack  -> Sonoff ZBDongle-P (CC2652P)
  # ember   -> Sonoff ZBDongle-E / MG24 (EFR32)
  adapter: zstack

frontend:
  enabled: true
  port: 8080

homeassistant:
  enabled: true                    # bật MQTT Discovery, HA tự tạo entity

advanced:
  channel: 25
  network_key: GENERATE            # tự sinh key ngẫu nhiên lần đầu
  pan_id: GENERATE
  ext_pan_id: GENERATE
  log_level: info
  # Cho phép join lại thiết bị bằng install code nếu có
  transmit_power: 20               # chỉ có tác dụng với chip "P"

availability:
  enabled: true                    # theo dõi thiết bị online/offline

# Mặc định TẮT — chỉ bật tạm khi cần thêm thiết bị mới
permit_join: false
```

Sau lần chạy đầu, Z2M sẽ ghi `network_key`/`pan_id` thật vào file này. **Backup file đó ngay** — mất nó là mất cả mạng Zigbee, phải pair lại tất cả.

## C7. Khởi động và kiểm tra

```bash
docker compose up -d
docker compose logs -f zigbee2mqtt
```

Log tốt trông như thế này:
```
info:  z2m: Logging to console
info:  zh:zstack:znp: Opening SerialPort with ...
info:  z2m: zigbee-herdsman started (resumed)
info:  z2m: Coordinator firmware version: '...'
info:  z2m: Connecting to MQTT server at mqtt://mosquitto:1883
info:  z2m: Connected to MQTT server
info:  z2m: Zigbee2MQTT started!
```

Nếu thấy `Error: Adapter not found` → sai đường `devices:` hoặc thiếu quyền `dialout`.
Nếu container restart loop kèm `MQTT failed to connect` → sai user/password hoặc `mosquitto.conf` không được mount đúng chỗ (dùng file default nên chặn kết nối từ container khác).

Kiểm tra bằng tay:
```bash
docker exec -it mosquitto mosquitto_sub -u z2m -P 'mat_khau' -t 'zigbee2mqtt/#' -v
```

---

# Phần D — Pair thiết bị đầu tiên

1. Mở `http://<ip-host>:8080` — đây là Z2M frontend.
2. Cắm **ổ cắm/công tắc Zigbee** (router) trước tiên, đặt cách coordinator vài mét.
3. Bấm **Permit join (All)** ở góc trên bên phải. Nó có countdown, mặc định 254 giây.
4. Đưa thiết bị về chế độ pairing (thường là giữ nút 5–10 giây tới khi đèn nháy nhanh — đọc manual từng loại).
5. Thiết bị xuất hiện trong tab Devices → **đổi friendly name ngay** thành thứ có nghĩa (`o_cam_phong_khach`), vì topic MQTT sẽ dùng tên đó.
6. **Tắt permit join** sau khi xong. Để mở là lỗ hổng bảo mật.
7. Lặp lại với các router còn lại, xong mới tới sensor pin.
8. Vào tab **Map** xem topology mesh — anh sẽ thấy sensor nào đang đi qua router nào.

---

# Phần E — Một vài thứ nên làm ngay sau khi chạy được

- **Backup**: `zigbee2mqtt/data/` (đặc biệt `configuration.yaml` và `coordinator_backup.json`) + `homeassistant/config/`. Đẩy lên git private repo, trừ file password.
- **HomeKit**: vì anh làm iOS, thêm integration **HomeKit Bridge** trong Home Assistant → toàn bộ thiết bị Zigbee sẽ xuất hiện trong app Home của iPhone, điều khiển bằng Siri, mà vẫn hoàn toàn local. Đây là cách rẻ nhất để có "HomeKit-compatible" cho đồ Tuya 100k.
- **Đừng cho Home Assistant ra internet trực tiếp**. Dùng Tailscale hoặc WireGuard nếu cần truy cập từ xa.
- **Đợi 24–48 giờ** trước khi kết luận mạng có ổn định không. Mesh cần thời gian tự tối ưu route.

---

Anh muốn tôi làm tiếp phần nào: viết external converter cho một thiết bị Tuya cụ thể, cấu hình automation trong HA (YAML), hay hướng dùng ESP32-C6 làm Zigbee device tự chế để nối vào mạng này?
