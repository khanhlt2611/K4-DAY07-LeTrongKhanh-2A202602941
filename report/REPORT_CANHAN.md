# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Lê Trọng Khánh
**Nhóm:** G-08
**Ngày:** 20/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Độ tương tự cosine cao, gần 1, nghĩa là hai vector embedding có hướng gần nhau; với mô hình nhúng ngữ nghĩa tốt, điều này cho thấy hai đoạn văn bản có nội dung hoặc ý nghĩa tương đồng. Hai câu có thể dùng từ ngữ khác nhau nhưng vẫn có độ tương tự cao nếu cùng diễn đạt một ý.

**Ví dụ có độ tương tự CAO:**

- Câu A: Tôi muốn trả lại sản phẩm và nhận lại tiền.
- Câu B: Tôi muốn hoàn hàng để được hoàn tiền.
- Tại sao tương đồng: Cả hai câu đều diễn đạt mong muốn trả hàng và nhận lại tiền.

**Ví dụ có độ tương tự THẤP:**

- Câu A: Tôi muốn trả lại sản phẩm và nhận lại tiền.
- Câu B: Trái Đất quay quanh Mặt Trời.
- Tại sao khác: 2 câu nói về 2 chủ đề không liên quan đến nhau.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine tập trung so sánh hướng của vector, không phụ thuộc vào độ lớn, nên thường phù hợp để đo sự tương đồng ngữ nghĩa khi độ lớn của embedding không phản ánh mức độ liên quan; khoảng cách Euclid trên vector chưa chuẩn hóa chịu ảnh hưởng của cả hướng lẫn độ lớn. Nếu các vector đã được chuẩn hóa về độ dài 1, hai cách đo cho cùng thứ tự xếp hạng mức độ tương đồng.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
`ceil((10000 - 50) / (500 - 50)) = ceil(22,111...) = 23 chunks`.

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Khi overlap tăng lên 100, (10000-500)/400 + 1 = 24,75 => 25 chunks, số chunk tăng lên 2. Overlap lớn hơn giúp giữ ngữ cảnh ở ranh giới các chunk, hạn chế mất thông tin khi một ý bị chia đôi, nhưng làm tăng dữ liệu trùng lặp và chi phí tạo embedding, lưu trữ.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Tôi dùng regex `(?<=[.!?])\s+` để tách tại khoảng trắng sau dấu kết thúc câu mà vẫn giữ dấu câu. Sau khi loại khoảng trắng đầu/cuối, tôi gom tối đa `max_sentences_per_chunk` câu thành một chunk; chuỗi rỗng hoặc chỉ có khoảng trắng trả về `[]`. Cách này chưa phân biệt được dấu chấm kết thúc câu với chữ viết tắt như `TS.` hoặc `v.v.` khi phía sau có khoảng trắng.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Tôi ưu tiên tách theo đoạn, dòng, câu rồi khoảng trắng, giữ separator trong các mảnh để không mất nội dung. Các mảnh nhỏ liền kề được gom trong giới hạn `chunk_size`; mảnh quá dài được chia đệ quy với các separator còn lại. Trường hợp dừng gồm văn bản rỗng, văn bản đã nằm trong giới hạn và hết separator hoặc gặp separator rỗng thì cắt cứng theo kích thước; `chunk_size <= 0` gây `ValueError`.

**`compute_similarity` và `ChunkingStrategyComparator.compare`:**
> Tôi tính cosine bằng tích vô hướng chia cho tích độ dài hai vector, trả `0.0` nếu một vector có độ dài bằng 0. Comparator chạy ba chiến lược trên cùng văn bản, trả `count`, `avg_length`, `chunks` cho từng chiến lược; văn bản rỗng có độ dài trung bình bằng 0 để tránh chia cho 0.

