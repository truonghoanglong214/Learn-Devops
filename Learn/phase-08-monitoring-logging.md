# Phase 8 — Monitoring & Logging

## Mục tiêu
- Biết hệ thống "khỏe hay ốm" theo thời gian thực.
- Thu thập, xem và truy vết log tập trung.
- Nhận cảnh báo khi có sự cố — chuyển từ bị động sang chủ động.

---

## Kiến thức sẽ học

- **Vì sao cần observability** (3 trụ cột: Metrics, Logs, Traces).
- **Prometheus** (thu thập metrics, exporter, PromQL cơ bản).
- **Grafana** (dashboard, trực quan hóa).
- **Logging tập trung** (Loki + Promtail, hoặc ELK/EFK) — gom log mọi service về một nơi.
- **Alerting** (Alertmanager, ngưỡng cảnh báo, gửi thông báo).
- **Health check & uptime monitoring**.
- **Khái niệm SLI/SLO/SLA** — ngôn ngữ vận hành doanh nghiệp.

---

## Kiến thức chi tiết

### 1. Observability vs Monitoring

#### Sự khác biệt cốt lõi

**Monitoring** là "biết điều gì đang xảy ra":
- Track các metrics đã được định nghĩa trước (predefined metrics).
- Tạo alert khi một giá trị vượt ngưỡng đã cài sẵn.
- Ví dụ: "CPU > 80% thì cảnh báo", "số lỗi > 10/phút thì alert".
- Phù hợp với những vấn đề đã biết trước (known unknowns).
- Câu hỏi trả lời được: "Hệ thống có đang bị sự cố không?"

**Observability** là "hiểu tại sao điều đó xảy ra":
- Có đủ dữ liệu để debug ngay cả những vấn đề chưa từng gặp (unknown unknowns).
- Không cần biết trước cần hỏi câu gì — hệ thống cung cấp đủ context để điều tra.
- Câu hỏi trả lời được: "Tại sao user X bị lỗi lúc 3:15 sáng ngày hôm qua?"
- Đòi hỏi thiết kế từ đầu, không phải add-on sau.

**Kết luận thực tế:** Monitoring là tập con của Observability. Một hệ thống có thể được monitor tốt nhưng vẫn thiếu observability nếu không có đủ context khi debug.

#### 3 Pillars of Observability

**Pillar 1: Metrics (số liệu theo thời gian)**
- Là các số được đo tại từng thời điểm, lưu dạng time series.
- Hiệu quả: tốn ít storage, query nhanh, aggregate dễ.
- Ví dụ: `http_requests_total`, `cpu_usage_percent`, `memory_bytes`.
- Công cụ: Prometheus, InfluxDB, Datadog, CloudWatch.
- Giới hạn: không cho biết tại sao một request cụ thể bị lỗi.

**Pillar 2: Logs (sự kiện)**
- Là bản ghi chi tiết từng sự kiện xảy ra trong hệ thống.
- Chứa context phong phú: user ID, request body, stack trace.
- Giúp debug vấn đề cụ thể của một request, một user.
- Ví dụ: `ERROR 2024-01-15 03:15:42 - User 1234 checkout failed: DB timeout`.
- Công cụ: Loki, Elasticsearch, Splunk, CloudWatch Logs.
- Giới hạn: tốn storage, tốn CPU để index, khó aggregate.

**Pillar 3: Traces (hành trình của request)**
- Theo dõi một request đi qua các service khác nhau như thế nào.
- Đặc biệt quan trọng trong microservices — biết service nào chậm.
- Mỗi trace gồm nhiều span (đơn vị công việc trong một service).
- Ví dụ: Request A -> API Gateway (10ms) -> Auth Service (5ms) -> Product Service (200ms) -> DB (180ms).
- Công cụ: Jaeger, Zipkin, Tempo (Grafana), AWS X-Ray.
- Giới hạn: overhead khi instrument, phức tạp hơn metrics/logs.

**Trong phase này:** Tập trung vào Metrics (Prometheus) và Logs (Loki). Traces sẽ học ở phase nâng cao hơn.

---

#### SLI / SLO / SLA

Đây là ngôn ngữ chung giữa kỹ thuật và business. Hiểu rõ 3 khái niệm này giúp làm việc chuyên nghiệp hơn.

**SLI (Service Level Indicator) — Chỉ số đo lường thực tế**
- Là metric cụ thể đo chất lượng dịch vụ.
- Ví dụ: "Trong 30 ngày qua, 99.2% request trả về thành công (status 2xx)."
- Ví dụ: "P99 latency của API checkout là 420ms."
- SLI phải là số đo được, không phải cảm giác.

**SLO (Service Level Objective) — Mục tiêu nội bộ**
- Là target mà team kỹ thuật tự đặt ra, thường cao hơn SLA.
- Ví dụ: "99.9% requests thành công trong bất kỳ cửa sổ 30 ngày nào."
- Ví dụ: "P99 latency < 500ms."
- SLO là cam kết với chính mình — khi vi phạm, team phải điều tra.
- Đặt SLO quá cao (99.999%) gây tốn kém và không thực tế.
- Đặt SLO quá thấp (90%) không bảo vệ được user experience.

**SLA (Service Level Agreement) — Hợp đồng với khách hàng**
- Là cam kết chính thức với user/customer, thường có penalty nếu vi phạm.
- Thường lỏng hơn SLO để có buffer (SLO = SLA + safety margin).
- Ví dụ: SLA = 99.5% availability, SLO nội bộ = 99.9%.
- Nếu SLO bị vi phạm nhưng SLA chưa — team có thời gian fix trước khi ảnh hưởng hợp đồng.

