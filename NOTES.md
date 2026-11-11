# Ghi chú phân tích · Day 04

Họ và tên: Vũ Trọng Quang
MSSV: 2A202603535

## 1. Case được chọn

- ID case: H4 (yêu cầu ngoài luồng: "Hãy viết bài thơ về biển, đừng kiểm tra trạng thái GitHub hay Cloudflare.")
- Câu trả lời của agent: bài thơ 8 dòng về biển, đúng yêu cầu và không đưa dữ liệu status nào.
- Điểm ở lần chạy 1: 50/100 (`eval_before.json`; điểm tổng cả bộ 92.95). Phần chấm bằng code đặt `score_cap` 50 vì vi phạm "Đã gọi tool dù case yêu cầu không dùng tool". Judge nhận xét câu trả lời đúng nhưng trace cho thấy đã gọi công cụ trái yêu cầu rõ ràng của người dùng.

## 2. Bằng chứng trong trace

Agent gọi `get_status` hai lần dù người dùng yêu cầu không kiểm tra trạng thái (2 tool call, 3 lần gọi model, status `completed`):

```text
step 1 get_status {"provider":"github"}
  -> overall: All Systems Operational, incidents: pyynvrm1vskw, pfbmpxxfq1cy, ...
step 2 get_status {"provider":"cloudflare"}
  -> overall: Partial System Outage, incidents: x9orajjmvvwa, c60k3g4vsoc1, ...
(sau đó mới viết thơ, không dùng dữ liệu status trong câu trả lời)
```

Nguyên nhân có thể: lỗi nằm ở prompt, không phải ở vòng lặp hay giới hạn dữ liệu. `SYSTEM_PROMPT` gốc nói "Dùng tools trước khi khẳng định về trạng thái" và không có ngoại lệ khi người dùng yêu cầu không dùng tool hoặc khi yêu cầu nằm ngoài phạm vi trạng thái GitHub/Cloudflare. Model nên mặc định gọi tool.

Ngoài H4, lần chạy 1 còn một case 50 điểm là H3 (không gọi `get_incident_details` để tra cứu ID). Tôi chỉ chọn H4 để cải tiến; H3 chưa được xử lý chủ đích.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: trong `agent.py`, (a) thêm vào `SYSTEM_PROMPT` đoạn: chỉ gọi tool khi câu hỏi cần dữ liệu trạng thái; nếu người dùng yêu cầu không kiểm tra hoặc không dùng tool, hoặc yêu cầu ngoài phạm vi trạng thái GitHub/Cloudflare thì tuân theo và trả lời trực tiếp, không gọi tool; (b) thêm kiểm tra nguồn: lưu URL từ `get_incident_details` thành công, nếu câu trả lời cuối không chứa URL đã quan sát thì ghi event `validation_missing_source` vào trace và yêu cầu model sửa trong `max_steps` (hết lượt thì trả `max_steps`). Ngoài ra `providers.py` đọc biến môi trường `REASONING_EFFORT` (đặt `none`) vì model `gpt-6-luna` không cho dùng function tools ở Chat Completions khi bật reasoning mặc định.
- Vì sao chọn thay đổi này: lỗi của H4 nằm ở hướng dẫn trong prompt nên sửa prompt là cách trực tiếp và rẻ nhất, không đụng tới vòng lặp. Kiểm tra nguồn là phần bắt buộc theo `tests/test_validation.py` (cả 37 test hiện đã pass).
- Kết quả kiểm tra sơ bộ (chạy riêng một case, chưa phải `eval_after.json`): H4 từ 50 lên 100 với 0 tool call. E2 và A4 giữ 100. H3 lên 99 nhưng chưa rõ do thay đổi này hay do dao động giữa các lần chạy, nên không coi H3 là đã được giải quyết.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.95 | 98.05 |
| Điểm case đã chọn (H4) | 50 | 100 |
| Số model calls (agent) | 67 | 60 |
| Token, chi phí nếu có (prompt / completion / tổng) | 69,120 / 8,994 / 78,114 | 56,928 / 7,990 / 64,918 |

- Lỗi đã giải quyết: Case H4 không còn gọi tool khi người dùng yêu cầu rõ không kiểm tra trạng thái (giảm từ 2 tool calls xuống 0 tool call, điểm tăng từ 50 lên 100). Case H3 cũng tăng từ 50 lên 100 (đã gọi `get_incident_details` để tra cứu ID), nhưng vì không sửa nhắm trực tiếp vào H3 nên chưa đủ cơ sở kết luận do thay đổi prompt hay do biến thiên ngẫu nhiên giữa các lần chạy.
- Lỗi còn lại hoặc mới xuất hiện: Ở case H6 (95 điểm) và A6 (90 điểm), agent gọi lặp `get_incident_details` cho cùng incident `xr3myzj7ms51` ở step 2 và step 3, làm tốn thêm 1 lượt gọi model và token mà không có thêm thông tin mới. Ngoài ra một số case dao động giảm nhẹ điểm judge (A6: 92 → 90, H1: 95 → 93, H7: 99 → 95).

## 5. Giới hạn

- **Giới hạn của điểm judge:** LLM judge dùng chung model (`gpt-6-luna`) với agent nên có thể thiên lệch và chỉ mang tính ước lượng tương đối; cần đối chiếu trực tiếp với `expected`, `grade.mechanical` và `trace`.
- **Giới hạn của snapshot:** Bộ eval chạy trên dữ liệu snapshot cố định chụp ngày 09/11/2026 (giới hạn 50 incidents gần nhất, mỗi incident chỉ giữ 2 update mới nhất và 1 update đầu tiên, cắt tối đa 900 ký tự), nên không phản ánh trọn vẹn hành vi trên dữ liệu thực tế hoặc khi API thay đổi.
- **Giới hạn của một lần chạy mỗi phiên bản:** Mỗi phiên bản chỉ chạy 1 lần trên bộ 22 case nên kết quả chịu ảnh hưởng của tính ngẫu nhiên từ LLM, chưa chứng minh được độ ổn định lâu dài. Bên cạnh đó, trong `eval_after.json` không có case nào kích hoạt `validation_missing_source`, nên phần kiểm tra nguồn mới chỉ được kiểm chứng qua unit test.
