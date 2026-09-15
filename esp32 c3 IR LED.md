## Nguyên nhân chính: dòng qua IR LED quá thấp

Remote gốc và module của bạn khác nhau ở **peak current** chứ không phải ở "chất lượng LED".

| | Remote gốc | Module KY-005 / LED cắm thẳng GPIO |
|---|---|---|
| Peak current | 300 mA – 1 A (pulsed) | 8 – 15 mA |
| Driver | Transistor/MOSFET | Không có, hoặc chỉ có R 220–330Ω |
| Nguồn | 3V pin AA (dòng xả lớn) | GPIO 3.3V (max 40 mA) |

IR LED chỉ sáng theo burst 38 kHz, mỗi pulse ~13 µs. Vì duty cycle thấp nên datasheet cho phép chạy peak rất cao (TSAL6200: I_F liên tục 100 mA nhưng I_FSM lên tới 1.5 A). Remote khai thác đúng điều đó. Module của bạn thì GPIO không thể cấp nổi 100 mA → cường độ bức xạ thấp hơn cỡ 30–50 lần, mà khoảng cách tỉ lệ với căn bậc hai của công suất → range rớt xuống còn vài cm.

## Mạch driver nên dùng

```
        +5V
         │
        ┌┴┐
        │ │ R1 = 6.8Ω / 1W
        └┬┘
         │
        ─┴─  IR LED 940nm (TSAL6200 / TSAL6400)
        ─┬─
         │
         ├──────── C (collector)
      NPN S8050 hoặc MOSFET AO3400 / IRLML2502
         │
GPIO ──[R2 220Ω]── B (base)
         │
         E ─── GND  (GND chung với ESP32)
```

Song song với LED, đặt **C1 = 220 µF electrolytic + 100 nF ceramic** ngay sát LED. Đây là chi tiết hay bị bỏ qua: pulse 500 mA sẽ làm sụt áp nguồn nếu không có bulk cap, LED sáng được vài chục µs rồi tự yếu đi.

Tính R1: `(5V − 1.9V_LED − 0.2V_Vcesat) / 0.45A ≈ 6.8Ω`. Công suất trung bình ở duty 33%: ~0.45W → chọn 1W.

Tính R2: cần I_B ≈ 15 mA để transistor bão hòa ở 450 mA → `(3.3 − 0.7) / 0.015 ≈ 180Ω`, dùng 220Ω là an toàn cho GPIO ESP32.

Nếu muốn mạnh hơn nữa: mắc **2 LED nối tiếp** và hạ R1 xuống ~3.9Ω, hoặc 3 LED hướng ra 3 phía để không phải ngắm chính xác.

## Các yếu tố phụ cần kiểm tra

**Bước sóng.** Phải là **940nm**. LED 850nm nhìn hơi đỏ mắt thường và bị bandpass filter của TSOP làm suy giảm mạnh. Nhìn LED trong tối, nếu thấy ánh đỏ mờ thì bạn đang dùng loại sai.

**Carrier frequency.** Đa số điều hoà dùng 38 kHz, nhưng một số hãng dùng 36.7 hoặc 40 kHz. Lệch 2 kHz thôi là AGC của receiver mất khoảng 30–50% độ nhạy — đủ để bắt bạn phải dí sát. Kiểm tra trong IRremoteESP8266 xem protocol của máy bạn khai báo tần số bao nhiêu.

**Duty cycle.** Giữ ở **33%**, đừng để 50%. Với cùng ngân sách nhiệt, duty thấp cho phép peak current cao hơn → range xa hơn. Trên ESP32 dùng RMT peripheral thì cấu hình `carrier_duty_percent = 33`.

**Vị trí receiver.** IR receiver của điều hoà nằm sau cửa sổ nhựa ở panel hiển thị, không phải ở bo mạch. Bạn có thể chiếu vào tường/trần đối diện cho phản xạ — remote gốc vẫn hoạt động theo kiểu này.

## Verification

1. **Trước khi sửa** — đo dòng trung bình: nối multimeter chế độ mA nối tiếp với LED, gửi lệnh liên tục trong vòng lặp. Module không driver sẽ cho ~3–5 mA trung bình.
2. **Sau khi sửa** — đo lại, phải rơi vào ~120–150 mA trung bình (do duty 33%). Nếu vẫn thấp, transistor chưa bão hòa → giảm R2.
3. **Đo bằng oscilloscope** nếu có: đặt probe ở hai đầu R1, phải thấy burst vuông 38 kHz với biên độ ~3V, cạnh dốc và không bị sag dần trong burst. Nếu biên độ tụt dần → thiếu bulk cap.
4. **Test range** theo nấc: 50cm → 1m → 3m → chiếu phản xạ qua tường. Mạch làm đúng phải đạt 5–7m thẳng hướng.
5. **Kiểm tra nhiệt**: sau khi gửi 20 lệnh liên tiếp, chạm vào R1 và transistor. Ấm là bình thường, nóng rát tức là duty cycle sai hoặc GPIO đang bị kẹt mức HIGH.

Nếu sau khi lắp driver mà range vẫn kém dưới 1m, khi đó mới nghi ngờ đến protocol hoặc tần số carrier chứ không phải phần cứng nữa.

---

## 1. IR LED — thành phần quyết định nhất

Đây là chỗ đáng chi tiền nhất. LED IR bán theo bịch 100 con giá 20k thường không có datasheet, radiant intensity thấp và bước sóng trôi.

**Ưu tiên chọn theo thứ tự:**

