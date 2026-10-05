# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Dương Minh Hiếu  **MSSV:** 2A202602488  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    103.5
graph       196     34619     5357   0.00560    177.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       76   0.00010     1.43
graph       0.94   1.83     6017      140   0.00066     3.15
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00560 | +$0.00560 (20 lần gọi LLM) |
| Indexing giây | 103.5s | 177.4s | ×1.71 |
| Mỗi câu: USD | $0.00010 | $0.00066 | ×6.60 |
| Mỗi câu: giây | 1.43s | 3.15s | ×2.20 |
| Mỗi câu: in_tok | 696 | 6017 | ×8.65 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Ở giai đoạn Indexing, GraphRAG phải gọi thêm 20 lần LLM để trích xuất có cấu trúc (JSON) các vụ án, người, chất và tội danh từ 20 bài báo tin tức (tiêu tốn ~34,6k input tokens và ~5,3k output tokens). Khi truy vấn (Querying), GraphRAG mở rộng ngữ cảnh qua multi-hop traversal (kèm các quan hệ, tóm tắt vụ việc và điều khoản định khung), khiến số lượng token đầu vào tăng gấp 8.65 lần (6.017 so với 696), dẫn đến chi phí mỗi câu tăng 6.6 lần và độ trễ tăng 2.2 lần so với Flat RAG.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều tìm thấy và trích dẫn chuẩn xác định nghĩa tiền chất trong Điều 2 Luật PCMT 2021. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai đều tìm được bài báo vụ 36kg và chỉ đích danh 2 bị cáo nhận án tử hình (Trần Thanh Tuấn, Trần Minh Tâm). |
| **Q3** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ lấy được tin tức nên thiếu điều luật và khung phạt; GraphRAG đi qua node cầu nối `Crime` để lấy trọn vẹn Điều 251 khoản 1. |
| **Q4** | cross-kb | 0.33 / 1 | 0.67 / 1 | **Graph** | Flat RAG không biết hành vi thuộc điều nào; GraphRAG định danh được hành vi tổ chức sử dụng thuộc Điều 255 BLHS qua bí danh 'Hoàng Nato'. |
| **Q5** | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG bỏ sót điều khoản và khung phạt; GraphRAG nối Cái Quang Huy $\rightarrow$ Điều 250 và đối chiếu lượng MDMA $>9,6\text{kg}$ sang Khoản 4 (tử hình). |
| **Q6** | aggregation | 0.00 / 2 | 1.00 / 2 | **Graph** | Flat RAG bị trích thiếu tên đầy đủ nên recall = 0; GraphRAG truy xuất gom cụm đầy đủ 5 vụ việc liên quan đến `Substance {name: 'MDMA'}`. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (sai khung hình phạt dù graph có đủ Điều luật)

- **Hiện tượng:** Ở câu Q4 ("Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"), GraphRAG xác định đúng hành vi bị bắt là "tổ chức sử dụng trái phép chất ma túy" (Điều 255 BLHS), nhưng phần mức phạt tù tối đa lại trả lời: *"Không đủ thông tin ... ngoại trừ việc Điều 255 khoản 1 quy định mức phạt tù từ 02 năm đến 07 năm"*. Câu trả lời bỏ sót mức phạt tù tối đa là 20 năm hoặc tù chung thân (thuộc Khoản 4 Điều 255).
- **Bằng chứng:**
  * Trích nguyên văn câu trả lời Q4 trong `ket_qua_benchmark_kg.txt`:
    > "- Hành vi bị bắt: Giang hồ 'Hoàng Nato' (Dương Minh Tuấn) bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy.
    > - Mức phạt tù tối đa: Không đủ thông tin (ngữ cảnh không nêu rõ Điều luật cụ thể và mức phạt tù tối đa áp dụng riêng cho hành vi tổ chức sử dụng trái phép chất ma túy của Hoàng Nato, ngoại trừ việc Điều 255 khoản 1 quy định mức phạt tù từ 02 năm đến 07 năm cho tội danh này)."
  * Cypher kiểm tra cấu trúc Điều 255 trong graph:
    ```cypher
    MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
    RETURN cl.number, cl.penalty, collect(s.name) AS substances
    ORDER BY cl.number;
    ```
    Kết quả:
    ```text
    cl.number | cl.penalty                                      | substances
    1         | "phạt tù từ 02 năm đến 07 năm"                  | []
    2         | "phạt tù từ 07 năm đến 15 năm"                  | ["Heroine", "Cocaine", "Methamphetamine", ...]
    3         | "phạt tù từ 15 năm đến 20 năm"                  | ["Heroine", "Cocaine", ...]
    4         | "phạt tù 20 năm hoặc tù chung thân"             | []
    ```
