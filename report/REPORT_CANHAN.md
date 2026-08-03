# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Cao Hữu Phúc
**Nhóm:** Hiệp Đẹp Zai v1
**Ngày:** 03/08/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Khi hai đoạn văn bản có độ tương tự cosine cao, hai vector embedding của chúng hướng gần giống nhau, nghĩa là chúng có ý nghĩa hoặc ngữ cảnh tương đồng. Đây thường là dấu hiệu cho thấy hai câu/đoạn có nội dung liên quan.

**Ví dụ có độ tương tự CAO:**
- Câu A: "Sinh viên đang học bài trong thư viện."
- Câu B: "Một người học đang đọc sách ở thư viện."
- Tại sao tương đồng: Cả hai câu nói về cùng một tình huống học tập trong thư viện, nên ý nghĩa gần nhau.

**Ví dụ có độ tương tự THẤP:**
- Câu A: "Sinh viên đang học bài trong thư viện."
- Câu B: "Xe hơi đang chạy trên đường cao tốc."
- Tại sao khác: Hai câu mô tả hai chủ đề hoàn toàn khác nhau, nên vector embedding của chúng không giống nhau.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine similarity chú trọng vào hướng của vector, tức là mức độ giống về ý nghĩa, còn Euclidean distance lại bị ảnh hưởng nhiều bởi độ dài và kích thước vector. Với text embeddings, điều này khiến cosine phù hợp hơn để đo sự tương đồng ngữ nghĩa.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> Phép tính: $\lceil (10000 - 50) / (500 - 50) \rceil = \lceil 9950 / 450 \rceil = \lceil 22.11 \rceil = 23$
>
> **Đáp án:** Số chunks là 23.

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Nếu overlap tăng lên 100, số chunks sẽ là $\lceil (10000 - 100) / (500 - 100) \rceil = \lceil 9900 / 400 \rceil = 25$. Vì bước dịch chuyển giữa các chunk nhỏ hơn, nên số chunk tăng lên. Tăng overlap giúp giữ lại ngữ cảnh ở giữa các chunk, giảm nguy cơ làm mất thông tin quan trọng ở ranh giới chunk.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Tôi sẽ dùng regex để tách văn bản thành các câu dựa trên dấu kết thúc câu như `.`, `!`, `?`, rồi gom các câu lại thành các chunk theo số câu tối đa quy định. Ngoài ra, tôi sẽ xử lý các trường hợp như khoảng trắng thừa, câu không có dấu kết thúc hoặc các đoạn văn bản ngắn để tránh tạo chunk rỗng hoặc mất thông tin.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Thuật toán sẽ cố gắng chia đoạn văn theo các separator theo thứ tự ưu tiên như dấu xuống dòng đôi, xuống dòng đơn, dấu chấm câu, khoảng trắng, rồi đệ quy tiếp trên các đoạn còn quá dài. Base case là khi đoạn văn đã ngắn hơn hoặc bằng kích thước chunk cho phép, lúc đó sẽ dừng lại và trả về kết quả để tránh chia quá mức.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> Mỗi chunk sẽ được nhúng thành vector rồi lưu vào store cùng với metadata và doc_id để có thể truy xuất lại sau này. Khi tìm kiếm, câu truy vấn cũng được nhúng thành vector và so sánh bằng tích vô hướng với các vector đã lưu, sau đó sắp xếp theo độ tương đồng giảm dần.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> Tôi sẽ thực hiện lọc metadata trước khi tính toán độ tương đồng để giảm số lượng candidate và giữ kết quả sát với ngữ cảnh cần tìm. Khi xóa một tài liệu, hệ thống sẽ xoá tất cả các chunk có cùng doc_id để đảm bảo dữ liệu được đồng bộ và không để lại chunk rác trong store.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> Hàm answer sẽ lấy top-k chunk phù hợp nhất, nối chúng thành một ngữ cảnh đầy đủ rồi tạo prompt cho LLM để trả lời câu hỏi một cách có căn cứ. Cách tiếp cận này giúp agent không chỉ dựa vào một chunk duy nhất mà còn tận dụng nhiều đoạn liên quan để đưa ra câu trả lời ổn định hơn.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
============================= test session starts =============================
platform win32 -- Python 3.11.9, pytest-9.1.1, pluggy-1.6.0 -- Python 3.11.9
collected 42 items

