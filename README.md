
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

## PHẦN II — THỰC HÀNH: APP MONITOR + ALERT GIÁ VÀNG REALTIME

---

## Kiến trúc hệ thống

```
[API Giá Vàng SJC]
        │
        ▼
  [Node-RED]  ──────────────────────────────► [Telegram Bot]
        │                                        (Alert khi bất thường)
        ├──► [MariaDB]  ──► [Flask API] ──► [Nginx] :80
        └──► [InfluxDB] ──► [Grafana]  ──► [Nginx] /grafana
                                               │
                                          [Frontend HTML]
                                     giavang.nguyentienduc04.io.vn
```

### Danh sách service trong docker-compose.yml

| Service | Image | Vai trò |
|---|---|---|
| nodered | nodered/node-red | Lấy giá vàng, lưu DB, gửi Telegram |
| mariadb | mariadb:10.6 | Lưu giá trị tức thời |
| influxdb | influxdb:2.7 | Lưu lịch sử dữ liệu |
| flask_api | baitap5-flask_api | API trả dữ liệu từ MariaDB |
| grafana | grafana/grafana | Vẽ biểu đồ lịch sử |
| nginx | nginx:alpine | Web server, proxy |

---

## Các bước thực hiện

### Bước 1 — Tạo cấu trúc thư mục

```bash
mkdir ~/baitap5
cd ~/baitap5
mkdir html api nodered
sudo chown -R 1000:1000 ~/baitap5/nodered
```
<img width="1484" height="711" alt="image" src="https://github.com/user-attachments/assets/51bfd8da-9038-4c4d-9caf-14d784304463" />


---

### Bước 2 — Tạo file docker-compose.yml

```bash
nano docker-compose.yml
```

Nội dung file:

```yaml
services:

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "1880:1880"
    volumes:
      - ./nodered:/data
    networks:
      - backend

  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=golddb
      - MYSQL_USER=golduser
      - MYSQL_PASSWORD=goldpass
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - backend

  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=adminpass123
      - DOCKER_INFLUXDB_INIT_ORG=myorg
      - DOCKER_INFLUXDB_INIT_BUCKET=gold_history
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=mytoken123456
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - backend

  flask_api:
    build: ./api
    container_name: flask_api
    restart: unless-stopped
    environment:
      - DB_HOST=mariadb
      - DB_NAME=golddb
      - DB_USER=golduser
      - DB_PASS=goldpass
    depends_on:
      - mariadb
    networks:
      - backend

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_SERVER_ROOT_URL=http://localhost/grafana
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
      - GF_SECURITY_ALLOW_EMBEDDING=true
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
      - GF_AUTH_ANONYMOUS_ORG_NAME=Main Org.
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - backend

  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - flask_api
      - grafana
      - nodered
    networks:
      - backend

networks:
  backend:
    driver: bridge

volumes:
  mariadb_data:
  influxdb_data:
  grafana_data:
```

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/ef515983-b989-488a-a5bf-4862c2de38b3" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/20f5341d-6443-4ad9-af66-860b5c82937a" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d846571f-5682-45cb-be72-b2941de4432c" />


---

### Bước 3 — Tạo Flask API

**api/Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

**api/requirements.txt:**
```
flask
flask-cors
pymysql
```

