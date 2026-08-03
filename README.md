# Hệ thống giám sát tài nguyên Cloud

Hệ thống demo giám sát tài nguyên máy chủ và container theo thời gian thực, xây dựng bằng
Prometheus + Grafana, chạy hoàn toàn bằng Docker Compose trên một máy.

Đồ án học phần **Điện toán đám mây**.

## Tính năng

- Thu thập số liệu CPU, RAM, Disk, Network của host và của từng container
- Lưu trữ dạng time-series, truy vấn bằng PromQL
- Dashboard trực quan hóa theo thời gian thực
- Cảnh báo tự động 2 mức (warning / critical) khi tài nguyên vượt ngưỡng

## Kiến trúc

```
Node Exporter (host) ─┐
                        ├─► Prometheus ──► Grafana (Dashboard)
cAdvisor (container) ──┘        │
                                 └──► Alertmanager ──► kênh thông báo
```

Prometheus scrape số liệu theo mô hình **Pull** với chu kỳ 15 giây, lưu vào TSDB nội bộ.
Grafana đọc lại qua Data Source và vẽ Dashboard. Khi một biểu thức PromQL vượt ngưỡng đủ lâu,
Prometheus kích hoạt alert và đẩy sang Alertmanager để định tuyến thông báo theo mức độ.

## Thành phần

| Service       | Port | Vai trò                                  |
|---------------|------|------------------------------------------|
| prometheus    | 9090 | Scrape, lưu trữ time-series, đánh giá rule |
| node-exporter | 9100 | Metric CPU/RAM/Disk/Network của host      |
| cadvisor      | 8080 | Metric tài nguyên từng container          |
| grafana       | 3000 | Dashboard trực quan hóa                   |
| alertmanager  | 9093 | Nhận, gom nhóm và định tuyến cảnh báo      |

Tất cả chạy trong cùng một Docker network tên `monitoring`, giao tiếp với nhau bằng tên service.

## Yêu cầu

- Docker và Docker Compose
- 5 port trống: 9090, 9100, 8080, 3000, 9093

## Chạy hệ thống

```bash
git clone https://github.com/Ltuan126/Monitoring.git
cd Monitoring
docker compose up -d
```

Kiểm tra sau khi khởi động:

| Địa chỉ | Nội dung |
|---|---|
| http://localhost:9090/targets | Danh sách target, tất cả phải ở trạng thái **UP** |
| http://localhost:9090/alerts | Các Alert rule đang được đánh giá |
| http://localhost:3000 | Grafana — đăng nhập `admin` / `admin` |
| http://localhost:9093 | Alertmanager — cảnh báo đang hoạt động |

Data Source và Dashboard được nạp tự động qua cơ chế provisioning của Grafana, không cần cấu hình tay.

Dừng hệ thống:

```bash
docker compose down
```

## Cấu trúc thư mục

```
Monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml           # scrape config, alerting, rule_files
│   └── rules/alerts.yml         # ngưỡng cảnh báo CPU / RAM / Disk
├── alertmanager/
│   └── alertmanager.yml         # route theo severity + receiver
├── grafana/provisioning/
│   ├── datasources/datasource.yml
│   └── dashboards/
│       ├── dashboards.yml
│       └── json/resource-overview.json
└── docs/screenshots/
```

## Quy ước kỹ thuật

- Grafana trỏ Data Source tới `http://prometheus:9090` — dùng tên service, không dùng `localhost`
- Data Source đặt tên `Prometheus` với `uid: prometheus` cố định để Dashboard JSON tham chiếu được
- Prometheus trỏ `alerting.alertmanagers` tới `alertmanager:9093`
- Nhãn `severity` trong Alert rule chỉ dùng `warning` hoặc `critical`
- Dùng đúng tên metric gốc của Node Exporter: `node_cpu_seconds_total`,
  `node_memory_MemAvailable_bytes`, `node_memory_MemTotal_bytes`, `node_filesystem_avail_bytes`,
  `node_filesystem_size_bytes`, `node_network_receive_bytes_total`, `node_network_transmit_bytes_total`

## Ghi chú

**Trên Windows/macOS:** Docker chạy container trong một máy ảo Linux, nên Node Exporter đọc
`/proc` và `/sys` của **máy ảo đó** chứ không phải của hệ điều hành chủ:

- **CPU** — số nhân trùng với máy thật, phần trăm sử dụng phản ánh sát thực tế
- **RAM** — là RAM cấp cho máy ảo, thường chỉ bằng một nửa RAM máy thật
- **Disk** — các ổ của Windows được mount vào máy ảo qua giao thức `9p` tại `/mnt/host/c`,
  `/mnt/host/e`... nên dung lượng đọc được **là số liệu thật của ổ đĩa Windows**

Muốn giám sát đầy đủ CPU/RAM của Windows thì cần `windows_exporter` chạy trực tiếp trên host.

**Về secret:** không commit bot token, SMTP password hay webhook thật vào `alertmanager.yml`.
Dùng giá trị giả trong file được commit và chỉ điền giá trị thật ở máy local.

## Nhóm thực hiện

Nguyễn Lê Tuấn · Vũ Minh Quang · Lê Hoàng
