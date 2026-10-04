# K4-Track02-Day17 — Report cá nhân

**Họ tên / MSSV:** Nguyễn Tất Đạt / 2A202602578.

**Repo:** https://github.com/Gaohonggg/K4-Track02-Day17-NguyenTatDat-2A202602578-DataPipelineEngineering

**Commit mã nguồn và bằng chứng đã kiểm tra:** `2040c10` — `fix: make CDC pipeline keyed, delete-aware and replay-safe`.

**Commit hồ sơ nộp:** commit tiếp sau có tên `docs: complete lab report and B2 pipeline design`, chứa REPORT, checklist và bonus; SHA cuối xem bằng `git rev-parse HEAD`.

**AI đã dùng và phạm vi hỗ trợ:** Codex hỗ trợ đọc rubric, phân tích baseline, sửa ba lỗi, rà soát log và soạn REPORT/thiết kế B2. Học viên chạy toàn bộ lệnh tạo bằng chứng trong `.venv` quản lý bằng uv; cần review và giải thích các thay đổi khi bảo vệ bài.

**Nguồn tham khảo:** README, `docs/RUBRIC.md`, `docs/SUBMISSION.md`, `docs/CHECKPOINTS.md`, `docs/bonus/BONUS-CHALLENGE.md` và mã nguồn đề bài.

## 1. Ba lỗi

- **Silver:** Triệu chứng: 24 hàng/12 ticket; T-91 có ba trạng thái; RAG 22 hàng/9 khóa chunk. Nguyên nhân: dedup chỉ trong batch, sau đó `INSERT` giữa các batch. Sửa `pipeline/silver.py`: `MERGE` theo `ticket_id`, update khi `s._lsn > t._lsn`. Khái niệm: Silver có khóa, idempotency và LSN guard; replay cũ không ghi đè mới.
- **Late data:** Triệu chứng: u05 ngày 08-12 chỉ có `(2,1,0)` events/clicks/down, cần `(5,3,1)`; feature khác full recompute. Nguyên nhân: lookback bằng 0, bỏ qua event ngày 08-12 nhận ngày 08-15. Sửa `pipeline/config.py`: lookback 3 theo P99 Bronze. Khái niệm: event time khác ingest time; recompute partition nhận dữ liệu muộn.
- **CDC delete:** Triệu chứng: T-97 chưa deleted, còn một hàng trong snapshot mới nhất và hai chunk RAG. Nguyên nhân: khóa chỉ đọc từ `after=null`, delete bị staging lọc mất. Sửa `pipeline/staging.py`: `coalesce(after.ticket_id,before.ticket_id)`; nội dung vẫn đọc từ `after` để null. Khái niệm: CDC delete khác Kafka tombstone; xoá phải lan xuống Gold.

## 2. Các con số

- Bronze 43 event records: P50=0, P95=2,90, P99=3, max=3 ngày; `LOOKBACK_DAYS=ceil(P99)=3`.
- Verify **18/18**, pytest **34 passed**; Silver **12 ticket/39 event**, quarantine **2**; RAG **8 chunk**.
- Rerun **C0=C1=C2=C3**, combined Gold `39e115c510ecdf526800eac227158a4f`; dbt **PASS=19, ERROR=0**; parity **PARITY**.

## 3. Lựa chọn kỹ thuật

- MERGE cập nhật thực thể theo khóa/LSN; overwrite partition tính lại aggregate trong `[day-3,day]`, tránh cộng event muộn hai lần. Cửa sổ lớn hơn tăng chi phí; dữ liệu ngoài P99 cần backfill.
- Tombstone giữ khóa/LSN chống hồi sinh khi replay; đánh đổi là giữ hàng metadata. Kafka tombstone `value=null` chỉ phục vụ compaction và được bỏ qua ở staging.
- Snapshot dựng từ Bronze theo batch `<= day`, feedback giới hạn thời điểm nhận, giữ priority lúc tạo để tránh leakage; snapshot cũ bất biến để tái lập.
- DuckDB phù hợp dữ liệu nhỏ và transaction cục bộ; dbt cung cấp merge, microbatch, contract/test SQL. Spark tăng chi phí vận hành khi lab chưa cần xử lý phân tán. uv quản lý dependencies trong `.venv`.

## 4. Hai câu hỏi suy ngẫm

