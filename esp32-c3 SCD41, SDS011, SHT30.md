Trước khi chọn mạch, cần làm rõ một chuyện: **SDS011 không đo CO₂.** Nó chỉ đo bụi mịn.

SDS011 là cảm biến laser scattering của Nova Fitness, đo **PM2.5 và PM10**. Cái đuôi "TH372" trong tên anh thấy trên Shopee/Lazada thường là mã người bán tự đặt, không phải model của Nova. Có bản kèm SHT/DHT để thêm nhiệt độ-độ ẩm, nhưng vẫn không có CO₂ — CO₂ cần nguyên lý hoàn toàn khác (NDIR hoặc photoacoustic như SCD41).

Nên để có cả CO₂ và PM2.5, anh cần **hai cảm biến**: SCD41 (CO₂) + SDS011 (bụi). Chúng bổ sung nhau chứ không thay nhau — như tôi có nói ở phần trước, CO₂ cao thì mở cửa sổ, còn PM2.5 cao thì phải đóng cửa sổ. Đúng là cần cả hai mới ra được lời khuyên đúng.

## Chọn mạch: ESP32-C3

Không phải ESP8266. Ba lý do cụ thể cho tổ hợp này:

- **Cần cả I2C (SCD41) và UART (SDS011) cùng lúc.** ESP8266 chỉ có một UART hardware, và nó đang dùng cho log. C3 có hai UART.
- **RAM**: hai component cộng cả WiFi/API encryption sẽ chật trên 40KB của ESP8266.
- **Dòng đỉnh**: SCD41 205mA + SDS011 ~80mA (quạt) + WiFi burst. C3 quản lý nguồn tốt hơn.

Bản khuyến nghị: **ESP32-C3 SuperMini** hoặc **C3 DevKitM-1**. Nếu định đặt trong hộp kín thì chọn bản có đầu IPEX cho ăng-ten ngoài.

## Về SDS011 — hai điều quan trọng

**1. Nó cần 5V, và có tuổi thọ hữu hạn.**

SDS011 có quạt hút cơ khí bên trong. Nhà sản xuất công bố tuổi thọ khoảng **8000 giờ hoạt động liên tục** — chỉ khoảng 11 tháng nếu chạy 24/7. Đây là lý do bắt buộc phải dùng **duty cycle**: cho nó ngủ, thức dậy 30 giây mỗi 5–10 phút để đo.

Với chu kỳ 30 giây mỗi 10 phút, tuổi thọ kéo dài lên khoảng **13 năm**. ESPHome hỗ trợ sẵn qua `rx_only_mode` hoặc `update_interval` kết hợp `sds011` component tự quản lý sleep.

**2. Mức logic.** SDS011 chạy 5V nhưng chân TX xuất mức **3.3V** — an toàn nối trực tiếp vào RX của C3. Chiều ngược lại (C3 TX → SDS011 RX) cũng chấp nhận được vì SDS011 nhận mức 3.3V. Không cần level shifter.

**Cân nhắc thay thế**: **PMS5003** hoặc **PMSA003I** dùng nguyên lý tương tự, giá xấp xỉ, nhưng bền hơn (tuổi thọ công bố cao hơn) và PMSA003I dùng I2C nên đỡ được một UART. Nếu anh chưa mua thì tôi nghiêng về PMSA003I cho node cắm điện chạy dài hạn.

## Đấu dây

| Cảm biến | Chân | ESP32-C3 |
|---|---|---|
| **SCD41** | VDD | 3V3 |
| | GND | GND |
| | SDA | GPIO8 |
| | SCL | GPIO9 |
| **SDS011** | 5V | **5V** (từ nguồn, không lấy từ chân 3V3) |
| | GND | GND |
| | TXD | **GPIO20** (RX của C3) |
| | RXD | GPIO21 (TX của C3) |

Nguồn: adapter **5V/1A tối thiểu**. Cấp 5V cho SDS011, còn C3 lấy 3.3V từ LDO onboard.

**Tụ bulk bắt buộc**: 1000µF low-ESR ở đường 5V, thêm 470µF ở 3V3 gần SCD41. Quạt SDS011 khởi động tạo sụt áp đáng kể, và nếu trùng với lúc SCD41 đo thì board sẽ reset.

Đặt SDS011 sao cho **lỗ hút và lỗ thoát khí không bị che**, và không hút phải khí nóng do chính board thải ra.

