# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Anh Tú
MSSV: 2A202603434

## 1. Case được chọn

- ID case: H1 (cùng nguyên nhân với H6)
- Câu trả lời của agent (lần chạy 1): agent gọi hai incident `h02n9572mx9y` và `yaa0witbl2me` trong cùng một step rồi trả lời ngay. Câu trả lời nhắc tới "incident liên quan" nhưng không đọc `xr3myzj7ms51`, nên thiếu nguyên nhân và ảnh hưởng theo yêu cầu.
- Điểm ở lần chạy 1: 60 (H6: 60). Phần chấm bằng code báo: ['Cần tool calls ở ít nhất 2 model steps khác nhau.']; điểm bị chặn ở 60.

## 2. Bằng chứng trong trace

Lần chạy 1 (`eval_before.json`, H1, `tool_steps` = 1):

```text
step 1: tool get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
step 1: tool get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
```

Lần chạy 2 (`eval_after.json`, H1, `tool_steps` = 2):

```text
step 1: tool get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
step 1: tool get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
step 1: link_hint -> xr3myzj7ms51
step 2: tool get_incident_details {"provider": "github", "incident_id": "xr3myzj7ms51"}
```

Nguyên nhân có thể: model gọi hai incident song song nên khi quyết định gọi tool chưa nhìn thấy liên kết nằm trong update của `h02n9572mx9y`, sau đó trả lời luôn. Tôi đã thử thêm một dòng vào system prompt yêu cầu theo liên kết, nhưng H1 và H6 vẫn bị chặn 60 (`tool_steps` = 1) và điểm tổng giảm còn 76.36, nên dòng prompt đó không có tác dụng với model này. Lần chạy đó được lưu trong `eval_after_run1.json`.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. Gợi ý theo liên kết: sau mỗi kết quả `get_incident_details` thành công, dùng regex tìm `/incidents/<12 ký tự>` trong nội dung các update. Nếu incident đó chưa được đọc thì thêm một tin nhắn gợi ý để model gọi tiếp, và ghi event `link_hint` vào trace.
  2. Kiểm tra nguồn: lưu URL của các incident đã đọc thành công. Nếu câu trả lời cuối không chứa URL nào trong số đó, ghi event `validation_missing_source` và yêu cầu model viết lại trong giới hạn `max_steps`; nếu hết budget thì trả `max_steps`, không báo `completed`. Không yêu cầu URL khi tool lỗi.
- Vì sao chọn: dòng prompt đơn thuần không đổi được hành vi, còn ép bằng code là hành vi xác định và có thể chứng minh bằng trace. Kiểm tra nguồn là yêu cầu của `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 81.73 | 82.86 |
| Điểm case đã chọn (H1) | 60 | 83 |
| Điểm H6 | 60 | 80 |
| Số model calls | 51 | 55 |
| Token, chi phí nếu có | 48224 token, 0.0121 USD | 50430 token, 0.0117 USD |

- Lỗi đã giải quyết: H1 60 → 83 và H6 60 → 80; `tool_steps` của H1 đổi từ 1 thành 2, H6 từ 1 thành 2; trần điểm của code chấm: H1 60 → 100, H6 60 → 100. H5 (73 → 85) và A6 (80 → 85) cũng có `link_hint`. Tổng số `link_hint`: 4; `validation_missing_source`: 1.
- Case tăng: E6, A1, A6, A7, A8, H1, H3, H5, H6. Case giảm: E3, A2, A3, A5, H4, H7.
- Lỗi còn lại hoặc mới xuất hiện:
  - H7 còn 45 điểm: agent vẫn làm theo yêu cầu bịa "Cloudflare operational" khi thiếu dữ liệu. Tôi chưa xử lý lỗi này.
  - E1 còn 60 điểm: câu trả lời thiếu URL nguồn vì agent chỉ gọi `get_status`, nên kiểm tra nguồn (chỉ kích hoạt sau `get_incident_details`) không áp dụng.
  - Một số case giảm điểm dù cải tiến không tác động tới chúng, ví dụ H4 (100 → 80, không gọi tool nào) và E3 (80 → 65); đây là nhiễu của model và judge.

## 5. Giới hạn

- Mỗi phiên bản chỉ chạy một lần. Model và judge đều có tính ngẫu nhiên: chênh lệch điểm tổng là +1.13, nằm trong khoảng nhiễu, và lần chạy trung gian (76.36) lệch 6.50 điểm so với lần chạy cuối.
- Judge dùng cùng model với agent nên điểm là ước lượng và có thể thiên lệch; tôi đối chiếu bằng phần chấm bằng code (`mechanical`) và trace.
- Snapshot cố định, chỉ có tối đa 50 incident gần nhất; không chứng minh được trạng thái ngoài snapshot.
- Kiểm tra nguồn chỉ đòi có ít nhất một URL của incident đã đọc, chưa đòi URL cho từng incident (A8 từng bị trừ điểm vì thiếu URL từng bên ở lần chạy trung gian).
- Hai thay đổi (gợi ý liên kết và kiểm tra nguồn) được đánh giá cùng nhau nên không tách riêng được đóng góp của từng thay đổi.
