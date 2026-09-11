## Cách hoạt động

Pi 5 có PMIC (DA9091) đo được điện áp và dòng trên từng rail nội bộ. Không có lệnh nào in ra "tổng công suất" sẵn — phải lấy toàn bộ rail rồi nhân `V × A` và cộng lại.

Xem dữ liệu thô trước:

```bash
vcgencmd pmic_read_adc
```

Bạn sẽ thấy khoảng 12 cặp rail: tên kết thúc `_A` là dòng, `_V` là điện áp. `EXT5V_V` là điện áp đầu vào (chỉ có volt, không có current).

## Script tính tổng

```bash
cat > ~/pwr.sh <<'EOF'
#!/usr/bin/env bash
vcgencmd pmic_read_adc | awk '
/current/ { n=$1; sub(/_A$/,"",n); v=$2; sub(/.*=/,"",v); amp[n]=v+0 }
/volt/    { n=$1; sub(/_V$/,"",n); v=$2; sub(/.*=/,"",v); vol[n]=v+0 }
END {
  printf "%-12s %8s %9s %8s\n", "RAIL", "VOLT", "CURRENT", "POWER"
  for (n in amp) if (n in vol) {
    p = amp[n] * vol[n]; total += p
    printf "%-12s %7.3fV %8.3fA %7.2fW\n", n, vol[n], amp[n], p
  }
  printf "%-12s %7.3fV %8s %7.2fW  <-- input\n", "EXT5V", vol["EXT5V"], "-", 0
  printf "\n%-12s %26s %7.2fW\n", "TONG (board)", "", total
}'
EOF
chmod +x ~/pwr.sh
~/pwr.sh
```

## Script này KHÔNG tính những gì

Đây là phần quan trọng nhất, đặc biệt với setup của bạn:

- **NVMe trên HAT M.2 không được tính.** HAT lấy nguồn từ chân 5V trên GPIO header và có regulator 3.3V riêng — nằm hoàn toàn ngoài các rail mà PMIC đo. Nghĩa là 3–5W của SSD **vô hình** với script này.
- Thiết bị cắm cổng USB cũng không được tính.
- Quạt tản nhiệt không được tính.
- Tổn thất chuyển đổi của chính PMIC (~10–15%) không được tính.

Nên con số "TONG (board)" thường **thấp hơn công suất thực ở ổ cắm khoảng 30–50%** trong trường hợp có HAT + NVMe như bạn. Dùng nó để so sánh tương đối giữa các trạng thái tải, không dùng làm số tuyệt đối.

## Theo dõi liên tục khi chạy tải

Cách này hữu ích hơn số đơn lẻ — xem công suất biến động lúc SSD ghi nặng:

```bash
# terminal 1: log mỗi giây
while true; do
  printf '%s ' "$(date +%T)"
  vcgencmd pmic_read_adc | awk '
  /current/{n=$1;sub(/_A$/,"",n);v=$2;sub(/.*=/,"",v);a[n]=v+0}
  /volt/{n=$1;sub(/_V$/,"",n);v=$2;sub(/.*=/,"",v);o[n]=v+0}
  END{for(n in a) if(n in o) t+=a[n]*o[n];
      printf "board=%.2fW  in=%.3fV\n", t, o["EXT5V"]}'
  sleep 1
done | tee ~/power.log
```

```bash
# terminal 2: tạo tải
stress-ng --cpu 4 --io 2 --timeout 120s
```

Rồi xem lúc nào áp tụt sâu nhất:

```bash
sort -t= -k3 -n ~/power.log | head -5
```

Với `EXT5V_V = 4.95V` lúc idle của bạn, mục đích ở đây là xem dưới tải nó rơi xuống bao nhiêu. Xuống gần `4.7V` là quá sát ngưỡng 4.63V.

## Đưa vào Home Assistant

Bạn đang chạy HA rồi nên tận dụng luôn — có history graph để nhìn xu hướng dài hạn. Nhưng container HA không có `vcgencmd`, nên phải qua đường khác: dùng ESPHome? Không phù hợp. Cách gọn nhất là ghi ra file rồi HA đọc file:

```bash
# trên Pi, cron mỗi phút
crontab -e
```

```
* * * * * /home/huy/pwr.sh | awk '/TONG/{print $3}' | tr -d 'W' > /home/huy/smarthome/homeassistant/power.txt
```

Trong `configuration.yaml`:

```yaml
sensor:
  - platform: file
    name: Pi5 Board Power
    file_path: /config/power.txt
    unit_of_measurement: W
```

Cần thêm `file` vào allowlist:

```yaml
homeassistant:
  allowlist_external_dirs:
    - /config
```

## Đo công suất thật — cần thiết bị đo

Với vấn đề undervoltage bạn đang gặp, đây mới là thứ trả lời đúng câu hỏi. Script trên không đo được thứ bạn cần biết: adapter đang cấp bao nhiêu W và có đủ hay không.

Mua một **USB-C inline power meter** (loại có màn hình, đo được V/A/W, giá vài trăm nghìn). Cắm giữa adapter và Pi. Nó cho biết:

- Điện áp thực tại đầu vào Pi dưới tải — số đáng tin hơn `EXT5V_V`
- Dòng thực đang rút, so với 5A mà adapter hứa
- **Sụt áp trên cáp**: đo ở hai đầu, chênh lệch chính là hao trên dây. Đây là cách chứng minh dứt điểm giả thuyết "do cáp" mà mình nêu ở lượt trước

Hoặc smart plug có đo điện (Tapo P110, Shelly Plug S) cắm ở ổ tường — đo tổng AC bao gồm cả tổn thất adapter, và tích hợp thẳng vào HA. Số này cao hơn DC khoảng 15–20% do hiệu suất adapter.

## Con số kỳ vọng

| Trạng thái | Board (script) | Thực ở ổ cắm |
|---|---|---|
| Idle, không NVMe | 2.5–4 W | 4–6 W |
| Idle + NVMe + HAT | 3–5 W | 7–10 W |
| HA + Docker chạy thường | 4–6 W | 9–12 W |
| CPU 4 core full + SSD ghi | 8–11 W | 15–22 W |
| Peak (boot, PCIe training) | — | có thể vọt 25W+ |

Dòng cuối giải thích đúng hai event undervoltage lúc `00:02:52` của bạn: peak lúc boot vượt khả năng tức thời của adapter/cáp.

Chạy `~/pwr.sh` rồi paste output, kèm phần theo dõi liên tục trong lúc `stress-ng` — mình đối chiếu xem mức tiêu thụ có bất thường hay chỉ là giới hạn đường dây.
