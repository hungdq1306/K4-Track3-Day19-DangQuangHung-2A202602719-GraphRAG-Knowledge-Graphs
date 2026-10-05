# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đặng Quang Hưng  **MSSV:** 2A202602719  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    112.6
graph       196     34619     6064   0.00589    199.6

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       71   0.00010     5.66
graph       1.00   2.00     6331      165   0.00070     5.85
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00589 | +$0.00589 |
| Indexing giây | 112.6s | 199.6s | ×1.77 |
| Mỗi câu: USD | $0.00010 | $0.00070 | ×7.00 |
| Mỗi câu: giây | 5.66s | 5.85s | ×1.03 |
| Mỗi câu: in_tok | 696 | 6331 | ×9.10 |

**Chi phí tăng thêm đến từ đâu?**
> 1. **Giai đoạn Indexing:** Chi phí của GraphRAG tăng thêm 20 lần gọi LLM ($0.00589 và thêm 87 giây) phục vụ việc trích xuất thực thể, thuộc tính và quan hệ có cấu trúc từ 20 bài báo tin tức bằng JSON mode, trong khi Flat RAG chỉ thuần túy tạo embedding cho các chunk văn bản mà không gọi model chat.
> 2. **Giai đoạn Querying:** Số lượng input tokens trung bình của GraphRAG cao gấp 9.1 lần (6331 vs 696 tokens) dẫn đến chi phí mỗi câu tăng 7 lần ($0.00070 vs $0.00010). Điều này bắt nguồn từ việc GraphRAG nhồi thêm các facts đồ thị multi-hop (thông tin vụ án, nhân thân, điều luật và toàn văn các khoản liên quan) vào prompt gửi LLM. Tuy nhiên, thời gian phản hồi (latency) chỉ tăng 3% (5.85s vs 5.66s) do tốc độ xử lý song song của Gemini rất nhanh đối với context dài.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa "tiền chất" nằm trọn vẹn trong một chunk của Điều 2 Luật PCMT nên Flat RAG tìm thấy trực tiếp, GraphRAG cũng truy xuất chính xác khoản 4 Điều 2. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Thông tin hai bị cáo Tuấn và Tâm bị tuyên tử hình nằm gọn trong 1 bài báo xét xử ngày 28-9, cả 2 pipeline đều trích xuất đủ và chính xác. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Bài báo về Lê Minh Thành không nêu số Điều luật và khung phạt cơ bản nên Flat RAG bỏ cuộc, trong khi GraphRAG đi qua cầu `Crime` để lấy Điều 251 khoản 1 BLHS. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Tin tức chỉ ghi Hoàng Nato bị bắt về hành vi mà không có khung phạt, GraphRAG duyệt từ bí danh sang Điều 255 BLHS để xác định mức phạt tối đa là tù chung thân. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG không biết áp dụng khoản nào cho 9,6kg MDMA, GraphRAG đối chiếu khối lượng tang vật với các khoản của Điều 250 để chỉ đúng Khoản 4 và án tử hình. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | **Graph** | Flat RAG bị giới hạn top-k=3 chỉ thấy các mẩu tin rời rạc không đủ tên vụ, GraphRAG gom cụm toàn cục qua node `Substance {name: 'MDMA'}` trả về trọn vẹn cả 3 vụ án lớn. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E4: Phép đo sai (Đánh giá chuỗi máy móc không phản ánh đúng ngữ nghĩa)

- **Hiện tượng:** Ở câu hỏi **Q6** (`aggregation`), Flat RAG đạt điểm `recall = 0.00` nhưng giám khảo LLM (`judge`) vẫn chấm **1 điểm**.
- **Bằng chứng:**
  Nguyên văn câu trả lời của Flat RAG trong `ket_qua_benchmark_kg.txt`:
  ```text
  --- Q6 [aggregation] flat recall=0.00 judge=1 5.95s
  Trả lời: Cả 3 vụ việc trong ngữ cảnh đều có liên quan đến ma túy MDMA:
  - Vụ việc [1]: Lực lượng chức năng phát hiện thùng hàng chứa các viên nén màu xanh là MDMA với tổng khối lượng gần 4,3kg.
  - Vụ việc [2]: Công an bắt quả tang Thành khi đang mang 5 viên nén màu trắng là ma túy MDMA đi bán.
  - Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA với tổng khối lượng hơn 5,3kg.
  ```
  Trong khi đó, danh sách từ khóa bắt buộc của benchmark (`data/benchmark_kg.json`) yêu cầu:
  ```json
  "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
  ```
- **Nguyên nhân:**
  Nằm ở **bước đo lường (Evaluation Metric)**: Hàm đo `recall` chỉ kiểm tra chuỗi con chính xác (`term.lower() in answer.lower()`). Flat RAG thực chất đã nhận diện đúng vụ việc của Thành (nhưng chỉ ghi "Thành" thay vì "Lê Minh Thành") và vụ việc tại sân bay Nội Bài (nhưng trong đoạn chunk trích xuất chỉ có tình tiết thùng hàng mà không có họ tên "Cái Quang Huy"). Do thiếu từ khóa nguyên văn, `recall` bị tính là 0/3 = 0.00, trong khi LLM-as-judge đọc hiểu được ngữ cảnh nên đánh giá đúng một phần (judge = 1).
- **Đề xuất sửa:**
  Cho phép danh sách alias linh hoạt trong `must_include` (ví dụ: `["Lê Minh Thành" OR "Thành"]`) hoặc áp dụng metric Semantic Coverage / LLM-based Fact Extraction để đánh giá nội dung thực chất thay vì regex string matching. Đánh đổi: tốn thêm chi phí gọi LLM evaluator hoặc thời gian chạy benchmark.

