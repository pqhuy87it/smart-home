Cảnh báo này trên Pi 5 gần như luôn xuất phát từ **USB power budget**, chứ không phải chuột/bàn phím "ăn" quá nhiều điện.

## Nguyên nhân gốc

Pi 5 yêu cầu adapter **USB-C PD 5V/5A (25W)**. Khi boot, firmware đọc PD negotiation:

- Nếu nhận được **5V/5A** → tổng current cho USB peripherals là **1.6A**
- Nếu chỉ **5V/3A** (adapter Pi 4 cũ, sạc điện thoại, PD trigger board) → firmware khóa USB xuống **600mA** tổng cho cả 4 port

Chuột + bàn phím có dây thường chỉ tốn 100–250mA, nhưng bàn phím có LED/RGB hoặc hub tích hợp có thể vượt 600mA rất nhanh. Cộng thêm **voltage drop** trên cáp USB-C mỏng/dài, điện áp vào tụt dưới ~4.8V → PMIC bắn low voltage warning.

Với hub Home Assistant chạy 24/7, cái này không chỉ là warning: under-voltage kéo dài gây **SD card / NVMe corruption** và database HA bị lỗi.

## Bước 1 — Xác định chính xác

```bash
# Kiểm tra throttling flags
vcgencmd get_throttled

# Đọc điện áp thực tế từ PMIC
vcgencmd pmic_read_adc | grep -E "EXT5V_V|VDD_CORE_V|3V3_SYS_V"

# Xem firmware có nhận đủ 5A không
dmesg | grep -iE "usb|power|voltage" | grep -iE "current|supply|voltage"
```

Giải mã `get_throttled`:

| Bit | Giá trị | Ý nghĩa |
|---|---|---|
| 0 | `0x1` | Under-voltage **đang xảy ra** |
| 1 | `0x2` | ARM frequency capped |
| 2 | `0x4` | Đang bị throttled |
| 16 | `0x10000` | Under-voltage **đã từng** xảy ra |
| 18 | `0x40000` | Throttling đã từng xảy ra |

`throttled=0x0` là sạch. `0x50000` nghĩa là đã có under-voltage + throttling trong quá khứ.

`EXT5V_V` là điện áp vào thực tế — nếu đọc ra dưới 4.9V khi có tải thì vấn đề nằm ở PSU hoặc cáp.

Nếu `dmesg` xuất hiện dòng kiểu *"power supply is not capable of providing 5A"* → firmware đã tự giới hạn 600mA.

## Bước 2 — Sửa theo thứ tự ưu tiên

**Cách đúng nhất: đổi PSU.** Dùng adapter Raspberry Pi 27W USB-C chính hãng (5.1V/5A). Điện áp 5.1V thay vì 5.0V là cố ý — nó bù sẵn voltage drop trên cáp. Cáp đi liền adapter cũng đúng gauge.

**Cáp cũng quan trọng như adapter.** Cáp USB-C dài trên 1.5m hoặc loại 28AWG có thể sụt 0.3–0.5V khi tải 3A. Ở Hà Nội hàng chính hãng dễ mua ở Hshop/Nshop/IoTmaker; nếu dùng adapter PD của laptop thì phải kèm cáp e-marked 5A.

**Chuyển peripherals sang powered USB hub.** Cách này giải quyết triệt để: chuột, bàn phím, USB dongle cắm vào hub có nguồn riêng, Pi chỉ còn cấp cho chính nó. Với HA hub thì đây là setup nên có sẵn.

**Chỉ khi PSU thực sự đủ khỏe** (ví dụ PD adapter 45W tốt nhưng negotiation không báo 5A), mới override giới hạn:

```bash
sudo nano /boot/firmware/config.txt
```

Thêm vào cuối, trong section `[all]`:

```ini
[all]
usb_max_current_enable=1
```

```bash
sudo reboot
```

Cần hiểu rõ: option này **không làm PSU mạnh hơn**, nó chỉ tắt cơ chế bảo vệ. Nếu PSU thật sự yếu, kết quả là brownout đột ngột giữa lúc HA đang ghi database → hỏng file. Đừng dùng như giải pháp lâu dài.

## Bước 3 — Verify sau khi sửa

Script này chạy trong lúc cắm/rút thiết bị để xem điện áp phản ứng thế nào:

```bash
#!/usr/bin/env bash
# pi5-power-monitor.sh — theo dõi under-voltage realtime
set -euo pipefail

printf "%-9s %-8s %-10s %s\n" "TIME" "EXT5V" "THROTTLED" "STATUS"

while true; do
    ts=$(date +%H:%M:%S)
    volt=$(vcgencmd pmic_read_adc EXT5V_V | awk -F= '{printf "%.3f", $2}')
    thr=$(vcgencmd get_throttled | cut -d= -f2)
    dec=$((thr))

    status="OK"
    (( dec & 0x1     )) && status="UNDER-VOLTAGE NOW"
    (( dec & 0x4     )) && status="$status | THROTTLED"
    [[ $status == "OK" ]] && (( dec & 0x10000 )) && status="OK (past under-voltage)"

    printf "%-9s %-8s %-10s %s\n" "$ts" "$volt" "$thr" "$status"
    sleep 2
done
```

```bash
chmod +x pi5-power-monitor.sh
./pi5-power-monitor.sh
```

Cắm chuột, bàn phím, rồi thử copy file lớn hoặc chạy `stress-ng --cpu 4 --timeout 60s` để tạo tải. `EXT5V_V` nên giữ ở khoảng 5.0–5.15V và không tụt dưới 4.9V.

Sau khi xác nhận ổn, xóa flag lịch sử và theo dõi lại từ đầu:

```bash
sudo reboot
# sau khi boot, cắm đủ thiết bị, chờ vài phút rồi:
vcgencmd get_throttled   # kỳ vọng: throttled=0x0
```

Với hub HA, thêm luôn sensor này vào Home Assistant để biết khi nào nguồn có vấn đề, thay vì phát hiện qua database bị hỏng:

```yaml
# configuration.yaml
command_line:
  - sensor:
      name: "Pi5 Throttled State"
      command: "vcgencmd get_throttled | cut -d= -f2"
      scan_interval: 60
  - sensor:
      name: "Pi5 Input Voltage"
      command: "vcgencmd pmic_read_adc EXT5V_V | awk -F= '{printf \"%.3f\", $2}'"
      unit_of_measurement: "V"
      scan_interval: 60
```

Nếu HA chạy trong Docker/container, cần mount `/opt/vc` và cấp quyền `/dev/vcio`, hoặc đơn giản hơn là dùng SSH command từ HA sang host.

Bạn đang dùng adapter nào và có gắn thêm NVMe HAT không? NVMe qua PCIe HAT tốn thêm 1–2A lúc peak, nếu có thì gần như chắc chắn phải dùng PSU 5A.

---

Log này khá rõ ràng, và tin tốt là nó chỉ ra một nguyên nhân cụ thể hơn giả thuyết ban đầu của tôi.

## Đọc kết quả

**`throttled=0x50000`** = bit 16 + bit 18 → under-voltage **đã từng** xảy ra, và ARM clock **đã thực sự bị throttle**. Các bit 0–3 đều sạch, nên ngay lúc chạy lệnh thì nguồn đang bình thường. Nghĩa là: sự cố có thật, đã ảnh hưởng performance, nhưng mang tính transient.

**`EXT5V_V = 4.946V`** — đây là con số quan trọng nhất. Đo lúc idle (thấy rõ qua `VDD_CORE_V = 0.72V`, tức CPU đang ở clock thấp nhất). Ngưỡng báo under-voltage của Pi 5 nằm quanh 4.8V, nên bạn chỉ còn khoảng **0.15V margin** ở trạng thái nhàn rỗi nhất. Bất kỳ transient nào — CPU burst, SD/NVMe write, USB enumeration — đều đủ để chọc xuống dưới ngưỡng.

`3V3_SYS_V = 3.318V` và `VDD_CORE_V = 0.72V` đều bình thường, không cần lo.

**Pattern trong dmesg mới là điểm đáng chú ý.** Mỗi lần under-voltage kéo dài 2–10 giây rồi tự normalise, không phải một cú sụt rồi chết. Và chúng phân bố theo cụm: dày đặc trong khoảng `41528–41681` (khoảng 2.5 phút, 8 lần), rồi thưa dần ở `41796` và `42145`.

Điều này phủ nhận giả thuyết "vượt 600mA USB budget". Nếu chạm giới hạn current, firmware sẽ **cắt port** và bạn sẽ thấy USB disconnect trong dmesg, chứ không phải cả rail 5V bị sag rồi hồi phục. Cái đang xảy ra là **PSU không giữ được điện áp khi tải tăng**, tức vấn đề nằm ở adapter hoặc điện trở cáp — chuột/bàn phím chỉ là phần tải cuối cùng đẩy nó qua ngưỡng.

## Ba lệnh để chốt chẩn đoán

**1. Firmware đã negotiate được bao nhiêu current?**

```bash
for f in /proc/device-tree/chosen/power/*; do
    printf "%-28s = " "$(basename "$f")"
    od -An -tu4 --endian=big "$f" 2>/dev/null | tr -d ' ' || cat "$f"
    echo
done
```

`max_current = 5000` là adapter báo đủ 5A. Nếu ra `3000` thì USB budget đang bị khóa ở 600mA, và bạn có thêm một vấn đề độc lập cần xử lý.

**2. Các cụm under-voltage đó trùng với việc gì?**

```bash
dmesg -T | grep -i "undervoltage" | tail -20
```

Có wall-clock time rồi, đối chiếu ngược:

```bash
# thay timestamp bằng con số thật từ lệnh trên
journalctl --since "2026-09-08 10:20:00" --until "2026-09-08 10:30:00" --no-pager | less
```

Nghi vấn hàng đầu với HA hub: **recorder purge** (mặc định chạy 04:12 hàng ngày, ghi rất nặng vào SD/NVMe), automatic backup, hoặc `apt` update. Nếu cụm dày đặc kia trùng một trong số đó thì đã có câu trả lời về "cái gì tạo transient".

**3. Sụt sâu đến đâu?** Script cũ poll 2 giây nên gần như chắc chắn bỏ lọt các dip 2 giây. Bản này sample 5Hz và ghi lại giá trị min:

```bash
#!/usr/bin/env bash
# pi5-rail-sampler.sh — sample EXT5V ở 5Hz, log min + sự kiện under-voltage
set -euo pipefail

LOG="${1:-/var/log/pi5-rail.log}"
MIN=99.0

trap 'echo; echo "Min EXT5V observed: ${MIN}V"; exit 0' INT TERM

echo "Sampling at 5Hz -> $LOG  (Ctrl-C để dừng)"

while true; do
    v=$(vcgencmd pmic_read_adc EXT5V_V | awk -F= '{printf "%.3f", $2}')
    thr=$(vcgencmd get_throttled | cut -d= -f2)

    if (( $(echo "$v < $MIN" | bc -l) )); then
        MIN=$v
        printf "\rnew min: %sV   " "$v"
    fi

    if (( $((thr)) & 0x1 )); then
        printf "\n[%s] UNDER-VOLTAGE at %sV (throttled=%s)\n" \
            "$(date '+%F %T')" "$v" "$thr" | tee -a "$LOG"
    fi

    sleep 0.2
done
```

```bash
sudo apt install -y bc stress-ng
chmod +x pi5-rail-sampler.sh
sudo ./pi5-rail-sampler.sh &

# tạo tải để reproduce
stress-ng --cpu 4 --vm 2 --vm-bytes 256M --timeout 90s
```

Nếu min tụt xuống 4.7–4.8V khi stress, đó là bằng chứng đóng lại vụ này.

## Hành động

Với 4.946V ở idle, **đổi adapter là việc phải làm**, không phải tùy chọn. Adapter Raspberry Pi 27W chính hãng ra 5.1V có chủ đích để bù drop trên cáp — chênh 0.1V đó chính là phần margin bạn đang thiếu. Ở Hà Nội thì Hshop, Nshop, IoTmaker đều có hàng chính hãng.

