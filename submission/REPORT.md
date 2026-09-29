# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Thọ Đạt
- **MSSV:** 2A202602484
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/Usagshfil/K4-L3-DAY13-NguyenThoDat-2A202602484-Monitoring-LLMOps.git
- **Commit SHA cuối:** 0aceaae0e531b783bcf6dc9c4fb3ddefdbbb9ee1
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-02484`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | Estimated Score: 30/100 | Estimated Score: 100/100 | Đã bổ sung correlation_id, context enrichment và PII scrubbing hoàn chỉnh |
| `validate_dashboard.py` | HỢP LỆ: 6/6 panel | HỢP LỆ: 6/6 panel | Đạt chuẩn contract yaml với đầy đủ threshold, units, aggregations |
| `pytest` | 22 passed in 1.69s | 24 passed in 1.61s | Pass 100% tests bao gồm bổ sung unit test CCCD và Credit Card |
| Số traces hợp lệ | 0 trace | > 20 traces | Đầy đủ root, retrieval span và generation observation trên Langfuse cá nhân |
| Số PII leak | Còn tồn tại PII thô | 0 leak | Bộ lọc đệ quy scrub_event che toàn bộ Email, SĐT, CCCD, Thẻ ngân hàng |
| Latency P95 / TTFT P95 | 394 ms / 50 ms | 165 ms / 50 ms (Normal) | P95 đo lúc hoạt động bình thường đạt chuẩn; vọt lên ~2654ms khi có incident |
| Retrieval success rate | 100% | 100% | Retrieval hoạt động ổn định trong mọi lượt test |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, request trước tiên được xóa context cũ (`clear_contextvars()`). Middleware đọc header `x-request-id` do client gửi lên; nếu không có, tự sinh ID mới theo định dạng `req-<8-hex>` (`req-` + 8 ký tự hex từ uuid4). ID được gán vào `structlog` contextvars qua `bind_contextvars(correlation_id=...)` và lưu vào `request.state.correlation_id`. Khi trả response, middleware đính kèm header `x-request-id` và `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** Các trường bối cảnh toàn cục gồm `user_id_hash` (băm SHA-256 rút gọn 12 ký tự), `session_id`, `feature`, `model`, `env` được bind ngay đầu hàm `chat()` trước khi ghi log sự kiện `request_received`. Các log sau đó tự động kế thừa các trường này cùng với `correlation_id`, `service`, `level`, và `ts`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Xây dựng hàm `scrub_event` duyệt đệ quy toàn bộ các trường dữ liệu dạng string trong `event_dict` và áp dụng bộ regex `PII_PATTERNS` (Email, Phone VN, CCCD 12 số, Credit Card 16 số). Hàm này được đăng ký làm processor trong `structlog.configure()` ngay trước `JsonlFileProcessor` và `JSONRenderer()`, đảm bảo PII bị che thành `[REDACTED_...]` trước khi được serialize hoặc ghi xuống đĩa.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đạt 100/100; chạy unit tests `python -m pytest tests/test_pii.py`; kiểm tra trực tiếp các file log trong `data/logs.jsonl` không còn bất kỳ chuỗi PII thô nào.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Project Langfuse mang tên `day13-k4-l3a-02484` với public key và secret key cá nhân được nạp qua file `.env`. Mọi traces sinh ra đều có `tags=["lab", feature, self.model]` và liên kết trực tiếp với account cá nhân.
- **Cấu trúc root/retrieval/generation observations:** Sử dụng Langfuse Python SDK v4:
  - Root observation: `lab-agent-run` (loại `agent`).
  - Child observation 1: `retrieval` (loại `span`) bao bọc hàm `_retrieve_docs()`.
  - Child observation 2: `generation` (loại `generation`) bao bọc hàm `_generate_response()`, truyền đầy đủ model, input/output đã scrub, `usage_details` (tokens in/out) và `cost_details`.
