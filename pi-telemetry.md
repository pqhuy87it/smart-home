Cách gọn và bền nhất là đi qua **MQTT** — bạn đã có mosquitto chạy sẵn, và MQTT Discovery sẽ tự tạo entity trong HA, không cần sửa `configuration.yaml` mỗi lần thêm chỉ số.

Không dùng `command_line` sensor được, vì `vcgencmd` là binary của host, không có trong container HA. Cũng không nên dùng `file` sensor như mình gợi ý ở lượt trước — integration `file` đã chuyển sang cấu hình qua UI từ 2024, YAML kiểu cũ không còn dùng được trên 2026.8.

## Bước 1 — Cài công cụ trên host

```bash
sudo apt update
sudo apt install -y mosquitto-clients jq nvme-cli
mosquitto_pub --help | head -2
```

## Bước 2 — File credentials

Tách riêng để không nhúng password vào script:

```bash
sudo tee /etc/pi-telemetry.env >/dev/null <<'EOF'
MQTT_HOST=127.0.0.1
MQTT_PORT=1883
MQTT_USER=hass
MQTT_PASS=MẬT_KHẨU_MQTT_CỦA_BẠN
NODE_ID=pi5_exlinct
NVME_DEV=/dev/nvme0n1
EOF
sudo chmod 600 /etc/pi-telemetry.env
sudo chown root:root /etc/pi-telemetry.env
```

Sửa `MQTT_PASS` cho khớp user `hass` bạn đã tạo lúc set up mosquitto.

## Bước 3 — Script thu thập

```bash
sudo tee /usr/local/bin/pi-telemetry.sh >/dev/null <<'SCRIPT'
#!/usr/bin/env bash
set -uo pipefail

set -a; . /etc/pi-telemetry.env; set +a

STATE_TOPIC="pi5/${NODE_ID}/state"
VCG=/usr/bin/vcgencmd

# --- công suất board + điện áp đầu vào ---
read -r BOARD_W EXT5V < <(
  "$VCG" pmic_read_adc | awk '
    /current/ { n=$1; sub(/_A$/,"",n); v=$2; sub(/.*=/,"",v); sub(/A$/,"",v); amp[n]=v+0 }
    /volt/    { n=$1; sub(/_V$/,"",n); v=$2; sub(/.*=/,"",v); sub(/V$/,"",v); vol[n]=v+0 }
    END {
      for (n in amp) if (n in vol) total += amp[n]*vol[n]
      printf "%.3f %.4f\n", total, vol["EXT5V"]
    }'
) || { BOARD_W=0; EXT5V=0; }

# --- throttled flags ---
THR_RAW=$("$VCG" get_throttled | cut -d= -f2)
THR=$((THR_RAW))
uv_now=$(( (THR & 0x1)     ? 1 : 0 ))
cap_now=$(( (THR & 0x2)    ? 1 : 0 ))
thr_now=$(( (THR & 0x4)    ? 1 : 0 ))
uv_ever=$(( (THR & 0x10000)? 1 : 0 ))
thr_ever=$(((THR & 0x40000)? 1 : 0 ))

# --- CPU ---
CPU_TEMP=$("$VCG" measure_temp | tr -dc '0-9.')
CPU_MHZ=$(( $("$VCG" measure_clock arm | cut -d= -f2) / 1000000 ))
LOAD1=$(cut -d' ' -f1 /proc/loadavg)

# --- NVMe ---
NVME_TEMP=null
NVME_PCT=null
if [ -e "$NVME_DEV" ]; then
  NJ=$(nvme smart-log "$NVME_DEV" -o json 2>/dev/null) || NJ=""
  if [ -n "$NJ" ]; then
    t=$(jq -r '.temperature // empty' <<<"$NJ")
    # nvme-cli trả Kelvin ở một số version
    if [ -n "$t" ]; then
      [ "$t" -gt 200 ] 2>/dev/null && t=$((t - 273))
      NVME_TEMP=$t
    fi
    NVME_PCT=$(jq -r '.percent_used // "null"' <<<"$NJ")
  fi
fi

PAYLOAD=$(jq -nc \
  --argjson board_w "$BOARD_W" \
  --argjson ext5v "$EXT5V" \
  --argjson cpu_temp "$CPU_TEMP" \
  --argjson cpu_mhz "$CPU_MHZ" \
  --argjson load1 "$LOAD1" \
  --argjson nvme_temp "$NVME_TEMP" \
  --argjson nvme_pct "$NVME_PCT" \
  --arg thr_raw "$THR_RAW" \
  --argjson uv_now "$uv_now" --argjson cap_now "$cap_now" --argjson thr_now "$thr_now" \
  --argjson uv_ever "$uv_ever" --argjson thr_ever "$thr_ever" \
  '{board_w:$board_w, ext5v:$ext5v, cpu_temp:$cpu_temp, cpu_mhz:$cpu_mhz,
    load1:$load1, nvme_temp:$nvme_temp, nvme_pct:$nvme_pct,
    throttled_raw:$thr_raw, uv_now:$uv_now, cap_now:$cap_now,
    thr_now:$thr_now, uv_ever:$uv_ever, thr_ever:$thr_ever}')

exec mosquitto_pub -h "$MQTT_HOST" -p "$MQTT_PORT" \
  -u "$MQTT_USER" -P "$MQTT_PASS" \
  -t "$STATE_TOPIC" -m "$PAYLOAD"
SCRIPT

sudo chmod 755 /usr/local/bin/pi-telemetry.sh
```