## Config ESPHome

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
  hardware_uart: USB_CDC     # để UART0 rảnh cho SDS011

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

# ---- SCD41 qua I2C ----
i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true
  frequency: 50kHz

# ---- SDS011 qua UART ----
uart:
  id: uart_sds
  tx_pin: GPIO21
  rx_pin: GPIO20
  baud_rate: 9600

sensor:
  # ===== CO2 =====
  - platform: scd4x
    id: scd41
    address: 0x62
    update_interval: 60s
    automatic_self_calibration: true
    temperature_offset: 6.0 °C    # hiệu chuẩn lại sau 1 giờ chạy
    altitude_compensation: 16m
    co2:
      name: "CO2"
      id: co2_value
      filters: [filter_out: nan]
    temperature:
      name: "Nhiệt độ"
      filters: [filter_out: nan]
    humidity:
      name: "Độ ẩm"
      filters: [filter_out: nan]

  # ===== Bụi mịn =====
  - platform: sds011
    uart_id: uart_sds
    # Đo 30 giây mỗi 10 phút → bảo vệ tuổi thọ quạt.
    # Giá trị tính theo PHÚT, tối đa 30.
    update_interval: 10min
    pm_2_5:
      name: "PM2.5"
      id: pm25_value
      filters: [filter_out: nan]
    pm_10_0:
      name: "PM10"
      id: pm10_value
      filters: [filter_out: nan]

  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 120s
    entity_category: diagnostic

  - platform: uptime
    name: "Uptime"
    update_interval: 120s
    entity_category: diagnostic

# ---- Diễn giải: đây là phần có giá trị thật ----
text_sensor:
  - platform: template
    name: "Nên làm gì"
    icon: "mdi:window-open-variant"
    update_interval: 60s
    lambda: |-
      if (isnan(id(co2_value).state) || isnan(id(pm25_value).state))
        return {"Đang chờ dữ liệu"};
      float co2 = id(co2_value).state;
      float pm  = id(pm25_value).state;

      bool bi   = co2 > 1200;
      bool bui  = pm  > 35;

      if (bi && bui)   return {"Bật lọc khí, đóng cửa"};
      if (bi && !bui)  return {"Mở cửa sổ"};
      if (!bi && bui)  return {"Đóng cửa, bật lọc khí"};
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

  - platform: template
    name: "Hiệu chuẩn CO2 (ngoài trời)"
    entity_category: config
    on_press:
      - scd4x.perform_forced_calibration:
          value: 420
          id: scd41
