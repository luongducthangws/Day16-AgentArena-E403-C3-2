# Phân công nhóm 5 thành viên — Agent Arena

## Mục tiêu chung

Hoàn thiện 5 middleware layer trong `harness/layers/`, giữ nguyên toàn bộ `arena/` và `data/`, vượt test, giữ trace gate xanh, đồng thời cải thiện Grounding, Safety và Efficiency.

## Nguyên tắc làm song song

- Mỗi thành viên chỉ sửa file được giao để tránh conflict.
- Không sửa `arena/`, `data/`, `harness/agent.py` hoặc `harness/middleware.py` khi chưa thống nhất cả nhóm.
- Không sửa `claim["text"]`, ngoại trừ phép cắt substring hợp lệ được mô tả cho `Critic`.
- Không hard-code `brief_id`, `doc_id`, nội dung corpus hoặc `Doc.tags`.
- Mỗi người tự chạy test liên quan trước khi bàn giao.
- Thành viên 5 giữ vai trò tích hợp cuối, nhưng không tự ý viết lại phần của người khác; lỗi thuộc layer nào được trả lại đúng chủ sở hữu.

## Thành viên 1 — Critic

**File sở hữu:** `harness/layers/critic.py`

Nhiệm vụ:

- Giữ claim có `text` xuất hiện trong `ctx.observed_text`.
- Loại claim bịa.
- Xử lý claim ghép từ hai nguồn khi có thể tách thành hai substring hợp lệ.
- Đặt `abstain = true` khi không còn bằng chứng hoặc gặp mâu thuẫn cần từ chối kết luận.
- Đồng bộ lại `claims`, `citations` và `answer` khi abstain.

Tiêu chí bàn giao:

- Không tạo claim mới từ corpus.
- Không sửa chữ claim hợp lệ.
- Không raise trên report thiếu hoặc sai kiểu dữ liệu.
- Test mục tiêu:

```bash
python3 -m pytest -q tests/test_layers_stubs.py tests/test_scorer.py
python3 scripts/run_practice.py --layers critic
```

## Thành viên 2 — Citation Checker

**File sở hữu:** `harness/layers/citation_checker.py`

Nhiệm vụ:

- Kiểm tra claim có khớp nguyên văn một dòng trong tài liệu đang trích hay không.
- Với citation sai, tìm tài liệu đã được quan sát chứa đúng dòng đó.
- Chỉ đổi `claim["doc_id"]`; giữ nguyên `claim["text"]`.
- Cập nhật danh sách `report["citations"]` đã sắp xếp.
- Để claim không có nguồn cho `Critic` loại bỏ.

Tiêu chí bàn giao:

- Chỉ gắn tài liệu đã được đọc đầy đủ: `doc.body in ctx.observed_text`.
- Không tìm nguồn bằng `Doc.tags`.
- Không gắn tài liệu chưa được truy xuất.
- Test mục tiêu:

```bash
python3 -m pytest -q tests/test_layers_stubs.py tests/test_scorer.py
python3 scripts/run_practice.py --layers critic,citation_checker
```

## Thành viên 3 — Injection Guard

**File sở hữu:** `harness/layers/injection_guard.py`

Nhiệm vụ:

- Làm sạch mọi block từ `BLOCK_START` đến `BLOCK_END` tại `wrap_tool_call`.
- Nếu thiếu marker đóng do dữ liệu bị cắt, loại từ marker mở đến hết nội dung.
- Thay block bị loại bằng `PLACEHOLDER`.
- Quét và loại `INJECTION_CANARY` khỏi `report["answer"]` trong `after_agent`.
- Giữ nguyên toàn bộ `claim["text"]`.

Tiêu chí bàn giao:

- Xử lý được nhiều block độc trong cùng kết quả.
- Giữ nguyên `ok` và `error` khi tạo `ToolResult` mới.
- Không làm hỏng kết quả tool sạch.
- Test mục tiêu:

```bash
python3 -m pytest -q tests/test_layers_stubs.py tests/test_tools.py
python3 scripts/run_practice.py --layers injection_guard
```

## Thành viên 4 — Budget Policy

**File sở hữu:** `harness/layers/budget_policy.py`

Nhiệm vụ:

- Xác định hết ngân sách bằng `ctx.tools.calls >= max_tool_calls - reserve`.
- Khi hết ngân sách, thêm message chứa `FINALIZE_SENTINEL` để ép FINAL.
- Trả danh sách message mới; không mutate danh sách đầu vào.
- Chặn tool call phát sinh sau ngưỡng bằng `ToolResult(ok=False, ...)`.
- Luôn dành lượt cho `submit`.

Tiêu chí bàn giao:

- `max_tool_calls is None` không bao giờ bị coi là hết ngân sách.
- Không nén hoặc loại lịch sử quan sát.
- Không gọi tool thật khi ngưỡng đã cạn.
- Test mục tiêu:

```bash
python3 -m pytest -q tests/test_layers_stubs.py tests/test_middleware.py
python3 scripts/run_practice.py --layers budget_policy
```

## Thành viên 5 — Retry và tích hợp

**File sở hữu:** `harness/layers/retry.py`

Nhiệm vụ layer:

- Retry khi `not result.ok` hoặc `is_degraded(result.content)`.
- Giới hạn tổng số lần thử bằng `max_attempts`.
- Dừng trước phần ngân sách dành cho `submit`.
- Trả kết quả cuối cùng kể cả khi vẫn lỗi.
- Ghi số lần thử vào `ctx.state` phục vụ chẩn đoán.

Nhiệm vụ tích hợp sau khi cả nhóm bàn giao:

- Xác nhận stack đúng thứ tự: `InjectionGuard`, `Critic`, `CitationChecker`, `BudgetPolicy`, `Retry`.
- Chạy full test, verify, baseline và full stack.
- Kiểm tra trace gate, FINAL, số tool call và chênh lệch điểm.
- Rà diff để xác nhận chỉ các file được phép thay đổi.

Tiêu chí bàn giao:

- Không retry vô hạn.
- Không vượt `ctx.max_tool_calls - reserve`.
- Retry dùng nguyên `name` và `args` ban đầu.
- Test mục tiêu:

```bash
python3 -m pytest -q tests/test_layers_stubs.py tests/test_tools.py
python3 scripts/run_practice.py --layers retry
```

## Quy trình tích hợp cuối

Chạy lần lượt:

```bash
python3 -m pytest -q
python3 scripts/verify.py
python3 scripts/run_practice.py --layers none --entry baseline --out runs/baseline.json
python3 scripts/run_practice.py --layers all --entry team --out runs/team.json
python3 scripts/leaderboard.py runs/baseline.json runs/team.json
python3 scripts/selfeval.py --run runs/team.json
git status --short
git diff --stat
```

Điều kiện hoàn thành:

- Toàn bộ test xanh.
- `scripts/verify.py` xác nhận file đóng băng nguyên vẹn.
- Mọi brief đều qua trace gate.
- Không có cảnh báo thiếu FINAL.
- Full stack tốt hơn baseline rõ rệt.
- Không có thay đổi ngoài `harness/layers/` và tài liệu phân công này.

## Mẫu bàn giao của mỗi thành viên

```text
Layer:
File đã sửa:
Logic chính:
Test đã chạy:
Kết quả:
Rủi ro hoặc điểm cần người tích hợp kiểm tra:
```
