# Buổi 07 — Monitoring, Metrics và Drift Detection

> **Dataset:** [House Sales in King County, USA](https://www.kaggle.com/datasets/harlfoxem/housesalesprediction) (`data/raw/kc_house_data.csv`), raw `data/raw/kc_house_data.csv` (~21510 rows; `sqft_living→area`, `yr_built→age`, `zipcode→location`, `floors`).
> Chuẩn bị lại: `# column mapping in src/ingestion/ingest.py`


## Mục tiêu buổi học

- Instrument FastAPI với Prometheus metrics
- Thu thập logs có cấu trúc (JSON logging)
- Cấu hình Prometheus scrape metrics
- Cấu hình Loki + Promtail thu thập logs
- Viết script phát hiện data drift
- Thiết lập alert rules

---

## Kiến thức lý thuyết

### 3 tầng Monitoring

| Tầng | Mô tả | Ví dụ metrics |
|------|--------|---------------|
| **Service metrics** | Giám sát hiệu năng hệ thống | Latency (p50, p95, p99), error rate, throughput (req/s) |
| **ML metrics** | Giám sát chất lượng mô hình | Accuracy, R², drift score, prediction distribution |
| **Business metrics** | Giám sát tác động kinh doanh | Conversion rate, revenue, số lượng dự đoán sai ảnh hưởng nghiệp vụ |

### Prometheus

- **Pull-based**: Prometheus chủ động kéo (scrape) metrics từ các endpoint `/metrics`
- **Time-series DB**: Lưu trữ dữ liệu dạng chuỗi thời gian
- **PromQL**: Ngôn ngữ truy vấn mạnh mẽ (ví dụ: `rate(request_count[5m])`)
- **Alerting**: Định nghĩa rules, khi điều kiện thỏa mãn → gửi cảnh báo qua Alertmanager

### Loki

- **Log aggregation**: Thu thập và lưu trữ logs tập trung
- **LogQL**: Ngôn ngữ truy vấn logs (tương tự PromQL)
- Kết hợp với **Grafana** để hiển thị logs trực quan
- Không index nội dung log (chỉ index labels) → tiết kiệm tài nguyên

### Promtail

- Agent chạy trên mỗi máy, đọc file log và đẩy vào Loki
- Cấu hình đường dẫn log, labels, và parsing rules
- Hỗ trợ pipeline stages: regex, json, labels, timestamp

### Grafana

- Nền tảng visualization dashboards
- Hỗ trợ nhiều data sources: Prometheus, Loki, PostgreSQL, ...
- Tạo dashboard với nhiều panel: graph, stat, table, logs

### Các loại Drift

| Loại Drift | Mô tả | Phương pháp phát hiện |
|------------|--------|----------------------|
| **Data Drift** | Phân phối input thay đổi theo thời gian | So sánh thống kê: z-score, KS test, PSI |
| **Model Drift** | Performance mô hình giảm dần | Theo dõi metrics: R², MAE, RMSE theo thời gian |
| **Concept Drift** | Mối quan hệ giữa input và output thay đổi | So sánh prediction distribution, cần ground truth |

### Delayed Evaluation

Trong nhiều bài toán, **ground truth đến muộn** so với thời điểm dự đoán:

- **Ví dụ**: Dự đoán giá nhà hôm nay, nhưng giá bán thực tế chỉ biết sau 3 tháng
- **Hệ quả**: Không thể tính accuracy ngay → phải dùng proxy metrics hoặc data drift để giám sát tạm thời
- **Chiến lược**: Khi có ground truth → tính metrics thực tế → quyết định retrain

---

## Cấu trúc file mới thêm

```
session-07-monitoring/
├── app/
│   └── metrics.py                        # PrometheusMiddleware, Counter, Histogram
├── infra/
│   ├── prometheus.yml                    # Cấu hình Prometheus scrape (stack Compose - buổi 08)
│   ├── prometheus.local.yml              # Cấu hình scrape uvicorn chạy trên host (lab buổi 07)
│   └── promtail.yml                      # Cấu hình Promtail đọc logs
├── monitoring/
│   ├── prometheus/
│   │   └── alerts.yml                    # Alert rules
│   ├── grafana/
│   │   ├── dashboards/
│   │   │   └── model-api-overview.json   # Dashboard có sẵn: traffic, latency, lỗi, logs
│   │   └── provisioning/                 # Tự động nạp data source + dashboard
│   │       ├── datasources/datasources.yml
│   │       └── dashboards/dashboards.yml
│   └── generate_drift_report.py          # Script phát hiện data drift
└── docs/
    └── retraining-trigger.md             # Tài liệu chiến lược retrain
```

---

## Hướng dẫn thực hành

### Bước 1: Checkout branch

```bash
git checkout session-07-monitoring
```

### Bước 2: Xem `app/metrics.py`

Hiểu cách tích hợp Prometheus với FastAPI:
- `PrometheusMiddleware`: middleware tự động đo latency và đếm request
- `Counter`: đếm số lần xảy ra sự kiện (ví dụ: tổng request, tổng prediction)
- `Histogram`: đo phân phối giá trị (ví dụ: latency theo percentile)

### Bước 3: Chạy API local và kiểm tra metrics

```bash
uvicorn app.main:app --reload --port 8000
```

Gửi vài request rồi truy cập endpoint metrics:
```bash
curl http://localhost:8000/metrics
```

Kết quả sẽ hiển thị dạng Prometheus exposition format:
```
# HELP request_count_total Tổng số request
# TYPE request_count_total counter
request_count_total{method="GET",endpoint="/health",status="200"} 3.0
...
```

### Bước 4: Xem cấu hình Prometheus

Mở file `infra/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "model-api"
    static_configs:
      - targets: ["model-api:8000"]

rule_files:
  - "/etc/prometheus/alerts.yml"
```

### Bước 5: Xem alert rules

Mở file `monitoring/prometheus/alerts.yml` — có 3 rules:

| Alert | Biểu thức | `for:` | Mức độ |
|-------|-----------|--------|--------|
| **HighErrorRate** | `rate(http_requests_total{status=~"5.."}[5m]) > 0.1` | 2 phút | critical |
| **HighLatency** | `http_request_duration_seconds_sum / ..._count > 0.5` | 5 phút | warning |
| **ServiceDown** | `up{job="model-api"} == 0` | 1 phút | critical |

Trường `for:` là thời gian điều kiện phải **đúng liên tục** trước khi alert chuyển sang
trạng thái Firing — xem tận mắt ở Bước 9.3.

### Bước 6: Chạy drift report

```bash
python monitoring/generate_drift_report.py
```

Xem kết quả:
```bash
type monitoring\reports\drift_report.json
```

Kết quả mẫu:
```json
{
  "generated_at": "2025-01-15T10:30:00",
  "features_analyzed": 5,
  "drifted_features": ["area", "location_encoded"],
  "details": {
    "area": {"z_score": 3.2, "drifted": true},
    "bedrooms": {"z_score": 0.5, "drifted": false}
  }
}
```

### Bước 7: Đọc tài liệu chiến lược retrain

Mở `docs/retraining-trigger.md` — mô tả:
- Khi nào cần retrain (drift phát hiện, performance giảm, dữ liệu mới đủ lớn)
- Quy trình retrain tự động (CT pipeline)
- Rollback strategy nếu model mới kém hơn

### Bước 8: Chạy Prometheus + Grafana với dashboard có sẵn

Repo đã có sẵn dashboard `monitoring/grafana/dashboards/model-api-overview.json` và cấu hình
provisioning, nên Grafana sẽ tự nạp data source + dashboard khi khởi động — không cần click tay.

Giữ API chạy — **phải bind `0.0.0.0`** thì Prometheus trong container mới gọi được:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Rồi bật hai container:

```bash
docker network create mlops-lab

docker run -d --name mlops-prom --network mlops-lab --network-alias prometheus -p 9090:9090 \
  --add-host=host.docker.internal:host-gateway \
  -v "$PWD/infra/prometheus.local.yml:/etc/prometheus/prometheus.yml:ro" \
  -v "$PWD/monitoring/prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro" \
  prom/prometheus

docker run -d --name mlops-graf --network mlops-lab -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  -v "$PWD/monitoring/grafana/provisioning:/etc/grafana/provisioning:ro" \
  -v "$PWD/monitoring/grafana/dashboards:/etc/grafana/dashboards:ro" \
  grafana/grafana
```

> `--network-alias prometheus` là bắt buộc: data source trong
> `monitoring/grafana/provisioning/datasources/datasources.yml` trỏ tới `http://prometheus:9090`,
> giống tên service ở stack Compose buổi 08. Thiếu alias thì Grafana báo lỗi DNS.
>
> PowerShell: thay `$PWD` bằng `${PWD}`, và bỏ dấu `\` xuống dòng (viết mỗi lệnh trên một dòng)
> hoặc dùng backtick `` ` `` để nối dòng.

Kiểm tra:

1. Prometheus: http://localhost:9090 → **Status → Targets** → job `model-api` phải ở trạng thái **UP**
2. Grafana: http://localhost:3000 (`admin` / `admin`) → menu **Dashboards** → thư mục **MLOps** →
   **Model API — Overview (Buổi 07)**
3. Sinh traffic để dashboard có dữ liệu:

```bash
# vài request hợp lệ
for i in 1 2 3 4 5; do python scripts/sample_predict.py; done

# một request sai schema -> 422
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" \
  -d '{"area": "abc", "bedrooms": 3, "bathrooms": 2, "age": 10, "floors": 1, "location": "98178"}'

# một request có location chưa từng xuất hiện khi train -> 400
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" \
  -d '{"area": 2000, "bedrooms": 3, "bathrooms": 2, "age": 10, "floors": 1, "location": "downtown"}'
```

Hai request cuối cố tình gây lỗi để panel lỗi và donut status code đổi màu.
Lưu ý `location` trong dataset `kc_house_data.csv` là **zipcode** (ví dụ `98178`), không phải tên khu.

Dashboard gồm:

| Panel | Ý nghĩa | PromQL chính |
|-------|---------|--------------|
| Trạng thái API | Prometheus có scrape được không | `up{job="model-api"}` |
| Request rate | Throughput tổng | `sum(rate(http_requests_total[5m]))` |
| Error rate | Tỷ lệ 4xx/5xx, xanh → cam (>1%) → đỏ (>10%) | `sum(rate(http_requests_total{status=~"4..\|5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| Latency trung bình | `sum / count` của duration | `rate(http_request_duration_seconds_sum[5m]) / rate(..._count[5m])` |
| Tổng số prediction | Số lần `/predict` trả 200 | `sum(http_requests_total{path="/predict",status="200"})` |
| Request rate theo endpoint | Biểu đồ vùng xếp chồng theo `path` | `sum by (path) (rate(http_requests_total[1m]))` |
| Latency theo endpoint | Kèm ngưỡng đỏ 500ms | `sum by (path) (rate(..._sum[1m])) / sum by (path) (rate(..._count[1m]))` |
| Phân bố status code | Donut, 200 xanh / 4xx cam / 5xx đỏ | `sum by (status) (http_requests_total)` |
| Request lỗi theo thời gian | Bar chart 4xx/5xx | `sum by (status) (rate(http_requests_total{status=~"4..\|5.."}[1m]))` |
| Logs model-api | Hàng thu gọn, cần Loki (buổi 08) | `{container=~".*model-api.*"}` |

Biến `Endpoint` ở đầu dashboard cho phép lọc theo `/predict`, `/health`, `/model-info`.

> **Lưu ý về p95**: `app/metrics.py` hiện chỉ expose `_sum` và `_count`, chưa có histogram
> bucket, nên chỉ tính được latency trung bình. Muốn có p95 (`histogram_quantile`) thì phải
> thêm bucket — xem bài tập 2.

Muốn import thủ công (không dùng provisioning): Grafana → **Dashboards → New → Import** →
**Upload JSON file** → chọn `monitoring/grafana/dashboards/model-api-overview.json` → chọn
data source Prometheus.

### Bước 9: Khám phá Prometheus UI — 4 màn hình cần xem

Grafana chỉ là lớp vẽ; muốn hiểu số liệu từ đâu ra thì phải mở Prometheus
(http://localhost:9090). Đi theo đúng 4 mục dưới đây, mỗi mục trả lời một câu hỏi.

#### 9.1. Status → Target health — Prometheus *kéo*, app không *đẩy*

Chỉ có một dòng: job `model-api`, endpoint `http://host.docker.internal:8000/metrics`,
State **UP**, cột "Last scrape" đếm lại mỗi 5 giây.

Điều cần nhấn: Prometheus chủ động gọi HTTP vào `/metrics` theo `scrape_interval` — app hoàn
toàn không biết Prometheus tồn tại.

**Demo sống**: tắt `uvicorn` (Ctrl+C) → refresh trang → State chuyển **DOWN** kèm
`connection refused` → bật lại → **UP**. Đây luôn là chỗ debug đầu tiên khi dashboard trống.

#### 9.2. Tab Query — từ metric thô đến chỉ số nghiệp vụ

Prometheus UI **không tự chạy query nào**, phải gõ rồi bấm **Execute** (hoặc Shift+Enter),
nên mở lên thấy "No data queried yet" là bình thường. Nếu bảng trả về rỗng, sinh traffic
trước bằng `python scripts/sample_predict.py`.

Gõ lần lượt 4 câu sau, mỗi câu dạy một khái niệm:

**(1) Metric thô — counter và labels**

```promql
http_requests_total
```

Tab **Table** hiện mỗi tổ hợp label là một dòng riêng: `{method="POST", path="/predict",
status="200"}`. Mỗi tổ hợp = một time series độc lập.

**(2) `rate()` — counter chỉ tăng nên số tuyệt đối vô nghĩa**

```promql
rate(http_requests_total[1m])
```

`http_requests_total` chỉ tăng từ lúc API khởi động, nên "21" không nói lên điều gì.
`rate(...[1m])` cho biết **req/s trung bình trong 1 phút gần nhất**. Chuyển sang tab **Graph**
để thấy đường biểu diễn.

**(3) `sum by` — gộp bớt chiều label**

```promql
sum by (path) (rate(http_requests_total[1m]))
```

Bỏ qua `method` và `status`, chỉ giữ lại `path` → ra đúng 3 đường: `/health`, `/predict`,
`/model-info`. Đây chính là panel "Request rate theo endpoint" trên Grafana.

**(4) Ghép hai query thành chỉ số nghiệp vụ — error rate**

```promql
sum(rate(http_requests_total{status=~"4..|5.."}[5m]))
  / clamp_min(sum(rate(http_requests_total[5m])), 0.0001)
```

Tử số là request lỗi, mẫu số là tổng request. `status=~"4..|5.."` là regex match.
`clamp_min` chặn mẫu số bằng 0 khi không có traffic (nếu không sẽ ra `NaN`).
Kết quả `0.077` nghĩa là 7,7% request đang lỗi.

Muốn thấy số đổi màu: gửi vài request lỗi rồi chạy lại query (xem lệnh curl ở Bước 8).

#### 9.3. Tab Alerts + Status → Rule health — alert cũng chỉ là PromQL

Tab **Alerts** liệt kê 3 rule nạp từ `monitoring/prometheus/alerts.yml`. Mỗi rule có một
trong ba trạng thái:

| Trạng thái | Nghĩa là |
|---|---|
| **Inactive** (xám) | Biểu thức đang sai — mọi thứ bình thường |
| **Pending** (vàng) | Biểu thức đã đúng nhưng chưa đủ thời gian `for:` |
| **Firing** (đỏ) | Đã đúng liên tục đủ `for:` — cảnh báo thật sự phát ra |

Ý nghĩa của `for:`: nó lọc nhiễu. Một spike 10 giây không đáng đánh thức ai lúc 3 giờ sáng.

**Demo sống**: tắt `uvicorn`, chờ khoảng 1 phút → rule `ServiceDown`
(`up{job="model-api"} == 0`, `for: 1m`) chuyển vàng rồi đỏ ngay trên màn hình.

`Status → Rule health` để xác nhận file rule đã nạp thành công và không lỗi cú pháp.

#### 9.4. Status → Configuration & TSDB status — thứ đang chạy là file đã mount

**Configuration** in ra đúng nội dung file mà container đang dùng — tức là
`infra/prometheus.local.yml` đã mount vào `/etc/prometheus/prometheus.yml`, **không phải**
file trong repo. Sửa file xong phải restart container thì mới có hiệu lực.

**TSDB status** cho thấy Prometheus lưu dữ liệu vào time-series database cục bộ (retention
mặc định 15 ngày) và số lượng series đang giữ — dẫn tự nhiên sang khái niệm **cardinality**:
đừng bao giờ đặt label kiểu `user_id` hay `request_id`, vì mỗi giá trị sinh ra một series mới
và sẽ làm nổ bộ nhớ.

> **Về việc "lưu query" trong Prometheus**: Prometheus không có nút save query như Grafana.
> Query gõ ở tab Query là ad-hoc, mất khi refresh. Muốn lưu vĩnh viễn có hai cách:
> **alert rules** (`alerts.yml` — đã có sẵn) hoặc **recording rules** (đặt tên cho query,
> Prometheus tính sẵn theo chu kỳ để dashboard gọi lại bằng tên ngắn).

Sau khi xem hết 4 mục này, quay lại Grafana và đối chiếu: mỗi panel trên dashboard chính là
một trong các query vừa gõ, chỉ khác là đã được vẽ sẵn và tô màu theo ngưỡng.

Dọn dẹp sau lab:

```bash
docker rm -f mlops-prom mlops-graf
docker network rm mlops-lab
```

### Bước 10: Đọc cấu hình Loki + Promtail

Mở `infra/promtail.yml` và (nếu có) cấu hình Loki. Hiểu luồng:

```
API logs (file/stdout) → Promtail → Loki → Grafana Explore (LogQL)
```

Chạy đủ stack Loki/Promtail cùng Compose ở **buổi 08**. Buổi này tập trung đọc config + biết chỗ gắn labels/job.

---

## Chi tiết code

### `app/metrics.py`

```python
from prometheus_client import Counter, Histogram, generate_latest
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import time

REQUEST_COUNT = Counter(
    "request_count",
    "Tổng số HTTP request",
    ["method", "endpoint", "status"],
)

REQUEST_LATENCY = Histogram(
    "request_latency_seconds",
    "Latency của HTTP request (giây)",
    ["method", "endpoint"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.2, 0.5, 1.0],
)

PREDICTION_COUNT = Counter(
    "prediction_count",
    "Tổng số lần gọi prediction",
)

PREDICTION_LATENCY = Histogram(
    "prediction_latency_seconds",
    "Latency của prediction (giây)",
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25],
)

class PrometheusMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        latency = time.perf_counter() - start

        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.url.path,
            status=response.status_code,
        ).inc()

        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.url.path,
        ).observe(latency)

        return response

async def metrics_endpoint(request: Request) -> Response:
    return Response(
        content=generate_latest(),
        media_type="text/plain",
    )
```

### `app/main.py` — cập nhật

Thêm middleware và JSON logging:

```python
import logging
import json

logging.basicConfig(
    filename="logs/app.log",
    level=logging.INFO,
    format="%(message)s",
)

app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", metrics_endpoint)

@app.post("/predict", response_model=PredictResponse)
def predict(request: PredictRequest):
    prediction, latency_ms = model_holder.predict(request.features)

    logging.info(json.dumps({
        "event": "prediction",
        "features": request.features,
        "prediction": prediction,
        "latency_ms": latency_ms,
        "model_version": model_holder.model_version,
    }))

    PREDICTION_COUNT.inc()
    PREDICTION_LATENCY.observe(latency_ms / 1000)

    return PredictResponse(
        prediction=prediction,
        latency_ms=latency_ms,
        model_version=model_holder.model_version or "unknown",
    )
```

### `monitoring/generate_drift_report.py`

```python
import json
import numpy as np
from datetime import datetime
from pathlib import Path

def compute_stats(values: list[float]) -> dict:
    return {
        "mean": float(np.mean(values)),
        "std": float(np.std(values)),
        "min": float(np.min(values)),
        "max": float(np.max(values)),
    }

def detect_drift(
    baseline_mean: float,
    baseline_std: float,
    current_mean: float,
    threshold: float = 2.0,
) -> tuple[float, bool]:
    if baseline_std == 0:
        return 0.0, False
    z_score = abs(current_mean - baseline_mean) / baseline_std
    return z_score, z_score > threshold

def generate_report(
    baseline_data: dict[str, list[float]],
    current_data: dict[str, list[float]],
    output_path: str = "monitoring/reports/drift_report.json",
):
    details = {}
    drifted_features = []

    for feature in baseline_data:
        baseline_stats = compute_stats(baseline_data[feature])
        current_stats = compute_stats(current_data[feature])

        z_score, drifted = detect_drift(
            baseline_stats["mean"],
            baseline_stats["std"],
            current_stats["mean"],
        )

        details[feature] = {
            "baseline": baseline_stats,
            "current": current_stats,
            "z_score": round(z_score, 4),
            "drifted": drifted,
        }

        if drifted:
            drifted_features.append(feature)

    report = {
        "generated_at": datetime.now().isoformat(),
        "features_analyzed": len(baseline_data),
        "drifted_features": drifted_features,
        "details": details,
    }

    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w") as f:
        json.dump(report, f, indent=2)

    print(f"📊 Báo cáo drift đã được tạo: {output_path}")
    print(f"   Tổng features phân tích: {len(baseline_data)}")
    print(f"   Features bị drift: {drifted_features or 'Không có'}")

    return report
```

### `monitoring/prometheus/alerts.yml`

```yaml
groups:
  - name: model-api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(request_count{status=~"5.."}[2m]))
            /
            sum(rate(request_count[2m]))
          ) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Tỷ lệ lỗi API cao"
          description: "Tỷ lệ lỗi 5xx vượt quá 1% trong 2 phút qua"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(request_latency_seconds_bucket[2m])) > 0.2
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Latency API cao"
          description: "P95 latency vượt quá 200ms trong 2 phút qua"
```

---

## Bài tập sau buổi học

1. **Thêm metric cho model confidence** — tạo thêm Histogram `prediction_confidence` để theo dõi phân phối độ tin cậy của mô hình. Cập nhật endpoint `/predict` để trả về và ghi nhận confidence score.

2. **Thêm panel p95 vào dashboard** — sửa `app/metrics.py` để expose histogram bucket (`http_request_duration_seconds_bucket`), sau đó thêm panel `histogram_quantile(0.95, sum by (le, path) (rate(http_request_duration_seconds_bucket[5m])))` vào `monitoring/grafana/dashboards/model-api-overview.json`. Thêm tiếp một panel phân phối giá trị dự đoán (`prediction_value`).

3. **Mở rộng drift detection** — thêm phương pháp KS test (Kolmogorov-Smirnov) bên cạnh z-score trong `generate_drift_report.py`. So sánh kết quả hai phương pháp.

4. **Viết alert cho model drift** — thêm rule trong `alerts.yml` cảnh báo khi `prediction_latency` tăng đột biến (> 500ms) hoặc khi tỷ lệ prediction có giá trị bất thường (ngoài khoảng mong đợi).

---

## Buổi tiếp theo

**Buổi 08 — Tích hợp End-to-End với Docker Compose**: Tích hợp tất cả thành phần (PostgreSQL, MinIO, MLflow, FastAPI, Prometheus, Loki, Promtail, Grafana) vào một stack duy nhất bằng Docker Compose. Khởi động toàn bộ hệ thống bằng một lệnh và chạy full flow: train → register → predict → monitor.