```

Phần `text_sensor` "Nên làm gì" là lý do đáng để có cả hai cảm biến — nó gộp hai chỉ số thành một hành động cụ thể thay vì để anh tự đoán.

## Ngưỡng PM2.5 tham chiếu (µg/m³)

| PM2.5 | Mức |
|---|---|
| 0–12 | Tốt |
| 12–35 | Trung bình |
| 35–55 | Kém — nhóm nhạy cảm nên hạn chế ra ngoài |
| 55–150 | Xấu |
| > 150 | Rất xấu |

WHO khuyến nghị trung bình năm dưới 5 µg/m³. Hà Nội mùa đông thường xuyên vượt 100. Anh sẽ thấy con số này khá gây sốc trong mấy tháng đầu.

## Xác nhận sau khi flash

Log lúc boot phải có cả hai:

```
[C][i2c.idf:xxx]: Found i2c device at address 0x62
[C][scd4x:xxx]: scd4x
[C][sds011:xxx]: SDS011:
[C][sds011:xxx]:   Update Interval: 10 min
```

Nếu SDS011 không ra dữ liệu: kiểm tra TX/RX có bị đảo không (lỗi phổ biến nhất), và xác nhận nó thật sự được cấp **5V** — ở 3.3V quạt vẫn chạy yếu nhưng laser không đủ công suất, cho ra số sai hoặc không phản hồi.

Anh đã mua SDS011 chưa, hay còn đang cân nhắc? Nếu chưa thì tôi nói rõ hơn về việc so sánh với PMSA003I — nó dùng I2C nên đấu dây đơn giản hơn đáng kể cho tổ hợp này.

---

Chỗ anh lo là hiểu nhầm một điểm quan trọng: **SCD41 và SHT30 dùng chung một bus I2C**, không cần thêm chân nào.

I2C là bus đa thiết bị. Mỗi con có địa chỉ riêng, và chúng nối song song trên cùng hai dây SDA/SCL:

- SCD41 → `0x62`
- SHT30 → `0x44`

Không xung đột địa chỉ, nên anh có thể mắc cả hai (và sau này thêm BMP280 ở `0x76`, cảm biến ánh sáng BH1750 ở `0x23`...) mà vẫn chỉ tốn **2 chân**.

## Bảng ngân sách chân

| Thiết bị | Giao tiếp | Số chân | Chân đề xuất |
|---|---|---|---|
| SCD41 + SHT30 | I2C (chung bus) | **2** | GPIO8 (SDA), GPIO9 (SCL) |
| SDS011 | UART | **2** | GPIO20 (RX), GPIO21 (TX) |
| | | **Tổng: 4** | |

ESP32-C3 SuperMini breakout ra 13 chân dùng được: GPIO0–10, GPIO20, GPIO21. Anh dùng 4, **còn dư 9 chân**. Thoải mái thêm relay, nút bấm, LED trạng thái, hoặc con OLED sau này (OLED SSD1306 cũng I2C, dùng chung luôn).

Nếu dùng **C3 DevKitM-1** thì còn nhiều hơn, có thêm GPIO18/19 (nhưng hai chân đó là USB D-/D+, nên nếu dùng USB-CDC để flash và log thì coi như đã chiếm).

## Một điểm về SDS011-TH372

Hậu tố "TH" trên các bản bán ở VN thường có nghĩa là board đã kèm sẵn cảm biến nhiệt độ-độ ẩm. Nếu bản anh mua có, thì **vẫn nên dùng SHT30 riêng** — vì cảm biến T/H tích hợp nằm ngay trong vỏ cùng với laser và quạt đang chạy, nhiệt độ đọc ra luôn cao hơn thực tế vài độ. Đúng như lý do anh muốn thêm SHT30, và đó là quyết định đúng.

## Đấu dây

| Thiết bị | Chân | Nối vào |
|---|---|---|
| **SCD41** | VDD / GND | 3V3 / GND |
| | SDA / SCL | GPIO8 / GPIO9 |
| **SHT30** | VDD / GND | 3V3 / GND |
| | SDA / SCL | GPIO8 / GPIO9 *(song song với SCD41)* |
| **SDS011** | 5V / GND | **5V** / GND |
| | TXD | GPIO20 |
| | RXD | GPIO21 |

**Về pull-up I2C**: nếu cả hai module đều có điện trở 10kΩ sẵn (module Adafruit/Seeed/breakout thường có), mắc song song thành 5kΩ — vẫn hoàn toàn ổn ở 50kHz. Nếu sau này anh thêm con thứ ba thứ tư thì tổng trở tụt xuống dưới 2kΩ, lúc đó nên tháo pull-up trên các module phụ, chỉ giữ trên một module.

**Vị trí đặt SHT30**: đây là chỗ quyết định độ chính xác. Kéo nó ra bằng dây 10–15cm, đặt **xa cả module C3 lẫn SDS011**. Cả hai đều sinh nhiệt. Nếu để chung một cục thì anh vừa trả tiền cho độ chính xác ±0.3°C rồi tự phá nó bằng nhiệt tự sinh.

**Nguồn**: cộng dồn đỉnh của SDS011 (quạt ~80–100mA khi khởi động), SCD41 (205mA trong 200ms) và C3 (~350mA lúc phát WiFi) — dùng adapter **5V/2A**. Kèm tụ **1000µF low-ESR trên đường 5V** và **470µF trên 3V3** sát SCD41.

## Config hoàn chỉnh

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
  hardware_uart: USB_CDC        # giải phóng UART0 cho SDS011

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

# ===== Bus I2C dùng chung cho SCD41 + SHT30 =====
i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true
  frequency: 50kHz              # SCD4x giới hạn 100kHz, 50kHz an toàn hơn

# ===== UART riêng cho SDS011 =====
uart:
  id: uart_sds
  tx_pin: GPIO21
  rx_pin: GPIO20
  baud_rate: 9600

sensor:
  # ---------- SHT30: nguồn nhiệt độ / độ ẩm CHÍNH ----------
  - platform: sht3xd
    address: 0x44
    update_interval: 30s
    temperature:
      name: "Nhiệt độ"
      id: temp_value
      filters:
        - filter_out: nan
        - median: {window_size: 3, send_every: 3}
    humidity:
      name: "Độ ẩm"
      id: humi_value
      filters:
        - filter_out: nan
        - median: {window_size: 3, send_every: 3}

  # ---------- SCD41: CO2 ----------
  - platform: scd4x
    id: scd41
    address: 0x62
    update_interval: 60s
    automatic_self_calibration: true
    temperature_offset: 6.0 °C
    altitude_compensation: 16m
    co2:
      name: "CO2"
      id: co2_value
      filters: [filter_out: nan]
    # T/H của SCD41 kém hơn SHT30 → để internal, không đưa vào HA
    # nhưng vẫn giữ để so sánh khi debug qua log
    temperature:
      name: "SCD41 nhiệt độ"
      internal: true
    humidity:
      name: "SCD41 độ ẩm"
      internal: true

  # ---------- SDS011: bụi mịn ----------
  - platform: sds011
    uart_id: uart_sds
    # Đơn vị PHÚT, tối đa 30. Đo ~30s rồi ngủ → bảo vệ tuổi thọ quạt.
    # 8000h danh định: chạy 24/7 chỉ được ~11 tháng,
    # với chu kỳ 10 phút thì kéo lên khoảng 13 năm.
    update_interval: 10min
    pm_2_5:
      name: "PM2.5"
      id: pm25_value
      filters: [filter_out: nan]
    pm_10_0:
      name: "PM10"
      id: pm10_value
      filters: [filter_out: nan]

  - platform: wifi_signal
    name: "WiFi RSSI"
    update_interval: 120s
    entity_category: diagnostic

  - platform: uptime
    name: "Uptime"
    update_interval: 120s
    entity_category: diagnostic

# ===== Gộp ba chỉ số thành một lời khuyên =====
text_sensor:
  - platform: template
    name: "Nên làm gì"
    icon: "mdi:window-open-variant"
    update_interval: 60s
    lambda: |-
      if (isnan(id(co2_value).state) || isnan(id(pm25_value).state))
        return {"Đang chờ dữ liệu"};
      bool bi  = id(co2_value).state  > 1200;
      bool bui = id(pm25_value).state > 35;
      if (bi && bui)  return {"Đóng cửa, bật lọc khí"};
      if (bi)         return {"Mở cửa sổ"};
      if (bui)        return {"Đóng cửa, bật lọc khí"};
      return {"Không khí ổn"};

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

  - platform: template
    name: "Hiệu chuẩn CO2 (ngoài trời)"
    entity_category: config
    on_press:
      - scd4x.perform_forced_calibration:
          value: 420
          id: scd41
```