- **Cách nối trace với log:** Cả structured log và Langfuse trace metadata đều lưu chung một trường `correlation_id` (định dạng `req-<8-hex>`), cho phép tra cứu 1-1 giữa hai hệ thống.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 mang nhãn `baseline` và `production`.
- **Version/label candidate:** Version 2 mang nhãn `candidate` (bổ sung chỉ dẫn "Please respond concisely.").
- **Trace ID của mỗi version:**
  - Trace ID dùng version 1 (`baseline`): 60ff578cc5871c9c51b36b44c70de6f6
  - Trace ID dùng version 2 (`candidate`): 86819222f4525109e772120a8204a40d
- **Cách promote và rollback `production`:**
  - Promote: Trên giao diện Langfuse Prompts, gán nhãn `production` cho Version 2.
  - Rollback: Chuyển nhãn `production` quay trở lại Version 1. Quá trình chuyển nhãn diễn ra ngay lập tức mà không cần sửa code hay build lại ứng dụng.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Dựng Live Dashboard tại `http://localhost:8501` theo đúng hợp đồng `config/dashboard.yaml` gồm 6 panel: (1) Latency P50/P95/P99 & TTFT P95; (2) Request Traffic; (3) Errors & Retrieval Success Rate; (4) Cumulative Cost Over Time; (5) Input & Output Tokens; (6) Quality Proxy Score. Hỗ trợ hiển thị giờ Việt Nam (UTC+7) và lọc thời gian linh hoạt (All, Last 5m, Custom Range).
- **SLO và lý do chọn:** Chọn SLO `fast_successful_requests`: 99.5% requests thành công có latency <= 3000ms trong chu kỳ 28 ngày. Ngưỡng 3000ms được chọn vì ở điều kiện bình thường, hệ thống phản hồi trong ~160ms–400ms; ngưỡng 3000ms đủ an toàn để hấp thụ các biến động tải thông thường mà vẫn đảm bảo trải nghiệm người dùng không bị gián đoạn.
- **Cách tính error budget:** Error budget = 100% - 99.5% = 0.5% tổng số request. Ví dụ với 100,000 requests/tháng, ngân sách lỗi cho phép tối đa 500 requests bị chậm (> 3000ms) hoặc bị lỗi 5xx.
- **Ba alert và runbook tương ứng:**
  1. `high_latency_p95`: Warning khi P95 latency > 3000ms trong 5 phút. Runbook tại `docs/alerts.md#alert-1`.
  2. `elevated_error_rate`: Critical khi tỷ lệ lỗi > 2% trong 5 phút. Runbook tại `docs/alerts.md#alert-2`.
  3. `degraded_retrieval_success`: Warning khi tỷ lệ tìm kiếm tài liệu thành công < 90% trong 5 phút. Runbook tại `docs/alerts.md#alert-3`.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** 16:49:48 – 16:50:15 (UTC+7 ngày 29/09/2026)
- **Triệu chứng từ metrics:** Trên Live Dashboard panel Latency, P95 latency tăng vọt từ ~160ms lên 2654ms (> ngưỡng cảnh báo 2000ms của challenge). Tuy nhiên TTFT vẫn giữ nguyên ở mức 50ms, Error rate giữ ở mức 0%.
- **Log line và correlation ID liên quan:**
  - Correlation ID: `req-b8ac4de3`
  - Log line trích xuất từ `data/logs.jsonl`:
    `{"service": "api", "latency_ms": 2653, "ttft_ms": 50, "tokens_in": 36, "tokens_out": 173, "cost_usd": 0.002703, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "event": "response_sent", "model": "claude-sonnet-4-5", "correlation_id": "req-b8ac4de3", "env": "dev", "session_id": "k4-l3a-challenge-s04", "feature": "monitoring", "user_id_hash": "4570299f37e2", "level": "info", "ts": "2026-09-29T09:50:06.557104Z"}`
