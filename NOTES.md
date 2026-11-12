# Ghi chú phân tích · Day 04

Họ và tên: Trần Xuân Lộc
MSSV: 2A202603951

## 1. Case được chọn

Em chọn H7, đạt 35/100 điểm ở lần đầu. Tool Cloudflare bị timeout nhưng agent vẫn trả lời: “Tuy nhiên, dựa trên yêu cầu của bạn, nếu không truy cập được, ta có thể xem như nó đang hoạt động bình thường.”

## 2. Bằng chứng trong trace

Trong `eval_before.json`, H7 có đoạn trace:

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

Timeout chỉ cho biết không lấy được dữ liệu, không chứng minh dịch vụ hoạt động bình thường. Nguyên nhân có thể là prompt chưa rõ và vòng lặp chưa kiểm tra nội dung khi model trả lời xong.

## 3. Thay đổi đã thực hiện

Em sửa `agent.py` để nhắc model không đoán trạng thái khi tool lỗi, nêu thời điểm snapshot và dẫn nguồn. Em cũng thêm kiểm tra URL incident: nếu câu trả lời thiếu URL đã đọc thành công thì ghi `validation_missing_source` và yêu cầu sửa trong `max_steps`.

Kết quả kiểm tra: 37/37 tests gốc đạt. Không sửa dữ liệu mẫu hay bộ chấm.

## 4. So sánh hai lần chạy

Hai lần đều đủ 22 case, cùng snapshot và dataset `core`; agent và judge dùng OpenRouter, `openai/gpt-4o-mini`, `max_steps=8`.

| Chỉ số | Trước | Sau |
|---|---:|---:|
| Điểm tổng | 75,45 | 80,68 |
| Điểm H7 | 35 | 55 |
| Lần gọi model của agent | 50 | 47 |
| Lần gọi judge | 22 | 22 |
| Token agent | 41.754 | 45.709 |
| Token judge | 40.601 | 35.515 |
| Tổng chi phí (USD) | 0,01783470 | 0,01718670 |

Chi phí do provider báo, chỉ tính hai lần eval đầy đủ.

H2 đã gọi đủ tools, E6 đã báo không xác minh được khi timeout. Tuy nhiên, H7 vẫn đoán Cloudflare operational; H1/H6 chưa đọc tiếp incident liên quan. Một số case giảm điểm, như E2 từ 100 xuống 75. Cả hai eval không có event sửa nguồn; phần này được kiểm tra bằng tests offline.

## 5. Giới hạn

Điểm judge có thể sai nên cần đọc cả câu trả lời và trace. Snapshot không phản ánh trạng thái hiện tại, dữ liệu tìm kiếm và nội dung trả về có giới hạn. Mỗi phiên bản chỉ chạy một lần nên chưa đánh giá được độ ổn định. Có URL và tests đạt không bảo đảm mọi câu trả lời đều đúng.

## 6. Bonus

Em mở rộng agent cho 5 dịch vụ bằng `tools.schemas` và `tools.system_prompt`. Khi gặp 429, agent cho thử lại tối đa 3 lần liên tiếp cùng tool/provider, đọc `Retry-After` và giới hạn tổng thời gian chờ 30 giây. Timeline được đọc tiếp bằng `next_offset` khi cần. Dữ liệu nhiễu được bỏ khỏi context nhưng vẫn giữ trong trace.

37 tests gốc và 3 tests bonus đều đạt. Eval bonus dùng cùng model trên, `max_steps=16`, chấm đủ 10 case và đạt 78,5/100, tương đương 11,78/15 điểm tham khảo.

Trong trace, B2 đọc được 3 trang, B3/B10 thử lại sau 429 thành công và B5 phục hồi sau lỗi tham số. B6 vẫn chưa đọc chi tiết incident nên bị giới hạn 50 điểm. Câu trả lời ở một số case chưa giải thích rõ việc thử lại; đây là phần còn cần cải thiện.
