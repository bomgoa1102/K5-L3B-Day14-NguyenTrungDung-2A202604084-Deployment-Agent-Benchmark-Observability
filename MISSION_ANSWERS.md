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