42 passed in 0.30s
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | The cat is sleeping on the sofa. | The cat is resting on the couch. | cao | -0.2015 (thấp) | Không |
| 2 | Students are studying in the library. | A car is driving on the highway. | thấp | 0.1224 (cao nhất trong 5 cặp) | Không |
| 3 | Python is a programming language. | Python is used for software development. | cao | -0.0360 (thấp) | Không |
| 4 | The sun rises in the east. | The moon rises in the night sky. | thấp | 0.0277 (thấp) | Có |
| 5 | Machine learning helps computers learn from data. | Learning from data is useful for machine learning systems. | cao | -0.0007 (thấp) | Không |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Cặp 1 và cặp 3 là những ví dụ đáng ngạc nhiên nhất vì câu có ý nghĩa gần nhau nhưng lại cho điểm tương tự thấp hoặc âm khi dùng mock embeddings. Điều này cho thấy embeddings trong lab này không phản ánh ngữ nghĩa hoàn toàn như cách con người cảm nhận; chúng chủ yếu là các vector xác định bằng quá trình hashing/định tuyến ngẫu nhiên, nên có thể không phản ánh đúng mức độ liên quan ngữ nghĩa.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|---|---|---:|---|---|
| 1 | Sinh viên chương trình chuẩn phải đăng ký tối thiểu bao nhiêu tín chỉ trong học kỳ chính? | Chunk `hus-course-registration` về khối lượng đăng ký, nêu tối thiểu 14 tín chỉ. | 0,762857 | Có | Trả lời đúng: tối thiểu 14 tín chỉ và cần có sự đồng ý của Thủ trưởng đơn vị đào tạo nếu đăng ký ít hơn. |
| 2 | Sinh viên phải học tối thiểu bao nhiêu tín chỉ để được xét học bổng khuyến khích học tập? | Chunk `hus-academic-scholarship` về điều kiện xét học bổng. | 0,731460 | Có | Trả lời đúng: tối thiểu 18 tín chỉ trong học kỳ xét, không tính các học phần bị loại trừ. |
| 3 | Chính sách học bổng ngành khoa học cơ bản bắt đầu tính cấp từ ngày nào? | Chunk `hus-basic-science-scholarship-2026` về thời điểm áp dụng. | 0,712989 | Có | Trả lời đúng: bắt đầu tính cấp từ ngày 01/09/2026 và áp dụng cho người tuyển sinh từ năm 2025. |
| 4 | Học bổng cao nhất cho sinh viên chương trình ưu tiên đầu tư năm 2026 là bao nhiêu? | Chunk `hus-admissions-scholarship-2026` về học bổng tuyển sinh. | 0,747856 | Có | Trả lời đúng: 35 triệu đồng/SV/năm, có thể nhận tới 140 triệu đồng/SV. |
| 5 | Sinh viên cần điểm trung bình bao nhiêu trong bốn học kỳ gần nhất để nộp hồ sơ học bổng du học? | Chunk `hus-study-abroad-scholarship` về điều kiện tham khảo. | 0,720324 | Có | Trả lời đúng: từ 8,2/10 trở lên trong bốn học kỳ gần nhất. |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Qua demo, tôi học được rằng chỉ dùng cùng một embedding chưa đủ để có retrieval tốt; cách chia chunk và metadata như `audience` ảnh hưởng rất lớn đến độ chính xác. Với các tài liệu học bổng, việc giữ đúng ngữ cảnh và lọc đúng đối tượng giúp tránh nhầm lẫn giữa các chính sách có từ khóa tương tự.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10/ 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |
