# PO Pulse MVP

## Adjust · Growth & LTV (v4)

Tab mới đọc report JSON `{rows,totals,warnings}`, CSV/TSV và sheet đầu tiên của Excel. Có dataset demo phân theo ngày/app/country/OS/channel/campaign/adgroup/creative, installs/revenue/cost, các cột cumulative cohort revenue và eligible cohort users để thử LTV. Filter date presets, app token, country code, OS, channel, network, campaign attribution/network, adgroup, creative; advanced include/exclude/contains/prefix/suffix và metric comparisons. API settings: UTC offset, currency, attribution source/status/type, ad spend source, IAP gross mode, cohort maturity, environment, ad revenue sources và period-over-period. Cấu trúc mô phỏng Datascape, không là bản sao pixel của Adjust.

API phía máy chủ: `POST /api/adjust/catalog` → Adjust filters_data; `POST /api/adjust/report` → Adjust JSON Report Service. Host cố định automate.adjust.com, Bearer token trong header upstream, không redirect. Cần đăng nhập và same-origin; token từ ô password chỉ trong tab và request memory. Có thể dùng hosted secret `ADJUST_API_TOKEN` qua Sites environment cho owner-private site, nhưng MVP chưa lưu token hay đồng bộ lịch tự động. Không gửi token vào query string, source, report export, logs hoặc localStorage. Source audience phải giữ owner-private; nếu mở rộng audience cần bổ sung authorization theo user.

Luồng: nhập token → Kết nối/tải catalog → chọn app và filters → chọn đúng API Metric IDs (có tìm metric catalog) → Pull report. Metric/package khác nhau theo tài khoản, không đoán slug LTV/cohort từ tên hiển thị. Default pull chỉ installs + cost là IDs xác nhận trong tài liệu. Metadata metrics bị tài khoản từ chối thì lấy API Metric ID từ glossary. Chưa kiểm chứng live bằng token thật của người dùng.

LTV quan sát Dn được tính bằng ratio của tổng cumulative cohort revenue Dn / tổng eligible cohort users Dn, loại install cohort chưa đủ Dn theo observation cutoff. Không dùng revenue activity-day / installs làm LTV. Các cột `cohort_revenue_dN` và `cohort_users_dN` là schema demo, không tự coi là Adjust API slugs; chọn hai cột tương ứng khi có report thật. Chart revenue và LTV riêng. Không cộng các tỷ lệ hoặc LTV; unknown metrics nhiều dòng mặc định null đến khi cấu hình aggregation. CPI dùng tổng cost/tổng installs. Không gộp nhiều currency. API source settings thay đổi yêu cầu pull lại, không fake conversion/attribution trên file import. Chỉ lọc dimension có trong source table.

MVP từ chối pagination, >50.000 rows, >15 MB API response; mỗi request report <=366 ngày ở UI. Rate limit không retry liên tục. Token không hợp lệ / thiếu quyền / timeout hiển thị lỗi rõ ràng. Không lưu report bền vững.

Kiến trúc hosting v4 chuyển từ static sang Worker ESM: `scripts/build-worker.cjs` embed toàn bộ UI assets vào `worker/index.js` và `dist/server/index.js`, không cần dependency. Giữ toàn bộ UI cũ trong `dist/` để tiếp tục chỉnh; build sao chép manifest vào dist. `node serve-adjust.cjs` preview local có API proxy với identity giả chỉ cho loopback, không dùng ngoài dev. `node adjust-check.cjs` kiểm tra filters/LTV/maturity, proxy auth/origin/host/token, pagination và Worker assets. Remote build dùng package.json → build script.

API docs: https://dev.adjust.com/en/api/rs-api/reports ; https://dev.adjust.com/en/api/rs-api/filters-data ; https://dev.adjust.com/en/api/rs-api/authentication .

## Retention và Custom metric chart (v3)

