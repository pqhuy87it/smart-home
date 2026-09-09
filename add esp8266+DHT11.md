Tốt, DHT11 dùng để test toàn bộ chuỗi thì hoàn hảo — rẻ, có sẵn, và nếu nó lên được thì mọi thứ khác cũng sẽ lên.

Một lưu ý về độ chính xác để anh không nhầm sau này: DHT11 cho **±2°C, ±5%RH và độ ẩm chỉ trả về số nguyên**. Đủ để xác nhận hệ thống chạy, không đủ để dùng thật lâu dài. Khi nào chốt xong thì thay bằng SHT30 (~50k) là xong, config chỉ đổi vài dòng.

## Bước 1 — Đấu dây

Cấp nguồn cho DHT11 từ **3.3V**, không phải 5V. GPIO của ESP8266 **không chịu được 5V**, và nếu cấp 5V cho DHT11 thì chân DATA sẽ đưa mức 5V vào GPIO. DHT11 hoạt động bình thường ở 3.3V.

| DHT11 | ESP8266 (D1 mini / NodeMCU) |
|---|---|
| VCC / `+` | **3V3** |
| GND / `-` | GND |
| DATA / `S` | **D2** (= GPIO4) |

Chọn GPIO4 vì nó không phải strapping pin. **Tránh GPIO0, GPIO2, GPIO15** — chúng quyết định chế độ boot, và nếu DHT11 kéo mức logic sai lúc reset thì board không khởi động được.

Nếu anh dùng **module DHT11 3 chân** (loại có PCB nhỏ) thì đã có điện trở pull-up 10kΩ sẵn, không cần thêm gì. Nếu là **cảm biến 4 chân trần** thì phải tự mắc **10kΩ giữa DATA và 3V3**, nếu không sẽ đọc ra NaN liên tục.

## Bước 2 — Tạo file config

Trên Pi:

```bash
mkdir -p ~/smarthome/esphome/common
```

File dùng chung cho các node ESP8266 — `~/smarthome/esphome/common/base-8266.yaml`:

```yaml
esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

esp8266:
  board: ${board}
  restore_from_flash: false

logger:
  level: INFO
  baud_rate: 115200

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # ESP8266 rất dễ rớt WiFi nếu bật power save
  power_save_mode: none
  reboot_timeout: 15min
  ap:
    ssid: ${device_name}-setup
    password: !secret ota_password

captive_portal:

sensor:
  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 120s
    entity_category: diagnostic
  - platform: uptime
    name: "Uptime"
    update_interval: 120s
    entity_category: diagnostic

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "IP"
      entity_category: diagnostic

button:
  - platform: restart
    name: "Restart"
    entity_category: config
```

File node — `~/smarthome/esphome/sensor-t2.yaml`:

```yaml
substitutions:
  device_name: sensor-t2
  friendly_name: "Cảm biến tầng 2"
  board: d1_mini        # nodemcuv2 | esp01_1m | esp12e — chọn đúng board của anh

packages:
  base: !include common/base-8266.yaml

sensor:
  - platform: dht
    pin: GPIO4
    model: DHT11        # BẮT BUỘC ghi rõ, mặc định ESPHome đoán là DHT22
    update_interval: 60s
    temperature:
      name: "Nhiệt độ"
      filters:
        - filter_out: nan
        - median:
            window_size: 3
            send_every: 3
    humidity:
      name: "Độ ẩm"
      filters:
        - filter_out: nan
        - median:
            window_size: 3
            send_every: 3
```

`filter_out: nan` quan trọng với DHT11 — nó thỉnh thoảng trả về giá trị lỗi, và nếu không lọc thì HA sẽ hiện `unavailable` nhấp nháy.

## Bước 3 — Sinh api_key

Mở `http://exlinct.local:6052`, bấm **New Device** → **Continue** → nhập tên `sensor-t2` → nó sẽ sinh ra một `api.encryption.key`. Copy key đó vào `~/smarthome/esphome/secrets.yaml`:

```yaml
wifi_ssid: "TenWifi"
wifi_password: "MatKhauWifi"
ota_password: "mat_khau_ota"
api_key: "key_base64_vua_sinh_ra"
```