Test ngay, xem JSON có đúng không:

```bash
sudo bash -c 'set -a; . /etc/pi-telemetry.env; set +a; /usr/local/bin/pi-telemetry.sh && echo "PUBLISH OK"'
```

Xem thẳng payload trước khi publish:

```bash
sudo sed 's/^exec mosquitto_pub/echo "$PAYLOAD"; exec true; mosquitto_pub/' /usr/local/bin/pi-telemetry.sh | sudo bash
```

Kỳ vọng một dòng JSON có đủ các field. Nếu `board_w` ra 0 thì `vcgencmd` bị lỗi quyền — script chạy bằng root nên không nên xảy ra.

## Bước 4 — Publish MQTT Discovery

Chạy **một lần**, các message `retained` nên HA nhận lại được sau mỗi lần restart:

```bash
sudo tee /usr/local/bin/pi-telemetry-discovery.sh >/dev/null <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
set -a; . /etc/pi-telemetry.env; set +a

ST="pi5/${NODE_ID}/state"
DEV=$(jq -nc --arg id "$NODE_ID" \
  '{identifiers:[$id], name:"Raspberry Pi 5 (exlinct)",
    model:"Raspberry Pi 5", manufacturer:"Raspberry Pi"}')

pub() { # $1=component $2=object_id $3=config-json
  mosquitto_pub -h "$MQTT_HOST" -p "$MQTT_PORT" -u "$MQTT_USER" -P "$MQTT_PASS" \
    -t "homeassistant/$1/${NODE_ID}/$2/config" -m "$3" -r
}

sensor() { # name key unit device_class state_class icon
  pub sensor "$2" "$(jq -nc \
    --arg n "$1" --arg k "$2" --arg u "$3" --arg dc "$4" --arg sc "$5" --arg ic "$6" \
    --arg st "$ST" --arg id "${NODE_ID}_$2" --argjson dev "$DEV" \
    '{name:$n, state_topic:$st, unique_id:$id, device:$dev,
      value_template:("{{ value_json." + $k + " }}"),
      expire_after:180}
     + (if $u  != "" then {unit_of_measurement:$u} else {} end)
     + (if $dc != "" then {device_class:$dc} else {} end)
     + (if $sc != "" then {state_class:$sc} else {} end)
     + (if $ic != "" then {icon:$ic} else {} end)')"
}

binsens() { # name key device_class
  pub binary_sensor "$2" "$(jq -nc \
    --arg n "$1" --arg k "$2" --arg dc "$3" \
    --arg st "$ST" --arg id "${NODE_ID}_$2" --argjson dev "$DEV" \
    '{name:$n, state_topic:$st, unique_id:$id, device:$dev,
      value_template:("{{ value_json." + $k + " }}"),
      payload_on:1, payload_off:0, expire_after:180}
     + (if $dc != "" then {device_class:$dc} else {} end)')"
}

sensor "Board power"        board_w   W    power       measurement ""
sensor "Input voltage"      ext5v     V    voltage     measurement ""
sensor "CPU temperature"    cpu_temp  "°C" temperature measurement ""
sensor "CPU frequency"      cpu_mhz   MHz  frequency   measurement ""
sensor "Load 1m"            load1     ""   ""          measurement mdi:speedometer
sensor "NVMe temperature"   nvme_temp "°C" temperature measurement ""
sensor "NVMe wear"          nvme_pct  "%"  ""          measurement mdi:harddisk
sensor "Throttled flags"    throttled_raw "" ""        ""          mdi:flag

binsens "Undervoltage now"        uv_now   problem
binsens "Frequency capped now"    cap_now  problem
binsens "Throttled now"           thr_now  problem
binsens "Undervoltage since boot" uv_ever  problem
binsens "Throttled since boot"    thr_ever problem

echo "Discovery published."
SCRIPT

sudo chmod 755 /usr/local/bin/pi-telemetry-discovery.sh
sudo /usr/local/bin/pi-telemetry-discovery.sh
```