| Part | I_e (mW/sr) | Góc chiếu (2θ½) | I_FSM | Ghi chú |
|---|---|---|---|---|
| **Vishay TSAL6200** | 72 @100mA | ±17° | 1.5 A | Lựa chọn tốt nhất, chùm hẹp, đi xa |
| **Vishay TSAL6400** | 40 @100mA | ±25° | 1.5 A | Góc rộng hơn, dễ ngắm hơn |
| **Everlight IR333-A** | 20 @20mA | ±20° | 1 A | Rẻ, phổ biến ở VN, chấp nhận được |
| **Osram SFH 4545** | 180 @100mA | ±10° | 1 A | Rất mạnh nhưng chùm quá hẹp |

Thông số phải nhìn là **I_e (radiant intensity, mW/sr)**, không phải mcd — mcd là đơn vị ánh sáng nhìn thấy, dùng cho IR là vô nghĩa và là dấu hiệu người bán không có datasheet thật.

**Bắt buộc 940nm.** Cách kiểm tra nhanh khi mua: dùng camera điện thoại (camera trước, vì camera sau nhiều máy có IR filter mạnh) soi vào LED đang phát — thấy sáng tím/trắng là ổn. Nhìn bằng mắt thường trong phòng tối mà thấy đỏ rõ → đó là 850nm, sai loại.

**Chiến lược thực tế:** mua 2 con TSAL6200 + 1 con TSAL6400, mắc **nối tiếp**. TSAL6200 đi xa, TSAL6400 phủ góc rộng. Nối tiếp thì cùng một dòng đi qua cả 3, không cần cân dòng như mắc song song.

## 2. Switching element

**Phương án A — NPN transistor (dễ mua, dễ hàn):**

| Part | I_C max | h_FE | Vỏ |
|---|---|---|---|
| **S8050** | 700 mA | 120–400 | TO-92 |
| **2N2222A** | 800 mA | 100–300 | TO-92 |
| **BC337-40** | 800 mA | 250–630 | TO-92 |

S8050 là lựa chọn mặc định — 500đ/con, có ở mọi cửa hàng. Tránh 2N3904 (chỉ 200 mA, không đủ).

**Phương án B — Logic-level MOSFET (tốt hơn về hiệu suất):**

| Part | R_DS(on) @ 3.3V_GS | I_D | Vỏ |
|---|---|---|---|
| **AO3400** | 28 mΩ | 5.7 A | SOT-23 |
| **IRLML2502** | 45 mΩ | 4.2 A | SOT-23 |
| **IRLZ44N** | 22 mΩ | 47 A | TO-220 |

MOSFET không có V_CE(sat) 0.2V như BJT nên LED được nhiều điện áp hơn, nhưng phải là **logic-level** (V_GS(th) < 2.5V). IRF540 phổ biến nhưng **không dùng được** với 3.3V GPIO — nó cần 10V mới mở hoàn toàn.

Nếu bạn hàn tay và không quen SMD, cứ dùng S8050. Chênh lệch hiệu suất không đáng kể ở mức dòng này.

## 3. Tính resistor cho từng cấu hình

Công thức: `R1 = (V_supply − N × V_F − V_drop_switch) / I_peak`

V_F của LED IR ở dòng cao là **~1.6V @ 20mA nhưng lên 2.4V @ 500mA** — đừng dùng con số 1.2V trong datasheet ở dòng thấp, sẽ tính sai và LED cháy.

**Cấu hình 1 — 1 LED, 5V, target 500 mA:**
```
R1 = (5 − 2.4 − 0.2) / 0.5 = 4.8Ω  →  chọn 4.7Ω
P = I²R × duty = 0.5² × 4.7 × 0.33 = 0.39W  →  chọn 1W
```

**Cấu hình 2 — 2 LED nối tiếp, 5V, target 400 mA:**
```
R1 = (5 − 4.8 − 0.2) / 0.4 = 0Ω   ← không đủ headroom!
```
5V không đủ cho 2 LED nối tiếp. Phải nâng nguồn lên 9V hoặc 12V.

**Cấu hình 3 — 3 LED nối tiếp, 12V, target 500 mA (khuyến nghị):**
```
R1 = (12 − 7.2 − 0.2) / 0.5 = 9.2Ω  →  chọn 10Ω
P = 0.5² × 10 × 0.33 = 0.83W  →  chọn 2W (dư nhiệt cho an toàn)
```

Đây là cấu hình mạnh nhất mà vẫn an toàn. Cả 3 LED nhận cùng 500 mA peak, tổng radiant output gấp ~3 lần.

**Base resistor R2** (nếu dùng BJT), cần I_B = I_C / 20 để bão hòa sâu:
```
I_B = 500/20 = 25 mA
R2 = (3.3 − 0.7) / 0.025 = 104Ω  →  chọn 100Ω
```
Lưu ý: 25 mA gần giới hạn 40 mA của GPIO ESP32. Nếu lo, dùng MOSFET (gate gần như không tiêu dòng, chỉ cần R2 = 100Ω để giới hạn inrush, thêm R_pulldown 10kΩ từ gate xuống GND).

## 4. Capacitor — đừng bỏ qua

| Vị trí | Giá trị | Loại |
|---|---|---|
| Bulk, sát LED | 220–470 µF / 16V | Electrolytic, low-ESR |
| Decoupling, sát LED | 100 nF | Ceramic X7R |
| Ngõ vào nguồn | 100 µF | Electrolytic |

Bulk cap là nguồn cấp dòng thật sự trong lúc burst. Nguồn 5V qua dây dài không thể cấp 500 mA trong 13 µs — cap làm việc đó. Thiếu nó thì dù tính toán đúng, LED vẫn yếu.

Đặt cap **cách LED dưới 2cm**. Xa hơn thì inductance của dây làm mất tác dụng.

## 5. Nguồn cấp