**Kiểm chứng Checkpoint 3:** chạy `.venv\Scripts\python.exe -m pytest tests/ -k "Chunker or Similarity or Compare" -v` trên Python 3.13.14 cho kết quả `23 passed, 19 deselected, 1 warning in 0.04s`. Cảnh báo do pytest không ghi được cache, không phải lỗi test. Kiểm tra trực tiếp bằng `FixedSizeChunker` xác nhận overlap 50 cho 23 chunks và overlap 100 cho 25 chunks.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> Tôi dùng store trong bộ nhớ, mỗi `Document` tạo một record chứa id, content, bản sao metadata và embedding; chunking thực hiện trước khi nạp vào store. Khi tìm kiếm, tôi embed câu hỏi, tính dot product với từng embedding rồi sắp xếp điểm giảm dần và lấy top-k; dot product tương đương cosine khi vector đã chuẩn hóa. Kết quả không chứa vector embedding để dễ đọc; store rỗng hoặc `top_k <= 0` trả `[]`.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> Tôi lọc các record khớp tất cả trường metadata trước khi tìm kiếm, rồi dùng chung `_search_records` với `search` để bảo đảm cách xếp hạng nhất quán. `delete_document` xóa mọi record có `metadata['doc_id']` khớp và trả `True` khi có bản ghi bị xóa. Khi nạp các chunk, tôi giữ `doc_id` của tài liệu gốc nếu được cung cấp; nếu thiếu trường này, store mặc định dùng `Document.id`.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> Agent truy xuất top-k, đánh số các chunk `[1]`, `[2]` cùng nguồn và chunk id, rồi đưa câu hỏi và ngữ cảnh vào prompt trước khi gọi `llm_fn`. Prompt yêu cầu chỉ trả lời từ ngữ cảnh, dẫn nguồn theo số và nói rõ khi thiếu thông tin; nội dung tài liệu được coi là dữ liệu tham khảo. Nếu không có kết quả truy xuất, agent trả thông báo không tìm thấy thông tin và không gọi LLM.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
.venv\Scripts\python.exe -m pytest tests/ -v -p no:cacheprovider
============================= test session starts =============================
platform win32 -- Python 3.13.14, pytest-9.1.1, pluggy-1.6.0 -- D:\\VIN AI th\u1ef1c chi\u1ebfn\\Lab\\K4-DAY07-LeTrongKhanh-2A202602941\\.venv\\Scripts\\python.exe
rootdir: D:\\VIN AI th\u1ef1c chi\u1ebfn\\Lab\\K4-DAY07-LeTrongKhanh-2A202602941
collecting ... collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED   [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED    [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED   [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

============================= 42 passed in 0.07s ==============================
```

**Số lượng bài test vượt qua (pass):** 42 / 42.

Kiểm thử thực tế trên Python 3.13.14 trong `.venv` hiện có; chưa xác minh riêng trên Python 3.11. Tùy chọn `-p no:cacheprovider` tắt cache để tránh cảnh báo quyền ghi thư mục `.pytest_cache`, không bỏ qua tests.

**Kiểm tra bổ sung Checkpoint 4:** dùng vector xác định trước để kiểm tra filter trước top-k, sao chép metadata, `top_k=0`, kết quả search nhất quán, prompt có câu hỏi và nguồn, xóa nhiều chunk cùng `doc_id`, và không gọi LLM khi store rỗng; tất cả đều đạt.

**Demo:** chạy `main.py "Chunking là gì?"` với `EMBEDDING_PROVIDER=mock`, nạp và lưu 5 tài liệu, trả top-3 và gọi thành công `demo_llm`. File mẫu `data/customer_support_playbook.txt` không tồn tại được bỏ qua đúng thiết kế. Demo dùng embedding và LLM giả lập nên chỉ xác minh luồng chạy; chưa chứng minh chất lượng trả lời ngữ nghĩa trên corpus thương mại điện tử.

---

### Checkpoint 5 — Chiến lược benchmark cá nhân

Tôi chọn `SentenceChunker(max_sentences_per_chunk=5)`. Mục tiêu là giữ nguyên câu trong các điều khoản trả hàng/hoàn tiền; đánh đổi là kích thước chunk không cố định và danh sách dài có thể không được tách tốt. Corpus gồm 8 file được khai báo trong `data/marketplace-returns/sources.csv`; chỉ phần thân được chunk, frontmatter được giữ làm metadata cho từng chunk. Mỗi chunk có id `doc_id#chunk_index` và giữ `metadata['doc_id']` của nguồn.

Lệnh đã chạy:

```powershell
.venv\Scripts\python.exe bench.py --chunker sentence --embedding gemini --compare-chunkers --output ket_qua_benchmark.txt
```

Kết quả chạy lúc `2026-09-20T04:53:56Z`: Gemini `gemini-embedding-001` tạo embedding cho 100 chunks và chạy đủ top-3 của 5 câu hỏi. File `ket_qua_benchmark.txt` lưu score, `doc_id`, audience, category, chunk index, preview và kết quả A/B filter; baseline trên 3 tài liệu được ghi ở mục 2 báo cáo nhóm. Q1 dùng `audience=both`, Q3 dùng `buyer`, Q5 dùng `seller`; Q2 và Q4 không lọc.

Theo phép đo cấp tài liệu, Gold@1=4/5, Gold@3=5/5 và document score=9/10. Q1 có gold document ở hạng 2; Q2–Q5 có gold document ở hạng 1. A/B filter không đổi hạng của gold document: Q1 giữ hạng 2, Q3 và Q5 giữ hạng 1. Tuy nhiên, kiểm tra nội dung cho thấy Q1 có bằng chứng đầy đủ ở top-3 và Q2 có ở top-1; Q3 chỉ có các phần bằng chứng rải trong top-3, còn Q4 và Q5 đúng `doc_id` nhưng top-3 chưa chứa chunk trả lời chuẩn. `bench.py` chỉ đo retrieval; sau đó tôi chạy riêng `KnowledgeBaseAgent` với đúng top-3 đã ghi nhận và Gemini `gemini-3.5-flash-lite` làm `llm_fn` để kiểm tra câu trả lời. Kết quả xác nhận Q1–Q2 trả lời đúng, còn Q3–Q5 thiếu bằng chứng cần thiết; đây là các failure case dùng cho Checkpoint 6.

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Tôi muốn trả lại sản phẩm và nhận lại tiền. | Tôi muốn hoàn hàng để được hoàn tiền. | Cao | 0.893532 | Đúng |
| 2 | Shopee hoàn phí vận chuyển ban đầu khi trả toàn bộ đơn. | Phí giao hàng lúc mua được hoàn nếu người mua hoàn trả tất cả sản phẩm. | Cao | 0.830635 | Đúng |
| 3 | Người mua cần quay video mở kiện để làm bằng chứng. | Trái Đất quay quanh Mặt Trời. | Thấp | 0.549361 | Đúng; thấp nhất trong 5 cặp |
| 4 | Người bán phải khắc phục trong vòng 48 giờ. | Người bán cần thực hiện biện pháp xử lý trong hai ngày. | Cao | 0.857982 | Đúng |
| 5 | Người bán gửi trả sản phẩm cho người mua. | Người mua gửi trả sản phẩm cho người bán. | Thấp | 0.961765 | Sai; thực tế cao nhất |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Cặp 5 bất ngờ nhất: dù chủ thể và đối tượng bị đảo ngược, cosine vẫn đạt 0.961765, cao nhất trong năm cặp. Kết quả cho thấy embedding nắm rất mạnh chủ đề và các từ cùng xuất hiện, nhưng một vector duy nhất có thể làm mờ quan hệ vai trò giữa “người bán” và “người mua”; vì vậy cosine cao chưa đủ để kết luận hai câu có cùng nghĩa. Các điểm được đo bằng Gemini `gemini-embedding-001` và hàm `compute_similarity` trong `src`.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Hoàn Tiền Ngay và Trả hàng & Hoàn tiền khác nhau thế nào? | `shopee-return-refund-policy#21`: các trường hợp hoàn tiền, có nhắc hoàn tiền không cần trả hàng nhưng thiếu mốc gửi trả 6 ngày | 0.812345 | Một phần; chunk đủ đáp án nằm trong top-3 | Trả lời đúng: Hoàn Tiền Ngay không cần trả hàng; Trả hàng & Hoàn tiền phải gửi trả trong 6 ngày, dẫn nguồn `[3]` |
| 2 | Có hoàn phí vận chuyển ban đầu khi chỉ trả một số sản phẩm không? | `shopee-return-shipping-fees#3`: phí ban đầu chỉ hoàn khi trả toàn bộ; trả một số sản phẩm thì không hoàn | 0.811383 | Có, top-1 chứa đầy đủ đáp án | Trả lời đúng là không được hoàn phí vận chuyển ban đầu, dẫn nguồn `[1]` |
| 3 | Cần bằng chứng gì khi sản phẩm lỗi, hư hỏng hoặc khác mô tả? | `shopee-return-evidence#0`: phần mở đầu hướng dẫn bằng chứng, chưa có đầy đủ tiêu chí video | 0.864166 | Một phần; các chi tiết bằng chứng nằm rải trong top-3 | Agent nói rõ top-3 không có thông tin cụ thể để trả lời đầy đủ và không suy đoán |
| 4 | Người bán phải khắc phục trong bao lâu khi TikTok xử có lợi cho khách hàng? | `tiktok-aftersales-disputes#7`: mục lục và câu hỏi thường gặp, không chứa quy định khắc phục trong 48 giờ | 0.853233 | Không; đúng tài liệu nhưng sai chunk | Agent trả lời không tìm thấy thông tin trong ngữ cảnh |
| 5 | Sau khi được liên hệ, người bán có bao lâu và phải làm gì để gửi trả hàng? | `tiktok-seller-to-customer-returns#0`: phần tổng quan trường hợp gửi lại hàng, chưa có thời hạn và các bước thực hiện | 0.848914 | Không; đúng tài liệu nhưng sai chunk | Agent chỉ tìm thấy hướng dẫn chờ nhân viên chăm sóc khách hàng, không tìm thấy thời hạn và các bước gửi trả |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 3 / 5. Trong đó Q1 và Q2 có đủ bằng chứng để trả lời; Q3 chỉ có bằng chứng một phần. Q4 và Q5 là failure cases: hệ thống lấy đúng tài liệu nhưng sai section/chunk.

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Qua việc đối chiếu SentenceChunker với fixed-size và heading, tôi nhận ra không có một kích thước chunk tốt cho mọi câu hỏi. 

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 7 / 10 |
| **Tổng phần cá nhân** | **57 / 60** |
