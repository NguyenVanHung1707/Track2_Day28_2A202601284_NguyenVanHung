# Kịch bản Trình diễn Toàn diện (Demo Runbook) — Lab 28

Tài liệu hướng dẫn kịch bản trình diễn trực tiếp (Live Presentation & Technical Demo) cho Hội đồng Giám khảo/Giảng viên theo đúng trình tự 8 bước chuẩn mực tại `docs/demo-runbook.md` và `CHECKLIST.md`.

---

## Phân công Vai trò (Team Role Alignment)

- **Người thuyết trình chính (Presenter / Incident Commander):** Điều phối thời gian, giới thiệu kiến trúc, diễn tập sự cố và trả lời Q&A.
- **Kỹ sư Ingestion & Event-Driven (Team Ingestion - IP01, IP02):** Thao tác Kafka producer, topic headers, Airflow DAG execution, DLQ.
- **Kỹ sư Data Lakehouse & MLOps (Team Data - IP03, IP04, IP06):** Trình diễn Delta Lake time-travel log, Feast online store, MLflow model promotion & rollback.
- **Kỹ sư Serving & Retrieval (Team Serving - IP05, IP07):** Trình diễn Qdrant vector hybrid search, RAG pipeline, vLLM identity & degradation policy.
- **Kỹ sư Nền tảng & Giám sát (Team Platform - IP08, IP09, IP10):** Trình diễn Envoy Gateway, Jaeger distributed trace, Grafana golden signals dashboard, Kubernetes & GitOps.

---

## 8 Bước Trình diễn Trực tiếp (Live Demo Flow)

### Bước 1: Giới thiệu Kiến trúc & Phân chia Quyền sở hữu (Architecture & Ownership)
- **Thời lượng:** ~1.5 phút
- **Nội dung:** Mở sơ đồ `docs/images/lab28-architecture-overview.png` hoặc `.svg`:
  1. **Tầng 0 (External Ingestion):** Khách hàng/Hệ thống ngoài gửi phản hồi và truy vấn qua Envoy Gateway (`:8080`).
  2. **Tầng 1 (API & Event Bus):** FastAPI nhận dữ liệu, sinh traceparent header, đưa vào Apache Kafka topic `data.raw`.
  3. **Tầng 2 (Lakehouse & Feature Store):** Airflow DAG orchestrate Spark Connect thực hiện Delta Merge vào Delta Lake (`feedback` và `documents`), sau đó materialize vào Feast Online Store và Qdrant Vector Store.
  4. **Tầng 3 (ML & Model Registry):** MLflow lưu trữ Champion Release model v3 với đầy đủ provenance (git SHA, Delta version, signatures).
  5. **Tầng 4 (Observability & GitOps):** Prometheus, Grafana, Jaeger, Otel Collector thu thập 4 tín hiệu vàng và distributed traces; Argo CD quản trị K8s manifests.
- **Nhấn mạnh:** 10 Integration Points (IP01–IP10) có ranh giới rõ ràng, không có thành phần nào bị cô lập.

---

### Bước 2: Luồng Hoạt động Chuẩn (Happy Path Journey)
- **Thời lượng:** ~2.5 phút
- **Thao tác 1: Gửi Feedback qua Envoy Gateway:**
  ```bash
  curl -X POST http://localhost:8080/api/v1/feedback \
    -H "Content-Type: application/json" \
    -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
    -d '{"asker_id": "demo-user-01", "text": "Hệ thống RAG rất nhanh và chính xác", "rating": 5, "label": "positive"}'
  ```
  *Kết quả:* Nhận HTTP 202 Accepted kèm `trace_id`.
- **Thao tác 2: Kích hoạt Pipeline Xử lý trên Airflow:**
  Mở Web UI Airflow (`http://localhost:8088`), quan sát DAG `lab28_ingestion_pipeline` chạy các task:
  `consume_kafka` → `spark_delta_merge` → `materialize_feast` → `index_qdrant`.
- **Thao tác 3: Kiểm chứng Dữ liệu đã Cập nhật Đồng bộ:**
  - Delta Lake: Lệnh `lab28 delta-history` cho thấy phiên bản tăng lên, log commit ACID được ghi nhận.
  - Feast: Entity `demo-user-01` có điểm trung bình 5 sao.
  - Qdrant: Vector collection `lab28_documents` duy trì đầy đủ điểm vector có thể tìm kiếm.
  - Gọi endpoint hỏi đáp: `POST /api/v1/ask` nhận câu trả lời grounded từ ngữ cảnh tài liệu mới.

---

### Bước 3: Phân tích Dấu vết Phân tán (Distributed Trace Inspection)
- **Thời lượng:** ~1.5 phút
- **Thao tác:** Mở giao diện Jaeger UI (`http://localhost:16686`), tìm kiếm theo Trace ID vừa tạo.
- **Trình bày:**
  - Chứng minh sự liên tục của ngữ cảnh dấu vết qua ranh giới tiến trình (Cross-process trace context propagation).
  - Chỉ rõ các span chính mang cùng Trace ID:
    1. `lab28.gateway.request` (Envoy)
    2. `lab28.api.ingest` (FastAPI)
    3. `lab28.kafka.produce` (Producer)
    4. `lab28.kafka.consume` (Airflow Consumer)
    5. `lab28.spark.delta_merge` (Spark Connect)
    6. `lab28.api.ask`
    7. `lab28.feast.get_online_features`
    8. `lab28.qdrant.query`
    9. `lab28.mlflow.resolve_release`
