# Lab 25B — Report

**Họ tên:** Hà Anh Tuấn     **MSSV**: 2A202602376
**Commit:** `a89e840`
**Điểm tự động:** 84/88 (4 điểm SLO ẩn chờ giảng viên chấm).

Số liệu dùng seed mô phỏng 25, seed client 1 như factory mặc định.

## 1. Kết quả Game Day (naive vs của bạn)

Availability tính theo lượt user thành công; p95 chỉ tính lượt thành công. Amplification là attempts/logical calls, retry_ratio chỉ tính retry cùng provider. `N/A` nghĩa là băng không có mốc restore.

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0 | 4.78 | 1.063 | 0.001 | 2.104 | 0.0 | PASS |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0 | 9.847 | 1.037 | 0.005 | 2.105 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | PASS |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0 | 6.099 | 1.053 | 0.017 | 2.202 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao, gắn với số liệu |
|---|---:|---|
| attempt_timeout_s | 8 | G6 availability 1,0; nâng lên 30 làm giảm còn 0,9488. |
| max_attempts_per_provider | 2 | Giới hạn 1 retry/provider; G2 amplification 1,037, G3 retry_ratio 0,035. |
| min_time_left_s | 1 | Không bắt đầu attempt nếu còn dưới 1 giây; test deadline đạt, G6 không mất lượt với cấu hình 8 giây. |
| backoff_base_s / cap_s | 0,25 / 4 | Full jitter giảm đồng bộ retry; G1 amplification 1,063 và 0 vi phạm retry-after. |
| retry_budget_ratio | 0,1 | G3 retry_ratio 0,035; với breaker không trip, bật budget giữ retry_ratio 0,068 thay vì 0,158. |
| retry_budget_min_per_s / max_tokens | 0,5 / 20 | Cho phép retry sau thời gian ít traffic và burst nhỏ; G2 với breaker cấu hình sai vẫn đạt availability 0,9983 khi giữ budget. |
| quota_cooldown_s | 300 | G5 chỉ gặp 2 lỗi insufficient_quota, tránh gọi liên tục account A đã hết quota. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ hữu hạn và đủ mẫu giúp chuyển tải khi provider xuống sức; G2 amplification 1,037 so với 1,378 khi min_calls quá lớn. |
| breaker.failure_rate_threshold | 0,5 | Cấu hình phát hiện provider hỏng trong G6, availability 1,0 và recovery 0 giây. |
| breaker.slow_call_s / slow_rate_threshold | 5 / 0,6 | Phát hiện cả thành công chậm; G4 p95 5,684 giây so với naive 13,764 giây. |
| breaker.open_duration_s | 10 | Cho provider thời gian hồi phục rồi probe; G2 recovery 0 giây và primary share 1,0. |
| breaker.half_open_max_probes / successes | 2 / 2 | Hạn chế request probe đang bay; test concurrency đạt, G2 quay lại primary với share 1,0. |

| Loại lỗi | Ghi breaker | Retry cùng provider | Failover | Cooldown |
|---|---|---|---|---|
| CLIENT_ERROR | release permit | Không | Không, GiveUp | Không |
| QUOTA_EXHAUSTED | release permit | Không | Ngay | 300 giây |
| RATE_LIMITED | release permit | Không ngay | Ngay | retry_after; thiếu header thì dùng base 0,25 giây |
| OVERLOADED / SERVER_ERROR / TIMEOUT | on_failure | Có nếu còn attempt, breaker cho phép, budget và deadline đủ | Nếu không retry được | Không |

## 3. Ba thí nghiệm

### 3.1. Retry budget là lớp phòng thủ cuối

Cấu hình breaker sai: `min_calls=1000000000`, `max_attempts_per_provider=5`. Bản tắt budget dùng ratio 100, max_tokens 1e9, min_per_s 1e6 theo mẫu; mỗi lượt chạy tạo client mới.

| Băng | Cấu hình | Availability | Amplification | Retry ratio | Recovery (s) |
|---|---|---:|---:|---:|---:|
| G2 | breaker sai, budget bật | 0,9983 | 1,378 | 0,058 | 0 |
| G2 | breaker sai, budget tắt | 0,9754 | 1,580 | 0,378 | 24 |
| G2 | breaker đúng, budget bật | 1,0000 | 1,037 | 0,005 | 0 |
| G2 | breaker đúng, budget tắt | 1,0000 | 1,037 | 0,005 | 0 |
| G3 | breaker sai, budget bật | 1,0000 | 1,136 | 0,068 | 0 |
| G3 | breaker sai, budget tắt | 1,0000 | 1,162 | 0,158 | 0 |
| G3 | breaker đúng, budget bật | 1,0000 | 1,065 | 0,035 | 0 |
| G3 | breaker đúng, budget tắt | 1,0000 | 1,060 | 0,041 | 0 |

