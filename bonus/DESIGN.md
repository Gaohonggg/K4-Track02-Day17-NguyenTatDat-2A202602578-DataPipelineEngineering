# B2 — Thiết kế pipeline dữ liệu cho chatbot CSKH tiếng Việt

## Bài toán và ràng buộc

Thiết kế một chatbot hỗ trợ sản phẩm SaaS tại Việt Nam: tra cứu hướng dẫn, phân loại ticket và chuyển trường hợp khó cho nhân viên. Dữ liệu gồm CDC ticket từ PostgreSQL, click/feedback từ ứng dụng, transcript hội thoại và tài liệu trợ giúp có phiên bản. Khó khăn là event đến muộn khi mất mạng, change giao lại, nội dung có tên/email/số điện thoại và phản hồi sai từ chính chatbot.

Các con số sau là **giả định thiết kế**, chưa phải số đo production: 10.000 ticket và 100.000 event/ngày, 500 tài liệu trợ giúp, đội hai kỹ sư. Mục tiêu ban đầu: tài liệu được duyệt xuất hiện trong retrieval trong 15 phút, feature routing cập nhật hằng ngày, dataset train xuất bản hằng tuần. Yêu cầu thu hồi phải chặn sử dụng nội dung trước khi hoàn tất xoá vật lý. Mục tiêu độ trễ trả lời p95 dưới 3 giây cần kiểm chứng bằng load test; không suy ra từ tốc độ pipeline lab.

Lab đã kiểm chứng khóa, LSN, lookback và checksum trên seed. Thiết kế này mở rộng các nguyên tắc đó; chưa triển khai CDC thật, vector search hay NER. Vector hash 16 chiều trong lab chỉ minh họa cache, không được xem là embedding ngữ nghĩa cho sản phẩm.

## 1. Nguồn và schema: lưu dữ liệu thô hay chỉ giữ bản sạch?

**Quyết định:** giữ Bronze có payload, source, ingest time, event time, khóa và vị trí CDC; Silver dùng contract có schema version, dedup và chuẩn hoá Unicode NFC. Batch có manifest số hàng/checksum để truy vết. Change ticket nhận diện bằng khóa và LSN; event dùng event_id. Field mới được xem xét có chủ đích; thiếu khóa hoặc kiểu dữ liệu sai đi quarantine thay vì âm thầm bỏ dòng.

**Đánh đổi:** chỉ giữ bản sạch tiết kiệm dung lượng và giảm bề mặt PII, nhưng khó replay khi parser sai. Raw giúp điều tra và backfill, đổi lại cần mã hoá, quyền truy cập hẹp và retention; chọn retention raw 7 ngày là giả định vận hành cần được thống nhất. Bronze bất biến trong thời gian lưu thông thường, có ngoại lệ purge theo yêu cầu thu hồi. Văn bản public chỉ lấy từ tài liệu được duyệt; transcript khách hàng không tự động trở thành tri thức dùng chung.

## 2. Batch hay streaming: độ tươi nào đủ cho nghiệp vụ?

**Quyết định:** microbatch 5 phút cho CDC/tài liệu để đáp ứng mục tiêu freshness 15 phút; feature theo event date chạy hằng ngày và dataset train hằng tuần. Luồng thu hồi có ưu tiên riêng: cập nhật denylist tại serving ngay sau khi nhận yêu cầu. Một code path xử lý batch và backfill; checkpoint chỉ tiến sau khi output/manifest được xác nhận.

**Đánh đổi:** streaming mọi dữ liệu giảm độ trễ nhưng tăng quản lý state, watermark và trực vận hành, trong khi training không cần độ tươi từng giây. Batch hằng đêm đơn giản hơn nhưng có thể trả hướng dẫn đã lỗi thời trong cả ngày. Microbatch cân bằng ở quy mô giả định, song không đủ cho routing dưới một giây. Khi có yêu cầu đó, cần đo lợi ích thực rồi bổ sung feature online. Lookback khởi điểm 3 ngày lấy từ seed lab, không mặc định áp dụng production; đo lại P99 và theo dõi lượng event ngoài cửa sổ để backfill.

## 3. Chất lượng và PII: xử lý sai dữ liệu ở đâu?

