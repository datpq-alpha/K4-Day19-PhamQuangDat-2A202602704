# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phạm Quang Đạt  **MSSV:** 2A202602704  **Ngày:** 05/10/2026

> Số liệu dưới đây được lấy từ `ket_qua_benchmark_kg.txt`. Không chạy lại benchmark trong quá trình hoàn thiện báo cáo.

## 1. Chi phí (10 điểm)

Provider và cấu hình: `openai:gpt-4o-mini`, embedding `openai:text-embedding-3-small`, `top_k=3`, `chunk_size=800`, 176 chunks, graph 205 nodes / 386 relationships.

### Indexing

| pipeline | calls | in_tok | out_tok | USD | seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| flat | 176 | 56072 | 0 | 0.00112 | 61.4 |
| graph | 196 | 91958 | 4902 | 0.00945 | 130.3 |

### Querying (mean per question)

| pipeline | recall | judge | in_tok | out_tok | USD | seconds |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| flat | 0.43 | 1.00 | 694 | 47 | 0.00013 | 1.69 |
| graph | 0.74 | 1.50 | 3059 | 84 | 0.00050 | 2.52 |

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00112 | 0.00945 | ×8.44 |
| Indexing giây | 61.4 | 130.3 | ×2.12 |
| Mỗi câu: USD | 0.00013 | 0.00050 | ×3.85 |
| Mỗi câu: giây | 1.69 | 2.52 | ×1.49 |
| Mỗi câu: in_tok | 694 | 3059 | ×4.40 |

**Chi phí tăng thêm đến từ đâu?** GraphRAG phải xây và truy vấn thêm knowledge graph, đồng thời đưa các facts multi-hop vào prompt. Vì vậy GraphRAG dùng nhiều input token hơn và tốn thêm thời gian/chi phí, đổi lại recall trung bình tăng từ 0.43 lên 0.74.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi chỉ cần một đoạn luật; Flat đã lấy đủ ngữ cảnh. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin hai bị cáo đã nằm trong một bài báo phù hợp. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | GraphRAG | Graph đi từ Lê Minh Thành qua vụ án và tội danh tới Điều 251. |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | GraphRAG | Graph lấy được hành vi nhưng thiếu khoản chứa mức phạt tối đa. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | GraphRAG | Graph lấy được tội danh, MDMA và khối lượng, nhưng chọn nhầm Điều 251 thay vì Điều 250. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | GraphRAG | Graph tìm được một phần các vụ liên quan MDMA nhưng chưa bao phủ đúng toàn bộ danh sách gold. |

Trên riêng ba câu `cross-kb` Q3–Q5, recall trung bình của Flat RAG là **0.20**, còn GraphRAG là **0.71**. Như vậy node cầu nối và KG-3 không bị gãy hoàn toàn.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật

- **Hiện tượng:** Q4 xác định đúng hành vi “tổ chức sử dụng trái phép chất ma túy” nhưng không trả lời được mức phạt tối đa.
- **Bằng chứng:** Kết quả GraphRAG:

  > Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Tuy nhiên, trong ngữ cảnh hiện tại không đủ thông tin để xác định mức phạt tù tối đa cho hành vi này theo Bộ luật Hình sự.

  Đáp án chuẩn yêu cầu Điều 255 và “20 năm hoặc tù chung thân”. Điều 255 khoản 4 chứa mức phạt này, nhưng không nhắc tên chất ma túy cụ thể.

  ```cypher
  MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)
        <-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause)
  WHERE a.id = 'Điều 255 BLHS'
  RETURN k.name, c.name, a.id, cl.number, cl.penalty;
  ```

- **Nguyên nhân:** `context()` chỉ giữ khoản 1 và các khoản có `MENTIONS` một `Substance` mà vụ án `INVOLVES`. Điều 255 khoản 4 không có `MENTIONS` chất, nên bị loại khỏi context.
- **Đề xuất sửa:** với câu hỏi có từ khóa “tối đa”, “cao nhất”, “chung thân” hoặc “tử hình”, giữ thêm khoản có mức phạt cao nhất của Article tương ứng. Đổi lại, prompt dài hơn và tăng input token.