ESPHome sẽ tự tạo một file `sensor-t2.yaml` mặc định — **ghi đè nó bằng nội dung ở bước 2**. Sửa trực tiếp trong dashboard hoặc qua VS Code Remote-SSH.

## Bước 4 — Flash lần đầu (phải qua USB)

Cách dễ nhất: **cắm board vào cổng USB của Pi**. Kernel Linux có sẵn driver CH340/CP2102, không cần cài gì.

```bash
# Xác nhận Pi nhận board
ls -l /dev/ttyUSB*
dmesg | tail -5
```

Rồi trong ESPHome dashboard: card `sensor-t2` → **Install** → **Plug into the computer running ESPHome Dashboard** → chọn `/dev/ttyUSB0`.

Container ESPHome đã được mount `/dev` và chạy `privileged` nên nó thấy được port. Nếu dropdown trống, restart container:

```bash
docker compose restart esphome
```

**Cách thay thế** nếu muốn flash từ Mac mini: chọn **Manual download** → tải file `.factory.bin` → mở `web.esphome.io` bằng **Chrome hoặc Edge** (Safari không hỗ trợ WebSerial) → Connect → Install. macOS 11+ đã có driver CH34x sẵn.

Từ lần thứ hai trở đi thì **OTA qua WiFi**, không cần cắm dây nữa.

## Bước 5 — Xác nhận

Trong dashboard bấm **Logs**. Chuỗi log đúng:

```
[C][wifi:xxx]: WiFi:
[C][wifi:xxx]:   Local MAC: XX:XX:...
[C][wifi:xxx]:   IP Address: 192.168.1.x
[C][wifi:xxx]:   Signal strength: -58 dB
[C][dht:xxx]: DHT:
[C][dht:xxx]:   Model: DHT11
[D][dht:xxx]: Got temperature=28.0°C humidity=65.0%
[D][sensor:xxx]: 'Nhiệt độ': Sending state 28.00000 °C
```

Nếu thấy `Requesting data from DHT failed!` thì kiểm tra theo thứ tự: sai chân GPIO → thiếu pull-up 10kΩ → cấp nguồn 5V thay vì 3.3V → dây jumper lỏng.

## Bước 6 — Thêm vào Home Assistant

HA sẽ **tự phát hiện** node qua mDNS trong vòng một hai phút. Vào `http://exlinct.local:8123`, sẽ có notification "New device discovered: sensor-t2". Bấm Configure → dán `api_key` → Submit.

Nếu không thấy: Settings → Devices & Services → Add Integration → **ESPHome** → nhập IP của board (`192.168.1.x`), port `6053`, rồi dán key.

Xong là anh có 5 entity: nhiệt độ, độ ẩm, RSSI, uptime, IP.

Lưu ý: node này **không dùng MQTT chút nào**. ESPHome nói trực tiếp với HA qua native API port 6053. Mosquitto mình vừa dựng là để dành cho các board firmware anh tự viết.

## Bước 7 — Đặt DHCP reservation

Sau khi board lên, lấy MAC từ log ESPHome rồi vào router đặt reservation, giống cách anh làm với K1C. Với ESPHome thì mDNS thường đủ, nhưng IP cố định giúp anh debug nhanh hơn khi mDNS chập chờn.

## Ba điều nên biết về ESP8266

1. **Chỉ có ~40KB RAM khả dụng.** API encryption ăn khá nhiều. Nếu board reboot ngẫu nhiên hoặc log báo hết heap, bỏ khối `api.encryption` đi (chỉ dùng `api:` trơn) — vẫn chạy tốt trong LAN nhà.

2. **Đặt cảm biến xa board.** Module ESP8266 tự sinh nhiệt 3–5°C. Kéo DHT11 ra bằng dây 10–15cm, nếu không số nhiệt độ sẽ luôn cao hơn thực tế.

3. **Nguồn phải khoẻ.** ESP8266 hút tới 300mA lúc phát WiFi. Sạc điện thoại cũ 500mA hay cắm vào USB hub yếu sẽ gây reboot ngẫu nhiên mà không có log gì. Thêm **tụ 470µF** ở chân 3V3–GND nếu thấy bất ổn.

Khi node này chạy ổn định vài giờ, anh có nền tảng đã kiểm chứng để nhân bản ra các tầng và sang node bể nước. Cần tôi viết luôn dashboard HA để xem các node theo tầng chưa?
