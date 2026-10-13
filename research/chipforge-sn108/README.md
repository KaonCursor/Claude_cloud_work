# Nghiên cứu chuyên sâu: ChipForge — Bittensor Subnet 108

> Ngày nghiên cứu: 13/10/2026  
> Nguồn chính: mã nguồn [TatsuProject/ChipForge_SN108](https://github.com/TatsuProject/ChipForge_SN108) (commit `4b8de64`, 10/10/2026), các repo liên quan trong tổ chức [TatsuProject](https://github.com/TatsuProject), tài liệu *ChipForge – Technical Architecture v1.0* (10/2025, có trong repo), cùng các bài báo/bài PR trên web.  
> **Giới hạn:** môi trường nghiên cứu không truy cập được `chipforge.io`, `api.chipforge.io`, taostats hay RPC của chain. Vì vậy báo cáo **không có số liệu on-chain trực tiếp** (giá alpha, emission, số UID, stake của validator). Thông tin về website lấy gián tiếp qua kết quả tìm kiếm. Các mục có dấu ⚠️ là chưa xác minh được.

---

## 1. Tóm tắt nhanh (TL;DR)

| Hạng mục | Nội dung |
|---|---|
| **Là gì** | Subnet thiết kế phần cứng số (digital/RTL design). Miner nộp mã Verilog/SystemVerilog để giải các "challenge" như CPU RISC-V hoặc bộ tăng tốc AI (NPU). Validator chấm điểm bằng công cụ EDA mã nguồn mở. |
| **Đội ngũ** | Tatsu (Taτsu Ecosystem), một đội validator Bittensor. Trước đó họ đã làm Identity Subnet và Document Understanding Subnet. Code chủ yếu do các tài khoản GitHub `atkinson-ai` (subnet/validator) và `LogicX-Tatsu` (EDA, RTL, verification) viết. |
| **Lịch sử netuid** | Ra mắt với tên **SN84** (khoảng 08/2025). **SN84 bị deregister**: commit "slot 84 removed after deregistration" ngày 05/05/2026. **Đăng ký lại ở netuid 108**: commit ngày 24/09/2026 ghi "Subnet re-registered on netuid 108". |
| **Cơ chế thưởng** | Người thắng lấy hết (winner-takes-all). Người thắng challenge nhận `MINER_EMISSION_PERCENTAGE`, mặc định **10%** trọng số; phần còn lại **burn vào UID 0**. Thời hạn nhận thưởng mặc định 24 giờ và do challenge server quy định. |
| **Mức tập trung** | **Cao.** Challenge server do đội ngũ vận hành quyết định đề bài, batch, baseline, tỷ lệ emission, danh sách ban và việc cấp secret key cho validator. Bộ chấm điểm (evaluator bundle) của mỗi challenge được giữ kín. |
| **Sản phẩm thực tế** | [chipforge-mcu](https://github.com/TatsuProject/chipforge-mcu): lõi RISC-V RV32IMC 5 tầng có các extension mật mã Zk*/Zbb và FPU (RV32F). [rvgen](https://github.com/TatsuProject/rvgen): bộ sinh lệnh RISC-V viết bằng Python thuần. Theo blog của dự án, họ còn đóng góp bản sửa lỗi RTL cho Google CoralNPU. |
| **Challenge hiện tại** | Theo code của EDA server, challenge 15 là "npu-v1.0", ra mắt khoảng 23/09/2026: thiết kế bộ tăng tốc NPU gắn vào SoC và chấm bằng chế độ `fom_realmacro`. |
| **Roadmap đã công bố** | Tapeout trên OpenMPW (shuttle silicon của Google) vào Q1/2026, có chip thật vào Q3/2026. ⚠️ **Không tìm được bằng chứng nào cho thấy hai mốc này đã đạt.** |
| **Rủi ro lớn nhất** | (1) Đã từng bị deregister vì giá alpha thấp. (2) Burn 90% có thể kéo giảm emission của subnet theo công thức emission sau 06/2026 (xem §7.3). (3) Phụ thuộc gần như hoàn toàn vào hạ tầng tập trung của đội ngũ. (4) Roadmap tapeout bị trễ. |

---

## 2. Danh tính, đội ngũ và dòng thời gian

### 2.1. Tatsu / TatsuProject
- Tổ chức GitHub [TatsuProject](https://github.com/TatsuProject) có 9 repo. Ngoài ChipForge còn có `Identity-Subnet` (2024), `Document_Understanding_Subnet` (2025) và `document_insights_base_model`.
- Theo các trang giới thiệu, Tatsu là một dự án AI phi tập trung xây subnet trên Bittensor và có vận hành validator riêng. README của subnet viết: *"Backed by the Tatsu validator team"*.
- Người đóng góp code:
  - `ChipForge_SN108`: 76 commit, phần lớn của `atkinson-ai`; có một PR từ cộng đồng là miner CLI của `SmartDever02`.
  - `chipforge_eda_server`: 42 commit của `LogicX-Tatsu` và 19 commit của `atkinson-ai`.
  - `chipforge-mcu` và `rvgen`: `LogicX-Tatsu`.
- Website: `chipforge.io` (có leaderboard, blog, docs tại `docs.chipforge.io`). API: `api.chipforge.io`. Discord nằm trong server Bittensor, kênh ChipForge.
- Lưu ý trùng tên: `chipforge.ai` (nền tảng sinh RTL bằng AI) và `chisuhua/ChipForge` là **các dự án không liên quan**.

### 2.2. Dòng thời gian (dựng lại từ git log và tin tức)

| Thời điểm | Sự kiện | Nguồn |
|---|---|---|
| ~08/2025 | Tatsu công bố "ChipForge (SN84) — the first decentralized hardware design project" | Bài đăng X của @TatsuEcosystem |
| 09/09/2025 | README v1 được hoàn thiện; validator tải testcase | git log |
| 19/09/2025 | Challenge 0005/0006: lõi RV32I pipeline (mức beginner) | metadata.json trong `chipforge-challenges` |
| 07/10/2025 | Challenge 0007: RV32IMC | như trên |
| 19/10/2025 | Challenge 0008/0009: RV32IMCK (thêm crypto), tối ưu hiệu năng và diện tích | như trên |
| 10/2025 | Công bố tài liệu Technical Architecture v1.0 | `docs/ChipForge - Technical Architecture.pdf` |
| 23/11/2025 | Thêm baseline score trên challenge server và "centralized emergency emissions ban system" | git log |
| 25/11/2025 | Challenge 0010: RV32IMCK tối ưu công suất, tính điểm 1/3 diện tích + 1/3 delay + 1/3 công suất | metadata.json |
| 12/2025 | Loạt bài PR (U.Today, Bitget, Coinpaper, KuCoin…) về "Open Competition Model" và NPU cho edge AI | web |
| 15/01/2026 | Blog: pipeline verification tìm ra nhiều lỗi RTL trong CoralNPU và gửi bản sửa ngược lên upstream | kết quả tìm kiếm về chipforge.io |
| 12/03/2026 | Tỷ lệ emission cho miner được định nghĩa trong code | git log |
| 25/04/2026 | Challenge 013: thêm FPU RV32F vào chipforge-mcu | git log của chipforge-mcu |
| 27/04/2026 | Đồng bộ và thực thi danh sách coldkey bị ban | git log |
| **05/05/2026** | **"slot 84 removed after deregistration"**: NETUID bị để trống | commit `b476630` |
| 18/09/2026 | Nâng cấp lên bittensor 10.5 (SDK classic) và thêm bộ test | PR #2 |
| 23/09/2026 | Ra mắt challenge NPU (npu-v1.0) với cách tính điểm FoM mới | comment trong `chipforge_eda_server/gateway/main.py` |
| **24/09/2026** | **"Subnet re-registered on netuid 108"**: đổi tên repo thành `ChipForge_SN108` | commit `3e8c957` |
| 26–30/09/2026 | Viết lại lớn ("v2", Phase A→G): sửa logic set weights, ký request v2, đóng gói Docker, mở reveal thiết kế thắng cuộc, public API | PR #3 |
| 06–10/10/2026 | `min_improvement_percent` và `baseline_epoch` chuyển sang do server quyết định | git log |

**Nhận xét:** subnet đã mất slot 84 sau khoảng 8–9 tháng hoạt động. Theo docs Bittensor, subnet bị prune khi có alpha price EMA thấp nhất trong nhóm không còn được miễn trừ và số slot đã đầy. Như vậy ở thời điểm đó, thị trường định giá token SN84 thấp nhất mạng. Subnet quay lại ở SN108 với một codebase được làm lại khá kỹ. ⚠️ Đáng chú ý là slot 108 trước đó đã được các dự án khác dùng (một số bài 05/2026 gọi SN108 là "Frontier Compute"). Vì vậy dữ liệu lịch sử của "SN108" trên các trang tổng hợp có thể bị lẫn.

---

## 3. Tầm nhìn sản phẩm

- **Luận điểm:** thiết kế và chế tạo chip đang rất tập trung vào một số ít tập đoàn nắm IP và năng lực fab. ChipForge muốn biến việc thiết kế thành một "thị trường cạnh tranh mở": ai thiết kế tốt nhất theo số đo khách quan (PPA, tức hiệu năng, công suất và diện tích) thì thắng.
- **Trọng tâm:** hardware–software co-design cho **edge AI**, nhất là NPU, để dùng cho IoT, robot, wearable và bảo mật hậu lượng tử.
- **Danh mục challenge:** (1) bộ xử lý RISC-V, (2) bộ tăng tốc AI. README còn nhắc tới module mật mã và mini-GPU.
- **Mô hình doanh thu (theo tuyên bố):** bán IP thiết kế và chip đã chế tạo, rồi tái đầu tư vào hệ sinh thái, kể cả mua lại token (buyback). ⚠️ Chưa thấy bằng chứng nào về doanh thu thực tế.
- **Website (qua kết quả tìm kiếm):** tuyên bố "850+ engineers competing" và giới thiệu "ChipForge NPU — Edge AI Accelerator • 512 GOP/s @ 6mW". ⚠️ Đây là số liệu marketing chưa được kiểm chứng; con số 512 GOP/s @ 6 mW nhiều khả năng là mục tiêu thiết kế chứ chưa phải đo trên silicon.
- **Roadmap trên blog:** OpenMPW Tapeout vào Q1/2026, Real Silicon vào Q3/2026, chip vật lý trước cuối 2026. Một bài báo học thuật (ASEE) ghi rằng Efabless không còn mở shuttle OpenMPW từ MPW-8 (2022). Hiện chương trình shuttle miễn phí của Google chạy qua các kênh khác như IHP/wafer.space/ChipFoundry. Mốc tapeout vì thế phụ thuộc vào kênh shuttle nào đang mở. ⚠️ **Tính đến 10/2026, chưa tìm thấy xác nhận tapeout.**

---

## 4. Kiến trúc kỹ thuật

### 4.1. Các thành phần

```
          ┌──────────────────────────── Đội ngũ ChipForge (tập trung) ───────────────────────────┐
          │  Chip Design Team ──► soạn đề bài + testbench công khai + evaluator bundle KÍN       │
          │                                                                                      │
          │  Challenge Server (api.chipforge.io)                                                 │
          │   • FastAPI + PostgreSQL + S3 (pre-signed URL)                                       │
          │   • quản lý challenge, submission, batch (≤8 bài/batch), baseline, ban list,         │
          │     emission %, reveal, leaderboard, public API                                      │
          └──────────────▲──────────────────────────────────────────▲───────────────────────────┘
                         │ HTTPS (ký sr25519)                        │ HTTPS (ký sr25519 + X-Validator-Secret)
                         │                                           │
   ┌─────────────────────┴──────┐                       ┌────────────┴──────────────────────────────┐
   │ MINER                      │   axon (SimpleMessage)│ VALIDATOR (neurons/validator.py)          │
   │ • miner.py: chỉ nghe thông │◄──────────────────────│ • /validator/sync, tải batch, kiểm sha256  │
   │   báo + tải gói đề         │  "có challenge mới"   │ • gửi tới EDA server, nộp điểm            │
   │ • miner_cli.py: download / │                       │ • chọn winner, set_weights lên chain      │
   │   submit / logs / reveals  │                       └────────────┬──────────────────────────────┘
   │ • Công việc thật: viết RTL │                                    │ HTTP localhost:8080
   │   offline (người + AI)     │                       ┌────────────▼──────────────────────────────┐
   └────────────────────────────┘                       │ EDA SERVER (chipforge_eda_server)          │
                                                        │ gateway ─► verilator-api (mô phỏng)        │
                                                        │         └► openlane-api (tổng hợp sky130)  │
                                                        └───────────────────────────────────────────┘
```

**Điểm quan trọng:** miner **không** phục vụ tính toán qua axon như phần lớn các subnet khác. Axon của miner chỉ nhận thông báo (synapse `SimpleMessage`) và mặc định chỉ trả lời hotkey có validator permit. Bài nộp đi qua **HTTP API tập trung**. Hotkey trên chain chủ yếu dùng để định danh, ký request và nhận emission.

### 4.2. Luồng một vòng
1. Đội ngũ công bố challenge. Miner tải gói đề gồm spec PDF, testbench công khai và testcase cho miner (`miner_cli.py download`).
2. Miner thiết kế RTL, đóng gói ZIP tối đa 50 MB có `rtl.f`, rồi `submit`. Request được ký bằng hotkey và kèm sha256. Server từ chối file trùng từng byte với thiết kế đã thắng.
3. Server gom bài nộp thành **batch**, mỗi batch có cửa sổ tải và cửa sổ chấm. Mỗi lúc chỉ có một batch được mở cho validator. Trạng thái batch: EXPOSED → EVALUATING → COMPLETED.
4. Validator gọi `/validator/sync` (một request có ETag, trả về challenge, batch, ban list và phiên bản testcase), tải bài và kiểm sha256 so với hash miner đã ký.
5. Validator gửi từng bài tới EDA server, tối đa `EDA_MAX_CONCURRENCY` (mặc định 8) bài cùng lúc, trong hạn chót của batch. Mỗi bài có giới hạn 45 phút.
6. Validator nộp điểm lên server gồm functionality, area, delay (perf), power, overall và hai gate.
7. **Validator tự chọn winner từ kết quả chấm của chính nó** (không đồng bộ winner giữa các validator), rồi `set_weights` ngay khi winner thay đổi và làm mới định kỳ mỗi `WEIGHTS_REFRESH_SECONDS` (300 giây).
8. Khi batch đóng, server công bố điểm (điểm của một submission là điểm cao nhất có qua gate mà nó nhận được từ các validator) và cập nhật leaderboard cùng "score to beat".

### 4.3. Bảo mật và toàn vẹn
- Ký request bằng **sr25519**. Chữ ký v2 gắn với `method:path:hotkey:timestamp:nonce:sha256(body)`, có tiền tố domain `chipforge-sn108:v2`, nhờ đó chống replay. Tài liệu PDF ghi "Ed25519" nhưng code thực tế dùng sr25519 của hotkey Bittensor.
- Validator cần thêm `VALIDATOR_SECRET_KEY` do đội ngũ cấp. Đây là một **lớp cấp phép ngoài chain**: có validator permit trên chain mà không có secret thì vẫn không chấm được.
- Bài nộp có hash, file thiết kế thắng được ký hash khi reveal, và download bị giới hạn tốc độ.
- EDA server **thực thi `run.py` trong evaluator ZIP** mà caller gửi lên. Ai truy cập được cổng 8080 là có thể chạy code trên máy đó, nên cổng mặc định chỉ bind `127.0.0.1`. Validator phải tin evaluator bundle do server phát.
- "Emergency emissions ban": server có thể bật `ban_emissions` để mọi validator chạy code chuẩn burn 100%, và có thể ban coldkey vĩnh viễn hoặc theo từng challenge.

### 4.4. Yêu cầu phần cứng cho validator
| | Nhân vật lý | RAM | Ổ đĩa |
|---|---|---|---|
| Khuyến nghị | 24 | 48 GB | 100 GB SSD |
| Tối thiểu | 16 | 32 GB | 100 GB SSD |

- Chỉ chạy trên x86-64; image EDA không chạy được trên ARM. Một lần tổng hợp (OpenLane) chạy đơn luồng và cần tới khoảng 10 GB RAM.
- Số đo của dự án: trên cấu hình tối thiểu, 8 bài NPU chạy song song xong trong khoảng 22 phút.
- Không cần GPU. Chi phí tương đương một máy cloud cỡ `c7a.4xlarge`.

---

## 5. Cơ chế chấm điểm

### 5.1. Pipeline EDA
1. **Functional (Verilator):** chạy chương trình test (sinh bằng rvgen hoặc riscv-dv) trên RTL của miner, ghi trace RVFI rồi so **từng lệnh** với golden trace của trình mô phỏng ISA **Spike**. `func = số lệnh khớp / tổng số lệnh`. Có thêm một bước kiểm tra chống gian lận: nếu RTL khi mô phỏng khác với khi tổng hợp (ví dụ dùng `` `ifdef verilator `` để giấu logic), mọi điểm bị gán −1.
2. **Physical (Yosys/OpenLane, PDK sky130):** đo diện tích cell (µm²), fmax và WNS. Công suất mới được đưa vào từ challenge 0010.
3. **Performance:** `IPS = IPC × fmax`, chuẩn hóa theo IPS mục tiêu.

### 5.2. Công thức "cổ điển" (challenge CPU 0005–0013)
```
area_c  = min(area_target / area,  ratio_cap)      # nhỏ hơn thì tốt hơn
perf_c  = min(IPS / IPS_target,    ratio_cap)      # ratio_cap mặc định 2.0
power_c = min(power_target / P,    ratio_cap)
overall = Σ wᵢ·cᵢ / Σ wᵢ   (×100)
functional_gate = func ≥ func_threshold
overall_gate    = overall ≥ overall_threshold
```
Trọng số và ngưỡng nằm trong `weights.json` của evaluator bundle (kín). Lịch sử các challenge (lấy từ metadata):

| Challenge | Đề | Trọng số F/A/D/P | Gate chức năng | Mục tiêu |
|---|---|---|---|---|
| 0005, 0006 | RV32I pipeline | 0.5 / 0.25 / 0.25 / 0 | ≥ 60% | 300 000 µm², 100 MIPS |
| 0007 | RV32IMC | 0.5 / 0.25 / 0.25 / 0 | 100% | 200 000 µm², 100 MIPS |
| 0008, 0009 | RV32IMCK (crypto) | 0.5 / 0.25 / 0.25 / 0 | ≥ 80% | 200 000 µm², 100 MIPS |
| 0010 | RV32IMCK tối ưu công suất | 0 / ⅓ / ⅓ / ⅓ | 100% bắt buộc | ~50 mW (< 100 mW); overall ≥ 75 |
| 013 | Thêm RV32F (FPU) | ⚠️ không có metadata công khai | | |

### 5.3. Công thức FoM cho NPU (challenge 15, `scoring_mode = fom_realmacro`)
- **Gate 1:** chức năng đúng 100% (khớp từng bit).
- **Gate 2:** speedup so với baseline không có tăng tốc phải ≥ `min_speedup` (mặc định 2×). Speedup là trung bình nhân (geomean) của speedup theo từng model nhân với tỷ lệ fmax.
- **Điểm:** `FoM = (IPS_gain^α) / (max(area, area_floor)^β) × 1000`, với α = 0.7, β = 0.3 và `IPS_gain = IPS × (1 − 1/speedup)`.
  - Chỉ phần **thông lượng mà bộ tăng tốc thêm vào** mới được tính. Một NPU "rỗng" vì vậy được 0 điểm. Trong code có ghi chú: trước khi sửa, một stub trơ được 90.8 điểm, còn một accelerator thật 16×16 chỉ được 15.0.
  - `area_floor` (quyết định ngày 26/09/2026) ngăn kiểu gian lận "tăng tốc bằng phần mềm, gần như không tốn diện tích".
- **Nhận xét:** đội ngũ đã chủ động vá các lỗ hổng làm lệch thang điểm (reward hacking) ngay trong công thức. Đây là dấu hiệu tốt về năng lực kỹ thuật. Mặt khác, công thức thay đổi theo từng challenge và evaluator được giữ kín, nên miner khó tự kiểm chứng trước khi nộp.

---

## 6. Cơ chế phần thưởng và trọng số (đọc từ code)

### 6.1. Chọn người thắng (`batch_processor.py`)
Một bài nộp chỉ trở thành **champion** khi thỏa cả bốn điều kiện:
1. `functional_gate` **và** `overall_gate` đều đúng;
2. `overall > challenge_best × (1 + m%)`;
3. `overall > baseline_snapshot × (1 + m%)`, trong đó baseline là "score to beat" lấy **trước** khi nộp điểm;
4. `m` = `min_improvement_percent` do server gửi. Giá trị này bằng 0 cho tới khi challenge có người thắng đầu tiên, sau đó lấy theo cấu hình. `.env.example` để fallback 20%.

Nếu hai bài bằng điểm, bài nộp sớm hơn thắng. Muốn thay người đang dẫn thì phải vượt họ một biên độ đáng kể; nhờ vậy khó "xén" điểm bằng cách sửa vài chi tiết nhỏ.

### 6.2. Chính sách emission (`weight_manager.py`, `emission_manager.py`)
```
Có winner hợp lệ  → weights = { UID0: 1 − p,  winner: p }      p = MINER_EMISSION_PERCENTAGE (mặc định 10%)
Không có winner   → weights = { UID0: 1.0 }                      (burn 100%)
Bị ban / không đăng ký / ban_emissions → burn
```
- `p` **do challenge server quyết định** nếu server gửi giá trị; khi đó validator ghi log "local value is not used".
- Cửa sổ thưởng là `winner_reward_hours` do server gửi, tối đa 24 giờ trong `.env.example` (trước đây là 48 giờ). Nếu chính người thắng cải thiện điểm thì đồng hồ được đặt lại. Hết cửa sổ mà không có ai thắng mới thì burn.
- Các pha: `initial_burn` → `normal` / `winner_reward` → `burn_until_submission`.
- `baseline_epoch`: khi server hủy một kỷ lục (do exploit hoặc gian lận) hoặc admin đổi score to beat, mọi validator reset winner cục bộ về đúng winner của server. Đây là một **công tắc trung tâm** có thể thay đổi việc ai được trả thưởng.

### 6.3. Ý nghĩa
- Tại mỗi thời điểm, **chỉ một miner** nhận incentive. Phần lớn emission phía miner đi vào UID 0: với 10%, khoảng 90% bị burn khi có winner và 100% khi không có. Về bản chất đây là mô hình **"bounty/cuộc thi"**, không phải "trả lương liên tục".
- Validator không cần đồng thuận về winner. Tuy vậy, tất cả đều chấm cùng evaluator, cùng baseline và cùng margin từ server, nên trên thực tế kết quả của họ thường hội tụ. Yuma Consensus vẫn phạt validator lệch khỏi số đông.
- README tự nhận: *"The chain decides who is actually paid (highest incentive); the leaderboard shows it and which validators agree."*

---

## 7. Phân tích kinh tế và rủi ro

### 7.1. Điểm mạnh
1. **Bài toán có giá trị và đo được khách quan.** PPA là thước đo chuẩn của ngành bán dẫn, so với golden model Spike rất khó bị gian lận về chức năng, và đây là ngách gần như chưa subnet nào làm.
2. **Có sản phẩm thật, mã nguồn mở.** chipforge-mcu (RV32IMC + Zk* + Zbb + F, 5 tầng) đạt 100% khớp Spike trên 700 lần chạy với khoảng 3.76 triệu lệnh. rvgen có hơn 1 000 test và khớp Spike 213/213. Theo blog, họ còn gửi bản sửa lỗi cho CoralNPU của Google.
3. **Kỹ thuật validator tốt lên rõ ở bản v2:** sửa logic set weights, state chịu được crash, kiểm hash, chữ ký v2 chống replay, Docker kèm healthcheck, bộ test pytest khoảng 16 file, public API và leaderboard minh bạch, reveal thiết kế thắng cuộc.
4. **Chống reward hacking chủ động:** sentinel `ifdef`, area floor, tính trên IPS gain, từ chối bản sao, yêu cầu cải thiện tối thiểu.
5. Rào cản cho validator vừa phải: chỉ cần CPU và RAM, không cần GPU.

### 7.2. Điểm yếu và rủi ro
1. **Tập trung hóa cao.** Server của đội ngũ nắm đề bài, evaluator kín, batch, baseline, tỷ lệ emission, ban list, `baseline_epoch` và secret key của validator. Validator thực chất là "người chạy EDA được cấp phép"; nếu `api.chipforge.io` ngừng hoạt động thì subnet dừng. Đây là kiểu mô hình "owner-run competition" khá phổ biến nhưng bị cộng đồng Bittensor đánh giá thấp về độ phi tập trung.
2. **Đã từng bị deregister.** SN84 bị prune vào khoảng 04–05/2026. Theo luật pruning dựa trên giá, điều đó nghĩa là alpha SN84 từng có giá EMA thấp nhất trong nhóm subnet không còn miễn trừ. Holder alpha SN84 khi đó đã bị chuyển về TAO theo giá thanh lý. SN108 đang trong thời gian miễn trừ (docs ghi khoảng 4–6 tháng tùy phiên bản), nên lại có thể bị prune nếu giá không cải thiện.
3. **Burn cao có thể kéo giảm emission của chính subnet.** ⚠️ Taostats ghi rằng từ 30/06/2026, emission của subnet ∝ *giá EMA × (1 − miner_burned)* (subtensor PR #2800). Nếu cách hiểu này đúng, mức burn 90–100% mặc định sẽ làm phần emission của SN108 giảm mạnh so với các subnet có cùng giá. Cần kiểm tra lại trên chain xem server đang đặt `miner_emission_percentage` là bao nhiêu (có thể đã khác 10%) và taostats hiển thị "miner burned" cho SN108 thế nào.
4. **Winner-takes-all và cửa sổ thưởng ngắn (≤ 24 giờ).** Thu nhập của miner biến động rất mạnh. Với miner trung bình, kỳ vọng thu nhập gần 0 trừ khi có năng lực RTL thật sự mạnh. Các bài báo PR nói "850+ engineers", nhưng số người từng thắng có lẽ rất ít. ⚠️ Không lấy được leaderboard để kiểm.
5. **Roadmap trễ.** Mốc tapeout Q1/2026 và chip thật Q3/2026 chưa được xác nhận. Doanh thu IP/chip và buyback hiện vẫn chỉ là tuyên bố.
6. **IP và pháp lý.** Thiết kế thắng cuộc được reveal cho mọi hotkey đã đăng ký. Chưa thấy điều khoản rõ ràng về quyền sở hữu IP, giấy phép của bài nộp và cách chia doanh thu nếu IP được thương mại hóa.
7. **Rủi ro vận hành cho validator.** EDA server chạy code từ evaluator bundle và chạy công cụ trên RTL của miner, nên phải để cổng 8080 đóng với internet. Giới hạn 45 phút mỗi bài cùng hạn chót của batch có thể khiến bài bị chấm timeout nếu máy yếu.
8. **Không nhất quán tài liệu ở mức nhỏ:** PDF ghi Ed25519 nhưng code dùng sr25519; README liệt kê Icarus/Yosys, còn EDA server thực chất chỉ dùng Verilator và OpenLane.

### 7.3. Checklist cần kiểm thêm trên chain / taostats (chưa làm được do bị chặn mạng)
- [ ] Ngày đăng ký SN108 và thời điểm hết miễn trừ.
- [ ] Giá alpha SN108, giá EMA, thứ hạng giá EMA so với nhóm có nguy cơ bị prune.
- [ ] Emission/ngày, tỷ lệ "miner burned" hiển thị trên taostats.
- [ ] Số validator có permit, tổng stake, tỷ lệ stake của Tatsu so với các validator lớn khác.
- [ ] Dữ liệu từ `api.chipforge.io/api/v1/public/snapshot`, `/validators`, `/rules`: challenge đang chạy, score to beat, winner hiện tại, `miner_emission_percentage` thực tế.
- [ ] Phân bố coldkey của những người từng thắng, để xem có dấu hiệu tập trung hay "người nhà" hay không.

---

## 8. Gợi ý theo vai trò

### Miner
- Cần kỹ năng **thiết kế RTL và verification thật sự** (pipeline RISC-V, tối ưu PPA, hiểu OpenLane/sky130). Dùng AI hỗ trợ được nhưng không thay được kỹ năng.
- Nên tự dựng `chipforge_eda_server` ở máy local để chấm thử trước khi nộp. Chỉ có 5 lần nộp mỗi challenge, và giới hạn 2 request/phút, 5 request/giờ cho mỗi IP.
- Học từ thiết kế thắng cuộc đã reveal (`miner_cli.py reveals` / `reveal-download`), chipforge-mcu và `Solution1/2.zip` trong `chipforge-challenges`.
- Với challenge NPU: chú ý gate speedup ≥ 2×, `area_floor` và việc điểm chỉ tính IPS gain. Phần mềm tối ưu kernel không giúp được.
- Chi phí đăng ký UID và rủi ro bị đẩy khỏi subnet: miner không tạo trọng số thì có thể bị deregister khỏi slot UID khi có người đăng ký mới.

### Validator
- Phần cứng 16–24 nhân vật lý, 32–48 GB RAM, x86. Cần **stake đủ để có permit** và **xin secret key từ đội ngũ**.
- Lợi nhuận chủ yếu đến từ dividend theo stake, có điều chỉnh theo mức đồng thuận với các validator khác (vtrust). Nếu chọn winner lệch số đông thì bị thiệt. Đánh giá rủi ro alpha trước khi stake lớn.

### Staker / nhà đầu tư alpha
- Đây là luận điểm **"DeSci/DePIN cho bán dẫn"** có câu chuyện tốt và sản phẩm mã nguồn mở thật. Tuy vậy, cần cân với lịch sử bị deregister, mức tập trung cao, roadmap trễ và rủi ro burn làm giảm emission (§7.2 mục 3).
- Các mốc nên theo dõi: (1) xác nhận tapeout hoặc chip thật; (2) có doanh thu hoặc khách hàng IP đầu tiên; (3) số người thắng khác nhau theo thời gian; (4) giá EMA khi gần hết miễn trừ.

---

## 9. Các repo liên quan

| Repo | Vai trò | Cập nhật gần nhất |
|---|---|---|
| [ChipForge_SN108](https://github.com/TatsuProject/ChipForge_SN108) | Code miner, validator và CLI (Python, bittensor 10.5) | 10/10/2026 |
| [chipforge_eda_server](https://github.com/TatsuProject/chipforge_eda_server) | Gateway + Verilator API + OpenLane API (Docker) | 01/10/2026 |
| [chipforge-mcu](https://github.com/TatsuProject/chipforge-mcu) | Lõi RISC-V "do cộng đồng thiết kế" (sản phẩm của các challenge) | 25/04/2026 |
| [rvgen](https://github.com/TatsuProject/rvgen) | Bộ sinh lệnh RISC-V viết bằng Python thuần, có coverage (Apache-2.0, có trên PyPI) | 14/06/2026 |
| [chipforge-challenges](https://github.com/TatsuProject/chipforge-challenges) | Lưu trữ đề và lời giải mẫu tới challenge 0010 | 28/11/2025 |
| `coralnpu` (fork) | Fork của google-coral/coralnpu, nền cho các challenge NPU | 19/02/2026 |

---

## 10. Nguồn tham khảo

**Mã nguồn và tài liệu gốc (đã đọc trực tiếp):**
- https://github.com/TatsuProject/ChipForge_SN108 (README, `.env.example`, `neurons/`, `docs/validator_requirements.md`, `docs/ChipForge - Technical Architecture.pdf`, git log)
- https://github.com/TatsuProject/chipforge_eda_server (`gateway/main.py`: hàm `compute_weighted_score`)
- https://github.com/TatsuProject/chipforge-mcu
- https://github.com/TatsuProject/chipforge-challenges
- https://github.com/TatsuProject/rvgen
- https://github.com/TatsuProject/ChipForge_SN108/pull/2, https://github.com/TatsuProject/ChipForge_SN108/pull/3

**Web (qua kết quả tìm kiếm; không mở trực tiếp được):**
- https://www.chipforge.io/, https://www.chipforge.io/blog/introduction-to-chipforge, https://docs.chipforge.io/
- https://x.com/TatsuEcosystem/status/1958898282316668956
- https://u.today/chipforge-introduces-new-generation-of-hardware-for-ai-with-its-open-competition-model
- https://www.bitget.com/news/detail/12560605116450
- https://coinpaper.com/13297/chip-forge-the-bittensor-subnet-where-better-chips-win-literally
- https://www.kucoin.com/news/flash/chipforge-aims-to-revolutionize-edge-ai-chip-development-with-decentralized-design-model
- https://docs.learnbittensor.org/subnets/subnet-deregistration
- https://docs.learnbittensor.org/learn/emissions, https://docs2.taostats.io/concepts/tao-emission/
- https://taostats.io/subnets/84/registration
- https://cryptonews.net/news/analytics/32892554/ (về một dự án khác từng dùng slot SN108)
- https://nemo.asee.org/public/conferences/365/papers/47365/download (bối cảnh shuttle OpenMPW)
