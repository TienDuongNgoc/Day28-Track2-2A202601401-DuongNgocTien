# Reflection — Day 28 Track 2

## 1. Các lựa chọn và trade-off

### Hợp đồng sự kiện và idempotency

Mỗi bản tin Kafka luôn mang `idempotency-key` dưới dạng bytes và chỉ mang
`traceparent` khi có một W3C trace context hợp lệ từ request. Cách này giữ được
liên kết giữa HTTP, Kafka và trace backend, đồng thời không phát tán một header
rỗng không hợp lệ. Đổi lại, producer và consumer phải cùng tuân thủ tên header
và encoding UTF-8; production nên quản lý contract bằng schema registry và có
kiểm thử tương thích giữa các phiên bản.

Nguồn cho Delta MERGE được khử trùng theo `idempotency_key`. Nếu có nhiều sự
kiện cùng khóa, sự kiện có cặp `(occurred_at, event_id)` lớn nhất thắng; kết quả
được sắp xếp theo khóa để không phụ thuộc thứ tự Kafka giao bản tin. Cách này
làm replay xác định và an toàn, nhưng dùng thời gian do producer cung cấp nên
production cần chính sách clock-skew, watermark và xử lý late event rõ ràng.

### Offline/online features và retrieval

Yêu cầu Feast lấy tên feature trực tiếp từ `FEATURE_REFS`, tránh lặp lại contract
ở client. Delta vẫn là nguồn lịch sử có version; Feast phục vụ feature online;
Qdrant dùng ID xác định từ `doc_id`. Sự tách biệt này tối ưu độ trễ đọc nhưng tạo
ra eventual consistency giữa Delta, Feast và Qdrant. Vì vậy response và evidence
cần mang `delta_version`, thời điểm materialize và embedding model ID để phát
hiện dữ liệu cũ hoặc index lệch phiên bản.

### Release và rollback

Một release MLflow gói prompt, retrieval config, model ID, embedding model ID,
feature service và Delta version; alias `champion` là con trỏ triển khai. Rollback
chỉ di chuyển alias thay vì sửa mã, nên thao tác nhanh và có thể audit. Trade-off
là cache của serving phải được invalidation đúng lúc và release chỉ đáng tin khi
evaluation artifact, signature và provenance đầy đủ.

### Readiness và degraded mode

Probe bắt buộc lỗi trả `not_ready`; probe không bắt buộc lỗi trả `degraded`; chỉ
khi mọi probe đạt mới trả `ready`. Chính sách này giữ traffic khỏi instance không
thể phục vụ contract tối thiểu, trong khi lỗi feature/retrieval tùy chọn vẫn có
thể trả một response có cảnh báo. Liveness được tách khỏi readiness để sự cố phụ
thuộc không gây restart loop.

### Observability

Trace ID được giữ qua các boundary, còn Prometheus/Grafana theo dõi rate, error,
duration, saturation và Kafka lag. Evidence được lấy từ API/control plane thay
vì suy diễn từ log. Chi phí là telemetry tăng lưu lượng và storage; production
cần sampling, retention, redaction và giới hạn cardinality.

## 2. Khoảng trống trước production

- IP07 chỉ được xác nhận khi có endpoint vLLM chạy trên GPU thật, trả `/version`,
  model list và các metric `vllm:`. Không thay thế gate này bằng mock.
- Nhánh export LangSmith của IP10 cần credential thật; khi thiếu credential phải
  báo `UNVERIFIED`, trong khi Jaeger local vẫn dùng để kiểm tra trace continuity.
- Kafka hiện chưa có schema registry, ACL/mTLS, multi-broker replication và kế
  hoạch partition/rebalance theo tải thực tế.
- Delta/Feast/Qdrant/MLflow cần backup, restore drill, HA, encryption, retention,
  data quality checks và quy trình migration contract.
- Secrets cần secret manager; Grafana/Airflow credential mặc định chỉ phù hợp môi
  trường lab. Kubernetes cần image digest, admission policy, NetworkPolicy được
  kiểm thử trên cluster thật và resource requests/limits dựa trên load profile.
- Cần SLO/error budget chính thức, distributed load/soak test, capacity planning,
  cost monitoring, alert routing/on-call và runbook đã diễn tập.
- Corpus lab quá nhỏ để kết luận retrieval/model quality. Production cần bộ eval
  có ground truth, kiểm tra safety, drift/bias và promotion gate tự động.

## 3. Đóng góp

Bài làm cá nhân, phụ trách toàn bộ 5 vai trò trong `docs/team-role-cards.md`:

- Ingestion & Orchestration: hoàn thiện propagation header cho IP01/IP10 và xác
  nhận contract Kafka/Airflow trong integration matrix.
- Data & ML: hoàn thiện nguồn dedupe xác định cho Delta MERGE và request Feast
  theo registry contract.
- Serving & Retrieval: kiểm tra stable vector IDs, release provenance, fallback
  và các latency budget bằng fast suite.
- Platform & Observability: hoàn thiện readiness semantics; xác thực gateway,
  telemetry, Prometheus/Grafana, Kubernetes và GitOps manifests.
- Presenter / Incident Commander: dùng sơ đồ kiến trúc, demo runbook, evidence
  index và phần production gaps trong tài liệu này để trình bày.

## 4. Cách tái lập kết quả

Fast gate:

```text
uv run pytest starter-tests tests -q
uv run ruff check .
uv run python scripts/verify_matrix.py
uv run python scripts/check_portability.py
uv run python scripts/validate_manifests.py
```

Live gate sau khi full stack healthy:

```text
docker compose --env-file ports.template --profile full up -d --build --wait
uv run lab28 topics
uv run lab28 index --source file
uv run lab28 release
uv run lab28 seed --via-gateway
uv run pytest integration-tests -m "not gpu and not langsmith" -q
uv run lab28 evidence
uv run python load-tests/run_profile.py --requests 200 --workers 8
```

Hai gate GPU và LangSmith phải được báo theo trạng thái thật của môi trường; không
được xem `skip` hoặc mock là bằng chứng đạt.

## 5. Trạng thái xác minh ngày 2026-09-04

- `pytest starter-tests tests -q`: **87 passed**.
- Ruff, integration matrix (245 checks), portability và Kubernetes/GitOps
  manifest validation: **passed**.
- Integration test tĩnh: **1 passed, 71 deselected**.
- Compose core/full config: **valid**. Image Airflow đã build thành công sau khi
  tái sử dụng PySpark 4.2.0 từ image Spark và loại MLflow khỏi image Airflow vì
  DAG không sử dụng dependency này.
- Full live suite: **UNVERIFIED trên máy hiện tại**. Docker Desktop chỉ được cấp
  3.52 GiB RAM; khi bật Spark cùng core stack, Feast lỗi `Cannot allocate memory`
  và Docker engine tiếp tục trả HTTP 500. Không tạo evidence giả từ lần chạy này.
- GPU vLLM và LangSmith: **UNVERIFIED** vì môi trường không cung cấp endpoint GPU
  và credential tương ứng.
