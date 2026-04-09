# Security Fixes — Ghi Chú Tìm và Fix 3 Lỗ Hổng

**Người fix:** Nguyễn Tiến Đạt  
**Từ version:** v1.0 → v1.1  
**Ref:** `CHANGES.md` mục Critical Fixes

---

## Fix 1 — Rate Limit Bypass qua `X-Forwarded-For` giả

### Phát hiện như thế nào

Đang đọc `src/api/server.py` để thêm rate limiting, thấy code lấy IP client:

```python
# Code cũ (server.py v1)
client_ip = request.headers.get("X-Forwarded-For", request.client.host)
```

Nghĩ ngay: nếu client tự gửi header `X-Forwarded-For: 1.2.3.4` thì server sẽ tưởng request đến từ `1.2.3.4`, không phải IP thật. Rate limit vô nghĩa.

Test thử:
```bash
curl -X POST http://localhost:8000/chat \
  -H "X-Forwarded-For: 99.99.99.99" \
  -d '{"message": "test"}'
```
→ Server nhận IP `99.99.99.99`, bypass rate limit hoàn toàn.

### Fix

**Nginx** — không tin bất kỳ header nào từ client, tự set:
```nginx
# nginx.conf — trước fix
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # ❌ tin client

# nginx.conf — sau fix
proxy_set_header X-Real-IP      $remote_addr;                  # ✅ IP thật từ TCP
proxy_set_header X-Forwarded-For $remote_addr;                 # ghi đè header client
```

**FastAPI** — dùng `X-Real-IP` thay vì `X-Forwarded-For`:
```python
# server.py — sau fix
client_ip = request.headers.get("X-Real-IP", request.client.host)
```

### Tại sao quan trọng

Rate limit là hàng phòng thủ cuối cùng trước khi Groq API bị spam. Nếu bypass được thì attacker có thể tốn hết quota Groq trong vài phút.

---

## Fix 2 — CORS `allow_origins=["*"]`

### Phát hiện như thế nào

Review `server.py` v1, thấy:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],   # ❌
    allow_methods=["*"],
    allow_headers=["*"],
)
```

`allow_origins=["*"]` nghĩa là **bất kỳ domain nào** cũng có thể gọi API này từ browser — bao gồm trang web độc hại.

Scenario tấn công: user đang đăng nhập app Vinmec, bấm vào link lạ → trang đó silently gọi `/chat` API với cookie của user → exfiltrate data.

### Fix

Đọc từ env var, chỉ whitelist những origin cần thiết:

```python
# config.py
ALLOWED_ORIGINS = os.getenv(
    "ALLOWED_ORIGINS",
    "http://localhost:3000,http://localhost:5173"  # dev default
).split(",")

# server.py
app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,  # ✅
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Content-Type", "X-API-Key"],
)
```

Production `.env`:
```
ALLOWED_ORIGINS=https://vinmec-prep.vinmec.com
```

---

## Fix 3 — SearXNG Engine `FileNotFoundError`

### Phát hiện như thế nào

Chạy `docker compose up`, SearXNG container crash với:

```
FileNotFoundError: [Errno 2] No such file or directory: 
'.../searx/engines/google images.py'
```

Đọc `settings.yml` v1:

```yaml
engines:
  - name: google images
    disabled: true   # ❌ SearXNG không có syntax này
  - name: youtube
    disabled: true
```

SearXNG không support `disabled: true` per-engine trong custom settings — nó cố load file engine Python, không tìm thấy thì crash.

### Fix

Dùng `keep_only` — chỉ bật những engine cần:

```yaml
# settings.yml — sau fix
use_default_settings:
  engines:
    keep_only:
      - google
      - bing
      - duckduckgo
      - wikipedia
```

Đơn giản hơn, không cần liệt kê tất cả engine muốn disable.

### Bài học

Đây là loại bug **silent fail** nguy hiểm — nếu không chạy docker compose ngay từ đầu thì sẽ không biết SearXNG crash, cứ nghĩ web search đang hoạt động. Cần smoke test infrastructure ngay khi setup, không để đến lúc demo mới phát hiện.
