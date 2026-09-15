Đặt tên node là `m1` cho khớp với convention `t1`/`t2` bạn đang dùng — đổi lại trong `substitutions` nếu muốn tên theo vị trí.

**Bổ sung vào `secrets.yaml`** (file này nằm cùng thư mục config ESPHome trong stack `~/smarthome`). `wifi_ssid`/`wifi_password` chắc đã có sẵn từ t1/t2, chỉ cần thêm:

```yaml
api_encryption_key: "<32 bytes base64>"
ota_password: "<chuỗi bất kỳ>"
ap_password: "<ít nhất 8 ký tự>"
```

Sinh key:
```bash
openssl rand -base64 32
```

**Deploy.** Copy `m1.yaml` vào thư mục config của ESPHome rồi flash lần đầu qua USB (từ Mac mini hoặc từ Pi 5 nếu cắm trực tiếp):

```bash
cd ~/smarthome
docker compose exec esphome esphome run config/m1.yaml
```

Lần đầu bắt buộc phải cắm USB; các lần sau chọn OTA. Nếu bạn quen dùng dashboard thì vào `http://exlinct:6052` cũng được, kết quả giống nhau.

## Ba điểm quan trọng trong config

**Warm-up được xử lý ở firmware, không phải bằng `delay()`.** Global `pir_ready` chặn mọi state change trong 60s đầu bằng lambda filter trả về `{}` — Home Assistant sẽ không thấy entity nhảy loạn sau mỗi lần reboot hay mất điện. Đây là chỗ hầu hết các config trên mạng làm sai.

**Hai entity thay vì một.** `Motion` giữ 30s (phản ánh trạng thái vật lý), `Occupancy` giữ 5 phút (dùng cho automation). Tách ra như vậy để bạn không phải nhét `for: minutes: 5` vào từng automation, và khi debug thì nhìn được raw behavior.

**Biến trở Tx trên HC-SR501 vặn về thấp nhất.** Nếu để sensor tự giữ output HIGH lâu, ESPHome mất khả năng biết có chuyển động mới hay không — bạn sẽ không phân biệt được "vẫn đang có người" và "sensor còn đang trong thời gian hold". Để phần cứng xung ngắn, phần mềm lo giữ trạng thái.

## Verification

1. **Kiểm tra boot trước khi tin vào sensor.** Sau khi flash, xem log:
   ```bash
   docker compose exec esphome esphome logs config/m1.yaml
   ```
   Phải thấy `Boot OK — bat dau warm-up PIR`, rồi đúng 60s sau là `PIR warm-up xong`. Nếu board reboot loop, kiểm tra lại xem có vô tình cắm OUT vào D3/D4/D8 không.

2. **Test tại chỗ bằng LED on-board** — không cần mở HA. Bật entity `Motion 1 LED`, quơ tay trước sensor, LED xanh trên D1 mini phải sáng. Nếu LED nhấp nháy khi bạn *không* làm gì, đó là false trigger do RF/nguồn → xem lại tụ decoupling và khoảng cách PIR–ESP.

3. **Đo false positive qua đêm.** Để node chạy 8–12 tiếng trong phòng trống, rồi vào HA → entity `Motion 1` → History. Zero trigger là đạt. Nếu có trigger rải rác, tương quan chúng với entity `WiFi Signal`: nếu trigger trùng với lúc RSSI dao động mạnh thì chắc chắn là nhiễu RF chứ không phải sensor hỏng.

4. **Chỉnh sensitivity sau cùng.** Biến trở Sx vặn ngược chiều kim đồng hồ để giảm range. Chỉ chỉnh khi đã xác nhận không có false trigger ở bước 3 — nếu chỉnh trước, bạn sẽ không biết mình đang sửa nhiễu hay sửa độ nhạy.

Nếu con PIR này để làm gate cho bộ phát hiện gas ở bếp, tôi viết tiếp cho bạn phần automation trong Home Assistant — logic kiểu "chỉ báo động khi gas vượt ngưỡng **và** không có chuyển động trong 10 phút gần nhất" cần cẩn thận ở chỗ fail-safe, vì nếu node `m1` offline thì automation phải mặc định là báo động chứ không phải im lặng.

---

