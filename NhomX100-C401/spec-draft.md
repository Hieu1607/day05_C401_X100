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
## 2. User Stories — 4 paths

### Feature 1: Tra cứu thông tin cá nhân học sinh (Private Layer)

**Trigger:** Phụ huynh đã đăng nhập hỏi "Con tôi học kỳ này điểm Toán bao nhiêu?" → Agent xác định học sinh linked với tài khoản → gọi `get_student_grades()` → trả lời có cấu trúc

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | Điểm hiện ra theo bảng: môn / điểm thành phần / tổng kết / xếp loại. Có ghi rõ "Cập nhật lần cuối: [timestamp]". Phụ huynh đọc xong, không cần làm thêm gì |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | Phụ huynh có 2 con → agent hỏi "Bạn đang hỏi về Minh Khoa (lớp 9) hay Minh Anh (lớp 5)?". Chỉ 1 lần clarification — nếu vẫn mơ hồ, agent lấy con lớn hơn và thông báo rõ |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | API VinschoolOne trả về điểm cũ chưa cập nhật → agent hiển thị timestamp rõ ràng, user so sánh với app → bấm "Báo sai" → auto-escalate tới GVCN qua email template |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | Bấm "Phản hồi không đúng" → chọn loại lỗi (sai số liệu / thiếu thông tin / không liên quan) → ghi log `{query_hash, error_type, timestamp}` → review queue weekly → nếu là lỗi API sync thì ticket tới team VinschoolOne |

---

### Feature 2: Tra cứu chính sách & nội quy (Public RAG Layer)

**Trigger:** Bất kỳ user nào (kể cả chưa đăng nhập) hỏi về quy định, biểu phí, lịch tuyển sinh → Agent tìm trong vector DB (Cẩm nang PDF đã được index) → trả lời kèm trích dẫn nguồn

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | "Học phí THCS Advanced tại Hà Nội năm 2024–2025 là 280 triệu/năm (Nguồn: Biểu phí VinSchool 2024-2025, mục 3.2)." User đọc, có link nguồn để verify, không cần hỏi thêm |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | Câu hỏi không rõ cơ sở/cấp học → agent hỏi 1 lần: "Bạn đang hỏi cho cơ sở nào và cấp học nào?" Nếu user không biết → agent trả lời theo trường hợp phổ biến nhất và ghi chú "Có thể khác theo cơ sở, vui lòng xác nhận với nhà trường" |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | Agent dùng Cẩm nang 2023–2024 (index chưa cập nhật) → trả lời biểu phí cũ → user phát hiện khi đối chiếu với thông báo nhà trường → báo sai → team cập nhật PDF mới vào index |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | Nút "Tài liệu này đã lỗi thời" → log kèm `{source_chunk_id, academic_year}` → trigger re-index pipeline nếu có PDF mới, hoặc thêm vào watchlist để admin kiểm tra |

---

### Feature 3: Cá nhân hoá câu trả lời chính sách theo profile học sinh

**Trigger:** Phụ huynh đã đăng nhập hỏi câu hỏi chính sách chung ("Học phí bao nhiêu?") → Agent tự động kết hợp profile học sinh (cấp, chương trình, cơ sở) với RAG → trả lời cụ thể không cần hỏi lại

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | "Dựa trên hồ sơ của Minh Khoa (THCS Advanced, Royal City), học phí kỳ 2 là 140 triệu. Bao gồm: học phí chính 130tr + bán trú 10tr." Phụ huynh không cần nhập thêm thông tin |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | Profile học sinh có trường thiếu (ví dụ chương trình chưa xác định) → agent trả lời cả 2 trường hợp Standard và Advanced với note "Vui lòng xác nhận chương trình con đang học" |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | Agent dùng sai cơ sở (lấy phí Hà Nội cho học sinh TP.HCM) → user thấy số sai, báo sai → log lỗi mapping `campus_id` → fix trong metadata chunking |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | "Thông tin này không đúng với tôi" → chọn cụ thể điều gì sai → log chi tiết → nếu lỗi do metadata → fix chunk và re-embed |

