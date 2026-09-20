# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

**Nhóm:** [Tên nhóm]
**Thành viên:** [Họ tên từng thành viên]
**Ngày:** [Ngày nộp]

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Chủ đề (Domain) & Lý Do Chọn

**Chủ đề:** Chính sách trả hàng, hoàn tiền và xử lý hậu mãi Shopee/TikTok Shop.

**Tại sao nhóm chọn chủ đề này?**
> Corpus có các điều kiện, thời hạn và yêu cầu bằng chứng cụ thể để kiểm chứng câu trả lời. Nội dung hướng đến người mua và người bán giúp thiết kế thử nghiệm metadata filter, đồng thời kiểm tra việc chunking có giữ đủ điều kiện và ngoại lệ hay không.

### Danh sách tài liệu (Data Inventory)

| # | Tên tài liệu | Nguồn (Source URL) | Ngày lấy / Phiên bản | Số ký tự | Metadata đã gán |
|---|--------------|------------|--------------------|----------|-----------------|
| 1 | Quy trình Shopee xử lý yêu cầu trả hàng và hoàn tiền | [Nguồn](https://help.shopee.vn/portal/4/article/190242) | 2026-09-20 / not-stated | 8114 | audience=both; category=return-process; language=vi |
| 2 | Chuẩn bị bằng chứng khi yêu cầu trả hàng và hoàn tiền Shopee | [Nguồn](https://help.shopee.vn/portal/4/article/79467) | 2026-09-20 / not-stated | 3469 | audience=buyer; category=return-evidence; language=vi |
| 3 | Chính sách trả hàng và hoàn tiền Shopee | [Nguồn](https://help.shopee.vn/portal/4/article/77251) | 2026-09-20 / effective-2026-03-11 | 19616 | audience=buyer; category=return-refund; language=vi |
| 4 | Phương thức gửi hàng hoàn trả và phí hoàn trả Shopee | [Nguồn](https://help.shopee.vn/portal/4/article/189477) | 2026-09-20 / not-stated | 5933 | audience=buyer; category=return-shipping; language=vi |
| 5 | Nâng cấp tranh chấp hậu mãi trên TikTok Shop | [Nguồn](https://seller-vn.tiktok.com/university/essay?knowledge_id=101756645132049) | 2026-09-20 / 2025-06-18 | 7281 | audience=seller; category=aftersales-dispute; language=vi |
| 6 | Trả hàng do đổi ý trên TikTok Shop | [Nguồn](https://seller-vn.tiktok.com/university/essay?knowledge_id=6988871880738576) | 2026-09-20 / 2026-08-21 | 9147 | audience=buyer; category=change-of-mind; language=vi |
| 7 | Chính sách trả hàng và hoàn tiền TikTok Shop | [Nguồn](https://seller-vn.tiktok.com/university/essay?knowledge_id=1766935302801169) | 2026-09-20 / 2026-08-21 | 20071 | audience=both; category=return-refund; language=vi |
| 8 | Người bán gửi trả sản phẩm cho người mua trên TikTok Shop | [Nguồn](https://seller-vn.tiktok.com/university/essay?knowledge_id=4041059496167184) | 2026-09-20 / 2026-08-20 | 5468 | audience=seller; category=seller-return; language=vi |

Số ký tự tính trên phần thân sau khi bỏ frontmatter, trước khi chunk. Dữ liệu hiện còn tiêu đề lặp, footer và bảng bị dàn phẳng; số liệu dưới đây đo trên bản crawl hiện có.

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**
- [ ] Tập tài liệu (Corpus) chỉ chứa nguồn công khai/được phép dùng và không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ.
- [x] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata.

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho truy xuất (retrieval)? |
|----------------|------|---------------|-------------------------------|
| doc_id | string | shopee-return-evidence | Liên kết mọi chunk với nguồn gốc. |
| title | string | Chuẩn bị bằng chứng | Hiển thị tên nguồn. |
| source_url | string | URL bài chính sách | Truy vết và kiểm chứng. |
| retrieved_at | string YYYY-MM-DD | 2026-09-20 | Ngày thu thập. |
| document_version | string | not-stated | Theo dõi phiên bản do corpus khai báo. |
| audience | string | buyer / seller / both | Lọc đúng vai trò; so khớp chính xác không tự bao gồm both. |
| category | string | return-evidence | Phân loại chủ đề. |
| language | string | vi | Ngôn ngữ nội dung. |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

> Mỗi thành viên thử **một chiến lược khác nhau** trên cùng bộ tài liệu; nhóm tổng hợp và so sánh ở đây.

### Phân tích đường cơ sở (Baseline Analysis)

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Chiến lược (Strategy) | Số lượng Chunk | Độ dài trung bình | Giữ được ngữ cảnh không? |
|-----------|----------|-------------|------------|-------------------|
| shopee-return-evidence | fixed_size | 8 | 477.38 | Giới hạn 500 ký tự; overlap 50 nhưng có thể cắt ngang câu. |
| shopee-return-evidence | by_sentences | 12 | 286.08 | Giữ dấu câu, gom tối đa 3 câu; không giới hạn 500 ký tự. |
| shopee-return-evidence | recursive | 9 | 385.44 | Ưu tiên đoạn/dòng; có thể tách điều kiện khỏi tiêu đề. |
| shopee-return-refund-policy | fixed_size | 44 | 494.68 | Giới hạn 500 ký tự; overlap 50 nhưng có thể cắt ngang câu. |
| shopee-return-refund-policy | by_sentences | 48 | 405.79 | Giữ dấu câu, gom tối đa 3 câu; không giới hạn 500 ký tự. |
| shopee-return-refund-policy | recursive | 71 | 276.28 | Ưu tiên đoạn/dòng; có thể tách điều kiện khỏi tiêu đề. |
| tiktok-aftersales-disputes | fixed_size | 17 | 475.35 | Giới hạn 500 ký tự; overlap 50 nhưng có thể cắt ngang câu. |
| tiktok-aftersales-disputes | by_sentences | 14 | 516.86 | Giữ dấu câu, gom tối đa 3 câu; không giới hạn 500 ký tự. |
| tiktok-aftersales-disputes | recursive | 17 | 428.29 | Ưu tiên đoạn/dòng; có thể tách điều kiện khỏi tiêu đề. |

Baseline dùng cùng phần thân tài liệu; Fixed-size và Recursive đặt 500 ký tự, còn comparator dùng Sentence mặc định 3 câu/chunk. Độ dài trung bình của Sentence trên tài liệu tranh chấp là 516,86 ký tự, cho thấy giới hạn số câu không bảo đảm giới hạn ký tự. Lượt benchmark cá nhân bên dưới dùng 5 câu/chunk nên không lấy trực tiếp số chunk của baseline làm kết quả benchmark.

### Chiến lược của từng thành viên

> Mỗi thành viên điền một khối dưới đây (copy thêm nếu nhóm có nhiều hơn 3 người).

**Lê Trọng Khánh — SentenceChunker**

- Tham số: `max_sentences_per_chunk=5`; lần benchmark mới nhất tạo 100 chunks từ 8 tài liệu.
- Tách tại khoảng trắng sau dấu `.`, `!`, `?`, giữ dấu câu và gom tối đa 5 câu.
- Lý do chọn: giữ câu nguyên vẹn để đọc điều khoản và bằng chứng, giảm cắt ngang câu so với fixed-size.
- Hạn chế: chữ viết tắt có thể gây tách sai; danh sách không có dấu kết thúc câu có thể tạo chunk dài; tiêu đề không tự được gắn lại cho mọi chunk.
- Các thành viên còn lại: chờ xác nhận tên và chiến lược. Theo yêu cầu L3B, ít nhất một người phải thử heading/section; phần này chưa được xác nhận hoàn thành.

### So Sánh Giữa Các Thành Viên

| Thành viên | Chiến lược (Strategy) | Điểm truy xuất (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| | | | | |
| | | | | |
| | | | | |

**Chiến lược nào tốt nhất cho chủ đề này? Tại sao?**
> *Viết 2-3 câu — đây là phần được đánh giá cao nhất (khả năng suy nghĩ & giải thích):*

---

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

> **Đúng 5 câu hỏi**, đa dạng, có thể kiểm chứng; **ít nhất 1 câu** cần lọc metadata mới trả lời tốt. Đây là bộ câu hỏi chung cho mọi thành viên chạy.

| # | Câu hỏi (Query) | Câu trả lời chuẩn (Gold Answer) | Chunk nào chứa thông tin? |
|---|-------|-------------------------------|--------------------------|
| Q1 | Khi Shopee chấp nhận yêu cầu, Hoàn Tiền Ngay và Trả hàng & Hoàn tiền khác nhau như thế nào? | Hoàn Tiền Ngay không yêu cầu người mua trả hàng; với Trả hàng & Hoàn tiền, người mua phải chọn phương thức trả hàng và gửi hàng về kho Shopee hoặc người bán trong vòng 6 ngày từ khi nhận thông báo. | `shopee-request-processing#1` — Mục 3 |
| Q2 | Shopee có hoàn phí vận chuyển ban đầu khi người mua chỉ trả lại một số sản phẩm trong đơn không? | Không. Phí vận chuyển ban đầu chỉ được hoàn khi yêu cầu áp dụng cho toàn bộ sản phẩm và toàn bộ giá trị đã thanh toán được hoàn; nếu chỉ trả một số sản phẩm thì phí này không được hoàn. | `shopee-return-shipping-fees#3` — Mục 2.1 |
| Q3 | Người mua Shopee nên chuẩn bị những bằng chứng nào khi sản phẩm bị lỗi, hư hỏng hoặc khác mô tả? | Người mua nên quay hoặc chụp toàn bộ kiện hàng, thông tin vận chuyển và niêm phong; video mở kiện nên liên tục và thể hiện rõ quá trình mở gói, tình trạng sản phẩm cùng lỗi, hư hỏng, thiếu hàng hoặc điểm khác mô tả. | `shopee-return-evidence#1`, `#2`, `#3` — Mục 2 |
| Q4 | Nếu TikTok Shop đưa ra quyết định có lợi cho khách hàng trong tranh chấp hậu mãi, người bán phải thực hiện hành động khắc phục trong bao lâu? | Người bán phải thực hiện biện pháp khắc phục trong vòng 48 giờ, chẳng hạn hoàn tiền hoặc thay sản phẩm, và chịu phí vận chuyển nếu có. | `tiktok-aftersales-disputes#1` — Bước 3 |
| Q5 | Sau khi nhân viên chăm sóc khách hàng TikTok Shop liên hệ, người bán có bao lâu và phải làm gì để gửi trả sản phẩm cho người mua? | Người bán có 1 ngày làm việc để đóng gói an toàn, gắn nhãn vận chuyển và gửi qua đơn vị vận chuyển tiết kiệm có cung cấp mã theo dõi. | `tiktok-seller-to-customer-returns#2`, `#3`, `#4` |

Bộ câu hỏi lấy từ `BENCHMARK_QUERIES` trong `bench.py`; đây là bộ đề xuất để cả nhóm dùng chung, chưa có xác nhận của các thành viên khác.

**Metadata filter:** Q1 dùng `audience=both`, Q3 dùng `audience=buyer`, Q5 dùng `audience=seller`; Q2 và Q4 không lọc. A/B cho thấy Q1 cải thiện gold document từ hạng 2 lên hạng 1. Q3 và Q5 giữ nguyên hạng 1, nhưng filter loại các tài liệu dành cho đối tượng khác. Bộ lọc so khớp chính xác, nên tài liệu gắn `both` không tự xuất hiện khi lọc `buyer` hoặc `seller`.

**Lệnh tái lập Checkpoint 5:**

```powershell
.venv\Scripts\python.exe bench.py --chunker sentence --embedding gemini --compare-chunkers --output ket_qua_benchmark.txt
```

Kết quả Gemini (`gemini-embedding-001`) lưu ở `ket_qua_benchmark.txt`: 8 nguồn, 101 chunks và top-3 cho đủ 5 câu. Gold document đứng top-1 ở 5/5 câu, nên chỉ số cấp tài liệu là Gold@1=5/5, Gold@3=5/5 và document retrieval score=10/10.

Kiểm tra thủ công ở mức nội dung cho thấy chỉ số cấp tài liệu đang đánh giá quá cao chất lượng thực: Q1 có chunk chứa đủ đáp án ở top-2 và Q2 có ở top-1; Q3 chỉ lấy được các phần bằng chứng rải trong top-3; Q4 và Q5 lấy đúng tài liệu ở top-1 nhưng top-3 chưa chứa chunk có câu trả lời chuẩn. Benchmark chưa gọi LLM, vì vậy 10/10 không phải điểm rubric cuối cùng và không chứng minh agent trả lời đúng. Đây là dữ liệu đầu vào cho phân tích lỗi ở Checkpoint 6.

### Tổng hợp chất lượng truy xuất của nhóm

> Cách chấm (theo `docs/SCORING.md`): **2 điểm/câu** — top-3 chứa chunk liên quan + agent trả lời đúng (2), có liên quan nhưng thiếu/không ở top-1 (1), không có trong top-3 (0).

| # | Câu hỏi | Chiến lược tốt nhất cho câu này | Có chunk liên quan trong top-3? | Ghi chú |
|---|---------|-------------------------------|-------------------------------|---------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**
> *Viết 2-3 câu:*

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**
> *Liệt kê 2-3 ý:*

**Bài học rút ra khi so sánh trong nhóm:**
> *Viết 2-3 câu — cùng tài liệu nhưng chiến lược khác nhau dẫn tới khác biệt gì?*

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**
> *Viết 2-3 câu:*

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Lựa chọn tài liệu (Document Set Quality) | / 10 |
| Thiết kế chiến lược (Strategy Design) | / 15 |
| Chất lượng truy xuất (Retrieval Quality) | / 10 |
| Thuyết trình (Demo) | / 5 |
| **Tổng phần nhóm** | **/ 40** |
