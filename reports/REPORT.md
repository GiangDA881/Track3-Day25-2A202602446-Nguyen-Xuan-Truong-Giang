# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Xuân Trường Giang (MSV: 2A202602446)  **Commit:** 0105cbdf2c0ad9ac0bf11a9f146045d36070f2fb  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4 điểm tập dữ liệu ẩn khi chấm)

> Mọi nhận định trong báo cáo được trích xuất trực tiếp từ các file kết quả thực nghiệm: `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, và `reports/cache/*.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | PASS |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0s | Giới hạn thời gian chờ mỗi attempt. Ở G6 (`alpha-eu` treo), nếu để 30s thì chỉ 1 attempt thất bại là cạn sạch deadline 30s của turn, khiến availability rớt còn 94.88%. Giá trị 8.0s đủ cho 1–2 lần thử lại và failover sang `beta`. |
| `max_attempts_per_provider` | 2 | Cho phép thử lại 1 lần trên cùng provider để cứu lỗi transient (G3 brownout 500), nhưng ngắt ngay attempt thứ 2 để chuyển sang provider kế tiếp, ngăn chặn retry storm (naive client thử 36 lần khiến ampl vọt lên 9.9 ở G2 và 17.0 ở G5). |
| `backoff_base_s` / `cap_s` | 0.25s / 4.0s | Áp dụng Full Jitter Exponential Backoff. Base 0.25s giúp phản ứng nhanh khi lỗi nhẹ, Cap 4.0s giữ độ trễ retry nằm trong ngân sách thời gian mà vẫn phân tán thundering herd sau đợt rate limit (G1). |
| `retry_budget_ratio` | 0.1 | Mỗi request đầu tiên chỉ nạp 0.1 token, buộc tổng lượng retry không thể vượt quá 10% lưu lượng chính ngạch. Nhờ đó ở G3 `retry_ratio` chỉ là 0.034 (đạt chuẩn ≤ 0.10) thay vì 0.164 của naive. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Đo lường tỷ lệ lỗi trên 20 request gần nhất, chỉ kích hoạt khi đã thu thập tối thiểu 10 mẫu để tránh ngắt mạch giả khi tải thấp ban đêm hoặc gặp vài lỗi đơn lẻ. |
| `breaker.failure_rate_threshold` | 0.5 | Khi tỷ lệ lỗi ≥ 50% trong cửa sổ 20 call, trip sang OPEN ngay lập tức. Ở G2 và G6, mạch ngắt nhanh chóng giúp chuyển hướng sang `alpha-eu` và `beta`, giảm tải cho `alpha`. |
| `breaker.slow_call_s` | 5.0s | Ở G4, provider `alpha` phản hồi chậm hơn 5s (bình thường 1.2s). Mức 5.0s bắt trọn tình trạng nghẽn hàng đợi trước khi dẫn tới timeout, kéo p95 từ 13.764s (naive) xuống 5.562s (yours). |
| `breaker.slow_rate_threshold` | 0.6 | Khi 60% request trong cửa sổ bị chậm trên 5s, breaker trip sang OPEN để bảo toàn SLA độ trễ của khách hàng. |
| `breaker.open_duration_s` | 10.0s | Thời gian giữ OPEN để provider có đủ 10 giây xả sạch hàng đợi phía sau (G2 `max_queue_wait_s: 10`). Sau 10s chuyển sang HALF-OPEN thăm dò. |
| `breaker.half_open_max_probes` | 2 | Chỉ cho phép 2 request thăm dò đồng thời ở trạng thái HALF-OPEN. Tránh thundering herd dồn dập khiến provider vừa hồi phục lại sập tiếp. |
| `breaker.half_open_successes` | 2 | Cần 2 request probe liên tiếp thành công để xác nhận provider đã thực sự khỏe mạnh trước khi đóng mạch (CLOSED) hoàn toàn. |

### Sơ đồ quyết định của `on_error`

| Loại lỗi (`ErrorKind`) | Ghi Breaker? | Retry cùng Provider? | Failover Provider khác? | Cooldown Provider? | Hành động kế tiếp |
|---|:---:|:---:|:---:|:---:|---|
| **CLIENT_ERROR** (400, 401, 422) | `release` (không đếm) | **Không** | **Không** | Không | `GiveUp("client_error")` ngay lập tức; không retry làm phí tài nguyên. |
| **QUOTA_EXHAUSTED** (429 `insufficient_quota`) | `release` (không đếm) | **Không** | **Có** | `now + 300s` | Đưa vào cooldown 300s; chuyển ngay sang provider kế tiếp trong ladder. |
| **RATE_LIMITED** (429 `rate_limit_exceeded`) | `release` (không đếm) | **Không** | **Có** | `now + retry_after` | Cooldown chính xác theo `retry_after`; chuyển ngay sang provider khác trong ladder. |
| **OVERLOADED** (503, 529), **SERVER_ERROR** (500, 502), **TIMEOUT** (0, 408) | `on_failure` | **Có** (nếu `tries < 2`, còn `budget`, kịp deadline) | **Có** (nếu hết lượt retry hoặc breaker OPEN) | Không | Nếu đủ điều kiện: tính `backoff.delay(tries)` và gửi lại. Nếu không: chuyển sang provider kế tiếp với `delay=0`. |

---

## 3. Ba thí nghiệm

### 1. Retry budget là lớp phòng thủ cuối
- **Thiết lập:** Vô hiệu hóa circuit breaker (`breaker.min_calls: 1000000000`) và tăng `max_attempts_per_provider: 5`.
  - **(a) Giữ Retry Budget:** 
    - G2: `amplification = 1.382`, `retry_ratio = 0.058`, `recovery = 0.0s`, `availability = 99.96%`.
    - G3: `amplification = 1.135`, `retry_ratio = 0.067`, `availability = 100%`.
  - **(b) Tắt Retry Budget:**
    - G2: `amplification = 1.672`, `retry_ratio = 0.491` (tăng gấp 8.5 lần!), `recovery = 30.0s`, `availability = 96.49%`.
    - G3: `amplification = 1.163`, `retry_ratio = 0.162` (vượt ngưỡng 0.10), `availability = 100%`.
- **Nhận xét:** Khi breaker bị cấu hình sai hoặc không hoạt động, Retry Budget là chốt chặn cuối cùng ngăn chặn khuếch đại lưu lượng. Ngay cả khi breaker hoạt động bình thường, hệ thống vẫn bắt buộc phải có cả hai: Circuit Breaker bảo vệ downstream provider khi lỗi tập trung ở một node, còn Retry Budget bảo vệ toàn hệ thống khỏi bão retry khi sự cố xảy ra trên diện rộng hoặc lỗi phân tán khắp các cụm.

### 2. Thử nghiệm `attempt_timeout_s: 30`
- **Kết quả thực nghiệm:**
  - Ở băng **G6** (`alpha-eu` bị treo hoàn toàn): `availability` tụt từ 100% xuống **94.88%** (FAIL), lý do give-up ghi nhận 108 lượt `deadline` và 14 lượt `no_provider_available`. Trong khi đó, `latency_p95_s` đạt 5.425s (giảm so với 5.961s ban đầu).
  - Ở băng **G4**: `availability = 100%`, `latency_p95_s = 5.683s` (hầu như không đổi so với 5.562s).
- **Giải thích hiện tượng Survivorship Bias:** Ở G6, các request gửi vào `alpha-eu` bị ngâm trọn 30s. Lúc này turn deadline của user (30s) cạn kiệt, client buộc phải hủy turn. Vì metric p95 chỉ tính trên *những lượt thành công*, những lượt thất bại nặng nề do timeout bị loại khỏi mẫu thống kê, tạo ra ảo giác rằng hệ thống phản hồi nhanh hơn dù độ sẵn sàng thực tế đã sập. Ở G4 không đổi vì breaker slow-call bắt được các call > 5s và mở mạch chỉ sau 10 request, không để request bị ngâm tới 30s.
- **Tiêu chuẩn cấu hình timeout "đúng":** Timeout không được đặt theo user deadline (30s) mà phải dựa trên `p99 latency lúc bình thường + buffer` và thời gian cần thiết để thực hiện ít nhất 1 lần failover trước khi hết deadline của turn (`turn_deadline / (số_bậc_fallback + 1)`).

### 3. Bỏ lọc tool calling khỏi `ladder()`
- **Kết quả trên băng G6:** Ghi nhận **256** lượt `incompatible_routes`, `availability` sụt từ 100% xuống **89.27%** (FAIL).
- **Giải thích cho Product Manager:** "Model `gamma-mini` là model siêu nhỏ, giá rẻ chỉ dùng để trả lời hội thoại văn bản cơ bản, hoàn toàn không có khả năng hiểu cấu trúc gọi hàm (tool calling) như truy vấn cơ sở dữ liệu đơn hàng hay kích hoạt hoàn tiền. Khi provider chính gặp sự cố, hệ thống thiếu bộ lọc đã mù quáng điều hướng 256 yêu cầu hoàn tiền sang `gamma-mini`. Hậu quả là 256 khách hàng bị hệ thống từ chối hoặc trả về phản hồi vô nghĩa, giao dịch hoàn tiền bị đứt đoạn, khiến trải nghiệm khách hàng sụt giảm nghiêm trọng dù hạ tầng máy chủ vẫn đang chạy."

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 ¢ ($14,967) | 1.123 | FAIL |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 ¢ ($0) | 0.303 | PASS |

- **Vì sao chỉ Idempotency Key là chưa đủ?** Idempotency key bảo đảm nhà cung cấp thanh toán/email không thực hiện 2 lần cùng một tác vụ. Tuy nhiên, nếu không có Checkpoint, mỗi khi worker bị crash (chết tiến trình), worker mới sẽ phải chạy lại toàn bộ quy trình từ bước đầu tiên (classify, lookup). Điều này làm lãng phí token LLM (`wasted_llm_per_crash` vọt lên 1.123), tăng chi phí và làm chậm thời gian xử lý của khách hàng.
- **Vì sao chỉ Checkpoint là chưa đủ?** Nếu chỉ có checkpoint mà không có idempotency key, khi worker gọi lệnh chuyển tiền thành công nhưng bị rớt mạng timeout trước khi nhận phản hồi hoặc crash ngay trước dòng lưu checkpoint, worker mới khởi động lại sẽ không thấy checkpoint và gọi API chuyển tiền lần thứ hai, gây ra thảm họa trừ tiền trùng lặp (naive client đã hoàn tiền trùng 128 lần, thiệt hại gần 15.000 USD).
- **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?** Nếu lưu checkpoint trước, trong trường hợp worker bị crash đúng lúc vừa ghi checkpoint xong nhưng chưa kịp gọi API thanh toán, worker mới khi resume sẽ thấy checkpoint đã tồn tại và đinh ninh rằng tiền đã được hoàn, từ đó bỏ qua bước này. Hậu quả là khách hàng không bao giờ nhận được tiền hoàn (`lost_refunds > 0`), gây vi phạm pháp lý nghiêm trọng.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page (delay 1m) | **page** | 27 min |
| `provider_outage` | page | page (delay 0m) | **page** | 4 min |
| `deploy_blip` | none | **page (False Alert!)** | **none** | Không báo động giả |
| `slow_leak` | ticket | not detected (bỏ lọt) | **ticket** | 1337 min |
| `degraded_evening` | page | page (delay 214m) | **page** | 291 min |
| `night_spike` | none | **page (False Alert!)** | **none** | Không báo động giả |
| `small_bump` | none | **page (False Alert!)** | **none** | Không báo động giả |

- **Số liệu Error Budget 30 ngày:**
  - Tổng số request: 25,901,755
  - Số lượng lỗi cho phép (SLO 99.5%): 129,508.8
  - Số lượng lỗi thực tế: 55,618
  - Availability đạt được: **99.7853%** (vượt chỉ tiêu SLO 99.5%)
  - Ngân sách lỗi đã tiêu hao (`budget_consumed`): **42.95%**
  - Ngân sách lỗi còn lại (`budget_remaining`): **57.05%**
  - **Quyết định Release Freeze:** **KHÔNG** đóng băng release (`freeze_releases: false`), vì ngân sách lỗi còn dư hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức on-call lúc 11h trưa?** Sự cố chỉ kéo dài 20 phút và tiêu tốn chưa đầy 0.3% ngân sách lỗi của cả tháng. Với quy tắc multi-window burn rate, cửa sổ ngắn 5 phút có thể tăng nhưng cửa sổ dài 1 giờ không đạt ngưỡng tiêu hao 14.4× (chỉ đốt khoảng 4% ngân sách nếu kéo dài trọn 1 giờ). Việc bắn page đánh thức kỹ sư cho những biến động nhỏ tự hết sẽ gây kiệt sức cảnh báo (alert fatigue), làm giảm độ nhạy bén của đội ngũ khi có thảm họa thực sự.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình ảnh phân tích trích xuất từ `reports/gameday/G2_overload_storm.png` và số liệu chuỗi thời gian từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

### Tóm tắt (3 câu)
Vào lúc virtual time t=120s, deployment `alpha` của ShopAgent đột ngột mất 75% năng lực GPU trong 3 phút, khiến hàng đợi quá tải và trả về lỗi 529. Do client v1 naive triển khai 3 tầng retry chồng chéo không jitter, không breaker và delay cố định, lượng request gửi vào `alpha` bị khuếch đại gấp 9.9 lần tạo thành bão retry storm. Sự cố khiến tỷ lệ sẵn sàng sụp đổ xuống 45.6%, và hệ thống bị kẹt trong trạng thái sập siêu bền (metastable failure) kéo dài thêm 155 giây sau khi provider đã hoàn toàn bình phục.

### Ảnh hưởng
- **1,285 lượt người dùng thất bại** (chỉ có 1,077 / 2,362 turn thành công, availability chỉ đạt 45.60%).
- Thời gian downtime kéo dài từ t=120s đến t=455s (tổng cộng **335 giây**, trong đó có 155 giây sập kéo dài vô ích sau khi hạ tầng đã hồi phục).
- Nếu sự cố này xảy ra trên hệ thống thực tế với lưu lượng 4 turn/s trong 1 tháng (10.3 triệu turn, cho phép 51,800 lỗi ở SLO 99.5%), chỉ riêng 1 sự cố này đã làm hỏng 1,285 turn (chiếm 2.48% error budget tháng).

### Timeline (giây ảo)

| t (s) | Sự kiện | Bằng chứng định lượng từ CSV |
|---:|---|---|
| **120** | `alpha` mất 75% GPU capacity (từ 18 server xuống còn 4.5 slot). | `attempts_alpha` = 5, `calls_ok` bắt đầu sụt giảm (từ 7 xuống 2). |
| **136** | Hàng đợi server đầy 10s, bắt đầu trả 529; client bắt đầu retry dồn dập. | `attempts_alpha` vọt từ 21 lên **53 attempts/s**, `calls_ok` rớt xuống 3. |
| **143** | Bão retry storm bùng nổ toàn diện; toàn bộ lượt user bị đóng băng. | `attempts_alpha` đạt đỉnh **254 attempts/s** (gấp 25 lần bình thường!), `calls_ok = 0`. |
| **200** | Quá tải trầm trọng; client timeout nhưng server vẫn tiếp tục xử lý việc vô ích trong queue. | Duy trì 150–180 attempts/s, 100% request người dùng bị lỗi hoặc timeout (`calls_ok = 0`). |
| **300** | `alpha` khôi phục 100% năng lực GPU (18 slot). | Năng lực vật lý đã hồi phục, nhưng `attempts_alpha` vẫn ở mức 115 req/s; `calls_ok` vẫn tiếp tục bằng **0**. |
| **301–452** | Hệ thống kẹt trong vòng lặp sập siêu bền (Metastable Failure). | Trong suốt 151 giây này, **mỗi giây đều có 0 calls_ok**, dù GPU đã bình thường, do server mải xử lý các request zombie mà client đã hủy từ lâu. |
| **453** | Hàng đợi cũ xả hết, hệ thống bắt đầu hồi phục. | `calls_ok` ghi nhận 4, sau đó tăng lên 8 (t=456) và trở lại bình thường ~12 calls_ok/s. |

### Trigger vs Root Cause
- **Trigger (Ngòi nổ):** Sự cố phần cứng/hạ tầng downstream khiến deployment `alpha` mất 75% số lượng GPU trong khoảng thời gian từ t=120s đến t=300s.
- **Root Cause (Nguyên nhân gốc rễ):** Kiến trúc client v1 thiếu khả năng tự vệ: 3 tầng retry lồng nhau (app × gateway × SDK = tối đa 36 attempt/call), độ trễ cố định không jitter, không có circuit breaker và không có cơ chế failover. Khi downstream chậm lại, client liên tục gửi lại request cũ, biến một đợt suy giảm hiệu năng cục bộ 3 phút thành thảm họa sập hệ thống toàn diện kéo dài gần 6 phút.
- **Cơ chế tự duy trì của Metastable Failure:** Server provider áp dụng cơ chế hàng đợi FIFO: khi client timeout hoặc user hủy turn sau 30s, server *vẫn tiếp tục xử lý* request đó đến cùng. Các request mới đến bị đẩy xuống cuối hàng đợi chờ quá 10s rồi bị hủy tiếp, trong khi CPU/GPU bị lãng phí 100% cho các request "ma" (zombie requests). Vòng lặp này tự nuôi sống chính nó cho tới khi áp lực tải mới giảm đột ngột mới có thể thoát ra.

### Điều gì giúp / Điều gì làm tệ hơn
- **Làm tệ hơn:** Cơ chế retry mù quáng không kiểm tra ngân sách; timeout không hủy tác vụ phía server; thiếu circuit breaker ngắt tải; không failover sang `alpha-eu` hay `beta`.
- **Giúp ích:** Circuit breaker sliding-window phát hiện lỗi và ngắt mạch ngay lập tức sang các provider dự phòng (`alpha-eu`, `beta`); Retry Budget khống chế trần retry ≤ 10%; Full Jitter giải phóng thundering herd.

### Action Items

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient` trang bị Sliding-Window Circuit Breaker và Retry Budget token-bucket (tỷ lệ 0.1). | **Prevent** | Client Team / Giang | Chạy benchmark G2 trong pipeline CI: `amplification ≤ 1.2`, `recovery_s ≤ 30s`. |
| 2 | Cấu hình HTTP Timeout chặt chẽ (8.0s) kèm header `X-Request-Deadline` để server tự động drop request đã quá hạn trong queue thay vì tiếp tục tính toán. | **Mitigate** | Platform / Infra Team | Tái hiện bài test hàng đợi nghẽn: server CPU load giảm về 0 ngay khi client hủy kết nối. |
| 3 | Triển khai hệ thống cảnh báo đa cửa sổ Multi-window Multi-burn-rate Alerting (page ở 14.4× và 6× burn rate). | **Detect** | SRE Team / Giang | Đảm bảo phát hiện sự cố tương tự trong vòng dưới 5 phút, 0 false page trên tập dữ liệu SLI 30 ngày. |

---

## 7. Một điểm yếu còn lại

- **Điểm yếu hiện tại:** Khi provider chính `alpha` gặp sự cố (như ở G2 hoặc G6), toàn bộ traffic tool call dồn sang `beta`. Deployment `beta` có chi phí gọi là **1.4 / call** (cao hơn 40% so với `alpha` là 1.0). Ở kịch bản G5 và G6, chi phí cho mỗi lượt thành công (`cost_per_good_turn`) tăng vọt lên **2.504** và **2.196**. Ngoài ra, nếu `beta` cũng bị suy giảm năng lực đồng thời do toàn bộ các client khác trên thị trường cùng đổ xô vào, hệ thống sẽ thiếu giải pháp thay thế cho các tool call.
- **Giải pháp khắc phục:**
  1. **Dynamic Cost-Aware Load Balancing:** Thay vì fallback tuần tự cố định, triển khai weighted routing có đo lường tỷ lệ lỗi và chi phí thời gian thực để phân bổ tải hài hòa giữa `alpha-eu` và `beta`.
  2. **Graceful Degradation / Model Cascading:** Phân loại độ phức tạp của tool call: với các tác vụ trích xuất đơn giản, có thể dùng prompt engineering để nén ngữ cảnh cho phép các model nhỏ hơn (như gamma-mini với structured JSON output) xử lý thay vì phụ thuộc 100% vào B-large.

---

## 8. Cache (Part G)

*Số liệu thực nghiệm từ `reports/cache/cached.json` và `reports/cache/resilient.json` (`make cache`):*

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.592** | **192** | **0** | **2** | **0.955** | **PASS** |

### Cấu hình `cache:` và lý do

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | **0.90** | Starter để 0.80 quá lỏng, gây ra **181 lỗi `other_question`** (trả nhầm câu hỏi này sang câu hỏi khác). Nâng lên 0.90 triệt tiêu hoàn toàn lỗi nhầm lẫn (`wrong_answers = 0`), trong khi vẫn giữ `cache_hit_ratio = 59.2%` (vượt ngưỡng yêu cầu ≥ 50%). Nếu tăng lên 0.97 thì hit ratio bị tụt xuống 49.9% (FAIL). |
| `ttl_s` | **30s** | Starter để 600s quá dài, khiến thay đổi phí vận chuyển không báo trước ở t=480s làm phát sinh tới 80 câu trả lời lỗi thời. Đặt TTL = 30s giới hạn số câu trả lời cũ xuống chỉ còn **2** (`outdated_answers = 2`, đạt chuẩn ≤ 15). |
| `max_stale_s` | **300s** | Sự cố outage toàn bộ provider kéo dài 120s (t=240 đến t=360s). Starter để `max_stale_s = 0` khiến tính năng stale-if-error bị vô hiệu hóa hoàn toàn, làm `faq_availability` chỉ đạt 85.29% (FAIL). Đặt 300s cho phép phục vụ 192 lượt câu trả lời cũ an toàn, nâng `faq_availability` lên **96.45%** (đạt chuẩn ≥ 93%). |

### Thí nghiệm

1. **Ngưỡng similarity:**
   - `threshold = 0.80`: `cache_hit_ratio = 0.610`, `wrong_answers = 181` (`other_question: 181`), `cost/turn = 1.015`. Câu hỏi về thời gian hoàn tiền bị trả nhầm thành chính sách hoàn tiền chung.
   - `threshold = 0.90` (chọn): `cache_hit_ratio = 0.592`, `wrong_answers = 0`, `cost/turn = 0.955`.
   - `threshold = 0.97`: `cache_hit_ratio = 0.499` (bị loại vì không đạt ngưỡng tối thiểu 50%), `wrong_answers = 0`, `cost/turn = 1.018`.
2. **Thời gian sống TTL:**
   - Chạy với `ttl_s = 30`: `outdated_answers = 2`, `hit_ratio = 0.592`.
   - Chạy với `ttl_s = 60`: `outdated_answers = 3`, `hit_ratio = 0.652`.
   - Chạy với `ttl_s = 300`: `outdated_answers = 95` (vượt xa trần cho phép 15), `hit_ratio = 0.731`.
   - *Lý do Webhook không giúp được:* Ở t=480s, nhà vận chuyển thay đổi bảng giá âm thầm trong hệ thống mà không phát sự kiện webhook nào. Khi không có tín hiệu invalidate từ nguồn, client không có cách nào biết dữ liệu đã cũ; do đó, TTL ngắn là cơ chế bảo vệ duy nhất để cưỡng chế làm mới dữ liệu.
3. **Stale-if-error:**
   - Khi đặt `max_stale_s: 0`: `stale_served = 0`, `faq_availability` tụt từ 96.45% xuống **85.29%** (hệ thống không thể trả lời câu hỏi FAQ nào trong 120s downtime của provider).
   - Khi đặt `max_stale_s: 300`: Có **192 lượt** câu hỏi FAQ được cứu nguy bằng dữ liệu stale trong đợt sập, duy trì trải nghiệm khách hàng ở mức 96.45%.

### Trả lời câu hỏi
- **Vì sao câu hỏi cá nhân không được cache, kể cả khi cache theo từng user?** Dữ liệu cá nhân (số dư điểm, trạng thái đơn hàng #48213, tiến trình hoàn tiền) biến động liên tục theo thời gian thực (stateful). Nếu cache, người dùng sẽ nhận được thông tin sai lệch (ví dụ: đã chuyển tiền nhưng cache vẫn báo đang xử lý). Nguy hiểm hơn, việc lưu trữ thông tin nhận dạng cá nhân (PII) trong cache có nguy cơ vi phạm bảo mật nghiêm trọng (như rò rỉ dữ liệu xuyên tài khoản khi có lỗi logic băm key hoặc session hijacking).
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?** Alert **không bao giờ** được dựa đơn thuần vào User-Facing Availability. On-call phải thiết lập cảnh báo dựa trên:
  1. Tỷ lệ lỗi trực tiếp của downstream provider (`provider_error_rate` hoặc số lượng mã 5xx/timeout trả về từ LLM).
  2. Số lượng hoặc tỷ lệ response phục vụ từ nguồn cũ (`stale_served_rate`).
  3. `cache_hit_ratio` tăng vọt bất thường hoặc lưu lượng egress gọi ra provider bằng 0.
