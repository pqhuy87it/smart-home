Được — Pi 5 là lựa chọn tốt. Anh đã có Pi Zero 2 rồi nên chắc quen với hệ sinh thái này, nhưng Zero 2 quá yếu cho Home Assistant nên Pi 5 là bước nâng cấp hợp lý.

Tôi viết từ danh sách mua đến khi stack chạy được.

---

# Phần 1 — Danh sách phần cứng (BOM)

## Bắt buộc

| # | Món | Chọn gì | Giá tham khảo |
|---|---|---|---|
| 1 | **Raspberry Pi 5** | Bản **8GB**. 4GB đủ cho HA + Z2M nhưng sẽ chật khi anh thêm InfluxDB, Frigate, ESPHome... 16GB thì thừa | ~2,5–3 tr |
| 2 | **Nguồn chính hãng 27W USB-C PD** | **Đừng tiết kiệm chỗ này** — xem giải thích bên dưới | ~400–600k |
| 3 | **Active Cooler** (quạt chính hãng) hoặc case có quạt | Pi 5 nóng thật, không phải như Pi 4 | ~150–250k |
| 4 | **Zigbee coordinator** | Sonoff ZBDongle-P (CC2652P) | ~500–700k |
| 5 | **Cáp nối dài USB 2.0, 1–2m** | Bắt buộc, không phải tuỳ chọn | ~50k |
| 6 | **Cáp Ethernet** | Nối thẳng vào AP/switch, đừng dùng WiFi | — |

## Lưu trữ — chọn 1 trong 3

| Phương án | Giá | Nhận xét |
|---|---|---|
| **M.2 HAT+ + NVMe 256GB** | ~600k + ~800k | Tốt nhất. NVMe nhanh gấp 50–100× thẻ microSD ở random 4K IOPS, và đó chính là loại truy cập mà database của HA và Docker sinh ra nhiều nhất |
| **SSD USB 3.0 (SATA)** | ~700k | Rẻ hơn, không cần HAT, đủ nhanh. Nhưng chiếm 1 cổng USB 3.0 và cáp lằng nhằng |
| **microSD A2 128GB** | ~300k | Chỉ để thử nghiệm. Database của HA ghi liên tục sẽ giết thẻ trong ~1 năm |

Tôi khuyên đi NVMe. Nếu chọn NVMe, cân nhắc mua luôn **case Argon NEO 5 M.2 NVMe** — nó gộp cả tản nhiệt lẫn khe NVMe vào một bộ, gọn hơn là chồng HAT lên nhau.

## Tuỳ chọn nên có

- **UPS nhỏ (500VA)** — Hà Nội thỉnh thoảng mất điện chớp nhoáng. Mất điện đột ngột lúc HA đang ghi SQLite là cách nhanh nhất để corrupt database. Một con UPS 500VA giữ được Pi + router/AP chạy tới 30 phút.

---

# Phần 2 — Ba cái bẫy phần cứng

## Bẫy 1: Nguồn không chính hãng → dongle Zigbee chập chờn

Đây là bẫy quan trọng nhất và ít người biết. Pi 5 tự dò công suất nguồn qua USB-C PD:

- Với **nguồn chính hãng 27W (5V/5A)** → Pi cho phép tổng dòng USB tới **1.6A**.
- Với nguồn thường 5V/3A → Pi **giới hạn tổng dòng USB xuống 600mA**.

600mA phải chia cho dongle Zigbee + bất cứ thứ gì khác. Kết quả: dongle brown-out ngẫu nhiên, Z2M mất kết nối coordinator, và anh sẽ đi debug nhầm hướng suốt mấy ngày.

Có cờ `usb_max_current_enable=1` trong `config.txt` để ép mở khoá, nhưng nếu nguồn không thật sự cấp đủ thì Pi sẽ tự reset. Mua đúng nguồn 27W chính hãng rẻ hơn nhiều so với thời gian debug.

## Bẫy 2: Cắm dongle vào cổng USB 3.0

Pi 5 có 2 cổng **USB 3.0 (xanh)** và 2 cổng **USB 2.0 (đen)**. Cổng USB 3.0 phát nhiễu điện từ đúng dải 2.4GHz.

Cắm dongle Zigbee vào **cổng USB 2.0 màu đen, qua cáp nối dài**. NVMe/SSD thì dùng cổng xanh.

## Bẫy 3: WiFi onboard của Pi cũng là nguồn nhiễu

