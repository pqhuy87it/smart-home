Đây là sơ đồ đấu nối cho hai cảm biến này. Tôi bỏ SHT30 ra như anh nói, nhưng vẫn để dành chân I2C để thêm sau mà không phải đấu lại.## Bảng đấu chân

| Từ | Chân | Đến | Chân | Ghi chú |
|---|---|---|---|---|
| Adapter | 5V | ESP32-C3 | 5V / VBUS | LDO onboard hạ xuống 3V3 |
| Adapter | 5V | SDS011 | 5V | **Không lấy từ chân 3V3** |
| Adapter | GND | GND chung | — | Nối một điểm duy nhất |
| ESP32-C3 | 3V3 | SCD41 | VDD | |
| ESP32-C3 | GPIO8 | SCD41 | SDA | |
| ESP32-C3 | GPIO9 | SCD41 | SCL | |
| ESP32-C3 | GPIO20 (RX) | SDS011 | **TXD** | |
| ESP32-C3 | GPIO21 (TX) | SDS011 | **RXD** | Bắt buộc — dùng để set working period |

Chú ý cặp UART: **TXD của cảm biến đi vào RX của board**, chéo nhau. Đây là lỗi đấu sai phổ biến nhất, và triệu chứng là SDS011 im lặng hoàn toàn.

## Năm điểm cần làm đúng

**1. SDS011 phải lấy 5V từ adapter, không qua board.** Chân 3V3 trên C3 SuperMini chỉ cấp được khoảng 400–600mA từ LDO, và LDO đó đang phải gánh cả C3. Quạt SDS011 khởi động kéo dòng đột ngột sẽ làm sụt áp và reset board.

**2. Hai tụ bulk là bắt buộc, không phải tuỳ chọn.** SCD41 hút 205mA trong 200ms mỗi lần đo, quạt SDS011 có dòng khởi động cao. Nếu hai đỉnh này trùng nhau mà không có tụ, board reboot ngẫu nhiên và anh sẽ debug sai hướng cả buổi.

**3. Chân TX phải nối.** Nếu chỉ nối RX một chiều thì ESPHome không gửi được lệnh working period, và SDS011 sẽ chạy quạt liên tục — hỏng trong 11 tháng thay vì 18 năm.

**4. Đặt SCD41 xa nguồn nhiệt.** Kéo ra bằng dây 10–15cm, tránh cả module C3 và SDS011 (quạt thổi khí ấm). Điều này quyết định giá trị `temperature_offset` mà anh phải hiệu chuẩn sau.

**5. Không che lỗ hút/thoát khí của SDS011.** Nó cần luồng khí tự do. Nếu đặt trong hộp, khoét hai lỗ thẳng hàng với lỗ của cảm biến, và đừng để nó hút phải khí nóng do chính board thải ra.

## Chân dự phòng cho SHT30

Khi nào anh thêm SHT30, chỉ cần mắc **song song lên đúng GPIO8/GPIO9** — không tốn chân mới, vì địa chỉ 0x44 không đụng 0x62 của SCD41. Nên khi đi dây bây giờ, để sẵn hai điểm nối trên đường SDA/SCL là sau này cắm thêm rất nhanh.

## Test I2C trước khi lắp cố định

Cắm tạm trên breadboard, flash firmware với `scan: true`, rồi xem log lúc boot:

```
[C][i2c.idf:xxx]: Results from i2c bus scan:
[C][i2c.idf:xxx]: Found i2c device at address 0x62
```

Nếu không thấy `0x62` thì SDA/SCL đang bị đảo — đổi hai dây cho nhau. Chạy bus scan trên breadboard trước khi hàn cố định tiết kiệm rất nhiều thời gian.

---

Config cho node `air-t1` với đúng hai cảm biến, khớp sơ đồ chân vừa rồi.

## `air-t1.yaml`

