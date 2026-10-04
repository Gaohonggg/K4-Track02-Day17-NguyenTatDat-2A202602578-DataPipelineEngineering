# Checklist nộp Lab 17

Nguồn yêu cầu: [SUBMISSION](../docs/SUBMISSION.md), [RUBRIC](../docs/RUBRIC.md), [RULES](../docs/RULES.md). Phần lõi 100 điểm, B2 thiết kế tối đa +5; không làm B1 hoặc Airflow.

## Nội dung và bằng chứng trong repo

- [x] Giữ mã nguồn đề bài, tests, seed và công cụ chấm; chỉ sửa ba file pipeline cho core.
- [x] Verify 18/18 và pytest 34 passed: [verify](final/verify.txt), [pytest](final/pytest.txt).
- [x] Checksum fresh build bằng ba rerun: [checksums.txt](checksums.txt), [rerun](final/rerun3.txt).
- [x] P99 đo từ Bronze bằng 3 ngày, lookback bằng 3: [lateness](final/lateness.txt).
- [x] dbt build PASS=19, ERROR=0 và parity đạt PARITY: [dbt](final/dbt.txt), [parity](final/parity.txt).
- [x] [REPORT](REPORT.md) có triệu chứng, nguyên nhân, cách sửa, khái niệm, lựa chọn kỹ thuật, hai câu suy ngẫm, khai báo AI và sáu output thực tế.
- [x] [B2 DESIGN](../bonus/DESIGN.md) có trên 600 từ, sáu quyết định với đánh đổi, phương án bị loại và sơ đồ. Không cần prototype, ảnh Airflow hoặc output B1 cho lựa chọn này.
- [x] `.venv`, lake, warehouse, dbt database/target/logs và `.env` không thuộc file bài nộp.

Baseline được giữ riêng trong `baseline/` để đối chiếu triệu chứng trước khi sửa; kết quả cuối dùng `final/` và `checksums.txt`.

## Các bước học viên cần làm trước khi nộp

- [x] Xác nhận họ tên/MSSV trong REPORT: Nguyễn Tất Đạt / 2A202602578.
- [ ] Đọc và giải thích được code sửa và thiết kế B2.
- [ ] Khi xem theo định dạng trang, kiểm tra mục 1–4 (459 từ theo khoảng trắng, không tính metadata/output) theo A4, Arial 12, giãn dòng 1,15, lề 2 cm; không cần nộp PDF.
- [x] Hồ sơ cục bộ gồm commit code/bằng chứng `2040c10` và commit tài liệu tiếp sau; kiểm tra lại `git status --short` trước khi push.
- [ ] Push commit bài nộp lên remote và xác nhận SHA trên GitHub khớp `git rev-parse HEAD`.
- [ ] Đặt/xác nhận repo public; mở URL khi chưa đăng nhập và kiểm tra các file bắt buộc có thể đọc được.
- [ ] Nộp **một URL repo** vào ô **K4 / Track 02 / Day 17** trên LMS, không nộp bằng PR; giữ repo truy cập được đến khi chấm xong.
- [ ] Xác nhận deadline theo lịch lab/thông báo key coach: mặc định 23:59 ngày diễn ra lab, Asia/Ho_Chi_Minh. Ngày seed 2026-08-10..16 không phải deadline.

URL bài nộp: https://github.com/Gaohonggg/K4-Track02-Day17-NguyenTatDat-2A202602578-DataPipelineEngineering

## Lệnh chốt và push

Chạy từ repo root, kiểm tra branch hiện hành là `main` trước khi dùng lệnh push sau:

```bash
git status --short
git log -2 --oneline
git rev-parse HEAD
git push origin main
```

Commit được thực hiện cục bộ không đồng nghĩa đã push hoặc đã nộp LMS. Trạng thái public và deadline cần học viên xác nhận.

## Tái lập khi người chấm cần

Repo này sử dụng môi trường `.venv` quản lý bằng uv. Nếu chưa có môi trường, người chấm có thể tạo rồi cài dependencies bằng các lệnh tương đương setup của đề bài:

```bash
uv venv --python 3.12 .venv
uv pip install --python .venv/bin/python -r requirements.txt -r requirements-dbt.txt
make verify
make test
make rerun3
make lateness
make dbt
make parity
```

Môi trường của học viên đã có sẵn; không cần tạo lại hoặc chạy lại các kiểm tra đã pass chỉ để hoàn thiện tài liệu. Các lệnh tái lập sẽ dựng lại warehouse và checksum.
