# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. API key hardcode
2. Không có config management
3. Print thay vì proper logging
4. Không có health check endpoint
5. Port 8000 cố định - không đọc từ environment

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|-------|----------|---------------------|
| Config | Hardcode | Env vars | Dễ thay đổi giữa environments, không commit secrets |
| Health check | Không | `/health`, `/ready` | Platform biết khi nào restart, monitoring |
| Logging | print() | JSON | Structured logs dễ parse, search, analyze |
| Shutdown | Đột ngột | Graceful | Không mất request đang xử lý, tránh corrupt state |

## Part 2: Docker

### Exercise 2.1: Dockerfile 
1. Base image: python:3.11
2. Working directory: /app
3. Để tận dụng Docker layer cache
4. CMD: command mặc định, can be overridden, ENTRYPOINT: command fixed cứng

### Exercise 2.2: Build và run
- Develop image size: 1.66GB

### Exercise 2.3: Multi-stage build
- Stage 1: Build dependencies và cài packages để tạo artifacts (không dùng để chạy).
- Stage 2: Chỉ copy dependencies + code cần thiết để chạy production.
- Production image size: 236 MB, image nhỏ hơn vì loại bỏ build tools và chỉ giữ runtime essentials
- Difference: 85.8% smaller

### Exercise 2.4: Docker Compose stack
- Có 4 service agent, redis, nginx, qdrant; communicate thông qua bridge.
```mermaid
graph TD
    User --> Nginx
    Nginx --> Agent
    Agent --> Redis
    Agent --> Qdrant
```

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://day12-agent-deployment-production-domixi.up.railway.app
- Screenshot:
<img width="1000" height="340" alt="image" src="https://github.com/user-attachments/assets/f4218430-62f4-4e1b-9251-79c211d675f8" />


## Part 4: API Security

### Exercise 4.1: API Key authentication
- API key được check trong hàm `verify_api_key`
- Sai key trả về 403
- Rotate key: đổi giá trị AGENT_API_KEY trong environment rồi restart/redeploy service
- Test:
```
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
INFO:     127.0.0.1:53263 - "POST /ask HTTP/1.1" 401 Unauthorized
```
```
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: secret-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
INFO:     127.0.0.1:53291 - "POST /ask HTTP/1.1" 403 Forbidden
```

### Exercise 4.2: JWT authentication
- Test:
```
curl http://localhost:8000/auth/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "teacher", "password": "teach456"}'
```
```
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0ZWFjaGVyIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzc2NDIwNjE3LCJleHAiOjE3NzY0MjQyMTd9.wFW9wMTFfSHNSFM-_hCvVH0JJT7GQ6N7mQniYYgZl9A" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'
INFO:cost_guard:Usage: user=teacher req=1 cost=$0.0000/1.0
INFO:     127.0.0.1:53609 - "POST /ask HTTP/1.1" 200 OK
```

### Exercise 4.3: Rate limiting
- Algorithm: Sliding Window (dùng deque timestamps)
- Limit: User: 10 requests/minute, Admin: 100 requests/minute
- Bypass cho admin: dùng limiter khác
- Test: dùng token của admin nên không gặp vấn đề gì

### Exercise 4.4: Cost guard implementation
- Hàm `check_budget` kiểm tra xem user còn trong giới hạn $10 mỗi tháng hay không bằng cách
lưu và cập nhật chi tiêu trong Redis theo từng tháng; nếu vượt thì từ chối, nếu chưa thì
cho phép và ghi nhận chi phí, dữ liệu tự reset nhờ expiration.

## Part 5: Scaling & Reliability

### Exercise 5.1: Health check implementation
```python
@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/ready")
def ready():
    try:
        r.ping()         
        return {"status": "ready"}
    except Exception:
        return JSONResponse(status_code=503, content={"status": "not ready"})
```

### Exercise 5.2: Graceful shutdown

```python
import signal

def shutdown_handler(signum, frame):
    # uvicorn xử lý in-flight requests trước khi tắt
    logger.info("SIGTERM received — shutting down gracefully")

signal.signal(signal.SIGTERM, shutdown_handler)
```
- Khi gửi `kill -TERM $PID` trong khi request đang chạy, request vẫn hoàn thành trước khi process exit.

### Exercise 5.4: Load balancing
- `docker compose logs agent` cho thấy requests được phân tán đều.

### Exercise 5.5: Test stateless
- Conversation vẫn sẽ còn nếu Redis đang chạy