```yaml
esphome:
  name: air-t1
  friendly_name: "Không khí tầng 1"

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

logger:
  level: INFO
  # Đẩy log qua USB-CDC để UART0 rảnh cho SDS011.
  # Nếu board của anh không phải USB native thì đổi thành:
  #   baud_rate: 0   (tắt log serial hoàn toàn)
  hardware_uart: USB_CDC

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
  reboot_timeout: 15min
  ap:
    ssid: "air-t1-setup"
    password: !secret ota_password

captive_portal:

# ============ BUS I2C — SCD41 (chỗ trống sẵn cho SHT30 sau này) ============
i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true
  frequency: 50kHz        # SCD4x tối đa 100kHz; 50kHz ổn định hơn khi dây dài

# ============ UART — SDS011 ============
uart:
  id: uart_sds
  tx_pin: GPIO21          # → RXD của SDS011, cần để set working period
  rx_pin: GPIO20          # ← TXD của SDS011
  baud_rate: 9600

sensor:
  # ---------------- SCD41: CO2 ----------------
  - platform: scd4x
    id: scd41
    address: 0x62
    update_interval: 60s
    automatic_self_calibration: true
    temperature_offset: 6.0 °C     # HIỆU CHUẨN LẠI sau 1 giờ chạy, xem cuối bài
    altitude_compensation: 16m     # Hà Nội
    co2:
      name: "CO2"
      id: co2_value
      filters:
        - filter_out: nan
    # Chưa có SHT30 nên tạm đưa T/H của SCD41 vào HA.
    # Khi thêm SHT30, đổi hai cái này thành internal: true
    temperature:
      name: "Nhiệt độ"
      id: temp_value
      filters:
        - filter_out: nan
    humidity:
      name: "Độ ẩm"
      id: humi_value
      filters:
        - filter_out: nan

  # ---------------- SDS011: bụi mịn ----------------
  - platform: sds011
    uart_id: uart_sds
    # Đơn vị PHÚT (0–30). Cảm biến tự ngủ rồi thức 30s để đo.
    update_interval: 10min
    pm_2_5:
      name: "PM2.5"
      id: pm25_value
      filters:
        - filter_out: nan
    pm_10_0:
      name: "PM10"
      id: pm10_value
      filters:
        - filter_out: nan

  # ---------------- Diagnostics ----------------
  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 120s
    entity_category: diagnostic

  - platform: uptime
    name: "Uptime"
    update_interval: 120s
    entity_category: diagnostic

  - platform: internal_temperature
    name: "Nhiệt độ chip"
    update_interval: 300s
    entity_category: diagnostic

# ============ Diễn giải thành hành động ============
text_sensor:
  - platform: template
    name: "Nên làm gì"
    icon: "mdi:window-open-variant"
    update_interval: 60s
    lambda: |-
      if (isnan(id(co2_value).state) || isnan(id(pm25_value).state))
        return {"Đang chờ dữ liệu"};
      bool bi  = id(co2_value).state  > 1200;   // phòng bí
      bool bui = id(pm25_value).state > 35;     // ngoài trời bẩn
      if (bi && bui)  return {"Bật lọc khí, giữ cửa đóng"};
      if (bi)         return {"Mở cửa sổ"};
      if (bui)        return {"Đóng cửa, bật lọc khí"};
      return {"Không khí ổn"};

  - platform: template
    name: "Mức bụi"
    icon: "mdi:blur"
    update_interval: 60s
    lambda: |-
      if (isnan(id(pm25_value).state)) return {"Chưa có dữ liệu"};
      float p = id(pm25_value).state;
      if (p <= 12)  return {"Tốt"};
      if (p <= 35)  return {"Trung bình"};
      if (p <= 55)  return {"Kém"};
      if (p <= 150) return {"Xấu"};
      return {"Rất xấu"};

  - platform: wifi_info
    ip_address:
      name: "IP"
      entity_category: diagnostic

binary_sensor:
  - platform: template
    name: "Nên mở cửa sổ"
    device_class: problem
    lambda: |-
      if (isnan(id(co2_value).state) || isnan(id(pm25_value).state)) return {};
      return id(co2_value).state > 1200 && id(pm25_value).state < 35;

button:
  - platform: restart
    name: "Restart"
    entity_category: config

  # Bấm khi đã để cảm biến ngoài trời ≥ 5 phút
  - platform: template
    name: "Hiệu chuẩn CO2 (ngoài trời)"
    entity_category: config
    on_press:
      - scd4x.perform_forced_calibration:
          value: 420
          id: scd41
```