`expire_after: 180` là chi tiết quan trọng: nếu script chết, entity sẽ chuyển sang `unavailable` thay vì đứng im ở giá trị cũ và làm bạn tưởng mọi thứ vẫn bình thường.

## Bước 5 — systemd timer

Dùng timer thay cho cron vì log rõ hơn và kiểm soát được thời điểm chạy:

```bash
sudo tee /etc/systemd/system/pi-telemetry.service >/dev/null <<'EOF'
[Unit]
Description=Publish Pi 5 telemetry to MQTT
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/pi-telemetry.env
ExecStart=/usr/local/bin/pi-telemetry.sh
EOF

sudo tee /etc/systemd/system/pi-telemetry.timer >/dev/null <<'EOF'
[Unit]
Description=Run Pi 5 telemetry every 30s

[Timer]
OnBootSec=60
OnUnitActiveSec=30
AccuracySec=1s
Unit=pi-telemetry.service

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now pi-telemetry.timer
```

Verify:

```bash
systemctl list-timers pi-telemetry.timer --no-pager
sudo systemctl status pi-telemetry.service --no-pager -n 20
journalctl -u pi-telemetry.service -n 20 --no-pager
```

Xem message có thật sự vào broker không:

```bash
mosquitto_sub -h 127.0.0.1 -u hass -P 'MẬT_KHẨU' -t "pi5/pi5_exlinct/state" -C 2 -v
```

Phải in ra 2 dòng JSON trong vòng ~60 giây.

## Bước 6 — Kiểm tra trong HA

**Settings → Devices & Services → MQTT → devices**. Phải thấy device *Raspberry Pi 5 (exlinct)* với 13 entity.

Nếu chưa thấy: **Developer Tools → MQTT → Listen to a topic**, nhập `homeassistant/#` rồi bấm Start listening. Không có gì hiện ra nghĩa là HA và script đang nói với hai broker khác nhau — kiểm tra lại config MQTT integration trỏ đúng `127.0.0.1:1883`.

## Bước 7 — Dashboard

Thêm card vào dashboard (**Edit → Add card → Manual**):

