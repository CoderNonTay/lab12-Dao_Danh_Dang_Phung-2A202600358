# Day 12 Lab - Câu Trả Lời Mission

## Part 1: Localhost vs Production

### Exercise 1.1: Các anti-pattern tìm thấy trong `01-localhost-vs-production/develop/app.py`

1. **Hardcode secrets trực tiếp trong source code**
   - `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`
   - `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`
   - Vì sao nguy hiểm: secrets dễ lộ qua Git history, log, ảnh chụp màn hình, hoặc khi chia sẻ code.

2. **Không quản lý cấu hình bằng environment variables**
   - `DEBUG = True` và `MAX_TOKENS = 500` bị cố định trong code.
   - Vì sao nguy hiểm: đổi môi trường (dev/staging/prod) phải sửa code thay vì đổi config.

3. **Cách logging không an toàn**
   - Dùng `print()` debug và còn in ra cả API key.
   - Vì sao nguy hiểm: log thường được gom tập trung; lộ key trong log là sự cố bảo mật.

4. **Không có endpoint health/readiness**
   - Không có `/health` hoặc `/ready`.
   - Vì sao nguy hiểm: platform và load balancer khó phát hiện instance lỗi để restart/khoanh vùng.

5. **Hardcode host và port**
   - Dùng `host="localhost"` và `port=8000` trong `uvicorn.run(...)`.
   - Vì sao nguy hiểm: cloud thường inject `PORT`; app chạy container cần bind `0.0.0.0`.

6. **Bật reloader dành cho development mặc định**
   - Dùng `reload=True`.
   - Vì sao nguy hiểm: auto-reload chỉ phù hợp local dev, không ổn định cho runtime production.

7. **Không có xử lý graceful shutdown**
   - Không có lifecycle/signal handler cho SIGTERM.
   - Vì sao nguy hiểm: request đang xử lý có thể bị rớt khi deploy/restart.

---

### Exercise 1.2: Chạy bản basic

Lệnh đã dùng:

```bash
cd 01-localhost-vs-production/develop
pip install -r requirements.txt
python app.py
```

Lệnh test trên Windows PowerShell:

```powershell
curl.exe -X POST "http://localhost:8000/ask?question=Hello"
```

Nhận xét:
- Bản basic chạy được trên localhost và trả response.
- Tuy nhiên chưa production-ready vì còn hardcode secrets/config, thiếu health checks, và shutdown/logging chưa đạt chuẩn.

---

### Exercise 1.3: Bảng so sánh (`develop/app.py` vs `production/app.py`)

| Tiêu chí | Basic (`develop`) | Advanced (`production`) | Tại sao quan trọng? |
|---------|-------------------|--------------------------|---------------------|
| Nguồn config | Hardcode trực tiếp trong code | Gom config qua `config.py` + env vars | Cho phép đổi môi trường mà không sửa code. |
| Quản lý secrets | Hardcode secrets, thậm chí log ra | Đọc secrets từ env vars, không log secret | Tránh rò rỉ thông tin nhạy cảm, đúng nguyên tắc 12-factor. |
| Host/port binding | Cố định `localhost:8000` | `settings.host`, `settings.port` từ env (mặc định `0.0.0.0`) | Cần thiết để chạy container/cloud đúng cách. |
| Format request `/ask` | Query parameter (`question: str`) | JSON body (`await request.json()`) | API contract rõ ràng, dễ mở rộng payload. |
| Kiểu logging | `print()` debug | Structured JSON logging theo mức độ | Dễ theo dõi/giám sát trong môi trường production. |
| Health check | Không có | Có `GET /health` | Cho phép kiểm tra liveness và tự động restart khi lỗi. |
| Readiness check | Không có | Có `GET /ready` với cờ sẵn sàng | Tránh route traffic vào instance chưa sẵn sàng. |
| Quản lý vòng đời app | Không có startup/shutdown rõ ràng | Có FastAPI lifespan cho startup + shutdown | Khởi tạo và dọn dẹp tài nguyên an toàn hơn. |
| Graceful shutdown | Không xử lý signal | Có SIGTERM handler + shutdown flow | Giảm rớt request khi tắt app/deploy. |
| CORS | Không cấu hình | Có middleware CORS theo config | Cần cho frontend gọi API an toàn. |
| Chế độ runtime | Luôn `reload=True` | Chỉ reload khi `DEBUG=true` | Tránh hành vi không ổn định trên production. |
| Metrics | Không có | Có `GET /metrics` | Hỗ trợ monitoring và vận hành hệ thống. |

---

### Checkpoint 1

- [x] Hiểu vì sao hardcoded secrets là nguy hiểm.
- [x] Hiểu cách env vars giúp cấu hình an toàn khi deploy.
- [x] Hiểu vai trò của health/readiness endpoint trong cloud.
- [x] Hiểu graceful shutdown giúp bảo vệ in-flight requests.

---

### Vì sao mình trả lời theo cách này