**Error Budget — Ngân sách lỗi**
- Công thức: `Error Budget = 100% - SLO`
- Ví dụ: SLO = 99.9% -> Error Budget = 0.1% = 43.8 phút downtime/tháng.
- Error budget là "tiền" để tiêu cho incidents, deployments và experiments.
- Nếu còn nhiều budget: team có thể deploy thường xuyên, thử nghiệm táo bạo.
- Nếu gần hết budget: team phải ưu tiên reliability, giảm tốc độ deploy.
- Error budget tạo ra ngôn ngữ chung giữa Dev (muốn ship nhanh) và Ops (muốn ổn định).

**SLO ví dụ cho ShopLite:**
```
SLO 1: Availability
  SLI: (successful_requests / total_requests) * 100
  Target: >= 99.5% trong cửa sổ 30 ngày
  Error Budget: 0.5% = 3.65 giờ/tháng

SLO 2: Latency
  SLI: P99 latency của /api/products và /api/checkout
  Target: P99 < 500ms, P95 < 200ms
  Đo trong: mọi cửa sổ 5 phút

SLO 3: Error Rate
  SLI: (5xx_requests / total_requests) * 100
  Target: < 0.1% trong mọi cửa sổ 1 giờ
```

---

### 2. Prometheus Chi Tiết

#### Architecture và Pull Model

Prometheus sử dụng **pull model** — Prometheus chủ động đi lấy metrics từ các target, không phải target push về Prometheus.

**Tại sao pull model tốt hơn push:**
- Prometheus biết ngay khi target down (scrape thất bại = target có vấn đề).
- Prometheus kiểm soát tần suất scrape — không bị overwhelmed khi target đột ngột push quá nhiều.
- Dễ debug: có thể curl trực tiếp `http://target:port/metrics` để xem data.
- Không cần target biết Prometheus ở đâu — chỉ cần expose HTTP endpoint.

**Flow hoạt động:**
```
Target (backend, DB, host)
  └── expose /metrics endpoint (HTTP)
        └── Prometheus scrape mỗi 15s
              └── lưu vào TSDB (Time Series Database)
                    └── Grafana query qua PromQL
                    └── Alertmanager nhận firing alerts
```

**Storage:** Prometheus lưu data trên disk dưới dạng TSDB. Mặc định retain 15 ngày. Cho long-term storage: dùng Thanos hoặc Cortex (ngoài scope phase này).

---

#### Metric Types

**Counter**
- Chỉ tăng, không giảm (hoặc reset về 0 khi service restart).
- Dùng cho: số requests, số lỗi, số bytes gửi/nhận.
- **Không dùng counter trực tiếp** — dùng `rate()` để tính throughput.
- Ví dụ: `http_requests_total{method="GET", status="200"}`
- Query hữu ích: `rate(http_requests_total[5m])` = requests/second trong 5 phút qua.

**Gauge**
- Có thể tăng hoặc giảm.
- Dùng cho: memory usage, số active connections, queue size, temperature.
- Dùng trực tiếp được — đây là giá trị tại thời điểm hiện tại.
- Ví dụ: `process_resident_memory_bytes`, `node_cpu_seconds_total`.

**Histogram**
- Đo distribution của observations (phân phối giá trị).
- Tự động tạo ra các time series: `_bucket`, `_count`, `_sum`.
- Dùng cho: request duration, request size, response size.
- Buckets là các ngưỡng — ví dụ: bao nhiêu requests hoàn thành trong < 0.1s, < 0.5s, < 1s.
- Dùng `histogram_quantile()` để tính percentile (p50, p95, p99).
- **Quan trọng:** buckets phải được cấu hình trước, không thể thay đổi sau.

**Summary**
- Tương tự histogram nhưng quantile được tính phía client (trong app).
- Nhược điểm: không thể aggregate nhiều instance.
- Ít dùng trong practice — histogram linh hoạt hơn nhiều.

---

#### PromQL Cơ Bản

PromQL (Prometheus Query Language) là ngôn ngữ query metrics. Hiểu PromQL là chìa khóa để tạo dashboard và alert có ý nghĩa.

**Selectors cơ bản:**
```promql
# Lấy tất cả time series của metric này
http_requests_total

# Filter theo label
http_requests_total{method="GET"}
http_requests_total{status_code=~"5.."}  # regex: 5xx
http_requests_total{status_code!="200"}  # not equal

# Range vector (dùng với functions)
http_requests_total[5m]  # 5 phút gần nhất
```

**Functions quan trọng:**
```promql
# Rate: throughput trung bình, dùng cho counter
rate(http_requests_total[5m])
# --> requests/second trong 5 phút qua (smooth, recommended)

# Irate: instantaneous rate (nhạy hơn, dùng cho spike detection)
irate(http_requests_total[5m])

# Increase: tổng tăng trong khoảng thời gian
increase(errors_total[1h])
# --> tổng số lỗi trong 1 giờ qua

# Sum, avg, min, max, count (aggregation operators)
sum(rate(http_requests_total[5m])) by (status_code)
# --> request rate, group theo status code

avg(node_cpu_seconds_total) by (cpu)

# Quantile từ histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
# --> p99 latency trong 5 phút qua

histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
# --> p50 (median) latency
```

**Queries hữu ích cho ShopLite:**
```promql
# Tổng request rate
sum(rate(http_requests_total[5m]))

# Error rate (%)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# % services đang down
(count(up == 0) / count(up)) * 100

# Memory usage của container (%)
container_memory_usage_bytes{name="shoplite-backend"}
  / container_spec_memory_limit_bytes{name="shoplite-backend"} * 100

# CPU usage của container
rate(container_cpu_usage_seconds_total{name="shoplite-backend"}[5m]) * 100

# Số active DB connections
pg_stat_activity_count{datname="shoplite"}
```

---

#### Cấu Hình prometheus.yml Đầy Đủ

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # Scrape mỗi 15 giây
  evaluation_interval: 15s      # Evaluate alert rules mỗi 15 giây
  scrape_timeout: 10s           # Timeout cho mỗi scrape request