Radio 2.4GHz của Pi nằm cách dongle vài centimet. Dùng Ethernet và **tắt hẳn WiFi + Bluetooth onboard** — vừa giảm nhiễu vừa tiết kiệm ~0.5W. Anh đã chuyển các router tầng sang AP mode với subnet phẳng rồi nên cắm dây là tiện.

---

# Phần 3 — Lắp ráp

Thứ tự chồng (từ dưới lên): **Pi 5 → Active Cooler → M.2 HAT+**. Active Cooler bắt vít vào 2 lỗ mounting, HAT+ chồng lên trên qua standoff cao.**Vài lưu ý khi lắp:**

- **Cáp FFC PCIe** là chỗ dễ sai nhất. Kéo nhẹ chốt đen của connector lên, cắm cáp vào, ấn chốt xuống. Cáp chỉ vào đúng một chiều — nếu thấy phải dùng lực thì đang sai chiều. Kiểm tra cả hai đầu đã cắm sâu và chốt đã khoá.
- **Quạt Active Cooler** cắm vào jack JST 4-pin nhỏ nằm cạnh cổng Ethernet, không phải header GPIO.
- Standoff vặn tay vừa đủ, siết mạnh sẽ nứt PCB.

---

# Phần 4 — Cài hệ điều hành

## 4.1. Flash Raspberry Pi OS Lite 64-bit

Trên Mac mini M4, cài **Raspberry Pi Imager**. Nếu anh có adapter USB-to-NVMe thì cắm SSD vào Mac và ghi thẳng — nhanh nhất. Nếu không, ghi ra microSD trước rồi clone sang NVMe sau.

Chọn: **Raspberry Pi OS Lite (64-bit)** — bản Trixie (Debian 13). Lite, không desktop. Anh sẽ chỉ dùng SSH.

Bấm **⚙️ (Edit Settings)** trước khi ghi:

```
Hostname:        smarthome
Username:        huy
☑ Use SSH → Allow public-key authentication only
                 (dán nội dung ~/.ssh/id_ed25519.pub từ Mac)
☐ Configure wireless LAN     ← bỏ trống, dùng Ethernet
Locale:          Asia/Ho_Chi_Minh
Keyboard:        us
```

## 4.2. Boot lần đầu và bật boot từ NVMe

Nếu ghi thẳng vào NVMe, có thể Pi vẫn chưa biết boot từ đó. Cắm màn hình hoặc dùng microSD tạm, rồi:

```bash
# Cập nhật EEPROM lên bản mới nhất trước
sudo rpi-eeprom-update -a
sudo reboot

# Đặt thứ tự boot: NVMe → SD → USB
sudo raspi-config
#   6 Advanced Options → A4 Boot Order → B2 NVMe/USB Boot
```

Hoặc sửa trực tiếp:

```bash
sudo rpi-eeprom-config --edit
# tìm dòng BOOT_ORDER, đổi thành:
BOOT_ORDER=0xf416
# đọc từ phải sang trái: 6=NVMe, 1=SD, 4=USB, f=lặp lại
```

Rút microSD ra, reboot. Kiểm tra:

```bash
lsblk
# phải thấy root filesystem nằm trên nvme0n1p2
findmnt /
```

## 4.3. `/boot/firmware/config.txt`

```bash
sudo nano /boot/firmware/config.txt
```

Thêm vào cuối:

```ini
# Tắt WiFi + Bluetooth onboard: giảm nhiễu 2.4GHz cạnh dongle Zigbee
dtoverlay=disable-wifi
dtoverlay=disable-bt

# Bật NVMe qua M.2 HAT+
dtparam=nvme

# PCIe Gen 3 — KHÔNG chính thức, tăng tốc SSD nhưng có thể mất ổn định.
# Nếu SSD biến mất hoặc filesystem lỗi ngẫu nhiên, XOÁ dòng này trước tiên.
dtparam=pciex1_gen=3
```

Dòng Gen 3 đáng lưu ý: khi Pi 5 + NVMe rơi vào trạng thái treo hoặc SSD không nhận, việc đầu tiên cần làm là gỡ cấu hình Gen 3 ra. Nếu anh không cần tốc độ tối đa thì cứ để Gen 2 cho yên tâm — Home Assistant không cần 800MB/s.

## 4.4. Cập nhật và kiểm tra sức khoẻ

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y nvme-cli git vim htop

# Nhiệt độ — idle nên dưới 50°C với Active Cooler
vcgencmd measure_temp