**api/app.py** — API trả về giá vàng tức thời từ MariaDB:
```python
from flask import Flask, jsonify
from flask_cors import CORS
import pymysql, os

app = Flask(__name__)
CORS(app)

@app.route('/api/gold', methods=['GET'])
def get_gold():
    try:
        db = pymysql.connect(
            host=os.environ.get('DB_HOST', 'mariadb'),
            user=os.environ.get('DB_USER', 'golduser'),
            password=os.environ.get('DB_PASS', 'goldpass'),
            database=os.environ.get('DB_NAME', 'golddb'),
            charset='utf8mb4'
        )
        cursor = db.cursor()
        cursor.execute("SELECT gia_mua, gia_ban, thoi_gian FROM gold_price ORDER BY id DESC LIMIT 1")
        row = cursor.fetchone()
        db.close()
        if row:
            return jsonify({"gia_mua": row[0], "gia_ban": row[1],
                           "thoi_gian": str(row[2]), "status": "ok"})
        return jsonify({"status": "no_data"})
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/2ea22213-b78a-4598-9e3e-410a1540a812" />


---

### Bước 4 — Khởi động hệ thống

```bash
docker compose up -d
docker compose ps
```

<img width="1481" height="716" alt="image" src="https://github.com/user-attachments/assets/4cf22f1c-3169-4dc2-bd33-7caf451f6a1f" />


---

### Bước 5 — Tạo bảng trong MariaDB

```bash
docker compose exec mariadb mariadb -ugolduser -pgoldpass golddb -e "
CREATE TABLE IF NOT EXISTS gold_price (
  id INT AUTO_INCREMENT PRIMARY KEY,
  gia_mua BIGINT,
  gia_ban BIGINT,
  thoi_gian DATETIME DEFAULT CURRENT_TIMESTAMP
);"
```

<img width="1478" height="706" alt="image" src="https://github.com/user-attachments/assets/bd9ecdca-2b15-49c5-8925-f14875e5978d" />


---

### Bước 6 — Cấu hình Node-RED

Truy cập: `[http://<IP>:1880](http://172.19.193.244:1880)`

**6.1 Cài các node cần thiết:**

Menu (☰) → Manage palette → Install:
- `node-red-node-mysql`
- `node-red-contrib-influxdb`
- `node-red-contrib-telegrambot`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/90178459-77cb-4127-881c-c0982f841c5e" />



**6.2 Import flow:**

Menu (☰) → Import → paste JSON flow → Import

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/dc39b663-8a90-44d8-bc6e-38126e326c2c" />


**6.3 Cấu hình node Lưu MariaDB:**

Double-click → điền User: `golduser`, Password: `goldpass`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/c9a8a952-9c13-48aa-90db-2841eec2a3bd" />


**6.4 Cấu hình node Lưu InfluxDB:**

Double-click → URL: `http://influxdb:8086`, Token: `mytoken123456`, Org: `myorg`, Bucket: `gold_history`

<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/6fba9a87-f266-4af7-b0d8-8ee2251944f2" />

**6.5 Cấu hình Telegram bot:**

Double-click node Gửi Telegram → điền Token bot, ChatIds: `-5088493997`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/421d7d85-4404-49d3-b34b-ff6203dd6a57" />


**6.6 Deploy và kiểm tra:**

Nhấn Deploy → node Lưu MariaDB hiển thị "OK"

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/ffeaa228-4426-40df-8f56-1c182c156b1f" />


---

### Bước 7 — Kiểm tra dữ liệu vào MariaDB

```bash
docker compose exec mariadb mariadb -ugolduser -pgoldpass golddb \
  -e "SELECT * FROM gold_price ORDER BY id DESC LIMIT 5;"
```

<img width="1483" height="712" alt="image" src="https://github.com/user-attachments/assets/8dbf1096-9ad9-405b-b106-95ad41739d8c" />


---

### Bước 8 — Cấu hình Grafana

Truy cập: `http://172.19.193.244:3000` 

**8.1 Thêm InfluxDB data source:**

Connections → Data sources → Add → InfluxDB

- Query language: **Flux**
- URL: `http://influxdb:8086`
- Token: `mytoken123456`
- Org: `myorg`
- Bucket: `gold_history`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/70962f39-7e16-43d6-b2c0-cc00f83c4533" />


**8.2 Tạo dashboard:**

New dashboard → Add visualization → paste query Flux:

```flux
from(bucket: "gold_history")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "gold_price")
  |> filter(fn: (r) => r._field == "gia_mua" or r._field == "gia_ban")
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9719cafb-fdb1-4a67-ba01-069324cf19d9" />



**8.3 Bật public dashboard:**

```bash
curl -s -X POST -u admin:admin123 \
  -H "Content-Type: application/json" \
  -d '{"isEnabled":true}' \
  "http://localhost:3000/api/dashboards/uid/<UID>/public-dashboards"
```

<img width="1477" height="707" alt="image" src="https://github.com/user-attachments/assets/edc59a44-3c19-4678-a82f-5697bb75296b" />


---


### Bước 9 — Kết quả giao diện web

Truy cập: ` https://giavang.nguyentienduc04.io.vn`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/02b4a3be-9fa5-46a9-9e9b-db16a8418b99" />

> - Header "Monitor Giá Vàng Thời Gian Thực"
> - Card Giá Mua + Giá Bán hiển thị số
> - Alert đỏ khi giá bất thường
> - Iframe Grafana hiển thị biểu đồ lịch sử

---

### Bước 10 — Test Alert Telegram

Tạm thời đặt ngưỡng LOW = 12,000,000 để trigger alert → Deploy → chờ 30 giây

<img width="1290" height="2796" alt="image" src="https://github.com/user-attachments/assets/4e8b6841-d503-4745-8d77-8b050abbcb42" />


---

### Bước 11 — Xuất và load lại Docker images

**11.1 Dừng hệ thống và xuất images:**

```bash
docker compose down

docker save nodered/node-red mariadb:10.6 influxdb:2.7 \
  grafana/grafana nginx:alpine baitap5-flask_api \
  | gzip > ~/baitap5_images.tar.gz

ls -lh ~/baitap5_images.tar.gz
```

<img width="1479" height="712" alt="image" src="https://github.com/user-attachments/assets/eb8abc03-183a-4128-b440-01dd2a2505db" />


**11.2 Xóa các images:**

```bash
docker rmi nodered/node-red mariadb:10.6 influxdb:2.7 \
  grafana/grafana nginx:alpine baitap5-flask_api

docker images
```

<img width="1512" height="796" alt="image" src="https://github.com/user-attachments/assets/6d2e6215-7cf9-4830-8a9c-80e383eab4e9" />
<img width="1512" height="801" alt="image" src="https://github.com/user-attachments/assets/605faa09-10b8-4355-8b57-e01a3d1cc231" />


**11.3 Load lại từ file nén:**

```bash
docker load < ~/baitap5_images.tar.gz
```

<img width="1512" height="801" alt="image" src="https://github.com/user-attachments/assets/8d470e1c-3661-462a-adeb-8621ba823735" />


**11.4 Khởi động lại và kiểm tra:**

```bash
docker compose up -d
docker compose ps
```
<img width="1479" height="710" alt="image" src="https://github.com/user-attachments/assets/cd124b96-1091-4df3-b5b3-6e678167096a" />


---

### Bước 12 — Tên miền với Cloudflare Tunnel

**12.1 Tạo tunnel trên Cloudflare Zero Trust:**

Vào `dash.cloudflare.com` → Zero Trust → Networks → Connectors → Create a tunnel

Tên tunnel: `baitap5-tunnel`

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/34becb79-ceb9-4612-b85e-f639ac4ab4fa" />



**12.2 Chạy cloudflared trên Ubuntu:**

```bash
docker run -d --name cloudflared \
  --restart unless-stopped \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run \
  --token <TOKEN>
```

**12.3 Cấu hình Route:**

- Subdomain: `giavang`
- Domain: `nguyentienduc04.io.vn`
- Service Type: `HTTP`
- URL: `172.19.193.244:80`



**12.4 Kết quả:**

Truy cập `https://giavang.nguyentienduc04.io.vn` từ bất kỳ thiết bị nào có internet

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fddcfa91-ea00-4b53-88b4-d80cf9674312" />


---

## Tổng kết

| Thành phần | Trạng thái | URL/Ghi chú |
|---|---|---|
| Web frontend |   https://giavang.nguyentienduc04.io.vn |
| API giá vàng |   /api/gold |
| Grafana dashboard |  /grafana |
| Node-RED flow |  Cập nhật 30 giây/lần |
| Alert Telegram |  Bot: @giavang_alert_k58_bot |
| docker save/load |  baitap5_images.tar.gz (852MB) |
| Tên miền HTTPS |  Cloudflare Tunnel |