Mình trả lời dựa trên việc đọc trực tiếp các file:
- `01-localhost-vs-production/develop/app.py`
- `01-localhost-vs-production/production/app.py`
- `01-localhost-vs-production/production/config.py`

Mục tiêu Part 1 không chỉ là "chạy được", mà là giải thích rõ **vì sao code localhost khác code production**.  
Vì vậy mỗi ý đều gắn chi tiết kỹ thuật với tác động thực tế: bảo mật, độ tin cậy, khả năng deploy và khả năng quan sát hệ thống.

## Part 2: Docker

### Exercise 2.1: Trả lời câu hỏi về Dockerfile cơ bản (`02-docker/develop/Dockerfile`)

1. **Base image là gì?**  
   `python:3.11` (bản full) được dùng làm nền để chạy ứng dụng.

2. **Working directory là gì?**  
   `WORKDIR /app` — toàn bộ lệnh sau đó chạy trong thư mục `/app` bên trong container.

3. **Tại sao COPY requirements.txt trước?**  
   Để tận dụng Docker layer cache. Khi code thay đổi nhưng dependencies không đổi, Docker không cần `pip install` lại, giúp build nhanh hơn.

4. **CMD và ENTRYPOINT khác nhau thế nào?**  
   - `CMD`: lệnh mặc định, có thể bị ghi đè khi chạy `docker run ... <command>`.
   - `ENTRYPOINT`: lệnh chính, thường luôn được giữ cố định; tham số thêm vào sẽ nối vào sau.

---

### Exercise 2.2: Build và chạy bản develop

Lệnh đã chạy:

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run -p 8000:8000 my-agent:develop
```

Kết quả test:

```powershell
curl.exe -X POST "http://localhost:8000/ask?question=What%20is%20Docker%3F"
curl.exe "http://localhost:8000/health"
```

Quan sát:
- Endpoint `/ask` trả về `answer` hợp lệ.
- Endpoint `/health` trả về `status: ok` và `container: true`.
- Kết luận: image develop chạy đúng trong container.

---

### Exercise 2.3: Multi-stage build và so sánh kích thước image

Build advanced image:

```bash
docker build -f "02-docker/production/Dockerfile" -t my-agent:advanced .
```

Kết quả đo image size:
- `my-agent:develop` = **1.66 GB**
- `my-agent:advanced` = **236 MB** (0.236 GB)

Tính toán:
- Dung lượng giảm = `1.66 - 0.236 = 1.424 GB`
- Tỷ lệ giảm = `(1.424 / 1.66) * 100 ≈ 85.8%`

Giải thích vì sao giảm mạnh:
- Bản advanced dùng multi-stage để tách **builder stage** và **runtime stage**.
- Runtime chỉ giữ phần cần để chạy app (packages + source cần thiết), loại bỏ build tools và layer dư thừa.
- Hệ quả: image nhẹ hơn, pull/deploy nhanh hơn và giảm bề mặt tấn công.

---

### Exercise 2.4: Docker Compose stack

Đọc `02-docker/production/docker-compose.yml`, kiến trúc stack gồm:
- `agent`: FastAPI service xử lý API `/ask`, `/health`
- `redis`: cache/session store
- `qdrant`: vector database
- `nginx`: reverse proxy và điểm vào public (`localhost:80`)

Luồng giao tiếp:
1. Client gọi `http://localhost/...`
2. Nginx nhận request và proxy vào `agent:8000`
3. Agent sử dụng Redis/Qdrant qua mạng nội bộ compose

Kết quả chạy stack:
- `docker compose up -d` chạy được các service.
- `docker compose ps` cho thấy:
  - `agent`: Up (healthy)
  - `redis`: Up (healthy)
  - `nginx`: Up
  - `qdrant`: Up (health: starting)

Kết quả test endpoint qua Nginx:

```powershell
curl.exe --% -X POST http://localhost/ask -H "Content-Type: application/json" -d "{\"question\":\"Explain microservices\"}"
```

Response nhận được:
- `{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}`

Kết luận:
- Full stack compose hoạt động.
- Nginx proxy về agent thành công.
- Request POST JSON qua `/ask` hoạt động đúng.

---

### Checkpoint 2

- [x] Hiểu cấu trúc Dockerfile cơ bản.
- [x] Hiểu lợi ích của multi-stage build.
- [x] Chạy được Docker Compose multi-service stack.
- [x] Biết cách debug lỗi build/context/healthcheck trong Compose.

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- **Platform:** Railway
- **Project ID:** `466132bd-99d3-4607-aa75-47f48b13e7b7`
- **Public URL:** `https://agent-ai-production-98ad.up.railway.app/`
- **Trạng thái deploy:** Thành công (build xong, healthcheck pass, container running)

Lệnh chính đã chạy:

```bash
cd 03-cloud-deployment/railway
railway login
railway link
railway service
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
railway variables set ENVIRONMENT=production
railway up
railway domain
```

Kết quả test public endpoint:

```powershell
curl.exe "https://agent-ai-production-98ad.up.railway.app/health"
```

Response:
- `{"status":"ok","uptime_seconds":...,"platform":"Railway","timestamp":"..."}`

