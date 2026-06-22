# 🐳 AIoT Dockerized Multi-Model Inference Service — Lab 5 v4

<div align="center">

![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker)
![ONNX](https://img.shields.io/badge/ONNX-SqueezeNet1.1-blueviolet?logo=onnx)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-Orchestration-2496ED?logo=docker)

**Đóng gói một AI inference service nhiều loại input (telemetry JSON + ảnh upload) bằng Docker**
Từ chạy local bằng FastAPI → kiểm thử từng endpoint → build Docker image → chạy container → Docker Compose

</div>

---

## 📌 Tổng quan

Một model AI dù chính xác đến đâu cũng không có giá trị thực tế nếu không thể chạy ổn định, chạy lặp lại được trên máy khác, và dễ kiểm thử. Lab 5 không tập trung train lại model — trọng tâm là biến các model đã có (anomaly rule-based, forecast baseline, vision SqueezeNet ONNX) thành một **service có thể chạy, quan sát, kiểm thử và đóng gói lại bằng Docker**.

> **Câu hỏi trung tâm:** Model đã có được đóng gói và chạy lại như thế nào để dùng được trong một hệ thống AIoT thật?

```
Model đã có sẵn (anomaly rule, forecast baseline, SqueezeNet ONNX)
  → Đóng gói thành FastAPI service (sensor endpoints + vision endpoints)
  → Chạy local bằng uvicorn, test từng endpoint qua Swagger
  → Build Docker image (Dockerfile)
  → Chạy container (docker run), kiểm tra port mapping + volume mount
  → Chạy lại bằng Docker Compose (docker-compose.yml)
  → Quan sát logs, so sánh chạy local và chạy Docker
```

---

## 🧩 Kiến trúc service

Service có 2 nhóm endpoint phục vụ 2 loại input khác nhau: telemetry JSON (kế thừa tinh thần Lab 3 và Lab 4) và ảnh upload (model vision mới của Lab 5).

### Nhóm endpoint Telemetry (sensor)

| Endpoint | Method | Ý nghĩa |
|---|---|---|
| `/health` | GET | Kiểm tra service còn hoạt động, vision model đã load chưa |
| `/model-info` | GET | Thông tin model sensor + vision đang dùng |
| `/detect-anomaly` | POST | Phát hiện bất thường bằng z-score, kế thừa tinh thần Lab 3 |
| `/forecast` | POST | Dự báo giá trị tiếp theo bằng moving average baseline, kế thừa tinh thần Lab 4 |
| `/predict-risk` | POST | Chuyển giá trị dự báo thành risk level (NORMAL/WARNING/HIGH) + khuyến nghị |

### Nhóm endpoint Vision (ảnh)

| Endpoint | Method | Kết quả trả về |
|---|---|---|
| `/vision/model-info` | GET | Tên model, input size, số class, trạng thái load |
| `/classify-image` | POST | JSON top-k class, confidence, inference time |
| `/classify-image-annotated` | POST | Ảnh PNG có gắn nhãn dự đoán top-1 |
| `/classify-image-demo` | GET | Giao diện web upload ảnh, hiển thị ảnh gốc, ảnh annotated, bảng top-k và JSON response |

**Model ảnh:** SqueezeNet 1.1 ONNX pretrained trên ImageNet-1K (1000 class) — model nhẹ, chạy tốt trên CPU, phù hợp demo trong lớp học. Model **không** được nhúng sẵn trong repo để giữ dung lượng nhỏ, cần tải bằng `python scripts/download_vision_model.py`.

---

## 🏗️ Cấu trúc project

```
AIoT_Dockerized_Inference_Service_Lab5/
├── app/                          # FastAPI application (main.py, schemas, sensor/vision inference)
├── docs/                         # Tài liệu: hướng dẫn chạy, định dạng model, so sánh môi trường Docker
├── models/
│   └── vision/                   # ← KHÔNG push lên GitHub (file .onnx nặng)
│       └── imagenet_classes.txt
├── optional_examples/
├── outputs/                      # Log inference (service_log.csv, vision_inference_log.csv)
├── sample_images/                # Ảnh mẫu để test upload
├── sample_requests/               # JSON mẫu + curl mẫu để gọi API sensor/vision
├── scripts/                      # download_vision_model.py, smoke_test_local.py
├── tests/
├── KQ_IMAGES/                    # Ảnh minh chứng chạy local + Docker
├── .dockerignore
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── README.md
└── RUN_GUIDE.md
```

---

## ⚙️ Cài đặt môi trường local

```bash
# 1. Clone repository
git clone https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5.git
cd AIoT_Dockerized_Inference_Service_Lab5

# 2. Tạo môi trường ảo
python -m venv .venv

# 3. Kích hoạt (Windows)
.venv\Scripts\activate

# 3. Kích hoạt (macOS/Linux)
source .venv/bin/activate

# 4. Cài thư viện
pip install -r requirements.txt

# 5. Tải model ảnh (không nhúng sẵn trong repo)
python scripts/download_vision_model.py
```

Sau bước 5, cần có đủ 2 file:
```
models/vision/squeezenet1.1-7.onnx
models/vision/imagenet_classes.txt
```

---

## ▶️ Chạy local và kiểm thử

```bash
# Smoke test toàn bộ pipeline trước khi chạy server
python scripts/smoke_test_local.py
```

Kết quả mong đợi:
```
GET  /model-info             200 PASS
POST /detect-anomaly         200 PASS
POST /forecast                200 PASS
GET  /vision/model-info       200 PASS
GET  /classify-image-demo     200 PASS
LOCAL_PIPELINE_TEST_PASS
```

```bash
# Chạy server
uvicorn app.main:app --reload
```

Mở:
- `http://127.0.0.1:8000/health` → `service_status: ok`
- `http://127.0.0.1:8000/docs` → Swagger UI
- `http://127.0.0.1:8000/classify-image-demo` → giao diện upload ảnh

### Kết quả thực tế — giao diện upload ảnh (chạy local)

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-18%20183803.png?raw=true" alt="Classify image demo local" width="850"/>
</p>

Ảnh test cho top-1 là **desktop computer (11.3%)** — confidence thấp vì ảnh test là ảnh placeholder mẫu của lab, không phải vật thể thật. JSON response trả về đầy đủ top-5 class kèm `confidence_level: "LOW"` và khuyến nghị `REVIEW_TOP_K_RESULTS` — đúng tinh thần "không tin riêng top-1 khi confidence thấp" của lab.

---

## 🔍 Kiểm thử endpoint Telemetry (Swagger)

### `POST /detect-anomaly`

Input test: nhiệt độ hiện tại 34.0°C trong khi các giá trị gần đây ổn định quanh 27°C.

```json
{
  "target": "temperature",
  "current_value": 34.0,
  "recent_values": [27.1, 27.3, 27.2, 27.4, 27.5],
  "threshold_z": 2.5
}
```

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-18%20101149.png?raw=true" alt="Detect anomaly Swagger result" width="700"/>
</p>

Kết quả: `anomaly_score = 47.376154` (so với `threshold_used = 2.5`) → `is_anomaly: true`, `severity: HIGH`, quyết định `CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK`. Field `safety_note` nhắc rõ: **không tự động điều khiển thiết bị chỉ dựa trên một điểm anomaly.**

### `POST /forecast`

Input giống endpoint trên, dùng moving average baseline.

| Field | Giá trị |
|---|---|
| `predicted_value` | 27.3 |
| `last_value` | 27.5 |
| `forecast_delta` | -0.2 |
| `forecast_horizon_minutes` | 15 |
| `model_version` | `moving_average_baseline_v1` |

### `POST /predict-risk`

Test 2 trường hợp với target CO2, `warning_threshold = 1000`, `high_threshold = 1200`:

| `predicted_value` | `risk_level` | `recommendation` |
|---|---|---|
| 0 | NORMAL | CONTINUE_MONITORING |
| 1300 | HIGH | REQUIRE_HUMAN_CHECK_BEFORE_ACTUATOR_CONTROL |

Kết quả xác nhận endpoint phân biệt đúng mức độ rủi ro và luôn yêu cầu con người kiểm tra trước khi điều khiển thiết bị khi rủi ro cao — không để model tự ý hành động.

---

## 🖼️ Kiểm thử endpoint Vision (Swagger)

### `POST /classify-image`

Test với ảnh `classroom_object.jpg`, kết quả khớp với giao diện demo:

```json
{
  "model_output": {
    "model_name": "squeezenet1.1_onnx_imagenet1k",
    "model_format": "ONNX",
    "predictions": [
      { "rank": 1, "class_name": "desktop computer", "confidence": 0.113012 },
      { "rank": 2, "class_name": "analog clock", "confidence": 0.086008 },
      { "rank": 3, "class_name": "switch", "confidence": 0.049992 }
    ],
    "inference_time_ms": 8.564
  },
  "decision": {
    "confidence_level": "LOW",
    "recommendation": "REVIEW_TOP_K_RESULTS",
    "safety_note": "This is a general ImageNet-1K classifier, not a domain-specific safety, medical, or plant-disease model."
  }
}
```

Lưu ý: lần gọi đầu tiên (qua giao diện demo) mất 52.467ms vì model mới được load vào RAM; lần gọi sau qua Swagger chỉ mất 8.564ms vì model đã nằm sẵn trong bộ nhớ — minh chứng rõ cho việc model nhẹ giúp inference nhanh khi đã warm-up.

### `POST /classify-image-annotated`

Trả về ảnh PNG có gắn nhãn top-1 đè lên ảnh gốc, đã test thành công qua Swagger với cùng ảnh test.

### 🐞 Lỗi đã gặp và cách sửa (debugging note)

Trong lúc test giao diện `/classify-image-demo`, gặp lỗi JavaScript:
```
Uncaught (in promise) ReferenceError: formData is not defined
```

**Nguyên nhân:** trong `app/main.py`, đoạn JavaScript phục vụ giao diện demo có lỗi gõ thiếu chữ — biến được khai báo là `formDa` nhưng dòng sau lại gọi `formData.append(...)` (chưa từng được khai báo).

**Cách sửa:** đổi `const formDa = new FormData();` thành `const formData = new FormData();`, và `body: formDa` thành `body: formData`. Vì `uvicorn --reload` đang chạy, server tự nạp lại code mới ngay sau khi lưu file, không cần khởi động lại thủ công.

---

## 🐳 Build Docker image

```bash
docker build -t lab5-aiot-inference:v4 .
```

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-18%20184645.png?raw=true" alt="Docker images build result" width="700"/>
</p>

Image `lab5-aiot-inference:v4` được tạo thành công, dung lượng **195MB**. Lưu ý ở lần build thứ 2, toàn bộ các bước từ `[2/14]` đến `[14/14]` đều hiện **CACHED**, chỉ mất 1.6s — minh chứng rõ cho cơ chế cache layer của Docker khi không có gì thay đổi trong code.

---

## ▶️ Chạy container

```powershell
docker run --rm --name lab5-aiot-api -p 8000:8000 `
  -v ${PWD}/outputs:/app/outputs `
  -v ${PWD}/models/vision:/app/models/vision `
  lab5-aiot-inference:v4
```

| Thành phần lệnh | Ý nghĩa |
|---|---|
| `--rm` | Tự động xóa container khi dừng |
| `--name lab5-aiot-api` | Đặt tên container để dễ tìm trong Docker Desktop |
| `-p 8000:8000` | Map port 8000 của host vào port 8000 của container |
| `-v ${PWD}/outputs:/app/outputs` | Ghi log từ container ra thư mục `outputs/` trên host |
| `-v ${PWD}/models/vision:/app/models/vision` | Cho container dùng đúng model ONNX đã tải sẵn trên host |

### Docker Desktop — Logs container đang chạy

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-19%20120106.png?raw=true" alt="Docker Desktop logs" width="850"/>
</p>

Log xác nhận container `lab5-aiot-api` khởi động đúng: `Started server process`, `Application startup complete`, `Uvicorn running on http://0.0.0.0:8000`. Sau khi chạy, đã test lại `http://127.0.0.1:8000/docs` và `http://127.0.0.1:8000/classify-image-demo` ngay trên container — cả hai hoạt động giống hệt lúc chạy local, log ghi nhận `GET /health 200 OK` từ địa chỉ nội bộ container (`172.17.0.1`).

---

## 🧱 Chạy bằng Docker Compose

```bash
docker compose up --build
```

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-19%20121008.png?raw=true" alt="Docker compose up build" width="850"/>
</p>

Khác với `docker run`, Compose tự tạo thêm một **Network** riêng ngoài Image và Container — tiện cho việc kết nối nhiều service (database, MQTT broker...) nếu hệ thống mở rộng sau này. Mỗi dòng log có prefix `lab5-aiot-api |` để phân biệt log của từng service khi chạy nhiều container cùng lúc.

```bash
docker compose ps
docker compose down
```

<p align="center">
  <img src="https://github.com/khanhly-dn/AIoT_Dockerized_Inference_Service_Lab5/blob/main/KQ_IMAGES/%E1%BA%A2nh%20ch%E1%BB%A5p%20m%C3%A0n%20h%C3%ACnh%202026-06-19%20121337.png?raw=true" alt="Docker compose ps and down" width="850"/>
</p>

`docker compose ps` xác nhận container đang `Up 10 seconds` với port mapping đúng (`0.0.0.0:8000->8000/tcp`). `docker compose down` dọn sạch cả container lẫn network, không để lại rác.

---

## ⚖️ So sánh chạy Local và chạy Docker

| Tiêu chí | Chạy local bằng Python | Chạy bằng Docker |
|---|---|---|
| Môi trường | Phụ thuộc trực tiếp vào Python/package cài trên máy (`.venv`) | Đóng gói sẵn trong image, không phụ thuộc máy đang chạy |
| Debug code | Sửa file, `uvicorn --reload` tự nạp lại ngay (đã trải nghiệm khi sửa lỗi `formData`) | Phải build lại image mới áp dụng thay đổi, không tự reload |
| Tốc độ khởi động | Nhanh, chỉ cần `pip install` | Lâu hơn ở lần build đầu (tải base image, cài thư viện); các lần sau nhanh nhờ cache (build lần 2 chỉ 1.6s) |
| Ghi log | Ghi thẳng vào `outputs/` trên máy | Cần mount volume (`-v outputs:/app/outputs`) mới giữ được log, nếu không sẽ mất khi container bị xóa |
| Đường dẫn model | Theo filesystem thật trên máy | Theo đường dẫn trong container (`/app/models/vision`), cần mount volume để dùng đúng model |
| Port | Mặc định localhost của uvicorn | Cần khai báo port mapping rõ ràng (`-p 8000:8000`) |
| Chia sẻ cho máy khác | Dễ lỗi version Python, thiếu package, sai đường dẫn | Chỉ cần có Docker, chạy lại y hệt nhờ image đã đóng gói |
| Gần với triển khai thật | Thấp hơn | Cao hơn, nhất là khi dùng Docker Engine/Compose trên server |

---

## 📋 Checklist sản phẩm

| Sản phẩm | Trạng thái |
|---|---|
| Ảnh `/health` khi chạy local | ✅ |
| Ảnh `/classify-image-demo` sau upload local | ✅ |
| Ảnh Docker Desktop tab Images (`lab5-aiot-inference:v4`) | ✅ |
| Ảnh Docker Desktop tab Containers (Running) | ✅ |
| Ảnh Docker Desktop Logs | ✅ |
| Ảnh Swagger `/docs` khi chạy container | ✅ |
| Ảnh `/classify-image-demo` khi chạy container | ✅ |
| File `outputs/service_log.csv` / `vision_inference_log.csv` | ✅ |
| Chạy `docker compose up --build`, `ps`, `down` | ✅ |
| Bảng so sánh local vs Docker | ✅ |
| Trả lời câu hỏi phân tích sau lab | ⏳ Đang hoàn thiện |

---

## 💡 Kết luận & bài học

| Quan sát | Kết luận |
|---|---|
| `anomaly_score = 47.376` >> `threshold = 2.5` → severity HIGH | Endpoint phát hiện đúng bất thường rõ ràng, vẫn yêu cầu con người xác nhận trước khi hành động |
| `risk_level` đổi từ NORMAL → HIGH khi `predicted_value` vượt `high_threshold` | Logic phân loại rủi ro hoạt động đúng theo ngưỡng đã định nghĩa |
| `confidence_level: LOW` ở top-1 chỉ 11.3% | Không nên tin riêng top-1, cần đọc top-k trước khi kết luận — đúng tinh thần SqueezeNet là classifier tổng quát, không phải model chuyên ngành |
| Build lần 2 chỉ 1.6s nhờ cache | Docker cache giúp vòng lặp phát triển (develop → build → test) nhanh hơn nhiều sau lần đầu |
| Local debug nhanh hơn Docker (nhờ `--reload`) | Nên luôn chạy và sửa lỗi ở local trước, chỉ build Docker khi code đã ổn định |

> ⚠️ **Safety note:** Output của các model trong service này (anomaly score, forecast, risk level, vision classification) đều chỉ mang tính tham khảo/hỗ trợ quyết định — không nên dùng để tự động điều khiển thiết bị thật mà không có bước xác nhận của con người.

---

## 🛠️ Công nghệ sử dụng

| Công nghệ | Vai trò |
|---|---|
| Python 3.11 | Ngôn ngữ chính |
| FastAPI | Backend service, Swagger UI tự sinh |
| Uvicorn | ASGI server chạy FastAPI |
| ONNX Runtime | Inference model vision (SqueezeNet 1.1) |
| Pillow | Xử lý ảnh upload, vẽ nhãn annotated |
| Docker | Đóng gói môi trường runtime thành image |
| Docker Compose | Mô tả và chạy service bằng file cấu hình |

---

## 📚 Tài liệu tham khảo

- [Docker Desktop WSL 2 backend](https://docs.docker.com/desktop/features/wsl/)
- [Docker Engine trên Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose documentation](https://docs.docker.com/compose/)
- [FastAPI file upload](https://fastapi.tiangolo.com/tutorial/request-files/)
- [ONNX overview](https://onnx.ai/)

---

<div align="center">
  <sub>Lab 5 v4 — Triển khai, phát triển ứng dụng AI và IoT</sub>
</div>
