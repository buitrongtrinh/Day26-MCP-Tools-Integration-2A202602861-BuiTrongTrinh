# Báo cáo Lab Day 26 — MCP & Tools Integration

- **Sinh viên:** Bùi Trọng Trình — 2A202602861
- **Ngày chạy:** 2026-10-17
- **Môi trường:** Fedora Linux, Python 3.13, `uv`, `mcp` 2.x (bước 01–03), `google-adk` 2.3.0 + `mcp` 1.28.1 (lab 04), model `gemini-2.5-flash`

> API key (`GOOGLE_API_KEY`, `GEMINI_API_KEY`, `WEATHERAPI_KEY`) chỉ được đặt qua biến môi trường /
> file `04-lab/mcp-client/.env` (đã nằm trong `.gitignore`) — **không có key nào trong repo**.

## Tổng kết

| Bước | Lệnh | Kết quả |
|---|---|---|
| 01 — Function Calling | `python weather_function_calling.py` | ✅ Gemini tự quyết định gọi `get_weather` 2 lần, app thực thi, model tổng hợp câu trả lời |
| 02 — MCP basics | `python weather_client.py` | ✅ `list_tools` + `call_tool` qua stdio |
| 03a — Auth | `auth_server.py` + `auth_client.py` | ✅ Có token → gọi được tool; không token / token sai → HTTP 401 |
| 03b — Tool Registry | `python registry_client.py` | ✅ Tìm tool theo tag `weather`, chọn best match `get_weather_v2` và gọi |
| 03c — Versioning | `python versioned_client.py` | ✅ v1 và v2 chạy song song, `server://info` công bố deprecation |
| 04 — MCP server | `uv run python weather.py` | ✅ Streamable HTTP tại `localhost:8085/mcp`, 3 tool |
| 04 — `verify_setup.py` | `uv run python verify_setup.py` | ✅ All checks passed |
| 04 — ADK agent end-to-end | ADK runner / `adk web` | ✅ Gemini → ADK `McpToolset` → MCP server → WeatherAPI.com, dữ liệu thời tiết thật |

## Ghi chú khi chạy

- Trên máy chạy lab, port `8000` đã bị một tiến trình khác chiếm, nên:
  - Demo auth (03a) được chạy trên port `8001` (import `auth_server.mcp` rồi `run(port=8001)`, client đổi `SERVER_URL` tương ứng) — không sửa code.
  - `adk web` chạy với `--port 8001`; endpoint `/list-apps` trả `["weather_agent"]`, `/dev-ui/` trả HTTP 200.
- README ghi token sai → 403, nhưng MCP SDK thực tế trả **401** (`invalid_token`) cho cả trường hợp thiếu và sai token — đúng theo chuẩn OAuth 2.0 Bearer (RFC 6750).
- Gemini free tier giới hạn số request/phút; khi hỏi liên tục có thể gặp `429 RESOURCE_EXHAUSTED` — đợi ~1 phút rồi hỏi lại.

---

## Output chi tiết

### 01 — Function Calling thuần

```
User: Thời tiết Hà Nội và Đà Nẵng hôm nay thế nào?

  [model yêu cầu] get_weather({'city': 'Hà Nội'})
  [app thực thi]  -> {"city": "Hà Nội", "nhiệt_độ": "29°C", "thời_tiết": "trời mưa nhẹ", "độ_ẩm": "82%", "gió": {"hướng": "Đông Nam", "tốc_độ": "12 km/h"}}
  [model yêu cầu] get_weather({'city': 'Đà Nẵng'})
  [app thực thi]  -> {"city": "Đà Nẵng", "nhiệt_độ": "30°C", "thời_tiết": "nhiều mây", "độ_ẩm": "78%", "gió": {"hướng": "Đông", "tốc_độ": "10 km/h"}}
Trả lời: Chào bạn! Dưới đây là thông tin thời tiết hôm nay cho Hà Nội và Đà Nẵng:

**Hà Nội** 🌧️:
*   **Nhiệt độ:** 29°C
*   **Thời tiết:** Trời mưa nhẹ
*   **Độ ẩm:** 82%
*   **Gió:** Đông Nam, 12 km/h
    👉 Nên mang theo ô hoặc áo mưa khi ra ngoài nhé!

**Đà Nẵng** ☁️:
*   **Nhiệt độ:** 30°C
*   **Thời tiết:** Nhiều mây
*   **Độ ẩm:** 78%
*   **Gió:** Đông, 10 km/h
    👉 Thời tiết khá dễ chịu, nhưng có thể hơi oi bức một chút.
Chúc bạn một ngày tốt lành!
```

