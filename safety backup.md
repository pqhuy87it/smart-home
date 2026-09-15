Rủi ro thật sự khi di chuyển không nằm ở phần cứng mà ở **database của Home Assistant**. Recorder ghi SQLite liên tục; rút điện đột ngột là cách nhanh nhất để có một file `home-assistant_v2.db` hỏng và mất toàn bộ lịch sử. Nên quy trình dưới đây tập trung vào việc dừng sạch trước khi cắt điện.

## Bước 1 — Ghi lại trạng thái hiện tại

```bash
ssh huy@exlinct.local
hostname -I                    # ghi lại IP đang dùng
ip -br link                    # ghi lại MAC của eth0 và wlan0
cd ~/smarthome && docker compose ps
```

Chụp màn hình hoặc lưu lại. MAC của `eth0` và `wlan0` là **khác nhau**, nên nếu tầng 2 bạn phải chuyển từ dây sang WiFi thì router sẽ cấp IP khác — cái này ảnh hưởng tới HomeKit, nói ở bước 7.

Nếu có dongle USB (Zigbee, Z-Wave), ghi lại luôn:

```bash
ls -l /dev/serial/by-id/
```

## Bước 2 — Tạo backup trong HA

Settings → System → Backups → **Create backup**. Đợi nó chạy xong hẳn rồi mới sang bước tiếp. File nằm ở `~/smarthome/homeassistant/backups/`.

## Bước 3 — Dừng stack đúng cách

```bash
cd ~/smarthome
docker compose stop -t 60
```

`-t 60` quan trọng: mặc định Compose chỉ chờ 10 giây rồi `SIGKILL`. Home Assistant cần lâu hơn thế để đóng recorder DB và ghi state cuối cùng — đây chính là chỗ hay hỏng DB nhất, và nó âm thầm, vài ngày sau mới lộ ra.

Kiểm tra:

```bash
docker compose ps          # tất cả phải ở trạng thái Exited
```

## Bước 4 — Backup ra ngoài Pi

Giờ container đã dừng, file mới nhất quán để nén:

```bash
sudo tar czf /tmp/smarthome-$(date +%F).tar.gz -C /home/huy smarthome
ls -lh /tmp/smarthome-*.tar.gz
```

Kéo về Mac (chạy trên Mac, không phải trên Pi):

```bash
scp huy@exlinct.local:/tmp/smarthome-*.tar.gz ~/Downloads/
```

Bước này tốn vài phút nhưng là thứ duy nhất cứu bạn nếu SSD/thẻ nhớ chết đúng lúc bê máy — thao tác vật lý là lúc ổ dễ hỏng nhất.

## Bước 5 — Shutdown

```bash
sync
sudo shutdown -h now
```

Session SSH sẽ đứt, đó là bình thường.

## Bước 6 — Chờ đúng tín hiệu rồi mới rút điện

Pi 5 sau khi halt sẽ vào standby: LED chuyển **đỏ đứng yên**, không còn nháy xanh. Đợi thêm khoảng 10 giây sau khi mọi nháy dừng hẳn rồi mới rút USB-C. Nếu bạn dùng NVMe HAT, đèn hoạt động trên HAT cũng phải tắt.

Thứ tự tháo: nguồn USB-C → dây mạng → các USB khác → tháo vít.

Khi bê, đừng cầm vào cánh tản nhiệt hay HAT — cầm mép PCB. NVMe cắm trên HAT chỉ giữ bằng một vít, rung mạnh có thể làm cong chân connector.

## Bước 7 — Lắp ở tầng 2

Cắm lại theo thứ tự ngược: USB thiết bị → mạng → nguồn sau cùng.

**Nếu vẫn cắm dây LAN:** Pi tự boot khi có điện, không cần bấm nút. IP giữ nguyên nếu router cấp lại theo MAC cũ — nên đặt DHCP reservation cho `eth0` để chắc chắn.

**Nếu tầng 2 chỉ có WiFi:** trước đây bạn đã chạy `sudo rfkill block wifi` khi Pi dùng dây. Phải mở lại (cắm màn hình + bàn phím, hoặc cắm tạm dây mạng lần cuối ở tầng 1 để cấu hình trước khi bê):

```bash
sudo rfkill unblock wifi
sudo nmcli device wifi connect "TenWifi" password "..."
```

Rồi đặt DHCP reservation cho MAC của `wlan0`.

**Lưu ý HomeKit:** bạn đã set **Advertise IP** trong HomeKit Bridge. Nếu IP hoặc interface đổi, accessories sẽ mất kết nối trong app Home. Vào HomeKit Bridge → Configure → sửa Advertise IP thành IP mới. Đây là thứ hay bị quên nhất sau khi di chuyển máy.

## Bước 8 — Khởi động lại stack

Đây là cái bẫy phổ biến: container có `restart: unless-stopped` mà bạn dừng thủ công ở bước 3 thì **nó sẽ không tự chạy lại sau khi Pi boot**. Phải start tay:

```bash
ssh huy@exlinct.local
cd ~/smarthome
docker compose start
docker compose ps
```

## Bước 9 — Verify

```bash
uptime                                    # xác nhận vừa boot, không có reboot lặp
dmesg | grep -iE "error|i/o|nvme" | tail -20    # ổ có lỗi sau khi bê không
docker compose logs --tail=80 homeassistant
```

Trong log HA, thứ cần tìm là **không có** dòng nào kiểu `database is locked` hay `corrupt`. Nếu thấy, dừng ngay và restore từ backup ở bước 4 thay vì để nó chạy tiếp.

Sau đó kiểm tra theo thứ tự:

1. Mở `http://exlinct.local:8123` — vào được là mDNS ổn.
2. Settings → Devices → ESPHome: các node `t1`, `t2` phải xanh trở lại trong 1–2 phút. Chúng tự reconnect, không cần làm gì.
3. Mở app Home trên iPhone, bật/tắt thử một accessory. Nếu treo → quay lại phần Advertise IP ở bước 7.
4. Nếu có dongle USB: `ls -l /dev/serial/by-id/` xem còn đúng đường dẫn cũ không. Nếu compose file đang trỏ tới `/dev/ttyUSB0` thì cắm sang cổng khác sẽ đổi số — sửa sang dùng đường dẫn `by-id` luôn cho khỏi tái diễn.

## Một điểm về vị trí mới

Tầng 2 cao hơn nên mùa hè nóng hơn đáng kể. Pi 5 throttle ở 80–85°C. Sau khi lắp xong, theo dõi vài ngày:

```bash
vcgencmd measure_temp
vcgencmd get_throttled     # 0x0 là sạch
```

Và bạn sắp có sẵn node `t2` đo nhiệt độ ngay tầng đó — dùng luôn để làm automation cảnh báo khi phòng đặt Pi vượt ngưỡng.