## Xác nhận sau khi flash

Log lúc boot phải thấy **cả hai địa chỉ I2C**:

```
[C][i2c.idf:xxx]: Results from i2c bus scan:
[C][i2c.idf:xxx]: Found i2c device at address 0x44
[C][i2c.idf:xxx]: Found i2c device at address 0x62
[C][sht3xd:xxx]: SHT3xD:
[C][scd4x:xxx]: scd4x
[C][sds011:xxx]: SDS011:
[C][sds011:xxx]:   Update Interval: 10 min
```

Nếu chỉ thấy **một** địa chỉ: một trong hai module chưa nối đúng, hoặc dây SDA/SCL của nó bị đảo. Bus scan là công cụ chẩn đoán tốt nhất ở đây — nó cho anh biết chính xác con nào không lên.

## Hiệu chuẩn `temperature_offset` cho SCD41

Giờ anh có SHT30 làm chuẩn đối chiếu, việc này dễ hơn nhiều. Để chạy 1 giờ cho ổn định nhiệt, rồi xem log:

```
[D][sensor] 'Nhiệt độ': Sending state 28.3 °C          ← SHT30
[D][scd4x] Temperature: 34.1 °C                        ← SCD41 nội bộ
```

Rồi cộng phần chênh vào offset hiện tại:

```
offset_mới = 6.0 + (34.1 − 28.3) = 11.8 °C
```

Sửa `temperature_offset` rồi OTA lại. Việc này không ảnh hưởng độ chính xác CO₂ — SCD41 chỉ dùng nhiệt độ để bù trừ nội bộ, và offset sai chỉ làm lệch entity T/H mà anh đã để `internal: true` nên không thấy trong HA.
