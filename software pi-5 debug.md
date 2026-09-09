Xin lỗi bạn, mình đã đào quá sâu vào một thứ không phải vấn đề. Nói ngắn lại:

**Cài đặt của bạn đang hoạt động bình thường, không có gì cần sửa.**

Container tạo hôm qua 17:44, hiện `running`, chưa crash lần nào. Hai dòng WARNING về sqlite chỉ là do bạn restart HA vài lần trong lúc cấu hình hôm qua — restart trong lúc setup thì recorder không kịp đóng DB, HA tự dọn ở lần khởi động sau. Đây là chuyện hoàn toàn bình thường với một cài đặt mới và không cần xử lý gì.

## Check lại cho yên tâm — 3 lệnh

```bash
cd ~/smarthome

# 1. Container có chạy không
docker compose ps

# 2. HA có trả lời không
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8123

# 3. Có lỗi thật sự nào không
docker compose logs homeassistant | grep -E 'ERROR|CRITICAL'
```

Kỳ vọng: `Up`, `200`, và lệnh 3 **không in ra gì**. Được cả 3 là xong, mở `http://<ip-pi>:8123` dùng bình thường.

Bỏ qua hết các dòng `s6-rc:` — đó là log khởi động nội bộ của container, luôn xuất hiện, không phải lỗi.

## Một thay đổi nhỏ nên làm

Thêm 1 dòng vào `docker-compose.yml` để lần sau restart HA không sinh warning sqlite nữa:

```yaml
services:
  homeassistant:
    stop_grace_period: 90s
```

Rồi `docker compose up -d`. Việc này để Docker chờ HA đóng database tử tế thay vì kill sau 10 giây mặc định. Không bắt buộc, nhưng làm một lần rồi quên luôn.

Bạn chạy 3 lệnh trên xem sao. Nếu ổn cả thì coi như cài xong, và mình dừng phần điều tra log ở đây.

---

Cả 3 đều pass. Cài đặt xong, dùng được.

- **`docker compose ps`** — cả 3 container `Up`. ESPHome còn báo `(healthy)`.
- **`302`** — mình nói kỳ vọng `200` là không chính xác, xin lỗi. HA luôn redirect `/` sang trang login (hoặc `/onboarding` nếu chưa tạo account), nên `302` chính là kết quả đúng. Web server đang phục vụ bình thường.
- **`grep` không in gì** — không có ERROR hay CRITICAL nào. Đây là tín hiệu quan trọng nhất.

Stack `homeassistant + esphome + mosquitto` đã đủ để bắt đầu. Mở `http://<ip-pi>:8123` và làm onboarding nếu chưa.

Một chi tiết: HA `Up 24 seconds` trong khi hai container kia `Up About an hour` — HA vừa restart. Nếu bạn vừa chạy `docker compose restart` hoặc bấm Restart trên UI thì đúng như vậy, bỏ qua. Còn cột `CREATED` vẫn là `22 hours ago` cho cả 3, nghĩa là container chưa được recreate — nên nếu bạn đã thêm `stop_grace_period` vào file thì nó **chưa có hiệu lực**. Kiểm tra bằng:

```bash
docker inspect homeassistant --format '{{.HostConfig.StopTimeout}}'
```

Ra `90` là đã apply. Ra `0` hoặc rỗng thì chạy `docker compose up -d` (compose sẽ recreate, mất khoảng 30 giây HA offline). Không gấp — chỉ ảnh hưởng chất lượng shutdown, không ảnh hưởng vận hành.

---

## A. Lấy IP của Pi

```bash
hostname -I | awk '{print $1}'
```

Ghi lại IP đó (ví dụ `192.168.1.50`). Nếu bạn đã đặt DHCP reservation cho Pi thì IP này sẽ không đổi — nên dùng luôn `http://192.168.1.50:8123`. Truy cập từ máy Mac mini trong cùng subnet.

Nếu Pi đã bật mDNS (mặc định có `avahi-daemon`), `http://exlinct.local:8123` cũng chạy.

## B. Onboarding trong browser

Mở `http://<ip-pi>:8123`. Vì `curl` trả về `302`, HA sẽ tự redirect sang `/onboarding.html`.

**B1. Tạo owner account.** Form đầu tiên: Name / Username / Password. Đây là account `owner` — quyền cao nhất, không xoá được, không có cơ chế "quên mật khẩu" qua email. Dùng password manager lưu lại ngay. Username nên viết thường, không dấu, không khoảng trắng.

**B2. Đặt tên nhà và vị trí.** HA cần toạ độ để tính `sun.sun` (bình minh/hoàng hôn) — thứ hầu hết automation ánh sáng đều dựa vào. Kéo pin trên map về đúng nhà bạn, hoặc bấm nút detect. Sau đó kiểm tra 4 field:

- Time zone: `Asia/Ho_Chi_Minh` (phải khớp với `TZ` trong compose, nếu lệch thì automation theo giờ sẽ sai)
- Elevation: khoảng `16` m cho Hà Nội
- Unit system: **Metric**
- Currency: `VND`

**B3. Analytics.** Trang tiếp theo hỏi có gửi usage data không. Tuỳ bạn, bỏ trống hết cũng được, không ảnh hưởng chức năng.

**B4. Discovered devices.** HA quét mạng và liệt kê thiết bị tìm thấy. Vì đang chạy `network_mode: host`, việc quét này hoạt động đúng (đây chính là lý do cần host network). **Bấm Finish và bỏ qua hết ở bước này** — cấu hình từng device sau sẽ gọn hơn là click vội trong onboarding.

Xong là vào dashboard chính.

## C. Verify onboarding đã ghi xuống disk

Quan trọng, vì nếu bind mount sai thì account vừa tạo sẽ mất khi recreate container:

```bash
ls -la ~/smarthome/homeassistant/.storage/ | grep -E 'auth|onboarding|core.config'
```

Phải thấy `auth`, `auth_provider.homeassistant`, `onboarding`, `core.config_entries`. Có nghĩa dữ liệu nằm trên disk của Pi, không phải trong container layer.

Kiểm tra thật sự bằng cách restart rồi login lại:

```bash
cd ~/smarthome && docker compose restart homeassistant
sleep 30
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8123
```

Vẫn `302` và login được bằng account vừa tạo → persistence OK.

## D. ESPHome — dùng native API, không cần MQTT

Đây là điểm mình muốn nói rõ vì nó đổi hẳn việc bạn phải làm: **ESPHome nói chuyện với HA qua native API, không qua MQTT.** Với các node ESP32-C3/ESP8266 của bạn, mosquitto là **không bắt buộc**. Native API nhanh hơn, tự động discovery, không cần broker ở giữa.

Mở dashboard ESPHome: `http://<ip-pi>:6052`

Trong file YAML của mỗi node chỉ cần:

```yaml
api:
  encryption:
    key: !secret api_encryption_key   # tạo bằng nút "Generate" trên dashboard
ota:
  - platform: esphome
    password: !secret ota_password
```

Sau khi flash, HA sẽ tự phát hiện node qua mDNS và hiện notification "New devices discovered" → **Settings → Devices & Services** → Configure → dán encryption key.

Chỉ cần MQTT khi: node phải hoạt động lúc HA offline, hoặc bạn muốn tích hợp thiết bị không phải ESPHome (Tasmota, Zigbee2MQTT sau này). Nếu chưa cần thì để mosquitto đó, chưa phải cấu hình gì.

## E. Nếu muốn bật MQTT

Mosquitto 2.x mặc định chạy `/mosquitto-no-auth.conf` — cho anonymous. Không nên để vậy khi có node WiFi kết nối vào. Kiểm tra compose đã bind mount config chưa:

```bash
grep -A8 'mosquitto:' ~/smarthome/docker-compose.yml
```

Cần có:

```yaml
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
```

Nếu chưa mount `config`, password file sẽ mất mỗi lần recreate container. Thêm vào rồi:

```bash
cd ~/smarthome
mkdir -p mosquitto/config mosquitto/data

cat > mosquitto/config/mosquitto.conf <<'EOF'
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
persistence true
persistence_location /mosquitto/data/
EOF

# tạo user (đổi mật khẩu bên dưới)
docker compose up -d mosquitto
docker compose exec mosquitto mosquitto_passwd -c -b /mosquitto/config/passwd hass 'MẬT_KHẨU_CỦA_BẠN'
sudo chown 1883:1883 mosquitto/config/passwd
sudo chmod 600 mosquitto/config/passwd
docker compose restart mosquitto
```

Verify broker nhận auth đúng:

```bash
# phải THÀNH CÔNG
docker compose exec mosquitto mosquitto_pub -h localhost -u hass -P 'MẬT_KHẨU_CỦA_BẠN' -t test/x -m ok

# phải BỊ TỪ CHỐI (Connection refused: not authorised)
docker compose exec mosquitto mosquitto_pub -h localhost -t test/x -m ok
```

Rồi thêm integration trong HA: **Settings → Devices & Services → Add Integration → MQTT**, điền Broker `127.0.0.1`, Port `1883`, Username `hass`, Password như trên. Dùng `127.0.0.1` được vì cả hai container đều ở host network.

## F. Bật backup ngay

Việc cuối và đừng bỏ qua: **Settings → System → Backups → Automatic backups**. Đặt lịch hàng ngày, giữ 3–7 bản. Sau đó copy định kỳ ra ngoài Pi:

```bash
rsync -av ~/smarthome/homeassistant/backups/ /đường/dẫn/nas/hoặc/mac/
```

Backup nằm cùng thẻ nhớ với HA thì thẻ chết là mất cả hai — đó là lý do phải copy ra ngoài.