---

## 3. Eval metrics + threshold
**Optimize precision hay recall?** ☑ Precision
Tại sao? Với dữ liệu cá nhân (điểm, học phí), sai 1 con số gây mất tin tưởng nghiêm trọng hơn là không trả lời được. Phụ huynh chấp nhận "Tôi không tìm thấy thông tin này" hơn là nhận số liệu sai.
Nếu sai ngược lại (chọn recall): Agent trả lời nhiều hơn nhưng sai thường xuyên hơn → phụ huynh nhận học phí sai, điểm sai → mất tin tưởng hoàn toàn, abandon product.

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| Answer Correctness (human eval, dữ liệu private) | ≥ 90% | < 80% trong 1 sprint |
| Hallucination Rate (câu trả lời bịa đặt không có trong nguồn) | ≤ 3% | > 8% trong bất kỳ ngày nào |
| RAG Faithfulness (RAGAS score, câu trả lời bám nguồn) | ≥ 0.80 | < 0.65 |
| Citation Rate (% câu trả lời chính sách có dẫn nguồn) | ≥ 95% | < 85% |
| Task Completion Rate (% session user resolve được ý định) | ≥ 75% | < 60% |
| Auth Bypass Detection (% prompt injection bị chặn) | 100% | Bất kỳ 1 incident bypass thành công |
| P95 Latency | < 5 giây | > 10 giây liên tục 1 giờ |

---

## 4. Top 3 failure modes

*"Failure mode nào user KHÔNG BIẾT bị sai? Đó là cái nguy hiểm nhất."*

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | Agent trả lời điểm số hoặc học phí từ cache/index cũ trong khi giáo viên đã cập nhật dữ liệu mới trong VinschoolOne — user không biết đang xem dữ liệu stale | Phụ huynh tin điểm đúng, không theo dõi kết quả học thật. Phụ huynh tin học phí đã đóng đủ nhưng thực ra còn thiếu → bị phạt trễ hạn. **User không biết mình bị sai** vì không có dấu hiệu cảnh báo | Timestamp bắt buộc trên mọi câu trả lời dữ liệu private. TTL cache ≤ 30 phút. Nếu API call fail → KHÔNG dùng cache, báo rõ "Không thể truy xuất dữ liệu mới nhất lúc này" |
| 2 | Phụ huynh hỏi về chính sách mà agent lấy từ Cẩm nang năm học cũ (PDF chưa được re-index sau khi nhà trường cập nhật) — agent trả lời tự tin, có trích dẫn nguồn nhưng nguồn đó đã lỗi thời | Phụ huynh chuẩn bị hồ sơ sai, đóng học phí theo biểu cũ, hoặc cho con vi phạm nội quy theo quy định đã thay đổi. Trích dẫn nguồn tạo cảm giác tin tưởng giả → **nguy hiểm hơn là không trích dẫn** | Mỗi chunk metadata phải có `academic_year` và `indexed_date`. Query luôn filter theo năm học hiện tại. Nếu tài liệu năm mới chưa có → agent thông báo rõ "Thông tin dưới đây từ năm học 2024–2025, chưa có cập nhật mới hơn" |
| 3 | Agent hiển thị đúng dữ liệu nhưng sai học sinh — xảy ra khi phụ huynh có nhiều con, session context bị nhầm, hoặc lỗi mapping `parent_id → student_id` | Phụ huynh theo dõi nhầm tiến độ học tập của con — có thể không phát hiện nếu điểm hai con tương đương. Trong trường hợp tệ hơn: staff tra cứu nhầm hồ sơ học sinh và tư vấn sai cho phụ huynh khác | Server-side enforce mapping `parent_id → [student_ids]` — không tin vào bất kỳ `student_id` nào client truyền lên mà không qua verify. Luôn hiển thị tên đầy đủ + lớp + cơ sở của học sinh trong mọi response để phụ huynh cross-check |

---