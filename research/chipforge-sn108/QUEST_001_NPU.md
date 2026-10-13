# Phân tích đề `npu_challenge_001` — "NPU with Kernels optimized for 4 AI models"

> Nguồn: gói đề `quest_001_npu.zip` (gồm `npu_v1.0.md`, `EXPECTED.json` ghi ngày 2026-10-05, `evaluator.zip`, các file template) và ảnh chụp leaderboard ngày 13/10/2026.
> Mọi con số dưới đây đều tính lại từ file trong gói đề, trừ những chỗ ghi là *ước lượng*.

## 1. Tình hình trên leaderboard (13/10/2026)

| | |
|---|---|
| Kết thúc | **20/10/2026 20:07** (còn khoảng 7 ngày 2 giờ) |
| Số lượng | 96 bài nộp, 41 miner, 19 lần có winner, 1 bài bị void |
| Winner | UID 124: **2058.31**. Phải đạt **> 2161.23** (+5%) mới vượt |
| Tiếp theo | 1933.37 (UID 139), 1914.42 (UID 124), 1817.91 và 1759.11 (UID 145), 1812.84 (UID 142), 1755.61 (UID 136) |
| Chain | 157 neuron, **5 validator**; phí đăng ký **0.05 TAO (≈ $13.8)**; alpha 0.0127 TAO (≈ $3.52); TAO = $275.77 |
| Thưởng | Ô "Daily miner rewards" đang hiện "—", và dòng "**0% of validator stake agrees**" với winner. Như vậy **chưa chắc winner đang thực sự được trả** |

## 2. Đề bài, tóm gọn

- Bạn nộp **phần cứng** (`module npu`, giữ đúng port trong `interface/npu_if.svh`: AXI4 slave để CPU cấu hình, AXI4 master 128-bit để NPU đọc/ghi DRAM) và **14 hàm kernel C** (`Conv2DEval`, `DepthwiseConv2DEval`, `FullyConnectedEval`, `AddEval`, `PadEval`, …).
- Evaluator cung cấp phần còn lại: SoC với CPU RV32IMC, ITCM 48 KB, DTCM 64 KB, DRAM 1 MB và crossbar AXI. NPU **chỉ truy cập được DRAM** (`0x8040_0000`–`0x804F_FFFF`); 1 KB đầu DRAM dành riêng cho harness.
- C được biên dịch bằng `-march=rv32imac_zicsr -O2 -ffreestanding`, không có libc, không được dùng inline asm. Mã thực thi tối đa **48 KB**.
- Chấm trên 4 model với input int8 ngẫu nhiên. Mỗi byte output phải khớp với `template/kernels.c`:

| Model | MAC/ảnh | Phép toán | Weight int8 | Chu kỳ baseline (CPU) | Chu kỳ/MAC |
|---|---:|---|---:|---:|---:|
| resnet8 | 12.53 M | 9 conv, 3 add, 7 pad, avgpool, fc | 279 KB | 312 238 374 | 24.9 |
| mobilenet_v1_025 | 7.49 M | 14 conv (pointwise), 13 dwconv, 14 pad | 673 KB | 229 215 551 | 30.6 |
| ds_cnn | 2.66 M | 5 conv, 4 dwconv, 5 pad, fc | 144 KB | 86 431 727 | 32.5 |
| ad_fcae | 0.26 M | 10 fully-connected (8 ảnh mỗi seed) | 267 KB | 5 462 408 | 20.7 |

- Ngoài 4 model chính còn có **4 model "audit"** (`mnist_tiny`, `cnn_mid`, `dwcnn`, `mlp`) nằm sẵn trong evaluator. Luật ghi rõ kernel phải "**work for any model weights, not only the published ones**". Hãy coi như validator có thể chạy kiểm tra trên các model này, nên **đừng hard-code riêng cho 4 model**.

## 3. Công thức điểm và hệ quả

```
speedup = geomean_m(baseline_cyc_m / cyc_m) × (fmax_scored / 62.89)
IPS     = fmax_scored(Hz) / geomean_m(cyc_m)
Score   = 1000 × [IPS × (1 − 1/speedup)]^0.7 / max(area_NPU_µm², 1 000 000)^0.3
Gate    : đúng 100% mọi ảnh, mọi seed · 0 lỗi giao thức AXI · speedup ≥ 2.0 · score > 50
```

Năm điều rút ra từ công thức:

1. **Tần số bị chặn ở 62.89 MHz.** `fmax_scored = min(fmax_NPU, 62.89)`, nên làm NPU chạy nhanh hơn không được thêm gì. Chỉ cần đường găng (đo theo `1000/(10 − slack)`) **≤ 15.9 ns**. Lưu ý: STA đặt input/output delay bằng **30% chu kỳ** ở mỗi cổng, nên **hãy đăng ký (register) mọi tín hiệu AXI** để đường đi từ port không trở thành đường găng.
2. **Diện tích dưới 1 000 000 µm² là "miễn phí"**, vì mẫu số tối thiểu là 1M^0.3 = 63.1. Bạn nên dùng gần hết 1 mm² này.
   - SRAM macro rất đắt: macro 1 KB chiếm **190 713 µm²**, macro 2 KB chiếm **284 539 µm²**. Nghĩa là 1 mm² chỉ chứa được khoảng **4–5 KB SRAM**. RAM bằng flip-flop cũng chiếm diện tích thật.
   - Hệ quả là **không thể giữ weight trên chip**; phải stream weight từ DRAM qua bus 16 B/beat.
   - Vượt quá 1M thì phải trả giá theo `area^0.3`. Gấp đôi diện tích chỉ đáng khi speedup tăng hơn khoảng **×1.35** (điều kiện: speedup tăng nhanh hơn `area^0.43`).
3. **Speedup mà các winner đang đạt là khổng lồ.** Giả sử diện tích ≤ 1M µm² (nếu lớn hơn thì speedup thực còn phải cao hơn nữa):

| Điểm | Speedup cần (≤ 1M µm²) | Nếu 2M µm² |
|---:|---:|---:|
| 50 (ngưỡng tối thiểu) | 7.3× | 9.4× |
| 1755 (hạng 7) | ~1 011× | ~1 360× |
| 2058 (winner) | **~1 268×** | ~1 707× |
| **2161.23 (mốc cần vượt)** | **~1 360×** | ~1 830× |

   Điểm tăng theo `(speedup − 1)^0.7`, nên muốn **+5% điểm** cần **+7.2% speedup**, cùng điều kiện diện tích.

4. **Mức thông lượng cần có** (để đạt 1 360× nếu speedup đều cho cả 4 model):
   - resnet8: khoảng 230K chu kỳ/ảnh, tức ~55 MAC/chu kỳ *thực tế*.
   - mobilenet: khoảng 169K chu kỳ/ảnh, ~44 MAC/chu kỳ.
   - ds_cnn: khoảng 64K chu kỳ/ảnh, ~42 MAC/chu kỳ.
   - ad_fcae: khoảng 4K chu kỳ/ảnh, ~66 MAC/chu kỳ.

5. **ad_fcae bị giới hạn bởi băng thông bus** (*ước lượng*, giả sử tốt nhất là 1 beat/chu kỳ):
   - Riêng việc đọc 267 KB weight đã tốn ≥ 16.6K chu kỳ/ảnh, nên speedup của ad_fcae tối đa chỉ khoảng **330×**.
   - Ngoại lệ: nếu tận dụng được việc "loads weights on the first inference" (comment trong `run.py`), nhưng bộ nhớ trên chip lại quá đắt để giữ chừng ấy weight.
   - Vì điểm dùng geomean, **ba model còn lại phải bù**: cần khoảng **2 200×** mỗi model, tức resnet8 ≈ 142K chu kỳ (~88 MAC/chu kỳ), mobilenet ≈ 105K (~71), ds_cnn ≈ 40K (~67).
   - Mobilenet có **673 KB weight**, nên tốn tối thiểu khoảng 42K chu kỳ chỉ để đọc weight. Băng thông bus vì vậy là ràng buộc thật, không chỉ số lượng MAC.

**Định luật Amdahl:** khi đã ở mức hơn 1 000×, **CPU gần như không được làm gì**. Pad, add, requantize, pool, ghi output, và cả vòng lặp cấu hình từng layer đều phải chạy trên NPU hoặc được gộp lại (luật cho phép kernel "fuse, skip or overlap layers", và NPU "may keep working between kernel calls"). Mô hình mà các winner gần như chắc chắn đang dùng: kernel đầu tiên đẩy một **chuỗi descriptor cho cả mạng**, NPU tự chạy hết, và các kernel sau chỉ chờ hoặc không làm gì.

## 4. Kiến trúc gợi ý (khung để đạt khoảng 1 400× trở lên)