```powershell
curl.exe --% -X POST https://agent-ai-production-98ad.up.railway.app/ask -H "Content-Type: application/json" -d "{\"question\":\"Hello from cloud\"}"
```

Response:
- `{"question":"Hello from cloud","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","platform":"Railway"}`

Kết luận:
- Agent đã chạy public trên cloud thành công.
- Endpoint `/health` và `/ask` hoạt động đúng qua domain Railway.

---

### Exercise 3.2: Render deployment (phần đọc cấu hình + so sánh)

Đã đọc `03-cloud-deployment/render/render.yaml`, các điểm chính:
- Deploy theo mô hình **Blueprint (Infrastructure as Code)**.
- `buildCommand`: `pip install -r requirements.txt`
- `startCommand`: `uvicorn app:app --host 0.0.0.0 --port $PORT`
- `healthCheckPath`: `/health`
- Secrets (`OPENAI_API_KEY`, `AGENT_API_KEY`) quản lý qua Render Dashboard.

---

### So sánh Railway và Render

| Tiêu chí | Railway | Render |
|---------|---------|--------|
| Cách deploy chính | CLI-first (`railway up`) | GitHub + Blueprint (`render.yaml`) |
| Quản lý cấu hình | `railway.toml` + variables CLI/dashboard | `render.yaml` + env vars dashboard |
| Tốc độ làm lab | Rất nhanh, thao tác terminal trực tiếp | Nhanh nhưng phụ thuộc flow kết nối GitHub |
| Health check | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| Phù hợp | MVP/demo/lab nhanh | Workflow IaC rõ ràng, dễ mở rộng GitOps |

---

### Checkpoint 3

- [x] Deploy thành công lên ít nhất 1 platform (Railway).
- [x] Có public URL hoạt động.
- [x] Hiểu cách set environment variables trên cloud.
- [x] Biết cách xem logs (`railway logs`).

## Part 4: API Security

### Exercise 4.1: API Key Authentication (develop)

Thư mục thực hiện: `04-api-gateway/develop`

Kết quả test:

1. Không gửi API key:
```powershell
curl.exe -X POST "http://localhost:8000/ask?question=hello"
```
Response:
```json
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

2. Gửi API key sai:
```powershell
curl.exe -X POST "http://localhost:8000/ask?question=hello" -H "X-API-Key: wrong-key"
```
Response:
```json
{"detail":"Invalid API key."}
```

3. Gửi API key đúng:
```powershell
curl.exe -X POST "http://localhost:8000/ask?question=hello" -H "X-API-Key: demo-key-change-in-production"
```
Response:
```json
{"question":"hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Kết luận:
- Thiếu key -> `401`
- Key sai -> `403`
- Key đúng -> `200`
- API key được check trong dependency `verify_api_key()` của `app.py`.

---

### Exercise 4.2: JWT Authentication (production)

Thư mục thực hiện: `04-api-gateway/production`

Lấy token:
```powershell
curl.exe --% -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" -d "{\"username\":\"student\",\"password\":\"demo123\"}"
```

Response mẫu nhận được:
```json
{"access_token":"<JWT>","token_type":"bearer","expires_in_minutes":60,...}
```

Dùng token gọi API:
```powershell
curl.exe --% -X POST http://localhost:8000/ask -H "Authorization: Bearer <JWT>" -H "Content-Type: application/json" -d "{\"question\":\"Explain JWT\"}"
```

Response nhận được:
```json
{"question":"Explain JWT","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
```

Kết luận:
- JWT flow hoạt động đúng: login lấy token -> dùng Bearer token để gọi endpoint protected.

---

### Exercise 4.3: Rate Limiting

Đã gửi liên tiếp 20 requests với user `student` và nhận kết quả:
- Request 1 -> 10: `OK`
- Request 11 -> 20: `ERROR - 429`

Kết luận:
- Rate limiter hoạt động đúng.
- Thuật toán theo code: **Sliding Window Counter**.
- Mức giới hạn cho user thường: **10 requests / 60 giây**.

---

### Exercise 4.4: Cost Guard

Đã kiểm tra usage sau khi gọi API:

```powershell
curl.exe -H "Authorization: Bearer <JWT>" "http://localhost:8000/me/usage"
```

Response thực tế:
```json
{"user_id":"student","date":"2026-04-17","requests":11,"input_tokens":24,"output_tokens":330,"cost_usd":0.000202,"budget_usd":1.0,"budget_remaining_usd":0.999798,"budget_used_pct":0.0}
```

Nhận xét:
- Hệ thống ghi nhận đúng số request, token, chi phí và ngân sách còn lại theo user.
- `check_budget()` và `record_usage()` hoạt động đúng vai trò cost guard.
- Theo thiết kế: vượt budget user trả `402`, vượt global budget trả `503`.

---

### Checkpoint 4

- [x] Implement API key authentication.
- [x] Hiểu và test JWT flow.
- [x] Test rate limiting (429 sau khi vượt ngưỡng).
- [x] Xác minh cost guard qua usage/cost tracking.