# =============================================================================
#  m1.yaml — PIR motion node
#  Hardware: Wemos D1 mini (ESP8266) + HC-SR501
#
#  Wiring:
#    HC-SR501 VCC  -> D1 mini 5V   (KHONG dung 3V3: HC-SR501 can >= 4.5V)
#    HC-SR501 GND  -> D1 mini G
#    HC-SR501 OUT  -> D1 mini D1 (GPIO5)   [OUT xuat muc HIGH 3.3V, an toan]
#
#    Them 100nF ceramic // 100uF electrolytic ngay tai chan VCC/GND cua PIR.
#    Jumper tren HC-SR501 de o vi tri H (repeat trigger).
#    Chinh bien tro Tx (delay) ve MUC THAP NHAT (~3s) — de ESPHome lo phan
#    giu trang thai bang delayed_off, khong de sensor tu giu.
# =============================================================================

substitutions:
  node_name: m1
  friendly_name: "Motion 1"
  pir_pin: GPIO5          # D1 tren Wemos D1 mini
  warmup_time: 60s        # HC-SR501 spam false trigger trong ~30-60s dau
  motion_hold: 30s        # giu trang thai ON sau xung cuoi cung
  occupancy_hold: 5min    # cho automation "co nguoi trong phong"

esphome:
  name: ${node_name}
  friendly_name: ${friendly_name}
  on_boot:
    priority: -100
    then:
      - logger.log: "Boot OK — bat dau warm-up PIR"
      - delay: ${warmup_time}
      - lambda: 'id(pir_ready) = true;'
      - logger.log: "PIR warm-up xong, bat dau nhan su kien"

esp8266:
  board: d1_mini
  restore_from_flash: true

logger:
  level: INFO
  baud_rate: 115200

api:
  encryption:
    key: !secret api_encryption_key
  # Node cam dien lien tuc -> reboot neu mat ket noi HA qua lau
  reboot_timeout: 15min

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: true
  # Tat power save: giam do tre bao dong va tranh burst RF khi wake radio,
  # vi burst chinh la thu pham gay false trigger cho BISS0001.
  power_save_mode: none
  ap:
    ssid: "${node_name}-fallback"
    password: !secret ap_password

captive_portal:

globals:
  - id: pir_ready
    type: bool
    restore_value: false
    initial_value: 'false'

# -----------------------------------------------------------------------------
# Binary sensors
# -----------------------------------------------------------------------------
binary_sensor:
  # Sensor chinh — da loc warm-up va debounce
  - platform: gpio
    id: pir_motion
    name: "${friendly_name} Motion"
    pin:
      number: ${pir_pin}
      mode:
        input: true
        pullup: false       # HC-SR501 push-pull, khong can pull-up
    device_class: motion
    filters:
      # 1. Chan moi thay doi trang thai trong giai doan warm-up.
      #    Tra ve {} = discard, HA se khong nhan duoc gi.
      - lambda: |-
          if (!id(pir_ready)) return {};
          return x;
      # 2. Loc nhieu xung rat ngan (glitch RF thuong < 100ms)
      - delayed_on: 120ms
      # 3. Gom cac lan trigger lien tiep thanh mot phien
      - delayed_off: ${motion_hold}

  # Suy ra "co nguoi" voi thoi gian giu dai hon — dung cho automation
  - platform: template
    id: pir_occupancy
    name: "${friendly_name} Occupancy"
    device_class: occupancy
    lambda: 'return id(pir_motion).state;'
    filters:
      - delayed_off: ${occupancy_hold}

  - platform: status
    name: "${friendly_name} Status"
    entity_category: diagnostic

# -----------------------------------------------------------------------------
# Diagnostics
# -----------------------------------------------------------------------------
sensor:
  - platform: wifi_signal
    name: "${friendly_name} WiFi Signal"
    update_interval: 60s
    entity_category: diagnostic
    filters:
      - median:
          window_size: 5
          send_every: 5

  - platform: uptime
    name: "${friendly_name} Uptime"
    update_interval: 60s
    entity_category: diagnostic

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP"
      entity_category: diagnostic

button:
  - platform: restart
    name: "${friendly_name} Restart"
    entity_category: diagnostic

# -----------------------------------------------------------------------------
# LED on-board: nhay khi co chuyen dong (tien debug tai cho, khong can mo HA)
# GPIO2 = LED_BUILTIN tren D1 mini, active LOW.
# -----------------------------------------------------------------------------
output:
  - platform: gpio
    id: led_out
    pin:
      number: GPIO2
      inverted: true