- **Nguyên nhân:** Nằm ở câu lệnh Cypher lọc khoản luật trong hàm `Neo4jGraph.context`:
  ```cypher
  WHERE cl.number = 1 OR EXISTS { MATCH (k)-[:INVOLVES]->(sub:Substance)<-[:MENTIONS]-(cl) }
  ```
  Khoản 4 của Điều 255 BLHS quy định về các tình tiết định khung đặc biệt nghiêm trọng ("làm chết 02 người trở lên", "gây thương tích...", "đối với 03 người trở lên...") chứ không liệt kê theo tên chất ma túy. Do đó, điều kiện `EXISTS` không thỏa mãn và `cl.number = 4` bị loại bỏ khỏi danh sách facts cung cấp cho LLM.
- **Đề xuất sửa:** Trong hàm `context()`, khi truy xuất các khoản của một Điều luật liên quan đến câu hỏi về "hình phạt tối đa", cần bổ sung truy vấn lấy khoản có số thứ tự lớn nhất (`max(cl.number)`) hoặc luôn lấy kèm khoản có khung hình phạt cao nhất của Điều luật đó. Đánh đổi: Tăng thêm 1-2 fact cho mỗi điều luật, tốn thêm khoảng 50-80 tokens vào prompt nhưng đảm bảo không bao giờ trượt khung phạt cao nhất.

---

### Lỗi E4: Phép đo sai (recall và judge mâu thuẫn nhau)

