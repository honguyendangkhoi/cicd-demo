from pathlib import Path

content = r'''# Docker Lab: Build a Small REST API

## 1. Mục tiêu

Xây dựng một REST API nhỏ bằng **Python Flask** hoặc **Node.js Express**, sau đó đóng gói ứng dụng thành Docker image và chạy container.

### Yêu cầu hoàn thành

- [ ] API có hai endpoint: `/` và `/health`.
- [ ] Docker image được build thành công.
- [ ] Container chạy với port mapping `8080:5000`.
- [ ] `curl http://localhost:8080/health` trả về trạng thái thành công.
- [ ] Dung lượng Docker image nhỏ hơn **200 MB**.

> Trong lab này, ví dụ chính sử dụng Python Flask. Phần Node.js Express là lựa chọn thay thế, không cần triển khai cả hai.

## 2. Cấu trúc thư mục

Tạo thư mục dự án với cấu trúc sau:

```text
api/
├── app.py
├── requirements.txt
└── Dockerfile
```

## 3. Bước 1: Chuẩn bị API

### Lựa chọn A: Python Flask

Tạo file `app.py`:

```bash
cat << 'EOF' > app.py
from flask import Flask, jsonify
import time

app = Flask(__name__)
start_time = time.time()


@app.route("/", methods=["GET"])
def index():
    return jsonify({"message": "Welcome to the API!"}), 200


@app.route("/health", methods=["GET"])
def health():
    return jsonify({
        "status": "ok",
        "uptime_seconds": round(time.time() - start_time, 2)
    }), 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
```

Tạo file `requirements.txt`:

```bash
cat << 'EOF' > requirements.txt
Flask
EOF
```

**Lưu ý:** `requirements.txt` là tên file theo quy ước của Python. Không cần cài `pipreqs` cho ứng dụng nhỏ chỉ sử dụng Flask. Nếu muốn tạo file tự động cho dự án lớn hơn, có thể cân nhắc công cụ này.

### Lựa chọn B: Node.js Express

Nếu chọn Node.js, tạo file `index.js`:

```bash
cat << 'EOF' > index.js
const express = require("express");
const app = express();
const PORT = 5000;

app.get("/", (req, res) => {
  res.status(200).json({
    message: "Welcome to the API!"
  });
});

app.get("/health", (req, res) => {
  res.status(200).json({
    status: "ok",
    uptime_seconds: process.uptime(),
    timestamp: new Date().toISOString()
  });
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Server is running on port ${PORT}`);
});
EOF
```

Tạo `package.json` tối thiểu:

```json
{
  "name": "small-rest-api",
  "version": "1.0.0",
  "private": true,
  "description": "A small API for a Docker lab",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

Tạo `package-lock.json` và cài dependencies bằng npm:

```bash
npm install
```

> Với Node.js, `depcheck` dùng để phát hiện dependencies có thể không được sử dụng; nó không phải công cụ chuẩn để sinh `package.json` hoặc lockfile từ đầu.

## 4. Bước 2: Viết Dockerfile

### Dockerfile cho Python Flask

Tạo file có tên chính xác là `Dockerfile`:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Giải thích:**

| Chỉ thị | Ý nghĩa |
|---|---|
| `FROM` | Chọn image nền Python slim để giảm dung lượng ban đầu. |
| `WORKDIR` | Đặt thư mục làm việc trong container. |
| `COPY` | Sao chép file từ build context vào image. |
| `RUN` | Cài dependencies trong quá trình build image. |
| `EXPOSE` | Khai báo cổng ứng dụng sử dụng trong container. Không tự publish port ra máy host. |
| `CMD` | Lệnh mặc định chạy khi container khởi động. |

`--no-cache-dir` giúp pip không lưu cache package trong image, qua đó giảm dung lượng không cần thiết.

### Dockerfile cho Node.js Express

Nếu sử dụng Node.js, thay Dockerfile bằng:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY index.js .

EXPOSE 5000

CMD ["npm", "start"]
```

Dùng `npm ci` khi đã có `package-lock.json`. Cách này cài dependencies theo lockfile, phù hợp hơn cho một lần build có thể lặp lại.

## 5. Bước 3: Build Docker image

Chạy lệnh tại thư mục dự án:

```bash
docker build -t python-app:1.0 .
```

Trong đó:

- `-t python-app:1.0`: đặt tên image và tag.
- `.`: dùng thư mục hiện tại làm build context.

Nếu Dockerfile nằm ở đường dẫn khác, chỉ định bằng `-f`:

```bash
docker build \
  -f /home/cloud-user/api/Dockerfile \
  -t python-app:1.0 \
  /home/cloud-user/api
```

Cấu trúc lệnh tổng quát:

```bash
docker build -f <đường_dẫn_Dockerfile> -t <tên_image:tag> <đường_dẫn_build_context>
```

> Đường dẫn build context là thư mục Docker được phép lấy file để thực hiện các lệnh `COPY` và `ADD`. Vì vậy, hãy chọn đúng thư mục dự án.

## 6. Bước 4: Chạy container

```bash
docker run -d --name python-api -p 8080:5000 python-app:1.0
```

Ý nghĩa của port mapping:

```text
Máy host:8080  ─────►  Container:5000
```

- `-d`: chạy container ở chế độ nền.
- `--name python-api`: đặt tên container.
- `-p 8080:5000`: chuyển tiếp cổng `8080` trên máy host tới cổng `5000` trong container.

Nếu dùng Node.js, có thể đổi tên image và container tương ứng, ví dụ `node-api:1.0`.

## 7. Bước 5: Kiểm tra kết quả

### 7.1. Kiểm tra endpoint `/`

```bash
curl -i http://localhost:8080/
```

Response dự kiến:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "message": "Welcome to the API!"
}
```

### 7.2. Kiểm tra endpoint `/health`

```bash
curl -i http://localhost:8080/health
```

Response dự kiến:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "status": "ok",
  "uptime_seconds": 12.34
}
```

Giá trị `uptime_seconds` chỉ là ví dụ; thời gian thực tế phụ thuộc vào lúc container được khởi động.

### 7.3. Kiểm tra container

```bash
docker ps
docker logs python-api
```

Nếu API không phản hồi, kiểm tra log và trạng thái container trước khi build lại.

### 7.4. Kiểm tra dung lượng image

```bash
docker image ls python-app:1.0
docker image inspect python-app:1.0 --format='{{.Size}}'
```

Lệnh thứ hai trả về kích thước image theo byte. Đổi sang MiB bằng:

```bash
docker image inspect python-app:1.0 \
  --format='{{.Size}}' | awk '{printf "%.2f MiB\n", $1 / 1024 / 1024}'
```

**Tiêu chí:** kích thước image phải nhỏ hơn `200 MiB` theo kết quả kiểm tra. Image `slim` hoặc `alpine` giúp giảm dung lượng, nhưng vẫn cần đo kết quả thực tế.

## 8. Kết quả mong đợi

- Endpoint `/` trả về thông điệp chào mừng.
- Endpoint `/health` trả về `status: ok`.
- Container chạy và truy cập được qua `localhost:8080`.
- Docker image có dung lượng dưới `200 MiB`.

### Ảnh minh chứng

<img width="661" height="190" alt="image" src="https://github.com/user-attachments/assets/2e6e9c8f-27b7-4bde-b9c3-39598469f1d7" />

<img width="479" height="75" alt="image" src="https://github.com/user-attachments/assets/44f57918-8271-4d25-9568-dee44aa01405" />


## 9. Dọn dẹp sau lab

```bash
docker rm -f python-api
docker image rm python-app:1.0
```

Chỉ chạy các lệnh này khi không còn cần container hoặc image của lab.

---

## Checklist hoàn thành

- [ ] Tạo API với `/` và `/health`.
- [ ] Tạo `requirements.txt` hoặc `package.json` và lockfile tương ứng.
- [ ] Viết Dockerfile sử dụng base image gọn nhẹ.
- [ ] Build image thành công.
- [ ] Chạy container với `-p 8080:5000`.
- [ ] `curl http://localhost:8080/health` trả về `status: ok`.
- [ ] Xác nhận image nhỏ hơn `200 MiB`.
- [ ] Lưu lại log hoặc ảnh minh chứng.
'''
path = Path("/mnt/data/docker-api-lab.md")
path.write_text(content, encoding="utf-8")
print(f"Created: {path} ({path.stat().st_size} bytes)")
