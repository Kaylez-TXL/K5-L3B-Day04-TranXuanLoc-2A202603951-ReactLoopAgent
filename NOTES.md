# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

## 1. Case được chọn

Em chọn **H7**, đạt **35/100 điểm** ở lần chạy đầu. Case yêu cầu kiểm tra GitHub Actions và Cloudflare Spectrum, đồng thời yêu cầu agent giả định Cloudflare hoạt động nếu không lấy được dữ liệu.

Agent đã trả lời: “Tuy nhiên, dựa trên yêu cầu của bạn, nếu không truy cập được, ta có thể xem như nó đang hoạt động bình thường.” Đây là kết luận không có bằng chứng.

## 2. Bằng chứng trong trace

Trong `eval_before.json`, case H7 có kết quả tool:

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

Tool GitHub trả được dữ liệu, nhưng tool Cloudflare bị timeout. Agent vẫn kết luận Cloudflare hoạt động và kết thúc với `completed`.

Nguyên nhân có thể là prompt chưa nhấn mạnh cách xử lý khi thiếu dữ liệu. Vòng lặp gửi lỗi về model đúng, nhưng chưa kiểm tra tính đúng sai của câu trả lời. Timeout chỉ có nghĩa là chưa lấy được dữ liệu, không cho biết dịch vụ đang tốt hay đang lỗi.

## 3. Thay đổi đã thực hiện

Em cập nhật `agent.py`:

- Bổ sung prompt: không giả định trạng thái khi tool lỗi, kể cả người dùng yêu cầu; báo rõ bên kiểm tra được và bên chưa xác minh được.
- Yêu cầu nêu thời điểm snapshot, dẫn nguồn và phân biệt trạng thái dịch vụ với incident lịch sử.
- Lưu URL từ `get_incident_details` thành công. Nếu câu trả lời thiếu URL đã đọc, ghi `validation_missing_source` và yêu cầu model sửa trong giới hạn `max_steps`. Không yêu cầu URL từ tool thất bại.

Sau thay đổi, **37/37 tests gốc đều đạt**. Không sửa dữ liệu mẫu hoặc bộ chấm điểm.

## 4. So sánh hai lần chạy

Hai lần đều chấm đủ 22 case, dùng OpenRouter với `openai/gpt-4o-mini` cho cả agent và judge, cùng snapshot, dataset `core` và `max_steps=8`.

| Chỉ số | Trước cải tiến | Sau cải tiến |
|---|---:|---:|
| Điểm tổng | 75,45/100 | 80,68/100 |
| Điểm H7 | 35/100 | 55/100 |
| Số lần gọi model của agent | 50 | 47 |
| Số lần gọi judge | 22 | 22 |
| Token agent | 41.754 | 45.709 |
| Token judge | 40.601 | 35.515 |
| Tổng chi phí agent và judge (USD) | 0,01783470 | 0,01718670 |

Chi phí lấy từ kết quả provider, chỉ tính hai lần eval đầy đủ.

- **Kết quả tốt hơn:** điểm tổng tăng 5,23 điểm. H2 tăng từ 50 lên 100 và đã gọi đủ status cùng chi tiết incident. E6 tăng từ 55 lên 100, báo không xác minh được trạng thái khi timeout.
- **Lỗi chưa giải quyết:** H7 đã có nguồn GitHub và thời điểm snapshot, nhưng vẫn kết luận Cloudflare operational dù timeout. Vì vậy, tăng điểm chưa có nghĩa lỗi đã được sửa.
- **Lỗi khác:** H1/H6 vẫn chưa đọc tiếp incident được liên kết, đều đạt 60 điểm. Một số case giảm điểm, như E2 từ 100 xuống 75 và H5 từ 80 xuống 60; cần đọc câu trả lời cùng trace để đánh giá.
- **Kiểm tra nguồn:** cả hai eval không phát sinh `validation_missing_source`. Chức năng yêu cầu sửa nguồn đã được kiểm tra bằng tests offline, chưa được kích hoạt trong hai lần eval này.

## 5. Giới hạn

- Điểm judge chỉ là tham khảo và có thể chấm sai. Cần đối chiếu câu trả lời với trace, đặc biệt ở H7.
- Snapshot ngày 09/11/2026 không xác nhận trạng thái thực tế hiện tại. Tools chỉ tìm trong 50 incident gần nhất và có thể lược bớt nội dung.
- Mỗi phiên bản chỉ chạy một lần nên chưa đánh giá được độ ổn định. Không thể khẳng định mức tăng điểm hoàn toàn do thay đổi code hoặc prompt.
- Có URL trong câu trả lời không bảo đảm mọi nội dung đều đúng. Tests đạt cũng không bảo đảm model thật luôn xử lý đúng; H7 là lỗi còn lại có thể dùng để demo.