Nếu adapter hiện tại là loại 5A tốt nhưng cáp là hàng đi kèm sạc điện thoại, thử đổi cáp trước — cáp 28AWG dài 1m có thể sụt 0.2–0.3V ở 2–3A, đúng bằng lượng bạn đang thiếu. Cách test nhanh: chạy sampler ở trên, đổi cáp, so sánh min.

Chuột và bàn phím thì chuyển sang **powered USB hub**. Với HA hub 24/7 thì đây là setup nên có sẵn, kể cả sau khi đã đổi PSU.

Đừng thêm `usb_max_current_enable=1` trong tình huống này — rail đang sag thật, override chỉ tắt cảnh báo và làm tăng nguy cơ brownout giữa lúc recorder đang commit.

## Kiểm tra thiệt hại

Bit 0x40000 nghĩa là clock đã bị hạ thật, và các dip 2–10s đó rơi vào lúc HA đang ghi dữ liệu. Nên kiểm tra:

```bash
# SQLite DB của HA (đổi path nếu bạn dùng container/Supervised)
sqlite3 ~/.homeassistant/home-assistant_v2.db "PRAGMA integrity_check;"

# filesystem
sudo dmesg | grep -iE "ext4|mmcblk|nvme" | grep -iE "error|corrupt|I/O"
```

`integrity_check` trả về `ok` là ổn. Nếu ra lỗi thì restore từ backup thay vì cố sửa — SQLite bị hỏng do power event thường sửa không sạch.

Sau khi đổi PSU, reboot để reset flags rồi chạy lại `vcgencmd get_throttled` sau 24h. Kỳ vọng `0x0`.

Cho tôi biết `max_current` ra bao nhiêu và min voltage lúc stress là mấy — hai số đó sẽ cho biết bạn cần đổi cả adapter hay chỉ cáp.

---

`usbpd_power_data_objects = 0000 0000 ...` toàn số 0 là câu trả lời cuối cùng. Đây không phải chuyện margin nữa.

## PDO rỗng nghĩa là gì

PDO (Power Data Object) là danh sách các mức điện áp/current mà adapter tự khai báo trong quá trình PD negotiation. Adapter 27W chính hãng sẽ khai ít nhất `5.1V/5A`. Trường này rỗng hoàn toàn nghĩa là **PD negotiation chưa bao giờ diễn ra** — Pi không nhận được PDO nào cả.

Khi đó firmware rơi về fallback an toàn: giả định 5V/3A, chính là `max_current = 3000` bạn thấy. Và hệ quả kéo theo là USB peripheral budget bị khóa ở **600mA** (`usb_max_current_enable = 0`).

Vậy bạn đang có hai vấn đề chồng nhau, và cả hai đều từ một gốc:

1. **Rail sag** — 4.946V ở idle, các dip 2–10s trong dmesg
2. **USB budget 600mA** — do PD không negotiate được

Hai chỉ số khác trong log giúp loại trừ nghi vấn: `usb_over_current_detected = 0` xác nhận chuột/bàn phím **không** hề chạm giới hạn 600mA — đúng như dmesg đã gợi ý, đây là sag của cả rail 5V chứ không phải overcurrent ở port. Và `power_reset = 0` là tin tốt: chưa có brownout reset nào, nên rủi ro filesystem tới giờ vẫn chỉ là rủi ro.

## Nguyên nhân: adapter hay cáp?

PD negotiation cần CC line hoạt động trên cả hai đầu. PDO rỗng nên thuộc một trong ba trường hợp:

**Adapter output là USB-A**, dùng cáp A-to-C. Trường hợp này PD về mặt vật lý không thể xảy ra — không có CC pin để đàm phán. Đây là nguyên nhân phổ biến nhất và cũng khớp nhất với triệu chứng của bạn: cố định 5.0V không bù drop, sụt tải kém.

**Cáp C-to-C nhưng là loại charge-only** — thiếu CC conductor hoặc không có e-marker. Loại này rất nhiều trên thị trường, đặc biệt cáp tặng kèm.

**Adapter là PD nhưng PD controller không bắt tay được** với Pi. Ít gặp hơn, thường ở adapter đa cổng giá rẻ.

Cách phân biệt trong 30 giây:

```bash
# Nhìn đầu adapter: output là USB-A hay USB-C?
```

Nếu là USB-A → xong, phải đổi cả adapter. Nếu là USB-C, cắm chính adapter + cáp đó vào MacBook hoặc điện thoại và xem có báo sạc nhanh / PD không. Không nhận PD ở thiết bị khác → adapter hoặc cáp lỗi, thử đổi cáp trước.

## Việc cần làm

Mua **Raspberry Pi 27W USB-C Power Supply** chính hãng. Với tình huống PDO rỗng, tôi khuyên đúng con này chứ không phải "một adapter PD 5A nào đó", vì hai lý do: nó ra **5.1V** (bù sẵn phần 0.15V margin bạn đang thiếu), và **cáp liền adapter** nên loại bỏ hẳn biến số cáp — vốn đang là nghi phạm chính. Hshop, Nshop, IoTmaker ở Hà Nội đều có, khoảng 350–450k.

Nếu muốn dùng PD adapter sẵn có (laptop 45W+ chẳng hạn), phải kèm cáp **C-to-C e-marked 5A**. Nhưng lưu ý: nhiều adapter laptop bỏ qua PDO 5V/5A và chỉ khai `5V/3A` rồi nhảy lên 9V/15V/20V. Trường hợp đó `max_current` vẫn ra 3000 dù adapter 65W. Chỉ dùng nếu bạn verify được bằng lệnh ở dưới.

Chuột/bàn phím vẫn nên chuyển sang **powered USB hub** — không phải vì 600mA (bạn chưa chạm), mà để tách hoàn toàn tải peripheral khỏi rail của Pi trên một hub 24/7.

Tuyệt đối **không** set `usb_max_current_enable=1` lúc này. Rail đang sag thật ở 3A, mở lên 1.6A USB sẽ đẩy nhanh tới brownout — và `power_reset` hiện là 0, đừng làm nó thành 1 giữa lúc recorder đang commit.

## Verify sau khi đổi

```bash
sudo reboot
```

Sau khi boot, cắm đủ chuột + bàn phím:

```bash
# 1. PD đã negotiate chưa
for f in max_current usb_max_current_enable usbpd_power_data_objects; do
    printf "%-26s = " "$f"
    od -An -tx4 --endian=big "/proc/device-tree/chosen/power/$f" 2>/dev/null
done

# 2. Rail voltage
vcgencmd pmic_read_adc EXT5V_V

# 3. Flags sạch chưa
vcgencmd get_throttled
```

Kỳ vọng:

| Trường | Giá trị mong đợi |
|---|---|
| `max_current` | `5000` |
| `usb_max_current_enable` | `1` (firmware tự set khi thấy 5A) |
| `usbpd_power_data_objects` | **khác 0** — có PDO thật |
| `EXT5V_V` | 5.05–5.20V ở idle |
| `get_throttled` | `0x0` |

Nếu `usbpd_power_data_objects` vẫn rỗng sau khi đổi adapter chính hãng, vấn đề nằm ở board Pi (CC pin) — lúc đó mới là chuyện RMA.

Rồi chạy lại sampler + `stress-ng --cpu 4 --vm 2 --vm-bytes 256M --timeout 90s`, min voltage nên giữ trên 4.95V. Sau 24h uptime kiểm tra `get_throttled` lần nữa — vẫn `0x0` là đóng case.

Trong lúc chờ adapter mới, chạy `PRAGMA integrity_check;` trên DB của HA như đã nói ở trên, và hạn chế tải nặng (đừng chạy backup/purge thủ công).

---

Cần đính chính một chỗ trong bước trước: lệnh đọc device-tree tôi đưa cho bạn có lỗi. Thấy rõ ở dòng `name = 18863532531912602624` — đó là một string bị `od -tu4` diễn giải thành số nguyên. Nghĩa là `usbpd_power_data_objects` cũng có thể đã bị đọc sai hoặc bị cắt (output của bạn hiện `0000` rồi xuống dòng `000`, trông giống bị truncate hơn là rỗng thật).

Con số **đáng tin duy nhất** trong log đó là `max_current = 3000`, vì đó thực sự là một u32 và 3000 là giá trị hợp lý. Ta cứ giữ lại kết luận này và đọc lại PDO cho đúng.

## Đọc lại PDO đúng cách

```bash
sudo xxd /proc/device-tree/chosen/power/usbpd_power_data_objects
```

Và script decode ra dạng người đọc được:

```python
#!/usr/bin/env python3
"""pdo-decode.py — decode USB-PD Power Data Objects từ device-tree của Pi 5."""
import struct
import sys

PATH = "/proc/device-tree/chosen/power/usbpd_power_data_objects"

KIND = {0: "Fixed", 1: "Battery", 2: "Variable", 3: "APDO/PPS"}

def decode_fixed(pdo: int) -> str:
    voltage = ((pdo >> 10) & 0x3FF) * 0.05   # 50mV units
    current = (pdo & 0x3FF) * 0.01           # 10mA units
    return f"{voltage:.2f}V @ {current:.2f}A  ({voltage * current:.1f}W)"

def decode_pps(pdo: int) -> str:
    vmax = ((pdo >> 17) & 0xFF) * 0.1
    vmin = ((pdo >> 8) & 0xFF) * 0.1
    imax = (pdo & 0x7F) * 0.05
    return f"PPS {vmin:.1f}-{vmax:.1f}V @ {imax:.2f}A"

def main() -> int:
    try:
        raw = open(PATH, "rb").read()
    except OSError as e:
        print(f"cannot read {PATH}: {e}", file=sys.stderr)
        return 1

    if len(raw) % 4:
        print(f"warning: {len(raw)} bytes, không chia hết cho 4", file=sys.stderr)

    pdos = struct.unpack(f">{len(raw) // 4}I", raw[: len(raw) // 4 * 4])
    live = [p for p in pdos if p]

    if not live:
        print("KHÔNG có PDO nào → PD negotiation chưa diễn ra")
        return 0

    print(f"{len(live)} PDO(s):\n")
    for i, pdo in enumerate(live, 1):
        kind = (pdo >> 30) & 0x3
        label = KIND[kind]
        if kind == 0:
            detail = decode_fixed(pdo)
        elif kind == 3:
            detail = decode_pps(pdo)
        else:
            detail = "(không decode)"
        print(f"  [{i}] 0x{pdo:08X}  {label:8s} {detail}")

    has_5a = any(
        ((p >> 30) & 0x3) == 0
        and 4.5 <= ((p >> 10) & 0x3FF) * 0.05 <= 5.5
        and (p & 0x3FF) * 0.01 >= 4.9
        for p in live
    )
    print()
    print("Có PDO 5V/5A:", "CÓ" if has_5a else "KHÔNG  ← đây là lý do max_current=3000")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```bash