**Không lấy 5V từ USB của ESP32.** Pulse 500 mA sẽ làm sụt áp rail, ESP32 có thể brownout reset giữa lúc gửi lệnh.

| Phương án | Đánh giá |
|---|---|
| Adapter 12V/1A riêng + buck 3.3V cho ESP32 | Tốt nhất |
| Adapter 5V/2A, tách 2 nhánh dây từ nguồn | Chấp nhận được |
| USB 5V chung với ESP32 | Chỉ dùng khi test, có bulk cap lớn |
| Pin 18650 | Được, nhưng cần boost lên 12V |

## 6. BOM hoàn chỉnh (cấu hình 3 LED / 12V)

| # | Linh kiện | SL | Giá ước tính |
|---|---|---|---|
| 1 | TSAL6200 (940nm, 5mm) | 2 | 8.000đ |
| 2 | TSAL6400 (940nm, 5mm) | 1 | 4.000đ |
| 3 | S8050 NPN TO-92 | 1 | 500đ |
| 4 | Resistor 10Ω 2W | 1 | 1.000đ |
| 5 | Resistor 100Ω 1/4W | 1 | 200đ |
| 6 | Electrolytic 470µF/16V | 1 | 2.000đ |
| 7 | Ceramic 100nF | 1 | 300đ |
| 8 | PCB lỗ 4×6cm | 1 | 5.000đ |
| 9 | Adapter 12V 1A | 1 | 40.000đ |
| 10 | Buck LM2596 hoặc MP1584 | 1 | 12.000đ |

Tổng ~75.000đ. Chỗ mua ở Hà Nội: chợ Trời (Phố Huế), các shop trên đường Giải Phóng, hoặc đặt Shopee từ shop có bán linh kiện Vishay chính hãng.

## 7. Lưu ý khi lắp ráp

**Phân cực LED IR:** chân dài = anode, và nhìn vào vỏ LED trong suốt sẽ thấy **cup phản xạ to hơn nằm ở phía cathode**. LED IR không sáng nhìn thấy nên lắp ngược rất khó phát hiện — kiểm tra bằng diode mode của multimeter trước khi hàn (đọc ~1.0–1.2V khi đúng chiều).

**Hàn nhanh:** LED IR nhạy nhiệt hơn LED thường. Hàn dưới 3 giây mỗi chân, để nguội giữa hai chân.

**Bố trí 3 LED:** nếu muốn phủ góc rộng, uốn chân LED sao cho 3 LED tỏa ra 3 hướng lệch nhau ~30°. Nếu chỉ cần bắn thẳng vào một máy cố định, để song song hết.

**Dây nối từ mạch tới LED** giữ ngắn dưới 10cm. Dây dài làm tăng inductance, cạnh xung bị bo tròn, TSOP có thể decode sai.

## 8. Verification từng bước

**Bước 1 — Test tĩnh, chưa cắm ESP32.**
Nối chân base qua 100Ω lên 3.3V trực tiếp. LED phải sáng liên tục (nhìn qua camera trước điện thoại). Đo dòng bằng multimeter nối tiếp: phải đọc ~500 mA. Nếu chỉ vài chục mA → transistor chưa bão hòa, kiểm tra lại R2 và chiều lắp transistor (S8050 nhìn mặt phẳng: E–B–C từ trái sang).

**Bước 2 — Đo nhiệt sau 10 giây DC.**
Ở chế độ DC liên tục, R1 sẽ rất nóng — đây là bình thường và cũng là lý do không được để GPIO kẹt HIGH. Ngắt ngay sau 10 giây. Nếu R1 bốc khói ở 10 giây → công suất resistor chọn thiếu.

**Bước 3 — Test carrier.**
Cắm vào ESP32, chạy code phát 38 kHz duty 33%. Đo dòng trung bình: phải rơi vào **150–170 mA**. Nếu đọc ~500 mA → duty cycle bị sai thành 100%, GPIO không toggle. Nếu đọc dưới 50 mA → duty quá thấp hoặc code chưa chạy.

**Bước 4 — Test decode bằng receiver riêng.**
Đây là bước quan trọng nhất, đừng bỏ. Lắp một TSOP4838 lên ESP32 thứ hai (hoặc GPIO khác), chạy `IRrecvDumpV2`. Đặt hai mạch cách nhau **3 mét**, bắn lệnh. Phải decode ra đúng protocol và đúng số bit. Nếu ở 3m mà decode lỗi/thiếu bit → vấn đề nằm ở carrier frequency hoặc timing, không phải công suất.

**Bước 5 — Test range thực tế theo nấc.**
1m → 3m → 5m → 7m, mỗi nấc gửi 10 lệnh, ghi lại tỉ lệ thành công. Mạch làm đúng phải đạt **100% ở 5m** khi bắn thẳng vào panel hiển thị của điều hoà.

**Bước 6 — Test phản xạ.**
Quay LED vào tường hoặc trần, không ngắm trực tiếp vào điều hoà. Nếu vẫn bật/tắt được → công suất đã ngang remote gốc. Đây là bài test khắt khe nhất và cũng là bằng chứng rõ ràng nhất rằng driver hoạt động đúng.

**Bước 7 — Soak test.**
Gửi lệnh mỗi 5 giây trong 30 phút. Sau đó chạm R1 và transistor: ấm (dưới 50°C, tay chịu được) là đạt. Nếu nóng đến mức không chạm nổi → duty cycle sai hoặc firmware không tắt carrier sau khi gửi xong.

Nếu qua hết bước 6 mà range vẫn dưới 2m, lúc đó vấn đề chắc chắn nằm ở protocol/carrier frequency chứ không còn ở phần cứng — quay lại kiểm tra tần số mà thư viện IRremoteESP8266 đang dùng cho model điều hoà của bạn.