1. **Snapshot và xoá:** Lab giữ snapshot cũ; production cần sổ thu hồi, chặn sử dụng ngay, purge PII ở raw/transcript/cache/index và snapshot liên quan; tạo version thay thế, đánh dấu bản cũ không còn được dùng. Theo dõi bản sao/model đã tiêu thụ; audit giữ metadata tối thiểu, không giữ văn bản bị xoá.
2. **Tên người còn lọt:** Đặt NER tiếng Việt kết hợp regex tại Bronze→Silver, chặn xuất sang Gold khi chưa xử lý, kiểm tra lại Gold/cache. Đo precision/recall riêng cho tên/email/phone trên mẫu có nhãn, gồm viết tắt/không dấu; kiểm tra thủ công false negative, cảnh báo drift. Quarantine hạn chế truy cập; masking không đồng nghĩa ẩn danh hoàn toàn.

## 5. Output thực tế

Mục 1–4 là phần phân tích; thông tin học viên, output và bonus nằm ngoài giới hạn một trang theo `docs/SUBMISSION.md`.

Các output dưới đây lấy từ file do học viên chạy trong `submission/final/`; không chạy lại hoặc tự viết kết quả. Riêng dbt bỏ escape ANSI màu và khoảng trắng cuối dòng để Markdown dễ đọc, giữ nguyên nội dung và thứ tự dòng; bản raw vẫn ở `final/dbt.txt`.

### Môi trường và cách chạy

Python 3.12.13, uv 0.11.18; dbt-core 1.12.5, dbt-duckdb 1.11.0. Dùng `.venv` hiện có; cài dbt bằng `uv pip install --python .venv/bin/python -r requirements-dbt.txt` thay cho target setup dùng pip. Log cài đặt: [setup-dbt.txt](final/setup-dbt.txt).

Baseline: [verify 8/18](baseline/verify.txt), [pytest 9 failed/25 passed](baseline/pytest.txt), [rerun FAIL](baseline/rerun3.txt), [lateness](baseline/lateness.txt). Đây là bằng chứng trước khi sửa, không phải kết quả nộp cuối.

### make verify

Nguồn: [final/verify.txt](final/verify.txt).

```text
$ make verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt
```

### make test

Nguồn: [final/pytest.txt](final/pytest.txt).

```text
$ make test
..................................                                       [100%]
34 passed in 0.69s
```

### make rerun3

Nguồn: [final/rerun3.txt](final/rerun3.txt).

```text
$ make rerun3
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums
```

### make lateness

Nguồn: [final/lateness.txt](final/lateness.txt).

```text
$ make lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3
```

### make dbt

Nguồn: [final/dbt.txt](final/dbt.txt).

