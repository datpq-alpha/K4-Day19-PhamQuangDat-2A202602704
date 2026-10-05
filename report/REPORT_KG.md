# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phạm Quang Đạt  **MSSV:** 2A202602704  **Ngày:** 05/10/2026

> Số liệu dưới đây được lấy từ `ket_qua_benchmark_kg.txt`.

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
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Q4 GraphRAG có `recall=0.33`, `judge=1` và trả lời:

  > Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Tuy nhiên, trong ngữ cảnh hiện tại không đủ thông tin để xác định mức phạt tù tối đa cho hành vi này theo Bộ luật Hình sự.

  Trong `benchmark_kg.json`, gold và `must_include` yêu cầu Điều 255 và “chung thân”. Điều 255 khoản 4 quy định mức cao nhất là 20 năm hoặc tù chung thân, nhưng khoản này không nhắc tên một chất ma túy cụ thể.

  ```cypher
  MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)
        <-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause)
  WHERE a.id = 'Điều 255 BLHS'
  RETURN k.name, c.name, a.id, cl.number, cl.penalty;
  ```

- **Nguyên nhân:** `context()` chỉ giữ khoản 1 và các khoản có `MENTIONS` một `Substance` mà vụ án `INVOLVES`. Điều 255 khoản 4 không có `MENTIONS` chất, nên bị loại khỏi context.
- **Đề xuất sửa:** với câu hỏi có từ khóa “tối đa”, “cao nhất”, “chung thân” hoặc “tử hình”, giữ thêm khoản có mức phạt cao nhất của Article tương ứng. Đổi lại, prompt dài hơn và tăng input token.

### Lỗi E4: Phép đo chưa phản ánh đúng mức độ nghiêm trọng của lỗi

- **Hiện tượng:** Q5 GraphRAG đạt recall khá cao (`0.80`) dù câu trả lời sai một dữ kiện pháp lý cốt lõi; đồng thời Q6 Flat RAG có recall `0.00` nhưng judge vẫn cho `1`.
- **Bằng chứng:** Q5 GraphRAG trả lời:

  > Cái Quang Huy bị truy tố về tội "vận chuyển trái phép chất ma túy" ... khoản 4 của Điều 251 ...

  Trong `benchmark_kg.json`, đáp án đúng là khoản 4 **Điều 250**, không phải Điều 251. Tuy nhiên, bốn chuỗi còn lại trong `must_include` vẫn xuất hiện (`vận chuyển`, `MDMA`, `khoản 4`, `tử hình`), nên recall đạt 4/5 = 0.80. Judge cho `1`, phản ánh đây chỉ là câu trả lời đúng một phần.

  Ở Q6 Flat RAG, recall bằng 0 vì không có đúng các chuỗi “Cái Quang Huy”, “Lê Minh Thành”, “Pháp y tâm thần”, nhưng judge vẫn cho `1` vì câu trả lời có nhắc đến ba nhóm sự việc bằng các cách gọi “Đức”, “Thành”, “Đông”.

- **Nguyên nhân:** recall đang coi mỗi chuỗi trong `must_include` có trọng số ngang nhau và chỉ kiểm tra sự xuất hiện của chuỗi. Vì vậy một lỗi pháp lý quan trọng như sai số Điều chỉ làm mất 1/5 điểm, trong khi cách gọi tắt tên người lại có thể làm mất toàn bộ recall. Judge đánh giá ngữ nghĩa tổng thể nên cho kết quả khác.
- **Đề xuất sửa:** gán trọng số cao hơn cho dữ kiện pháp lý quyết định như số Điều/khoản, đồng thời chuẩn hóa alias thực thể trước khi tính recall. Nên giữ judge để bổ sung đánh giá ngữ nghĩa; đổi lại phép đo sẽ phức tạp hơn và cần cấu hình trọng số cho từng câu.

### Lỗi E5: Câu trả lời aggregation lệch khỏi tập thực thể cần tổng hợp

- **Hiện tượng:** Q6 GraphRAG lấy được ba nhóm sự việc liên quan MDMA nhưng không nêu đúng hai thực thể mà benchmark yêu cầu, đồng thời thêm các Điều luật không được hỏi.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Q6 GraphRAG có `recall=0.33`, `judge=1` và trả lời ba mục: (1) Cái Quang Huy, (2) “Ba thanh niên góp tiền mua ma túy”, (3) Lê Văn Đông tại Sầm Sơn. Câu trả lời còn thêm Điều 249, 250, 251 và 252.

  Trong `benchmark_kg.json`, ba thực thể bắt buộc là:

  ```text
  Cái Quang Huy; Lê Minh Thành; Pháp y tâm thần
  ```

  Chỉ “Cái Quang Huy” xuất hiện nguyên văn, nên recall là 1/3 = 0.33. “Ba thanh niên” thuộc vụ có Lê Minh Thành nhưng LLM bỏ tên cầu nối cần báo cáo; mục Lê Văn Đông/Sầm Sơn chỉ mô tả một phần của nhóm vụ Viện Pháp y tâm thần và không gọi đúng thực thể chuẩn. Các Điều luật 249–252 là thông tin thừa so với câu hỏi “những vụ việc nào”.

  Truy vấn Cypher đối chứng trực tiếp cho Q6 nên là:

  ```cypher
  MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})
  OPTIONAL MATCH (p:Person)-[:INVOLVED_IN]->(k)
  RETURN DISTINCT k.name, k.doc_id,
         collect(DISTINCT p.name) AS people,
         k.summary
  ORDER BY k.name;
  ```

- **Nguyên nhân:** bước trả lời bằng LLM đã tóm tắt các Case theo mô tả chung thay vì giữ tên thực thể `Person`/nguồn vụ án trong facts; context còn đưa các khoản luật liên quan MDMA vào prompt dù Q6 chỉ yêu cầu tổng hợp vụ việc. Vì vậy câu trả lời vừa thiếu tên chuẩn vừa có dữ kiện luật thừa. File kết quả chỉ ghi một lần chạy nên chưa đủ bằng chứng để kết luận lỗi này có lặp lại ổn định giữa nhiều lần benchmark.
- **Đề xuất sửa:** trong `Neo4jGraph.context`, nhận diện câu hỏi aggregation và tạo facts theo từng Case với `case name`, `doc_id`, danh sách Person và summary; không mở rộng sang Article/Clause nếu câu hỏi không hỏi luật. Trong prompt trả lời, yêu cầu giữ nguyên tên thực thể từ facts và liệt kê đủ các Case trước khi diễn giải. Đánh đổi là cần thêm nhánh Cypher theo loại câu hỏi và prompt dài hơn khi có nhiều Case.

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

- [Q-A](img/kg_count.png)
- [Q-B](img/kg_cross_kb.png)
- [Q-D](img/kg_my_case.png)

Tên người đã chọn cho `kg_my_case.png`: **Dương Minh Tuấn** (doc_id: `news-100260920221957595`).

## Vấn đề gặp phải (không tính điểm)

Pytest có cảnh báo không ghi được thư mục cache `.pytest_cache` do quyền filesystem; không ảnh hưởng kết quả, vì toàn bộ 48 test đều passed.