- **Trace ID và span gây ảnh hưởng:** Mở Langfuse trace tìm kiếm theo `correlation_id: req-b8ac4de3`. Trong trace waterfall, quan sát thấy span con `retrieval` chiếm đến 2500ms (chiếm >94% tổng thời gian 2653ms của request), trong khi span con `generation` chỉ tốn 150ms.
- **Root cause:** Sự cố `rag_slow` gây tắc nghẽn ở tầng truy xuất dữ liệu vector/knowledge base (`mock_rag.retrieve`), khiến bước retrieval bị delay 2.5 giây cho mỗi truy vấn thuộc feature `monitoring`.
- **Fix action:**
  1. Cấu hình Semantic Cache / Redis Cache cho các câu truy vấn tài liệu phổ biến để trả về ngay kết quả mà không cần quét lại vector database.
  2. Tối ưu hóa chỉ mục vector (HNSW/IVF index) và nâng cấp tài nguyên cho cụm vector store.
  3. Cài đặt timeout cứng cho bước retrieval (ví dụ 1000ms), nếu quá thời gian sẽ fallback sử dụng tài liệu ngữ cảnh mặc định.
- **Preventive measure:**
  1. Thiết lập alert giám sát riêng cho độ trễ span retrieval (> 1500ms trong 3 phút).
  2. Bổ sung Circuit Breaker ngăn chặn tình trạng thắt cổ chai dây chuyền khi vector store bị quá tải.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Quyết định thực hiện PII scrubbing ở cấp độ `structlog processor` trước khi serialize JSON thay vì làm thủ công ở từng endpoint. Lý do: Giúp che PII tập trung, loại trừ hoàn toàn nguy cơ lập trình viên quên scrub khi thêm các endpoint mới hoặc các log sự kiện mới.
- **Một lỗi/blocker đã gặp:** Ban đầu chạy `validate_logs.py` sau khi sửa code nhưng điểm số vẫn bị trừ do các bản ghi log cũ chưa được scrub còn lưu trong file `data/logs.jsonl`.
- **Cách tìm nguyên nhân và xử lý:** Nhận diện được cơ chế của `validate_logs.py` là quét toàn bộ lịch sử file log. Đã giải quyết bằng cách lưu log baseline, sau đó dọn dẹp file log cũ và khởi động lại API để sinh tập log mới hoàn toàn sạch PII.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - **Metrics:** Cho biết *Hệ thống có vấn đề gì và xảy ra khi nào* (cung cấp bức tranh tổng thể và khoảng thời gian bất thường).
  - **Logs:** Cho biết *Request cụ thể nào bị ảnh hưởng* (khoanh vùng bản ghi chi tiết và lấy `correlation_id`).
  - **Traces:** Cho biết *Bước nào (span con nào) bên trong request là nguyên nhân gốc rễ* (waterfall bóc tách chi tiết retrieval vs generation).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Quản lý prompt version và rollback cho phép cập nhật prompt linh hoạt mà không cần redeploy mã nguồn, đồng thời khôi phục tức thì nếu prompt mới gây suy giảm chất lượng hoặc tăng token bất thường. Giám sát token và chi phí giúp phát hiện sớm các hiện tượng prompt injection hoặc loop generation gây cạn kiệt ngân sách.
- **Điều quan trọng nhất đã học:** Nắm vững kiến trúc Observability toàn diện cho ứng dụng LLM trong môi trường Production, từ việc che giấu dữ liệu nhạy cảm PII, gắn kết bối cảnh xuyên suốt qua Correlation ID, đến việc thiết lập các tầng cảnh báo symptom-based.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Các metric chất lượng (quality score) hiện tại mới dừng ở mức heuristic đơn giản, trong thực tế có thể mở rộng tích hợp LLM-as-a-judge hoặc thu thập phản hồi thumbs-up/thumbs-down từ người dùng thực.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.