---

## Cả 4 con đều dùng được — đều là Vishay 940nm, khác nhau chỉ ở góc chiếu

Đây là dòng TSAL của Vishay, chính là loại tôi khuyến nghị. Không có con nào sai bước sóng hay thiếu dòng. Khác biệt duy nhất là **trade-off giữa góc chiếu và cường độ**.

| Part | Kích thước | Góc (±θ½) | I_e @100mA | I_F liên tục | I_FSM |
|---|---|---|---|---|---|
| TSAL4400 | 3mm | ±25° | ~16 mW/sr | 100 mA | 1.5 A |
| TSAL6100 | 5mm | **±10°** | **~130 mW/sr** | 100 mA | 1.5 A |
| TSAL6200 | 5mm | ±17° | ~72 mW/sr | 100 mA | 1.5 A |
| TSAL6400 | 5mm | ±25° | ~40 mW/sr | 100 mA | 1.5 A |

Tổng công suất bức xạ của cả 4 con gần như bằng nhau. TSAL6100 không "mạnh hơn" TSAL6400 — nó chỉ **dồn cùng lượng ánh sáng đó vào chùm hẹp hơn**, nên đo được ở tâm chùm cao gấp 3 lần. Ra ngoài góc 10° thì nó lại yếu hơn TSAL6400.

## Chọn con nào

**Nếu chỉ mua một loại → TSAL6200.** Đây là điểm cân bằng tốt nhất, cũng là loại nhiều remote gốc dùng. ±17° đủ rộng để không phải ngắm chính xác, mà cường độ vẫn cao gấp gần 2 lần TSAL6400.

**Nếu mạch gắn cố định, ngắm thẳng vào điều hoà → TSAL6100.** Bạn lắp một lần rồi thôi, không cần góc rộng. Với 130 mW/sr, chỉ 1 con cũng đủ vượt xa remote gốc khi bắn thẳng.

**Nếu mạch để bàn, hay xê dịch, hoặc muốn bắn phản xạ qua tường → TSAL6400.** Con này đang được chọn sẵn trong hình. Góc rộng giúp việc phản xạ qua trần/tường hiệu quả hơn nhiều.

**TSAL4400 3mm — bỏ qua.** Cùng góc ±25° với TSAL6400 nhưng cường độ chỉ bằng 40%. Chỉ có lợi khi bạn bị giới hạn không gian vỏ hộp.

## Combo tối ưu cho cấu hình 3 LED nối tiếp

Nếu shop cho mua lẻ từng loại, đặt:

- **2× TSAL6100** — bắn thẳng, tạo chùm chính đi xa
- **1× TSAL6400** — uốn chân lệch ~30° sang bên, phủ vùng góc rộng

Cách này cho bạn vừa range xa vừa dung sai hướng tốt. Nếu shop bán theo bịch cùng loại thì **mua 3 con TSAL6200** là xong, đơn giản và hiệu quả gần tương đương.

## Điểm thuận lợi: mạch không cần tính lại

Cả 4 con có V_F giống hệt nhau (~1.35V typ @ 100mA, lên ~2.4V ở dòng pulse cao). Nghĩa là **toàn bộ tính toán R1 = 10Ω / 2W với nguồn 12V ở trên vẫn đúng nguyên** cho bất kỳ tổ hợp nào bạn chọn. Bạn có thể đổi LED sau này mà không phải thay resistor.

Điểm chung khác đáng lưu ý:
- Cả 4 đều chịu **I_FSM 1.5 A** (pulse ngắn), nên mức 500 mA peak của mạch là rất an toàn — chạy ở khoảng 1/3 giới hạn
- Cả 4 đều là **940nm**, khớp với TSOP4838 và receiver trong điều hoà
- Rise/fall time ~800 ns, thừa nhanh cho carrier 38 kHz

## Verification khi hàng về

**Bước 1 — Xác nhận đúng hàng Vishay.** LED thật có vỏ trong suốt hơi ám xanh nhạt, chân dập phẳng, và in mờ mã trên vành nhựa. Hàng nhái thường vỏ trong veo hoàn toàn và chân tròn.

**Bước 2 — Đo V_F bằng diode mode.** Đặt multimeter ở chế độ diode, đo đúng chiều: phải đọc **1.0–1.2V**. Nếu đọc trên 1.6V hoặc báo OL cả hai chiều → hàng lỗi hoặc không phải LED IR. Nếu đọc ~1.8–2.0V → có thể là 850nm, không phải 940nm.

**Bước 3 — Test phát sáng qua camera.** Cấp 3.3V qua resistor 150Ω, soi bằng camera trước điện thoại. Phải thấy chấm sáng trắng/tím rõ. Đồng thời nhìn bằng mắt thường trong tối: **không được thấy màu đỏ**. Thấy đỏ mờ nghĩa là bạn nhận nhầm 850nm.

**Bước 4 — So sánh góc chiếu.** Nếu mua nhiều loại, đây là cách phân biệt khi chúng trông giống hệt nhau: cấp điện, đặt camera cách 30cm rồi di chuyển camera sang ngang. TSAL6100 sẽ tắt rất nhanh khi lệch góc, TSAL6400 vẫn sáng khá đều. Làm bước này **trước khi hàn**, vì sau khi hàn lên board thì không phân biệt được nữa.

**Bước 5 — Đánh dấu.** Sau khi phân loại xong, chấm sơn móng tay lên chân cathode của từng nhóm để không lẫn. LED IR không sáng nhìn thấy nên lắp ngược là lỗi rất khó tìm.

---

