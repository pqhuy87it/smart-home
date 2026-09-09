Được. Vì anh đã chốt đi hướng WiFi/ESPHome (chưa cần dongle Zigbee), stack sẽ gọn hơn bản tôi viết ở đầu. Tôi viết lại từ đầu cho đúng scope hiện tại.

## Cần cài những gì, và tại sao

| Phần mềm | Vai trò | Có bắt buộc? |
|---|---|---|
| **Docker** | Chạy mọi thứ trong container, dễ backup/migrate | Có |
| **Home Assistant** | Trung tâm: entity, automation, dashboard, HomeKit bridge | Có |
| **ESPHome** | Sinh firmware + OTA cho các board C3/8266 | Có |
| **Mosquitto** | MQTT broker cho firmware anh **tự viết** | Xem ghi chú dưới |
| **Tailscale** | Truy cập từ xa an toàn, không mở port | Nên có |

**Một điểm cần nói rõ trước khi cài**: node dùng **ESPHome không cần MQTT**. ESPHome nói với HA qua **native API riêng** (port 6053, có encryption), nhanh hơn và ít thành phần hơn MQTT.

Nên Mosquitto ở đây không phải để phục vụ ESPHome, mà để phục vụ **các board firmware anh tự viết** — InfoClock, print monitor A1 mini, ClaudeMonitor, IR blaster. Những con đó publish MQTT thì HA đọc được. Vẫn nên cài, chỉ là đừng nhầm vai trò của nó.

---

