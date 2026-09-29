# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: high_latency_p95
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#ai-alerts)
- SLI/SLO liên quan: fast_successful_requests (SLO: 99.5% requests <= 3000ms)
- Điều kiện và thời gian duy trì: P95 latency > 3000ms kéo dài liên tục trên 5 phút
- Ảnh hưởng tới người dùng: Người dùng cảm nhận hệ thống phản hồi chậm hoặc bị treo giao diện
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra dashboard Latency panel xem TTFT có tăng theo tổng latency không.
  2. Lọc `data/logs.jsonl` tìm các request có `latency_ms > 3000` và trích xuất `correlation_id`.
  3. Mở Langfuse trace tìm span con (`retrieval` hay `generation`) chiếm phần lớn thời gian thực thi.
- Mitigation tạm thời: Bật fallback cache cho retrieval, giảm bớt số lượng tài liệu hoặc chuyển sang mô hình LLM dự phòng có tốc độ cao hơn.
- Owner: oncall-ai-ops

## Alert 2

- Tên: elevated_error_rate
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack (#ai-alerts)
- SLI/SLO liên quan: primary_slo (Tỉ lệ thành công của request)
- Điều kiện và thời gian duy trì: Tỉ lệ lỗi `error_rate_pct > 2%` liên tục trên 5 phút
- Ảnh hưởng tới người dùng: Người dùng nhận thông báo lỗi HTTP 500 hoặc không nhận được câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard Errors panel để xem phân bố `error_type` (RuntimeError, Timeout, v.v.).
  2. Tìm trong log các sự kiện `request_failed` và trích xuất `correlation_id` cùng `payload.detail`.
  3. Mở trace trên Langfuse để xác định span bị lỗi (bước retrieval hay do LLM provider lỗi).
- Mitigation tạm thời: Trả về câu trả lời mặc định thân thiện (fallback message), khởi động lại pod service nếu phát hiện rò rỉ bộ nhớ.
- Owner: oncall-ai-ops

## Alert 3

- Tên: degraded_retrieval_success
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#ai-alerts)
- SLI/SLO liên quan: guardrails.retrieval_success_rate_pct_min (>= 90%)
- Điều kiện và thời gian duy trì: Tỉ lệ truy xuất thành công `retrieval_success_rate_pct < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: Câu trả lời của bot bị mất ngữ cảnh (chỉ trả về câu trả lời chung chung hoặc fallback)
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Errors xem tỷ lệ `tool_success == false` và các lỗi liên quan đến vector store/knowledge base.
  2. Lọc log sự kiện có `tool_name == "retrieval"` và `tool_success == false`.
  3. Kiểm tra trace trên Langfuse để xem span `retrieval` bị timeout hay trả về lỗi cụ thể nào.
- Mitigation tạm thời: Kích hoạt fallback retrieval nguồn tài liệu cục bộ hoặc tạm ngắt các truy vấn tìm kiếm phức tạp.
- Owner: oncall-ai-ops
