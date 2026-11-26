# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Nguyễn Trung Dũng
**Ngày chạy:** 26/11/2026
**Môi trường:** Linux, 8 CPU logic, 15 GiB RAM; Python 3.12, đo trên máy cục bộ.

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | ≤ 500 ms | Giữ thời gian chờ phần lớn request dưới nửa giây. |
| Error rate `/ask` | ≤ 2% | Giới hạn tỷ lệ request lỗi. |
| Accuracy trên golden dataset | ≥ 100% | Mục tiêu bài tập sau khi sửa knowledge base. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.28 | 191.6 | 220.6 | 1700.3 | 0.67% | PASS |
| 10 | 35.8 | 190.7 | 222.4 | 1700.1 | 0.00% | PASS |
| 50 | 109.8 | 190.3 | 227.8 | 276.0 | 0.33% | PASS |
| 100 | 73.72 | 274.8 | 2203.8 | 2408.7 | 1.33% | FAIL |

**Nhận xét:** RPS tăng từ 4,28 lên 109,8 khi concurrency tăng từ 1 lên 50, rồi giảm còn 73,72 ở mức 100. Tải cao có thể gây xếp hàng và tăng overhead; kết quả còn phụ thuộc số request chậm ngẫu nhiên và mẫu 300 request.

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 191.1 | 217.3 | 227.1 |
| 0.03 (mặc định) | 236.0 | 224.6 | 1696.9 |
| 0.1 | 304.7 | 1676.0 | 1715.9 |

**Nhận xét:** p99 tăng từ 227,1 ms lên 1696,9 ms khi tỷ lệ chậm tăng từ 0 lên 3%. Khi tỷ lệ chậm lên 10%, p95 cũng tăng lên 1676 ms. Trung bình che khuất trải nghiệm của nhóm request chậm nên không đủ để đặt SLO.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4,3% (1/23) | 91,3% (21/23) | 100% (23/23) |
| Avg tokens / câu | 57,0 | 109,0 | 112,0 |
| Tổng cost (USD) | 0,000402 | 0,000760 | 0,000777 |
| Cost / câu đúng (USD) | 0,00040215 | 0,00003619 | 0,00003377 |
| Quality gate | FAIL | PASS | PASS |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| security-02 | token, hết hạn, chữ ký | Thêm tài liệu về JWT và API key |
| scaling-03 | CPU, memory, custom metrics | Thêm tài liệu về HPA |

Các lượt eval chạy với `MOCK_ERROR_RATE=0` để so sánh chất lượng, giữ các tham số latency mặc định.

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:** v2 sau sửa dùng nhiều token hơn (112 so với 57 token/câu) và tổng chi phí cao hơn, nhưng cost trên câu đúng thấp hơn vì trả lời đúng nhiều hơn. Latency eval có thể dao động do mẫu nhỏ và request chậm ngẫu nhiên; cần dùng load test để đánh giá latency.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [x] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [x] Một trace lỗi trong Jaeger (có event `exception`)
- [x] Trang Alerts của Prometheus với ít nhất 1 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | AgentHighErrorRate | Khoảng 90 giây | Error rate |
| B | `MOCK_SLOW_RATE=0.3` | AgentHighLatencyP95 | Khoảng 150 giây | Latency p95 |
| C | `LLM_MODEL=mock-llm-large` | LLMCostBurnRateHigh | Khoảng 155 giây | Cost burn rate |
| D | `docker pause` 1 instance | AgentInstanceDown | Khoảng 40 giây | Instances up |
| E | `AGENT_VERSION=v1` | Không có alert riêng cho chất lượng; alert cost cũ còn firing | Không áp dụng | Version, retrieval hit ratio; cần eval |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Chạy eval định kỳ trên golden dataset và so sánh với baseline; đặt quality gate để chặn release nếu accuracy giảm hoặc có regression. Bổ sung theo dõi phản hồi người dùng và kiểm tra các trường hợp thực tế.

A và B được bật đồng thời từ 16:25:21. Thời gian là lần đầu series `ALERTS{alertstate="firing"}` xuất hiện trong Prometheus, truy vấn theo bước 5 giây tính từ lúc khởi động cấu hình; có thể lệch vài giây do scrape và bước truy vấn. C và D được đo riêng theo cùng cách; dữ liệu series được lưu trong `reports/alert_firing_history.json`. Với E, eval v1 qua stack đạt 4,3%, có 22 câu regression so với baseline v2 và exit code 1 (`reports/eval_v1_regression.json`). Alert cost của C còn firing do cửa sổ đo lịch sử, nên không quy kết nó là alert do chất lượng v1.

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: `5ff718883eece6e43c79bcadc561d2a6`
- Tổng thời gian: 1672,757 ms
- Span con chiếm nhiều thời gian nhất: `chat mock-llm-small` (1654,740 ms, khoảng 98,9%).
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts":"2026-11-26T09:33:58.288981+00:00","level":"INFO","service":"observable-agent","instance":"05550c12995c","event":"llm_call","trace_id":"5ff718883eece6e43c79bcadc561d2a6","span_id":"56c07c04791c6287","request_id":"f70da291beb1","model":"mock-llm-small","agent_version":"v2","input_tokens":37,"output_tokens":17,"cost_usd":0.000016,"llm_ms":1654.6}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 59,81 | 190 | 240 | 1700 | 66/7125 (0,93%) |
| 3 | 60,34 | 190 | 220 | 1700 | 79/7186 (1,10%) |

**Giải thích kết quả:** Bảng dùng riêng dòng POST /ask trong CSV. p95 giảm từ 240 xuống 220 ms và throughput gần như không đổi. Với tải này, latency giả lập của LLM có thể là yếu tố chi phối; một lần đo chưa đủ để khẳng định nút thắt. Dùng `docker compose run --no-deps` để Locust không tự thay đổi số instance.

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. p95/p99 cho thấy các request chậm mà trung bình có thể che mất.
  2. Eval giúp phát hiện suy giảm chất lượng mà metric hạ tầng không phản ánh.
  3. Metrics giúp phát hiện vấn đề; trace và log cùng trace ID giúp tìm nguyên nhân.
- Nếu đưa agent này lên production thật, tôi sẽ theo dõi lỗi và latency ở load balancer, chất lượng câu trả lời, Redis, cùng cảnh báo dựa trên error budget và chi phí theo từng khoảng thời gian.