light:
  - platform: binary
    id: status_led
    name: "${friendly_name} LED"
    output: led_out
    entity_category: config
    restore_mode: ALWAYS_OFF

---

Câu trả lời ngắn: **`m1.yaml` gần như không phải sửa gì** — nó chỉ là sensor node, cứ để nó báo trạng thái lên HA. Toàn bộ việc cần làm nằm ở node relay và ở automation. Nhưng ESP-01S có mấy cái bẫy phần cứng mà bạn nên biết trước khi mua/hàn.

## Kiểm tra board relay ESP-01S trước đã

Đây là bước quan trọng nhất, làm sai là viết config vô ích. Board relay ESP-01S trên thị trường có **hai kiểu hoàn toàn khác nhau**:

**Kiểu A — GPIO trực tiếp.** Trên board chỉ có ESP-01S, một transistor, một relay, một LDO. GPIO0 đi thẳng qua transistor vào cuộn relay. Đây là kiểu dễ dùng.

**Kiểu B — có STM8S003F3 làm trung gian.** Nhìn trên board sẽ thấy thêm một con IC 20 chân nhỏ (TSSOP20) bên cạnh ESP-01S. Kiểu này **không điều khiển bằng GPIO được** — ESP-01S phải gửi lệnh qua UART 115200: `A0 01 01 A2` để bật, `A0 01 00 A1` để tắt. Rất nhiều người mua về cắm GPIO không thấy gì rồi tưởng board hỏng.

Lật board lên đếm IC trước khi làm tiếp.

## Ba hạn chế của ESP-01S cần thiết kế quanh nó

**1. GPIO0 là chân boot-strapping.** Nó phải ở mức HIGH lúc khởi động, nếu LOW thì ESP vào flash mode. Với board kiểu A active-HIGH, hệ quả là **relay sẽ đóng một nhịp ngắn (~300ms) mỗi lần cấp nguồn hoặc reboot** — đèn sẽ chớp một cái. Không có cách sửa bằng firmware, vì lúc đó firmware chưa chạy. Nếu bạn không chấp nhận được thì phải chọn board active-LOW, hoặc chuyển sang dùng GPIO2.

**2. Flash chỉ 1MB.** Vẫn OTA được, nhưng phải chia đôi nên mỗi slot ~512KB, trong khi binary ESPHome với WiFi + API đã ~430–460KB. Vì vậy config relay phải **tối giản**: bỏ `captive_portal`, bỏ `web_server`, bỏ hết sensor diagnostic không cần. Nhồi thêm component là hết chỗ, OTA fail giữa chừng.

**3. Không có mạch auto-program.** Lần flash đầu phải nối GPIO0 xuống GND bằng tay rồi mới cấp nguồn, dùng USB-TTL riêng. Đừng cấp nguồn 3.3V từ chân của FTDI — không đủ dòng cho peak 300mA lúc WiFi TX + relay, board sẽ brownout reset liên tục và bạn sẽ tưởng là lỗi firmware.

**Về an toàn 220V:** loại board relay này khoảng cách creepage giữa phía cao áp và phía low-voltage rất mỏng, và không có vỏ. Nếu điều khiển đèn 220V thì đặt trong hộp nhựa kín, tách hẳn dây AC khỏi phần 3.3V, và đừng bao giờ cắm USB-TTL vào board khi phía AC đang có điện.

Giờ đến config.**Thêm vào `secrets.yaml`:**
```yaml
r1_api_key: "<32 bytes base64 mới, khác của m1>"
```

## Chọn đường điều khiển

Ba cách, tôi khuyên cách 1:

**1. Qua Home Assistant automation** — độ trễ thực tế 100–300ms, đủ nhanh cho đèn. Ưu điểm là logic nằm một chỗ, sửa được từ UI, và bạn thêm điều kiện (giờ, độ sáng, người vắng nhà) mà không phải flash lại node nào. Nhược điểm: HA chết thì đèn không tự động.

**2. MQTT trực tiếp qua Mosquitto** — bỏ HA khỏi đường critical, `m1` publish vào topic, `r1` subscribe. Nhanh hơn chút, nhưng vẫn phụ thuộc Pi 5, mà HA và Mosquitto chạy chung một máy nên lợi ích thật sự khá nhỏ. Chỉ đáng làm nếu HA của bạn hay restart.

