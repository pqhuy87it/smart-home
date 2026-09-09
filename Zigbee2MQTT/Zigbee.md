Zigbee2MQTT là một trong những lựa chọn hay nhất nếu anh muốn làm smart home **local-first**, không phụ thuộc cloud. Tôi giải thích từ dưới lên nhé.

## 1. Zigbee là cái gì trước đã

Zigbee là một giao thức wireless chạy trên chuẩn radio **IEEE 802.15.4**, band 2.4GHz (cùng band với WiFi/BLE). Đặc điểm:

- **Low power**: một cảm biến cửa dùng pin CR2032 chạy 1–2 năm. Vì nó không giữ kết nối liên tục như WiFi, chỉ thức dậy khi có event.
- **Mesh network**: đây là điểm mạnh nhất. Trong một mạng Zigbee có 3 vai trò:
  - **Coordinator**: chỉ có duy nhất 1, là "gốc" của mạng, quản lý network key và cho phép device join.
  - **Router**: thiết bị cắm điện (bóng đèn, ổ cắm, công tắc). Chúng vừa hoạt động vừa **relay** message cho device khác → càng nhiều router, mạng càng phủ xa và càng ổn định.
  - **End device**: thiết bị chạy pin (sensor). Ngủ phần lớn thời gian, không relay cho ai.
- **ZCL (Zigbee Cluster Library)**: tầng application. Mỗi chức năng là một "cluster" (`genOnOff`, `genLevelCtrl`, `msTemperatureMeasurement`...). Đây là lý do Zigbee có thể chuẩn hoá được — nhưng thực tế các hãng Trung Quốc (Tuya) hay tự nghĩ ra cluster riêng, và đó chính là chỗ Zigbee2MQTT toả sáng.

## 2. Vấn đề mà Zigbee2MQTT giải quyết

Bình thường anh mua đồ Zigbee thì kèm theo cái **gateway/hub của hãng**: Aqara hub, Tuya Zigbee gateway, Philips Hue Bridge, IKEA Dirigera... Hậu quả:

- Mỗi hãng một hub → nhà anh có 4 cái hub cắm điện.
- Data đi qua cloud của hãng → mất net là mất điều khiển, latency cao, và hãng có thể khai tử server bất cứ lúc nào.
- Vendor lock-in: đèn Hue không nói chuyện được với sensor Aqara mà không qua trung gian.

Zigbee2MQTT (gọi tắt Z2M) là một **application Node.js** làm đúng một việc: thay thế toàn bộ các hub đó bằng **một cái USB dongle duy nhất**, rồi dịch mọi thứ ra MQTT.

## 3. Kiến trúcĐiểm quan trọng: **Z2M không phải là hệ thống automation**. Nó chỉ là một *bridge*. Nó không biết "trời tối thì bật đèn" — nó chỉ biết "sensor này vừa báo illuminance = 12 lux, publish lên topic". Logic nằm ở Home Assistant / Node-RED.

## 4. MQTT topic trông như thế nào

Đây là phần làm dev như anh thích Z2M. Base topic mặc định là `zigbee2mqtt`:

```
# Z2M publish state của device (JSON)
zigbee2mqtt/cam_bien_cua_bep
  → {"contact": false, "battery": 87, "linkquality": 132, "voltage": 3000}

# Anh publish để điều khiển
zigbee2mqtt/den_phong_khach/set
  → {"state": "ON", "brightness": 180, "color_temp": 350}

# Đọc lại state hiện tại
zigbee2mqtt/den_phong_khach/get
  → {"state": ""}

# Topic quản trị
zigbee2mqtt/bridge/state          → online / offline
zigbee2mqtt/bridge/devices        → danh sách toàn bộ device
zigbee2mqtt/bridge/request/permit_join
```

Vì nó là MQTT thuần, anh có thể `mosquitto_sub -t 'zigbee2mqtt/#' -v` để xem mọi thứ chạy real-time, hoặc viết một MQTT client trên ESP32 subscribe trực tiếp mà không cần Home Assistant. Đó là điểm mà một hub đóng của hãng không bao giờ cho anh.

Z2M cũng tự publish **MQTT Discovery** messages để Home Assistant tự tạo entity, anh không phải khai báo YAML thủ công cho từng device.

## 5. Phần cứng cần mua

Chỉ cần đúng một thứ: **coordinator dongle**. Hai họ chip chính:

| Chip | Dongle phổ biến | Ghi chú |
|---|---|---|
| TI CC2652P | Sonoff ZBDongle-P | Firmware zStack, cực kỳ ổn định, cộng đồng lớn |
| Silabs EFR32MG21 | Sonoff ZBDongle-E | Firmware EmberZNet (driver `ember`), hỗ trợ Thread |
| Silabs EFR32MG24 | Sonoff Dongle Max, HA Connect ZBT-2 | Thế hệ mới, multiprotocol Zigbee + Thread |

ZBDongle-P (CC2652P) vẫn là lựa chọn được recommend nhiều nhất cho Zigbee2MQTT vì đã được kiểm chứng lâu năm và có user base rất lớn — nghĩa là nhiều guide và dễ troubleshoot hơn. Lưu ý chip: các chip kết thúc bằng "P" có power amplifier lên tới 20dBm so với 5dBm của loại R/RB.

Nếu server của anh nằm ở góc nhà tệ (tủ rack, tầng hầm), có loại **SLZB-06 của SMLIGHT** cắm qua Ethernet/PoE — đặt dongle giữa nhà, server ở đâu cũng được.

## 6. Z2M vs ZHA vs Matter/Thread

- **ZHA**: integration built-in của Home Assistant, không cần cài thêm gì. Đơn giản hơn nhưng Zigbee2MQTT hỗ trợ nhiều model thiết bị hơn, expose nhiều feature hơn, và cộng đồng phát triển active hơn — trade-off là setup ban đầu mất công hơn một chút. Với đồ Tuya/Aqara lạ, Z2M gần như luôn thắng vì có hệ thống **external converter** (anh tự viết một file JS định nghĩa device là xong).
- **Matter over Thread**: Thread cũng chạy trên 802.15.4 giống Zigbee (cùng tầng radio) nhưng là IP-based, mỗi device có IPv6 address riêng. Về lâu dài Thread sẽ thay Zigbee, nhưng hiện tại ecosystem Zigbee vẫn rẻ hơn và phong phú hơn nhiều lần. Nhiều dongle MG24 làm được cả hai.

## 7. Những cái sẽ làm anh mất buổi tối

1. **Dùng USB extension cable.** Không phải joke. Cổng USB 3.0 phát nhiễu điện từ đúng dải 2.4GHz — cắm dongle trực tiếp vào cổng USB 3.0 (hoặc kể cả cổng bên cạnh) sẽ làm pairing fail và device rớt mạng ngẫu nhiên.
2. **Chọn Zigbee channel tránh WiFi.** Zigbee channel 11/15/20/25 là các channel ít chồng lấn WiFi nhất. Chọn ngay từ đầu, vì đổi channel sau có thể phải pair lại device.
3. **Build mesh đúng thứ tự**: thêm các router (thiết bị cắm điện) trước, đặt gần coordinator, rồi mới thêm sensor chạy pin. Nếu pair sensor trước khi có router, nó sẽ bám trực tiếp vào coordinator và không tự đổi route sau đó.
4. **Dùng path ổn định** cho serial port: `/dev/serial/by-id/...` thay vì `/dev/ttyUSB0` — cái sau đổi số khi reboot.
5. **Migrate từ ZHA sang Z2M phải pair lại toàn bộ device**, không có in-place conversion. Nếu có hơn 10 device thì dành hẳn một buổi sáng.
6. Z2M gần đây đã chuyển từ npm sang pnpm, deprecate driver `ezsp` cũ thay bằng `ember`, và thay việc sửa YAML thủ công bằng onboarding wizard ở lần chạy đầu. Nên guide cũ trên YouTube có thể đã lệch.

## 8. Điểm nối với đồ anh đang có

Có hai chỗ thú vị:

- **ESP32-C6** của anh có radio 802.15.4 native, nên nó có thể làm **Zigbee end device hoặc router thật** (qua `esp-zigbee-sdk`). Anh có thể tự làm sensor mực nước Zigbee thay vì WiFi — tiết kiệm pin hơn nhiều, và pair thẳng vào Z2M bằng một external converter tự viết. Đây là hướng rất hợp với cái project solar water level monitoring trước đây.
- **SmartFace (ESP32-S3)** thì không có 802.15.4, nhưng nó có WiFi → cho nó làm **MQTT client** subscribe `zigbee2mqtt/#` là hiển thị được nhiệt độ/độ ẩm từ sensor Zigbee lên màn TFT, không cần gọi API weather nào cả.

Anh muốn tôi đi sâu vào phần nào — setup cụ thể bằng Docker trên Pi, cách viết external converter cho device Tuya lạ, hay hướng dùng ESP32-C6 làm Zigbee device?
