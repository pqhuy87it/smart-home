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

---

## 1. Trên Pi — lấy IP

```bash
hostname -I | awk '{print $1}'
```

Giả sử ra `192.168.1.50`. Mình dùng IP này làm ví dụ ở dưới, bạn thay bằng IP thật.

## 2. Trên MacBook — kiểm tra cùng subnet

Mở Terminal:

```bash
ipconfig getifaddr en0    # WiFi
```

Nếu rỗng, thử `en1` (một số máy) hoặc dùng `ifconfig | grep 'inet 192'`. IP của Mac phải cùng 3 octet đầu với Pi (`192.168.1.x`). Vì bạn đã chuyển router các tầng sang AP mode nên toàn nhà chung một subnet, chỉ cần Mac không nằm trên **guest network** — guest network thường bật client isolation và sẽ chặn hoàn toàn.

Thông đường chưa:

```bash
ping -c 3 192.168.1.50
nc -zv 192.168.1.50 8123
```

Kỳ vọng: ping có reply, và `nc` báo `succeeded`. Cả hai đều pass thì mở browser là chắc chắn vào được.

## 3. Mở trong browser

```
http://192.168.1.50:8123
```

Ba điểm dễ sai:

- Phải là **`http://`**, không phải `https://`. Nếu gõ thiếu scheme, Safari/Chrome sẽ tự thêm `https` và báo lỗi kết nối. Gõ đủ `http://`.
- Phải có **`:8123`**. Thiếu port thì browser gọi port 80, không có gì lắng nghe ở đó.
- Đừng gõ vào ô search — Chrome sẽ đem đi Google. Gõ vào address bar rồi Enter luôn.

## 4. Dùng tên thay vì IP

macOS có Bonjour sẵn nên mDNS chạy tốt, thử luôn:

```
http://exlinct.local:8123
```

Được thì dùng cái này, khỏi lo IP đổi. Nếu `.local` không phân giải (`ping exlinct.local` fail), thêm alias ngắn vào hosts file của Mac:

```bash
sudo nano /etc/hosts
```

Thêm dòng cuối:

```
192.168.1.50    ha exlinct.local
```

Lưu (`Ctrl+O`, `Enter`, `Ctrl+X`), rồi flush cache:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

Giờ gõ `http://ha:8123` là vào. Cách này cần Pi có IP tĩnh — bạn đã làm DHCP reservation cho K1C rồi, làm thêm cho Pi trong trang admin của router chính.

## 5. Biến thành app trên Dock

Đỡ phải mở tab mỗi lần:

- **Safari 17+**: mở HA → menu **File → Add to Dock** → đặt tên "Home Assistant".
- **Chrome**: icon ba chấm → **Cast, Save and Share → Install page as app**.

Cả hai mở ra cửa sổ riêng không có address bar, dùng như native app. Đây là cách phổ biến nhất trên máy tính vì HA vốn là PWA.

Ngoài ra có **Home Assistant Companion** trên Mac App Store (miễn phí) — thêm được notification vào Notification Center của macOS và tạo sensor theo dõi máy Mac. Cài rồi điền URL `http://192.168.1.50:8123` là dùng được, nhưng không bắt buộc; PWA đủ cho việc bấm nút và xem dashboard.

## 6. SSH từ MacBook sang Pi

Để không phải ngồi trước Pi mỗi lần chạy `docker compose`:

```bash
ssh huy@192.168.1.50
```

Đặt key cho khỏi nhập password:

```bash
ls ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -C "macbook"
ssh-copy-id huy@192.168.1.50
ssh huy@192.168.1.50 'docker compose -f ~/smarthome/docker-compose.yml ps'
```

Lệnh cuối chạy được mà không hỏi password là xong.

Thêm shortcut vào `~/.ssh/config` trên Mac:

```
Host pi
    HostName 192.168.1.50
    User huy
```

Sau đó chỉ cần `ssh pi`.

