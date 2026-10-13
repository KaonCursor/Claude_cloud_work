# Kiến trúc SoC chấm điểm và vị trí của NPU (npu-v1.0)

> Đọc từ `evaluator.zip` của `npu_challenge_001`: `verilator/rtl/npu_soc_top.sv`, `axi_xbar.sv`, `axi_sram_dp.sv`, `mem2axi.sv`, `tb/tb_npu_soc.sv`, `sw/main.c`, `sw/interpreter.c`, `sw/link_tcm.ld` và `template/npu.sv`.

## 1. Sơ đồ

```
                 ┌──────────── npu_soc_top (của evaluator, KHÔNG tính diện tích) ────────────┐
  ITCM 48 KB ───►│ rv32imc_top (CPU, RV32IMAC)                                               │
  (.text, chỉ    │   │ cổng bộ nhớ đơn giản (addr/wdata/we/re → rdata/ack)                   │
   fetch lệnh)   │   ├──► 0x8080_xxxx → DTCM 64 KB (1 chu kỳ, ngoài bus): stack, biến C       │
                 │   └──► còn lại → mem2axi (mỗi truy cập = 1 giao dịch AXI single-beat)    │
                 │                    │ AXI 128-bit                                          │
                 │              axi_xbar (1 master → 2 slave)                                │
                 │               ├── 0x8040_0000/4 MB → DRAM cổng A ─┐                      │
                 │               └── 0x3000_0000/16 MB → npu.s_* ──┐ │                      │
                 │                                                 ▼ │                      │
                 │   ┌───────────────── npu (CỦA BẠN, tính diện tích) ──┐                    │
                 │   │ AXI4 slave  s_*  : CPU ghi cấu hình / đọc trạng thái│                  │
                 │   │ AXI4 master m_*  : DMA 128-bit ───────────────────────► DRAM cổng B  │
                 │   └────────────────────────────────────────────────────┘   (riêng NPU)  │
                 └────────────────────────────────────────────────────────────────────────────┘
  DRAM = axi_sram_dp 1 MB, HAI CỔNG ĐỘC LẬP: A (CPU qua xbar) và B (riêng cho NPU)
  axi_checker + axi_usage_monitor trong testbench kiểm tra giao thức trên cả hai cổng của npu
```

## 2. Các thành phần

| Khối | Đặc điểm quan trọng với người thiết kế |
|---|---|
| **CPU** `rv32imc_top` | Lõi 5 tầng in-order (cùng họ chipforge-mcu). Code ≤ 48 KB trong ITCM. Không dùng được `mcycle` để đo. |
| **DTCM** 64 KB | Dữ liệu riêng của CPU, độ trễ 1 chu kỳ: `static` trong `kernels.c`, stack 16 KB. **NPU không nhìn thấy.** |
| **DRAM** 1 MB `0x8040_0000` | Chứa weight, bias, multiplier, shift, buffer activation (`.dram_noinit`), input và output. 1 KB đầu là vùng kết quả của harness. Buffer mà bạn muốn NPU thấy phải khai báo `SHARED` hoặc `SHARED_BSS`. |
| **CPU → DRAM** | Đi qua `mem2axi` + crossbar: **mỗi lần load/store 32-bit là một giao dịch AXI riêng**, nên chậm hơn nhiều so với DTCM. Đây là lý do baseline tốn 20–32 chu kỳ/MAC. |
| **Cửa sổ cấu hình NPU** `0x3000_0000` (16 MB) | CPU đọc/ghi vào `npu.s_*`. Không có ngắt, nên CPU phải **poll** thanh ghi trạng thái. |
| **NPU master → DRAM cổng B** | Cổng riêng, không tranh chấp với CPU. Đọc: nhận AR, mỗi chu kỳ trả 1 beat 16 B, và **chỉ một burst đọc tại một thời điểm** (AR mới chỉ được nhận sau khi burst trước xong). Ghi cũng tương tự, nhưng **kênh đọc và ghi chạy song song được**. Đỉnh khoảng 16 B/chu kỳ mỗi chiều; dùng burst dài (tối đa 256 beat, không vượt ranh giới 4 KB) để bù chi phí mỗi burst. Địa chỉ ngoài 1 MB sẽ bị *alias* theo 20 bit thấp chứ không báo lỗi. |

## 3. Luồng phần mềm (`main.c` → `interpreter.c` → kernel của bạn)

