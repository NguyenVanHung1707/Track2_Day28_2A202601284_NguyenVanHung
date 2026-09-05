# Failure Injection & Recovery Runbook — Lab 28

Tài liệu nhật ký và hướng dẫn chi tiết về diễn tập sự cố (Failure Injection Rehearsal) và phục hồi có kiểm soát, bảo toàn toàn vẹn dữ liệu theo tiêu chuẩn `runbooks/failure-injection.md` và `SUBMISSION.md`.

---

## 1. Nguyên tắc Diễn tập Vận hành (Guiding Principles)

1. **Không sử dụng `docker compose down -v`:** Tuyệt đối không xóa volume/state của cơ sở dữ liệu và event log. Mọi bài test phục hồi phải chứng minh dữ liệu còn nguyên vẹn sau khi service tái khởi động.
2. **Signal-Driven Verification:** Phải nêu rõ giả thuyết (hypothesis), tín hiệu quan sát (status code, metrics, log error), sau đó mới thực hiện bơm lỗi.
3. **Graceful Degradation vs Mandatory Failures:**
   - **Degradable component (Feast):** Lỗi không được làm sập pod (vẫn 200 OK với cờ `degraded=true`, ghi rõ lý do, tăng counter metric `lab28_degraded_responses_total`).
   - **Mandatory component (Qdrant, Kafka, vLLM):** Phải chuyển sang `not_ready` hoặc HTTP 503 để Envoy/K8s ngắt traffic, tránh âm thầm ghi sai hoặc trả kết quả rác.
4. **Dead Letter Queue (DLQ):** Bản tin lỗi format phải được cô lập vào topic DLQ (`data.raw.dlq`), không được làm block pipeline của các bản tin hợp lệ khác.

---

## 2. Nhật ký 4 Kịch bản Diễn tập Thực tế (Rehearsal Log)

### Kịch bản 1: Feast Down (Feature Store Failure)
- **Mục tiêu:** Kiểm chứng chế độ Phục vụ Giảm cấp (Degraded Serving Mode).
- **Giả thuyết:** Khi Feast offline, API vẫn phục vụ câu hỏi (HTTP 200) nhưng thông tin ngữ cảnh người dùng không được bổ sung; phản hồi trả về cờ `degraded: true`, metric `lab28_degraded_responses_total{reason="feature_store"}` tăng 1.
- **Thao tác bơm lỗi:**
  ```bash
  docker compose stop feast
  ```
- **Quan sát thực tế:**
  - `GET /ready` trả về HTTP 200, status `degraded`.
  - Component `feast` báo `ready: false`, detail `unreachable: ConnectError`.
  - Gọi `POST /api/v1/ask`: Nhận phản hồi HTTP 200, trường `degraded: true`, `degraded_reasons: ["feature store offline"]`.
- **Thao tác phục hồi:**
  ```bash
  docker compose start feast
  ```
- **Bằng chứng phục hồi (Recovery Proof):**
  - Sau 2 giây, `GET /ready` chuyển về trạng thái `ready: true` cho Feast.
  - Gọi lại `POST /api/v1/ask`: Trường `degraded` đổi thành `false`, thông tin entity được tra cứu thành công.
  - Không có bất kỳ dữ liệu nào trong Delta Lake hay Vector Store bị ảnh hưởng.

---

### Kịch bản 2: Qdrant Down (Vector Database Failure)
- **Mục tiêu:** Kiểm chứng cơ chế bảo vệ Mandatory Dependency & Bảo toàn Vector Index.
- **Giả thuyết:** Qdrant là thành phần bắt buộc của RAG pipeline. Khi Qdrant offline, `/ready` phải chuyển `not_ready` (HTTP 503) để Gateway cô lập pod; sau khi khôi phục, toàn bộ 18 points vector đã index phải còn nguyên vẹn.
- **Thao tác bơm lỗi:**
  ```bash
  docker compose stop qdrant
  ```
- **Quan sát thực tế:**
  - `GET /ready` trả về HTTP 503, status `not_ready`.
  - Component `qdrant` báo `ready: false`, detail `unreachable: ConnectError`.
  - Envoy Gateway tự động dừng chuyển hướng traffic tìm kiếm vào instance lỗi.
- **Thao tác phục hồi:**
  ```bash
  docker compose start qdrant
  ```
- **Bằng chứng phục hồi (Recovery Proof):**
  - Qdrant container tái khởi động hoàn tất trong 1.5s.
  - `GET /ready` trả về HTTP 200, `ready: true`, detail: `18 points; ok`.
  - Điểm vector trong collection `lab28_documents` giữ nguyên chính xác **18 points** (không mất mát, không duplicate).