# Bước 0 — Chuẩn bị host

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git vim htop curl avahi-daemon jq

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Xác nhận
docker --version
docker compose version
```

Kiểm tra mDNS (cần cho việc ESPHome tìm board và cho anh gõ `smarthome.local` từ Mac):

```bash
systemctl is-active avahi-daemon      # → active
```

---

# Bước 1 — Cấu trúc thư mục

```bash
mkdir -p ~/smarthome/{mosquitto/{config,data,log},homeassistant,esphome}
cd ~/smarthome
git init                               # version control cho config, rất đáng làm
```

Tạo `.env` để tập trung các biến:

```bash
cat > .env <<'EOF'
TZ=Asia/Ho_Chi_Minh
PUID=1000
PGID=1000
EOF
```

Và `.gitignore`:

```bash
cat > .gitignore <<'EOF'
mosquitto/data/
mosquitto/log/
mosquitto/config/passwd
homeassistant/*.db*
homeassistant/.storage/
homeassistant/home-assistant.log*
homeassistant/tts/
esphome/.esphome/
esphome/secrets.yaml
.env
EOF
```

---

# Bước 2 — Mosquitto

## 2.1. File cấu hình

```bash
cat > mosquitto/config/mosquitto.conf <<'EOF'
listener 1883
protocol mqtt

# WebSocket — hữu ích nếu sau này anh viết web dashboard hoặc debug bằng MQTT Explorer
listener 9001
protocol websockets

persistence true
persistence_location /mosquitto/data/
autosave_interval 60

log_dest stdout
log_type error
log_type warning
log_type notice

allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl

# Giữ message cuối của mỗi topic, để HA có state ngay khi restart
max_queued_messages 1000
EOF
```

## 2.2. Tạo user

```bash
cd ~/smarthome

# User đầu tiên (-c tạo file mới, CHỈ dùng lần này)
docker run --rm -it -v ./mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 mosquitto_passwd -c /mosquitto/config/passwd hass

# Các user tiếp theo (KHÔNG có -c, nếu không sẽ xoá file cũ)
for u in infoclock printmon-a1 irblaster claudemonitor; do
  docker run --rm -it -v ./mosquitto/config:/mosquitto/config \
    eclipse-mosquitto:2 mosquitto_passwd /mosquitto/config/passwd $u
done
```

**Mỗi board một user riêng** — không dùng chung một tài khoản. Nếu một board bị chiếm quyền hoặc firmware lỗi publish bừa, nó chỉ phá được topic của chính nó.

## 2.3. ACL

```bash
cat > mosquitto/config/acl <<'EOF'
# Home Assistant: toàn quyền, cần để discovery và điều khiển
user hass
topic readwrite #

# Mỗi board chỉ được ghi vào namespace của mình,
# và chỉ đọc topic lệnh của mình
user infoclock
topic write  devices/infoclock/#
topic read   devices/infoclock/cmd/#

user printmon-a1
topic write  devices/printmon-a1/#
topic read   devices/printmon-a1/cmd/#
# đọc thêm state máy in do HA publish lại
topic read   ha/printer/a1mini/#

user irblaster
topic write  devices/irblaster/#
topic read   devices/irblaster/cmd/#

user claudemonitor
topic write  devices/claudemonitor/#
topic read   devices/claudemonitor/cmd/#
EOF
```

Quy ước topic: `devices/<tên>/state`, `devices/<tên>/cmd/<gì>`. Cố định ngay từ đầu, đừng để mỗi board một kiểu — sau 8 board là không quản lý được nữa.

---

# Bước 3 — `docker-compose.yml`

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    network_mode: host          # đơn giản hoá: board ESP nối trực tiếp IP LAN của Pi
    volumes:
      - ./mosquitto/config:/mosquitto/config:ro
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    environment:
      - TZ=${TZ}

  homeassistant:
    # Ghim version thay vì :stable — để anh chủ động chọn thời điểm update
    image: ghcr.io/home-assistant/home-assistant:2026.8
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host          # BẮT BUỘC: cần cho mDNS discovery + HomeKit bridge
    privileged: true
    volumes:
      - ./homeassistant:/config
      - /run/dbus:/run/dbus:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=${TZ}

  esphome:
    image: ghcr.io/esphome/esphome:stable
    container_name: esphome
    restart: unless-stopped
    network_mode: host          # cần cho mDNS để tìm board khi OTA
    privileged: true            # cần khi flash board qua USB lần đầu
    volumes:
      - ./esphome:/config
      - /etc/localtime:/etc/localtime:ro
      - /dev:/dev                # cho phép flash USB
    environment:
      - TZ=${TZ}
      - ESPHOME_DASHBOARD_USE_PING=true   # fallback khi mDNS không ổn
```

**Về `network_mode: host` cho cả ba**: đánh đổi network isolation để lấy sự đơn giản. Trong LAN nhà thì hợp lý, và nó tránh hẳn cái bẫy "HA không thấy Mosquitto" mà tôi đã cảnh báo ở phần trước. Các port: HA `8123`, ESPHome `6052`, Mosquitto `1883`/`9001` — không xung đột.

**Về việc ghim version HA**: HA release mỗi tháng và **có breaking change thật**. Ghim version rồi đọc release notes trước khi nâng là cách duy nhất để không bị hỏng hệ thống vào một buổi tối bất kỳ. Kiểm tra version mới nhất trên trang release của HA rồi thay số vào.

---

# Bước 4 — Khởi động và nghiệm thu từng service

```bash
cd ~/smarthome
docker compose up -d
docker compose ps          # cả 3 phải là "running", không phải "restarting"
```

## Kiểm tra Mosquitto

```bash
# Terminal 1 — subscribe
docker exec -it mosquitto mosquitto_sub -u hass -P '<mat_khau>' -t 'test/#' -v

# Terminal 2 — publish
docker exec -it mosquitto mosquitto_pub -u hass -P '<mat_khau>' -t 'test/hello' -m 'ok'
```

Terminal 1 phải hiện `test/hello ok`. Nếu báo `Connection Refused: not authorised` → sai mật khẩu hoặc ACL chặn.

Test ACL đang hoạt động (phải **thất bại**):

```bash
docker exec -it mosquitto mosquitto_pub -u infoclock -P '<mat_khau>' \
  -t 'devices/irblaster/state' -m 'hack'
# → phải bị từ chối
```

## Kiểm tra Home Assistant

```bash
docker compose logs homeassistant | tail -30
# → "Home Assistant initialized in Xs"
```

Mở `http://smarthome.local:8123` từ Mac, tạo tài khoản admin.

## Kiểm tra ESPHome

Mở `http://smarthome.local:6052`. Không có màn hình login mặc định — **đặt password ngay** trong `esphome/secrets.yaml` hoặc chặn bằng firewall nếu anh có lo ngại.

---

# Bước 5 — Cấu hình Home Assistant

## 5.1. MQTT integration

Settings → Devices & Services → Add Integration → MQTT:

```
Broker:   127.0.0.1        (vì HA đang ở host network)
Port:     1883
Username: hass
Password: <mật khẩu đã tạo>
```

## 5.2. Recorder — làm ngay, đừng để sau

Mặc định HA ghi **mọi** state change vào SQLite. Với vài chục entity kèm RSSI/uptime cập nhật liên tục, database sẽ phình lên vài GB và làm dashboard chậm dần.

Tạo `homeassistant/configuration.yaml`:

```yaml
default_config:

homeassistant:
  name: Nhà
  latitude: 21.0278
  longitude: 105.8342
  elevation: 16
  unit_system: metric
  time_zone: Asia/Ho_Chi_Minh
  country: VN
  currency: VND

recorder:
  purge_keep_days: 30
  commit_interval: 30          # gộp ghi, giảm I/O
  exclude:
    domains:
      - automation
      - update
      - script
    entity_globs:
      - sensor.*_wifi_rssi
      - sensor.*_uptime
      - sensor.*_ip
      - sensor.*_free_memory
    entities:
      - sun.sun

# Giữ lịch sử dài hạn ở dạng thống kê (min/max/mean theo giờ)
# — nhẹ hơn recorder rất nhiều, dùng cho nhiệt độ và mực nước
logger:
  default: warning
  logs:
    homeassistant.components.mqtt: info

frontend:
  themes: !include_dir_merge_named themes/
```

Các entity nhiệt độ/độ ẩm/mực nước có `state_class: measurement` sẽ tự vào **long-term statistics**, giữ được nhiều năm ở dạng tổng hợp theo giờ mà không phình database. Đây là lý do nên loại RSSI/uptime khỏi recorder nhưng vẫn giữ nhiệt độ.

Restart HA sau khi sửa:

```bash
docker compose restart homeassistant
```

## 5.3. HomeKit Bridge

Settings → Devices & Services → Add Integration → **HomeKit Bridge**. Chọn domain muốn expose (`switch`, `sensor`, `light`, `climate`).

Nó sinh mã QR trong notification. Quét bằng app Home trên iPad là xong.

Hai lưu ý:
- Bridge chỉ hoạt động khi HA ở **host network** — nếu để bridge network thì iPad không tìm thấy.
- Entity **mực nước %** sẽ không map được sang characteristic HomeKit nào hợp lý. Cứ **loại nó ra khỏi bridge**, để dành cho app SwiftUI hoặc dashboard HA. Đừng cố nhồi nó thành sensor độ ẩm.

---

# Bước 6 — Node ESPHome đầu tiên

```bash
cat > esphome/secrets.yaml <<'EOF'
wifi_ssid: "TenWifiCuaAnh"
wifi_password: "MatKhauWifi"
ota_password: "mat_khau_ota_dai"
api_key: ""          # để trống, ESPHome sẽ sinh khi tạo device đầu tiên
EOF
chmod 600 esphome/secrets.yaml
```

Vào `http://smarthome.local:6052` → **New Device** → đặt tên `sensor-t2` → chọn ESP32-C3 (hoặc ESP8266). ESPHome tự sinh `api.encryption.key`.

Lần flash đầu **phải qua USB**: cắm board vào Pi, chọn Install → Plug into this computer. Từ lần thứ hai trở đi là OTA qua WiFi.

Sau khi board online, HA sẽ tự phát hiện qua mDNS và hiện notification "New device discovered" — bấm configure, dán api key, xong.

Dùng đúng file `common/base-c3.yaml` và các config tôi đã viết ở tin nhắn trước, đặt vào `esphome/`.

---

# Bước 7 — Tailscale (truy cập từ xa)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Cài app Tailscale trên iPad và Mac, đăng nhập cùng account. Sau đó `http://100.x.x.x:8123` chạy được từ bất cứ đâu, **không mở một port nào ra internet**.

Nếu muốn dùng hostname ngắn, bật MagicDNS trong admin console của Tailscale.

Đừng port-forward 8123 ra internet. Đó là cách nhanh nhất để HA của anh xuất hiện trên Shodan.

---

# Bước 8 — Backup tự động

```bash
mkdir -p ~/backups
cat > ~/smarthome/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

SRC="$HOME/smarthome"
DST="$HOME/backups"
STAMP=$(date +%Y%m%d-%H%M)
KEEP=14

cd "$SRC"

# Dừng HA để SQLite ở trạng thái nhất quán
docker compose stop homeassistant

tar czf "$DST/smarthome-$STAMP.tar.gz" \
  --exclude='homeassistant/.storage/*.log' \
  --exclude='homeassistant/tts' \
  --exclude='esphome/.esphome/build' \
  --exclude='mosquitto/log' \
  -C "$HOME" smarthome

docker compose start homeassistant

# Xoá bản cũ
ls -1t "$DST"/smarthome-*.tar.gz | tail -n +$((KEEP+1)) | xargs -r rm --

echo "Backup xong: $DST/smarthome-$STAMP.tar.gz"
EOF

chmod +x ~/smarthome/backup.sh
```

Chạy hàng đêm 3h sáng:

```bash
crontab -e
```

```cron
0 3 * * * /home/huy/smarthome/backup.sh >> /home/huy/backups/backup.log 2>&1
```

**Đẩy sang máy khác** — backup nằm cùng ổ với dữ liệu thì không phải backup. Anh có Mac mini M4 chạy thường xuyên, rsync sang đó:

```cron
30 3 * * * rsync -az --delete /home/huy/backups/ huy@<ip-mac-mini>:~/pi-backups/
```

Nhớ tạo SSH key từ Pi sang Mac mini để cron chạy không cần mật khẩu.

Ngoài ra bật cả **HA Backup** built-in (Settings → System → Backups) — nó backup ở tầng logic, phục hồi entity/automation dễ hơn tar file.

---

# Bước 9 — Cập nhật an toàn

```bash
cd ~/smarthome

# 1. Backup TRƯỚC
./backup.sh

# 2. Đọc release notes của HA, rồi sửa số version trong docker-compose.yml
vim docker-compose.yml

# 3. Pull và restart
docker compose pull
docker compose up -d

# 4. Xem log ngay
docker compose logs -f homeassistant

# 5. Nếu hỏng: sửa version về số cũ, up -d lại
```

**Đừng dùng Watchtower để auto-update Home Assistant.** Nó sẽ nâng version giữa đêm, gặp breaking change, và anh thức dậy với hệ thống chết mà không biết vì sao.

OS thì bật auto security update được:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

# Checklist nghiệm thu

```bash
# 1. Ba container chạy, không restart loop
docker compose ps

# 2. Pi không bị throttle/thiếu điện
vcgencmd get_throttled              # → 0x0

# 3. Boot từ NVMe
findmnt / | grep nvme

# 4. Mosquitto nhận auth + ACL đúng
docker exec -it mosquitto mosquitto_sub -u hass -P '<pw>' -t '$SYS/broker/version' -v -C 1

# 5. HA lên
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8123   # → 200

# 6. ESPHome lên
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:6052   # → 200

# 7. mDNS hoạt động (chạy từ Mac)
ping smarthome.local

# 8. Backup chạy được
~/smarthome/backup.sh && ls -lh ~/backups/

# 9. Dung lượng ổ
df -h /
```

Xong bước này thì nền tảng đã sẵn sàng — từ đây chỉ còn thêm node và viết automation.

Anh muốn tôi làm tiếp phần nào: file `common/base-c3.yaml` hoàn chỉnh kèm node nhiệt độ đầu tiên để anh flash luôn, hay dashboard HA cho nhiều tầng?

---

Nó chỉ là một file text nằm trong `~/smarthome/`. Có bốn cách, tôi xếp theo mức tiện cho anh.

## Cách 1 — Heredoc, paste một lần là xong

Đây là cách nhanh nhất qua SSH. Dán cả block này vào terminal:

```bash
cd ~/smarthome

cat > docker-compose.yml <<'EOF'
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./mosquitto/config:/mosquitto/config:ro
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    environment:
      - TZ=${TZ}

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:2026.8
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host
    privileged: true
    volumes:
      - ./homeassistant:/config
      - /run/dbus:/run/dbus:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=${TZ}

  esphome:
    image: ghcr.io/esphome/esphome:stable
    container_name: esphome
    restart: unless-stopped
    network_mode: host
    privileged: true
    volumes:
      - ./esphome:/config
      - /etc/localtime:/etc/localtime:ro
      - /dev:/dev
    environment:
      - TZ=${TZ}
      - ESPHOME_DASHBOARD_USE_PING=true
EOF
```

**Chi tiết quan trọng**: dấu nháy đơn quanh `'EOF'`. Có nháy thì shell **không** thay thế biến, nên `${TZ}` được ghi nguyên văn vào file để Docker Compose tự đọc từ `.env`. Nếu anh viết `<<EOF` không nháy, shell sẽ thay `${TZ}` bằng chuỗi rỗng ngay lúc tạo file và anh mất timezone.

## Cách 2 — Editor trên Pi

```bash
cd ~/smarthome
nano docker-compose.yml     # Ctrl+O lưu, Ctrl+X thoát
```

Nếu dùng `vim`, bật chế độ paste trước khi dán để tránh auto-indent làm hỏng thụt lề:

```
:set paste
i
[dán]
Esc
:wq
```

## Cách 3 — VS Code Remote-SSH (tôi khuyên cách này)

Với một dev thì đây là cách thoải mái nhất. Trên Mac mini:

1. Cài extension **Remote - SSH** trong VS Code.
2. `Cmd+Shift+P` → Remote-SSH: Connect to Host → `huy@smarthome.local`.
3. Open Folder → `/home/huy/smarthome`.

Từ đó anh sửa `docker-compose.yml`, `configuration.yaml`, các file ESPHome bằng editor quen, có syntax highlight và YAML validation. Cài thêm extension **Docker** và **Home Assistant Config Helper** thì được autocomplete luôn.

Terminal tích hợp trong VS Code chạy trực tiếp trên Pi, nên `docker compose up -d` gõ ngay tại đó.

## Cách 4 — Viết trên Mac rồi đẩy sang

```bash
# Trên Mac
scp docker-compose.yml huy@smarthome.local:~/smarthome/

# Hoặc sync cả thư mục
rsync -az ~/projects/smarthome/ huy@smarthome.local:~/smarthome/
```

Hợp nếu anh muốn giữ config trong git repo trên Mac làm nguồn chính.

---

## Kiểm tra sau khi tạo

```bash
cd ~/smarthome

# Xem Compose parse ra như thế nào SAU KHI thay biến từ .env.
# Đây là bước quan trọng nhất — nó bắt lỗi cú pháp và cho anh
# thấy TZ có được điền đúng không.
docker compose config
```

Nếu `${TZ}` hiện ra thành rỗng, kiểm tra `.env` có tồn tại và đúng thư mục:

```bash
cat .env      # phải thấy TZ=Asia/Ho_Chi_Minh
```

Rồi chạy:

```bash
docker compose up -d
docker compose ps
```

## Hai lỗi YAML anh sẽ gặp nếu tự gõ

1. **Tuyệt đối không dùng tab.** YAML chỉ nhận space. Lỗi báo ra là `found character '\t' that cannot start any token` — nếu thấy dòng này thì đúng nguyên nhân. Trong nano thì phím Tab chèn tab thật, nên nhớ gõ space.

2. **Thụt lề phải đều 2 space mỗi cấp.** `services:` ở cột 0, tên service thụt 2, thuộc tính thụt 4, phần tử list thụt 6. Lệch một space là Compose hiểu sai cấu trúc.

Chạy `docker compose config` mỗi lần sửa file là cách rẻ nhất để bắt hai lỗi này trước khi chúng làm container không lên.

---

Log khu trú vấn đề rất gọn: dòng `Restored 0 base messages` chứng tỏ **thư mục `data/` đã ghi được**, nên phần chown data/log đã đúng. Chỉ còn đúng một thứ sai: file `passwd`.

Mosquitto báo cùng một message `Unable to open pwfile` cho **cả hai** trường hợp — file không tồn tại, và file không đọc được. Ta phân biệt trước.

## Chẩn đoán — 3 lệnh

```bash
cd ~/smarthome

# 1. File có tồn tại trên host không, quyền gì, kích thước bao nhiêu
ls -la mosquitto/config/

# 2. Container nhìn thấy gì qua mount (container đang restart nên phải
#    dùng docker run riêng, exec sẽ không vào được)
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 ls -la /mosquitto/config/

# 3. Có volume "lạc" nào được tạo ra không
docker volume ls
```

Ba kết quả có thể:

- **Không thấy file `passwd`** → nó chưa bao giờ được tạo. Đây là trường hợp tôi nghi nhất, xem mục dưới.
- **`passwd` là một thư mục (`drwx...`)** → Docker tự tạo thư mục thay vì thấy file. Xoá đi rồi tạo lại.
- **File có, size 0** → `mosquitto_passwd` chạy nhưng không nhập được mật khẩu.

## Nguyên nhân có khả năng nhất

Lệnh tôi đưa trước dùng **đường dẫn tương đối**:

```bash
docker run --rm -it -v ./mosquitto/config:/mosquitto/config ...
```

`docker run -v` với đường dẫn tương đối chỉ được hỗ trợ ở Docker khá mới. Trên bản cũ hơn, `./mosquitto/config` không được hiểu là bind mount — nên file `passwd` đã được ghi vào một chỗ khác (hoặc lệnh im lặng không làm gì), và host không có gì cả. Đây là lỗi trong hướng dẫn của tôi, xin lỗi anh.

## Sửa — tạo lại bằng đường dẫn tuyệt đối

Dùng `-b` (batch mode) để không phải nhập tay, chạy chắc chắn hơn:

```bash
cd ~/smarthome

# Xoá thứ cũ nếu nó là thư mục hoặc file rỗng
sudo rm -rf mosquitto/config/passwd

# User đầu tiên: -c tạo file mới
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 \
  mosquitto_passwd -c -b /mosquitto/config/passwd hass 'MatKhauCuaAnh'

# Các board: KHÔNG có -c
for u in infoclock printmon-a1 irblaster claudemonitor; do
  docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
    eclipse-mosquitto:2 \
    mosquitto_passwd -b /mosquitto/config/passwd "$u" "MatKhau_$u"
done
```

Mật khẩu xuất hiện trong shell history khi dùng `-b`. Gõ **một dấu space ở đầu lệnh** để bash không lưu, hoặc dọn sau bằng `history -d`. Sau khi xong, đổi mật khẩu thật bằng chế độ nhập tay nếu anh muốn.

## Đặt quyền và xác nhận

```bash
sudo chown 1883:1883 mosquitto/config/passwd
sudo chmod 640 mosquitto/config/passwd

# Kiểm tra: phải có 5 dòng, mỗi dòng dạng user:$7$...
wc -l mosquitto/config/passwd
cut -d: -f1 mosquitto/config/passwd
```

Dùng `640` thay vì `600`: file thuộc uid 1883 nên `600` vẫn đọc được, nhưng `640` cho anh đọc qua group nếu cần kiểm tra mà không phải `sudo`.

**Xác nhận container thật sự đọc được** — bước này quan trọng, đừng bỏ:

```bash
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 cat /mosquitto/config/passwd
```

Nếu lệnh này in ra nội dung file thì mount và quyền đã đúng. Nếu báo `Permission denied` hoặc `No such file` thì vẫn còn vấn đề ở mount.

## Khởi động lại

```bash
docker compose up -d --force-recreate mosquitto
docker compose logs -f mosquitto
```

Log đúng phải **không còn** dòng `pwfile`, và kết thúc bằng:

```
Opening ipv4 listen socket on port 1883.
Opening websockets listen socket on port 9001.
mosquitto version 2.0.x running
```

Test lần cuối:

```bash
docker exec -it mosquitto mosquitto_sub -u hass -P 'MatKhauCuaAnh' \
  -t '$SYS/broker/version' -v -C 1
```

Ra được version string là xong.

## Nếu vẫn lỗi sau tất cả những bước trên

Cho tôi output của ba lệnh chẩn đoán ở đầu, đặc biệt là lệnh thứ hai (`docker run ... ls -la`). Nếu container **không** thấy file mà host **có** thấy, thì vấn đề nằm ở mount chứ không phải quyền — lúc đó tôi sẽ chuyển hướng sang kiểm tra đường dẫn trong `docker-compose.yml` và `docker compose config`.

---

Đúng như mong đợi — bảng rỗng nghĩa là không có named volume nào, tất cả mount của mình đều là bind mount. Loại trừ được một nguyên nhân.

Nhưng cái quyết định là hai lệnh còn lại. Chạy gộp một lần:

```bash
cd ~/smarthome

echo "=== 1. HOST ==="
ls -la mosquitto/config/

echo "=== 2. CONTAINER ==="
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 ls -la /mosquitto/config/

echo "=== 3. MOUNT COMPOSE HIỂU ==="
docker compose config | grep -A3 -i mosquitto/config

echo "=== 4. USER HIỆN TẠI ==="
id
pwd
```

Mục 4 để tôi xác nhận đường dẫn home của anh thật sự là `/home/huy` — nếu username khác thì đường dẫn tuyệt đối tôi đưa ở trên sai, và đó chính là nguyên nhân.

Gửi output cho tôi, tôi chỉ ra ngay chỗ sai.

---

Phía host **đã đúng hoàn toàn**: file `passwd` tồn tại, 981 bytes, `1883:1883`, và mount trong `docker compose config` trỏ đúng đường dẫn. Username là `huy`, `/home/huy` khớp. Không có gì sai ở đây.

Vấn đề là **mục 2 không có output** — vì lệnh bị thiếu một dấu space: `ls -la/mosquitto/config/`. `ls` hiểu `-la/mosquitto/config/` là chuỗi option nên báo lỗi và không liệt kê gì.

Nghi ngờ chính của tôi lúc này: **log anh gửi là log cũ**, sinh ra từ trước lúc file `passwd` được tạo (00:07). Container đang trong vòng backoff nên `docker compose logs` in lại cả lịch sử.

## Chạy lại 3 lệnh này

```bash
cd ~/smarthome

echo "=== A. CONTAINER ĐỌC ĐƯỢC FILE? ==="
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 sh -c 'id; ls -la /mosquitto/config/; head -c 60 /mosquitto/config/passwd'

echo "=== B. DANH SÁCH USER ==="
cut -d: -f1 mosquitto/config/passwd

echo "=== C. RECREATE VÀ XEM LOG MỚI ==="
docker compose up -d --force-recreate mosquitto
sleep 8
docker compose logs -t --since 1m mosquitto
```

Ba mục này trả lời ba câu khác nhau:

- **A** cho biết container có thực sự đọc được file không, và `id` xác nhận nó chạy bằng uid nào. Nếu `head` in ra được `hass:$7$...` thì mount và quyền đã hết vấn đề.
- **B** kiểm tra file có đúng 5 dòng user không. 981 bytes hơi nhiều cho 5 entry, có thể vòng `for` đã chạy hai lần và có user trùng.
- **C** là phần quyết định — `--since 1m` chỉ lấy log mới, không lẫn với lỗi cũ, và `-t` in timestamp đọc được thay vì epoch.

## Nếu mục C vẫn báo `Unable to open pwfile`

Thử nới quyền một chút để loại trừ hẳn yếu tố permission:

```bash
sudo chmod 644 mosquitto/config/passwd
docker compose restart mosquitto
docker compose logs -t --since 30s mosquitto
```

File hash bcrypt nên không phải secret dạng plaintext, `644` trong LAN nhà là chấp nhận được để test. Nếu đổi sang `644` mà chạy được, ta biết chắc nguyên nhân là quyền chứ không phải mount, rồi tinh chỉnh lại sau.

Gửi tôi output mục A và C.

---

Tin tốt: **log mới không còn dòng `Unable to open pwfile`**, và chỉ có **một** block `running mosquitto as user` trong cả phút vừa rồi. Nếu còn crash-loop thì block đó sẽ lặp lại 5–6 lần. Nghĩa là container đã lên và ở đó — đúng như tôi nghi ở lượt trước: log anh gửi ban đầu là log tích luỹ từ *trước* lúc anh chown lúc 00:09, không phải lỗi hiện tại.

Mục A thất bại vì lý do khác hẳn: **dấu `\` xuống dòng không sống sót qua paste**. Bash nhận `docker run --rm -v ...` thiếu image, rồi coi `eclipse-mosquitto:2` là lệnh riêng. Từ giờ tôi viết một dòng liền cho anh.

## Xác nhận

```bash
docker compose ps
```

```bash
docker compose logs --tail 12 mosquitto
```

Cần thấy `Up` (không có `Restarting`), và trong log có `Opening ipv4 listen socket on port 1883`.

## Test thật

```bash
docker exec -it mosquitto mosquitto_sub -u hass -P 'MatKhauCuaAnh' -t '$SYS/broker/version' -v -C 1
```

Ra được version string là Mosquitto đã đọc `passwd` thành công và auth hoạt động.

## Kiểm tra danh sách user

`cut` báo permission denied là bình thường — file thuộc uid 1883, mode 600, còn anh là uid 1000. Dùng sudo:

```bash
sudo cut -d: -f1 mosquitto/config/passwd
```

Phải ra đúng 5 dòng: `hass`, `infoclock`, `printmon-a1`, `irblaster`, `claudemonitor`. Nếu ra nhiều hơn hoặc có tên trùng thì vòng `for` đã chạy hai lần — không sao, nhưng nên tạo lại cho gọn.

Nếu cần chạy lại mục A, đây là bản một dòng:

```bash
docker run --rm -v /home/huy/smarthome/mosquitto/config:/mosquitto/config eclipse-mosquitto:2 sh -c 'id; wc -l /mosquitto/config/passwd'
```

Gửi tôi output `docker compose ps` và lệnh test là biết chắc.

---

Xong — Mosquitto đã chạy ổn định. Bằng chứng: `Up About a minute` không kèm `Restarting`, và chỉ có **một** block startup trong log thay vì lặp lại mỗi vài giây như trước.

Chuyện anh không thấy dòng `Opening ipv4 listen socket on port 1883` là **bình thường**, không phải lỗi. Trong `mosquitto.conf` mình chỉ bật `log_type error/warning/notice`, còn các dòng startup đó thuộc loại `information` nên bị lọc bỏ. Nếu muốn thấy chúng, thêm `log_type information` vào config.

## Test dứt điểm

Dùng retained message để test được bằng hai lệnh chạy tuần tự, không cần mở hai terminal:

```bash
docker exec mosquitto mosquitto_pub -u hass -P 'MatKhauCuaAnh' -t 'test/ping' -m 'ok' -r
```

```bash
docker exec mosquitto mosquitto_sub -u hass -P 'MatKhauCuaAnh' -t 'test/ping' -v -C 1
```

Ra `test/ping ok` là auth và pwfile đều hoạt động.

Rồi kiểm tra auth **thực sự bị bắt buộc** (lệnh này phải **thất bại**):

```bash
docker exec mosquitto mosquitto_sub -u hass -P 'mat_khau_sai' -t 'test/ping' -C 1
```

Phải báo `Connection Refused: not authorised`. Nếu nó lại kết nối được thì `allow_anonymous false` chưa có tác dụng.

Dọn retained message sau khi test:

```bash
docker exec mosquitto mosquitto_pub -u hass -P 'MatKhauCuaAnh' -t 'test/ping' -r -n
```

Đừng dùng topic `$SYS/...` để test như tôi gợi ý ở lượt trước — đó là chỗ tôi sai. Trong MQTT, wildcard `#` **không** khớp với topic bắt đầu bằng `$`, nên ACL `topic readwrite #` của user `hass` không cho phép đọc `$SYS`. Lệnh đó sẽ không trả về gì và làm anh tưởng broker lỗi.

## Bước tiếp theo

Nền tảng đã đủ. Vào HA ở `http://smarthome.local:8123`:

1. Settings → Devices & Services → **Add Integration** → MQTT
2. Broker `127.0.0.1`, port `1883`, user `hass`, password vừa dùng ở trên
3. Sau đó tạo `homeassistant/configuration.yaml` với phần `recorder` mà tôi đưa trước, rồi `docker compose restart homeassistant`

Anh muốn tôi viết `common/base-c3.yaml` + node `sensor-t2` hoàn chỉnh để flash con C3 đầu tiên chưa?

---