Retention explorer có preset R1/R3/R7/R14/R30/R60/R90/R180, chọn khoảng ngày R hoặc các preset, lọc cohort từ X đến Y, observation cutoff, active event, platform và quốc gia. Vẽ đường retention theo tuổi cohort, chart một Rx theo ngày cohort và ma trận cohort × Rx. Có thể so nhóm có/không event D0. Cohort chưa đủ ngày hoặc nhóm không có user hiển thị null/“—”, không vẽ thành 0. Retention là exact-day theo UTC, mẫu số riêng cho từng Rx; tổng dùng user count chứ không trung bình tỷ lệ cohort. Nhận event log từ file import hoặc BigQuery.

A/B có **Custom metric · Chart explorer** ở đầu màn hình: chọn bất kỳ metric đã tạo, ngày X–Y, và trục ngày phát sinh event hoặc ngày cohort exposure. Thêm metric tự phân tích và chọn chart metric đó. Biểu đồ daily dùng các user đủ measurement window thuộc cohort đã lọc; mẫu số tại từng ngày là các user đang ở trong measurement window tại ngày đó. Không đồng nghĩa DAU hay tổng KPI toàn app. Chart theo cohort dùng metric toàn measurement window cho cohort của ngày tương ứng. Thẻ lift dưới chart so toàn cohort/window đã lọc, không phụ thuộc khoảng ngày riêng của chart. Tối đa 366 ngày lịch/chart.

Kiểm tra `node chart-check.cjs` bao phủ R tùy chỉnh, maturity, nhóm event, lọc ngày và daily denominator. Đã kiểm tra trình duyệt: R3 từ X–Y, R60 thiếu thời gian, chart metric mới và thay đổi khoảng ngày.

## Firebase A/B testing (v2)

Vào **Firebase A/B testing → Dùng A/B demo**, hoặc import Firebase event log. Hỗ trợ raw JSON array / `{events:[...]}`, CSV/TSV/Excel đã flatten, và kết quả BigQuery. Membership đọc từ `user_properties.firebase_exp_XX`, hoặc cột `experiment_id` + `variant`; variant 0 là baseline mặc định theo Firebase. Experiment ID giữ chuỗi, ví dụ `01`.

Metric builder: conversion (% unique exposed user có event), events/user, sum(parameter)/user, tổng event, tổng parameter, và trung bình các mean giá trị theo user có event. Chọn goal event, parameter, hướng kỳ vọng, điều kiện bằng/khác/chứa/≥/≤. Có thể thêm tối đa 12 metric, xuất/nhập cấu hình JSON và kết quả CSV. Phải lọc currency khi metric revenue có nhiều tiền tệ.

Filter: experiment, baseline, variant hiển thị, exposure date, observation cutoff, measurement window 1–90 ngày, country, platform, app version, source, app ID, device category. Thêm filter từ custom user properties hoặc event parameters tại exposure. Multi-select: Ctrl/Cmd. Không chọn = tất cả.

Kết quả: sample size, metric từng variant, delta và lift so baseline, tăng/giảm so kỳ trước cùng độ dài, biểu đồ variant và biểu đồ cohort exposure. Khoảng tin cậy 95% và p-value là **normal approximation frequentist**, cấp user, n>=30/nhóm; conversion cần ít nhất 5 success và 5 failure mỗi nhóm. Không tương đương kết luận Bayesian trong Firebase, chưa điều chỉnh multiple comparisons/sequential testing. Tổng metric không có CI, không dùng để chọn winner khi kích thước nhóm khác nhau.

Định nghĩa: identity=(app, platform, user_id). Exposure là timestamp đầu tiên thấy membership trong export; chỉ event sau exposure trong cửa sổ cố định được tính. User đổi variant bị loại, cohort chưa đủ thời gian bị loại. User không có goal event vẫn nằm trong mẫu số. User attributes filter được đóng băng tại exposure. Event_id nếu có sẽ deduplicate theo user; thiếu ID giữ từng dòng (không suy đoán event trùng). Property missing và numeric param lỗi có cảnh báo; không dùng aggregated Firebase summaries thay event log. Dữ liệu gồm nhiều experiment được giữ membership mà không nhân bản event. Quan sát chỉ đến cutoff đã chọn.

Firebase SQL adapter tạo query GA4 daily tables (`project.analytics_x.events_*`), serialize event_params/user_properties thành JSON để tương thích kết quả BigQuery dạng scalar. Phải lấy tất cả event/user từ lịch sử experiment, cả kỳ trước và follow-up; bị cắt ở giới hạn BigQuery 50.000 dòng thì nút nhận dữ liệu từ chối phân tích. Thư viện chỉ phân tích export, chưa tự đồng bộ Firebase. Chưa chạy trên dataset thật của người dùng.

Nguồn schema: https://firebase.google.com/docs/ab-testing/ab-concepts và https://support.google.com/analytics/answer/7029846.

Kiểm tra bổ sung: `node ab-check.cjs` — raw/flat Firebase, zero-event denominator, dedup, switched variants, maturity, fixed measurement horizon, filter tại exposure, previous cohorts, numeric missing, uncertainty và SQL adapter.

Mở `dist/index.html` hoặc chạy `node serve.cjs` rồi truy cập http://127.0.0.1:4173.

## Có thể sử dụng
- Import CSV/TSV/JSON/XLSX/XLS, chọn sheet, nhận diện và chỉnh vai trò cột.
- Lọc sản phẩm, chọn metric, biểu đồ thời gian, ranking, biến động kỳ liền trước và xuất phân tích Markdown.
- Phân tích downloads, revenue, DAU, spend, retention từ file; không suy ra KPI thiếu dữ liệu.
- Event retention D1/D3/D7/D14/D30, phân nhóm event ở D0, chỉ lấy cohort đủ thời gian quan sát.
- BigQuery REST với token OAuth ngắn hạn, dry run, giới hạn bytes billed, polling và pagination tối đa 50.000 dòng. Tạo SQL hoặc nhập SELECT/WITH tùy chỉnh; kết quả đưa vào dashboard.

## Phạm vi
Phân tích theo quy tắc, chưa dùng LLM và chưa tích hợp API Sensor Tower/AppMagic trực tiếp. File được giữ trong RAM, không tải lên server, không lưu bền vững. Tải lại trang xóa dữ liệu và token. Chỉ số nguồn giữ nguyên đơn vị; không quy đổi tiền tệ. Tỷ lệ và DAU lấy trung bình đơn giản khi thiếu mẫu số, không đại diện DAU tổng danh mục đã deduplicate.

Retention: UTC, ngày lịch, active = bất kỳ event nào, first_open đầu tiên trong dữ liệu. Export cần đầy đủ lịch sử cohort và event đến ngày cuối. Phân nhóm D0 tránh dùng event phát sinh sau thời điểm đo; khác biệt không chứng minh quan hệ nhân quả.

BigQuery cần billing project, location, access token có scope Google Cloud phù hợp và quyền jobs.create + đọc dữ liệu. Không dùng private key. Không có credentials trong source. Chưa kiểm chứng live vì chưa có dataset/token người dùng. OAuth đăng nhập lâu dài và backend quản lý token nằm trong backlog.

Schema SQL mẫu: user_id STRING, event_name STRING, event_date DATE. GA4 cần view chuẩn hóa trước. Dung lượng mặc định tối đa 1 GB/query, có thể chỉnh. Query tùy chỉnh phải SELECT/WITH một statement; SQL không phù hợp bị BigQuery từ chối. Cần quyền tối thiểu cho tài khoản.

## Kiểm tra
`node check.cjs` kiểm tra parser, ngày, cohort, exposure D0, SQL và đọc/ghi XLSX. `node --check dist/app.js` kiểm tra cú pháp. Browser preview có thể cần chạy trên cùng thiết bị với server.

## Thứ tự mở rộng
P1: OAuth + GA4 adapter, chuẩn hóa metric/currency, lưu report, cohort phân đoạn; UA ROAS/LTV/payback; A/B testing và funnel.
P2: LiveOps/economy, benchmark đối thủ/ASO/review, alert bất thường, roadmap RICE/decision log, hợp tác và phân quyền.

Thư viện Excel: SheetJS CE 0.20.3 (Apache 2.0), bản phân phối kèm trong dist. Font Google tải khi online; fallback sans-serif khi offline.
