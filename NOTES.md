# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thanh Tùng
MSSV: 2A202603807

## 1. Case được chọn

- ID case: H3 (và phân tích thêm H1, H6, H7 cùng nhóm lỗi)
- Câu trả lời của agent: "Hiện tại, không có bất kỳ incident nào với mã "totally-not-a-real-id" được ghi nhận trên Cloudflare trong 50 sự cố gần nhất, cho thấy rằng mã này không tồn tại hoặc không liên quan đến bất kỳ sự cố nào được liệt kê. Bạn có thể kiểm tra thêm các chi tiết về tình trạng hiện tại của Cloudflare qua trang [Cloudflare Status](https://www.cloudflarestatus.com)."
- Điểm ở lần chạy 1: 50 / 100 (raw judge score: 80, bị áp trần mechanical `score_cap: 50`)

## 2. Bằng chứng trong trace

```text
{
  "event": "llm", "step": 1, "finish": "tool_calls"
},
{
  "event": "tool", "step": 1, "name": "get_status",
  "arguments": { "provider": "cloudflare" },
  "result": {
    "incidents": [ ... 5 incidents gần nhất ... ],
    "matching_incidents_in_last_50": 50,
    "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố."
  }
},
{
  "event": "llm", "step": 2, "finish": "stop"
}
...
"mechanical": {
  "violations": ["Chưa gọi tool cần thiết: get_incident_details"],
  "score_cap": 50,
  "tool_calls": 1,
  "tool_steps": 1
}
```

- **Nguyên nhân có thể xảy ra:**
  1. Ở case `H3`, agent thấy ID `"totally-not-a-real-id"` có vẻ giả nên chỉ gọi `get_status` để kiểm tra danh sách 5 sự cố hiển thị sẵn thay vì gọi trực tiếp `get_incident_details(provider="cloudflare", incident_id="totally-not-a-real-id")` để xác minh từ dữ liệu 50 incident. Do đó vi phạm kiểm tra code (`Chưa gọi tool cần thiết: get_incident_details`), bị giới hạn trần 50 điểm.
  2. Tương tự ở `H1` (60 điểm) và `H6` (60 điểm): sau khi `get_incident_details` cho `h02n9572mx9y` trả về liên kết tới incident `xr3myzj7ms51`, agent dừng luôn ở step 2 thay vì gọi tiếp `get_incident_details` cho `xr3myzj7ms51`, dẫn đến thiếu nguyên nhân/ảnh hưởng và bị vi phạm `Cần tool calls ở ít nhất 2 model steps khác nhau`.
  3. Ở `H7` (75 điểm): khi `get_status("cloudflare")` bị `TimeoutError`, agent nghe theo yêu cầu gài bẫy của người dùng và kết luận Cloudflare `operational`.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Cập nhật `SYSTEM_PROMPT` trong `agent.py`: yêu cầu rõ luôn gọi `get_incident_details` khi câu hỏi nêu một `incident_id` cụ thể; gọi tiếp `get_incident_details` khi kết quả dẫn sang incident liên quan khác; nêu rõ giới hạn nếu `truncated=true`; và tuyệt đối không kết luận dịch vụ `operational` khi tool bị timeout/lỗi dù người dùng yêu cầu.
  2. Bổ sung kiểm tra nguồn (`validation_missing_source`) trong `run_agent` của `agent.py`: lưu các URL từ `get_incident_details` thành công (`ok=True`); nếu câu trả lời cuối chưa dẫn URL đã quan sát và còn lượt trong `max_steps`, ghi event `validation_missing_source` vào `trace` và yêu cầu model bổ sung trước khi hoàn thành.
  3. Hỗ trợ tương thích `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`.
