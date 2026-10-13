# ChipForge SN108 — Hướng dẫn chuyên sâu cho Miner

> Cập nhật: 13/10/2026. Tài liệu viết dựa trên việc đọc trực tiếp mã nguồn của [ChipForge_SN108](https://github.com/TatsuProject/ChipForge_SN108) (`python_scripts/miner_cli.py`, `neurons/miner.py`, `neurons/validator_utils/*`), [chipforge_eda_server](https://github.com/TatsuProject/chipforge_eda_server) (`gateway/`, `verilator-api/`, `openlane-api/`) và gói đề/testcase mẫu trong [chipforge-challenges](https://github.com/TatsuProject/chipforge-challenges).
> Tài liệu tổng quan về subnet nằm ở [README.md](README.md).
> ⚠️ = chưa xác minh được vì môi trường nghiên cứu không truy cập được `api.chipforge.io` và chain.

---

## 0. Đọc trước khi bỏ tiền đăng ký UID

Miner ở SN108 **không giống** miner ở phần lớn các subnet khác:

| Subnet thông thường | ChipForge |
|---|---|
| Chạy model/GPU 24/7, validator gọi tới axon của bạn | **Không cần GPU, không phục vụ request.** Bạn tự thiết kế chip (RTL) offline rồi nộp file ZIP qua HTTP API |
| Nhiều miner cùng chia emission theo điểm | **Chỉ một người thắng** lấy incentive, trong một cửa sổ thời gian ngắn |
| Lợi thế đến từ phần cứng và vốn | Lợi thế đến từ **kỹ năng thiết kế vi kiến trúc + verification + tối ưu PPA** (hiệu năng, công suất, diện tích) |

**Thu nhập thực tế khi thắng** (đọc từ `weight_manager.py` và `emission_manager.py`):
- Validator đặt trọng số: `winner = p`, `UID 0 (burn) = 1 − p`, với `p = MINER_EMISSION_PERCENTAGE`. Giá trị mặc định là **10%**, và **challenge server có quyền đặt giá trị khác** ⚠️ (xem bằng `miner_cli.py status` hoặc trang leaderboard).
- Bạn chỉ được trả trong `winner_reward_hours`, **tối đa 24 giờ** trong cấu hình hiện tại, tính từ lúc bạn được công nhận thắng. Hết giờ mà không có ai vượt thì toàn bộ bị burn. Nếu bạn tự vượt điểm của chính mình thì đồng hồ được đặt lại.
- Ước lượng nhanh: `thu nhập ≈ (emission alpha/ngày của SN108 × 41% phần miner) × p × (số giờ thắng / 24)`.
  Ví dụ minh họa (số giả định): subnet nhận 300 α/ngày, p = 10%, thắng trọn 24 giờ thì được khoảng 300 × 0.41 × 0.10 ≈ **12 α**. Hãy thay bằng số thật lấy từ taostats.
- **Kết luận:** chỉ đáng làm nếu bạn thực sự có năng lực RTL hoặc muốn có portfolio về thiết kế chip. Nếu chỉ nộp bài trung bình thì thu nhập kỳ vọng gần như bằng 0.

**Rủi ro mất UID:** sau khi hết immunity period, UID có incentive thấp nhất sẽ bị đẩy ra khi có người đăng ký mới. Hầu hết miner ở đây có incentive bằng 0, nên **bạn có thể bị deregister trước khi kịp thắng**. Nhiều khả năng bạn chỉ cần hotkey **còn đăng ký vào đúng lúc** submit và lúc validator đặt trọng số ⚠️. Vì vậy chiến lược hợp lý là chuẩn bị thiết kế thật kỹ ở local trước, rồi mới đăng ký và nộp.
Kiểm tra trước khi đăng ký:
```bash
btcli subnets burn-cost --netuid 108 --network finney        # phí đăng ký hiện tại
btcli subnets hyperparameters --netuid 108 --network finney  # immunity_period, max_allowed_uids...
```

---

## 1. Chuẩn bị môi trường

### 1.1. Máy cần có
| Việc | Cấu hình |
|---|---|
| Chỉ chạy `miner.py` + CLI | Máy nhẹ bất kỳ (VPS 1–2 vCPU), cần mở cổng `AXON_PORT` (8091) ra internet nếu chạy axon |
| **Chạy EDA local để tự chấm (rất nên làm)** | x86-64 Linux, Docker, **≥ 16 GB RAM** (mỗi lần synthesis giữ khoảng 6 GB), ≥ 25 GB đĩa (image khoảng 9 GB). Với challenge NPU: 16 nhân trở lên; một lần chạy thiết kế tham chiếu 16×16 mất khoảng 18 phút trên laptop 10 nhân |
| Không chạy được | ARM (Apple Silicon, Graviton): image EDA không hỗ trợ |

### 1.2. Cài đặt
```bash
# Ví + đăng ký
btcli wallet create --wallet-name mywallet --hotkey default
btcli subnets register --netuid 108 --network finney --wallet-name mywallet --hotkey default

# Code subnet
git clone https://github.com/TatsuProject/ChipForge_SN108 && cd ChipForge_SN108
cp .env.example .env
#   sửa trong .env: MINER_WALLET_NAME=mywallet, MINER_HOTKEY=default, MINER_WALLET_DIR=~/.bittensor/wallets
make up ROLE=miner          # Docker; hoặc: conda env Python 3.12 + pip install -r requirements.txt && pip install -e . && ./start_miner.sh
make logs ROLE=miner
```
- Process `miner.py` **không bắt buộc** để được thưởng. Nó chỉ làm hai việc: nhận thông báo "có challenge mới" từ validator (synapse `SimpleMessage`) và tự tải gói đề về `downloaded_active_challenge/<id>/`, đồng thời poll server mỗi 300 giây. Thứ quyết định việc bạn được trả thưởng là **hotkey đã đăng ký và bài nộp thắng**.
- Axon chỉ trả lời hotkey có validator permit (`MINER_REQUIRE_VALIDATOR_PERMIT=true`), nên không lo bị spam.

### 1.3. Dựng EDA server ở máy local (vũ khí chính của miner)
```bash
git clone https://github.com/TatsuProject/chipforge_eda_server && cd chipforge_eda_server
cp .env.example .env
make start          # build xong thì có gateway :8080 + verilator-api + openlane-api
make health         # {"status": "ok"}
make test           # chạy thử với test/adder.zip
```
Đây **đúng là phần mềm validator dùng** để chấm bạn: cùng Verilator đã pin phiên bản và cùng image OpenLane đã pin theo digest. Kết quả synthesis có tính tất định; code ghi nhận đã tái lập được `4985134.034002 µm² / 57.94 MHz` tới chữ số cuối trên máy khác. Vì vậy **area và fmax đo ở local sẽ trùng với validator**. Phần có thể khác là functionality, vì validator chạy thêm test ẩn.

---

## 2. Quy trình một challenge, từng bước

```bash
# 1. Xem challenge đang chạy, thời gian còn lại, "score to beat", các bài bạn đã nộp
python3 python_scripts/miner_cli.py status

# 2. Tải gói đề (spec PDF, testbench, testcase cho miner) + challenge_<id>_info.json
python3 python_scripts/miner_cli.py download            # -> ./challenges/<challenge_id>/

# 3. Học từ người thắng trước (nếu subnet đang bật reveal)
python3 python_scripts/miner_cli.py reveals
python3 python_scripts/miner_cli.py reveal-download <submission_id>   # -> ./revealed_designs/<id>.zip

# 4. Thiết kế, rồi tự chấm ở local (xem §4) cho tới khi điểm vượt mục tiêu một khoảng an toàn

# 5. Kiểm tra file mà không nộp
python3 python_scripts/miner_cli.py submit solution.zip --dry_run

# 6. Nộp
python3 python_scripts/miner_cli.py submit solution.zip --check_status
#   hoặc: điền FILE_TO_SUBMIT trong .env rồi ./submit_solution.sh ; Docker: make submit FILE=solution.zip

# 7. Theo dõi: bài nằm ở trạng thái "In batch" cho tới khi batch đóng
python3 python_scripts/miner_cli.py submissions
#   leaderboard: https://chipforge.io/leaderboard

# 8. Sau khi batch đóng: lấy log chấm đầy đủ của từng validator (chỉ chủ bài nộp xem được)
python3 python_scripts/miner_cli.py logs <submission_id>              # -> ./evaluation_logs/<id>/
```

Cơ chế nộp (trong `miner_cli.py`):
1. `POST /challenges/{id}/generate-submission-id` trả về `submission_id` và `challenge_nonce`.
2. Ký sr25519 bằng hotkey lên chuỗi `challenge_id + nonce + sha256(file) + submission_id + unix_ts`.
3. Upload multipart. Đồng hồ máy phải lệch **dưới 10 phút** so với server, nếu không sẽ gặp "Signature verification failed".

### Giới hạn cứng
| Giới hạn | Giá trị |
|---|---|
| Số lần nộp mỗi hotkey cho một challenge | **5** (cấu hình theo từng challenge) |
| Rate limit submit/status/history | 2 lần/phút, 5 lần/giờ cho mỗi IP |
| Rate limit logs và reveal-download | 10 lần/phút, 60 lần/giờ cho mỗi IP, cộng thêm hạn mức theo ngày cho mỗi hotkey khi tải reveal |
| Kích thước ZIP | 50 MB ở challenge server (EDA gateway cho phép 300 MB nén / 4 GB giải nén, nhưng server chặn ở 50 MB) |
| Thời gian chấm mỗi bài | **45 phút** (`EVAL_TIMEOUT_S=2700`). Quá thời gian là `EVALUATION_TIMEOUT`, **tính là lỗi của miner, không được chấm lại** |
| Bản sao | File **trùng từng byte** với thiết kế đã thắng sẽ bị từ chối |

> 5 lần nộp là rất ít. **Đừng bao giờ dùng lần nộp chính thức để "thử xem"**; mọi thử nghiệm phải làm ở EDA local.

---

## 3. Định dạng bài nộp (sai là mất lượt)

Cấu trúc chuẩn, theo `Solution1.zip` của challenge 0010 và `verilator/run.py`:
```
solution.zip
├── rtl.f                 ← BẮT BUỘC, nằm ở GỐC zip
└── rtl/
    ├── lib.sv
    ├── ...
    └── rv32imc_top.sv    ← top module đúng tên đề yêu cầu
```
Nội dung `rtl.f` là danh sách file **theo thứ tự biên dịch**, đường dẫn tương đối so với gốc zip:
```
rtl/lib.sv
rtl/alu.sv
...
rtl/rv32imc_top.sv
```

Các quy tắc lấy từ code của evaluator:
- **Tên top module và tên port do evaluator quyết định.** Ví dụ ở challenge 0010: `DESIGN_NAME = rv32imc_top`, clock là `clk_i`, reset là `resetn_i` (reset tích cực mức thấp), và `verilator/top_module.txt` là top của testbench. Đọc kỹ spec PDF và `config.json` / `top_module.txt` trong gói testcase.
- **File SDC của bạn bị bỏ qua.** `openlane/run.py` chép `constraints.sdc` **của evaluator** đè lên. Chu kỳ clock đã cố định (challenge 0010 dùng 10 ns, tức 100 MHz) và bạn không đổi được.
- Mọi file trong `rtl.f` đều được đưa vào cả Verilator lẫn Yosys/OpenLane, nên **phải synthesizable**. Đề ghi `synthesizable_verilog_2005`, trên thực tế SystemVerilog vẫn được chấp nhận.
- Không được có đường dẫn tuyệt đối hay `..`, không để zip rỗng (gateway trả `SUBMISSION_UNSAFE_PATH` / `SUBMISSION_EMPTY`).
- **Không dùng `` `ifdef VERILATOR `` hay bất kỳ cách nào làm RTL khi mô phỏng khác RTL khi tổng hợp.** Evaluator có kiểm tra; nếu phát hiện thì mọi điểm thành **−1** và bài bị loại.
- Đóng gói từ bên trong thư mục để `rtl.f` nằm ở gốc:
  ```bash
  cd my_design && zip -r ../solution.zip rtl.f rtl/ && cd ..
  unzip -l solution.zip | head      # kiểm tra: rtl.f phải ở dòng đầu, không có thư mục cha
  ```

---

## 4. Bạn được chấm như thế nào, và cách tự chấm y hệt

### 4.1. Pipeline
```
design.zip ─┬─► verilator-api : biên dịch rtl.f + testbench → chạy N chương trình test
            │      → trace RVFI so từng lệnh với trace của Spike
            │      → functionality = số lệnh khớp / tổng số lệnh ; IPC = tổng lệnh / tổng chu kỳ
            └─► openlane-api : CHỈ synthesis + STA (không floorplan/place/route), PDK sky130
                   → area_um2   = "Chip area for module" của Yosys
                   → fmax_MHz   = 1000 / (chu_kỳ_SDC − WNS)
                   → power_mW   = tổng công suất trong báo cáo STA
gateway: IPS = IPC × fmax ; tính điểm theo gateway/weights.json
```
OpenLane chỉ được chạy khi trọng số `area` hoặc `performance` lớn hơn 0.

### 4.2. Công thức điểm (các challenge CPU)
```
area_c  = min(area_target / area_um2, 2.0)
perf_c  = min(IPS / perf_target_ips,  2.0)
power_c = min(power_target / power_mw, 2.0)
overall = 100 × (w_f·func + w_a·area_c + w_p·perf_c + w_pw·power_c) / Σw
functional_gate = func ≥ func_threshold      overall_gate = overall ≥ overall_threshold
```
Ví dụ thật, `weights.json` của challenge 0010:
```json
"weights": {"functionality": 0.0, "area": 0.33, "performance": 0.33, "power": 0.34},
"targets": {"func_threshold": 1.0, "overall_threshold": 0.75, "ratio_cap": 2.0,
            "area_target_um2": 200000, "perf_target_ips": 1e8, "power_target_mw": 100}
```
- Functionality có trọng số 0 nhưng **phải đạt 100%** (gate), nếu không thì bị loại.
- Có `ratio_cap = 2.0`: mỗi chỉ số tối đa được tính gấp đôi mục tiêu. Ví dụ diện tích 80 000 µm² chỉ được tính như 100 000 µm². **Vượt quá 2× ở một chỉ số là phí công**, hãy dồn công sức sang chỉ số chưa chạm trần.
- Ví dụ: area 150 000 µm² (1.33), IPC 0.8 × fmax 140 MHz = 112 MIPS (1.12), power 60 mW (1.67) thì `overall ≈ 0.33·1.33 + 0.33·1.12 + 0.34·1.67 ≈ 1.37`, tức **137 điểm**, qua ngưỡng 75.
- Lưu ý: tài liệu cũ ghi điểm thang 0–1, nhưng hiện tại điểm có thể vượt 100 (tối đa 200) do ratio_cap.

### 4.3. Tự chấm ở local bằng chính bundle miner được phát
Gói testcase cho miner (ví dụ `rv32imck_miner_testcases.zip` của challenge 0010, khoảng 264 MB, 2 155 file) **có cùng cấu trúc với evaluator bundle**: `gateway/weights.json`, `openlane/{config.json, constraints.sdc, flow.tcl, run.py}`, `verilator/{run.py, tb_top.sv, tracer.sv, tests/…/spike_trace_*.csv}`. Bạn có thể đưa nó thẳng vào gateway:
```bash
curl -s -X POST http://localhost:8080/evaluate \
  -F "design_zip=@solution.zip" \
  -F "evaluator_zip=@rv32imck_miner_testcases.zip" \
  -F "submission_id=local_try_01" | tee result.json | python3 -m json.tool | head -80
```
Các trường cần đọc trong kết quả: `final_score.{overall, functional_gate, overall_gate, func_score, area_score, perf_score, power_score}`, `verilator_results.results.details.ipc`, `openlane_results.results.{area_um2, fmax_mhz, wns_ns, power_mw}`, và `error.{code, stage, fault}` nếu có lỗi.

**Validator chấm khác local ở đâu:**
- Validator có **bộ test ẩn** (tài liệu CLI ghi "Hidden test cases stay with the validators"). Đạt 100% ở local là điều kiện cần, **chưa phải điều kiện đủ**. Phải kiểm tra thêm (xem §5.4).
- Area, fmax và power là tất định, nên sẽ khớp nếu validator dùng cùng phiên bản bundle và image.
- IPC có thể lệch đôi chút vì tính trên tập chương trình khác.
- Điểm chính thức của bài là **điểm cao nhất có qua gate** mà bài nhận được từ các validator.

### 4.4. Mã lỗi thường gặp (`error.code`, `fault`)
| Code | fault | Ý nghĩa và cách xử lý |
|---|---|---|
| `SUBMISSION_NOT_A_ZIP` / `_EMPTY` / `_UNSAFE_PATH` / `_TOO_LARGE` | miner | Đóng gói lại cho đúng §3 |
| `EVALUATION_TIMEOUT` | miner | Mô phỏng hoặc synthesis vượt 45 phút. Thường do thiết kế treo (deadlock, không bao giờ retire lệnh), mạch tổ hợp quá lớn, hoặc vòng lặp sinh khổng lồ. **Không được chấm lại** |
| Lỗi biên dịch Verilator, functional_gate = false | miner | Đọc `verilator_build_log` / `diff_*.log` để tìm lệnh đầu tiên lệch với Spike |
| `INTERNAL_ERROR`, `SERVICE_UNAVAILABLE`, `EVALUATOR_BUNDLE_MALFORMED` | system | Lỗi phía validator. Validator **không được** chấm bạn 0 điểm; bài sẽ được chấm lại |

---

## 5. Chiến lược để thắng

### 5.1. Hiểu luật "vượt ngưỡng" (đây là mấu chốt)
Theo `batch_processor.py`, một bài chỉ thành winner khi:
1. Qua **cả hai gate**;
2. `overall > best × (1 + m%)` **và** `overall > baseline × (1 + m%)`;
3. `m = min_improvement_percent` do server gửi: **bằng 0 cho tới khi challenge có người thắng đầu tiên**, sau đó là giá trị cấu hình (`.env.example` để fallback 20%) ⚠️.
4. Nếu bằng điểm, **bài nộp sớm hơn thắng**.

Hệ quả chiến thuật:
- **Vào sớm có lợi rất lớn.** Trước khi có winner đầu tiên, chỉ cần qua gate và vượt baseline (score to beat ban đầu do đội ngũ đặt) là thắng, không cần biên độ.
- Sau đó mỗi lần soán ngôi cần **hơn khoảng 20%** (nếu m = 20). Tinh chỉnh vài phần trăm là **vô ích**; cần một cải tiến vi kiến trúc thật sự. Khi đã soán ngôi, bạn cũng được bảo vệ khỏi các bản sửa nhỏ của người khác.
- Có thể **giữ lại một phiên bản tốt hơn**: nếu bạn đang là winner, tự vượt điểm của chính mình sẽ đặt lại đồng hồ thưởng (`update_winner` khi cùng hotkey, điểm cao hơn). ⚠️ Biên độ m có thể vẫn áp dụng cho chính bạn.
- Xem "score to beat" hiện tại bằng `miner_cli.py status` hoặc trên leaderboard trước khi quyết định nộp.

### 5.2. Tối ưu PPA cho các challenge CPU RISC-V
- **Hiệu năng = IPC × fmax**, và fmax = 1000 / (chu kỳ SDC − WNS). Với SDC cố định 10 ns, **slack dương sẽ trực tiếp tăng fmax**. Ví dụ đường găng 6 ns cho fmax khoảng 167 MHz. Hãy rút ngắn đường găng (ALU/bypass, bộ nhân nhiều tầng, tách decode), nhưng nhớ rằng pipeline sâu hơn làm IPC giảm (thêm stall/flush). **Tối ưu tích IPC × fmax, không tối ưu riêng từng thứ.**
- Có thể tăng IPC bằng forwarding đầy đủ, dự đoán nhánh đơn giản (BTB/bimodal) và giảm penalty load-use. Bộ chia multicycle ít ảnh hưởng vì phép chia hiếm.
- **Diện tích** do Yosys đo trên cell sky130. Phần lớn nằm ở register file (dùng flip-flop), bộ nhân và các khối crypto. Dùng chung tài nguyên (đã bật `SYNTH_SHARE_RESOURCES`), bỏ các thanh ghi pipeline thừa, và cân nhắc nhân nhiều chu kỳ thay vì nhân song song. **Không được bỏ tính năng ISA** mà đề yêu cầu: test ẩn sẽ bắt và gate 100% sẽ loại bạn.
- **Công suất** là ước lượng STA sau synthesis, tăng theo lượng cell và tần số chuyển mạch. Clock gating / enable các thanh ghi, operand isolation cho bộ nhân/crypto khi rảnh, và giảm số flip-flop đều có ích.
- **Nhớ ratio_cap = 2.** Chỉ số nào đã ≥ 2× mục tiêu thì dừng, và dồn sức sang chỉ số còn thấp.
- Điểm xuất phát nên dùng: thiết kế thắng đã reveal, [chipforge-mcu](https://github.com/TatsuProject/chipforge-mcu) (RV32IMC + Zk* + Zbb + F, IPC khoảng 0.58 trên bộ test của họ), `Solution1.zip`/`Solution2.zip` trong `chipforge-challenges`. Bản sao trùng từng byte bị từ chối, và chỉ sửa nhẹ thì không vượt nổi biên độ. Hãy dùng chúng làm baseline để cải tiến.

### 5.3. Challenge NPU (challenge 15, "npu-v1.0", ra mắt khoảng 23/09/2026)
Đọc từ `gateway/main.py` (chế độ `fom_realmacro`) và các commit của EDA server. Chưa có metadata công khai của đề, nên tham số cụ thể phải đọc trong `weights.json` của gói đề ⚠️.
- Bạn nộp một **SoC gồm CPU + bộ tăng tốc (NPU)**. Commit "on-server kernel compile" cho thấy evaluator **biên dịch kernel của miner bằng `riscv-none-elf-gcc` 14.2** ngay trên server, với **input bí mật cho mỗi lần chấm**, rồi chạy nhiều model ML. Vì vậy bài nộp nhiều khả năng gồm cả RTL lẫn mã kernel C/asm. Kiểm tra spec để biết cấu trúc thư mục chính xác.
- **Gate:**
  1. Functionality phải 100% (kết quả đúng từng bit).
  2. `speedup = geomean_theo_model(chu_kỳ_baseline / chu_kỳ_của_bạn) × (fmax_của_bạn / baseline_fmax)` phải **≥ `min_speedup` (mặc định 2.0)**. Giá trị này được làm tròn 3 chữ số trước khi so.
- **Điểm (FoM):** `overall = 1000 × IPS_gain^α / max(area, area_floor)^β`, với α = 0.7, β = 0.3 và `IPS_gain = IPS × (1 − 1/speedup)`.
  - Chỉ tính **phần thông lượng mà accelerator thêm vào**. Một NPU "không làm gì" được 0 điểm.
  - `area_scope = npu_only` ⚠️: diện tích được tính cho riêng accelerator, nhưng không thấp hơn `area_floor_um2`. Vì vậy **tối ưu bằng phần mềm (kernel CPU) mà không dùng silicon sẽ không ăn điểm**.
  - α = 0.7 > β = 0.3, nghĩa là **hiệu năng quan trọng hơn diện tích nhiều**. Tăng gấp đôi IPS_gain được ×1.62 điểm; giảm một nửa diện tích chỉ được ×1.23.
- **Số tham chiếu từ code:** thiết kế mẫu 16×16 có khoảng **4.99 triệu µm², 57.94 MHz, speedup 1.949× (chưa qua gate 2×)**, và mất khoảng 18–25 phút synthesis. Nói cách khác, chính thiết kế tham chiếu của đội ngũ còn trượt gate. Để vượt cần: tăng fmax (pipeline datapath MAC), tăng mức tái sử dụng dữ liệu, và kernel tận dụng accelerator cho **mọi** model (geomean nên một model chậm sẽ kéo cả điểm xuống).
- Thông tin thêm từ commit của [PR #4 trên EDA server](https://github.com/TatsuProject/chipforge_eda_server/pull/4) (merge ngày 01/10/2026):
  - File thiết kế accelerator là `npu.sv`, các biến môi trường mang tiền tố `NPUV1_`.
  - Bộ chấm gồm **"the scored four"**, tức 4 model được tính điểm. Ví dụ trong commit là một bài kiểu conv-accel.
  - Có một bài mẫu được ghi nhận **ACCEPTED với speedup 4.25× và 1 734 287 µm²**. Mức này cho thấy hoàn toàn có thể vượt gate 2×.
  - `dummy_sol_2` chỉ đạt 1.92× và bị loại.
  - Phiên bản công cụ: Verilator `1264184fb` (5.046+55), trình biên dịch kernel xPack `riscv-none-elf-gcc 14.2.0`.
  - Challenge 0014 trước đó dùng cách tính tuyến tính kiểu cũ.
- Điểm `overall` được báo **ngay cả khi trượt gate**, để bạn biết mình còn cách bao xa. Chỉ `overall_gated` (bằng 0 nếu trượt) mới được dùng để xét thắng.
- Đề này **nặng**: mỗi lần chấm local có thể mất 20–40 phút. Hãy làm vòng lặp nhanh bằng mô phỏng riêng, và chỉ chạy synthesis khi đã chắc chắn về chức năng. Đặc biệt chú ý **giới hạn 45 phút**.

### 5.4. Bảo hiểm cho bộ test ẩn
- Chạy **toàn bộ** test trong gói đề với nhiều seed. [rvgen](https://github.com/TatsuProject/rvgen) là bộ sinh lệnh mà đội ngũ dùng; rất có thể test ẩn sinh từ chính công cụ này hoặc riscv-dv. Hãy tự sinh thêm test ngẫu nhiên và so với Spike:
  ```bash
  pip install rvgen
  rvgen --target rv32imc --test riscv_arithmetic_basic_test --steps gen --output out/ --seed 1234
  ```
  Dùng các test-suite khác: `riscv_mmu_stress_test`, `riscv_loop_test`, `riscv_crypto_test`, `riscv_aes_test`, `riscv_sha_test`, `riscv_bitmanip_test`, cùng các phần CSR/exception nếu đề yêu cầu.
- Flow verification độc lập của `chipforge-mcu/verif` (`run_all.py`: biên dịch, chạy regression, so trace với Spike) là khung rất tốt để dựng regression riêng.
- Các vùng hay lệch: hazard sau load, các lệnh nén (C) nằm vắt ngang ranh giới 4 byte, chia cho 0 / tràn số (`DIV`/`REM` có kết quả đặc biệt theo spec), truy cập không thẳng hàng, ghi `x0`, và nhánh ngay sau nhánh.

---

## 6. Sau khi nộp

- Bài nằm ở trạng thái **"In batch"** cho tới khi batch đóng. Mỗi batch có tối đa khoảng 8 bài, chấm song song, và mất hàng chục phút. Validator không thấy kết quả của nhau trước khi nộp.
- Khi batch đóng: server công bố điểm, leaderboard ghi "functional gate failed" hoặc "passed but below the score to beat"… Sau đó dùng `miner_cli.py logs <id>` để lấy log đầy đủ của từng validator.
- Nếu thắng, validator sẽ `set_weights` gần như ngay lập tức (trong giới hạn rate limit của chain) và làm mới mỗi 5 phút. Incentive của bạn trên taostats sẽ tăng trong vài tempo (mỗi tempo khoảng 72 phút). **Chain mới là nơi quyết định ai được trả**; leaderboard có hiển thị những validator nào đang đồng ý.
- Thiết kế thắng của bạn sẽ **bị reveal** cho mọi hotkey trên subnet sau một khoảng thời gian, hoặc khi có người vượt, hoặc khi challenge kết thúc. Người khác sẽ học từ nó, nên lợi thế chỉ kéo dài trong cửa sổ đó.

---

## 7. Những điều tuyệt đối tránh

| Hành vi | Hậu quả |
|---|---|
| Logic khác nhau giữa mô phỏng và tổng hợp (`ifdef VERILATOR`, `translate_off`…) | Bị gán −1 điểm. Server có thể **void kỷ lục** (`baseline_epoch`) và **ban coldkey** vĩnh viễn hoặc theo từng challenge |
| Nộp lại thiết kế thắng của người khác | Bị từ chối nếu trùng byte. Nếu chỉ sửa nhẹ thì không vượt được biên độ, và có rủi ro bị ban |
| Nhiều hotkey cùng coldkey để lách giới hạn 5 lần nộp | Ban áp theo **coldkey**, nên mọi hotkey đều mất |
| Dùng lần nộp chính thức để debug | Mất một trong 5 lượt; lỗi timeout còn không được chấm lại |
| Bỏ tính năng ISA để giảm diện tích | Trượt test ẩn, gate 100% loại bạn |

---

## 8. Checklist trước khi bấm submit

- [ ] `miner_cli.py status`: đúng challenge, còn thời gian, đã ghi lại **score to beat** và số lượt nộp còn lại.
- [ ] Hotkey đang đăng ký trên netuid 108 (`btcli wallet overview --netuid 108`).
- [ ] `rtl.f` ở gốc zip, đúng thứ tự file, đúng tên top module và tên port như đề.
- [ ] Chấm ở local bằng gói testcase mới nhất: `functional_gate = true`, `overall_gate = true`.
- [ ] `overall_local > score_to_beat × (1 + m%)`, **có thêm biên an toàn** vì IPC trên test ẩn có thể thấp hơn.
- [ ] Đã chạy thêm regression ngẫu nhiên (rvgen, nhiều seed) với 0 lệnh sai.
- [ ] Tổng thời gian mô phỏng + synthesis ở local thấp hơn nhiều so với 45 phút.
- [ ] Không có `ifdef` theo công cụ, không có latch ngoài ý muốn, không có `initial` ảnh hưởng tới phần cứng.
- [ ] `submit --dry_run` qua, file dưới 50 MB, đồng hồ máy đã đồng bộ NTP.
- [ ] Đã tính thời điểm nộp: nộp sớm có lợi khi bằng điểm, và có lợi lớn khi challenge chưa có winner (m = 0).

---

## 9. Còn thiếu (làm được khi truy cập được `api.chipforge.io` / taostats)
- `GET /api/v1/public/rules`: luật chính thức dạng dữ liệu (số lượt nộp, biên độ, giờ thưởng, chính sách reveal).
- `GET /api/v1/public/snapshot`: challenge hiện tại, score to beat, winner hiện tại, `miner_emission_percentage` thực tế.
- `GET /api/v1/public/challenges/{id}/leaderboard`: để biết mức điểm của đối thủ.
- Taostats SN108: emission/ngày, phí đăng ký, immunity period. Thay các số này vào công thức ở §0 để biết có đáng làm hay không.