### 02 — MCP basics (stdio)

```
Tools server cung cấp:
  - get_weather: Lấy thời tiết hiện tại của một thành phố.

call_tool get_weather(city='Hanoi'):
  -> Hanoi: 29°C, trời mưa

call_tool get_weather(city='Danang'):
  -> Danang: 30°C, nhiều mây

call_tool get_weather(city='Haiphong'):
  -> Haiphong: 33°C, mưa rào
```

### 03a — Authentication (Bearer token)

```
Tools (có auth):
  - get_weather: Lấy thời tiết hiện tại của một thành phố.

Kết quả: Hanoi: 29°C, trời mưa

POST không token  -> HTTP 401
POST token sai    -> HTTP 401
```

### 03b — Tool Registry & Discovery

```
=== Tool Registry ===

  get_weather v1.0.0
    Lấy thời tiết hiện tại của một thành phố
    tags: ['weather', 'current', 'vietnam']  →  server: weather
  get_weather_v2 v2.0.0
    Lấy thời tiết chi tiết — JSON, hỗ trợ forecast và đơn vị đo
    tags: ['weather', 'forecast', 'detailed', 'vietnam']  →  server: weather-v2
  send_email v1.0.0
    Gửi email tới một hoặc nhiều người nhận
    tags: ['email', 'notification', 'communication']  →  server: email-service
  query_db v1.2.0
    Chạy truy vấn SQL trên database nội bộ
    tags: ['database', 'sql', 'query', 'analytics']  →  server: db-gateway

--- Agent cần tool có tag "weather" ---
Tìm thấy 2 tool(s):
  • get_weather v1.0.0 (server: weather)
  • get_weather_v2 v2.0.0 (server: weather-v2)

Best match: get_weather_v2 v2.0.0
Kết nối tới server [weather-v2]...
Kết quả: {"api_version": "2.0", "city": "Hanoi", "temp": 29, "units": "celsius", "condition": "trời mưa", "humidity": 82, "wind_speed_kmh": 12, "timestamp": "2026-10-17T11:31:25.933664+00:00"}

--- Agent tìm tool có keyword "forecast" ---
  • get_weather_v2: Lấy thời tiết chi tiết — JSON, hỗ trợ forecast và đơn vị đo
```

### 03c — Versioning & Backward Compatibility

```
Server: weather-v2 v2.0.0
Deprecated tools: ['get_weather']
Migration: Chuyển từ get_weather sang get_weather_v2. Tham số 'city' giữ nguyên, thêm include_forecast và units.

Tools:
  - get_weather: [v1] Lấy thời tiết hiện tại — trả chuỗi đơn giản. Deprecated, dùng get_weather_v2.
  - get_weather_v2: [v2] Lấy thời tiết chi tiết — JSON, hỗ trợ forecast và đơn vị đo.

[v1] get_weather('Hanoi'):
  Hanoi: 29°C, trời mưa

[v2] get_weather_v2('Hanoi', forecast=True):
  {
  "api_version": "2.0",
  "city": "Hanoi",
  "temp": 29,
  "units": "celsius",
  "condition": "trời mưa",
  "humidity": 82,
  "wind_speed_kmh": 12,
  "timestamp": "2026-10-17T11:31:26.801865+00:00",
  "forecast": [
    {"day": "tomorrow", "temp": 27, "condition": "mưa nhẹ"},
    {"day": "day_after", "temp": 31, "condition": "nắng"}
  ]
}
```

### 04 — Weather Agent (Google ADK + remote MCP server)

**Tool mà ADK `McpToolset` tự khám phá từ server** (schema sinh từ type hints, không viết tay):

```
get_current_weather {'properties': {'city': {'type': 'string'}}, 'required': ['city']}
get_forecast        {'properties': {'city': {'type': 'string'}, 'days': {'default': 3, 'type': 'integer'}}, 'required': ['city']}
health_check        {'properties': {}}
```

**`verify_setup.py`:**