**3. ESP-NOW peer-to-peer** — thực sự độc lập, nhưng ESPHome không hỗ trợ native trên ESP8266, phải dùng external component và bạn mất luôn khả năng điều khiển từ HA/HomeKit. Bạn đã làm ESP-NOW master-slave trên C3 rồi nên biết nó ổn, nhưng ở đây tôi thấy không đáng đánh đổi.

Automation cho cách 1 (`automations.yaml` hoặc tạo qua UI rồi dán YAML):

```yaml
alias: "Đèn theo chuyển động - m1"
mode: restart
triggers:
  - trigger: state
    entity_id: binary_sensor.m1_occupancy
    to: "on"
    id: on
  - trigger: state
    entity_id: binary_sensor.m1_occupancy
    to: "off"
    id: off
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: "on"
          - condition: sun
            after: sunset
            before: sunrise
        sequence:
          - action: light.turn_on
            target:
              entity_id: light.light_1
      - conditions:
          - condition: trigger
            id: "off"
        sequence:
          - action: light.turn_off
            target:
              entity_id: light.light_1
```

Thời gian giữ đèn do `occupancy_hold: 5min` trong `m1.yaml` quyết định, không phải `for:` ở đây. Muốn đổi thì sửa substitution rồi OTA lại m1 — tôi cố ý để ở đó để automation không phải giữ state.

## Sửa gì trong `m1.yaml`

Không bắt buộc sửa gì. Chỉ có một điểm nên cân nhắc: nếu đèn ở ngay trong tầm nhìn của PIR và bạn dùng đèn sợi đốt hoặc halogen, nhiệt từ đèn có thể tự kích PIR → đèn không bao giờ tắt. LED thì không sao. Nếu gặp, hướng lens PIR tránh đèn hoặc tăng `delayed_on` lên 500ms.

## Verification

1. **Test relay độc lập trước.** Sau khi flash `r1`, chưa đấu AC, chưa dính gì tới PIR — vào HA bật/tắt entity `Light 1` bằng tay. Nghe tiếng relay click và thấy LED trên board. Nếu không click: board kiểu B mà bạn đang dùng khối GPIO (hoặc ngược lại).

2. **Quan sát lúc cấp nguồn.** Rút điện r1 rồi cắm lại, nhìn relay. Nếu nó click một nhịp rồi nhả — đó là hiện tượng GPIO0 boot strapping đã nói ở trên, bình thường với board active-HIGH. Nếu nó đóng và **giữ luôn** thì board của bạn active-LOW → đổi `inverted: false` thành `true`.

3. **Kiểm tra headroom flash.** Trong log build, xem dòng `Flash: [====  ] xx.x%`. Nếu vượt ~48% của 1MB thì OTA sẽ fail. Lúc đó bỏ bớt `binary_sensor: status` và hạ `logger: level: NONE`.

4. **Ghép hai node.** Bật automation, quơ tay trước m1, bấm giờ. Đèn phải sáng trong dưới 1 giây. Nếu trễ vài giây, kiểm tra `power_save_mode: none` đã có ở cả hai node chưa — đây là nguyên nhân trễ phổ biến nhất.

5. **Test fail-safe.** Bật đèn, rồi `docker compose stop homeassistant`. Đèn phải **giữ nguyên trạng thái**, không tắt và không reboot loop — đó là tác dụng của `api: reboot_timeout: 0s`. Khởi động HA lại, entity phải trở về online trong ~30s mà đèn không chớp.

Watchdog 2 tiếng trong config là lưới an toàn cuối: nếu automation gửi lệnh bật rồi HA chết trước khi gửi lệnh tắt, đèn vẫn tự tắt chứ không sáng suốt đêm. Chỉnh `7200` nếu muốn khác.

---

# =============================================================================
#  r1.yaml — ESP-01S relay node (bat/tat den)
#
#  QUAN TRONG: chon dung 1 trong 2 khoi ben duoi tuy loai board.
#    KIEU A  = board chi co ESP-01S + transistor + relay   -> dung khoi "GPIO"
#    KIEU B  = board co them IC STM8S003F3 (TSSOP20)       -> dung khoi "UART"
#  Comment khoi con lai ra.
#
#  Flash lan dau:
#    - Noi GPIO0 -> GND, cap nguon, roi moi flash qua USB-TTL
#    - Cap nguon 5V vao chan VCC cua board relay (board co LDO san)
#      KHONG lay 3V3 tu FTDI
#
#  Flash chi 1MB -> config phai toi gian. Khong them web_server,
#  khong captive_portal, khong sensor thua. Con lai ~50KB headroom cho OTA.
# =============================================================================