# Kiểm tra có bị throttle / thiếu điện không. Kết quả tốt: throttled=0x0
vcgencmd get_throttled

# Tốc độ NVMe
sudo hdparm -t --direct /dev/nvme0n1
```

Nếu `get_throttled` khác `0x0`, bit 0 nghĩa là under-voltage → nguồn không đủ, quay lại Bẫy 1.

## 4.5. Cố định IP

Anh đã làm DHCP reservation cho con K1C rồi, làm y hệt cho Pi. Lấy MAC của cổng Ethernet:

```bash
ip link show eth0 | grep ether
```

Vào giao diện router chính (con đang làm DHCP server sau khi anh chuyển các router tầng sang AP mode), đặt reservation. Làm ở tầng router chứ đừng set static IP trên Pi — dễ quản lý hơn và tránh conflict.

---

# Phần 5 — Đặt Pi ở đâu

Nhà anh nhiều tầng nên chỗ đặt quyết định chất lượng mesh nhiều hơn cả loại dongle:

- Đặt Pi (và dongle) ở **tầng giữa**, không phải tầng trên cùng hay tầng hầm. Sóng Zigbee xuyên sàn bê tông kém.
- Dongle treo lơ lửng qua cáp nối dài, **cách xa Pi, router, AP, và bất cứ cục nguồn switching nào** ít nhất 50cm.
- Tránh để trong tủ kim loại hoặc tủ rack kín.
- Mỗi tầng nên có ít nhất **1–2 router Zigbee** (ổ cắm/công tắc cắm điện) để mesh nối được lên xuống. Đây là lý do tôi khuyên mua router trước sensor.

---

# Phần 6 — Cài stack

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
sudo usermod -aG dialout $USER
newgrp docker

mkdir -p ~/smarthome/{mosquitto/{config,data,log},zigbee2mqtt/data,homeassistant/config}
cd ~/smarthome
```

Từ đây dùng nguyên `docker-compose.yml`, `mosquitto.conf` và `configuration.yaml` tôi đã viết ở tin nhắn trước. Chỉ cần đổi đúng một chỗ — đường dẫn dongle:

```bash
ls -l /dev/serial/by-id/
```

rồi dán vào phần `devices:` của service `zigbee2mqtt`.

```bash
docker compose up -d
docker compose logs -f zigbee2mqtt
```

---

# Phần 7 — Checklist nghiệm thu

```bash
# 1. Boot từ NVMe, không phải SD
findmnt / | grep nvme

# 2. Không throttle
vcgencmd get_throttled          # → throttled=0x0

# 3. Nhiệt độ ổn dưới tải
vcgencmd measure_temp           # → < 60°C

# 4. WiFi/BT đã tắt
ip link | grep -E "wlan|hci"    # → không có kết quả

# 5. Dongle được nhận
ls -l /dev/serial/by-id/

# 6. Ba container chạy
docker compose ps

# 7. Z2M nối được MQTT
docker compose logs zigbee2mqtt | grep "Connected to MQTT"

# 8. Điện năng: đo bằng ổ cắm đo điện, idle nên ~5–7W
```

Truy cập: Z2M ở `http://smarthome.local:8080`, Home Assistant ở `http://smarthome.local:8123`.

---

# Vài thứ nên làm tiếp, theo đúng thứ tự

1. **Pair 2–3 ổ cắm Zigbee làm router trước**, rải các tầng. Đợi vài tiếng cho mesh ổn định.
2. **Backup ngay** `~/smarthome/zigbee2mqtt/data/` — đặc biệt `coordinator_backup.json`. Đẩy lên private repo, hoặc rsync sang Mac mini bằng cron.
3. **HomeKit Bridge** trong Home Assistant → toàn bộ thiết bị hiện trong app Home trên iPhone, chạy local, không cần cloud.
4. **Bambu A1 mini** của anh: HA có integration đọc LAN MQTT của máy in — nhiệt độ bed/nozzle, tiến độ, thời gian còn lại thành entity. Lúc đó con ESP32-C3 print monitor có thể subscribe từ Mosquitto thay vì nói chuyện thẳng với máy in, đỡ phải xử lý auth riêng.
5. **ESPHome add-on** — nếu anh muốn quản lý loạt board ESP32 hiện có tập trung, ESPHome sinh firmware và OTA từ chính Pi này.

Anh muốn tôi đi tiếp phần nào: script backup tự động, tích hợp A1 mini + ESP32 vào HA, hay viết automation YAML đầu tiên?