## Đúng, đây là lựa chọn tốt — nhưng phải chạy 12V

3× TSAL6200 nối tiếp là cấu hình chuẩn. Ưu điểm của mắc nối tiếp: cùng một dòng đi qua cả 3 con, không cần resistor cân dòng riêng cho từng LED như khi mắc song song, và không lo LED nào ăn dòng nhiều hơn rồi chết sớm.

**Ràng buộc quan trọng nhất:** 3 LED nối tiếp cần ~7.2V ở dòng pulse cao. Nguồn 5V **không đủ** — bạn sẽ không có headroom cho resistor và LED chỉ sáng lay lắt. Bắt buộc dùng 12V (hoặc tối thiểu 9V).

## Mạch chốt lại

```
       +12V
        │
        ├──── C_bulk 470µF/16V ──┐
        │                        │
        ├──── C 100nF ───────────┤
        │                        │
       ┌┴┐                       │
       │ │ R1 = 10Ω / 2W         │
       └┬┘                       │
        │                        │
       ─┴─  TSAL6200 (1)         │
       ─┬─                       │
       ─┴─  TSAL6200 (2)         │
       ─┬─                       │
       ─┴─  TSAL6200 (3)         │
       ─┬─                       │
        │                        │
        C  S8050                 │
        │                        │
GPIO ─[100Ω]─ B                  │
        │                        │
        E ───────────────────────┴──── GND (chung với ESP32)
```

Kiểm tra lại phép tính:
```
V_F tổng @ 500mA  = 3 × 2.4V = 7.2V
V_CE(sat) S8050   = 0.2V
R1 = (12 − 7.2 − 0.2) / 0.5 = 9.2Ω → chọn 10Ω
P_R1 = 0.5² × 10 × 0.33 duty = 0.83W → dùng 2W
```

500 mA peak so với I_FSM 1.5A của TSAL6200 → chạy ở 1/3 giới hạn, rất an toàn.

## Hai chi tiết dễ sai khi lắp 3 LED

**Chiều lắp.** Nối tiếp nghĩa là **cathode con trước nối anode con sau**. Chỉ cần một con ngược là cả chuỗi tắt hoàn toàn, và vì LED IR không sáng nhìn thấy nên bạn sẽ tưởng transistor hỏng. Đo diode mode toàn chuỗi trước khi hàn vào board: đúng chiều phải đọc ~3.0–3.6V (tổng 3 con).

**Hướng chiếu.** Cả 3 con đều ±17°, nên bạn có hai lựa chọn:

- *Bắn thẳng vào điều hoà cố định* → để 3 LED song song, chùm chồng lên nhau, range xa nhất
- *Muốn dung sai hướng tốt hơn* → uốn chân cho 3 LED tỏa ra lệch nhau ~25°, tổng góc phủ thành ~±40°, đổi lại range giảm còn khoảng 60%

Với điều hoà lắp cố định trên tường, tôi khuyên chọn phương án song song.

## Nguồn 12V cho LED + 3.3V cho ESP32

Đừng cấp 12V vào ESP32. Dùng buck converter:

| Linh kiện | Vào | Ra | Ghi chú |
|---|---|---|---|
| MP1584 mini | 12V | 5V | Nhỏ, đủ dòng cho ESP32 |
| LM2596 | 12V | 5V | To hơn, dễ chỉnh, rẻ |

Rồi cấp 5V vào chân VIN của ESP32. **GND của LED, GND của buck, GND của ESP32 phải nối chung** — nếu không transistor sẽ không nhận đúng mức logic từ GPIO.

Nối GND theo kiểu star ground: tất cả về một điểm chung gần nguồn, đừng nối nối tiếp chuyền từ mạch này sang mạch kia. Pulse 500 mA chạy qua đoạn dây GND chung sẽ tạo sụt áp làm nhiễu ESP32.

## Verification riêng cho cấu hình 3 LED

**Bước 1 — Đo chuỗi LED rời, chưa hàn.** Diode mode, đo hai đầu chuỗi: đọc **3.0–3.6V**. Đọc OL → có con lắp ngược hoặc mối nối hở. Đọc ~1.1V → chỉ có 1 con trong mạch, hai con kia bị nối tắt.

**Bước 2 — Xác nhận nguồn 12V trước khi cắm LED.** Đo tại điểm sẽ nối R1: phải đúng 12V (±0.5V). Adapter rẻ hay ra 11V hoặc 13.5V — nếu ra 11V thì đổi R1 xuống 8.2Ω, nếu 13.5V thì nâng lên 12Ω.

**Bước 3 — Test DC ngắn.** Nối base qua 100Ω lên 3.3V, đo dòng nối tiếp: phải đọc **~500 mA**. Ngắt sau **5 giây** — R1 2W ở chế độ DC liên tục sẽ chịu 2.5W, quá tải. Đây chỉ là test tức thời.

Nếu đọc dưới 200 mA: S8050 chưa bão hòa. Kiểm tra chân E–B–C (nhìn mặt phẳng có chữ, chân từ trái sang là E, B, C).

**Bước 4 — Soi từng con qua camera.** Trong lúc test DC, soi camera trước điện thoại vào **từng LED một**. Cả 3 phải sáng đều. Một con tối hơn rõ rệt → con đó bị hỏng do hàn quá nhiệt, thay ngay.

**Bước 5 — Test carrier 38 kHz.** Cắm ESP32, gửi lệnh. Đo dòng trung bình: phải rơi **150–170 mA** (500 mA × 33% duty). Đọc ~500 mA → duty sai thành 100%. Đọc dưới 50 mA → firmware chưa phát.