substitutions:
  node_name: r1
  friendly_name: "Light 1"

esphome:
  name: ${node_name}
  friendly_name: ${friendly_name}

esp8266:
  board: esp01_1m
  restore_from_flash: true    # giu trang thai den qua mat dien

# -----------------------------------------------------------------------------
# api / ota / wifi
# -----------------------------------------------------------------------------
api:
  encryption:
    key: !secret r1_api_key
  # 0s = KHONG tu reboot khi mat ket noi HA.
  # Voi node dieu khien den, reboot loop = den nhap nhay. De WiFi lo viec reboot.
  reboot_timeout: 0s

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: true
  power_save_mode: none
  reboot_timeout: 15min
  # Khong dung fallback AP: ton flash, va node nay khong co man hinh de cau hinh

logger:
  level: WARN
  baud_rate: 0        # tat log qua UART (KIEU B bat buoc; KIEU A cung nen tat
                      # de tranh rac tren TX va tiet kiem flash)

# =============================================================================
#  KIEU A — dieu khien bang GPIO
# =============================================================================
switch:
  - platform: gpio
    id: relay_out
    pin:
      number: GPIO0
      inverted: false     # doi thanh true neu board active-LOW
    restore_mode: RESTORE_DEFAULT_OFF
    internal: true        # an switch tho, chi expose "light" ben duoi

# =============================================================================
#  KIEU B — dieu khien bang UART toi STM8
#  Bo khoi "switch" o tren, bo comment khoi nay.
# =============================================================================
# uart:
#   tx_pin: GPIO1
#   rx_pin: GPIO3
#   baud_rate: 115200
#
# switch:
#   - platform: template
#     id: relay_out
#     optimistic: true
#     restore_mode: RESTORE_DEFAULT_OFF
#     internal: true
#     turn_on_action:
#       - uart.write: [0xA0, 0x01, 0x01, 0xA2]
#     turn_off_action:
#       - uart.write: [0xA0, 0x01, 0x00, 0xA1]

# -----------------------------------------------------------------------------
# Expose ra HA duoi dang light (de HomeKit bridge hien dung icon bong den)
# -----------------------------------------------------------------------------
light:
  - platform: binary
    id: main_light
    name: ${friendly_name}
    output: relay_output

output:
  - platform: template
    id: relay_output
    type: binary
    write_action:
      - if:
          condition:
            lambda: 'return state;'
          then:
            - switch.turn_on: relay_out
          else:
            - switch.turn_off: relay_out

# -----------------------------------------------------------------------------
# Watchdog cuc bo: neu den dang bat qua 2 tieng ma khong ai tat (vi du HA chet
# giua chung), tu tat. Chan truong hop den sang suot dem.
# -----------------------------------------------------------------------------
interval:
  - interval: 60s
    then:
      - if:
          condition:
            and:
              - light.is_on: main_light
              - lambda: 'return id(on_seconds) > 7200;'
          then:
            - light.turn_off: main_light
            - lambda: 'id(on_seconds) = 0;'
      - lambda: |-
          if (id(main_light).current_values.is_on()) {
            id(on_seconds) += 60;
          } else {
            id(on_seconds) = 0;
          }

globals:
  - id: on_seconds
    type: int
    restore_value: false
    initial_value: '0'

binary_sensor:
  - platform: status
    name: "${friendly_name} Status"
    entity_category: diagnostic

---

Tốt, ảnh này giải quyết được câu hỏi quan trọng nhất: **board của bạn là Kiểu A** — không có con STM8S003F3 nào trên PCB. Chỉ có socket 8 chân, một transistor/LDO dạng SOT, relay và terminal vít. Nghĩa là điều khiển bằng GPIO trực tiếp, dùng được khối `switch: platform: gpio` trong `r1.yaml`, xóa hẳn khối UART đang comment.

Vài điểm đọc được từ ảnh:

**Relay là SRD-03VDC-SL-C** — cuộn dây 3V, không phải 5V. Đây là tin tốt: nó được thiết kế để chạy thẳng từ rail 3.3V, dòng cuộn khoảng 80–100mA. Cộng với peak WiFi của ESP-01S (~300mA), tổng đỉnh tầm 400mA. LDO trên board loại này thường khá yếu, nên **nguồn cấp vào phải là adapter riêng ≥ 1A**, đừng lấy chung với nguồn của node khác.