- **Hiện tượng:** Ở câu Q6 (câu hỏi gom nhóm aggregation: *"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*), phương pháp Flat RAG nhận điểm đánh giá `recall = 0.00` nhưng điểm của LLM judge lại đạt mức tối đa `judge = 2`.
- **Bằng chứng:**
  * Trích xuất từ `ket_qua_benchmark_kg.txt`:
    > `--- Q6 [aggregation] flat recall=0.00 judge=2 1.69s`
    > "Dựa trên ngữ cảnh, tất cả 3 vụ việc đều có liên quan đến ma túy MDMA:
    > 1. Vụ việc thứ nhất ([1]): Lực lượng chức năng phát hiện các viên nén màu xanh bên trong thùng hàng là MDMA (khối lượng gần 4,3kg) liên quan đến Đạt và Huy.
    > 2. Vụ việc thứ hai ([2]): Công an bắt quả tang Thành mang 5 viên ma túy đến điểm hẹn để bán, kết luận giám định xác định đây là ma túy MDMA.
    > 3. Vụ việc thứ ba ([3]): Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng gửi bởi Huy là MDMA (khối lượng hơn 5,3kg)."
  * Đối chiếu cấu hình câu hỏi Q6 trong `data/benchmark_kg.json`:
    ```json
    {
      "id": "Q6",
      "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
    }
    ```
- **Nguyên nhân:**
  * Phép đo `recall` tính toán dựa trên exact substring match cơ học của mảng `must_include`. Do Flat RAG trả lời theo văn phong rút gọn: viết tắt là "Huy" thay vì "Cái Quang Huy", viết "Thành" thay vì "Lê Minh Thành", và trích 2 chunk cùng thuộc vụ Cái Quang Huy mà không trích được vụ "Pháp y tâm thần", nên tỷ lệ xuất hiện của 3 cụm từ khóa chuẩn là 0/3 $\rightarrow$ `recall = 0.00`.
  * Ngược lại, LLM-as-judge đánh giá theo ngữ nghĩa linh hoạt: nhận thấy câu trả lời đã chỉ ra được đúng 3 vụ án có liên quan đến MDMA dựa trên ngữ cảnh được cung cấp, nên giám khảo chấm điểm 2 (đúng/hợp lý).
- **Đề xuất sửa:**
  * Sửa hàm tính recall trong `bench_kg.py`: Chuẩn hóa danh từ riêng và chấp nhận tên gọi ngắn/bí danh (alias matching: "Huy" $\approx$ "Cái Quang Huy", "Thành" $\approx$ "Lê Minh Thành").
  * Chuẩn hóa prompt hệ thống: Yêu cầu mô hình trả lời bắt buộc ghi đầy đủ họ tên nhân vật và tên đơn vị/địa điểm thay vì chỉ ghi tên gọi vắn tắt.

---

### Lỗi E1: Cầu nối gãy (vụ án không nối được sang luật)

- **Hiện tượng:** Một số vụ án được trích xuất từ tin tức nhưng node `Case` hoàn toàn không có cạnh `CHARGED_WITH` để kết nối sang `Crime`, trở thành node mồ côi và không thể đi sang KB Luật.
- **Bằng chứng:**
  * Cypher kiểm tra các Case không có quan hệ `CHARGED_WITH`:
    ```cypher
    MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() 
    RETURN k.name, k.doc_id;
    ```
  * Kết quả thực tế trên graph:
    ```text
    k.name                                                                | k.doc_id
    "Vụ vận chuyển vũ khí và hơn 800kg chất nghi ma túy tại Preah Sihanouk" | news-100260924145818945
    "Triệt phá chuyên án A3-626P"                                         | news-100261002184934505
    ```
- **Nguyên nhân:**
  * Bài báo `news-100260924145818945` đưa tin về vụ án xảy ra tại Campuchia (tỉnh Preah Sihanouk), cơ quan công an mới thu giữ "chất nghi ma túy" và chưa có quyết định khởi tố tội danh chính thức theo Bộ luật Hình sự Việt Nam.
  * Trong prompt trích xuất, danh sách tội danh chỉ gồm 13 tội danh theo Chương XX BLHS Việt Nam. Do bài báo không có từ khóa tội danh cụ thể, hàm `link_entity` trả về rỗng, dẫn đến không thể tạo liên kết `CHARGED_WITH`.
- **Đề xuất sửa:**
  * Thêm node tội danh tổng quát hoặc trạng thái điều tra (ví dụ: `Crime {name: "đang điều tra xác minh"}`) để phân biệt các vụ án chưa khởi tố.
  * Bổ sung cơ chế fallback nối qua node `Substance` hoặc `Location` thay vì chỉ phụ thuộc vào `Crime`.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Nên dùng Flat RAG khi:** Hệ thống phục vụ các câu hỏi tra cứu thông tin đơn bước (single-hop), định nghĩa trực tiếp hoặc tìm kiếm sự kiện cục bộ trong một văn bản (như Q1 và Q2, cả hai pipeline đều đạt điểm tuyệt đối recall = 1.0 và judge = 2). Flat RAG vượt trội hoàn toàn về hiệu năng kinh tế và tốc độ: chi phí rẻ hơn 6.6 lần ($0.00010 vs $0.00066), độ trễ thấp hơn 2.2 lần (1.43s vs 3.15s) và không tốn chi phí xây dựng đồ thị ban đầu (Indexing 0 USD vs 0.0056 USD).
> - **Bắt buộc dùng GraphRAG khi:** Bài toán đòi hỏi suy luận xuyên văn bản (cross-KB), định khung pháp lý kết hợp tang vật nhiều bước nhảy (multi-hop như Q3, Q5), hoặc tổng hợp gom cụm trên toàn bộ kho tri thức (aggregation như Q6). Ở các câu hỏi này, Flat RAG hoàn toàn thất bại (recall chỉ đạt 0.00 – 0.40) do không thể gom đủ các đoạn văn bản rời rạc; trong khi GraphRAG đạt recall gần như tuyệt đối (0.94 trung bình toàn benchmark, judge 1.83) nhờ khả năng định tuyến tri thức qua các node cầu nối `Crime` và `Substance`.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.16s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (Vụ vận chuyển ma túy qua sân bay Nội Bài $\rightarrow$ Tội vận chuyển trái phép chất ma túy $\rightarrow$ Điều 250 BLHS).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> - **Lỗi 1 (Đổi model Gemini):** Model mặc định ban đầu `gemini-2.5-flash-lite` bị Google API trả về lỗi 404 (ngừng phục vụ cho tài khoản mới, yêu cầu dùng `gemini-3.5-flash-lite`). Đã giải quyết bằng cách cập nhật cấu hình model trong `src/llm.py` và `.env`.
> - **Lỗi 2 (Rate limit 429 trên Gemini Free Tier):** Khi chạy lệnh `bench_kg.py --judge`, việc nạp liên tiếp 20 bài báo đã chạm ngưỡng giới hạn 15 requests/phút (RPM) của gói miễn phí. Đã khắc phục triệt để bằng cách cài đặt cơ chế thử lại tự động (retry loop với exponential backoff 4s, 8s, 12s...) trong `MeteredLLM.chat` và `MeteredLLM.embed`, giúp benchmark chạy hoàn tất 100% mà không bị gián đoạn.