**Bước 6 — Test phản xạ.** Quay cụm LED vào tường đối diện, không ngắm điều hoà. Bật/tắt được → cấu hình 3 LED đã vượt công suất remote gốc. Đây là bài test quyết định.

**Bước 7 — Soak test 30 phút.** Gửi lệnh mỗi 5 giây. Sau đó sờ R1: ấm là bình thường, **nóng rát là duty cycle sai** hoặc GPIO bị kẹt HIGH — ngắt điện kiểm tra ngay, 3 LED nối tiếp ở chế độ DC sẽ chết trong vài phút.

Nếu qua được bước 6, mạch của bạn đã xong phần hardware. Vấn đề còn lại (nếu có) chỉ nằm ở protocol và carrier frequency trong IRremoteESP8266.

---

## Điện áp không quyết định độ mạnh — dòng điện mới quyết định

Đây là chỗ dễ nhầm nhất trong thiết kế LED. Độ sáng của LED tỉ lệ với **dòng qua nó**, không phải điện áp cấp. Remote 3V vẫn đẩy được 400–600 mA qua LED, tức là ngang hoặc hơn mạch 12V của bạn.

**Lý do 12V xuất hiện trong tính toán là vì bạn mắc 3 LED nối tiếp**, không phải vì cần "nhiều điện" hơn. Mỗi LED cần ~2.4V ở dòng cao, ba con xếp chồng thành 7.2V — chỉ riêng việc xếp chồng đã đòi hỏi nguồn cao hơn.

Remote gốc chỉ có **một LED duy nhất**.

## Tính toán bên trong remote

```
Pin alkaline AAA mới:        3.2V (2 viên nối tiếp)
V_F của IR LED @ 500mA:      2.4V
V_CE(sat) của transistor:    0.2V
                             ─────
Headroom còn lại:            0.6V
```

0.6V nghe rất ít, nhưng đủ để đẩy 500 mA nếu tổng trở trên đường dây chỉ khoảng 1.2Ω. Và đó chính xác là những gì remote làm:

| Thành phần trở kháng | Giá trị |
|---|---|
| Nội trở 2 viên AAA alkaline | ~0.6Ω |
| Series resistor trên board | 0 – 1Ω (nhiều remote không có) |
| Trở tiếp xúc lò xo + mạch in | ~0.2Ω |
| **Tổng** | **~1.2Ω** |

`I = 0.6V / 1.2Ω = 500 mA`

Nhiều remote **hoàn toàn không có series resistor** — họ cố tình dùng nội trở của pin làm phần tử giới hạn dòng. Mở remote ra bạn sẽ thấy LED nối gần như thẳng vào transistor.

## Hệ quả: đây là lý do pin yếu làm remote yếu

Pin alkaline cũ có nội trở tăng từ 0.3Ω lên 2–3Ω mỗi viên. Điện áp hở mạch vẫn đo được 1.4V/viên (nên bạn tưởng pin còn tốt), nhưng khi LED kéo dòng thì sụt áp gần hết:

```
Pin cũ: I = 0.6V / (5Ω + 0.2Ω) = 115 mA   ← chỉ còn 1/4
```

Range rớt từ 7m xuống còn 1–2m. Đúng hiện tượng bạn gặp với module KY-005 — cùng một nguyên nhân: dòng thấp, không phải điện áp thấp.

## So sánh hiệu suất

| Cấu hình | V_supply | I_peak | P_vào LED | P_đốt ở R1 | Hiệu suất |
|---|---|---|---|---|---|
| Remote gốc (1 LED, 3V) | 3.2V | 500 mA | 1.2W | ~0.3W | **80%** |
| Bạn: 3 LED nối tiếp, 12V | 12V | 500 mA | 3.6W | 2.5W | 59% |
| 3 LED nối tiếp, 9V | 9V | 500 mA | 3.6W | 0.8W | **82%** |
| 1 LED, 5V | 5V | 500 mA | 1.2W | 1.2W | 50% |

Remote hiệu suất cao vì gần như không lãng phí điện áp thừa. Mạch 12V của bạn đốt 2.5W vô ích trong resistor — vẫn chạy tốt, chỉ là resistor nóng và phải chọn loại 2W.

Điểm đáng chú ý: **9V thực ra tốt hơn 12V** cho cấu hình 3 LED nối tiếp. Cùng ánh sáng ra, nhưng R1 chỉ đốt 0.8W thay vì 2.5W, và bạn dùng được resistor 1W nhỏ gọn thay vì 2W.

## Ba lựa chọn cho bạn

**A — Giữ 3 LED nối tiếp, đổi sang adapter 9V.** Hiệu suất tốt nhất, ánh sáng gấp 3 lần remote.
```
R1 = (9 − 7.2 − 0.2) / 0.5 = 3.2Ω  →  3.3Ω
P  = 0.5² × 3.3 × 0.33 = 0.27W     →  1W là dư
```
Lưu ý: dùng **adapter 9V**, đừng dùng viên pin 9V vuông. Pin 9V có nội trở 1.5–3Ω, sẽ sụt áp ngay khi LED kéo dòng.

**B — Giữ nguyên 12V như đã chốt.** Vẫn hoàn toàn ổn, chỉ cần R1 10Ω/2W. Ưu điểm là adapter 12V dễ kiếm hơn 9V, và bạn đã có sẵn buck converter trong BOM.

**C — Bắt chước remote: 1 LED, nguồn 5V.** Đơn giản nhất, dùng chung nguồn USB với ESP32, không cần buck.
```
R1 = (5 − 2.4 − 0.2) / 0.5 = 4.8Ω  →  4.7Ω / 1W
```
Đạt đúng bằng remote gốc (5–7m). Nếu mục tiêu của bạn chỉ là "mạnh như remote" chứ không cần hơn, phương án này tiết kiệm và ít thứ để sai nhất.

