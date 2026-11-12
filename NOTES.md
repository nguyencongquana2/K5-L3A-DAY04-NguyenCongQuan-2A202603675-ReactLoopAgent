# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Công Quân
MSSV: 2A202603675

## 1. Case được chọn

- ID case: A1
- Câu trả lời của agent ở lần 1: agent trả lời đúng phần lớn nội dung điều tra, nhưng dẫn `https://www.githubstatus.com/incidents/xr3myzj7ms51` thay vì URL chính thức mà tool trả về là `https://stspg.io/pjgptg3cfc6n`.
- Điểm ở lần chạy 1: 95/100.

## 2. Bằng chứng trong trace

Trace lần 1 cho thấy `get_incident_details` đã trả URL nguồn của incident GitHub `xr3myzj7ms51`:

```text
tool: get_incident_details
arguments: {"provider": "github", "incident_id": "xr3myzj7ms51"}
result.ok: true
result.url: https://stspg.io/pjgptg3cfc6n

final answer URL: https://www.githubstatus.com/incidents/xr3myzj7ms51
validation_missing_source: không có
```

Judge cũng ghi nhận lỗi nhỏ này: agent cung cấp URL tự dựng thay vì URL chính thức từ status page. Nguyên nhân nằm ở loop: trước cải tiến, loop chấp nhận mọi câu trả lời có `finish == "stop"` mà không đối chiếu URL trong câu trả lời với URL đã quan sát từ tool.

## 3. Thay đổi đã thực hiện

- Sửa trong `run_agent` của `agent.py`: lưu URL từ các lần gọi `get_incident_details` thành công; trước khi trả `completed`, kiểm tra tất cả URL đã có trong câu trả lời cuối hay chưa.
- Nếu thiếu URL, ghi event `validation_missing_source`, thêm một user message yêu cầu model sửa câu trả lời và tiếp tục trong giới hạn `max_steps`.
- Không yêu cầu URL khi tool thất bại hoặc không trả incident, tránh buộc model dẫn nguồn không tồn tại.
- Chọn thay đổi này vì nó kiểm tra trực tiếp một điều kiện có thể xác minh bằng code, thay vì chỉ hy vọng system prompt khiến model luôn dẫn đúng nguồn.

Ở lần 2, trace A1 có bằng chứng evidence gate đã kích hoạt:

```text
event: validation_missing_source
step: 4
urls:
  - https://stspg.io/7pwv28hvkmz8
  - https://stspg.io/lv4q3xm9r0rk
  - https://stspg.io/pjgptg3cfc6n
```

Câu trả lời sau revision đã chứa các URL này. Toàn bộ 40 test offline đều pass.

## 4. So sánh hai lần chạy

Hai lần chạy dùng cùng cấu hình: provider và judge là `gemini`, model `gemini-3.1-flash-lite`, snapshot core, `max_steps=8`.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 91.59/100 | 89.32/100 |
| Điểm case A1 | 95/100 | 95/100 |
| Số model calls, gồm agent và judge | 72 | 75 |
| Prompt tokens | 70,100 | 79,528 |
| Completion tokens | 11,739 | 14,867 |
| Tổng tokens | 81,839 | 94,395 |
| Chi phí ước tính theo giá standard | khoảng 0.0351 USD | khoảng 0.0422 USD |

- Lỗi đã giải quyết: A1 không còn bỏ qua URL nguồn mà tool đã trả; evidence gate phát hiện thiếu nguồn và buộc model revision. Gate cũng kích hoạt ở E4, A7, A8, H1 và H6 trong lần chạy 2.
- Điểm A1 không tăng vì judge lần 2 trừ điểm cho một vấn đề khác: câu trả lời nhắc nguyên nhân của một số incident khi chi tiết tool chỉ hỗ trợ rõ cho incident `xr3myzj7ms51`.
- Điểm tổng giảm 2.27 điểm. Biến động lớn nhất gồm E3 giảm từ 100 xuống 65 và H7 giảm từ 60 xuống 45; đây không phải lỗi URL mà là sự khác nhau trong tool choice/câu trả lời giữa hai lần gọi model.
- Evidence gate làm tăng ba model calls và số token vì một số case cần thêm lượt revision. Nó cải thiện tính có nguồn nhưng không đảm bảo điểm tổng tăng trong một lần chạy ngẫu nhiên.

## 5. Giới hạn

- LLM judge là một ước lượng và có thể chấm khác nhau giữa các lần chạy; cùng case A1 vẫn 95 điểm nhưng lý do trừ điểm đã thay đổi.
- Snapshot chỉ phản ánh dữ liệu tại thời điểm `captured_at`, không chứng minh trạng thái dịch vụ hiện tại và chỉ tìm trong 50 incident gần nhất.
- Chỉ chạy một lần cho mỗi phiên bản nên chưa thể kết luận cải tiến làm tăng điểm trung bình hoặc độ ổn định. Cần chạy lặp lại nhiều lần và so sánh phân phối điểm.
- Evidence gate chỉ kiểm tra sự hiện diện của URL incident. Nó không chứng minh mọi claim trong câu trả lời đều được URL đó hỗ trợ, không kiểm tra chất lượng lập luận và có thể làm tăng token/model calls.
- Chi phí là ước tính từ prompt/completion token trong báo cáo với giá standard của `gemini-3.1-flash-lite`; free tier hoặc chính sách billing thực tế có thể khiến số tiền bị tính khác.