Khi breaker không thể trip, budget giảm retry_ratio G2 từ 0,378 xuống 0,058 và recovery từ 24 xuống 0 giây. Với G3, budget giữ retry_ratio dưới 0,10 thay vì 0,158. Khi breaker đúng, khác biệt nhỏ hơn: G2 bằng nhau, G3 tăng từ 0,035 lên 0,041 khi tắt budget; amplification không nhất thiết giảm đơn điệu vì retry và failover thay đổi đường đi của traffic. Vẫn cần cả hai vì breaker quyết định sức khỏe provider còn budget giới hạn tổng retry ngay cả lúc chưa đủ mẫu hoặc cấu hình breaker sai.

### 3.2. Timeout 30 giây

| Băng | Timeout (s) | Availability | p95 lượt thành công (s) |
|---|---:|---:|---:|
| G4 | 8 | 1,0000 | 5,684 |
| G4 | 30 | 1,0000 | 5,608 |
| G6 | 8 | 1,0000 | 6,099 |
| G6 | 30 | 0,9488 | 5,362 |

G4 vẫn được slow-call breaker phát hiện nên tăng timeout không làm p95 tăng trong lần chạy này. G6 có deployment treo: timeout 30 giây bị cắt theo deadline của cả lượt, có thể tiêu hết thời gian trước khi fallback, làm availability giảm 5,12 điểm phần trăm. p95 G6 giảm vì chỉ đo lượt thành công, bỏ những lượt đã timeout; do đó cần nhìn cả availability. Timeout nên dựa trên latency bình thường, ngân sách thời gian còn lại của workflow và thời gian cần cho fallback.

### 3.3. Bỏ lọc tool calling

| Ladder | Availability G6 | Incompatible routes | p95 (s) |
|---|---:|---:|---:|
| Lọc theo supports_tools | 1,0000 | 0 | 6,099 |
| Không lọc | 0,9015 | 235 | 5,702 |

Thí nghiệm dùng subclass chỉ trả nguyên fallback_order trong ladder, giữ nguyên mọi logic khác. Có 235 attempt gửi yêu cầu dùng tool đến model không hỗ trợ tool, khiến availability giảm còn 90,15%. Với người quản lý sản phẩm, đây là việc chuyển yêu cầu hoàn tiền đến một trợ lý không có chức năng thực hiện thao tác đó; model rẻ và còn hoạt động vẫn không hoàn thành được nhiệm vụ. Fallback cần giữ đúng khả năng bắt buộc của tác vụ.

## 4. Workflow hoàn tiền

Số liệu 500 session, seed 25; overpaid dùng đơn vị cents.

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cents) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1,0 | 81 | 128 | 81 | 0 | 1574200 | 1,123 |
| yours | 1,0 | 66 | 0 | 0 | 0 | 0 | 0,303 |

Idempotency key ổn định theo session và step tránh lặp side effect sau mất response hoặc crash, nhưng riêng key không tránh tính lại các LLM call đã hoàn thành. Checkpoint lưu kết quả để resume, nhưng riêng checkpoint vẫn có khoảng trống giữa payment đã chạy và kết quả chưa được ghi; retry lúc này sẽ trả tiền lần nữa nếu không có key. Ghi checkpoint trước payment có thể khiến worker mới tưởng step đã hoàn tất dù tiền chưa được chuyển. Executor thực thi thành công rồi mới lưu checkpoint, retry đúng ServiceError/ServiceTimeout và để WorkerCrash đi lên runner; kết quả chỉ có 20 LLM call dư trên 66 crash.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ burn-rate (phút) |
|---|---|---|---|---:|
| checkout_bug | page | page sau 1 phút | page | 27 |
| provider_outage | page | page ngay | page | 4 |
| deploy_blip | none | page sai | không page | N/A |
| slow_leak | ticket | bỏ sót | ticket, không page | 1337 |
| degraded_evening | page | page sau 214 phút | page | 291 |
| night_spike | none | page sai | không page | N/A |
| small_bump | none | page sai | không page | N/A |

30 ngày có 25.901.755 request, 55.618 lỗi; SLO 99,5% cho phép 129.508,8 lỗi. Availability là 99,7853%, budget đã tiêu 42,95%, còn 57,05%; không đóng băng release vì vẫn còn budget. Small_bump có 12% lỗi trong 20 phút nhưng khi trải trên cửa sổ 60 phút, mức xấp xỉ 4% lỗi cho traffic đều chỉ đạt burn khoảng 8, thấp hơn ngưỡng 14,4; cửa sổ 6 giờ cũng chưa đạt ngưỡng 6. Đo trên dữ liệu thực, burn 1 giờ cao nhất quanh small_bump là 8,2254, dưới ngưỡng 14,4; không rule page nào fire cho blip này, tránh gọi on-call chỉ vì ngưỡng 5 phút tăng ngắn hạn.

## 6. Postmortem — Băng G2 với client naive (blameless)

### Tóm tắt

Alpha mất 75% năng lực GPU từ t=120 đến t=300, khởi phát hàng đợi và timeout. Retry ba tầng khuếch đại thành 34.475 attempt cho 3.481 logical call, giữ áp lực lên provider cả sau khi GPU hồi phục. Trong 2.362 lượt user, chỉ 1.077 lượt thành công; client resilient đạt đủ 2.362 lượt và amplification 1,037 trên cùng băng.

### Ảnh hưởng

Có 1.285 lượt thất bại, tương đương 54,40% traffic toàn băng 600 giây; p95 của lượt thành công là 15,034 giây. Sự cố GPU kéo dài 180 giây, recovery_s là 155 giây sau restore, tương ứng mốc bắt đầu cửa sổ phục hồi t=455. Theo cửa sổ rolling 10 giây, tỷ lệ thành công về 0 tại t=146 và đã trở lại 100% tại t=480; không diễn giải đây là toàn bộ 335 giây đều mất 100% request.

Giả sử tháng 30 ngày có traffic ổn định 4 lượt/giây và chỉ xảy ra sự cố này: 10.368.000 lượt/tháng, budget 0,5% là 51.840 lượt lỗi. Sự cố tiêu 1.285 / 51.840 = 2,48% error budget tháng, còn 97,52%; không vượt toàn bộ budget. Tỷ lệ này dùng lượt user và giả định traffic, tách biệt với bộ dữ liệu SLI 30 ngày ở mục 5.

![Attempts theo provider và tỷ lệ call thành công rolling 10 giây](gameday/G2_overload_storm.png)

### Timeline

Timeline lấy từ kết quả mô phỏng G2 với client naive, seed 25. Attempts/s tính theo từng giây; tỷ lệ thành công tính trên cửa sổ 10 giây.

| t (s) | Sự kiện | Attempts/s alpha | Call success ratio rolling 10 s |
|---:|---|---:|---:|
| 60 | Trước sự cố, hoạt động bình thường | 5 | 100% |
| 120 | Bắt đầu mất 75% GPU; kết quả chưa phản ánh ngay hàng đợi mới | 5 | 100% |
| 180 | Retry storm đã hình thành | 138 | 0% |
| 299 | Ngay trước restore, storm vẫn tiếp diễn | 136 | 0% |
| 300 | GPU phục hồi, request vẫn thất bại | 115 | 0% |
| 400 | Sau restore 100 giây, chưa hồi phục | 51 | 0% |
| 455 | Bắt đầu cửa sổ 30 giây đạt tiêu chí recovery | 13 | 30% |
| 480 | Tỷ lệ rolling 10 giây đã trở lại bình thường | 8 | 100% |

`recovery_s=155` vì cửa sổ 30 giây từ t=455 đến trước t=485 đạt 95,65% call thành công; bảng timeline dùng cửa sổ 10 giây nên tại t=455 chỉ đạt 30%.

### Trigger và root cause

Trigger là giảm GPU của alpha xuống 25% năng lực trong 3 phút. Root cause là cơ chế retry ba tầng app × gateway × SDK, tối đa 36 attempt/call, delay cố định, thiếu jitter, retry budget, breaker và fallback. Khi call timeout, server tiếp tục xử lý request mà client đã bỏ; retry mới vào thêm hàng đợi, tiếp tục tăng latency rồi phát sinh timeout/retry. Sau t=300, dù năng lực phục hồi, backlog và retry vẫn giữ tải cao, tạo vòng lặp metastable failure; Kết quả mô phỏng cho thấy alpha còn nhận 115 attempt/s tại restore trong khi tenant rate limit bình thường chỉ 20 rps.

Dữ liệu mô phỏng không đo trực tiếp queue depth, nên kết luận về vòng lặp hàng đợi dựa trên cơ chế simulator đã mô tả trong README cùng số attempt và tỷ lệ thất bại; không coi số attempt là số request thật sự đã được GPU xử lý. Kết quả ghi nhận 26.298 lỗi rate limit, 3.194 overloaded_error, 2.737 timeout và 28.381 vi phạm retry-after. Thiết kế v1 chưa có giới hạn retry xuyên tầng và kịch bản kiểm thử phục hồi sau sự cố; bài học nằm ở cơ chế bảo vệ và kiểm thử hệ thống.

### Điều gì giúp và điều gì làm tệ hơn

Sáu băng lặp lại được và metric recovery giúp phân biệt GPU đã phục hồi với dịch vụ đã phục hồi. Client resilient kết hợp budget, breaker, cooldown và fallback giảm amplification từ 9,904 xuống 1,037, recovery từ 155 xuống 0 giây, primary share sau restore vẫn 1,0. Delay cố định, retry lỗi rate limit sớm và thiếu đường chuyển tải làm v1 giữ storm; timeout phía client không hủy công việc ở server.

### Action items

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Gom retry về một tầng, giữ full jitter và retry budget | prevent | Nhóm client SDK | CI G2 amplification ≤1,6; G3 retry_ratio ≤0,10; 0 retry lỗi 400. |
| 2 | Giám sát amplification, timeout, backlog nếu provider cung cấp và sức khỏe provider riêng với cache | detect | Nhóm SRE | Inject G2/C1; dashboard thấy retry tăng và breaker mở, stale không che outage; SLO public có 0 false page. |
| 3 | Dùng breaker có slow-call và fallback theo capability | mitigate | Nhóm ShopAgent | G2 recovery ≤30 giây, primary share ≥0,8; G4 p95 ≤10 giây; G6 incompatible_routes=0. |
| 4 | Thêm game day nhiều seed và thí nghiệm breaker sai vào kiểm tra trước release | prevent | Nhóm QA/Release | Chạy grader với seeds 25,26,27; giữ budget khi breaker sai cho G3 retry_ratio ≤0,10. |

## 7. Một điểm yếu còn lại

G2 resilient đạt availability 1,0 nhưng p95 vẫn 9,847 giây, p99 15,586 giây; một số request đang bay phải chờ timeout trước khi fallback. Hướng cải thiện là đo latency riêng từng tác vụ, thử timeout ngắn hơn và giới hạn concurrency trước provider, kiểm tra lại availability/p95/cost trên nhiều seed trước khi áp dụng.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0,8373 | 0,8424 | 0 | 0 | 0 | 0 | 1,386 | FAIL |
| cached | 0,9124 | 0,9645 | 0,652 | 157 | 0 | 3 | 0,915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao |
|---|---:|---|
| similarity_threshold | 0,9 | Không trả nhầm; hit ratio 0,652 cao hơn 0,570 ở ngưỡng 0,97. |
| ttl_s | 60 | Chỉ 3 câu lỗi thời, trong khi TTL 300 gây 95 câu lỗi thời. |
| max_stale_s | 300 | Giữ câu trả lời qua outage; FAQ availability 0,9645 so với 0,8733 khi tắt stale. |

### 8.1. Ngưỡng similarity

| Ngưỡng | Hit ratio | Wrong answers | Cost/turn |
|---|---:|---:|---:|
| 0,80 | 0,636 | 170 | 0,992 |
| 0,90 | 0,652 | 0 | 0,915 |
| 0,97 | 0,570 | 0 | 0,971 |

Ở ngưỡng 0,80, `wrong_answers_by_reason` ghi nhận lỗi `other_question`: các câu hỏi gần nhau về embedding nhưng khác ý bị dùng chung câu trả lời. Ví dụ, chính sách hoàn tiền khác thời gian hoàn tiền; phí ship nội thành khác phí ship đi tỉnh. Ngưỡng 0,90 loại được nhầm lẫn trong băng C1 mà vẫn giữ nhiều hit hơn ngưỡng 0,97.

### 8.2. TTL

| TTL (s) | Hit ratio | Outdated answers | Cost/turn |
|---|---:|---:|---:|
| 30 | 0,592 | 2 | 0,955 |
| 60 | 0,652 | 3 | 0,915 |
| 300 | 0,731 | 95 | 0,863 |

Webhook xử lý thay đổi có thông báo của chính sách hoàn tiền, nhưng thay đổi phí ship không phát webhook nên cache không biết để invalidate. TTL ngắn buộc lấy lại câu trả lời từ provider khi provider còn hoạt động, giới hạn thời gian dùng dữ liệu cũ. TTL 60 cân bằng hit ratio và độ mới; stale-if-error có thể kéo dài tuổi câu trả lời nếu provider đồng thời gặp sự cố.

### 8.3. Stale-if-error

Tắt stale (`max_stale_s: 0`) làm FAQ availability giảm từ 0,9645 xuống 0,8733, hit ratio giảm từ 0,652 xuống 0,561. Bật cửa sổ stale 300 giây phục vụ 157 câu trả lời stale và giúp C1 đạt ngưỡng FAQ availability 0,93.

### 8.4. An toàn và giám sát

Câu hỏi cá nhân không được cache theo allowlist của lab. Cache theo user vẫn có thể trả trạng thái đơn hàng, tiền hoàn hoặc điểm tích lũy đã cũ, và paraphrase gần nhau có thể thuộc các đơn hàng khác nhau của cùng user.

Khi cache giữ availability cao, vẫn cần alert theo tỷ lệ lỗi, timeout, trạng thái breaker của provider và số/tỷ lệ câu trả lời stale. Theo dõi riêng thành công của các call thực sự gọi provider để cache hit không che khuất outage.