Nếu đã mua 3 con TSAL6200 rồi thì chọn **A** — chỉ khác adapter, còn lại giữ nguyên thiết kế.

## Verification cho câu hỏi này

Bạn có thể tự kiểm chứng lập luận trên bằng chính cái remote đang dùng:

**Bước 1 — Đo dòng thật của remote.** Tháo pin, dùng dây nối multimeter (chế độ **A**, không phải mA — dòng vượt 200 mA sẽ đứt cầu chì thang mA) nối tiếp vào khe pin. Bấm giữ một nút. Đọc được **80–150 mA** trung bình. Nhân với 3 (do duty cycle 33%) ra peak ~300–450 mA. Đây là bằng chứng trực tiếp rằng remote 3V vẫn chạy dòng cao.

**Bước 2 — Đo nội trở pin.** Đo áp hở mạch của 1 viên AAA (`V_oc`). Sau đó lắp vào remote, bấm giữ nút và đo lại ngay tại cực pin (`V_load`).
```
R_internal = (V_oc − V_load) / I_đo_được_ở_bước_1
```
Pin mới ra ~0.3Ω/viên, pin cũ ra 2Ω trở lên. Nếu remote nhà bạn đang yếu, bước này sẽ chỉ ra ngay nguyên nhân.

**Bước 3 — Mở remote tìm series resistor.** Nhìn đoạn mạch từ LED về transistor. Phần lớn remote sẽ **không có resistor nào** ở đó, hoặc chỉ có một con 1–2.2Ω. Đây là xác nhận trực quan cho toàn bộ giải thích ở trên.

**Bước 4 — Test đối chứng sau khi mạch xong.** Đặt mạch của bạn và remote gốc cạnh nhau, cùng khoảng cách 5m, cùng bắn vào điều hoà. Nếu mạch bạn hoạt động ổn định trong khi remote thỉnh thoảng trượt → bạn đã vượt remote gốc.

---

Bạn đã có sẵn Home Assistant + ESPHome + Mosquitto chạy Docker trên Pi 5, nên hướng đi tối ưu là làm con này thành **một ESPHome node** thay vì viết firmware Arduino rồi tự lo MQTT. Lý do cụ thể tôi nói ở phần dưới.

Trước hết là sơ đồ nối chân cho ESP32-C3 (bản này thay cho sơ đồ ESP8266 tôi còn nợ bạn ở trên):## Tại sao nên làm ESPHome node thay vì firmware tự viết

Bạn đã có ESPHome dashboard trong stack Docker ở `~/smarthome`, nên con IR blaster này thêm vào gần như không tốn công hạ tầng:

- ESPHome có sẵn `remote_transmitter` với `carrier_duty_percent` — đúng thứ bạn cần điều chỉnh
- Các platform `climate_ir_*` đã encode sẵn protocol của hàng chục hãng điều hoà
- Ra HA thành một **climate entity** chuẩn, tự động có card điều hoà với nút bật/tắt, chỉnh nhiệt độ, chọn mode
- Qua cầu HomeKit bạn đang dựng, nó xuất hiện trong Apple Home như một thermostat, không cần cấu hình thêm
- OTA update — quan trọng vì mạch sẽ gắn cố định trên tường, không muốn tháo ra cắm USB mỗi lần sửa

## Điểm khác biệt lớn so với cái quạt

Remote điều hoà là **stateful** — mỗi lần bấm, nó gửi trọn gói trạng thái: on/off + mode + nhiệt độ + tốc độ quạt + swing, tất cả trong một packet dài 100–300 bit.

Hệ quả rất có lợi: HA và điều hoà **không bao giờ lệch trạng thái**. Bạn bấm "26°C, cool, quạt vừa" trên dashboard thì máy nhận đúng chừng đó, kể cả khi trước đó ai đó vừa chỉnh bằng remote tay.

Cái quạt thì ngược lại — nút toggle, ESP phải tự đoán trạng thái và rất dễ desync. Đó là hai bài toán khác nhau, nên tôi khuyên **tách thành hai entity riêng** trong cùng một node.

## Cấu hình ESPHome

```yaml
substitutions:
  device_name: ir-blaster-pn
  friendly_name: "IR Blaster phòng ngủ"

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

logger:
  hardware_uart: USB_SERIAL_JTAG

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
  ap:
    ssid: "${device_name} fallback"

# ---------- IR TX ----------
remote_transmitter:
  pin: GPIO3
  carrier_duty_percent: 50%

# ---------- IR RX (chỉ bật khi học lệnh) ----------
remote_receiver:
  pin:
    number: GPIO4
    inverted: true
    mode:
      input: true
      pullup: true
  tolerance: 55%
  dump: all

# ---------- Nhiệt độ phòng lấy từ node t2 sẵn có ----------
sensor:
  - platform: homeassistant
    id: room_temp
    entity_id: sensor.t2_temperature
    internal: true

  - platform: wifi_signal
    name: "${friendly_name} WiFi"
    update_interval: 120s

# ---------- Điều hoà ----------
climate:
  - platform: daikin          # đổi theo hãng máy của bạn
    name: "Điều hoà"
    sensor: room_temp

# ---------- Quạt: nút toggle rời ----------
button:
  - platform: template
    name: "Quạt nguồn"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0x00FF
          command: 0x8877

  - platform: template
    name: "Quạt tốc độ"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0x00FF
          command: 0x48B7

  - platform: template
    name: "Quạt đảo gió"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0x00FF
          command: 0x28D7

status_led:
  pin:
    number: GPIO8
    inverted: true
```

Ba mã NEC của quạt ở trên là **placeholder** — bạn phải thay bằng mã học được từ remote thật.

