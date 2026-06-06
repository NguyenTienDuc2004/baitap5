# BÀI TẬP LỚN - BÀI 5
**Môn:** Phát triển ứng dụng với Mã nguồn mở - TEE0421  
**Sinh viên:** Nguyễn Tiến Đức  
**Repo:** Tài liệu + source code BT5 - Docker Monitor & Alert

---

## MỤC LỤC
- [Phần I – Lý thuyết Docker](#phần-i--lý-thuyết-docker)
  - [1. Docker là gì?](#1-docker-là-gì)
  - [2. Các keyword trong docker-compose.yml](#2-các-keyword-trong-docker-composeyml)
  - [3. Ưu điểm khi triển khai ứng dụng bằng Docker](#3-ưu-điểm-khi-triển-khai-ứng-dụng-bằng-docker)
  - [4. Triển khai lên máy chủ không có Internet](#4-triển-khai-lên-máy-chủ-không-có-internet)
- [Phần II – Thực hành: APP Monitor & Alert giá vàng](#phần-ii--thực-hành-app-monitor--alert-giá-vàng)

---

## Phần I – Lý thuyết Docker

---

### 1. Docker là gì?

**Docker** là nền tảng mã nguồn mở dùng để **xây dựng, đóng gói và triển khai ứng dụng** dưới dạng **container**.  
Container là đơn vị phần mềm nhẹ, độc lập, chứa đầy đủ mọi thứ để chạy ứng dụng: mã nguồn, runtime, thư viện, cấu hình.

#### Kiến trúc Docker

| Thành phần | Vai trò |
|---|---|
| **Docker Client** | Giao diện CLI, gửi lệnh đến Docker Daemon |
| **Docker Daemon** | Tiến trình nền, thực thi các lệnh từ Client |
| **Docker Image** | Bản mẫu (template) chỉ đọc để tạo container |
| **Docker Container** | Instance đang chạy của Image |
| **Docker Registry** | Kho lưu Image (Docker Hub, private registry...) |

#### So sánh Container vs Virtual Machine

| Tiêu chí | Container (Docker) | Virtual Machine |
|---|---|---|
| Khởi động | Vài giây | Vài phút |
| Kích thước | Vài MB → vài trăm MB | Vài GB |
| Cô lập | Cô lập tiến trình (dùng chung OS kernel) | Cô lập hoàn toàn (có OS riêng) |
| Hiệu năng | Gần như native | Chậm hơn do lớp Hypervisor |
| Tính di động | Rất cao (chạy được mọi nơi có Docker) | Thấp hơn (phụ thuộc hypervisor) |

---

### 2. Các keyword trong docker-compose.yml

File `docker-compose.yml` dùng cú pháp **YAML** để định nghĩa các service, network, volume của ứng dụng đa container.

#### Bảng keyword chính

| Keyword | Ý nghĩa | Ví dụ |
|---|---|---|
| `image` | Chỉ định Docker image sẽ dùng để tạo container | `image: nginx:alpine` |
| `build` | Xây dựng image từ Dockerfile trong thư mục chỉ định | `build: ./myapp` |
| `container_name` | Đặt tên cụ thể cho container thay vì tên tự động | `container_name: my_api` |
| `ports` | Ánh xạ cổng giữa host và container `(host:container)` | `ports: ["8080:80"]` |
| `volumes` | Gắn thư mục/volume vào container để lưu trữ bền vững | `volumes: [./data:/var/lib/mysql]` |
| `environment` | Khai báo biến môi trường bên trong container | `environment: [DB_HOST=mariadb]` |
| `env_file` | Đọc biến môi trường từ file `.env` bên ngoài | `env_file: [.env]` |
| `depends_on` | Xác định thứ tự khởi động: service này chờ service kia | `depends_on: [mariadb]` |
| `networks` | Gán service vào một hoặc nhiều mạng Docker | `networks: [backend]` |
| `restart` | Chính sách khởi động lại container khi gặp lỗi | `restart: always` |
| `command` | Ghi đè lệnh mặc định của container | `command: python app.py` |
| `entrypoint` | Ghi đè entrypoint mặc định trong Dockerfile | `entrypoint: /start.sh` |
| `expose` | Khai báo cổng container mở nội bộ (không map ra host) | `expose: ["3000"]` |
| `healthcheck` | Kiểm tra tình trạng hoạt động của container định kỳ | `test: ["CMD","curl","-f","http://localhost"]` |
| `labels` | Gán metadata (nhãn) cho container/service | `labels: ["app=monitor"]` |
| `driver` | Chọn driver cho network hoặc volume | `driver: bridge` |
| `external` | Dùng network/volume đã tồn tại, không tạo mới | `external: true` |
| `ipam` | Cấu hình địa chỉ IP cho network (subnet, gateway) | `subnet: 172.20.0.0/16` |

#### Ví dụ docker-compose.yml đầy đủ

```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:10.6
    container_name: db_container
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=appdb
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3

  myapi:
    build: ./api
    container_name: flask_api
    restart: on-failure
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=mariadb
      - DB_NAME=appdb
    depends_on:
      - mariadb
    networks:
      - backend
      - frontend

  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    networks:
      - frontend
    depends_on:
      - myapi

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge

volumes:
  db_data:
    driver: local
```

> **Lưu ý quan trọng:** Các container cùng network gọi nhau bằng **tên service** (không dùng IP).  
> Ví dụ: `myapi` muốn kết nối MariaDB thì dùng host = `mariadb`, port = `3306`.

---

### 3. Ưu điểm khi triển khai ứng dụng bằng Docker

#### 3.1. Tính nhất quán môi trường (Environment Consistency)
Vấn đề kinh điển *"Works on my machine"* — app chạy tốt trên laptop lập trình viên nhưng lỗi trên server — được Docker giải quyết triệt để. Toàn bộ môi trường runtime được đóng gói vào Image, chạy giống nhau trên mọi hệ điều hành.

#### 3.2. Triển khai nhanh chóng
Chỉ cần **một lệnh** để khởi động toàn bộ hệ thống nhiều service:
```bash
docker compose up -d
```
Thay vì cài thủ công từng phần mềm (Python, Node.js, MariaDB, Nginx,...), chỉ cần Docker + file `docker-compose.yml`.

#### 3.3. Cô lập và bảo mật
- Lỗi ở container này không ảnh hưởng container khác
- Kiểm soát chính xác quyền truy cập mạng giữa các service
- Giới hạn tài nguyên CPU, RAM cho từng container

#### 3.4. Khả năng mở rộng (Scalability)
```bash
docker compose up -d --scale myapi=3   # Chạy 3 instance của myapi
```
Kết hợp load balancer (Nginx), hệ thống xử lý tải tăng đột biến mà không đổi code.

#### 3.5. Quản lý version và rollback
- Mỗi Image có tag version cụ thể (`nginx:1.24`, `nginx:alpine`)
- Rollback nhanh bằng cách `docker compose down` rồi `up` lại với image version cũ

#### 3.6. Tích hợp CI/CD
Docker tích hợp tốt với GitHub Actions, Jenkins, GitLab CI — mỗi lần commit, pipeline tự động build Image mới, test và deploy.

#### Tổng kết

| Ưu điểm | Vấn đề được giải quyết | Kết quả |
|---|---|---|
| Nhất quán môi trường | "Works on my machine" | Chạy đúng ở mọi nơi |
| Triển khai nhanh | Cài đặt thủ công mất nhiều giờ | Chỉ cần 1 lệnh |
| Cô lập service | Xung đột thư viện/phiên bản | Mỗi app có môi trường riêng |
| Dễ scale | Tăng tải phải nâng cấp server | Scale container trong vài giây |
| Rollback dễ dàng | Khó khôi phục khi cập nhật lỗi | Chuyển lại image version cũ |

---

### 4. Triển khai lên máy chủ không có Internet

> **Tình huống:** App đã test OK trên laptop (có Internet). Cần deploy lên máy chủ thực tế **không có kết nối Internet** — không thể `docker pull` trực tiếp.

#### Bước 1 — Trên laptop: Xuất Image ra file

```bash
# Xem danh sách image đang dùng
docker images

# Xuất tất cả image vào một file nén duy nhất
docker save nodered/node-red:latest mariadb:10.6 influxdb:2.7 \
  grafana/grafana:latest nginx:alpine my_flask_api:latest \
  | gzip > all_images.tar.gz
```

#### Bước 2 — Sao chép file sang máy chủ

```bash
# Qua SCP (nếu có mạng nội bộ)
scp all_images.tar.gz user@192.168.1.100:/home/user/deploy/
scp docker-compose.yml user@192.168.1.100:/home/user/deploy/
scp -r ./html ./api ./nginx.conf user@192.168.1.100:/home/user/deploy/

# Hoặc copy qua USB
cp -r /media/usb/deploy /home/user/deploy
```

#### Bước 3 — Trên máy chủ: Load Image từ file

```bash
# Load tất cả image từ file nén
docker load < all_images.tar.gz

# Kiểm tra image đã load thành công
docker images
```

#### Bước 4 — Khởi động ứng dụng

```bash
cd /home/user/deploy

# Khởi động toàn bộ service ở chế độ nền
docker compose up -d

# Kiểm tra trạng thái
docker compose ps
docker compose logs -f
```

#### Bước 5 — Xác nhận hoạt động

```bash
docker ps
curl http://localhost:5000/api/health   # Flask API
curl http://localhost:80                # Nginx frontend
```

#### Tóm tắt các bước

| Bước | Thực hiện tại | Lệnh / Hành động | Kết quả |
|---|---|---|---|
| 1 | Laptop | `docker save ... \| gzip > all_images.tar.gz` | File chứa tất cả image |
| 2 | Laptop → Máy chủ | SCP / USB / mạng nội bộ | Files đã sang máy chủ |
| 3 | Máy chủ | `docker load < all_images.tar.gz` | Image sẵn sàng trong Docker |
| 4 | Máy chủ | `docker compose up -d` | Toàn bộ service đang chạy |
| 5 | Máy chủ | `docker ps` / `curl` / browser | Xác nhận app hoạt động đúng |

> **Lưu ý:** Máy chủ vẫn cần cài **Docker + Docker Compose** trước (qua offline installer hoặc mạng nội bộ).

---

## Phần II – Thực hành: APP Monitor & Alert giá vàng

> 🔨 *Đang thực hiện — sẽ cập nhật kèm ảnh chụp từng bước*

### Kiến trúc hệ thống

```
[API giá vàng]
      │
      ▼
  [Node-RED]  ──────────────────────────────────┐
      │                                          │
      ├──► [MariaDB]  (giá trị tức thời)         │
      │         │                                │
      │         ▼                                │
      │    [Flask API] ◄── AJAX ──[Nginx/Web]    │
      │                                          │
      └──► [InfluxDB] (lịch sử)                  │
                │                                │
                ▼                                │
           [Grafana] ◄── iframe ── [Nginx/Web] ──┘
                                          │
                              [Alert] ──► [Telegram Bot]
```

### Cấu trúc thư mục dự án

```
baitap5/
├── docker-compose.yml
├── nginx.conf
├── html/               # Frontend (HTML + JS + CSS)
│   └── index.html
├── api/                # Flask API
│   ├── Dockerfile
│   └── app.py
├── nodered/            # Node-RED data (flows)
└── README.md           # File này
```

### Các service trong docker-compose.yml

| Service | Image | Cổng | Vai trò |
|---|---|---|---|
| `nodered` | `nodered/node-red` | (nội bộ) | Lấy giá vàng, lưu DB, gửi alert |
| `mariadb` | `mariadb:10.6` | (nội bộ) | Lưu giá trị tức thời |
| `influxdb` | `influxdb:2.7` | (nội bộ) | Lưu lịch sử giá vàng |
| `flask_api` | `python:3.11` | (nội bộ) | API trả dữ liệu tức thời cho web |
| `grafana` | `grafana/grafana` | (nội bộ) | Vẽ biểu đồ lịch sử |
| `nginx` | `nginx:alpine` | `80:80` | Web server, proxy tất cả service |

---

*Cập nhật lần cuối: 06/2026*