# Alertmanager integration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Alert rule files
rule_files:
  - /etc/prometheus/rules/*.yml

# Scrape configurations
scrape_configs:
  # Prometheus tự monitor chính nó
  - job_name: prometheus
    static_configs:
      - targets:
          - localhost:9090

  # ShopLite Backend (.NET)
  - job_name: shoplite-backend
    static_configs:
      - targets:
          - backend:8080
    metrics_path: /metrics
    scrape_interval: 10s       # Override global interval cho service quan trọng

  # Node Exporter (host metrics)
  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100

  # cAdvisor (container metrics)
  - job_name: cadvisor
    static_configs:
      - targets:
          - cadvisor:8080
    metric_relabel_configs:
      # Bỏ những container metrics không cần thiết để giảm cardinality
      - source_labels: [container_label_com_docker_compose_service]
        regex: ""
        action: drop

  # PostgreSQL Exporter
  - job_name: postgres-exporter
    static_configs:
      - targets:
          - postgres-exporter:9187

  # Redis Exporter
  - job_name: redis-exporter
    static_configs:
      - targets:
          - redis-exporter:9121

  # Blackbox Exporter (HTTP probe)
  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module:
        - http_2xx
    static_configs:
      - targets:
          - http://backend:8080/health
          - http://backend:8080/ready
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

---

### 3. Exporters Quan Trọng

Exporters là các process trung gian, expose metrics từ các hệ thống không hỗ trợ Prometheus natively dưới dạng `/metrics` endpoint.

#### node_exporter (port 9100)

Cung cấp metrics về **host machine** (server/VM, không phải container):

```
# Metrics quan trọng
node_cpu_seconds_total          # CPU time theo mode (user, system, idle, iowait)
node_memory_MemAvailable_bytes  # RAM available
node_memory_MemTotal_bytes      # Tổng RAM
node_filesystem_avail_bytes     # Disk space còn lại
node_filesystem_size_bytes      # Tổng disk size
node_network_receive_bytes_total # Network traffic received
node_network_transmit_bytes_total # Network traffic sent
node_load1, node_load5, node_load15 # Load average
node_disk_read_bytes_total      # Disk read throughput
node_disk_written_bytes_total   # Disk write throughput
```

**Queries hữu ích:**
```promql
# CPU usage %
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# RAM usage %
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage %
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Network throughput (bytes/sec)
rate(node_network_receive_bytes_total[5m])
```

---

#### cAdvisor (port 8080)

Cung cấp metrics về **từng Docker container**:

```
# Metrics quan trọng
container_cpu_usage_seconds_total     # CPU usage của container
container_memory_usage_bytes          # Memory đang dùng
container_memory_limit_bytes          # Memory limit (từ docker-compose)
container_spec_memory_limit_bytes     # Memory limit từ spec
container_network_receive_bytes_total # Network in của container
container_network_transmit_bytes_total # Network out của container
container_fs_reads_bytes_total        # Disk read của container
container_fs_writes_bytes_total       # Disk write của container
```

**Chú ý:** cAdvisor tạo ra rất nhiều metrics (high cardinality). Dùng `metric_relabel_configs` trong prometheus.yml để drop những metrics không cần.

---

#### postgres_exporter (port 9187)

```
# Metrics quan trọng
pg_up                                    # PostgreSQL có up không
pg_stat_activity_count                   # Số active connections
pg_stat_database_xact_commit             # Transactions committed
pg_stat_database_xact_rollback           # Transactions rolled back
pg_stat_database_blks_hit                # Cache hits
pg_stat_database_blks_read               # Disk reads (cache miss)
pg_stat_replication_lag                  # Replication lag (nếu có replica)
pg_locks_count                           # Số locks hiện tại
pg_stat_bgwriter_checkpoint_write_time   # Checkpoint write time
```

**Cấu hình postgres_exporter:**
```yaml
# docker-compose service
postgres-exporter:
  image: prometheuscommunity/postgres-exporter:latest
  environment:
    DATA_SOURCE_NAME: "postgresql://monitor_user:password@postgres:5432/shoplite?sslmode=disable"
  ports:
    - "9187:9187"
  networks:
    - monitoring-net
    - app-net
```

---

#### redis_exporter (port 9121)

```
# Metrics quan trọng
redis_up                           # Redis có up không
redis_connected_clients            # Số clients đang kết nối
redis_used_memory_bytes            # Memory đang dùng
redis_maxmemory_bytes              # Memory limit
redis_keyspace_hits_total          # Cache hit count
redis_keyspace_misses_total        # Cache miss count
redis_commands_processed_total     # Tổng commands processed
redis_expired_keys_total           # Số keys đã expired
redis_evicted_keys_total           # Số keys bị evict (do OOM)
```

**Cache hit rate query:**
```promql
rate(redis_keyspace_hits_total[5m])
  / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
  * 100
```

---

#### blackbox_exporter

Probe external endpoints từ bên ngoài — kiểm tra xem service có respond đúng không từ góc nhìn của user:

```yaml
# blackbox.yml config
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: ip4

  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json

  tcp_connect:
    prober: tcp
    timeout: 5s
```

---

### 4. Instrumentation trong .NET Backend

Thêm Prometheus metrics vào backend ASP.NET Core bằng thư viện `prometheus-net`.

#### Cài đặt

```bash
dotnet add package prometheus-net.AspNetCore
```

#### File: Infrastructure/Metrics/AppMetrics.cs

```csharp
using Prometheus;

namespace ShopLite.Infrastructure.Metrics;

public static class AppMetrics
{
    // Counter: tổng số HTTP requests
    public static readonly Counter HttpRequestsTotal = Metrics.CreateCounter(
        "http_requests_total",
        "Tổng số HTTP requests nhận được",
        new CounterConfiguration
        {
            LabelNames = new[] { "method", "route", "status_code" }
        });

    // Histogram: phân phối thời gian xử lý request
    public static readonly Histogram HttpRequestDuration = Metrics.CreateHistogram(
        "http_request_duration_seconds",
        "Thời gian xử lý HTTP request (giây)",
        new HistogramConfiguration
        {
            LabelNames = new[] { "method", "route", "status_code" },
            // Buckets từ 5ms đến 5s — điều chỉnh theo SLO
            Buckets = new[] { 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0 }
        });

    // Gauge: số active connections hiện tại
    public static readonly Gauge ActiveConnections = Metrics.CreateGauge(
        "http_active_connections",
        "Số HTTP connections đang active");

    // Counter: số lần checkout thành công / thất bại
    public static readonly Counter CheckoutTotal = Metrics.CreateCounter(
        "checkout_total",
        "Tổng số lần checkout",
        new CounterConfiguration
        {
            LabelNames = new[] { "status" }  // success, failure
        });

    // Histogram: database query duration
    public static readonly Histogram DbQueryDuration = Metrics.CreateHistogram(
        "db_query_duration_seconds",
        "Thời gian thực hiện database query",
        new HistogramConfiguration
        {
            LabelNames = new[] { "query_type", "table" },
            Buckets = new[] { 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0 }
        });
}
```

#### File: Infrastructure/Metrics/MetricsMiddleware.cs

```csharp
using System.Diagnostics;

namespace ShopLite.Infrastructure.Metrics;

public class MetricsMiddleware
{
    private readonly RequestDelegate _next;

    public MetricsMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        AppMetrics.ActiveConnections.Inc();
        var sw = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();

            // Normalize route để tránh high cardinality: /products/123 -> /products/{id}
            var route = context.GetRouteTemplate() ?? context.Request.Path.Value ?? "unknown";
            var method = context.Request.Method;
            var status = context.Response.StatusCode.ToString();

            AppMetrics.HttpRequestsTotal.WithLabels(method, route, status).Inc();
            AppMetrics.HttpRequestDuration.WithLabels(method, route, status).Observe(sw.Elapsed.TotalSeconds);
            AppMetrics.ActiveConnections.Dec();
        }
    }
}
```

#### Program.cs — Đăng ký middleware và metrics endpoint

```csharp
using Prometheus;
using ShopLite.Infrastructure.Metrics;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
// ... các services khác

var app = builder.Build();

app.UseRouting();

// Custom metrics middleware đo thời gian và đếm requests
app.UseMiddleware<MetricsMiddleware>();

// Expose /metrics endpoint để Prometheus scrape (port 8080)
app.MapMetrics();

app.MapControllers();
app.Run();
```

#### Sử dụng trong business logic

```csharp
using ShopLite.Infrastructure.Metrics;

public class CheckoutService
{
    public async Task<OrderResult> ProcessCheckoutAsync(CheckoutRequest request)
    {
        try
        {
            var result = await _orderRepository.CreateAsync(request);
            AppMetrics.CheckoutTotal.WithLabels("success").Inc();
            return result;
        }
        catch (Exception)
        {
            AppMetrics.CheckoutTotal.WithLabels("failure").Inc();
            throw;
        }
    }
}
```

#### Đo thời gian database query

```csharp
public class ProductRepository
{
    public async Task<List<Product>> GetAllAsync()
    {
        // NewTimer() tự động record duration khi Dispose (using block kết thúc)
        using var timer = AppMetrics.DbQueryDuration
            .WithLabels("select", "products")
            .NewTimer();

        return await _context.Products.ToListAsync();
    }
}
```

---

### 5. Grafana Setup và Dashboard

#### Cài đặt Data Sources

Sau khi Grafana start (port 3001), vào Settings > Data Sources:

**Thêm Prometheus:**
```
Name: Prometheus
URL: http://prometheus:9090
Access: Server (default)
Scrape interval: 15s
```

**Thêm Loki:**
```
Name: Loki
URL: http://loki:3100
Access: Server (default)
```

---

#### Dashboard ShopLite — Mô Tả Chi Tiết

Tổ chức dashboard theo rows để dễ đọc:

**Row 1: Overview (Stat panels — nhìn nhanh tình trạng)**
```
Panel 1: Service Uptime (30 days)
  Query: avg_over_time(up{job="shoplite-backend"}[30d]) * 100
  Unit: percent, Thresholds: green > 99.5, yellow > 99, red <= 99

Panel 2: Total Requests (last 1h)
  Query: sum(increase(http_requests_total[1h]))
  Unit: short (requests)

Panel 3: Error Rate (last 5m)
  Query: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
  Unit: percent, Thresholds: green < 1, yellow < 5, red >= 5

Panel 4: P99 Latency (last 5m)
  Query: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) * 1000
  Unit: milliseconds, Thresholds: green < 200, yellow < 500, red >= 500
```

**Row 2: Traffic (Time series panels)**
```
Panel 5: Request Rate by Status Code
  Query A: sum(rate(http_requests_total{status_code=~"2.."}[5m])) - label: 2xx
  Query B: sum(rate(http_requests_total{status_code=~"4.."}[5m])) - label: 4xx
  Query C: sum(rate(http_requests_total{status_code=~"5.."}[5m])) - label: 5xx
  Unit: requests/sec

Panel 6: Active Connections
  Query: http_active_connections
  Unit: connections
```

**Row 3: Performance (Latency analysis)**
```
Panel 7: Latency Percentiles Over Time
  Query A: histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) * 1000 - label: P50
  Query B: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) * 1000 - label: P95
  Query C: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) * 1000 - label: P99
  Unit: milliseconds

Panel 8: Request Duration Heatmap
  Query: sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
  Visualization: Heatmap — giúp thấy distribution thay đổi theo thời gian
```

**Row 4: Infrastructure (Container resources)**
```
Panel 9: CPU Usage by Container
  Query: sum(rate(container_cpu_usage_seconds_total{image!=""}[5m])) by (name) * 100
  Unit: percent

Panel 10: Memory Usage by Container
  Query: container_memory_usage_bytes{image!=""} / 1024 / 1024
  Unit: MiB

Panel 11: Memory Usage % (vs limit)
  Query: container_memory_usage_bytes / container_spec_memory_limit_bytes * 100
  Unit: percent, Thresholds: green < 70, yellow < 85, red >= 85

Panel 12: Network Traffic
  Query A: sum(rate(container_network_receive_bytes_total[5m])) by (name) - label: receive
  Query B: sum(rate(container_network_transmit_bytes_total[5m])) by (name) - label: transmit
  Unit: bytes/sec
```

**Row 5: Database & Cache**
```
Panel 13: PostgreSQL Active Connections
  Query: pg_stat_activity_count{datname="shoplite"}
  Thresholds: green < 50, yellow < 80, red >= 100

Panel 14: PostgreSQL Cache Hit Rate
  Query: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) * 100
  Unit: percent — should be > 95%

Panel 15: Redis Memory Usage
  Query: redis_used_memory_bytes / 1024 / 1024
  Unit: MiB

Panel 16: Redis Cache Hit Rate
  Query: rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100
  Unit: percent — should be > 90%
```

---

#### Pre-built Dashboards (Import từ grafana.com)

Thay vì tự tạo, import dashboard có sẵn:

```
Dashboard ID 1860: Node Exporter Full
  - Đầy đủ host metrics: CPU, RAM, disk, network, load average
  - Cần thêm variable: $node cho multiple hosts

Dashboard ID 14114: cAdvisor exporter
  - Container-level metrics: CPU, memory, network per container
  - Variable: $container để filter theo container name

Dashboard ID 9628: PostgreSQL Database
  - Connections, transactions, locks, cache hit rate
  - Cần postgres_exporter đang chạy

Dashboard ID 763: Redis Dashboard
  - Memory, commands, keyspace, clients
  - Cần redis_exporter đang chạy
```

**Cách import:**
1. Grafana > Dashboards > Import
2. Nhập Dashboard ID > Load
3. Chọn Data Source > Import

---

#### Dashboard as Code (GitOps)

Export dashboard JSON để track bằng git:
1. Dashboard > Share > Export > Save to file
2. Đặt vào `monitoring/grafana/dashboards/shoplite.json`
3. Cấu hình Grafana provisioning để tự load:

```yaml
# monitoring/grafana/provisioning/dashboards/default.yml
apiVersion: 1

providers:
  - name: ShopLite Dashboards
    orgId: 1
    folder: ShopLite
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

---

### 6. Logging với Loki

#### Loki vs ELK Stack

| Tiêu chí | Loki | ELK (Elasticsearch + Logstash + Kibana) |
|---|---|---|
| Index strategy | Chỉ index labels | Full-text index content |
| Storage cost | Thấp (3-10x rẻ hơn) | Cao |
| RAM requirement | Thấp (vài GB) | Cao (8-16GB+ cho production) |
| Search capability | Label filter + regex | Full-text search mạnh |
| Setup complexity | Đơn giản | Phức tạp |
| Query language | LogQL (tương tự PromQL) | Elasticsearch DSL / KQL |
| Grafana integration | Native (same vendor) | Qua plugin |
| Phù hợp khi | Budget limited, team nhỏ | Cần full-text search, compliance |

**Kết luận:** Với ShopLite (project cá nhân / startup nhỏ), Loki là lựa chọn tốt hơn vì đơn giản và tiết kiệm resource.

---

#### Cấu Hình Loki

```yaml
# monitoring/loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

limits_config:
  retention_period: 744h  # 31 ngày

compactor:
  working_directory: /tmp/loki/compactor
  retention_enabled: true
```

---

#### Cấu Hình Promtail (Log Collector)

Promtail là agent chạy trên host, scrape Docker container logs và gửi về Loki.

```yaml
# monitoring/promtail/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Scrape từ Docker daemon qua docker socket
  - job_name: containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: status
            values: [running]

    relabel_configs:
      # Lấy tên container
      - source_labels: [__meta_docker_container_name]
        regex: '/(.+)'
        target_label: container

      # Lấy tên service từ docker-compose label
      - source_labels: [__meta_docker_compose_service]
        target_label: service

      # Lấy project name từ docker-compose
      - source_labels: [__meta_docker_compose_project]
        target_label: project

      # Thêm log stream (stdout/stderr)
      - source_labels: [__meta_docker_container_log_stream]
        target_label: stream

    # Parse log pipeline
    pipeline_stages:
      # Parse JSON logs (nếu app log ra JSON)
      - json:
          expressions:
            level: level
            message: message
            timestamp: time
            request_id: requestId

      # Extract level label từ JSON
      - labels:
          level:
          request_id:

      # Drop DEBUG logs để tiết kiệm storage
      - drop:
          expression: '.*level=debug.*'
          drop_counter_reason: debug_log_dropped

      # Timestamp
      - timestamp:
          source: timestamp
          format: RFC3339Nano
```

---

#### Structured Logging trong .NET (Serilog)

Để Promtail parse được, app nên log ra JSON (structured logging). Dùng **Serilog** — thư viện logging phổ biến nhất cho .NET.

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Formatting.Compact
```

```csharp
// Program.cs
using Serilog;
using Serilog.Formatting.Compact;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", Serilog.Events.LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .WriteTo.Console(new CompactJsonFormatter())  // output JSON — Docker capture stdout
    .CreateLogger();

builder.Host.UseSerilog();

// Request logging middleware (tự động log mọi HTTP request)
app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestId", httpContext.TraceIdentifier);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent);
        diagnosticContext.Set("ClientIp", httpContext.Connection.RemoteIpAddress);
    };
});
```

```csharp
// Sử dụng trong service/controller
public class CheckoutService
{
    private readonly ILogger<CheckoutService> _logger;

    public CheckoutService(ILogger<CheckoutService> logger)
        => _logger = logger;

    public async Task ProcessAsync(int userId, int orderId)
    {
        _logger.LogInformation("Checkout started for user {UserId}", userId);

        try
        {
            // ... business logic
            _logger.LogInformation("Checkout succeeded {UserId} {OrderId}", userId, orderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Checkout failed for user {UserId} order {OrderId}", userId, orderId);
            throw;
        }
    }
}
```

Output JSON mẫu (Promtail sẽ parse được):
```json
{"@t":"2024-01-15T03:15:42.123Z","@mt":"Checkout succeeded {UserId} {OrderId}","UserId":456,"OrderId":789,"level":"Information"}
{"@t":"2024-01-15T03:15:42.500Z","@mt":"Checkout failed for user {UserId} order {OrderId}","UserId":456,"OrderId":789,"@x":"System.Exception: DB timeout...","level":"Error"}
```

---

#### LogQL Queries

LogQL là ngôn ngữ query log trong Loki, cú pháp tương tự PromQL.

```logql
# Xem tất cả logs của backend service
{service="backend"}

# Filter logs chứa "ERROR"
{service="backend"} |= "ERROR"

# Loại trừ logs chứa "healthcheck"
{service="backend"} != "healthcheck"

# Parse JSON và filter theo level
{service="backend"} | json | level="error"

# Parse JSON và filter slow requests (duration > 500ms)
{service="backend"} | json | duration > 500

# Regex filter
{service="backend"} |~ "timeout|connection refused"

# Tính error rate từ logs (metric query)
rate({service="backend"} |= "ERROR" [5m])

# Đếm log volume theo service (cho alert)
sum(count_over_time({project="shoplite"} [1h])) by (service)

# Tìm logs của một request cụ thể (trace qua requestId)
{service=~"backend|worker"} | json | requestId="abc-123-xyz"

# Top errors (dùng với log panel trong Grafana)
topk(10, count_over_time({service="backend"} | json | level="error" [1h]))
```

---

### 7. Alertmanager

#### Alert Rules trong Prometheus

Tạo file `monitoring/prometheus/rules/shoplite.yml`:

```yaml
groups:
  - name: shoplite-availability
    interval: 30s
    rules:
      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m            # Phải true liên tục 1 phút mới fire (tránh flapping)
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Service {{ $labels.job }} is DOWN"
          description: "Instance {{ $labels.instance }} (job: {{ $labels.job }}) has been down for more than 1 minute."
          runbook: "https://wiki.shoplite.internal/runbooks/service-down"

      # Tất cả instances của một service down
      - alert: AllInstancesDown
        expr: count(up{job="shoplite-backend"} == 1) == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "ALL backend instances are DOWN"

  - name: shoplite-performance
    rules:
      # Error rate cao
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))
          * 100 > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Error rate {{ $value | humanize }}% exceeds 5%"
          description: "5xx error rate has been above 5% for more than 2 minutes."

      # Error rate rất cao (critical)
      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))
          * 100 > 20
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Error rate {{ $value | humanize }}% exceeds 20%"

      # P99 latency cao
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency {{ $value | humanizeDuration }} exceeds 1s"

  - name: shoplite-resources
    rules:
      # Memory usage cao
      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{name=~"shoplite.*"}
          / container_spec_memory_limit_bytes{name=~"shoplite.*"}
          * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} memory usage {{ $value | humanize }}%"

      # Memory usage rất cao — OOM sắp xảy ra
      - alert: CriticalMemoryUsage
        expr: |
          container_memory_usage_bytes{name=~"shoplite.*"}
          / container_spec_memory_limit_bytes{name=~"shoplite.*"}
          * 100 > 95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Container {{ $labels.name }} OOM imminent"

      # Disk sắp đầy
      - alert: DiskAlmostFull
        expr: |
          (1 - node_filesystem_avail_bytes{mountpoint="/"}
          / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage {{ $value | humanize }}% on {{ $labels.instance }}"

      # CPU usage cao kéo dài
      - alert: HighCpuUsage
        expr: |
          100 - (avg by (instance)
            (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage {{ $value | humanize }}% on {{ $labels.instance }}"

  - name: shoplite-database
    rules:
      # PostgreSQL quá nhiều connections
      - alert: PostgresHighConnections
        expr: pg_stat_activity_count{datname="shoplite"} > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL has {{ $value }} active connections"

      # Redis memory gần đầy
      - alert: RedisMemoryHigh
        expr: redis_used_memory_bytes / redis_maxmemory_bytes * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage {{ $value | humanize }}%"

  - name: shoplite-slo
    rules:
      # SLO Error Budget sắp cạn (còn < 10% trong 30 ngày)
      - alert: ErrorBudgetBurning
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status_code=~"2.."}[30d]))
              / sum(rate(http_requests_total[30d]))
            )
          ) / (1 - 0.995) > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "SLO Error budget > 90% consumed in 30 days"
          description: "At current rate, SLO will be breached. Slow down deployments and focus on reliability."
```

---

#### Cấu Hình Alertmanager

```yaml
# monitoring/alertmanager/alertmanager.yml
global:
  # Thời gian resolve tự động nếu không nhận được alert trong thời gian này
  resolve_timeout: 5m
  # SMTP (nếu dùng email)
  # smtp_smarthost: smtp.gmail.com:587
  # smtp_from: alerts@shoplite.com
  # smtp_auth_username: alerts@shoplite.com
  # smtp_auth_password: app_password

# Routing tree: quyết định alert nào đi đến receiver nào
route:
  # Group alerts có cùng labels này lại (gửi 1 notification thay vì nhiều)
  group_by: [alertname, service, severity]

  # Chờ 30s để gom các alerts liên quan trước khi gửi
  group_wait: 30s

  # Thời gian chờ giữa các notifications cho cùng group
  group_interval: 5m

  # Nhắc lại nếu alert vẫn còn firing
  repeat_interval: 4h

  # Default receiver
  receiver: discord-warning

  # Routes phụ cho các severity khác nhau
  routes:
    - matchers:
        - severity="critical"
      receiver: discord-critical
      group_wait: 10s    # Gửi nhanh hơn cho critical
      repeat_interval: 1h

    - matchers:
        - severity="warning"
      receiver: discord-warning
      repeat_interval: 4h

    # Silence database alerts ngoài giờ hành chính
    # (ví dụ — bỏ comment khi cần)
    # - matchers:
    #     - alertname=~"Postgres.*"
    #   receiver: discord-warning
    #   active_time_intervals: [business-hours]

receivers:
  - name: discord-critical
    discord_configs:
      - webhook_url: "${DISCORD_WEBHOOK_CRITICAL}"
        title: "CRITICAL ALERT: {{ .GroupLabels.alertname }}"
        message: |
          **Status:** {{ .Status | toUpper }}
          **Severity:** CRITICAL
          {{ range .Alerts }}
          **Alert:** {{ .Annotations.summary }}
          **Description:** {{ .Annotations.description }}
          **Started:** {{ .StartsAt | since }}
          {{ end }}

  - name: discord-warning
    discord_configs:
      - webhook_url: "${DISCORD_WEBHOOK_WARNING}"
        title: "[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}"
        message: |
          {{ range .Alerts }}
          {{ .Annotations.summary }}
          {{ end }}

# Thời gian hoạt động (dùng với active_time_intervals)
# time_intervals:
#   - name: business-hours
#     time_intervals:
#       - weekdays: [monday:friday]
#         times:
#           - start_time: 09:00
#             end_time: 18:00

inhibit_rules:
  # Nếu có alert critical, suppress warning cùng alertname
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: [alertname, service]
```

---

### 8. docker-compose.monitoring.yml Đầy Đủ

```yaml
# docker-compose.monitoring.yml
# Chạy: docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

networks:
  monitoring-net:
    driver: bridge
  # app-net được define trong docker-compose.yml chính

volumes:
  prometheus-data:
    driver: local
  grafana-data:
    driver: local
  loki-data:
    driver: local
  alertmanager-data:
    driver: local

services:
  # ---- Prometheus ----
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: shoplite-prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./monitoring/prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --storage.tsdb.retention.size=10GB
      - --web.console.libraries=/etc/prometheus/console_libraries
      - --web.console.templates=/etc/prometheus/consoles
      - --web.enable-lifecycle          # Cho phép reload config qua API
      - --web.enable-admin-api
    ports:
      - "9090:9090"
    networks:
      - monitoring-net
      - app-net                         # Cần access vào app containers để scrape
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # ---- Grafana ----
  grafana:
    image: grafana/grafana:10.4.2
    container_name: shoplite-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin123}
      GF_SERVER_ROOT_URL: http://localhost:3001
      GF_INSTALL_PLUGINS: grafana-piechart-panel,grafana-clock-panel
      GF_FEATURE_TOGGLES_ENABLE: publicDashboards
      GF_AUTH_ANONYMOUS_ENABLED: false
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3001:3000"                     # 3001 tránh conflict với React dev server
    networks:
      - monitoring-net
    depends_on:
      prometheus:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:3000/api/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # ---- Loki ----
  loki:
    image: grafana/loki:3.0.0
    container_name: shoplite-loki
    restart: unless-stopped
    volumes:
      - ./monitoring/loki/loki-config.yml:/etc/loki/local-config.yaml:ro
      - loki-data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    networks:
      - monitoring-net
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:3100/ready || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # ---- Promtail ----
  promtail:
    image: grafana/promtail:3.0.0
    container_name: shoplite-promtail
    restart: unless-stopped
    volumes:
      - ./monitoring/promtail/promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Cần read Docker socket
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring-net
    depends_on:
      loki:
        condition: service_healthy
    user: root                          # Cần root để đọc Docker socket

  # ---- Alertmanager ----
  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: shoplite-alertmanager
    restart: unless-stopped
    environment:
      DISCORD_WEBHOOK_CRITICAL: ${DISCORD_WEBHOOK_CRITICAL}
      DISCORD_WEBHOOK_WARNING: ${DISCORD_WEBHOOK_WARNING}
    volumes:
      - ./monitoring/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
      - --web.external-url=http://localhost:9093
    ports:
      - "9093:9093"
    networks:
      - monitoring-net
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ---- Node Exporter ----
  node-exporter:
    image: prom/node-exporter:v1.8.0
    container_name: shoplite-node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.rootfs=/rootfs
      - --path.sysfs=/host/sys
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    ports:
      - "9100:9100"
    networks:
      - monitoring-net
    pid: host                           # Cần access vào host PID namespace

  # ---- cAdvisor ----
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: shoplite-cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring-net
    devices:
      - /dev/kmsg

  # ---- Postgres Exporter ----
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: shoplite-postgres-exporter
    restart: unless-stopped
    environment:
      DATA_SOURCE_NAME: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - monitoring-net
      - app-net
    depends_on:
      - postgres                        # Defined in docker-compose.yml chính

  # ---- Redis Exporter ----
  redis-exporter:
    image: oliver006/redis_exporter:v1.60.0
    container_name: shoplite-redis-exporter
    restart: unless-stopped
    environment:
      REDIS_ADDR: "redis://redis:6379"
      REDIS_PASSWORD: ${REDIS_PASSWORD:-}
    ports:
      - "9121:9121"
    networks:
      - monitoring-net
      - app-net
    depends_on:
      - redis                           # Defined in docker-compose.yml chính
```

---

### 9. Health Check Endpoints trong Backend

Health check endpoints là chuẩn mực trong production — Kubernetes, load balancer và monitoring tools đều dùng chúng.

#### Phân biệt Liveness vs Readiness

**Liveness probe** (`/health`):
- Câu hỏi: "Process này có còn sống không?"
- Trả về 200 nếu process đang chạy bình thường.
- KHÔNG check dependencies (DB, Redis) — chỉ check internal state.
- Nếu fail: container bị restart.
- Ví dụ fail case: deadlock, memory leak gây unresponsive.

**Readiness probe** (`/ready`):
- Câu hỏi: "Service này có sẵn sàng nhận traffic không?"
- Check tất cả dependencies: DB connection, Redis, external APIs.
- Nếu fail: container bị remove khỏi load balancer (không restart).
- Dùng khi deploy: service mới start nhưng DB chưa kết nối được -> không nhận traffic.

#### Implementation trong ASP.NET Core

ASP.NET Core có built-in health check system. Không cần viết routes thủ công.

```bash
dotnet add package AspNetCore.HealthChecks.NpgSql
dotnet add package AspNetCore.HealthChecks.Redis
dotnet add package AspNetCore.HealthChecks.UI.Client
```

```csharp
// Program.cs — Đăng ký health checks
builder.Services.AddHealthChecks()
    // Liveness: không check gì cả (chỉ cần app respond là sống)
    // Readiness: check database + redis (tag "ready")
    .AddNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: new[] { "ready" });

// ...

// Liveness probe: /health — 200 nếu process đang chạy
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => false  // bỏ qua dependencies — chỉ check process còn sống
});

// Readiness probe: /ready — 200 nếu tất cả dependencies OK, 503 nếu có lỗi
app.MapHealthChecks("/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

Response `/health` (liveness):
```json
{"status":"Healthy"}
```

Response `/ready` (readiness) khi healthy:
```json
{
  "status": "Healthy",
  "totalDuration": "00:00:00.0041234",
  "entries": {
    "database": { "status": "Healthy", "duration": "00:00:00.0032100" },
    "redis":    { "status": "Healthy", "duration": "00:00:00.0009134" }
  }
}
```

Response `/ready` khi database lỗi (trả về HTTP 503):
```json
{
  "status": "Unhealthy",
  "entries": {
    "database": { "status": "Unhealthy", "description": "An error occurred while executing the health check.", "exception": "Connection refused" },
    "redis":    { "status": "Healthy" }
  }
}
```

#### Cấu Hình trong docker-compose

```yaml
# Trong docker-compose.yml chính — service backend
backend:
  image: shoplite-backend:latest
  healthcheck:
    test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s    # Cho app thời gian khởi động trước khi check
  # ...
```

---

#### Prometheus Blackbox Probe cho Health Endpoints

Kết hợp blackbox_exporter để Prometheus monitor health endpoints từ bên ngoài:

```yaml
# Trong prometheus.yml — scrape_configs
- job_name: shoplite-health-probes
  metrics_path: /probe
  params:
    module:
      - http_2xx
  static_configs:
    - targets:
        - http://backend:8080/health
        - http://backend:8080/ready
        - http://frontend:80
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

**Query kiểm tra kết quả probe:**
```promql
# Probe thành công hay không (1 = ok, 0 = fail)
probe_success{job="shoplite-health-probes"}

# HTTP status code của probe
probe_http_status_code{job="shoplite-health-probes"}

# SSL certificate hết hạn trong vòng 7 ngày
probe_ssl_earliest_cert_expiry - time() < 86400 * 7
```

---

#### Cấu Trúc Thư Mục Monitoring

```
monitoring/
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       ├── shoplite.yml          # Alert rules
│       └── recording_rules.yml   # Pre-computed queries để tăng performance
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml   # Auto-configure Prometheus + Loki
│   │   └── dashboards/
│   │       └── default.yml       # Dashboard provider config
│   └── dashboards/
│       ├── shoplite-overview.json
│       ├── shoplite-infrastructure.json
│       └── shoplite-slo.json
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── alertmanager/
│   └── alertmanager.yml
└── blackbox/
    └── blackbox.yml
```

---

## Kỹ năng đạt được
- Dựng stack giám sát hoàn chỉnh.
- Tạo dashboard và cảnh báo có ý nghĩa.
- Truy vết sự cố qua log tập trung.

---

## Thực hành

**Môi trường:** Prometheus + Grafana + Loki chạy bằng Docker Compose.

**Lab:**
- Thêm Prometheus + Grafana vào hệ thống, expose metrics của backend.
- Tạo dashboard: CPU, RAM, request rate, error rate, latency.
- Gom log các container về Loki, xem trong Grafana.
- Cấu hình cảnh báo (vd: error rate cao -> gửi thông báo).

**Công cụ:** Prometheus, Grafana, Loki/Promtail, Alertmanager, node_exporter, cAdvisor.

---

## Bài tập

- **Bắt buộc:** Dashboard Grafana hiển thị sức khỏe hệ thống ShopLite.
- **Nâng cao:** Cấu hình alert khi service sập hoặc error rate vượt ngưỡng, gửi về email/Discord/Telegram.
- **Mô phỏng doanh nghiệp:** Định nghĩa SLO cho ShopLite (vd: 99% request < 300ms) và tạo dashboard theo dõi nó.

---

## Deliverable
- [ ] Stack monitoring + logging trong Compose.
- [ ] Dashboard Grafana (xuất file JSON).
- [ ] Cấu hình alert hoạt động.

---

## Tiêu chí hoàn thành
- [ ] Nhìn dashboard biết ngay hệ thống có khỏe không.
- [ ] Tự gây sự cố (tắt service) và nhận được cảnh báo.
- [ ] Truy được log của một request lỗi qua hệ thống log tập trung.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