## Chọn đúng climate platform

Danh sách ESPHome hỗ trợ sẵn: `daikin`, `daikin_arc`, `daikin_brc`, `panasonic`, `toshiba`, `mitsubishi`, `fujitsu_general`, `hitachi_ac344`, `hitachi_ac424`, `lg`, `gree`, `midea_ir`, `tcl112`, `whirlpool`, `coolix`, `delonghi`, `emmeti`, `heatpumpir`.

Vài lưu ý cho thị trường Việt Nam: **Casper** thường dùng protocol `gree` hoặc `tcl112`. **Funiki** và **Nagakawa** hay là `coolix`. **Sumikura** thường `gree`. Cứ thử lần lượt, ESPHome build lại chỉ mất 1–2 phút.

Nếu không platform nào khớp, `heatpumpir` phủ thêm rất nhiều model — nhưng nó cần khai `protocol` và `horizontal_default`/`vertical_default`.

Còn nếu vẫn không được, quay về dùng `remote_transmitter.transmit_raw` với mã học từ TSOP, rồi tạo template switch cho từng tổ hợp hay dùng (ví dụ chỉ cần 3 preset: tắt / 26°C cool / 28°C cool). Xấu hơn nhưng chạy được ngay.

## Lắp đặt cố định

**Hướng ngắm.** TSAL6200 góc ±17°. Ở khoảng cách 4m, vòng chiếu có bán kính khoảng 1.2m — thừa để trùm panel hiển thị của điều hoà, không cần ngắm quá chính xác. Nếu bạn lắp cách xa hơn 6m thì cân nhắc xoay một con LED lệch ra để nới góc.

**Vỏ in 3D.** Đây là chỗ hay hỏng việc: **PLA và PETG chặn 940nm rất mạnh**, kể cả loại trong suốt. Đừng in "cửa sổ" mỏng cho LED. Thiết kế trong Fusion 360 với **lỗ khoét hở hẳn** đường kính 6mm cho mỗi LED 5mm, LED nhô ra hoặc ngang mặt vỏ. Nếu sợ bụi thì dán một mẩu acrylic IR-pass (loại đen đục nhìn thấy nhưng cho IR qua), đừng dùng nhựa in.

Nhiệt không phải vấn đề ở đây — dù R1 chịu 0.4W lúc phát, mỗi lệnh chỉ dài ~100ms và ngày gửi vài chục lần, nên duty trung bình gần như bằng không. Không cần lỗ thoát nhiệt.

**Nguồn.** Adapter 9V cắm liên tục 24/7 → chọn loại có chứng nhận, đừng dùng adapter trôi nổi. Tiêu thụ standby của cả mạch khoảng 0.5–0.8W, không đáng kể.

**WiFi.** Bạn đã chuyển các router tầng sang AP mode cho flat subnet nên roaming ổn. Nhớ đặt `power_save_mode: none` như trong config — modem sleep làm ESP32-C3 trễ 100–300ms khi nhận lệnh từ HA, cảm giác "bấm mà lâu mới chạy".

## Verification

**Bước 1 — Học mã trước khi lắp vỏ.** Flash config có `remote_receiver`, mở log ESPHome (`docker compose logs -f esphome` hoặc web dashboard), chĩa remote điều hoà vào TSOP4838 và bấm. Log phải in ra protocol đã nhận dạng được, ví dụ `Received Daikin: ...`. Dòng này cho bạn biết ngay nên chọn platform nào.

Nếu chỉ thấy `Received Raw: [...]` mà không có tên protocol → máy bạn không nằm trong danh sách hỗ trợ, chuyển sang phương án raw.

**Bước 2 — Test phát ở khoảng cách gần, chưa lắp cố định.** Đặt mạch cách điều hoà 1m, bấm nút trên HA. Nếu máy phản hồi (kêu bíp) → protocol đúng. Nếu không → đổi platform khác, build lại.

**Bước 3 — Đo dòng xác nhận driver hoạt động.** Multimeter thang A nối tiếp với rail 9V, gửi một lệnh climate. Dòng đỉnh trong lúc phát phải thấy nhảy lên **120–200 mA** trung bình. Nếu chỉ vài mA → MOSFET không mở, kiểm tra gate resistor và pulldown.

**Bước 4 — Test ở vị trí lắp thật.** Gắn tạm bằng băng dính hai mặt, gửi 10 lệnh liên tiếp cách nhau 3 giây. Phải 10/10 thành công. Nếu trượt vài lệnh → lệch hướng ngắm, chỉnh lại rồi test lại trước khi khoan bắt vít.

**Bước 5 — Test đồng bộ trạng thái.** Bấm bật trên HA → tắt bằng remote tay → bấm bật lại trên HA. Máy phải bật đúng. Đây là bài test xác nhận climate platform gửi full state chứ không phải toggle.

**Bước 6 — Tắt receiver rồi flash lại.** Sau khi xong, comment toàn bộ block `remote_receiver` và OTA lại. Để nó chạy sẽ ngốn CPU liên tục decode nhiễu IR từ đèn huỳnh quang và ánh nắng, đôi khi gây log spam. Giữ lại phần config trong file (dạng comment) để sau này cần học thêm remote khác thì bật lên.

**Bước 7 — Soak test qua đêm.** Tạo automation trong HA gửi lệnh mỗi 30 phút. Sáng hôm sau kiểm tra log xem có lần nào `Climate command failed` hoặc node bị disconnect không. Đồng thời sờ vỏ mạch — phải mát hoàn toàn.

Nếu bạn cho biết hãng và model điều hoà, tôi giúp thu hẹp danh sách platform cần thử.