chmod +x pdo-decode.py
sudo ./pdo-decode.py
```

## Vì sao "27W 5V/5A" trên nhãn vẫn có thể ra max_current=3000

Đây là điểm hay bị bỏ qua. Trong USB PD, adapter khai báo năng lực bằng các **fixed PDO**. Hầu hết adapter 27W+ trên thị trường đạt công suất đó ở **điện áp cao**: `5V/3A`, `9V/3A`, `12V/2.25A`, `15V/1.8A`, `20V/1.35A`. Ở mức 5V chúng chỉ khai **3A**.

Pi 5 không nhận điện áp cao — nó chỉ chạy 5V. Nên nếu adapter không có riêng một PDO `5V/5A`, Pi buộc phải lấy `5V/3A` → `max_current = 3000`, đúng như bạn thấy. PSU 27W chính hãng của Raspberry Pi đặc biệt ở chỗ nó khai đúng cái PDO `5.1V/5A` mà đa số hãng khác không làm.

Nói cách khác: **"27W" và "cấp được 5V/5A cho Pi 5" là hai chuyện khác nhau.** Nhãn ghi 5V/5A cũng có thể là tổng công suất chia cho nhiều cổng, hoặc chỉ đúng khi dùng cáp riêng của hãng.

Thêm nữa: theo USB Type-C spec, mọi mức trên 3A đều **bắt buộc cáp e-marked 5A**. Cáp không có e-marker chip thì adapter sẽ tự hạ xuống 3A dù nó có khai 5A.

Con số `EXT5V_V = 4.946V` cũng hé lộ điều này. PSU chính hãng Pi ra **5.1V** có chủ đích. Đo được 4.95V nghĩa là adapter của bạn ra 5.0V (không bù drop), cộng thêm sụt trên cáp.

## Ba thứ cần xác nhận về cục sạc

Chạy script trên rồi cho tôi biết PDO thật. Song song, kiểm tra ba điểm sau trên chính cục sạc:

**Cáp liền hay cáp rời?** PSU 27W chính hãng của Raspberry Pi có **cáp hàn liền, không tháo được**, dài 1.2m, vỏ trắng hoặc đen. Nếu cục của bạn có cổng USB-C rời để cắm cáp thì đó **không phải** PSU chính hãng Pi — mà là adapter hãng khác. "Chính hãng" của Anker/Ugreen/Baseus vẫn là hàng thật, nhưng không có nghĩa nó khai PDO 5V/5A.

**Một cổng hay nhiều cổng?** Adapter nhiều cổng thường ghi công suất **tổng**. Cắm thêm thiết bị vào cổng khác là nó tự chia lại và hạ xuống 5V/3A.

**Nhãn ghi chính xác thế nào?** Nếu ghi kiểu `5V⎓5A / 9V⎓3A / 12V⎓2.25A` thì 5V/5A là thật. Nếu ghi `Output: 27W max` rồi liệt kê `5V/3A, 9V/3A...` thì mức 5A không tồn tại.

Nếu cục sạc là USB-A output và bạn đang dùng cáp A-to-C, thì kết luận có ngay không cần script: PD không thể xảy ra về mặt vật lý, và `max_current = 3000` là kết quả tất yếu.

Cho tôi output của `pdo-decode.py` cùng ba thông tin trên — nếu PDO có `5V/5A` thật mà `max_current` vẫn là 3000 thì vấn đề nằm ở cáp (thiếu e-marker), đổi cáp e-marked 5A là xong, không cần đổi adapter.

---

`xxd` nằm trong package `vim-common`, không có sẵn trên Raspberry Pi OS Lite. Không cần cài — dùng `od` đã có sẵn:

```bash
sudo od -An -tx4 --endian=big /proc/device-tree/chosen/power/usbpd_power_data_objects
```

Lệnh này in ra các PDO dưới dạng hex word 32-bit, đúng thứ ta cần. Kèm theo kích thước file để biết có bị truncate hay không:

```bash
sudo wc -c /proc/device-tree/chosen/power/usbpd_power_data_objects
```

Nếu ra 28 hoặc 32 bytes thì có 7–8 PDO slot. Toàn `00000000` mới là rỗng thật.

Còn script Python thì không cần `xxd`, nó đọc file trực tiếp. Nếu bạn đã lưu `pdo-decode.py` thì chạy luôn:

```bash
sudo python3 pdo-decode.py
```

Nếu muốn có `xxd` cho lần sau:

```bash
sudo apt install -y vim-common
```

Hoặc `hexdump` (package `bsdextrautils`) cũng thường có sẵn hơn:

```bash
sudo hexdump -C /proc/device-tree/chosen/power/usbpd_power_data_objects
```

Cho tôi output của lệnh `od` và `wc -c` là đủ để chốt.

---

Thông tin này đổi hướng chẩn đoán, và cũng loại bỏ khuyến nghị trước đó của tôi: cáp hàn liền + 1 cổng duy nhất + có dòng 5V/5A riêng chính là đặc điểm của PSU 27W chính hãng Raspberry Pi. Nên "đổi PSU" không còn là câu trả lời — bạn đã có đúng loại thiết bị cần có.

Nhưng có một chi tiết đáng để ý: **nhãn PSU chính hãng ghi `5.1V ⎓ 5A`, không phải `5V/5A`**. Con số 5.1V là đặc trưng, và nó khớp với việc bạn đo được 4.946V (một unit ra 5.1V thật, lúc idle, phải đọc quanh 5.10–5.15V). Bạn kiểm tra lại nhãn xem là `5.1V` hay `5.0V` — chi tiết này phân biệt hàng chính hãng với hàng dựng.

Giờ chỉ còn ba khả năng: **firmware chưa nhận PD đúng**, **hỏng phần cứng** (cáp/đầu cắm/PSU), hoặc **hàng dựng**. Xử lý theo thứ tự từ rẻ đến đắt.

## Bước 1 — Cập nhật firmware + EEPROM

Đây là việc tôi nên đưa ra sớm hơn. Firmware Pi 5 các bản đầu có vấn đề với PSU detection, và các bản sau đã sửa. Nếu firmware của bạn cũ, PD negotiation thất bại là hoàn toàn có thể dù phần cứng hoàn hảo.

```bash
vcgencmd version
sudo rpi-eeprom-update
```

Nếu báo có bản mới:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```

Sau khi boot, cắm đủ thiết bị rồi chạy lại:

```bash
sudo python3 pdo-decode.py
vcgencmd pmic_read_adc EXT5V_V
```

Nếu PDO xuất hiện → xong, không cần làm gì thêm. Bước này giải quyết được kha khá case tương tự.

## Bước 2 — Kiểm tra tiếp xúc cơ học

Pattern trong dmesg của bạn có điểm bất thường mà tôi chưa khai thác: các dip **dồn cụm** trong 2.5 phút rồi thưa dần. Nếu do tải (recorder purge, backup) thì sẽ trùng lịch cụ thể. Nhưng kiểu cụm-rồi-tắt cũng rất giống **tiếp xúc chập chờn**.

Cáp hàn liền hay đứt ngầm ở chỗ uốn gần đầu cắm, và đầu USB-C của Pi rất dễ bám bụi/xơ vải. Một sợi VBUS/GND bị đứt bớt gây sụt áp; một sợi CC bị đứt làm PD **thất bại hoàn toàn** — khớp chính xác với PDO rỗng.

```bash
# soi đầu cắm USB-C trên Pi bằng đèn, thổi khí nén nếu có bụi
# rút ra cắm lại cho chắc chân
```

Rồi test bằng cách rung cáp:

```bash
sudo ./pi5-rail-sampler.sh
```

