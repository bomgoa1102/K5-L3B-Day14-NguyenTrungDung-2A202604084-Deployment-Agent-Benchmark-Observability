# Day 14 — Mission Answers

## Part 1: Localhost và production

### Exercise 1.1: Các vấn đề ở bản develop

1. API key và thông tin kết nối database được ghi cứng trong mã nguồn.
2. API key bị in ra log khi xử lý request.
3. Cấu hình debug và giới hạn token được đặt trực tiếp trong code.
4. Dùng `print` thay cho logging có cấu trúc.
5. Không có endpoint health check để kiểm tra tiến trình.
6. Server chỉ bind vào localhost và dùng cổng cố định 8000.
7. Bật reload khi chạy server, không phù hợp cho production.

### Exercise 1.2: Chạy bản develop

Chạy ứng dụng bằng Uvicorn tại cổng 18765 và gửi `POST /ask?question=Hello`. Kết quả HTTP 200:

```json
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

### Exercise 1.3: So sánh

| Nội dung | Develop | Production | Vì sao quan trọng |
|---|---|---|---|
| Cấu hình | Ghi cứng trong code | Đọc qua settings và biến môi trường | Dễ đổi cấu hình theo môi trường |
| Log | Dùng print, có in key mẫu | Dùng logging, không in key | Dễ theo dõi và giảm nguy cơ lộ thông tin |
| Health | Không có | Có `/health` và `/ready` | Giúp nền tảng biết tiến trình còn sống và sẵn sàng |
| Kết nối | Localhost, cổng 8000 | Host và cổng cấu hình được | Container và nền tảng cloud có thể truy cập |
| Shutdown | Không có xử lý riêng | Có lifespan để dọn dẹp | Giúp đóng tài nguyên khi dừng dịch vụ |

## Part 2: Docker

### Exercise 2.1–2.3: Dockerfile và kích thước

- Base image bản develop là `python:3.11`; working directory là `/app`.
- Copy `requirements.txt` trước source để Docker dùng lại cache cài dependency khi chỉ đổi code.
- `CMD` cung cấp lệnh hoặc tham số mặc định có thể thay khi chạy container; `ENTRYPOINT` xác định chương trình chính và các tham số chạy thêm thường được truyền cho chương trình đó.
- Bản multi-stage cài dependency ở builder, sau đó chỉ copy các gói cần chạy sang runtime, nên không mang theo toàn bộ công cụ build.
- Docker inspect đo bản develop 1.667.780.304 bytes (khoảng 1.667,8 MB), bản production 239.815.377 bytes (khoảng 239,8 MB): giảm khoảng 85,6% theo đơn vị MB thập phân.
- Đã chạy container develop và gọi `POST /ask?question=What%20is%20Docker`: HTTP 200. Bản production trả HTTP 200 tại `/health`. Ví dụ develop nhận question qua query, không phải JSON body.
- Docker Hub trả 429 khi tải `python:3.11`; dùng image tương ứng từ public ECR rồi gắn tag local để build.

Exercise 2.4: Chưa kiểm tra stack Compose qua Nginx.

## Part 3: Cloud deployment

Agent được deploy trên Render tại [URL công khai](https://k5-l3b-day14-nguyentrungdung-2a202604084.onrender.com). `/health` và `/ready` trả 200; `POST /ask` không có key trả 401. Chi tiết lệnh kiểm tra nằm trong `DEPLOYMENT.md`; request có key được xác nhận trả 200 trên Render.

Trong cấu hình ví dụ `03-cloud-deployment/railway/railway.toml`, Railway dùng Nixpacks và khai báo lệnh khởi động Uvicorn; cấu hình `06-lab-complete/render.yaml` dùng Docker runtime, khai báo health check và các biến môi trường. Hai file thuộc hai ứng dụng mẫu khác nhau, nên lệnh chạy không thể đem thay nguyên xi cho nhau.

Với Cloud Run, `cloudbuild.yaml` mô tả chuỗi bước chạy test, build image, push image rồi deploy image theo commit lên Cloud Run. Cấu hình deploy chỉ định tài nguyên, giới hạn instance và secret từ Secret Manager. File `service.yaml` mô tả thêm cổng, concurrency, autoscaling và probe `/health`, `/ready`. Chưa triển khai pipeline này lên GCP.

## Part 6: Kiểm tra production

```text
📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 42/42 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

## AI log

- Dùng AI để đọc yêu cầu và hỗ trợ phân tích hai phiên bản Part 1, soạn bảng so sánh và chạy thử endpoint. Kết quả gọi API được lấy từ lệnh chạy tại máy.
- Part 6: Dùng AI hỗ trợ triển khai agent, cấu hình Docker và Render, và chạy script kiểm tra. Với Redis cục bộ, chế độ `--skip-docker` đạt 10/12 tiêu chí; hai tiêu chí cần Docker chưa kiểm tra được do Docker Hub trả 429 khi tải image. Cần chạy lại đầy đủ khi tải được image.
- Part 7: Dùng AI hỗ trợ chạy eval và bổ sung kiến thức về JWT, HPA. Đã chạy lại với cùng dataset và đối chiếu baseline: 23/23, không regression. Các lượt eval đặt `MOCK_ERROR_RATE=0`.
- Part 3: Dùng AI hỗ trợ kiểm tra URL Render công khai. `/health` và `/ready` trả 200; `POST /ask` không có key trả 401. Chưa kiểm tra request có key.
- Part 6: Chạy lại script chấm đầy đủ sau khi tải được Docker image; kết quả 12/12, 42/42 checks.
- Part 7: Dùng AI hỗ trợ chạy load test với các mức concurrency và tỷ lệ request chậm, rồi đối chiếu các con số trong báo cáo với JSON.
- Part 3: Dùng AI hỗ trợ đọc và so sánh các file cấu hình cloud, kiểm tra endpoint công khai.
- Part 7: Dùng AI hỗ trợ tìm lỗi container observability không tìm thấy uvicorn; đặt HOME phù hợp với thư mục cài thư viện và kiểm tra lại request qua Nginx trả HTTP 200.
- Part 7: Dùng AI hỗ trợ chạy Locust với 1 và 3 instance, dùng `--no-deps` để giữ đúng số instance và lấy số từ dòng POST /ask của CSV. `promtool test rules` trả SUCCESS.
- Part 2: Dùng AI hỗ trợ build hai Docker image, chạy container và đọc kích thước bằng Docker inspect; dùng mirror ECR khi Docker Hub trả 429.