```text
$ make dbt
cd dbt_project && DBT_PROFILES_DIR=. /Users/nguyendat/Documents/VIN/LAB/K4-Track02-Day17-NguyenTatDat-2A202602578-DataPipelineEngineering/.venv/bin/dbt build --event-time-start 2026-08-10 --event-time-end 2026-08-17
08:33:21  Running with dbt=1.12.5
08:33:21  Registered adapter: duckdb=1.11.0
08:33:21  Unable to do partial parsing because saved manifest not found. Starting full parse.
08:33:22  Found 5 models, 13 data tests, 2 sources, 502 macros, 1 unit test
08:33:22
08:33:22  Concurrency: 1 threads (target='dev')
08:33:22
08:33:22  1 of 19 START sql view model main.stg_events ................................... [RUN]
08:33:22  1 of 19 OK created sql view model main.stg_events .............................. [OK in 0.05s]
08:33:22  2 of 19 START sql view model main.stg_ticket_changes ........................... [RUN]
08:33:22  2 of 19 OK created sql view model main.stg_ticket_changes ...................... [OK in 0.01s]
08:33:22  3 of 19 START sql incremental model main.silver_events ......................... [RUN]
08:33:22  3 of 19 OK created sql incremental model main.silver_events .................... [OK in 0.03s]
08:33:22  4 of 19 START unit_test silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [RUN]
08:33:22  4 of 19 PASS silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [PASS in 0.05s]
08:33:22  8 of 19 START sql incremental model main.silver_tickets ........................ [RUN]
08:33:22  8 of 19 OK created sql incremental model main.silver_tickets ................... [OK in 0.03s]
08:33:22  5 of 19 START test not_null_silver_events_event_id ............................. [RUN]
08:33:22  5 of 19 PASS not_null_silver_events_event_id ................................... [PASS in 0.01s]
08:33:22  6 of 19 START test not_null_silver_events_user_id .............................. [RUN]
08:33:22  6 of 19 PASS not_null_silver_events_user_id .................................... [PASS in 0.01s]
08:33:22  7 of 19 START test unique_silver_events_event_id ............................... [RUN]
08:33:22  7 of 19 PASS unique_silver_events_event_id ..................................... [PASS in 0.01s]
08:33:22  9 of 19 START test accepted_values_silver_tickets_category__bug__billing__other  [RUN]
08:33:22  9 of 19 PASS accepted_values_silver_tickets_category__bug__billing__other ...... [PASS in 0.01s]
08:33:22  10 of 19 START test accepted_values_silver_tickets_priority__low__medium__high . [RUN]
08:33:22  10 of 19 PASS accepted_values_silver_tickets_priority__low__medium__high ....... [PASS in 0.01s]
08:33:22  11 of 19 START test accepted_values_silver_tickets_status__open__pending__closed  [RUN]
08:33:22  11 of 19 PASS accepted_values_silver_tickets_status__open__pending__closed ..... [PASS in 0.01s]
08:33:22  12 of 19 START test not_null_silver_tickets__lsn ............................... [RUN]
08:33:22  12 of 19 PASS not_null_silver_tickets__lsn ..................................... [PASS in 0.01s]
08:33:22  13 of 19 START test not_null_silver_tickets_is_deleted ......................... [RUN]
08:33:22  13 of 19 PASS not_null_silver_tickets_is_deleted ............................... [PASS in 0.01s]
08:33:22  14 of 19 START test not_null_silver_tickets_ticket_id .......................... [RUN]
08:33:22  14 of 19 PASS not_null_silver_tickets_ticket_id ................................ [PASS in 0.01s]
08:33:22  15 of 19 START test unique_silver_tickets_ticket_id ............................ [RUN]
08:33:22  15 of 19 PASS unique_silver_tickets_ticket_id .................................. [PASS in 0.01s]
08:33:22  16 of 19 START sql microbatch model main.gold_feature_daily .................... [RUN]
08:33:22  Batch 1 of 7 START batch 2026-08-10 of main.gold_feature_daily ....................... [RUN]
08:33:22  Batch 1 of 7 OK created batch 2026-08-10 of main.gold_feature_daily .................. [OK in 0.02s]
08:33:22  Batch 2 of 7 START batch 2026-08-11 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 2 of 7 OK created batch 2026-08-11 of main.gold_feature_daily .................. [OK in 0.02s]
08:33:23  Batch 3 of 7 START batch 2026-08-12 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 3 of 7 OK created batch 2026-08-12 of main.gold_feature_daily .................. [OK in 0.01s]
08:33:23  Batch 4 of 7 START batch 2026-08-13 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 4 of 7 OK created batch 2026-08-13 of main.gold_feature_daily .................. [OK in 0.01s]
08:33:23  Batch 5 of 7 START batch 2026-08-14 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 5 of 7 OK created batch 2026-08-14 of main.gold_feature_daily .................. [OK in 0.01s]
08:33:23  Batch 6 of 7 START batch 2026-08-15 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 6 of 7 OK created batch 2026-08-15 of main.gold_feature_daily .................. [OK in 0.01s]
08:33:23  Batch 7 of 7 START batch 2026-08-16 of main.gold_feature_daily ....................... [RUN]
08:33:23  Batch 7 of 7 OK created batch 2026-08-16 of main.gold_feature_daily .................. [OK in 0.01s]
08:33:23  16 of 19 OK created sql microbatch model main.gold_feature_daily ............... [SUCCESS in 0.09s]
08:33:23  17 of 19 START test dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [RUN]
08:33:23  17 of 19 PASS dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [PASS in 0.01s]
08:33:23  18 of 19 START test not_null_gold_feature_daily_event_date ..................... [RUN]
08:33:23  18 of 19 PASS not_null_gold_feature_daily_event_date ........................... [PASS in 0.01s]
08:33:23  19 of 19 START test not_null_gold_feature_daily_user_id ........................ [RUN]
08:33:23  19 of 19 PASS not_null_gold_feature_daily_user_id .............................. [PASS in 0.01s]
08:33:23
08:33:23  Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 0.43 seconds (0.43s).
08:33:23
08:33:23  Completed successfully
08:33:23
08:33:23  Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19
```

### make parity

Nguồn: [final/parity.txt](final/parity.txt).

```text
$ make parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree
```

## 6. Bonus B2 — Thiết kế

Chọn B2 brainstorm (+5 điểm tối đa): [bonus/DESIGN.md](../bonus/DESIGN.md), thiết kế pipeline chatbot CSKH tiếng Việt với sáu quyết định, đánh đổi, một kiến trúc bị loại và sơ đồ. Các quy mô/SLO là giả định được ghi rõ, không phải kết quả production. Không nộp B1 hay B2 Airflow; không yêu cầu ảnh hoặc prototype cho hướng này.

## 7. Hồ sơ nộp

Code sửa, checksum PASS, báo cáo, log gốc và DESIGN được commit cùng repo. Các bước push, xác nhận repo public và nộp URL LMS được liệt kê trong [CHECKLIST.md](CHECKLIST.md); chưa được xem là hoàn tất chỉ vì đã có commit cục bộ.