## Nếu không vào được

| Triệu chứng | Nguyên nhân |
|---|---|
| `ping` fail | Mac ở guest network, hoặc khác subnet. Đổi sang WiFi chính. |
| `ping` OK, `nc` fail | Container HA không chạy → SSH vào Pi kiểm tra `docker compose ps`. |
| Cả hai OK, browser vẫn lỗi | Gõ thiếu `http://` hoặc `:8123`. |
| Vào được rồi mất kết nối | IP Pi đổi do DHCP → đặt reservation. |

Truy cập ngoài mạng nhà (4G, đi công tác) là chuyện khác — cần Tailscale hoặc Nabu Casa, đừng port-forward 8123 ra internet. Khi nào cần thì nói mình hướng dẫn riêng.

---

## 1. Xác định interface đang dùng

Trước tiên phải biết Pi kết nối bằng Ethernet hay WiFi, vì **eth0 và wlan0 có MAC khác nhau** — đặt reservation cho interface không dùng thì vô tác dụng.

```bash
ip route get 1.1.1.1
```

Output sẽ có `dev eth0` hoặc `dev wlan0`. Đó là interface đang mang traffic.

## 2. Lấy MAC

```bash
ip -br link show
```

Output dạng:

```
lo               UNKNOWN  00:00:00:00:00:00
eth0             UP       2c:cf:67:aa:bb:cc
wlan0            DOWN     2c:cf:67:aa:bb:cd
```

Cột thứ 3 là MAC. Lấy MAC của interface có trạng thái `UP` và khớp với `dev` ở bước 1.

Cách gọn hơn nếu chỉ cần một dòng:

```bash
cat /sys/class/net/eth0/address     # MAC Ethernet
cat /sys/class/net/wlan0/address    # MAC WiFi
```

MAC của Pi 5 thường bắt đầu bằng `2c:cf:67`, `d8:3a:dd` hoặc `e4:5f:01` — đây là OUI của Raspberry Pi. Dấu hiệu này giúp bạn nhận ra Pi trong danh sách client của router.

Chú ý: MAC WiFi và MAC Ethernet của Pi 5 chỉ khác nhau ở byte cuối (thường lệch 1), rất dễ nhìn lẫn. Copy nguyên chuỗi thay vì gõ tay.

## 3. Kiểm tra MAC randomization (chỉ khi dùng WiFi)

Raspberry Pi OS Bookworm dùng NetworkManager, và nếu connection profile bị set random MAC thì reservation sẽ hỏng sau mỗi lần reconnect:

```bash
nmcli -f 802-11-wireless.cloned-mac-address connection show "$(nmcli -t -f NAME connection show --active | head -1)"
```

Kỳ vọng: `--` hoặc `permanent`. Nếu ra `random` hoặc `stable`, khoá lại:

```bash
sudo nmcli connection modify "TÊN_WIFI" 802-11-wireless.cloned-mac-address permanent
sudo nmcli connection up "TÊN_WIFI"
```

Dùng Ethernet thì bỏ qua bước này.

## 4. Đặt reservation trên router

Vào trang admin của **router chính** (router đang làm DHCP server — không phải các router tầng đã chuyển sang AP mode, chúng không cấp IP nữa).

Tìm mục tên đại loại: **DHCP Reservation** / **Address Reservation** / **Static DHCP** / **Bind IP to MAC**, thường nằm trong LAN Settings hoặc DHCP Server.

Điền:
- MAC: chuỗi lấy ở bước 2
- IP: **chọn IP nằm ngoài DHCP pool** nếu router cho phép. Ví dụ pool là `192.168.1.100–192.168.1.200` thì đặt Pi ở `192.168.1.50`. Cách này tránh xung đột về sau. Nếu router bắt buộc IP phải trong pool thì cứ giữ nguyên IP hiện tại của Pi.
- Tên: `pi5-ha` hoặc `exlinct`

