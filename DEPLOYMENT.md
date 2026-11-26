# Deploy agent

- Nền tảng: Render Web Service
- URL: https://k5-l3b-day14-nguyentrungdung-2a202604084.onrender.com
- Thư mục deploy: `06-lab-complete/`

## Kiểm tra

Ngày 26/11/2026, các lệnh sau trả về kết quả như bên dưới:

```bash
BASE=https://k5-l3b-day14-nguyentrungdung-2a202604084.onrender.com
curl -i "$BASE/health"       # HTTP 200, {"status":"ok"}
curl -i "$BASE/ready"        # HTTP 200, {"ready":true}
curl -i -X POST "$BASE/ask" -H 'Content-Type: application/json' \
  -d '{"question":"Hello"}'  # HTTP 401 khi không có key
```

Kiểm tra với API key (nhập key trong terminal, không ghi key vào file):

```bash
read -rsp 'API key: ' DAY14_API_KEY; echo
curl -i -X POST "$BASE/ask" -H 'Content-Type: application/json' \
  -H "X-API-Key: $DAY14_API_KEY" -d '{"question":"Hello"}'
unset DAY14_API_KEY
```

Người dùng xác nhận đã kiểm tra trên Render: request có key trả HTTP 200. `REDIS_URL` và `AGENT_API_KEY` được cấu hình trong Environment của web service, không đưa giá trị vào repo.
