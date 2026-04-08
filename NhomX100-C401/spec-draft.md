# SPEC — AI Product Hackathon

**Nhóm:** X100
**Track:** ☑ VinUni-VinSchool · ☐ VinFast · ☐ Vinmec · ☐ XanhSM · ☐ Open
**Problem statement (1 câu):** *Phụ huynh & học sinh VinSchool mất 10–15 phút/lần tra thủ công qua PDF Cẩm nang hoặc chờ hotline để hỏi điểm, lịch thi, học phí — AI Agent kết hợp RAG công cộng + dữ liệu cá nhân trả lời tức thì < 5 giây, đúng theo từng học sinh.*

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | Phụ huynh (~96.000 người) mất 10–15 phút/lần tra PDF hoặc chờ hotline để hỏi điểm, lịch thi, học phí của con — AI trả lời tức thì bằng ngôn ngữ tự nhiên, cá nhân hoá theo từng học sinh đã xác thực | AI trả lời sai điểm số hoặc học phí → User thấy ngay vì có con số cụ thể để đối chiếu, báo sai qua nút "Phản hồi không đúng" → Agent escalate tới GVCN/phòng tài chính. AI trả lời sai chính sách → luôn kèm trích dẫn nguồn (tên tài liệu, chương, mục) để user tự kiểm tra | ~$0.015/query (Claude Sonnet), latency < 3s. Risk chính: dữ liệu học sinh nhạy cảm → cần RBAC chặt + audit log mọi truy vấn private |

**Automation hay augmentation?** ☑ Augmentation
Justify: *User thấy câu trả lời + nguồn trích dẫn, có thể verify độc lập. Với dữ liệu private (điểm, học phí), agent chỉ đọc và hiển thị — không tự thay đổi dữ liệu. Cost of reject = 0, user chỉ cần bỏ qua câu trả lời và gọi hotline như trước.*

**Learning signal:**

1. User correction đi vào đâu? → Log "Phản hồi không đúng" gắn với `{query, response, user_role, student_id_hash}` → queue review thủ công hàng tuần → cập nhật RAG index hoặc sửa tool prompt
2. Product thu signal gì để biết tốt lên hay tệ đi? → CSAT sau mỗi session (1–5 sao), Task Completion Rate (session có kết thúc bằng escalation không?), Hotline call volume so sánh before/after
3. Data thuộc loại nào? ☑ User-specific (điểm, lịch, học phí cá nhân) · ☑ Domain-specific (nội quy, chính sách VinSchool) · ☐ Real-time · ☑ Human-judgment (câu hỏi kỷ luật, tư vấn học tập)
   Có marginal value không? Model chưa biết dữ liệu cá nhân học sinh (private) → **có marginal value rõ ràng**. Nội quy công khai model có thể biết một phần nhưng không đủ chi tiết theo năm học → RAG vẫn cần thiết.

---