### Lỗi E5: LLM lệch với graph

- **Hiện tượng:** Q5 lấy đúng nhiều dữ kiện nhưng ghi sai điều luật: câu trả lời nêu Điều 251, trong khi tội vận chuyển phải nối tới Điều 250.
- **Bằng chứng:** Kết quả GraphRAG ghi:

  > Cái Quang Huy bị truy tố về tội "vận chuyển trái phép chất ma túy" ... khoản 4 của Điều 251 ...

  Trong `benchmark_kg.json`, đáp án chuẩn là khoản 4 Điều 250.

- **Nguyên nhân:** câu hỏi nhắc `MDMA`, một Substance dùng chung cho nhiều vụ. Cơ chế seed có thể mở rộng sang nhiều Case liên quan MDMA và đưa nhiều Article vào context. LLM sau đó chọn nhầm Article 251 dù tội danh “vận chuyển” phải liên kết với Article 250.
- **Đề xuất sửa:** ưu tiên Article đi qua đúng `Case -[:CHARGED_WITH]-> Crime -[:DEFINES]-> Article`; chỉ thêm Article khác khi câu hỏi thật sự yêu cầu tổng hợp. Có thể ghi rõ trong facts quan hệ tội danh–điều luật để giảm nhầm lẫn. Đổi lại, truy vấn Cypher phức tạp hơn nhưng context chính xác hơn.

### Lỗi E4: Phép đo chưa phản ánh hoàn toàn chất lượng ngữ nghĩa

- **Hiện tượng:** Q6 của Flat RAG có `recall=0.00` nhưng `judge=1.00`; câu trả lời có nêu các vụ liên quan MDMA nhưng không dùng đúng các tên bắt buộc trong `must_include`.
- **Bằng chứng:** Flat RAG nêu “vụ việc của Đức”, “vụ việc của Thành” và “vụ việc của Đông”, trong khi `must_include` yêu cầu “Cái Quang Huy”, “Lê Minh Thành” và “Pháp y tâm thần”.
- **Nguyên nhân:** `recall` kiểm tra chuỗi bắt buộc nên nhạy với cách gọi tắt/biệt danh; `judge` đánh giá ngữ nghĩa rộng hơn. Hai phép đo đang đo hai khía cạnh khác nhau.
- **Đề xuất sửa:** giữ `recall` để kiểm tra độ bao phủ chính xác, nhưng bổ sung alias chuẩn hóa hoặc đánh giá entity-level trước khi kết luận câu trả lời sai hoàn toàn. Không nên bỏ `must_include`, vì alias quá rộng có thể làm tăng điểm giả.

## 4. Kết luận (5 điểm)

Flat RAG đủ tốt cho câu hỏi một bước như Q1 và Q2, với chi phí thấp hơn. GraphRAG phù hợp hơn cho câu hỏi xuyên hai KB và multi-hop: recall cross-KB trung bình tăng từ 0.20 lên 0.71, Q3 được cải thiện từ không trả lời được lên đầy đủ. Đổi lại, GraphRAG tốn khoảng 8.44 lần chi phí indexing và 3.85 lần chi phí mỗi câu; cần cải thiện lọc khoản luật và kiểm soát context khi một chất liên quan đến nhiều vụ.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
48 passed, 1 warning

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.1-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00168. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j:

- [Q-A — ảnh gốc](img/Q-A.png)
- [Q-B — ảnh gốc](img/Q-B.png)
- [Q-C — ảnh gốc](img/Q-C.png)
- [Q-D — ảnh gốc](img/Q-D.png)
- [Q-A — file nộp Q-A](img/kg_count.png)
- [Q-B — file nộp Q-B](img/kg_cross_kb.png)
- [Q-D — file nộp Q-D](img/kg_my_case.png)

Tên người đã chọn cho `kg_my_case.png`: **Dương Minh Tuấn** (doc_id: `news-100260920221957595`).

## Vấn đề gặp phải (không tính điểm)

Pytest có cảnh báo không ghi được thư mục cache `.pytest_cache` do quyền filesystem; không ảnh hưởng kết quả, vì toàn bộ 48 test đều passed.