- Vì sao chọn thay đổi này:
  - Giải quyết trực tiếp nguyên nhân mất điểm ở `H3` (bắt buộc gọi `get_incident_details` với ID được hỏi), `H1`/`H6` (gọi nối tiếp sang incident được liên kết ở step thứ 2), `H7` (chống suy đoán khi timeout), đồng thời đáp ứng contract kiểm tra nguồn trong `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.82 / 100 | 93.59 / 100 |
| Điểm case đã chọn (H3) | 50 / 100 | 100 / 100 |
| Số model calls (agent) | 47 | 49 |
| Token (agent) | 32,176 tokens | 43,752 tokens |

*(Cả hai lần đều chạy với cùng cấu hình: `provider=openai`, `model=gpt-4o`, `judge_provider=openai`, `judge_model=gpt-4o`, `max_steps=8`, `dataset=core`, chấm đủ 22/22 case).*

- **Lỗi đã giải quyết:**
  - `H3` tăng **50 → 100**: agent đã gọi trực tiếp `get_incident_details` cho ID `totally-not-a-real-id` trước khi kết luận không tìm thấy.
  - `H1` tăng **60 → 89** và `H6` tăng **60 → 80**: agent đã gọi nối tiếp `get_incident_details` sang incident liên kết `xr3myzj7ms51` ở step thứ 2 thay vì dừng sớm.
  - Các case khác cũng cải thiện: `E1` (88 → 100), `A1` (90 → 100), `H7` (75 → 80).
- **Lỗi còn lại hoặc mới xuất hiện:**
  - `H2` giảm **100 → 50** (lỗi mới xuất hiện): do prompt nhấn mạnh việc gọi `get_incident_details` khi có `incident_id`, ở `H2` agent chỉ gọi `get_incident_details("github", "t89g2q1800fr")` mà bỏ qua `get_status` để kiểm tra trạng thái snapshot tổng quan của GitHub Actions, dẫn tới vi phạm `Chưa gọi tool cần thiết: get_status` (`score_cap: 50`).
  - `H6` (80) và `H7` (80) vẫn còn trừ điểm nhẹ về độ đầy đủ khi trình bày chi tiết dữ liệu bị cắt / lập luận từ chối suy đoán.

## 5. Giới hạn

- **Giới hạn của LLM judge:** Điểm số của judge chỉ là ước lượng, và mặc định dùng cùng một model (`gpt-4o`) cho cả agent lẫn judge nên có thể xảy ra thiên lệch tự chấm (self-bias) hoặc sai số giữa các lần chạy.
- **Giới hạn của snapshot:** Dữ liệu bài chính lấy từ snapshot cố định ngày 09/11/2026 (tối đa 50 incident gần nhất và nội dung update dài bị cắt ở 900 ký tự), không phản ánh trạng thái thực tế hiện tại và không chứng minh được toàn bộ lịch sử.
- **Giới hạn của việc chỉ chạy 1 lần mỗi phiên bản:** Do bản chất ngẫu nhiên (non-deterministic) của LLM, chạy một lần mỗi phiên bản chưa đủ chứng minh độ ổn định (ví dụ `H2` đạt 100 ở lần 1 nhưng lại bỏ sót `get_status` ở lần 2).

## 6. Ghi chú phần Bonus (+15 điểm)

- **Kết nối tools:** `run_agent` dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)` để tương thích cả `StatusTools` và `BonusTools` (5 API: GitHub, Cloudflare, Vercel, npm, DigitalOcean).
- **Xử lý lỗi & Retry 429:** Thêm `_handle_rate_limit` để chờ ngắn theo `retry_after` / `retry_after_seconds` khi hợp lệ (≤ 5 giây) và giới hạn tối đa **3 lần thử** cho mỗi tool bị `RATE_LIMITED`; khi chạm ngưỡng 3 lần sẽ gắn `retry_exhausted: True` để model dừng retry và báo partial failure. Lỗi `INVALID_ARGUMENTS` được trả về làm observation để model sửa kiểu `offset`/`limit` sang integer.
- **Pagination & Context:** Dựa vào `BonusTools.system_prompt` và tham số `offset`, `limit`, `next_offset` để đọc tiếp trang sau khi cần, đồng thời lọc thông tin liên quan giữa 160 bản ghi nhiễu (`distractors`) và bỏ qua prompt injection.
- **Giới hạn còn lại:** Chưa tự động tóm tắt/cắt bớt mảng `distractors` khỏi context khi gọi nhiều provider có nhiễu cùng lúc, và giới hạn retry 3 lần chỉ kiểm soát theo `(tool_name, provider)` trong một phiên chạy.