```
reset → crt0 → main(): STAGE(3)
for img in NUM_IMAGES:                      # ad_fcae 8 ảnh/seed, các model khác 1 ảnh
    copy input (CPU, từ DTCM → DRAM)        # chi phí harness, tính vào chu kỳ của mọi người
    RunInterpreter(nodes):                  # vòng lặp cố định theo thứ tự node
        for node: kKernels[node.op](node, tensors)   ← 14 hàm của bạn
    copy output → outlog (CPU)
done → testbench so outlog với golden ở phía validator
```
- Đồng hồ đo chạy **từ reset tới done**, gồm cả việc cấu hình NPU, nạp weight và các lần copy của harness.
- Model là dữ liệu: bảng `tensors[]` cộng danh sách `nodes[]`. Ví dụ resnet8: `PAD → CONV → PAD → CONV → … → ADD → AVG_POOL → FC`. Mỗi `Node` mang các con trỏ `input / filter / bias / output`, `ConvParams`, `out_mult` / `out_shift` theo kênh, và `inputs[]` cho ADD/CONCAT.
- Layout tensor theo TFLite: activation **NHWC int8**, weight conv **OHWI**, weight depthwise **1HWO**, bias int32, requantize theo kênh (`MultiplyByQuantizedMultiplier`).

## 4. NPU "hiện tại" trong template = rỗng

`template/npu.sv` (`npu_stub`) chỉ có hai việc:
- **Slave:** nhận mọi lệnh ghi/đọc, trả `OKAY`, đọc ra luôn bằng 0, không lưu gì.
- **Master:** để trơ (`m_*valid = 0`, `m_bready = m_rready = 1`).

Vì vậy mọi phép tính chạy trên CPU bằng `kernels.c` tham chiếu. Kết quả là speedup 1.000×, diện tích 2 613.76 µm², `npu_fmax` 210.97 MHz (nhưng bị chặn ở 62.89). Bài này bị REJECTED vì không đạt mốc 2×.

## 5. Bộ khung NPU tối thiểu mà một bài thắng cần có (suy ra từ interface và các con số ở QUEST_001_NPU.md)

```
           s_* (AXI slave) ──► Register file / descriptor FIFO ──► Bộ điều khiển (FSM chạy chuỗi layer)
                                                                        │
  m_* AR ◄── DMA đọc (burst ≤ 256 beat, prefetch) ──► Buffer input/weight (flop/SRAM, vài KB)
                                                                        │
                                          ┌─────────────────────────────┴───────────────┐
                                          │ Mảng MAC int8 (GEMM: conv/pointwise/FC)     │
                                          │ Đường depthwise (theo kênh)                 │
                                          │ Sinh địa chỉ có "pad ảo" (điền zero-point)  │
                                          └─────────────────────────────┬───────────────┘
                                                                        ▼
                         Epilogue: +bias → MultiplyByQuantizedMultiplier(out_mult, out_shift)
                                   → +output_zp → clamp(act_min, act_max) → [ADD residual / avgpool]
                                                                        ▼
  m_* AW/W ◄── DMA ghi (burst) ◄── Buffer output
```
- **Kernel C** chỉ làm việc cấu hình: đọc `Node` / `ConvParams`, ghi descriptor (địa chỉ DRAM của tensor, kích thước, stride, pad, zero-point, con trỏ `out_mult` / `out_shift`) vào `0x3000_0000`, rồi chờ hoặc tiếp tục. Muốn đạt hơn 1 000× thì phải **gộp cả mạng**: kernel đầu tiên đẩy toàn bộ chuỗi descriptor, NPU chạy liên tục, các kernel sau gần như không làm gì. Với các op hiếm (`SOFTMAX`, `MUL`, `CONCAT`, `TANH`…) và các model audit, vẫn phải có đường dự phòng chạy trên CPU cho đúng.
- **Nút thắt cần lưu ý:**
  - Băng thông của cổng B (16 B/chu kỳ, một burst đọc tại một thời điểm).
  - SRAM trên chip rất đắt: macro 1 KB chiếm khoảng 0.19 mm².
  - Tần số tính điểm bị chặn ở 62.89 MHz.
  - Delay ở port bằng 30% chu kỳ, nên phải đăng ký (register) mọi cổng.
  - Requantize phải khớp **từng bit** với `kernels.c`.
