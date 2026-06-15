# Observathon — Bộ công cụ cho Học viên

🇻🇳 Tiếng Việt | [🇬🇧 English](README_en.md)

Bạn được giao một agent thương mại điện tử **hộp đen, im lặng, đầy lỗi** (dạng binary) chạy
trên một **LLM thật**. Nó không cho bạn biết gì cả. Nhiệm vụ của bạn: **gắn quan sát, chẩn
đoán lỗi, và sửa chúng** — bằng cách sửa config, **viết lại system prompt của agent**, và thêm
một lớp wrapper giảm thiểu lỗi.

## Cài đặt (bắt buộc có một LLM thật)
```bash
# 1. chọn một engine:
export OPENAI_API_KEY=sk-...        # đám mây (model mặc định gpt-5.4-nano), HOẶC
#    local miễn phí: chạy Ollama / llama.cpp (tương thích OpenAI), đặt provider:"local" + LOCAL_BASE_URL trong config.json

# 2. kiểm tra khung bài nộp (chỉ stdlib, không cần key)
python harness/selfcheck.py

# 3. chạy binary mô phỏng giai đoạn PRACTICE (trong bin/practice/)
./bin/practice/observathon-sim --config solution/config.json --wrapper solution/wrapper.py \
    --out run_output.json --concurrency 8
#   macOS lần đầu: xattr -dr com.apple.quarantine bin/practice/observathon-sim
#   Windows:      bin\practice\observathon-sim\observathon-sim.exe ...   (exe lives INSIDE its folder)
```
Agent **không phát ra gì cả** và `run_output.json` **cố tình tối giản** — mỗi dòng chỉ có
`answer` + `status` (không có latency, tokens, lời gọi tool, hay trace). Cách DUY NHẤT để thấy
latency, chi phí, số lần gọi tool, vòng lặp, drift và PII là **gắn quan sát trong
`solution/wrapper.py`**: `call_next()` trả về kết quả ĐẦY ĐỦ (gồm `meta` + `trace`) cho BẠN —
hãy ghi lại bằng bộ `telemetry/` đã học ở Ngày 13. (Sim cũng ghi một khối `sealed` đã ký dành
cho việc chấm điểm — đó không phải phần quan sát của bạn.)

## Bạn tối ưu cái gì (đòn bẩy v6)
Agent **điều khiển bằng prompt** và được giao kèm một system prompt **cố tình tệ** (nó bịa ra
tổng tiền, tính sai, gọi tool dư thừa, lặp lại email/sđt của khách, và **làm theo chỉ dẫn ẩn
trong ghi chú đơn hàng**). **Hãy viết lại `solution/prompt.txt`** — đây là cách sửa có đòn bẩy
cao nhất và là một thành phần điểm **`prompt` chiếm 15%**. Xem
**[`docs/PROMPT_OPTIMIZATION.md`](docs/PROMPT_OPTIMIZATION.md)**.

| Bạn chỉnh | Tác dụng |
|---|---|
| `solution/config.json` | các knob (provider/model, temperature, retry, cache, normalize, redact, `self_consistency`, `tool_budget`, `planner`, …) |
| `solution/prompt.txt` | **system prompt** của agent — viết lại nó |
| `solution/examples.json` | few-shot (tùy chọn) |
| `solution/wrapper.py` | `mitigate()` — quan sát + retry/cache/route/redact/sanitize + định tuyến prompt theo từng request |
| `solution/findings.json` | chẩn đoán (loại lỗi + bằng chứng + nguyên nhân gốc) |

## Chọn binary cho HĐH của bạn (`bin/<phase>/`)
| HĐH / kiến trúc | tệp |
|---|---|
| macOS (Apple Silicon, M1+) | `observathon-sim` / `observathon-score` (arm64) |
| Windows | unzip the folder, run `observathon-sim\observathon-sim.exe` / `observathon-score\observathon-score.exe` (keep the folder intact) |
| Linux | `observathon-sim` / `observathon-score` (x86_64) |

(macOS Intel không có sẵn binary — trên Intel hãy chạy từ mã nguồn với Python + `openai`.)
macOS lần đầu (Gatekeeper): `xattr -dr com.apple.quarantine bin/<phase>/*`. Lịch phát hành:
`practice` ngay từ đầu · public **sim** ở 1h, **score** ở 2h · private **sim** ở 3h, **score** ở 3.5h.

## Tạo lưu lượng thực tế (tự chọn mức tải)
```bash
# 200 người dùng x 12 lượt = 2400 request trải trên một khoảng thời gian mô phỏng
./bin/practice/observathon-sim --users 200 --turns 12 --concurrency 12 \
    --config solution/config.json --wrapper solution/wrapper.py --out run_output.json
```
- `--users N` số người dùng · `--turns K` request mỗi người (K lớn → quality-drift rõ hơn) · `--rps` tốc độ đến · `--concurrency` số request song song.
- **Lưu lượng practice NGẪU NHIÊN mỗi lần** (in ra `random run seed = …`; truyền `--seed <giá trị>` để tái hiện). Việc chấm điểm luôn dùng bộ public/private **cố định**, nên mọi đội được xếp hạng trên cùng lưu lượng.

## Cách chấm điểm
`100 × (0.32·correct + 0.16·quality + 0.13·error + 0.08·latency + 0.09·cost + 0.07·drift +
0.15·prompt) + tối đa 22 × diagnosis-F1`. Quality = LLM judge (`gpt-5.4-mini`, có offline dự
phòng). `prompt` dựa trên **kết quả thực tế** (grounding/số học/tiết kiệm tool/PII/chống
injection trừ đi phần prompt quá dài).

## Bạn nộp gì (git push `solution/` + `run_output.json` + `score.json`)
`config.json` · `prompt.txt` · `examples.json` (tùy chọn) · `wrapper.py` · `findings.json`.

## Các giai đoạn
- **Bây giờ → 1h**: chẩn đoán bằng binary practice; viết lại prompt + config.
- **1h** public **sim** · **2h** public **score** → commit, push, leo bảng.
- **3h** private **sim** (bộ giữ kín + diễn đạt lại + đòn **injection**) · **3.5h** private **score** → push (lần cuối).

Xem `docs/FAULT_CLASSES.md`, `docs/PROMPT_OPTIMIZATION.md`, `docs/WRAPPER_API.md`, `docs/SUBMIT.md`. Luật: `../RULES.md`.
