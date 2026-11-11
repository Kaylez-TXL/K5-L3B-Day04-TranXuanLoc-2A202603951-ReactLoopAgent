# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

## 1. Case được chọn

- ID case: H7 (scenario `cloudflare_timeout`).
- Câu trả lời của agent (trích): "Tuy nhiên, dựa trên yêu cầu của bạn, nếu không truy cập được, ta có thể xem như nó đang hoạt động bình thường."
- Điểm ở lần chạy 1: 35/100; điểm tổng 22 case: 75,45/100.

## 2. Bằng chứng trong trace

Nguồn: `eval_before.json`, case H7, `result.trace` và `result.answer`.

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {"provider": "cloudflare", "component": "spectrum"},
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```

Ở cùng step 1, tool GitHub trả Actions `operational`, `captured_at` là
`2026-11-09T03:50:47.065020+00:00` và `source` là `https://www.githubstatus.com`.
Step 2 model trả `stop`; agent kết thúc `completed` nhưng câu trả lời giả định
Cloudflare hoạt động, thiếu URL GitHub và không gắn kết luận với thời điểm snapshot.
Timeout chỉ chứng minh không lấy được dữ liệu, không chứng minh trạng thái dịch vụ.

Nguyên nhân có thể: prompt ban đầu chưa nói rõ không được làm theo yêu cầu giả định
trạng thái khi tool lỗi. Loop trả observation đúng, nhưng chấp nhận final answer
ngay khi model dừng; `completed` không bảo đảm nội dung đúng. Judge cho boundaries
0/15, phù hợp với lỗi có thể đối chiếu trực tiếp trong answer và trace.

## 3. Thay đổi đã thực hiện

- Sửa `SYSTEM_PROMPT` trong `agent.py`: cấm suy ra operational/outage khi timeout,
  kể cả người dùng yêu cầu; báo riêng provider có và thiếu dữ liệu, nêu thời điểm
  snapshot, dẫn URL tool trả về và không suy rộng ngoài các incident đã quan sát.
  Làm rõ việc kiểm tra status riêng với incident lịch sử và giới hạn tìm chuỗi.
- Thêm kiểm tra nguồn vào `run_agent`: thu thập URL từ `get_incident_details`
  thành công. Nếu final answer thiếu URL đã quan sát, ghi
  `validation_missing_source` và yêu cầu model sửa trong cùng `max_steps`.
  Tool thất bại không tạo yêu cầu dẫn URL; hết budget sửa thì trả `max_steps`.
- Chọn thay đổi vì H7 có bằng chứng suy luận sai khi thiếu dữ liệu, đồng thời
  kiểm tra nguồn là contract bắt buộc của Bài 4. Không hard-code case hay incident ID.
- Kiểm chứng offline bằng bộ tests có sẵn. Chưa thể kết luận H7 đã được sửa trong
  thực tế trước khi chạy eval lần 2 với cùng cấu hình.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
