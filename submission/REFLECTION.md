# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Đào Phước Thịnh - 2A202600029
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/TristanDao/Day23-Track2-Observability-Lab-2A202600029

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.3"
  },
  "ram_gb_available": 7.63,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Điều làm tôi ngạc nhiên nhất là khả năng tự động liên kết (exemplars) giữa các điểm dữ liệu trên đồ thị Prometheus trực tiếp sang Jaeger traces. Việc có thể click vào một "spike" trên biểu đồ latency và nhảy ngay đến trace chi tiết giúp giảm MTTR đáng kể so với việc phải tìm kiếm trace ID thủ công trong logs.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

Trace ID: `48639e5122df3ef27dd4c23367e6bba1`

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1547, "trace_id": "48639e5122df3ef27dd4c23367e6bba1", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T02:43:53.163341Z"}
```

### Tail-sampling math

Dựa trên cấu hình `tail_sampling` trong `otel-config.yaml`:
- Giữ lại 100% traces có status `ERROR`.
- Giữ lại 100% traces có latency > 2000ms.
- Giữ lại 1% (`probabilistic: 1`) của các traces "healthy" còn lại.

Giả sử hệ thống có 1000 traces/s, trong đó 5% là lỗi (50 traces) và 95% là bình thường (950 traces). Hệ thống sẽ giữ lại:
`50 (lỗi) + (1% * 950) = 50 + 9.5 = 59.5 traces/s`.
Tỷ lệ lưu trữ tổng cộng khoảng **5.95%**. Điều này giúp tiết kiệm băng thông và dung lượng lưu trữ đáng kể mà không làm mất đi các dữ liệu quan trọng về lỗi.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.798,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.019,
    "kl": 0.032,
    "ks_stat": 0.052,
    "ks_pvalue": 0.011681,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.016,
    "kl": 0.018,
    "ks_stat": 0.056,
    "ks_pvalue": 0.005117,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.849,
    "kl": 13.501,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```
 
 ### Evidently HTML report
 
 Drop `submission/screenshots/evidently-report.png`.

### Which test fits which feature?

- **prompt_length**: Sử dụng **PSI** (Population Stability Index) vì đây là đại lượng đo lường sự thay đổi tổng thể của phân phối dữ liệu đầu vào theo thời gian rất ổn định.
- **embedding_norm**: Sử dụng **KL Divergence** để đo lường mức độ "bất ngờ" (information gain) khi phân phối vector embedding thay đổi, giúp phát hiện model drift tinh tế hơn.
- **response_length**: Sử dụng **KS Test** (Kolmogorov-Smirnov) vì đây là test phi tham số, rất hiệu quả để so sánh hai phân phối liên tục (như số lượng tokens) mà không cần giả định về hình dạng phân phối.
- **response_quality**: Sử dụng **PSI** hoặc **MMD** để theo dõi điểm số chất lượng (0.0 - 1.0), giúp nhận biết khi nào model bắt đầu trả về kết quả kém hơn so với baseline benchmark.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Việc kết nối metric từ Day 19 (Qdrant) là thử thách nhất vì nó yêu cầu phải cấu hình lại `prometheus.yml` để scrape từ `host.docker.internal` do Qdrant chạy bên ngoài network của lab. Ngoài ra, việc map các label từ Qdrant (như collection name) vào dashboard chung yêu cầu sự thống nhất về đặt tên (naming convention) giữa các module khác nhau.
 
 ### Cross-day dashboard (screenshot)
 
 Drop `submission/screenshots/cross-day-dashboard.png`.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất giúp stack này trở nên thực sự hữu ích là việc **tích hợp Trace ID vào cấu trúc log JSON**. Ban đầu, metrics chỉ cho chúng ta biết "hệ thống đang chậm", nhưng không cho biết "tại sao nó chậm". Khi có Trace ID trong log, chúng ta có một "sợi chỉ đỏ" xuyên suốt từ Metrics (biết có lỗi) -> Tracing (biết lỗi ở span nào) -> Logs (biết chính xác thông báo lỗi hoặc payload gây lỗi).

Việc áp dụng **Tail-sampling** cũng là một bước ngoặt lớn về mặt thiết kế. Thay vì lưu trữ mọi thứ một cách mù quáng (head-sampling), việc ưu tiên giữ lại 100% lỗi và các request chậm giúp chúng ta tối ưu hóa chi phí lưu trữ mà vẫn đảm bảo 100% khả năng quan sát (observability) đối với các sự cố nghiêm trọng. Đây chính là concept "Observability is about data quality, not just volume" mà bài giảng đã đề cập.
