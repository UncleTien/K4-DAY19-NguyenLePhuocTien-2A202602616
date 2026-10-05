# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Lê Phước Tiến  
**MSSV:** 2A202602616  
**Ngày:** 05/10/2026

## 1. Chi phí (10 điểm)

Kết quả từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat          0         0        0   0.00000    105.1
graph        20     34619     5572   0.00000    194.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       78   0.00000     1.52
graph       0.94   1.83     5694      179   0.00000    10.86
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | $0.00000 | $0.00000 | N/A (0/0) |
| Indexing giây | 105.1 | 194.8 | ~1.85× |
| Mỗi câu: USD | $0.00000 | $0.00000 | N/A (0/0) |
| Mỗi câu: giây | 1.52 | 10.86 | ~7.14× |
| Mỗi câu: in_tok | 696 | 5,694 | ~8.18× |

**Chi phí tăng thêm đến từ đâu?**

GraphRAG cần thêm bước xây dựng Knowledge Graph từ 20 bài báo bằng LLM, tạo 200 node và 384 relationship, nên thời gian indexing tăng từ 105.1 giây lên 194.8 giây. Khi truy vấn, GraphRAG đưa cả vector chunks và các graph facts multi-hop vào prompt nên input token tăng khoảng 8.18 lần và độ trễ trung bình tăng khoảng 7.14 lần.

USD đều bằng `$0.00000` trong benchmark vì provider/free tier được sử dụng, do đó không thể tính tỷ lệ USD có ý nghĩa từ số liệu này.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Một chunk luật đã đủ để trả lời định nghĩa tiền chất, nên graph không tạo lợi thế rõ ràng. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Câu hỏi chỉ cần thông tin trong một bài báo nên Flat RAG đã lấy đủ hai bị cáo tử hình. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Graph nối Lê Minh Thành → Case → Crime → Điều 251 → khoản 1, trong khi Flat chỉ tìm được án 36 tháng và tội danh. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph về recall, nhưng vẫn lỗi | Graph tìm được Điều 255 nhưng chỉ lấy khoản 1 nên trả tối đa 7 năm thay vì khung cao nhất 20 năm hoặc chung thân. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Graph kết hợp Case, Crime, Article, Clause và Substance để suy ra khoản 4 Điều 250 và khung 20 năm/chung thân/tử hình. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Flat top-k chỉ thấy một số chunks, còn Graph có thể tổng hợp nhiều Case cùng liên kết tới MDMA. |

**Quy luật quan sát được:** Với câu single-hop Q1–Q2, Flat RAG đã đủ và nhanh hơn. Lợi thế của GraphRAG xuất hiện rõ ở câu cross-KB, multi-hop và aggregation Q3–Q6, nơi cần nối thông tin giữa nhiều tài liệu hoặc giữa KB tin tức và KB luật.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật khi hỏi mức phạt tối đa

- **Hiện tượng:** Q4 yêu cầu mức phạt tù tối đa của hành vi tổ chức sử dụng trái phép chất ma túy. GraphRAG tìm đúng tội danh và Điều 255 nhưng chỉ lấy khoản 1, vì vậy trả lời mức tối đa 07 năm. Gold answer yêu cầu khung cao nhất là 20 năm hoặc tù chung thân.

- **Bằng chứng:** Kết quả benchmark:

```text
--- Q4 [cross-kb] graph recall=0.67 judge=1 1.98s

Dựa trên ngữ cảnh và dữ kiện cung cấp:

* Hành vi bị bắt: Giang hồ "Hoàng Nato" (Dương Minh Tuấn)
  bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy.
* Mức phạt tù tối đa: Theo Điều 255 Bộ luật Hình sự
  khoản 1, mức hình phạt tù đối với tội danh này là
  từ 02 năm đến 07 năm...
```

Trong `data/benchmark_kg.json`, đáp án chuẩn là:

```text
Dương Minh Tuấn (Hoàng Nato) bị bắt về hành vi tổ chức
sử dụng trái phép chất ma túy (Điều 255 BLHS);
khung cao nhất là tù 20 năm hoặc tù chung thân.
```

Kết quả:

```text
Flat : recall=0.33, judge=1
Graph: recall=0.67, judge=1
```

- **Nguyên nhân:** Lỗi nằm ở bước graph context retrieval trong `Neo4jGraph.context()`. Cypher hiện chỉ giữ `cl.number = 1` hoặc các Clause có `MENTIONS` tới `Substance` mà Case `INVOLVES`. Với câu hỏi về mức hình phạt cao nhất, các khoản tăng nặng của Điều 255 không nhất thiết được chọn bởi hai điều kiện này. Vì vậy graph có Điều 255 nhưng context gửi vào LLM không chứa đủ các khoản cần thiết.

- **Đề xuất sửa:** Phân tích intent của câu hỏi. Nếu câu hỏi chứa các cụm như “tối đa”, “cao nhất”, “khung cao nhất”, sau khi xác định `Article` cần lấy toàn bộ `Clause` của Article đó hoặc ít nhất clause có penalty cao nhất.

Ví dụ hướng Cypher:

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})
      -[:HAS_CLAUSE]->(cl:Clause)