Lưu và reboot router nếu nó yêu cầu.

## 5. Verify

Trên Pi, xin lease mới rồi kiểm tra:

```bash
sudo dhclient -r eth0 2>/dev/null; sudo systemctl restart NetworkManager
sleep 10
hostname -I
```

IP phải đúng như đã reserve. Nếu vẫn ra IP cũ, reboot Pi cho chắc:

```bash
sudo reboot
```

Sau khi Pi lên lại, xác nhận stack vẫn ổn:

```bash
cd ~/smarthome && docker compose ps
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8123
```

Cả 3 container `Up` và `302` là xong.

## 6. Cập nhật lại phía MacBook

Nếu IP có thay đổi, sửa `/etc/hosts` trên Mac cho khớp:

```bash
sudo nano /etc/hosts
# sửa dòng: 192.168.1.50    ha exlinct.local
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
ping -c 2 ha
```

Nếu bạn có dùng ESPHome trỏ node về IP của Pi, hoặc MQTT broker address điền bằng IP ở đâu đó, kiểm tra lại luôn — nhưng ESPHome native API dùng mDNS nên thường tự tìm được, không cần sửa.

---

"Connection refused" là tin tốt — nó khác hẳn "Connection timed out":

- **Refused** = có host trả lời ở IP đó, nhưng **không có gì lắng nghe ở port 22**. Đường mạng thông.
- **Timed out** = không ai trả lời (sai IP, khác subnet, bị firewall drop).

Nên vấn đề gần như chắc chắn là: **SSH server chưa được bật trên Pi.** Raspberry Pi OS mặc định **tắt SSH** trừ khi bạn tick "Enable SSH" trong Raspberry Pi Imager lúc ghi thẻ.

Còn một khả năng nữa: `192.168.1.130` không phải Pi mà là thiết bị khác trong nhà — thiết bị đó cũng trả về refused y như vậy. Nên xác nhận IP trước.

## Trên Pi (dùng màn hình/bàn phím trực tiếp)

**1. Xác nhận IP đúng là 192.168.1.130**

```bash
hostname -I
```

Nếu ra IP khác thì dùng IP đó, các bước dưới vẫn làm bình thường.

**2. Kiểm tra sshd có chạy không**

```bash
systemctl status ssh
sudo ss -tlnp | grep -w 22
```

`ss` không in gì → đúng như dự đoán, không có gì listen ở port 22.

**3. Bật SSH**

```bash
sudo systemctl enable --now ssh
```

Nếu lệnh trên báo `Unit ssh.service not found`, cài đặt bằng `raspi-config`:

```bash
sudo raspi-config
```

Chọn **3 Interface Options** → **I1 SSH** → **Yes** → **Finish**.

Trên Raspberry Pi OS mới (Debian 13), SSH có thể dùng socket activation thay vì service. Nếu `systemctl status ssh` báo inactive mà không lỗi, thử thêm:

```bash
sudo systemctl enable --now ssh.socket
```

**4. Verify đã listen**

```bash
sudo ss -tlnp | grep -w 22
```

Phải thấy dòng có `0.0.0.0:22` và `users:(("sshd",...))`.

## Trên MacBook

```bash
nc -zv 192.168.1.130 22
```

Ra `succeeded` rồi mới ssh:

```bash
ssh huy@192.168.1.130
```

Lần đầu nó hỏi xác nhận fingerprint — gõ `yes`.

## Nếu vẫn refused sau khi bật

Kiểm tra firewall trên Pi:

```bash
sudo ufw status
```

`inactive` là không liên quan. Nếu `active` thì mở port:

```bash
sudo ufw allow 22/tcp
```

Kiểm tra sshd có bị bind chỉ vào localhost:

```bash
grep -E '^ListenAddress|^Port' /etc/ssh/sshd_config
```

Nếu có `ListenAddress 127.0.0.1` thì comment dòng đó lại rồi `sudo systemctl restart ssh`.