```
🔍 Checking environment configuration...
✅ GOOGLE_API_KEY configured (<redacted>)

🔍 Checking dependencies...
✅ Google ADK
✅ Google Generative AI
✅ MCP
✅ FastMCP
✅ python-dotenv
✅ httpx

🔍 Checking agent structure...
✅ weather_agent/agent.py
✅ weather_agent/__init__.py

🔍 Checking MCP server connectivity...
✅ MCP server reachable at http://localhost:8085/mcp (HTTP 406)

🔍 Checking agent import...
✅ Agent imported successfully: weather_agent
   Model: gemini-2.5-flash

============================================================
✅ All checks passed!
```

**Hội thoại với agent** (`[tool call]` là lời gọi Gemini quyết định, ADK chuyển tới MCP server; tool result được cắt ngắn):

```
### USER: Is the MCP server healthy?
  [tool call] health_check({})
  [tool result] ✅ Weather MCP Server is running! Ready to provide weather data for Australian cities and worldwide.
AGENT: The Weather MCP Server is running and ready to provide weather data.

### USER: Thời tiết Hà Nội hôm nay thế nào?
  [tool call] get_current_weather({'city': 'Hanoi'})
  [tool result] Current Weather for Hanoi, , Vietnam: Temperature: 29.3°C (84.8°F) Feels like: 31.4°C (88.5°F) ...
AGENT: Thời tiết hiện tại ở Hà Nội, Việt Nam:

Nhiệt độ: 29.3°C (84.8°F)
Cảm giác như: 31.4°C (88.5°F)
Tình trạng: Trời trong
Độ ẩm: 59%
Gió: 21.6 km/h (13.4 mph) hướng Nam Đông Nam
Áp suất: 1011.0 mb
Chỉ số UV: 0.0
Tầm nhìn: 10.0 km

Cập nhật lần cuối: 17:15 ngày 17 tháng 10 năm 2026

### USER: Dự báo 3 ngày tới ở Đà Nẵng?
  [tool call] get_forecast({'city': 'Danang', 'days': 3})
  [tool result] Weather Forecast for Danang, , Vietnam: 2026-10-17: High: 28.7°C (83.7°F) Low: 21.4°C (70.6°F) ...
AGENT: Dưới đây là dự báo thời tiết 3 ngày tới tại Đà Nẵng:

**Ngày 17 tháng 10 năm 2026:**
*   Nhiệt độ cao nhất: 28.7°C (83.7°F)
*   Nhiệt độ thấp nhất: 21.4°C (70.6°F)
*   Tình trạng: Mưa rào gần đó
*   Khả năng mưa: 29%
*   Gió tối đa: 7.9 km/h
*   Chỉ số UV: 9.4

**Ngày 18 tháng 10 năm 2026:**
*   Nhiệt độ cao nhất: 28.0°C (82.3°F)
*   Nhiệt độ thấp nhất: 21.8°C (71.2°F)
*   Tình trạng: Mưa rào gần đó
*   Khả năng mưa: 84%
*   Gió tối đa: 6.5 km/h
*   Chỉ số UV: 8.2

**Ngày 19 tháng 10 năm 2026:**
*   Nhiệt độ cao nhất: 30.3°C (86.5°F)
*   Nhiệt độ thấp nhất: 22.1°C (71.7°F)
*   Tình trạng: Sương mù
*   Khả năng mưa: 31%
*   Gió tối đa: 10.1 km/h
*   Chỉ số UV: 8.2
```

**`adk web --port 8001`:** `GET /list-apps` → `["weather_agent"]`, `GET /dev-ui/` → HTTP 200.

## Kết luận

- **Function Calling** là khả năng của model: ở bước 01, schema viết tay và app tự thực thi hàm.
- **MCP** là giao thức: ở bước 02 và 04, server tự công bố tool (schema sinh từ type hints) và client khám phá tool lúc runtime (`list_tools`). Lab 04 cho thấy ADK dùng Function Calling của Gemini *bên trên* MCP — ADK lo toàn bộ vòng lặp list_tools → model chọn tool → call_tool → model tổng hợp.
- **Production** (bước 03) bổ sung Bearer-token auth, Tool Registry để agent tìm tool theo tag/keyword, và versioning song song v1/v2 kèm deprecation notice.