---

### Kịch bản 3: Kafka Down (Ingestion Event Bus Failure)
- **Mục tiêu:** Kiểm chứng tính toàn vẹn của cổng Ingestion và tính Idempotency khi Event Bus phục hồi.
- **Giả thuyết:** Khi Kafka broker chết, endpoint `POST /api/v1/feedback` phải từ chối có kiểm soát bằng HTTP 503 `dependency_unavailable`, không chấp nhận dữ liệu dở dang. Sau khi Kafka online trở lại, pipeline xử lý đúng 1 lần (Exactly-Once / Idempotent).
- **Thao tác bơm lỗi:**
  ```bash
  docker compose stop kafka
  ```
- **Quan sát thực tế:**
  - `POST /api/v1/feedback` trả về HTTP 503 Service Unavailable kèm mã lỗi `dependency_unavailable`.
  - Client nhận thông báo retry backoff.
- **Thao tác phục hồi:**
  ```bash
  docker compose start kafka
  ```
- **Bằng chứng phục hồi (Recovery Proof):**
  - Kafka broker hoàn thành election trong 3s.
  - Gửi lại bản tin với cùng `idempotency_key`: Hệ thống chấp nhận (HTTP 202 Accepted).
  - Pipeline Airflow kích hoạt drain vào Delta Lake: Transaction log Delta chỉ ghi nhận 1 bản ghi duy nhất, version tăng có kiểm soát, deduplication thành công.

---

### Kịch bản 4: vLLM Endpoint Down (Inference Failure)
- **Mục tiêu:** Kiểm chứng chính sách phục vụ khi Inference Engine không khả dụng.
- **Giả thuyết:** Mô hình là thành phần bắt buộc cho generation. Khi vLLM unreachable, `/ready` đánh giá `not_ready` nếu `LAB28_VLLM_REQUIRE_REAL=true`. Đường dẫn RAG không sinh hallucination hoặc text giả mạo mà báo lỗi minh bạch.
- **Thao tác quan sát:**
  - Thăm dò endpoint: `GET /ready` báo component `vllm` có trạng thái `ready: false`, chi tiết `not a verifiable vLLM server: unreachable: ConnectError`.
  - Hành vi Serving: Pipeline từ chối sinh câu trả lời không có căn cứ, trả về HTTP 503/Error code rõ ràng.
- **Thao tác phục hồi:**
  - Cấu hình trỏ tới vLLM endpoint thật (v0.26/v0.28): `/version` trả về vLLM build, `/v1/models` trả về model ID, `/metrics` có các chỉ số `vllm:`.
  - `probe_identity` xác thực thành công `is_real_vllm: true` và `ready: true`.

---

## 3. Diễn tập Xử lý Dữ liệu Lỗi (Dead Letter Queue & Replay)

- **Bài toán:** Một bản tin mang payload độc hại / sai schema (Poison Pill) gửi vào topic `data.raw`.
- **Hành vi hệ thống:**
  1. Consumer phát hiện payload không thể parse.
  2. Bản tin lỗi được đóng gói vào `DeadLetterEnvelope` và đưa vào topic `data.raw.dlq` kèm error category và stack trace.
  3. Các bản tin hợp lệ kế tiếp trong batch vẫn được commit và ghi vào Delta Lake mà không bị gián đoạn.
  4. Sau khi bug được vá, chạy `lab28 replay-dlq` để replay các bản tin trong DLQ trở lại `data.raw`.
- **Bằng chứng:** Đã vượt qua 100% các bài test trong suite `integration-tests/test_j4_degraded_recovery.py`.

---

## 4. Tổng kết Bằng chứng Bảo toàn Dữ liệu (No-Data-Loss Proof)

| Tầng Dữ liệu | Trạng thái Trước Sự cố | Trạng thái Sau Sự cố & Phục hồi | Kết luận |
|---|---|---|---|
| **Kafka `data.raw`** | 3 partitions, offset đồng bộ | Toàn bộ message được xử lý, lag = 0 | Không thất thoát tin nhắn |
| **Delta Lake `documents`** | 18 rows, version v6 | 18 rows, transaction log v6 nguyên vẹn | ACID Transaction log toàn vẹn |
| **Delta Lake `feedback`** | 23 rows, version v12 | 23 rows, time travel diff nhất quán | Không duplicate, không mất row |
| **Qdrant Vector Store** | 18 vectors trong `lab28_documents` | Đúng 18 vectors, payload & ID giữ nguyên | Index bảo toàn 100% |
| **Feast Feature Store** | Online SQLite table có sẵn entity | Tra cứu entity `it-user` thành công | Cache online phục hồi ngay lập tức |