Trong lúc script chạy, nhẹ nhàng uốn cáp ở ba vị trí: sát đầu cắm vào Pi, sát thân adapter, và giữa cáp. Nếu `EXT5V_V` nhảy hoặc xuất hiện dòng UNDER-VOLTAGE khi bạn động vào cáp → hỏng cáp, và vì cáp hàn liền nên phải thay cả bộ.

## Bước 3 — Cross-test để khoanh vùng

Cần một trong hai phép thử để biết lỗi ở PSU hay ở Pi:

**Đổi PSU khác vào Pi.** Mượn/mua một PD adapter tốt kèm cáp C-to-C e-marked 5A. Nếu PDO xuất hiện → PSU cũ hỏng. Nếu vẫn rỗng → cổng USB-C hoặc PD controller trên board Pi có vấn đề.

**Cắm PSU đó vào thiết bị khác** — MacBook, điện thoại Android. Nếu thiết bị kia cũng không báo PD/sạc nhanh → PSU hỏng. Nhớ là PSU chính hãng Pi chỉ khai 5.1V nên laptop sẽ không sạc nhanh, nhưng điện thoại vẫn phải nhận được PD.

## Bước 4 — Nếu firmware mới nhất mà PDO vẫn rỗng

Có một cơ chế chính thức để khai báo thủ công năng lực PSU, đúng hơn `usb_max_current_enable` trong `config.txt`:

```bash
sudo -E rpi-eeprom-config --edit
```

Thêm dòng:

```ini
PSU_MAX_CURRENT=5000
```

Lưu, reboot. Cách này set ở tầng EEPROM nên firmware coi như PSU 5A ngay từ đầu quá trình boot.

Nhưng **chưa dùng bây giờ.** Điều kiện tiên quyết là rail phải ổn định — cụ thể `EXT5V_V` giữ trên 5.05V ở idle và không tụt dưới 4.95V khi stress. Ở mức 4.946V hiện tại, khai 5A chỉ khiến brownout đến nhanh hơn. Chỉ set sau khi đã xác nhận PSU khỏe thật (qua bước 2 và 3) mà PD vẫn không bắt tay được.

## Tóm lại

Chạy bước 1 trước, mất 10 phút và có khả năng khá cao là đủ. Nếu không được thì bước 2 để loại trừ lỗi cơ học. Bước 3 mới là bước tốn kém.

Cho tôi biết `vcgencmd version` ra bản nào, và Pi của bạn có gắn NVMe HAT hay PoE HAT gì không — HAT qua PCIe tốn thêm 1–2A lúc peak, nếu có thì bức tranh tải sẽ khác đi đáng kể.

---

Mặc định trên Raspberry Pi OS là **nano**:

- **Ctrl + O** → Enter (ghi file)
- **Ctrl + X** (thoát)

Nếu màn hình không có thanh gợi ý phím ở dưới thì đang là **vi/vim**: nhấn **Esc**, gõ `:wq`, Enter.

Sau khi thoát, script sẽ tự áp config và in ra thông báo kiểu *"EEPROM updated"* rồi yêu cầu reboot:

```bash
sudo reboot
```

Kiểm tra config đã lưu chưa:

```bash
sudo rpi-eeprom-config | grep -i psu
```

Ra `PSU_MAX_CURRENT=5000` là đã ghi. Nếu muốn hủy giữa lúc đang sửa thì **Ctrl + X** rồi chọn `N` (nano), hoặc `:q!` (vi) — EEPROM sẽ không bị thay đổi.

Nhắc lại một lần cho chắc: `EXT5V_V` của bạn hiện là 4.946V, tức rail đang sag thật. Khai `PSU_MAX_CURRENT=5000` sẽ mở USB budget lên 1.6A trên một nguồn chưa giữ nổi điện áp ở 3A — brownout reset giữa lúc HA commit database là rủi ro thật. Nếu chưa chạy `sudo rpi-eeprom-update` (bước 1) thì làm cái đó trước; nó không có rủi ro gì và có khả năng khá cao là đủ để PD tự bắt tay được.

---

Đây là chỗ dễ nhầm, và nó quan trọng với đúng vấn đề bạn đang gặp: **OS image mới không đồng nghĩa firmware mới.**

Có hai tầng riêng biệt:

**Bootloader EEPROM** — nằm trên chip flash của board Pi, không nằm trên thẻ SD. Flash OS image mới **không** chạm tới nó. Nó vẫn giữ nguyên firmware được nạp lúc board xuất xưởng, có thể từ 2023 hoặc 2024. Và đây chính là tầng thực hiện PD negotiation với PSU.

**Kernel + userland** — cái này thì đúng là mới nhất khi bạn cài OS tối qua.

Package `rpi-eeprom` được cài mới nhất chỉ nghĩa là **file firmware** mới nhất đã có trên thẻ SD. Việc nạp nó vào EEPROM là một hành động riêng, phải chạy tay.

Kiểm tra:

```bash
sudo rpi-eeprom-update
vcgencmd bootloader_version
```

Output sẽ có hai dòng `CURRENT` và `LATEST`. Nếu `CURRENT` cũ hơn `LATEST` thì EEPROM của bạn đang chạy firmware cũ, dù OS mới cài tối qua.

Nếu lệch:

```bash
sudo rpi-eeprom-update -a
sudo reboot
```

Còn một chi tiết nữa: Raspberry Pi OS mặc định dùng release channel `default` (bảo thủ), không phải `latest`. Nếu `rpi-eeprom-update` báo đã up-to-date mà bạn vẫn muốn thử bản mới hơn:

```bash
sudo nano /etc/default/rpi-eeprom-update
# đổi thành: FIRMWARE_RELEASE_STATUS="latest"
sudo rpi-eeprom-update -a
sudo reboot
```

Sau reboot, kiểm tra lại PD trước khi nghĩ đến `PSU_MAX_CURRENT`:

```bash
sudo python3 pdo-decode.py
vcgencmd pmic_read_adc EXT5V_V
```

Cho tôi biết `CURRENT` và `LATEST` ra ngày nào — nếu chúng thật sự giống nhau thì loại được tầng firmware, và ta chuyển sang test cơ học ở bước 2 (rung cáp trong lúc sampler chạy), vì lúc đó nghi vấn đứt ngầm sợi CC là cao nhất.