RETURN a.id, cl.number, cl.penalty, cl.text
ORDER BY cl.number;
```

Đánh đổi là prompt GraphRAG dài hơn, tăng token và độ trễ. Tuy nhiên benchmark hiện tại đã chứng minh việc chỉ lấy khoản 1 làm giảm chất lượng Q4 nên việc mở rộng có điều kiện là hợp lý.

### Lỗi E5: Câu aggregation của LLM có thể mở rộng hơn tập đáp án chuẩn

- **Hiện tượng:** Q6 hỏi những vụ việc trong tin tức liên quan đến MDMA. GraphRAG đạt recall `1.00` và judge `2`, nhưng câu trả lời liệt kê 4 nhóm vụ việc trong khi gold answer tập trung vào 3 nhóm chính.

- **Bằng chứng:** Gold answer trong benchmark:

```text
Vụ Cái Quang Huy vận chuyển hơn 9,6kg MDMA từ Đức về qua Nội Bài;
vụ Lê Minh Thành và 3 thanh niên mua bán ma túy (có MDMA);
vụ Viện Pháp y tâm thần Trung ương
(thu giữ MDMA, ketamine, methamphetamine).
```

GraphRAG trả lời:

```text
1. Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài
2. Vụ mua bán ma túy tổ chức tiệc sinh nhật tại Hà Nội
3. Vụ tổ chức sử dụng trái phép chất ma túy tại Sầm Sơn
   và Viện Pháp y tâm thần Trung ương
4. Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương
```

Kết quả benchmark:

```text
--- Q6 [aggregation] flat recall=0.00 judge=1
--- Q6 [aggregation] graph recall=1.00 judge=2
```

Flat RAG chỉ tổng hợp được các chunks gần nhất và không chứa đủ các thực thể bắt buộc. GraphRAG lấy được toàn bộ `must_include`, nhưng việc diễn đạt/tách vụ của LLM có thể rộng hơn cách nhóm vụ trong gold answer.

- **Nguyên nhân:** Graph traversal có thể đưa nhiều `Case` cùng liên kết với node `Substance` MDMA vào context. Sau đó LLM tự tổng hợp các graph facts thành câu trả lời. Nếu hai `Case` thực chất thuộc cùng một chuỗi sự kiện hoặc cùng một vụ lớn nhưng được LLM extraction đặt tên khác nhau, bước generation có thể trình bày chúng thành các mục riêng.

Ngoài ra `Case` hiện được `MERGE` theo `name`, nên entity resolution giữa các tên vụ khác nhau chưa đủ mạnh để đảm bảo các mô tả của cùng một sự kiện luôn được hợp nhất.

- **Đề xuất sửa:** Trước khi generation, chạy aggregation trực tiếp trên graph và trả về danh sách Case duy nhất có quan hệ với `Substance {name:'MDMA'}`. Có thể bổ sung bước canonicalization/deduplication Case dựa trên `doc_id`, ngày, người liên quan và source article trước khi gửi facts cho LLM.

Ví dụ truy vấn kiểm tra:

```cypher
MATCH (k:Case)-[r:INVOLVES]->(s:Substance)
WHERE toLower(s.name) = 'mdma'
RETURN DISTINCT k.name, k.doc_id, r.amount
ORDER BY k.name;
```

Cách này làm câu aggregation ổn định và kiểm chứng được hơn, nhưng cần thêm logic entity resolution và có thể tăng độ phức tạp của pipeline.

## 4. Kết luận (5 điểm)

Kết quả cho thấy Flat RAG phù hợp với câu hỏi single-hop khi thông tin cần thiết nằm trong một tài liệu hoặc một số chunk gần nhau. Q1 và Q2 đều đạt recall `1.00`, judge `2` ở cả hai pipeline, trong khi Flat RAG chỉ mất trung bình 1.52 giây/câu và 696 input tokens.

GraphRAG phù hợp hơn khi câu hỏi cần nối nhiều nguồn, multi-hop hoặc aggregation. Recall trung bình tăng từ `0.51` của Flat lên `0.94`, judge tăng từ `1.33` lên `1.83`. Ở Q3, GraphRAG tăng recall từ `0.33` lên `1.00`; Q5 từ `0.40` lên `1.00`; Q6 từ `0.00` lên `1.00`.

Đánh đổi là chi phí xử lý. GraphRAG mất trung bình `10.86s/câu`, khoảng `7.14×` Flat RAG, và dùng `5,694` input tokens/câu, khoảng `8.18×` Flat. Indexing cũng tăng từ `105.1s` lên `194.8s`.

Vì vậy, với dữ liệu đơn giản và câu hỏi single-hop, Flat RAG là lựa chọn hiệu quả hơn. Khi dữ liệu có nhiều thực thể và quan hệ, đặc biệt cần nối KB tin tức với KB luật hoặc tổng hợp nhiều vụ việc, Knowledge Graph mang lại lợi ích rõ rệt. Tuy nhiên KG retrieval vẫn cần thiết kế theo intent câu hỏi; lỗi Q4 cho thấy có graph đúng chưa đủ nếu bước chọn context bỏ sót các khoản luật quan trọng.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................ [100%]
48 passed in 0.04s
```

```text
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 1 cạnh
[OK] KG-3 context: 43 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000.
```

Ảnh Neo4j:

- `report/img/kg_count.png`
- `report/img/kg_cross_kb.png`
- `report/img/kg_my_case.png`

**Người đã chọn cho `kg_my_case.png`:** Cái Quang Huy