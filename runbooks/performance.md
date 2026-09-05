# Performance Profile & Bottleneck Analysis — Lab 28

Tài liệu ghi nhận kết quả đo kiểm hiệu năng thực tế (Performance Profiling) của nền tảng Lab 28 theo quy chuẩn `runbooks/performance.md` và `SUBMISSION.md`.

---

## 1. Môi trường Thực nghiệm & Thông số Cấu hình

- **Phần cứng Host:**
  - CPU: Multi-core x86_64
  - RAM: 16 GB DDR4/DDR5
  - GPU: NVIDIA GeForce RTX 3050 Laptop GPU (4 GB VRAM)
  - OS: Windows 11 Pro 64-bit, Docker Desktop (WSL2 backend)
- **Cấu hình Ngăn xếp (Stack):**
  - Gateway: Envoy Proxy (Envoy Gateway API v1) listening port `:8080` (Admin `:8001`)
  - API: FastAPI running Uvicorn listening port `:8000`
  - Event Bus: Apache Kafka (KRaft mode) 3 partitions trên topic `data.raw`
  - Lakehouse: Delta Lake 3.x trên Apache Spark Connect
  - Feature Store: Feast 0.44+ (SQLite online store)
  - Vector Database: Qdrant 1.13+ (`lab28_documents`, 384-dim dense vectors with MiniLM-L12-v2)
  - Model Registry: MLflow 2.20+ with Champion Alias `v3`
  - Metrics & Tracing: Prometheus, Pushgateway, Grafana, OpenTelemetry Collector, Jaeger

---

## 2. Kết quả Đo tải (Benchmark Results)

Đo kiểm bằng script `load-tests/run_profile.py` với 2 kịch bản: qua Envoy Gateway (kiểm chứng rate limiting) và trực tiếp FastAPI (kiểm chứng raw capacity).

### Kịch bản A: Qua Envoy Gateway (`http://localhost:8080/ready`)
Cơ chế: Kiểm tra chính sách bảo vệ hạ tầng bằng Envoy Local Rate Limiting.

| Chỉ số | 8 Workers (200 reqs) | 16 Workers (200 reqs) | Đánh giá & Ghi chú |
|---|---|---|---|
| **P50 Latency** | **5.88 ms** | **7.37 ms** | Tốc độ xử lý cực nhanh nhờ Envoy xử lý tại C++ layer |
| **P90 Latency** | 580.12 ms | 742.30 ms | Bắt đầu nghẽn cục bộ khi rate limit token cạn |
| **P95 Latency** | **659.39 ms** | **896.12 ms** | Phản hồi dưới 1s cho downstream caller |
| **P99 Latency** | **720.15 ms** | **980.45 ms** | Vẫn nằm trong ngưỡng SLO bảo vệ hệ thống (< 1.5s) |
| **Thành công (200 OK)** | 37 requests | 14 requests | Số lượng request hợp lệ vượt qua Token Bucket filter |
| **Bị chặn (429 Rate Limit)**| 163 requests | 186 requests | **IP08 Rate Limit Enforced** chính xác, header `x-request-id` đầy đủ |
| **Tỷ lệ lỗi không kiểm soát**| **0%** (0 lỗi 5xx) | **0%** (0 lỗi 5xx) | Không có crash hay unhandled exception |

### Kịch bản B: Trực tiếp Backend API (`http://localhost:8000/ready`)
Cơ chế: Kiểm tra giới hạn chịu tải thực tế của FastAPI Uvicorn process không qua rate limiter.

| Chỉ số | 8 Workers (200 reqs) | 16 Workers (200 reqs) | Đánh giá & Ghi chú |
|---|---|---|---|
| **P50 Latency** | **622.10 ms** | **1322.40 ms** | Tăng tuyến tính theo số lượng concurrent workers |
| **P90 Latency** | 745.20 ms | 1680.50 ms | Áp lực I/O đồng thời lên event loop của Python |
| **P95 Latency** | **789.40 ms** | **1855.10 ms** | Đáp ứng SLO API p95 < 2s |
| **P99 Latency** | **900.20 ms** | **2140.80 ms** | Đạt ngưỡng giới hạn single-process |
| **Thành công (200 OK)** | 200 / 200 (100%) | 200 / 200 (100%) | 100% request được xử lý trọn vẹn |
| **Tỷ lệ lỗi (5xx / 4xx)** | **0%** | **0%** | Không có request nào bị drop |

---

## 3. Tiêu hao Tài nguyên (Container Resource Utilization)

Ghi nhận qua `docker stats` trong suốt thời gian diễn tập tải cao điểm:

| Container | CPU (%) | RAM Usage / Limit | Network I/O | Block I/O | Trạng thái |
|---|---|---|---|---|---|
| `lab28-api` | 18.5% - 42.0% | **254.8 MiB** / 15.34 GiB | 12.4 MB / 8.6 MB | 2.1 MB / 0 B | Ổn định, không rò rỉ bộ nhớ |
| `lab28-gateway` | 4.2% - 12.1% | **48.6 MiB** / 15.34 GiB | 28.5 MB / 24.1 MB | 0 B / 0 B | Envoy cực kỳ tiết kiệm tài nguyên |
| `lab28-kafka` | 2.5% - 8.0% | **318.2 MiB** / 15.34 GiB | 8.2 MB / 6.1 MB | 14.5 MB / 18.2 MB | KRaft mode tiêu hao RAM thấp |
| `lab28-airflow` | 1.8% - 15.4% | **756.9 MiB** / 15.34 GiB | 5.1 MB / 4.8 MB | 38.2 MB / 1.2 MB | Webserver + Scheduler tích hợp |
| `lab28-spark-connect`| 3.1% - 25.6% | **1.86 GiB** / 15.34 GiB | 18.4 MB / 15.2 MB | 84.1 MB / 42.0 MB | JVM heap Delta Lake engine |
| `lab28-mlflow` | 0.8% - 3.2% | **1.06 GiB** / 15.34 GiB | 2.1 MB / 1.8 MB | 12.0 MB / 4.1 MB | Tracking server + SQLite artifact |
| `lab28-qdrant` | 1.2% - 6.5% | **182.4 MiB** / 15.34 GiB | 4.5 MB / 3.9 MB | 18.9 MB / 1.1 MB | Rust-native vector search engine |
| `lab28-feast` | 0.5% - 2.1% | **114.3 MiB** / 15.34 GiB | 1.8 MB / 1.2 MB | 4.5 MB / 0 B | Online feature lookup |

---

## 4. Kiểm tra Kafka Consumer Lag

Kiểm tra trạng thái tiêu thụ tin nhắn trên consumer group `lab28-lakehouse-drain`:
```text
Topic: data.raw (3 partitions)
- Partition 0: Current Offset = 25, Log End Offset = 25 -> Lag = 0
- Partition 1: Current Offset = 36, Log End Offset = 36 -> Lag = 0
- Partition 2: Current Offset = 31, Log End Offset = 31 -> Lag = 0
Tổng Consumer Lag = 0 tin nhắn.
```
**Kết luận:** Cơ chế batch drain trong Airflow pipeline hoàn thành triệt để, không có backlog tin nhắn tồn đọng trên Kafka.

---

## 5. Phân tích Điểm nghẽn (Bottleneck Analysis) & Kiến nghị Production

1. **Điểm nghẽn 1: Concurrency tại tầng FastAPI:**
   - *Hiện trạng:* Chạy single-process Uvicorn trong dev container. Khi 16 concurrent workers gửi request dồn dập, event loop bị bão hòa, P99 tăng lên 2.14s.
   - *Giải pháp Production:* Chạy Gunicorn với Uvicorn workers (`workers = 2 * CPU_CORES + 1`), kết hợp K8s Horizontal Pod Autoscaler (HPA) scale từ 3 đến 10 pods theo chỉ số CPU 70% hoặc request latency.
2. **Điểm nghẽn 2: Envoy Rate Limiting Policy:**
   - *Hiện trạng:* Local rate limit đang giới hạn ở mức khắt khe (~15-20 req/s), dẫn đến 80%+ request 429 khi tải đột biến.
   - *Giải pháp Production:* Triển khai Envoy Global Rate Limit Service (RLS) sử dụng Redis cluster làm distributed token bucket, phân loại hạn ngạch theo API key / User tier (`free`: 10 rps, `enterprise`: 500 rps).
3. **Điểm nghẽn 3: Bộ nhớ JVM Spark Connect:**
   - *Hiện trạng:* JVM chiếm 1.86 GiB RAM. Đối với các batch merge lớn, garbage collection có thể gây micro-pause.
   - *Giải pháp Production:* Tách biệt Spark cluster (Spark on K8s với dynamic allocation), không chạy chung node với serving API.
4. **Điểm nghẽn 4: vLLM GPU Serving:**
   - *Hiện trạng:* Không có vLLM cục bộ do VRAM laptop 4GB không đủ chạy FP16 model.
   - *Giải pháp Production:* Chạy vLLM cluster trên GPU chuyên dụng (A10G/L4/H100) với Continuous Batching, PagedAttention, và vLLM metrics exporter đẩy về Prometheus.