---

Firmware đã loại trừ dứt điểm: bản 26/05/2026 là mới nhất trên channel `default`, và `capabilities 0x7f` cho thấy bootloader báo đầy đủ năng lực. Không phải lỗi software.

Còn ba nghi phạm: **đứt ngầm cơ học**, **PSU không đạt spec**, hoặc **PD controller trên board Pi**.

## Bằng chứng mạnh nhất hiện tại

Quay lại con số 4.946V. PSU 27W chính hãng Raspberry Pi ra **5.1V**, và ở idle (CPU đang ở 0.72V core, tải rất thấp) thì sụt trên cáp gần như không đáng kể — một unit 5.1V khỏe mạnh phải đọc ra **5.08–5.15V**.

Bạn đọc được 4.946V, thấp hơn 0.15V. Cộng với việc bạn nói nhãn ghi `5V/5A` chứ không phải `5.1V/5A`, khả năng cao nhất là: PSU không phải hàng chính hãng Raspberry Pi (hàng dựng hoặc OEM khác có form factor tương tự), **hoặc** là hàng thật nhưng đã suy giảm.

Điều này cũng giải thích luôn PDO rỗng: PSU chính hãng bắt buộc phải khai PDO `5.1V/5A`. Một unit "5V/5A" không PD thì chỉ là dumb charger có cáp liền — cấp được 5A về mặt điện, nhưng không có CC line hoạt động nên Pi không bao giờ biết.

Xác nhận nhãn:

```
Chính hãng: "5.1V ⎓ 5.0A" + logo Raspberry Pi + mã kiểu SC1152/SC1153
Hàng dựng:  "5V 5A" hoặc "DC 5V/5A 27W"
```

## Test cơ học (làm trước, miễn phí)

```bash
sudo ./pi5-rail-sampler.sh
```

Trong lúc chạy, uốn nhẹ cáp ở ba chỗ: sát đầu cắm vào Pi, sát thân adapter, giữa cáp. Rồi rút ra cắm lại vài lần.

Nếu `EXT5V_V` nhảy hoặc hiện dòng UNDER-VOLTAGE khi bạn động vào cáp → đứt ngầm, thay cả bộ (cáp liền không sửa được). Nếu voltage đứng yên ở ~4.94V bất kể uốn thế nào → cáp lành, vấn đề là bản thân PSU ra thiếu điện áp.

Nhân lúc đó soi đèn vào cổng USB-C trên Pi xem có bụi/xơ vải, thổi khí nén nếu cần.

## Công cụ đáng mua

Với người làm hardware như bạn thì cái này nên có sẵn: **USB-C power meter** (loại inline, ví dụ FNIRSI FNB48/FNB58, ~250–400k ở Hshop/Nshop). Cắm giữa PSU và Pi, nó hiển thị realtime điện áp, current, và **PD protocol đã negotiate**.

Nó trả lời cả ba câu hỏi trong một lần cắm:
- PSU thực sự ra bao nhiêu V ở đầu ra (phân biệt lỗi PSU vs lỗi cáp)
- Pi đang kéo bao nhiêu A khi có chuột/bàn phím
- PD handshake có xảy ra không, và nếu có thì PDO nào

Đây là cách duy nhất để tách bạch "PSU yếu" khỏi "cáp sụt" mà không cần mượn thiết bị khác.

## Cross-test nếu không có power meter

Mượn một PD adapter tử tế (laptop 45W+) kèm cáp C-to-C **e-marked 5A**, cắm vào Pi rồi chạy:

```bash
sudo python3 pdo-decode.py
vcgencmd pmic_read_adc EXT5V_V
```

PDO xuất hiện → PSU cũ là vấn đề. PDO vẫn rỗng với adapter khác → nghi vấn chuyển sang cổng USB-C hoặc PD controller trên board Pi, lúc đó là chuyện bảo hành.

Lưu ý khi đọc kết quả: nhiều PD adapter laptop chỉ khai `5V/3A` rồi nhảy 9V/15V/20V, nên `max_current` có thể vẫn ra 3000. Cái ta quan tâm ở test này là **PDO có xuất hiện hay không**, không phải giá trị 5000.

## Về PSU_MAX_CURRENT

Chưa set. Điều kiện là `EXT5V_V` phải giữ trên 5.05V ở idle và không xuống dưới 4.95V khi stress. Với 4.946V ở idle, PSU đang không đủ khỏe — khai 5A chỉ mở USB budget lên 1.6A trên một rail vốn đã sag, đẩy nhanh tới brownout reset.

Nếu kết luận cuối là PSU không đạt spec, mua đúng bộ 27W chính hãng (Hshop/Nshop/IoTmaker, ~400k) là xong cả hai vấn đề một lượt: PD negotiate được và có 5.1V bù drop.

Bạn vẫn chưa cho tôi biết Pi có gắn HAT nào không (NVMe, PoE, fan HAT). Nếu có NVMe qua PCIe thì peak thêm 1–2A, và với PSU chỉ ra 4.94V thì đó là yếu tố quyết định — cho tôi biết để đánh giá lại toàn bộ power budget.

---

Câu trả lời trung thực: **rủi ro thấp hơn bạn có thể đang lo, nhưng không phải bằng không** — và có một chi tiết quan trọng làm nó thấp đi đáng kể.

## Chi tiết đó là: HA host chạy headless

Toàn bộ vấn đề của bạn được kích hoạt bởi chuột + bàn phím. Khi Pi đã setup xong và chạy headless, không có peripheral nào cắm vào, tải USB về gần 0. Trần 600mA trở thành vô nghĩa, và cái tải cuối cùng đẩy rail qua ngưỡng cũng biến mất.

Home Assistant bản thân nó rất nhẹ — Pi 5 chạy HA idle phần lớn thời gian, CPU ở clock thấp nhất, đúng như `VDD_CORE_V = 0.72V` bạn đo được. Ở trạng thái đó, 4.946V là mỏng nhưng vẫn trên ngưỡng.

Nên nếu bạn chỉ dùng chuột/bàn phím lúc cấu hình rồi rút ra, thì hoàn cảnh sinh ra các dip trong dmesg sẽ không lặp lại thường xuyên.