**Quyết định:** validate tại Bronze→Silver bằng contract, kiểm tra event_id, user_id, timestamp, enum và sự tương thích type/rating. Regex xử lý email/phone, NER tiếng Việt phát hiện tên/địa chỉ; văn bản chưa qua chốt PII không được xuất sang Gold. Quarantine lưu reason, source reference và schema version, hạn chế quyền truy cập vì payload có thể chứa thông tin cá nhân. Kiểm tra PII thêm ở Gold và trước khi đưa nội dung vào model.

**Đánh đổi:** chỉ dùng regex rẻ và dễ giải thích nhưng bỏ lọt tên; NER tăng recall nhưng có thể che nhầm thuật ngữ sản phẩm hoặc tên tổ chức. Đo precision/recall từng loại trên tập tiếng Việt có nhãn, gồm viết không dấu, viết tắt và lỗi gõ; ưu tiên kiểm tra false negative của nội dung đưa ra ngoài. Mốc cảnh báo quarantine trên 1% batch hoặc tăng hơn ba lần mức nền là giả định cần hiệu chỉnh. Người trực pipeline nhận cảnh báo; chủ nguồn xác nhận lỗi schema. Masking giảm rủi ro nhưng không bảo đảm ẩn danh; quyền truy cập và thu hồi vẫn cần thiết.

## 4. RAG hay knowledge graph: câu hỏi thực sự cần gì?

**Quyết định:** bắt đầu bằng hybrid retrieval từ keyword và embedding ngữ nghĩa cho kho tài liệu được duyệt, lọc tenant/quyền, product version và trạng thái thu hồi trước khi đưa context vào model. Chunk theo heading, giữ doc_id, source_version và nguồn trích dẫn. Cache embedding theo hash của nội dung đã xử lý PII và model version; nội dung hoặc model thay đổi thì tạo version mới. Quy tắc quyền truy cập không dựa riêng vào mức tương đồng.

**Đánh đổi:** vector hỗ trợ diễn đạt đa dạng, keyword hữu ích cho mã lỗi và từ khóa chính xác, nhưng hợp nhất hai nguồn và rerank làm tăng latency. Chọn candidate set nhỏ rồi đo recall@k và tỷ lệ câu trả lời có căn cứ trên bộ eval. KG có lợi cho quan hệ nhiều bước, nhưng cần entity resolution và quản lý provenance; FAQ hiện chủ yếu lookup, chưa đủ lý do trả chi phí đó. Chỉ thêm graph khi eval xuất hiện nhóm câu multi-hop mà retrieval hiện tại thất bại có hệ thống. Ticket thô không được index public; nội dung rút ra từ ticket phải qua biên tập và phê duyệt.

## 5. Train/serve parity và flywheel: feedback nào đáng tin?

**Quyết định:** trace chứa input đã xử lý PII, document/model/prompt version, timestamp quyết định và feedback liên kết đúng turn. Feature train dùng trạng thái được biết tại thời điểm quyết định, qua as-of join; lưu cả event time và thời điểm dữ liệu khả dụng để tránh đưa event đến muộn vào quá khứ. Cùng định nghĩa feature dùng cho train và serve; so mẫu online/offline trước phát hành.

**Đánh đổi:** join latest dễ triển khai nhưng rò rỉ tương lai; as-of tốn lưu history và xử lý correction. Feedback thumbs-up rẻ nhưng có thể thưởng câu trả lời sai; không dùng nó làm nhãn chuẩn duy nhất. Nhân viên xác nhận một mẫu trước khi tạo dữ liệu SFT/DPO. Giữ eval theo nhóm hội thoại và thời gian, kiểm tra trùng/paraphrase với train, loại dữ liệu thu hồi và nội dung độc hại. Dataset có manifest version, cutoff và document lineage; version mới không sửa lén kết quả eval cũ. Thêm kiểm duyệt làm chậm flywheel, đổi lại giảm nguy cơ tự khuếch đại lỗi của chatbot.

## 6. Replay, thu hồi và scale: lỗi nào gây hại lâu dài?