---

### Lỗi E3: Trùng thực thể (Entity Duplication trên đồ thị)

- **Hiện tượng:** Cùng một vụ việc thực tế hoặc cùng một bị can bị tách thành nhiều node `Case` riêng biệt trên đồ thị.
- **Bằng chứng:**
  Truy vấn Cypher kiểm tra các vụ án liên quan đến Cái Quang Huy:
  ```cypher
  MATCH (k:Case) WHERE toLower(k.name) CONTAINS 'cái quang huy' OR toLower(k.name) CONTAINS 'nội bài'
  RETURN k.name AS name, k.doc_id AS doc_id;
  ```
  Kết quả thực tế từ đồ thị Neo4j:
  ```text
  ╒══════════════════════════════════════════════════════════════════════════════════════════════════╤══════════════════════════╕
  │name                                                                                              │doc_id                    │
  ╞══════════════════════════════════════════════════════════════════════════════════════════════════╪══════════════════════════╡
  │"Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài"                             │"news-100260917203001265" │
  ├──────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────┤
  │"Vụ vận chuyển ma túy qua sân bay Nội Bài của Cái Quang Huy"                                      │"news-100260918080821054" │
  └──────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────────────────┘
  ```
- **Nguyên nhân:**
  Nằm ở **thiết kế Ontology** và **bước trích xuất LLM**:
  1. Trong ontology, khóa định danh của `Case` là `name`, nhưng giá trị của `name` lại do LLM tự do đặt tên theo tiêu đề từng bài báo riêng lẻ.
  2. Hai bài báo khác nhau đưa tin về cùng một vụ án vận chuyển ma túy của Cái Quang Huy qua Nội Bài nhưng đặt tiêu đề khác nhau, khiến lệnh `MERGE (k:Case {name: $name})` tạo ra hai node `Case` song song thay vì hợp nhất làm một.
- **Đề xuất sửa:**
  - Bổ sung bước Entity Resolution / Case Linking: dùng LLM hoặc thuật toán so khớp địa điểm + thời gian + danh sách người liên quan (`Person`) để xác định các bài báo viết về cùng một chuyên án, từ đó ánh xạ về một `case_id` chuẩn hóa trước khi `MERGE`.
  - Đánh đổi: Tăng độ phức tạp của pipeline trích xuất, cần thêm 1 lượt gọi LLM tổng hợp giữa các bài báo làm thời gian indexing tăng khoảng 30–40%.

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> 1. **Khi nào Flat RAG là đủ:** Đối với các câu hỏi tra cứu định nghĩa đơn lẻ (`single-hop-law` như Q1) hoặc truy xuất tình tiết cụ thể nằm trọn trong một tài liệu tin tức (`single-hop-news` như Q2). Ở các bài toán này, Flat RAG đạt kết quả hoàn hảo (recall 1.00, judge 2) với chi phí rẻ hơn 7 lần ($0.00010 vs $0.00070) và tiết kiệm 90% số token đầu vào so với GraphRAG. Nếu hệ thống chỉ phục vụ hỏi đáp sự kiện đơn giản, Flat RAG là lựa chọn tối ưu về chi phí và tài nguyên.
> 2. **Khi nào BẮT BUỘC dùng Knowledge Graph (GraphRAG):**
>    - **Truy vấn xuyên nguồn tri thức (Cross-KB):** Khi bài báo đời thực không dẫn chiếu số Điều luật (Q3, Q4), Flat RAG hoàn toàn thất bại (recall chỉ đạt 0.33, judge 1) vì không thể suy luận nhảy cóc giữa 2 không gian embedding khác nhau. GraphRAG giải quyết triệt để nhờ đi qua node cầu nối `Crime` (tội danh), đưa recall và judge lên tuyệt đối 100% (recall 1.00, judge 2).
>    - **Truy vấn đa chặng kết hợp định lượng (Multi-hop Reasoning):** Khi cần so khớp khối lượng tang vật với các khung hình phạt lũy tiến (Q5), Flat RAG chỉ đạt recall 0.40, trong khi GraphRAG kết nối chính xác từ tang vật MDMA > 9,6kg đến Khoản 4 Điều 250 với mức án tử hình.
>    - **Truy vấn tổng hợp toàn cục (Global Aggregation):** Khi cần liệt kê tất cả các vụ án liên quan đến một chất cụ thể (Q6), Flat RAG bị "mù" do giới hạn cửa sổ top-k (recall = 0.00), trong khi GraphRAG chỉ cần 1 bước duyệt đồ thị qua node trung tâm `Substance` là quét sạch toàn bộ các vụ việc liên quan trong hệ thống.
>
> **Kết luận:** Knowledge Graph là khoản đầu tư "đắt xắt ra miếng" — tốn thêm chi phí tạo index ban đầu và input token khi hỏi, nhưng là giải pháp duy nhất mang lại độ tin cậy và khả năng suy luận logic chính xác tuyệt đối cho các hệ sinh thái tri thức phức tạp, đa nguồn như Pháp luật & Tư pháp.

---

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.15s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.1-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 25 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00054. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ án vận chuyển ma túy qua sân bay Nội Bài, nối sang tội danh vận chuyển ma túy, Điều 250 BLHS, chất MDMA/Ketamine và địa điểm Hà Nội).

---

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: Không có lỗi tồn đọng.
Các vấn đề về giới hạn hạn ngạch tạm thời (rate limit 100 RPM cho embedding và model deprecation của Gemini 2.5) đã được xử lý triệt để thông qua cơ chế tự động chuyển đổi sang model `gemini-3.1-flash-lite` kết hợp thuật toán retry 5 lần với exponential backoff trong `src/llm.py`.