## Rủi ro thật sự còn lại

Chỉ có **một** failure mode đáng quan tâm, nhưng nó khá khó chịu:

Undervoltage đủ sâu → PMIC brownout → mất điện đột ngột giữa lúc SQLite đang commit → **database corrupt**. HA recorder ghi liên tục (mỗi state change), nên xác suất "đúng lúc đang ghi" không nhỏ.

Hiện `power_reset = 0`, tức chưa từng xảy ra. Đó là tin tốt và cũng là lý do tôi không nói đây là chuyện cấp bách.

Nhưng vẫn có những thời điểm tải nhảy vọt: recorder purge hàng ngày (mặc định 04:12), automatic backup, `apt full-upgrade`, HA core update. Đó là lúc rủi ro tập trung — và đúng là các cụm dip dày đặc trong dmesg của bạn có pattern giống một sự kiện tải nặng.

Hậu quả cụ thể khi nó xảy ra: mất history/statistics, HA không start được, hoặc tệ hơn là SD card corrupt ở tầng filesystem phải cài lại. Nó sẽ xảy ra vào một thời điểm ngẫu nhiên, không báo trước.

## Những thứ **không** cần lo

Throttling (`0x40000`) — với workload HA thì clock bị hạ vài trăm MHz hoàn toàn không ảnh hưởng gì. Đừng để bit này làm bạn lo.

Hỏng phần cứng Pi — undervoltage ở mức này không làm chết board. Nó gây mất ổn định, không gây hỏng vật lý.

Chuột/bàn phím — `usb_over_current_detected = 0`, bạn chưa bao giờ chạm trần.

## Nếu để nguyên, làm 4 việc này

**1. Đừng set `PSU_MAX_CURRENT=5000`.** Quan trọng nhất. Để nguyên fallback 3000 nghĩa là firmware đang tự bảo vệ bạn. Mở lên 5A trên rail đang sag là cách chắc chắn nhất để biến `power_reset` từ 0 thành 1.

**2. Giảm peak current bằng cách cap CPU.** Cách này giảm trực tiếp biên độ transient:

```bash
sudo nano /boot/firmware/config.txt
```

```ini
[all]
arm_freq=2000
arm_boost=0
```

HA không cần 2.4GHz. Cap về 2.0GHz và tắt boost giảm đáng kể current spike lúc tải nhảy, gần như không ảnh hưởng trải nghiệm.

**3. Giảm tải ghi của recorder.** Vừa giảm I/O current spike vừa thu nhỏ cửa sổ rủi ro corrupt:

```yaml
# configuration.yaml
recorder:
  db_url: sqlite:////home/huy/smarthome/ha.db
  purge_keep_days: 10
  commit_interval: 30
  auto_purge: true
  exclude:
    domains:
      - automation
      - updater
    entity_globs:
      - sensor.*_uptime
      - sensor.*_linkquality
```

`commit_interval: 30` gom write lại thay vì ghi liên tục — ít lần ghi hơn thì ít cơ hội brownout đúng lúc đang ghi hơn.

**4. Backup tự động ra khỏi Pi.** Đây là thứ biến "mất dữ liệu" thành "mất 20 phút restore". Bạn có Mac mini M4, dùng làm đích:

```bash
#!/usr/bin/env bash
# ha-backup.sh — snapshot HA config + DB sang Mac mini
set -euo pipefail

SRC="/home/huy/smarthome"
DEST="huy@macmini.local:/Users/huy/backups/ha"
STAMP=$(date +%Y%m%d-%H%M)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

# snapshot SQLite an toàn (không copy file đang mở)
sqlite3 "$SRC/ha.db" ".backup '$TMP/ha.db'"

tar czf "$TMP/ha-$STAMP.tar.gz" \
    -C "$SRC" configuration.yaml automations.yaml .storage \
    -C "$TMP" ha.db

rsync -az --remove-source-files "$TMP/ha-$STAMP.tar.gz" "$DEST/"
echo "backup ok: ha-$STAMP.tar.gz"
```

```bash
chmod +x ha-backup.sh
# chạy 03:00 hàng ngày, trước purge 04:12
(crontab -l 2>/dev/null; echo "0 3 * * * /home/huy/smarthome/ha-backup.sh >> /var/log/ha-backup.log 2>&1") | crontab -
```

`.backup` của sqlite3 tạo snapshot nhất quán ngay cả khi HA đang ghi — quan trọng hơn `cp` nhiều.

Kèm theo, thêm sensor giám sát vào HA để biết khi nào tình hình xấu đi thay vì phát hiện qua DB hỏng:

```yaml
command_line:
  - sensor:
      name: "Pi5 Throttled"
      command: "vcgencmd get_throttled | cut -d= -f2"
      scan_interval: 300
  - sensor:
      name: "Pi5 Input Voltage"
      command: "vcgencmd pmic_read_adc EXT5V_V | awk -F= '{printf \"%.3f\", $2}'"
      unit_of_measurement: "V"
      scan_interval: 300
```

Đặt automation báo khi `Pi5 Input Voltage` xuống dưới 4.85V. Nếu con số này bắt đầu tụt dần theo tuần/tháng thì PSU đang suy giảm và bạn có cảnh báo sớm.

## Quan điểm của tôi

Để nguyên là quyết định hợp lý nếu bạn chạy headless, không gắn NVMe HAT, và có backup tự động. Rủi ro trở thành "một ngày nào đó phải restore backup" thay vì "mất dữ liệu".

Nhưng nói thẳng: 400k cho bộ PSU 27W chính hãng đổi lại việc không phải nghĩ về chuyện này nữa, với một thiết bị chạy 24/7 điều khiển nhà, tôi thấy đáng. Nó cũng cho bạn 5.1V và PD hoạt động đúng — tức là mở đường nếu sau này bạn muốn gắn NVMe HAT hay thêm peripheral.

Con số quyết định vẫn là câu tôi hỏi mấy lượt: **Pi có gắn HAT nào không?** Nếu có NVMe qua PCIe thì mọi phân tích trên đổi hoàn toàn — peak thêm 1–2A trên rail 4.94V thì "để nguyên" không còn là lựa chọn an toàn nữa.