## Tạo device trong dashboard

Lần này bỏ qua wizard cho nhanh — tạo file trực tiếp:

```bash
cd ~/smarthome/esphome
nano air-t1.yaml     # dán config trên
```

Nhưng `api_key` của node này **phải khác** node `sensor-t1`. Mỗi device cần key riêng. Sinh một key mới:

```bash
openssl rand -base64 32
```

Rồi thêm vào Shared secrets trong dashboard với tên `api_key_air_t1`, và sửa dòng trong config:

```yaml
api:
  encryption:
    key: !secret api_key_air_t1
```

Nếu anh thấy rườm rà thì cứ chạy wizard **Add new device** như lần trước (nhập tên `air-t1`, chọn ESP32-C3), để nó tự sinh key, rồi dán đè phần còn lại của config. Cách nào cũng được.

## Log cần thấy sau khi flash

```
[C][i2c.idf:xxx]: Results from i2c bus scan:
[C][i2c.idf:xxx]: Found i2c device at address 0x62
[C][scd4x:xxx]: scd4x
[C][scd4x:xxx]:   Automatic self calibration: ON
[C][scd4x:xxx]:   Temperature offset: 6.00 °C
[C][sds011:xxx]: SDS011:
[C][sds011:xxx]:   Update Interval: 10 min
```

Bốn thứ phải đúng: **có `0x62`**, **ASC ON**, **`Update Interval: 10 min`** (không phải `0 min`), và sau khoảng 1 phút phải có dòng `Sending state` cho CO2.

Dữ liệu PM2.5 đầu tiên sẽ đến **sau 10 phút** — đừng sốt ruột tưởng nó lỗi. Nếu muốn test nhanh, tạm đặt `update_interval: 1min`, xác nhận có dữ liệu, rồi OTA lại với `10min`.

## Bảng tra lỗi

| Triệu chứng | Nguyên nhân |
|---|---|
| Không thấy `0x62` | SDA/SCL bị đảo — đổi hai dây |
| `Update Interval: 0 min` | Chân TX chưa nối vào RXD của SDS011 |
| PM2.5 luôn `unknown` | TXD/RXD đảo, hoặc SDS011 đang được cấp 3.3V thay vì 5V |
| Board reboot ngẫu nhiên | Thiếu tụ bulk, hoặc SDS011 lấy nguồn từ chân 3V3 của board |
| CO2 đọc ~400 cố định, không đổi | Buồng đo bị bịt kín — cần khe thông khí |
| Log toàn ký tự rác | `hardware_uart: USB_CDC` không đúng với board — thử `baud_rate: 0` |

## Hiệu chuẩn `temperature_offset`

Để chạy **1 giờ** cho ổn định nhiệt, rồi đo nhiệt độ phòng bằng nhiệt kế độc lập (hoặc con `sensor-t1` với DHT11 đặt cùng phòng). So với entity "Nhiệt độ" của node này:

```
offset_mới = 6.0 + (SCD41 đọc − nhiệt độ thật)
```

Thường ra 8–12°C khi SCD41 nằm gần module. Sửa rồi OTA lại — lần này không cần cắm USB nữa.

Việc này chỉ ảnh hưởng entity nhiệt độ/độ ẩm, **không ảnh hưởng độ chính xác CO₂**.

## Khi thêm SHT30 sau này

Ba thay đổi, không cần đấu lại dây gì ngoài việc mắc song song lên GPIO8/9:

```yaml
# 1. Thêm sensor mới
  - platform: sht3xd
    address: 0x44
    update_interval: 30s
    temperature:
      name: "Nhiệt độ"
      id: temp_value
    humidity:
      name: "Độ ẩm"
      id: humi_value

# 2. Trong khối scd4x, đổi T/H thành internal và xoá id trùng
    temperature:
      name: "SCD41 nhiệt độ"
      internal: true
    humidity:
      name: "SCD41 độ ẩm"
      internal: true
```

Log lúc đó sẽ thấy cả `0x44` và `0x62`.

Anh muốn tôi viết tiếp dashboard HA gộp cả `sensor-t1` và `air-t1` theo tầng, hay automation tự bật lọc khí khi PM2.5 cao?
