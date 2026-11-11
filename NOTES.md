# Ghi chú phân tích · Day 04

Họ và tên: Đặng Đức Minh
MSSV: 2A202603886

## 1. Case được chọn

- ID case: A5
- Câu trả lời của agent:
> "Dựa trên dữ liệu từ GitHub Status, không có sự cố nào liên quan đến GitHub Actions được ghi nhận trong năm 2025. Kết quả cho thấy tất cả các hệ thống đã hoạt động bình thường vào thời điểm ghi nhận trạng thái (ngày 9 tháng 11 năm 2026), cũng như không có sự cố nào được lưu lại cho năm 2025.
>
> Tuy nhiên, có một số sự cố liên quan đến GitHub Actions trong khoảng thời gian gần đây, cụ thể như sau:
> 1. Incident with Actions (ID: t89g2q1800fr) - 05/11/2026
> 2. Incident with Actions (ID: h02n9572mx9y) - 04/11/2026
> 3. Incident with Actions and Pages (ID: xr3myzj7ms51) - 04/11/2026
> 4. Incident with Pull Requests, Git Operations and Issues (ID: smtdswd1h2w4) - 04/11/2026
> 5. Degraded performance for multiple GitHub services (ID: pfbmpxxfq1cy) - 06/11/2026
>
> Đặc biệt, các sự cố này xảy ra trong thời gian gần đây và không liên quan đến năm 2025.
>
> **Kết luận**: Kết quả là đầy đủ với thông tin từ các sự cố gần đây, và không có sự cố nào đã được ghi nhận trong năm 2025. Dữ liệu hiển thị chỉ có các sự cố từ năm 2026."
- Điểm ở lần chạy 1: 30 / 100 (Factuality: 0/40, Completeness: 10/25, Evidence: 15/20, Boundaries: 5/15)

## 2. Bằng chứng trong trace

Đoạn trace của tool `get_status` trong lần chạy 1:
```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "provider": "github",
    "component": "Actions"
  },
  "result": {
    "provider": "github",
    "captured_at": "2026-11-09T03:50:47.065020+00:00",
    "overall": {
      "indicator": "none",
      "description": "All Systems Operational"
    },
    "components": [
      {
        "name": "Actions",
        "status": "operational"
      }
    ],
    "incidents": [
      {
        "id": "pfbmpxxfq1cy",
        "name": "Degraded performance for multiple GitHub services",
        "status": "resolved",
        "updated_at": "2026-11-06T00:20:49.369Z"
      },
      {
        "id": "t89g2q1800fr",
        "name": "Incident with Actions",
        "status": "resolved",
        "updated_at": "2026-11-05T17:42:35.455Z"
      }
    ],
    "matching_incidents_in_last_50": 23,
    "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.",
    "source": "https://www.githubstatus.com"
  }
}
```

**Nguyên nhân có thể xảy ra:**
- Tool `get_status` trả về danh sách nằm trong phạm vi 50 sự cố gần nhất (tất cả các mục trả về đều rơi vào đầu tháng 11/2026). Tool đã đính kèm ghi chú cảnh báo: `"Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố."`
- Tuy nhiên, Agent đã suy diễn quá mức (hallucination / false negative inference): Agent thấy 50 incident gần nhất không có ngày nào trong năm 2025 thì vội vàng kết luận *"không có sự cố nào trong năm 2025 và kết quả là đầy đủ"*.
- Judge đã trừ toàn bộ 40 điểm Factuality vì khẳng định sai phạm vi dữ liệu, vi phạm nghiêm trọng nguyên tắc ranh giới bằng chứng (evidence boundary).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: Cài đặt cơ chế **Evidence Gate** (Source validation). Khi các lần gọi `get_incident_details` thành công, lưu lại các URL incident (`observed_incident_urls`). Khi model chuẩn bị đưa ra câu trả lời cuối cùng (`finish == 'stop'`), kiểm tra xem câu trả lời có chứa ít nhất một URL nguồn hay không. Nếu thiếu, ghi event `validation_missing_source` vào `trace` và gửi phản hồi yêu cầu model bổ sung URL nguồn nếu còn trong ngân sách `max_steps`. Nếu hết ngân sách, trả về `status = 'max_steps'`.
  2. Trong `agent.py`: Nâng cấp `SYSTEM_PROMPT` với chỉ dẫn tường minh về giới hạn phạm vi dữ liệu:
     > *"Lưu ý quan trọng: Dữ liệu snapshot chỉ chứa tối đa 50 sự cố gần nhất. Khi người dùng hỏi về các năm trước (như 2025) hoặc yêu cầu danh sách đầy đủ, bạn phải nêu rõ giới hạn 50 sự cố gần nhất này, không được khẳng định là không có sự cố nào trong các giai đoạn đó và không được tự nhận là danh sách đầy đủ.*
     > *Khi đã tra cứu chi tiết sự cố bằng get_incident_details, bắt buộc phải dẫn URL nguồn chính xác trong câu trả lời cuối cùng."*
- Vì sao chọn thay đổi này:
  - Thay đổi này giải quyết trực tiếp gốc rễ lỗi ở case A5: nhắc nhở mô hình luôn ý thức về giới hạn dữ liệu 50 incidents, tránh ngộ nhận "không thấy tức là không có", đồng thời đảm bảo tính liêm chính bằng chứng bằng cách bắt buộc dẫn link nguồn từ tool.
  - Đồng thời hoàn thiện 100% hợp đồng kiểm tra nguồn (Evidence Gate) đã được định nghĩa trong bộ test chuẩn `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 76.82 / 100 | **82.05 / 100** (+5.23) |
| Điểm case đã chọn (A5) | 30 / 100 | **100 / 100** (+70.00) |
| Số model calls | 49 calls agent (+ 22 calls judge) | 49 calls agent (+ 22 calls judge) |
| Token, chi phí nếu có | 82,037 tokens (~$0.0179) | 78,887 tokens (~$0.0169) |

- Lỗi đã giải quyết:
  - **Case A5 tăng vọt từ 30 lên 100 điểm tuyệt đối:** Agent trả lời chính xác, nêu rõ chỉ có tối đa 50 sự cố gần nhất và từ chối khẳng định không có sự cố trong năm 2025, được Judge đánh giá Factuality 40/40, Completeness 25/25, Evidence 20/20, Boundaries 15/15.
  - Điểm số nhiều case khác tăng mạnh: Case E6 tăng từ 80 lên 100, Case A1 tăng từ 75 lên 100, Case H4 tăng từ 80 lên 100, Case H5 tăng từ 75 lên 90, Case E3 tăng từ 75 lên 80, Case E5 tăng từ 75 lên 80.
  - Cơ chế Evidence Gate vận hành trơn tru: Toàn bộ 37/37 unit tests pass 100%, đảm bảo mọi tuyên bố về chi tiết sự cố đều được neo với URL nguồn có thật từ tool.
- Lỗi còn lại hoặc mới xuất hiện:
  - Case H1 (60đ) và H6 (60đ) bị trần điểm cơ học (score_cap: 60) do ràng buộc `min_tool_steps: 2` (model có xu hướng gọi song song 2 tool calls trong 1 model step duy nhất thay vì chia thành 2 lượt gọi model liên tiếp).
  - Case H2 (50đ): Mô hình bị trừ điểm do chưa phân biệt rành mạch giữa sự cố đã giải quyết trong snapshot và khẳng định của người dùng.

## 5. Giới hạn

- **Giới hạn của LLM Judge:** Điểm số của LLM Judge là một phép đánh giá xấp xỉ mang tính heuristic. Do tính ngẫu nhiên (stochastic nature) của mô hình ngôn ngữ lớn, cùng một câu trả lời có thể nhận điểm số hơi chênh lệch giữa các lần chạy dựa trên văn phong hoặc cách đánh giá từng khía cạnh.
- **Giới hạn của Snapshot:** Dữ liệu snapshot được cố định tại ngày 09/11/2026 và chỉ bao gồm 50 sự cố gần nhất. Dữ liệu này không phản ánh tình trạng hệ thống thời gian thực hiện tại và không thể dùng để kết luận chắc chắn cho các giai đoạn lịch sử trước đó ngoài phạm vi snapshot.
- **Giới hạn của việc chạy một lần mỗi phiên bản:** Một lần chạy duy nhất cho mỗi phiên bản không đủ để chứng minh độ ổn định thống kê tuyệt đối. Tuy nhiên, mức tăng điểm rõ rệt từ 76.82 lên 82.05 (đặc biệt là bước nhảy từ 30 lên 100 điểm ở case A5) là minh chứng xác thực cho hiệu quả của việc thiết lập Evidence Gate và tinh chỉnh prompt ranh giới bằng chứng.