- **Khẳng định:** Không có span nào bị ngắt quãng hoặc tạo Trace ID mới giữa chừng.

---

### Bước 4: Giám sát Tín hiệu Vàng (Golden Signals & Grafana Dashboard)
- **Thời lượng:** ~1.5 phút
- **Thao tác:** Mở Grafana Dashboard (`http://localhost:3000`).
- **Phân tích 4 Tín hiệu Vàng (The 4 Golden Signals):**
  1. **Traffic (Lưu lượng):** Biểu đồ `lab28_request_seconds_count` và `envoy_http_downstream_rq_total`.
  2. **Latency (Độ trễ):** Histogram phân phối p50, p95, p99 của endpoint `/ready` và `/api/v1/ask`.
  3. **Errors (Tỷ lệ lỗi):** Đồ thị HTTP 4xx (rate limit 429) và HTTP 5xx (giữ ở mức 0%).
  4. **Saturation (Độ bão hòa):** Mức chiếm dụng CPU, RAM của các container và Kafka Consumer Lag = 0.

---

### Bước 5: Diễn tập Sự cố & Tự phục hồi (Incident Injection & Zero Data Loss)
- **Thời lượng:** ~2.0 phút
- **Thao tác:**
  1. *Nêu giả thuyết:* Dừng container Feast (`docker compose stop feast`), dự đoán API không crash mà chuyển sang `degraded mode`.
  2. *Thực hiện dừng:* `docker compose stop feast`.
  3. *Quan sát:* Endpoint `GET /ready` chuyển sang `degraded`, detail `unreachable: ConnectError`. Gửi câu hỏi RAG vẫn nhận HTTP 200 kèm cờ `degraded: true`.
  4. *Khôi phục:* `docker compose start feast`.
  5. *Chứng minh bảo toàn:* Sau 2s, `/ready` phục hồi `ready: true`, dữ liệu và vector không bị mất mát hay trùng lặp.

---

### Bước 6: Thăng cấp & Đảo ngược Mô hình (MLflow Promotion & Rollback)
- **Thời lượng:** ~1.5 phút
- **Thao tác:**
  1. Cho xem Champion hiện tại là version `v3`.
  2. Chuyển alias Champion sang version trước đó (`v2` hoặc `v1`) bằng MLflow Python Client:
     ```python
     client.set_registered_model_alias("lab28-rag-release", "champion", "2")
     ```
  3. Kiểm tra serving: API tự động resolve release mới mà **không cần restart container hay redeploy code**.
  4. Rollback lại alias về `v3`: Hệ thống ngay lập tức phục hồi phiên bản tối ưu.

---

### Bước 7: Quản trị Cấu hình GitOps & Tự phục hồi (GitOps & Drift Healing)
- **Thời lượng:** ~1.5 phút
- **Thao tác:**
  1. Mở tệp `deploy/kubernetes/base/api.yaml` và `gitops/application.yaml`.
  2. Giải thích cơ chế: Manifests tuân thủ 100% các tiêu chuẩn an toàn (non-root, pinned tag, PDB, HPA, NetworkPolicy, Gateway API v1).
  3. Chạy lệnh kiểm tra: `uv run python scripts/validate_manifests.py` → `Exit code 0`.
  4. Mô tả kịch bản Drift: Khi cấu hình trên cluster bị sửa đổi trái phép (ví dụ can thiệp sửa replicas), cơ chế `selfHeal: true` của Argo CD sẽ tự động đồng bộ Live State về đúng Desired State trên Git repo.

---

### Bước 8: Tổng kết & Trả lời Phản biện (Q&A & Production Gaps)
- **Thời lượng:** ~3.0 phút
- **Nội dung chuẩn bị sẵn:**
  - Sẵn sàng trả lời sâu về **Trade-offs kỹ thuật** (Kafka vs RabbitMQ, Delta Lake vs Iceberg, Feast vs Redis thô, Envoy vs NGINX).
  - Phân tích rõ ràng **4 khoảng cách tới Production thực tế** (HA/Multi-node clustering, Multi-region active-passive, Bảo mật mTLS/Vault, Quota & GPU Autoscaling).
  - Giải trình trung thực về tiêu chí GPU / LangSmith (báo `UNVERIFIED` do giới hạn phần cứng laptop 4GB VRAM, kiên quyết không làm giả mock server để đảm bảo tính liêm chính kỹ thuật).

---

## Quy định Video Demo Dự phòng (Fallback Recording)

Nếu buổi bảo vệ gặp sự cố đường truyền hoặc mạng, sử dụng video quay sẵn với các yêu cầu bắt buộc:
1. Hiển thị đồng hồ hệ thống (System Clock) trên màn hình góc phải.
2. Hiển thị rõ câu lệnh thực thi trong Terminal.
3. Hiển thị rõ các mã Run ID, Trace ID, Delta Version và MLflow Model Version.