## Sau khi vào được

Đặt key ngay cho đỡ nhập password, và vì Pi đang chạy HA thì không nên để password authentication lâu dài:

```bash
# trên Mac
ssh-copy-id huy@192.168.1.130
ssh huy@192.168.1.130 'cd ~/smarthome && docker compose ps'
```

Lệnh cuối chạy được mà không hỏi password là xong. Và nhớ đặt DHCP reservation cho IP `192.168.1.130` như bước trước, không thì mai IP đổi lại phải dò lại từ đầu.

---

Máy Mac chưa có SSH key pair nào — `ssh-copy-id` không tìm thấy gì để copy. Ở bước trước mình có đặt lệnh `ls ~/.ssh/id_ed25519.pub || ssh-keygen ...`, chỗ này dễ bỏ sót. Tạo key là xong.

## 1. Kiểm tra thư mục ~/.ssh trên Mac

```bash
ls -la ~/.ssh/
```

Có thể ra `No such file or directory` (bình thường nếu chưa dùng SSH bao giờ), hoặc chỉ có `known_hosts` và `config`. Điều cần thấy mà đang thiếu là cặp `id_ed25519` + `id_ed25519.pub`.

## 2. Tạo key pair

```bash
ssh-keygen -t ed25519 -C "macbook-pro"
```

Nó hỏi 3 câu:

- **Enter file in which to save the key** → Enter luôn để dùng đường dẫn mặc định `~/.ssh/id_ed25519`. Đừng đổi tên, vì `ssh-copy-id` và `ssh` tự tìm đúng tên mặc định này.
- **Enter passphrase** → Nên đặt một passphrase. macOS lưu được vào Keychain nên bạn chỉ nhập một lần duy nhất, không phải nhập mỗi lần ssh. Muốn nhanh thì Enter để trống cũng được — key không passphrase mà máy bị mất thì người khác vào Pi được ngay.
- **Enter same passphrase again** → nhập lại.

Xong sẽ thấy fingerprint và một hình ASCII art. Kiểm tra:

```bash
ls -l ~/.ssh/id_ed25519*
```

Phải có 2 file: `id_ed25519` (private, quyền `-rw-------`) và `id_ed25519.pub` (public).

## 3. Nếu có đặt passphrase — lưu vào Keychain

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Nhập passphrase một lần. Rồi thêm vào `~/.ssh/config` để tự load sau khi reboot:

```bash
cat >> ~/.ssh/config <<'EOF'

Host pi
    HostName 192.168.1.130
    User huy
    IdentityFile ~/.ssh/id_ed25519
    UseKeychain yes
    AddKeysToAgent yes
EOF
chmod 600 ~/.ssh/config
```

Bỏ qua bước này nếu để passphrase trống.

## 4. Copy key sang Pi

```bash
ssh-copy-id huy@192.168.1.130
```

Lần này nó sẽ hỏi password của user `huy` trên Pi (password login của Pi, không phải passphrase của key). Nhập xong sẽ báo:

```
Number of key(s) added: 1
```

## 5. Verify

```bash
ssh -o PasswordAuthentication=no huy@192.168.1.130 'hostname; cd ~/smarthome && docker compose ps'
```

Cờ `-o PasswordAuthentication=no` là điểm quan trọng: nó buộc SSH chỉ dùng key. Nếu lệnh chạy được và in ra `exlinct` cùng danh sách container thì key đã hoạt động thật, chứ không phải nó âm thầm fallback về password.

Nếu bạn đã tạo `~/.ssh/config` ở bước 3 thì từ giờ chỉ cần:

```bash
ssh pi
```

## Nếu vẫn bị hỏi password sau khi copy

Thường do quyền file trên Pi bị sai — sshd từ chối `authorized_keys` nếu thư mục ghi được bởi group/other:

```bash
ssh huy@192.168.1.130
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
exit
```

Rồi thử lại lệnh verify ở bước 5.
