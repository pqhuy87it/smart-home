Bắt đầu từ bench test là đúng thứ tự. Đây là sơ đồ nối chân:## Bốn điểm dễ chết trên D1 mini

**Bộ chia 10k/10k là bắt buộc, không phải tùy chọn.** Chân AO của MQ-2 swing tới gần 5V (vì heater và mạch chia tải đều tham chiếu VCC = 5V). Chân A0 của D1 mini chỉ chịu được 3.2V. Nối trực tiếp là chết ADC, có thể chết luôn cả chip.

Con số 3.2V đó là nhờ **D1 mini đã có sẵn divider 220k/100k trên board** — ESP8266 thuần chỉ đo được 0–1.0V ở chân TOUT. Điều này quan trọng khi tính hệ số nhân trong config: tín hiệu đi qua **hai** tầng chia liên tiếp.

Vì divider nội bộ 320kΩ mắc song song với nhánh 10k dưới, điện áp tại node thực tế ≈ 2.46V thay vì 2.50V, và hệ số tổng ≈ 6.50 chứ không phải 6.40. Sai số 1.5% này sẽ bị hiệu chỉnh hết ở bước đo multimeter bên dưới, nên đừng lo — nhưng đừng dùng con số 6.40 từ các tutorial trên mạng.

**Không cấp MQ-2 từ chân 3V3.** Heater cần đúng 5V để lên ~300°C. Cấp 3.3V thì sensor vẫn có output, vẫn nhìn như đang chạy, nhưng số đọc hoàn toàn vô nghĩa. Đây là lỗi thầm lặng khó phát hiện nhất.

**Dùng adapter tường 5V/1A trở lên.** Heater ngốn ~150mA liên tục (33Ω @ 5V, ~800mW). Chân 5V của D1 mini lấy trực tiếp từ USB VBUS nên cấp được, nhưng cắm vào cổng USB máy tính thì vừa giới hạn dòng vừa nhiễu — mà ESP8266 ADC đã rất nhiễu sẵn.

**Không nối chân DO.** Output của LM393 idle ở mức ~5V, đưa vào GPIO 3.3V là quá áp. Mà nó cũng vô dụng: ngưỡng do biến trở trên board quyết định, không calibrate được, và ta làm ngưỡng bằng software rồi. Nếu vẫn muốn nối thì phải có divider riêng cho nó.

Cuối cùng: MQ-2 nóng thật (vỏ 50–60°C). Đừng cắm trực tiếp lên board — dùng dây jumper 15–20cm, để sensor tách ra.

## Config ESPHome

Đây là bench config, mục tiêu là **đo và hiệu chỉnh**, chưa có alarm logic:

```yaml
# gas-test.yaml
substitutions:
  device_name: gas-test
  friendly_name: "Gas Test Bench"

  # === SỬA 3 GIÁ TRỊ NÀY THEO ĐO THỰC TẾ — xem quy trình bên dưới ===
  adc_scale: "6.50"    # hệ số nhân, hiệu chỉnh bằng multimeter (bước 2)
  rl_ohms:   "1000"    # điện trở tải RL trên module, đọc marking SMD (bước 4)
  r0_ohms:   "9800"    # Rs không khí sạch / 9.8, lấy từ nút calibrate (bước 5)

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  on_boot:
    priority: 600
    then:
      - lambda: 'id(warming_up) = true;'
      - delay: 5min           # heater cần thời gian ổn định nhiệt
      - lambda: 'id(warming_up) = false;'
      - logger.log: "MQ-2 warmup hoan tat, so doc da dung tin"

esp8266:
  board: d1_mini
  restore_from_flash: true

globals:
  - id: warming_up
    type: bool
    initial_value: 'true'

logger:
  level: INFO

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${device_name}-setup"

captive_portal:

sensor:
  # --- Tầng 1: ADC thô. ESP8266 chỉ có 1 chân ADC (A0) và cực nhiễu khi WiFi active.
  # median lọc spike, moving average làm phẳng phần còn lại.
  - platform: adc
    pin: A0
    id: adc_raw
    update_interval: 200ms
    internal: true
    filters:
      - median:
          window_size: 15
          send_every: 15
          send_first_at: 15

  # --- Tầng 2: điện áp thực tại chân AO của MQ-2
  # adc_raw là 0-1.0 V tại chip. x3.2 (divider trên board) x2.03 (divider ngoài) = 6.50
  - platform: copy
    source_id: adc_raw
    id: mq2_ao
    name: "${friendly_name} AO Voltage"
    unit_of_measurement: "V"
    accuracy_decimals: 3
    filters:
      - multiply: ${adc_scale}
      - exponential_moving_average:
          alpha: 0.2
          send_every: 1

  # --- Tầng 3: điện trở sensor Rs. Đây là đại lượng vật lý thật sự.
  # Rs = RL x (Vc - Vout) / Vout, với Vc = 5.0 V
  - platform: template
    name: "${friendly_name} Rs"
    id: mq2_rs
    unit_of_measurement: "Ω"
    accuracy_decimals: 0
    update_interval: 5s
    lambda: |-
      const float vc = 5.0f;
      const float rl = ${rl_ohms};
      float v = id(mq2_ao).state;
      if (isnan(v) || v < 0.02f || v >= vc) return {};
      return rl * (vc - v) / v;

  # --- Tầng 4: Rs/R0. Đây là con số nên dùng để đặt ngưỡng, KHÔNG dùng ppm.
  # Không khí sạch ~9.8. Càng nhiều gas, ratio càng nhỏ.
  - platform: template
    name: "${friendly_name} Rs/R0 Ratio"
    id: mq2_ratio
    unit_of_measurement: "ratio"
    accuracy_decimals: 2
    update_interval: 5s
    lambda: |-
      float rs = id(mq2_rs).state;
      if (isnan(rs)) return {};
      return rs / ${r0_ohms};

  # --- Tầng 5: ppm LPG ước lượng. Fit log-log từ datasheet MQ-2.
  # Đây là số ĐỊNH TÍNH, sai số dễ tới vài trăm phần trăm. Dùng để cảm nhận
  # độ lớn, không dùng để ra quyết định.
  - platform: template
    name: "${friendly_name} LPG (ước lượng)"
    id: mq2_lpg
    unit_of_measurement: "ppm"
    accuracy_decimals: 0
    update_interval: 10s
    lambda: |-
      float r = id(mq2_ratio).state;
      if (isnan(r) || r <= 0.0f) return {};
      float ppm = 574.25f * powf(r, -2.222f);
      if (ppm < 0.0f || ppm > 50000.0f) return {};
      return ppm;

  - platform: wifi_signal
    name: "${friendly_name} WiFi Signal"
    update_interval: 60s

  - platform: uptime
    name: "${friendly_name} Uptime"

binary_sensor:
  # Chưa phải alarm thật — chỉ để quan sát ngưỡng có hợp lý không.
  # Ngưỡng ratio 3.0 tương ứng khoảng vài trăm ppm LPG.
  - platform: template
    name: "${friendly_name} Gas Suspected"
    lambda: |-
      if (id(warming_up)) return false;
      float r = id(mq2_ratio).state;
      if (isnan(r)) return {};
      return r < 3.0f;
    filters:
      - delayed_on: 5s
      - delayed_off: 30s

text_sensor:
  - platform: template
    name: "${friendly_name} Status"
    update_interval: 10s
    lambda: |-
      if (id(warming_up)) return std::string("warming up");
      if (isnan(id(mq2_ao).state)) return std::string("no reading");
      if (id(mq2_ao).state < 0.05f) return std::string("check wiring");
      return std::string("ready");

button:
  # Bấm nút này khi sensor đã burn-in đủ và đang ở không khí sạch.
  # Đọc log để lấy R0, rồi paste vào substitutions ở trên.
  - platform: template
    name: "${friendly_name} Calibrate R0"
    on_press:
      - lambda: |-
          float rs = id(mq2_rs).state;
          if (isnan(rs)) {
            ESP_LOGW("cal", "Chua co gia tri Rs - kiem tra day noi");
            return;
          }
          if (id(warming_up)) {
            ESP_LOGW("cal", "Con dang warmup - doi them roi bam lai");
            return;
          }
          ESP_LOGI("cal", "=================================");
          ESP_LOGI("cal", "AO   = %.3f V", id(mq2_ao).state);
          ESP_LOGI("cal", "Rs   = %.0f ohm", rs);
          ESP_LOGI("cal", "R0   = %.0f ohm   <-- dat vao r0_ohms", rs / 9.8f);
          ESP_LOGI("cal", "=================================");
```

## Quy trình hiệu chỉnh

**1. Đo trước khi nối vào A0.** Nối VCC/GND/divider, **chưa nối node vào A0**. Cấp nguồn, chờ 5 phút, đo điện áp tại node giữa hai điện trở. Phải ≤ 2.6V. Nếu ra ~5V thì anh nối sai (AO đi trực tiếp, không qua divider) — nối vào A0 lúc này là chết board.

**2. Hiệu chỉnh `adc_scale`.** Nối A0, flash, chờ hết warmup. So sánh:

```
adc_scale mới = adc_scale cũ × (số đo multimeter / số ESPHome báo)
```

Ví dụ multimeter đo 2.44V mà ESPHome báo 2.51V → `6.50 × 2.44 / 2.51 = 6.32`. Bước này bù cả sai số điện trở (loại 5% thường lệch đáng kể) và sai số divider trên board.

**3. Burn-in 48 giờ.** Cấp nguồn liên tục, **đặt ngoài bếp**, nơi thoáng khí sạch. MQ-2 mới drift rất mạnh trong 24h đầu — R0 đo trước khi burn-in xong sẽ sai hoàn toàn và toàn bộ ngưỡng của anh sẽ lệch. Không có đường tắt cho bước này.

**4. Đọc `RL` trên board.** Tìm điện trở SMD nối giữa chân AO và GND. Marking: `1001` = 1kΩ, `5001` = 5kΩ, `1002` = 10kΩ, `2002` = 20kΩ. Module MQ-2 xanh phổ biến thường là 1kΩ nhưng **hàng clone rất hay khác** — đây là nguồn sai số lớn nhất nếu đoán bừa. Nếu không đọc được marking, tháo đo bằng ohm meter hoặc suy ra: cấp 5V trong không khí sạch, `RL = Rs_datasheet / ((5/Vout) - 1)` với `Rs ≈ 9.8 × R0`.

**5. Lấy R0.** Sau burn-in, trong không khí sạch, bấm nút `Calibrate R0` trong HA rồi mở log ESPHome. Paste giá trị vào `r0_ohms`, flash lại. Xong bước này thì `Rs/R0 Ratio` phải hiển thị ~9.8.

**6. Test bằng gas thật.** Bật bật lửa gas (loại butane) **không đánh lửa**, bấm van xả 1–2 giây ở cách sensor ~20cm. Làm ngoài trời hoặc nơi rất thoáng, không có nguồn lửa nào gần. Ratio phải tụt mạnh (xuống dưới 1) trong vài giây, rồi hồi về ~9.8 sau 1–2 phút. Nếu không phản ứng: kiểm tra sensor có nóng không (sờ nhẹ vỏ), và kiểm tra lại VCC đúng 5V.

**7. Ghi baseline 1 tuần.** Để nguyên chỗ đó, xem `Rs/R0 Ratio` trong History của HA. Anh cần biết ratio dao động tự nhiên bao nhiêu trước khi đặt ngưỡng thật.

## Hai điều sẽ ảnh hưởng lớn ở Hà Nội

**Độ ẩm.** Độ nhạy MQ-2 phụ thuộc mạnh vào độ ẩm — datasheet dựa trên 65% RH, và chênh lệch giữa mùa nồm (95%+) với mùa đông khô có thể dịch ratio khoảng 20–30%. Nghĩa là ngưỡng calibrate vào tháng 3 sẽ sai vào tháng 12. Nên **thêm một DHT22 hoặc BME280 vào node này ngay từ bench test** để ghi nhiệt/ẩm song song với ratio. Sau một tháng dữ liệu anh sẽ thấy tương quan và làm được compensation. Đơn giản hơn: chấp nhận đặt ngưỡng bảo thủ (ratio 3.0 thay vì 5.0) và recalibrate 2 lần/năm.

**Hàng clone.** MQ-2 trên thị trường VN chất lượng rất thất thường. Nếu sau burn-in mà Rs trong không khí sạch lệch quá xa dải datasheet (Rs ≈ 3–30kΩ ứng với R0 ≈ 300Ω–3kΩ tùy RL), hoặc ratio drift hơn 15% trong một ngày ở điều kiện ổn định, thì con sensor đó không dùng được. Mua 2–3 con từ nguồn khác nhau và so sánh là cách rẻ nhất để phát hiện.

## Nhắc lại một điều

Bench test này để anh hiểu sensor và có số liệu thật — rất đáng làm. Nhưng bản đi vào tủ gas thì tôi vẫn khuyên đầu báo gas certified có relay output, vì lý do đã nói: trong tủ kín đang tích tụ LPG, một module có heater 300°C với dây jumper lỏng là thứ không nên có. Kết quả calibrate từ bench này vẫn dùng được để anh biết đầu báo certified đang báo ở nồng độ nào, và để so sánh hai thiết bị với nhau.

Sau khi có số liệu bước 7, gửi tôi khoảng dao động ratio của anh — tôi sẽ giúp tính ngưỡng cho vùng tủ gas dựa trên số thật thay vì con 3.0 đặt tạm.