**Quyết định:** Silver MERGE theo khóa với guard thứ tự change, Gold aggregate overwrite partition; metadata tombstone chống replay cũ hồi sinh dữ liệu. Với nhiều nguồn CDC, thứ tự được quản lý trong từng nguồn thay vì so LSN không cùng miền. Update/delete index được ghi vào transactional outbox cùng thay đổi Silver; worker retry theo operation_id. Manifest xác định source version hiện hành, serving kiểm tra trạng thái thu hồi để từ chối chunk cũ trong lúc xoá index đang retry. Worker upsert cũng kiểm tra version/tombstone để backfill không tái tạo nội dung đã bị thu hồi.

**Đánh đổi:** database và vector index không có một transaction chung; outbox tạo eventual consistency và thêm state cần quan sát. Denylist thu hẹp khoảng thời gian phục vụ nội dung cũ, nhưng cần xử lý trường hợp không đọc được trạng thái quyền/thu hồi bằng cách từ chối tài liệu đó. Purge phải đi qua raw, Silver transcript, cache, index, snapshot và bản sao; sổ lineage chỉ giữ metadata tối thiểu để xác nhận hoàn tất. Backfill ghi vào staging, so contract/checksum rồi publish version mới; không tự gửi thông báo hoặc gọi API khách hàng khi replay.

Ở 10× dữ liệu, đo embedding cache-miss, số chunk, độ trễ quarantine review và small-files trước khi đổi công nghệ. Dự kiến chi phí đáng chú ý là token/embedding và nhân công kiểm duyệt, nhưng cần đo thay vì khẳng định tỷ trọng. Giảm chi phí bằng delta ingestion, cache và gộp file; ước tính token trước mỗi backfill. Nếu single-writer hoặc I/O trở thành bottleneck, chuyển dữ liệu phân tích sang table format/engine phù hợp và giữ semantic khóa/version. Chưa có số đo chứng minh cần hệ thống phân tán ngay ở quy mô giả định.

## Phương án bị loại

**Loại kiến trúc streaming toàn bộ bằng Spark/Flink, index trực tiếp mọi transcript và tự train từ thumbs-up.** Nó không đáp ứng quyền truy cập/thu hồi, dễ tạo leakage và tăng gánh vận hành cho đội hai người. Phương án thay thế là microbatch theo SLO, kho RAG được duyệt và flywheel có holdout/kiểm duyệt. Quyết định được xem xét lại khi số đo freshness hoặc throughput cho thấy microbatch không còn đủ.

## Sơ đồ kiến trúc

```text
PostgreSQL CDC     App events     Transcript export     Approved KB
      \                |                 |                   /
       +---------------+-----------------+------------------+
                                |
                Bronze: raw + lineage + retention
                                |
                Contract + PII gate ----> Quarantine
                                |         owner review/alert
                Silver: MERGE / dedup / history
                  |             |                 |
           Daily features   Weekly dataset   Approved doc chunks
           event-time       as-of + holdout   hash + model cache
                  |             |                 |
              Routing        Eval/SFT        Outbox -> RAG index
                  +-------------+-----------------+
                                |
                 Serving: tenant/ACL/version filters
                                |
                    Chatbot -> traces/feedback

Revocation ledger -> serving denylist (block use first)
                  -> purge raw/Silver/cache/index/datasets/copies
```

## Tiêu chí kiểm chứng trước khi triển khai

- Replay cùng batch và batch cũ không làm lùi trạng thái hoặc thay checksum ngoài correction được khai báo.
- Một case thu hồi phải bị chặn ở serving, kể cả khi worker index mất kết nối; replay không hồi sinh nó.
- Đo freshness tài liệu, P99 lateness, tỷ lệ quarantine và PII recall trên dữ liệu thử có nhãn.
- Kiểm tra feature parity và leakage; dataset train không chứa nhóm hội thoại thuộc eval.
- Load test xác nhận p95 latency và token budget. Các mục tiêu trên chưa được chứng minh bởi lab.

Tham khảo: `docs/bonus/BONUS-CHALLENGE.md`, `docs/RUBRIC.md`, cùng các nguyên tắc Bronze/Silver/Gold, LSN, snapshot và cache đã thực hành. Codex hỗ trợ brainstorm và soạn thảo; học viên cần review và giải thích các quyết định.