**ESP-01S tháo rời được** (ảnh 2 cho thấy nó là module cắm socket). Tận dụng điều này: flash lần đầu bằng USB-TTL riêng rồi mới cắm vào board relay — đỡ phải mò chân GPIO0 trên board relay, và tránh rủi ro cấp nhầm điện. Từ lần 2 trở đi OTA.

## Hai thứ ảnh không trả lời được, phải đo

**1. Điện áp cấp vào là 5V hay 3.3V.** Board này có hai biến thể. Nhìn mặt sau chỗ terminal vít / chân nguồn xem có silkscreen ghi `5V` hay `3.3V`. Nếu không rõ: dò thông mạch (continuity) từ chân nguồn đến chân `3V3` của socket ESP-01S. Thông mạch trực tiếp → board 3.3V, cấp 5V vào là **cháy ESP-01S ngay**. Không thông (đi qua con SOT) → board 5V có LDO.

**2. Active HIGH hay active LOW.** Cách nhanh nhất, chưa cần flash gì: cấp nguồn cho board nhưng **chưa cắm ESP-01S**, rồi lấy dây jumper chạm chân `IO0` của socket xuống `GND`. 
- Relay click khi chạm GND → **active LOW** → sửa `inverted: false` thành `inverted: true`.
- Relay không click, mà click khi chạm `IO0` lên `3V3` → **active HIGH** → giữ nguyên `inverted: false`.

Làm bước này trước khi đấu bất cứ dây AC nào.

## Về cú click lúc boot

Với board Kiểu A active-HIGH này, hiện tượng relay đóng một nhịp khi cấp nguồn là **chắc chắn xảy ra**, vì board có điện trở pull-up trên GPIO0 để ESP boot được ở chế độ bình thường. Firmware không can thiệp được — lúc đó ESP còn chưa chạy code.

Ba lựa chọn, theo thứ tự tôi khuyên:

**Chấp nhận.** Đèn chớp ~200–300ms mỗi lần mất điện rồi có lại. Với đèn hành lang thì không ai để ý.

**Đổi sang dùng GPIO2.** Cắt đường mạch từ IO0 đến chân base của transistor, chạy dây từ IO2 sang. Nhưng GPIO2 **cũng** phải HIGH lúc boot, nên bạn không giải quyết được gì — chỉ có ý nghĩa nếu bạn cần giải phóng GPIO0 cho việc khác. Tôi không khuyên.

**Thêm tầng đảo.** Chèn một transistor NPN làm inverter giữa GPIO0 và mạch drive hiện có, để GPIO0 HIGH = relay nhả. Lúc boot relay im hoàn toàn. Nhược điểm: khi relay đang đóng, GPIO0 đang bị ESP kéo LOW — nếu lúc đó có ai bấm reset cứng, ESP sẽ vào flash mode và treo cho đến khi cắt nguồn. Chỉ làm nếu cú click thật sự gây khó chịu, và nhớ để `restore_mode: RESTORE_DEFAULT_OFF`.

## Sửa trong `r1.yaml`

Cụ thể ba chỗ:

```yaml
# 1. Xóa toàn bộ khối "KIEU B — UART" đang comment (đỡ nhầm về sau)

# 2. Trong switch, xác nhận lại sau khi đo ở bước 2 phía trên:
    pin:
      number: GPIO0
      inverted: false     # <- đổi thành true nếu đo ra active LOW

# 3. logger: có thể bật lại vì Kiểu A không dùng UART
logger:
  level: WARN
  baud_rate: 115200
```

Để `baud_rate: 115200` giúp bạn xem được log qua USB-TTL khi debug lần đầu. Sau khi chạy ổn thì đặt về `0` — chân TX của ESP-01S trên board relay để hở, và log chạy liên tục ra một chân hở nằm ngay cạnh mạch relay là thêm một nguồn nhiễu không cần thiết.

Còn một điểm về đấu nối AC: terminal relay có 3 chân NO/COM/NC. Đấu **dây lửa (L) vào COM, ra NO** để mặc định khi mất điện điều khiển thì đèn tắt. Dây nguội (N) đi thẳng tới đèn, không qua relay. Đừng đảo lại — cắt dây nguội thì đèn tắt nhưng đui đèn vẫn còn điện, rất nguy hiểm khi thay bóng.