```yaml
type: vertical-stack
cards:
  - type: entities
    title: Pi 5 — nguồn & tải
    entities:
      - entity: sensor.raspberry_pi_5_exlinct_input_voltage
      - entity: sensor.raspberry_pi_5_exlinct_board_power
      - entity: sensor.raspberry_pi_5_exlinct_cpu_temperature
      - entity: sensor.raspberry_pi_5_exlinct_nvme_temperature
      - entity: sensor.raspberry_pi_5_exlinct_load_1m
      - type: divider
      - entity: binary_sensor.raspberry_pi_5_exlinct_undervoltage_now
      - entity: binary_sensor.raspberry_pi_5_exlinct_undervoltage_since_boot
      - entity: binary_sensor.raspberry_pi_5_exlinct_throttled_since_boot
  - type: history-graph
    hours_to_show: 24
    entities:
      - sensor.raspberry_pi_5_exlinct_input_voltage
  - type: statistics-graph
    chart_type: line
    period: 5minute
    days_to_show: 2
    stat_types: [min, mean]
    entities:
      - sensor.raspberry_pi_5_exlinct_input_voltage
```

Tên entity thực tế có thể khác chút — copy chính xác từ Developer Tools → States, filter `exlinct`.

Card `statistics-graph` với `stat_types: [min, mean]` là cái đáng giá nhất: **min** cho biết điện áp tụt sâu nhất trong từng khoảng 5 phút. Đây chính là dữ liệu bạn cần để chốt vấn đề nguồn — nhìn được lúc nào áp rơi và nó tương ứng với việc gì.

## Bước 8 — Automation cảnh báo

```yaml
alias: Pi5 - canh bao undervoltage
mode: single
triggers:
  - trigger: state
    entity_id: binary_sensor.raspberry_pi_5_exlinct_undervoltage_now
    to: "on"
  - trigger: numeric_state
    entity_id: sensor.raspberry_pi_5_exlinct_input_voltage
    below: 4.75
    for: "00:00:30"
actions:
  - action: persistent_notification.create
    data:
      title: Pi 5 sut dien ap
      message: >
        Vin={{ states('sensor.raspberry_pi_5_exlinct_input_voltage') }}V,
        board={{ states('sensor.raspberry_pi_5_exlinct_board_power') }}W,
        flags={{ states('sensor.raspberry_pi_5_exlinct_throttled_flags') }}
```

Ngưỡng `4.75V` chọn theo tình trạng của bạn: idle đang 4.954V, ngưỡng cứng là 4.63V, nên 4.75V là mức cảnh báo sớm còn kịp phản ứng.

## Bước 9 — Chạy stress test và xem trực tiếp

Giờ mới đến phần bạn hỏi ban đầu — theo dõi lúc chạy tải:

```bash
sudo reboot   # để flag throttled reset về 0
```

Sau khi lên, mở dashboard HA trên MacBook rồi chạy trên Pi:

```bash
stress-ng --cpu 4 --io 2 --timeout 300s
```

Vừa xem `history-graph` điện áp tụt tới đâu theo thời gian thực. Kết thúc, đối chiếu:

```bash
vcgencmd get_throttled
dmesg -T | grep -iE 'voltage|nvme.*(reset|error|timeout)'
```

Trong HA, mở entity *Input voltage* → chọn khoảng thời gian test → xem giá trị min. Đó là con số quyết định: nếu min vẫn trên 4.8V thì nguồn đủ, an toàn để clone sang NVMe. Nếu chạm quanh 4.63V hoặc `Undervoltage since boot` bật lên thì xử lý cáp/adapter trước.

## Bổ sung — System Monitor

Thêm luôn cho đủ bức tranh: **Settings → Devices & Services → Add Integration → System Monitor**. Nó cho RAM, disk, network của host (chạy được vì container đang ở `network_mode: host` và `privileged`). Không có PMIC nên không thay thế được script trên, nhưng bổ sung tốt phần RAM và dung lượng đĩa.
