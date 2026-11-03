# Lab4_FAP201_IC2001
# Lab4_FAP201 – Debouncing a Switch

## Thông tin chung
- **Môn học:** FAP201 - FPGA Programming
- **Bài thực hành:** Lab04 - Debounce switch (2FF sync -> counter -> stable -> release -> LED toggle) (CLO1 & CLO2)
- **Tác giả:** La Hoàng Khả Vy (CE210538)[cite: 12]
- **Mục tiêu:** Thiết kế, kiểm chứng và hiện thực mạch chống dội công tắc (Switch Debouncer) kết hợp mạch phát hiện sự kiện nhả nút (Release Event) để đảo trạng thái đèn LED một cách ổn định trên bo mạch FPGA[cite: 12].

## Môi trường và Phần cứng
- **Công cụ phát triển:** Gowin EDA[cite: 14, 15]
- **Bo mạch (Board):** Xiaomeige ACG525
- **Chip FPGA (Target Device):** GW5A-LV25UG324C2/I1[cite: 14]
- **Ngôn ngữ:** VHDL (IEEE 1164 STD_LOGIC, NUMERIC_STD)[cite: 12]

## Nguyên lý hoạt động & Phân tích Architecture
- **Chuỗi xử lý tín hiệu (Processing Pipeline):**
  1. **Đồng bộ hóa 2-FF (2FF Synchronizer):** Tín hiệu công tắc vật lý `i_sw_raw` (Active-Low) được lấy nghịch đảo (`1 = Pressed`) và đi qua 2 Flip-Flop đồng bộ (`sw_sync1`, `sw_sync2`) để triệt tiêu hiện tượng bất ổn định (Metastability) trước khi đưa vào bộ đếm[cite: 12].
  2. **Bộ đếm chống dội (Debounce Counter):** Sử dụng bộ đếm `stable_count` với tham số `STABLE_CYCLES = 1_000_000` chu kỳ (tương ứng 20ms ở tần số 50 MHz)[cite: 12]. Bộ đếm tự động tính toán số bit động qua hàm `clog2`[cite: 12]. Nếu `sw_sync2` khác với trạng thái `sw_stable`, bộ đếm sẽ đếm cho tới khi đạt `TERMINAL_COUNT` mới cập nhật `sw_stable`[cite: 12]. Mọi nhiễu/glitch ngắn hơn 20ms sẽ bị loại bỏ hoàn toàn (bộ đếm bị xóa về 0)[cite: 12].
  3. **Phát hiện sườn sự kiện nhả nút (Release Event Detector):** Bắt sườn xuống của tín hiệu `sw_stable` qua hai thanh ghi `evt_cur` và `evt_prev`[cite: 12]. Biểu thức `release_event <= evt_prev and (not evt_cur);` chỉ tạo ra xung tích cực `'1'` duy nhất 1 chu kỳ clock khi người dùng nhả nút bấm ra[cite: 12].
  4. **Đảo trạng thái LED (Toggle LED):** Thanh ghi `led_state` thực hiện đảo trạng thái khi `release_event = '1'`[cite: 12]. Tín hiệu reset đồng bộ `i_rst` (Active-Low) đưa mạch về trạng thái ban đầu (`led_state = '0'`, `sw_stable = '0'`)[cite: 12].

---

## Kịch bản kiểm thử Testbench (Self-Checking Testcases)
Testbench `lab04_tb.vhd` kiểm thử tự động toàn bộ 7 kịch bản hoạt động với tham số giả lập `STABLE_CYCLES = 5`:

| Kịch bản | Mô tả kiểm thử (Stimulus) | Kết quả kỳ vọng (Expected Output) | Trạng thái kiểm tra |
| :---: | :--- | :--- | :---: |
| **T0** | Kích hoạt Reset đồng bộ (`i_rst = '0'`) | `o_led = '0'`, `o_dbg_stable = '0'` | PASS |
| **T1a** | Tạo xung Glitch nhấn ngắn ($3 \text{ cycles} < N$) | Tín hiệu `o_dbg_stable` và `o_led` không đổi[cite: 13] | PASS[cite: 13] |
| **T1** | Tạo nhiễu Bounce khi nhấn rồi giữ ổn định $\ge N$ | `o_dbg_stable` chuyển lên `'1'`, `o_led` giữ nguyên `'0'`[cite: 13] | PASS[cite: 13] |
| **T2** | Giữ nút bấm trong thời gian dài (20 clock cycles) | `o_led` không bị đảo trạng thái bất ngờ[cite: 13] | PASS[cite: 13] |
| **T3** | Tạo nhiễu Bounce khi nhả rồi giữ ổn định $\ge N$ | `o_dbg_stable` về `'0'`, `o_led` **toggles ON** (`'1'`)[cite: 13] | PASS[cite: 13] |
| **T4** | Đứng yên ở trạng thái nhả nút | `o_led` duy trì mức logic `'1'`[cite: 13] | PASS[cite: 13] |
| **T5** | Chu kỳ nhấn/nhả thứ hai có dội nhiễu | Nhấn không đổi LED, nhả $\rightarrow$ `o_led` **toggles OFF** (`'0'`)[cite: 13] | PASS[cite: 13] |
| **T6** | Kích hoạt Reset khi LED đang bật | Reset lập tức xóa `o_led` về `'0'`, không tạo xung giả[cite: 13] | PASS[cite: 13] |

---

## Bảng ánh xạ chân vật lý & Timing Constraints

### Physical Constraints Mapping (`lab04.cst`)
Cấu hình ngõ vào/ngõ ra trong file `lab04.cst`[cite: 14]:

| Tín hiệu Port | Chân FPGA | Linh kiện phần cứng | Cấu hình I/O (3.3V Bank VCCIO) |
| :--- | :---: | :--- | :--- |
| `clk` | `T9` | System Clock (50 MHz) | `IO_TYPE=LVCMOS33`[cite: 14, 15] |
| `i_sw_raw` | `B16` | Switch / KEY0 (Raw Input) | `IO_TYPE=LVCMOS33`[cite: 14] |
| `i_rst` | `C15` | Switch / KEY2 (Active-Low Reset) | `IO_TYPE=LVCMOS33`[cite: 14] |
| `o_led` | `D14` | Output LED / LED0 (Active-High) | `IO_TYPE=LVCMOS33`[cite: 12, 14] |
| `o_dbg_stable` | `C14` | Debug LED / LED1 (Active-High Debug) | `IO_TYPE=LVCMOS33`[cite: 12, 14] |

### Timing Constraints (`lab04.sdc`)
Ràng buộc thời gian cho xung nhịp hệ thống 50 MHz định nghĩa trong file `lab04.sdc`:
```tcl
create_clock -name clk -period 20 -waveform {0 10} [get_ports {clk}]