- **Mảng MAC int8** 128–256 MAC (ví dụ 16×8 hoặc 16×16, output-stationary), cùng **pipeline requantize theo kênh** (nhân `out_mult`, dịch `out_shift`, cộng zero-point, clamp) làm bằng phần cứng để khớp **từng bit** với `MultiplyByQuantizedMultiplier` trong `kernels.c`. Đây là chỗ dễ sai nhất.
- **Hai đường dữ liệu:** một cho conv/pointwise/FC (dạng GEMM) và một cho depthwise (mỗi kênh độc lập, thường dùng làm đường vector). Mobilenet có 13 layer dwconv; nếu để dwconv chạy chậm thì mobilenet sẽ kéo cả geomean xuống.
- **DMA burst 128-bit** (`awlen` / `arlen` dài), có prefetch weight của layer tiếp theo trong khi đang tính layer hiện tại (double-buffer), và tăng mức tái sử dụng input/weight để giảm lưu lượng DRAM.
- **Pad ảo:** sinh zero-point ngay trong address generator thay vì ghi tensor đã pad ra bộ nhớ. Bỏ hẳn các op `PAD` trong chuỗi xử lý.
- Gộp `ADD` (residual của resnet8) và average pool vào phần epilogue.
- Phần buffer trên chip giữ trong khoảng 4–8 KB (dùng flop hoặc 1–2 SRAM macro). Diện tích còn lại dành cho mảng MAC.
- Đăng ký mọi port, và tuân thủ nghiêm AXI (không bỏ rơi response nào; `axi_checker` sẽ bắt lỗi).

## 5. Các điều cấm, vi phạm sẽ bị −1 hoặc bị loại ở bước intake

- RTL: không dùng `initial`, delay `#`, biến có giá trị khởi tạo, `===`/`!==`, `casex`, `$isunknown`, system task truy cập file hoặc in ấn, DPI, `translate_off`, `bind`/`force`/`defparam`/`$root`. Không được có latch, vòng tổ hợp hay clock sinh từ register. Chỉ dùng 3 loại SRAM macro được cho phép.
- C: không inline asm, không `#embed`. Không được có chuỗi `golden`, `NUM_IMAGES` hay `inputs.h` trong tên.
- Hành vi: chỉ dùng con trỏ được truyền vào và chỉ dùng input hiện tại. Không ghi vào 1 KB đầu DRAM. NPU chỉ được truy cập bộ nhớ từ kernel đầu tiên tới kernel cuối cùng của một inference. Không dựa vào nội dung của bộ nhớ chưa ghi. Phải chạy đúng với weight bất kỳ.
- Giới hạn: tối đa 2 tỷ chu kỳ mỗi lần chạy; **45 phút** cho mô phỏng và 45 phút cho synthesis; synthesis dùng tối đa 6 GB RAM.

## 6. Kế hoạch 7 ngày (cho tới 20/10 20:07)

1. **Ngày 0 — xem có đáng làm không:** kiểm tra trên taostats xem UID 124 có incentive hay không. Hỏi trên Discord vì sao "0% validator stake agrees". Nếu winner thực tế không được trả, cân nhắc dừng lại.
2. **Ngày 0–1 — tải các thiết kế thắng cuộc:** `miner_cli.py reveals` rồi `reveal-download` các bản 1933 / 1755 (và 2058 nếu đã được reveal). Đây là cách nhanh nhất để có một NPU khoảng 1 000×. Viết lại từ đầu trong 7 ngày là không thực tế.
3. **Ngày 1 — dựng evaluator:** `chipforge_eda_server` ở commit **`9e8644e`**, máy ≥ 8 nhân / 32 GB. Chạy `dummy_sol_0.zip` và đối chiếu với `EXPECTED.json` (area **2613.7568 µm²**, fmax 62.89 phải khớp chính xác). Mỗi lần chạy mẫu mất khoảng 14 phút.
4. **Ngày 1–5 — tối ưu bản đã reveal:** đo `per_model.cycles_per_inference` để tìm model đang kéo geomean xuống. Nhắm vào: dwconv của mobilenet, chuỗi layer của ds_cnn, việc stream weight của ad_fcae, overhead cấu hình, và chồng lấp DMA với tính toán. Cần đạt **≥ +7.2% speedup** so với winner; nên nhắm **+12–15%** để có biên an toàn (cycle dao động ±2% theo seed, cộng thêm khoảng 1.2% jitter).
5. **Ngày 5–6 — kiểm tra lại:** chạy nhiều seed và chạy các model audit. Kiểm tra `axi_protocol_errors = 0` và `result: ACCEPTED` trong một lần chạy đầy đủ. Mỗi lần chấm phải dưới 45 phút.
6. **Ngày 6 — nộp**, còn dư thời gian trước giờ đóng. Đề cũng nói biên độ cải thiện **có thể thay đổi**, nên hãy xem lại trên leaderboard trước khi nộp.
