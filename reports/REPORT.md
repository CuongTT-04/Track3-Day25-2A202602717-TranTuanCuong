# Lab 25B — Report

**Họ tên / nhóm:** Trần Tuấn Cường (MSV: 2A202602717)  **Commit:** `a3f9ce7`  **Điểm tự động (`make grade`):** 84.0 / 88 (Toàn bộ test public đạt 100%)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Ở băng G4 và G2, hàng đợi server tối đa 10 s, latency alpha bình thường là 1,2 s. Đặt 8,0 s cho phép phát hiện sớm call bị nghẽn để failover sang alpha-eu/beta trước turn deadline 30 s. Ở Thí nghiệm 2, nếu đặt 30 s trên G6 thì availability rớt từ 100% xuống 94,88% do 109 turn chết vì deadline. Với 8,0 s, p95 của G4 đạt 5,562 s (ngưỡng ≤ 10 s). |
| max_attempts_per_provider | 2 | Cho phép thử lại đúng 1 lần trên cùng provider trước khi failover. Ở G2 và G5, NaiveClient retry vô tội vạ khiến amplification vọt lên 9,904 và 17,030. Giới hạn = 2 chặn đứng bão retry; ở G3 retry_ratio giữ ở 0,034 (đạt yêu cầu ≤ 0,10). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter phân tán đồng đều các lần retry trong `[0, min(cap, base * 2^(n-1))]`. Base 0,25 s đủ ngắn để cứu nhanh các lỗi transient (G3 brownout p95 chỉ 4,660 s), cap 4,0 s tránh thundering herd dồn dập vào cùng thời điểm, giữ amplification G2 ở mức 1,041. |
| retry_budget_ratio | 0.1 | Token bucket giới hạn tổng số retry ≤ 10% request đầu vào. Ở G2 và G3, khi breaker bị lỗi (Thí nghiệm 1), budget là chốt chặn cuối giữ retry_ratio ở 0,058 (so với 0,495 khi tắt budget) và không bị trễ hồi phục (recovery 0,0 s vs 25,0 s). |
| breaker.window_size / min_calls | 20 / 10 | Cần tối thiểu 10 kết quả để tránh ngắt mạch oan do 1–2 lỗi lẻ tẻ trong lúc thấp điểm. Cửa sổ trượt 20 kết quả đủ nhạy để tính tỷ lệ lỗi/chậm tức thời trong 3–5 giây khi alpha quá tải ở G2 và G4. |
| breaker.failure_rate_threshold | 0.5 | Khi tỷ lệ lỗi (529, 503, 500, timeout) chạm 50%, provider đã suy thoái nghiêm trọng. Kích hoạt OPEN lập tức chuyển tải sang alpha-eu/beta, bảo vệ alpha khỏi quá tải và giữ availability G2 ở mức 100%. |
| breaker.slow_call_s | 5.0 | Năng lực bình thường của alpha là 1,2 s. Khi latency ≥ 5,0 s là dấu hiệu hàng đợi nghẽn sâu. Ở G4, alpha chậm thêm 5 s (latency ~6,5 s không có mã lỗi HTTP), breaker bắt đúng slow call để mở mạch, đưa p95 G4 về 5,562 s (thay vì 13,764 s như naive). |
| breaker.open_duration_s | 10.0 | Giữ mạch OPEN trong 10 s tương ứng với `max_queue_wait_s` của provider để hàng đợi xả sạch hoàn toàn. Ở G2, nhờ khoảng nghỉ 10 s này mà alpha hồi phục ngay lập tức sau t=300 s (recovery 0,0 s). |
| breaker.half_open_max_probes | 2 | Chỉ cho phép 2 request thăm dò đồng thời ở trạng thái HALF-OPEN. Tránh thundering herd dội ngược vào provider vừa hồi phục. Khi đủ 2 probe thành công và nhanh, breaker mới đóng an toàn (G2 primary share đạt 100%). |

Sơ đồ quyết định của `on_error` (ASCII):

```
                        [ on_error(attempt, err) ]
                                    |
                    Phân loại lỗi qua classify(err)
                                    |
        +---------------------------+---------------------------+
        |                           |                           |
[ CLIENT_ERROR ]         [ RATE_LIMITED / QUOTA ]     [ OVERLOAD / SERVER / TIMEOUT ]
(400, 401, 404, 422)         (429 / 429 quota)             (529, 503, 500, 0 timeout)
        |                           |                           |
  release breaker             release breaker             on_failure(breaker)
        |                           |                           |
  GiveUp("client_error")      Đặt cooldown provider:      Kiểm tra điều kiện retry:
  (Không retry,               - RATE_LIMITED: +retry_after - tries < max_attempts (2)
   Không failover)            - QUOTA: +quota_cooldown_s   - budget.try_acquire_retry() == True
                                    |                      - fits deadline (now+delay+1 <= deadline)
                              Failover ngay lập tức:                    |
                              idx += 1, tries = 0       +---------------+---------------+
                              gọi _route(delay=0)       |                               |
                                                      [ ĐỦ ĐIỀU KIỆN ]          [ KHÔNG ĐỦ ĐIỀU KIỆN ]
                                                        |                               |
                                                   delay = backoff            Failover ngay lập tức:
                                                   Send(same_provider)        idx += 1, tries = 0
                                                   tries += 1                 gọi _route(delay=0)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**
   - **Số liệu thí nghiệm:**
     * Băng G2: Có budget: `avail = 0.9987`, `ampl = 1.380`, `retry_ratio = 0.058`, `recovery = 0.0 s`, `cost/turn = 2.363`. Khi tắt budget: `avail = 0.9666`, `ampl = 1.675`, `retry_ratio = 0.495`, `recovery = 25.0 s`, `cost/turn = 2.473`.
     * Băng G3: Có budget: `avail = 1.0000`, `ampl = 1.136`, `retry_ratio = 0.068`, `cost/turn = 2.271`. Khi tắt budget: `avail = 1.0000`, `ampl = 1.162`, `retry_ratio = 0.159`, `cost/turn = 2.321`.
   - **Giải thích cơ chế:** Khi breaker bị vô hiệu hóa (`min_calls = 10^9`) và `max_attempts_per_provider = 5`, nếu tắt budget trên G2, `retry_ratio` tăng vọt gấp 8,5 lần (từ 0,058 lên 0,495), gây nghẽn kéo dài làm trễ hồi phục 25,0 giây. Token bucket của retry budget giới hạn nghiêm ngặt số lần retry không vượt quá tỷ lệ 10% request đầu vào, ngăn chặn bão retry ngay cả khi breaker hỏng hoàn toàn. Kể cả khi breaker hoạt động đúng, ta vẫn bắt buộc phải có cả hai: Circuit breaker bảo vệ theo từng provider khi tỷ lệ lỗi vượt ngưỡng, còn Retry budget kiểm soát tổng tải retry trên toàn hệ thống trong các khoảng thời gian quá độ trước khi breaker kịp nhảy hoặc khi nhiều client cùng thử lại.

2. **`attempt_timeout_s: 30`.**
   - **Số liệu thí nghiệm:**
     * Băng G6: Ở timeout 8,0 s: `availability = 1.0000`, `p95 = 5.961 s`, `give_up_reasons = {}`. Ở timeout 30,0 s: `availability = 0.9488`, `p95 = 5.315 s`, `give_up_reasons = {'deadline': 109, 'no_provider_available': 13}`.
     * Băng G4: Ở timeout 8,0 s: `availability = 1.0000`, `p95 = 5.562 s`, `give_up_reasons = {}`. Ở timeout 30,0 s: `availability = 1.0000`, `p95 = 5.608 s`, `give_up_reasons = {}`.
   - **Giải thích cơ chế:** Ở G6, khi tăng timeout lên 30 s, `availability` sụt giảm nghiêm trọng từ 100% xuống 94,88% với 109 turn bị bỏ cuộc do hết deadline. Tuy nhiên, `latency_p95_s` lại giảm nghịch lý từ 5,961 s xuống 5,315 s. Đây là minh chứng điển hình của **Survivorship Bias**: metric p95 chỉ tính trên các lượt *thành công*; những request bị treo 30 s trên provider gặp sự cố (`alpha-eu`) đã chết vì chạm turn deadline (30 s) và bị loại khỏi mẫu p95, tạo cảm giác sai lầm là hệ thống "chạy nhanh hơn". G4 gần như không đổi vì ở G4 alpha chỉ chậm thêm 5 s (latency ~6,5 s < 8 s), request vẫn kịp hoàn thành trước 8 s mà không rơi vào tình trạng treo cứng 30 s. Timeout "đúng" phải được tính toán dựa trên **turn deadline của user chia cho số lượt failover cần thiết** (ví dụ deadline 30 s cho 2 bước thì mỗi bước có ~15 s budget; đặt timeout 8 s để nếu lần 1 lỗi vẫn còn dư 7 s failover sang provider khác).

3. **Bỏ lọc tool calling khỏi `ladder()`.**
   - **Số liệu thí nghiệm:**
     * G6 có lọc tool: `availability = 1.0000`, `incompatible_routes = 0`, `give_up_reasons = {}`.
     * G6 bỏ lọc tool: `availability = 0.8918`, `incompatible_routes = 258`, `give_up_reasons = {'incompatible_model_output': 258}`, `errors = {'api_error': 11, 'service_unavailable': 47, 'client_timeout': 78, 'rate_limit_exceeded': 68}`.
   - **Giải thích góc nhìn Product Manager:**
     "Trong đợt sự cố provider A, việc bỏ kiểm tra tính tương thích khiến 258 yêu cầu hoàn tiền của khách hàng bị chuyển hướng sai sang model giá rẻ `gamma-mini`. Do model này hoàn toàn không hỗ trợ gọi tool (tool calling), nó không thể kích hoạt hành động tra cứu đơn hàng hay hoàn tiền thực tế, dẫn đến việc gần 11% khách hàng (availability giảm từ 100% xuống 89,18%) nhận thông báo thất bại hoặc phản hồi vô nghĩa. Điều này trực tiếp gây tổn thất uy tín dịch vụ CSKH, làm tăng khối lượng khiếu nại thủ công và làm hỏng trải nghiệm người dùng cốt lõi."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

**Trả lời:**
- **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key bảo đảm tính duy nhất tại các dịch vụ bên ngoài (cổng thanh toán, dịch vụ email) để không bị trừ tiền hay gửi thư 2 lần khi gọi lại. Nhưng nếu không có checkpoint trong `DurableStore`, khi worker bị crash, worker mới sẽ phải chạy lại từ đầu toàn bộ các bước tính toán trước đó (như `classify_request`), gây lãng phí chi phí gọi LLM không cần thiết (NaiveClient lãng phí 1,123 LLM call/crash, trong khi bản có checkpoint chỉ tốn 0,303).
- **Vì sao chỉ checkpoint cũng chưa đủ?** Nếu worker thực hiện gọi thanh toán thành công nhưng bị crash hoặc timeout mạng ngay *trước* khi kịp lưu checkpoint, worker thay thế khởi động lại kiểm tra thấy chưa có checkpoint sẽ gọi lại bước thanh toán. Nếu không có idempotency key, cổng thanh toán sẽ tạo một giao dịch hoàn tiền mới, dẫn đến hoàn tiền trùng và thất thoát tài chính (NaiveClient bị trùng 128 lần, thất thoát 1.496.700 cents = $14.967).
- **Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment?** Sẽ dẫn đến lỗi nghiêm trọng **Lost Refund (mất tiền hoàn)**: nếu ghi checkpoint "đã hoàn thành" trước khi gọi payment mà quá trình gọi payment sau đó bị lỗi hoặc worker crash giữa chừng, worker mới khởi động lại thấy checkpoint đã tồn tại nên bỏ qua bước hoàn tiền. Kết quả là hệ thống báo thành công nhưng tiền của khách hàng không bao giờ được hoàn (`lost_refunds > 0`). Quy tắc bắt buộc: thực thi hành động với Idempotency Key trước, thành công rồi mới ghi Checkpoint sau!

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (False Page) | none | - |
| slow_leak | ticket | none (Bỏ lọt) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (False Page) | none | - |
| small_bump | none | page (False Page) | none | - |

**Error budget 30 ngày:**
- Tổng requests: 25.901.755.
- Lỗi cho phép (`allowed_bad`): 129.508,8.
- Lỗi thực tế (`bad`): 55.618.
- Availability đạt: 0,997853 (99,79% so với SLO 99,50%).
- Ngân sách lỗi đã tiêu: **42,95%** (`budget_consumed = 0.4295`), còn lại: **57,05%** (`budget_remaining = 0.5705`).
- **Quyết định đóng băng release:** **KHÔNG** (`freeze_releases = False`) vì ngân sách lỗi còn dư 57,05% (> 0%).

**Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
Vì 20 phút ở 12% lỗi chỉ tiêu tốn: `(20 / 43.200) * 12% = 0,0055%` trên tổng lưu lượng cả tháng, tương đương `0,0055% / 0,5% = 1,1%` ngân sách lỗi của 30 ngày. Mức tiêu hao này quá nhỏ để đe dọa cam kết SLO tháng và sự cố tự chấm dứt nhanh chóng. Nếu page đánh thức kỹ sư on-call cho các bump nhỏ như vậy sẽ gây "alert fatigue" (chai lì cảnh báo), làm giảm khả năng tập trung xử lý các sự cố thực sự nghiêm trọng. Sự cố này chỉ cần ghi nhận log hoặc gửi ticket tự động nếu tiếp diễn.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào giây ảo thứ 120 của kịch bản G2, provider chính `alpha` bị suy giảm đột ngột 75% năng lực GPU trong 3 phút, làm tắc nghẽn hàng đợi và phát sinh lỗi 529 quá tải. Client Naive với cơ chế 3 tầng retry xếp chồng không jitter, không circuit breaker và không retry budget đã khuếch đại lưu lượng lên tới hơn 250 attempts/s, biến một sự cố cục bộ thành thảm họa sập toàn diện kéo dài 333 giây (từ t=120 đến t=453). Toàn bộ hệ thống mất thêm 155 giây sau khi provider đã hoàn toàn hồi phục (t=300) mới giải phóng được hàng đợi và đưa tỷ lệ phục vụ thành công trở lại bình thường.

**Ảnh hưởng:**
- Tổng số user turn thất bại: 1.285 lượt thất bại trên tổng số 2.362 lượt khách hàng (availability chỉ đạt 45,60%, 1.077 good turns).
- Thời gian gián đoạn hoàn toàn (tỷ lệ thành công = 0%): kéo dài 316 giây (từ t=137 đến t=452).
- Mức độ thấu chi error budget: Trong kịch bản này có 1.285 turn lỗi trên 2.362 turn (tỷ lệ lỗi 54,4%). Nếu ngoại suy trên quy mô 30 ngày (ngân sách lỗi cho phép là 129.508 request lỗi), sự cố này tiêu tốn tốc độ burn rate khổng lồ gấp hơn 108 lần mức cho phép, đủ để đốt sạch 100% error budget của toàn bộ tháng chỉ trong chưa đầy 4 giờ gián đoạn.

**Timeline** (giây ảo — lấy từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số từ CSV) |
|---:|---|---|
| 0–119 | Hệ thống hoạt động bình thường | Traffic ổn định 7–14 attempts/s trên `alpha`, `calls_ok = 7–15`, `calls_done = calls_ok`, tỷ lệ thành công 100%. |
| 120 | `alpha` mất 75% năng lực xử lý (GPU) | Năng lực phục vụ giảm từ 18 server xuống ~4 server, thời gian hàng đợi vượt 10 s bắt đầu xuất hiện. |
| 121–136 | Hàng đợi đầy, bão retry bắt đầu bùng phát | `calls_ok` giảm từ 7 xuống 2–3/s; attempts trên alpha tăng vọt từ 15 (t=133) lên 24 (t=134), 53 (t=136), và 176 (t=139). |
| 137 | Hệ thống sập hoàn toàn (`calls_ok = 0`) | `calls_ok = 0`, attempts đạt đỉnh 254 attempts/s tại t=143–144 (khuếch đại gấp >15 lần năng lực phục vụ của `alpha`). |
| 300 | `alpha` hồi phục hoàn toàn năng lực phần cứng | Năng lực phục vụ của `alpha` trở lại 18 server, nhưng `calls_ok` vẫn bằng 0, attempts duy trì ở mức 115 attempts/s. |
| 301–452 | Hệ thống vẫn chưa hồi phục — duy trì sập hoàn toàn | `calls_ok = 0` liên tục trong 152 giây sau khi provider đã sửa xong; attempts vẫn dao động 50–176 attempts/s (t=370–373 đạt 176 attempts/s). |
| 453 | Hàng đợi bắt đầu xả cạn, dịch vụ chớm hồi phục | `calls_ok` lần đầu tiên > 0 (`calls_ok = 4`, `calls_done = 8`, attempts = 7). |
| 456–478 | Hệ thống hồi phục hoàn toàn | `calls_ok` ổn định ở mức 8–16/s, attempts quay về 4–16 attempts/s. Độ trễ hồi phục `recovery_s = 155.0 s`. |

**Trigger vs root cause:**
- **Trigger:** Sự cố phần cứng/hạ tầng tại provider `alpha` khiến hệ thống mất 75% năng lực GPU trong 180 giây (t=120 đến t=300).
- **Root cause:** Khiếm khuyết cấu trúc trong client:
  1. Ba tầng retry xếp chồng độc lập (app × gateway × SDK = 36 attempt/call) không có điểm dừng;
  2. Retry với khoảng trễ cố định, không có Full Jitter gây cộng hưởng pha (thundering herd);
  3. Thiếu Circuit Breaker để ngắt mạch sang `alpha-eu` hay `beta`;
  4. Thiếu Retry Budget để khống chế trần số lượng retry trên toàn hệ thống;
  5. Cơ chế xử lý hàng đợi FIFO của server vẫn tiếp tục tính toán các request mà client đã timeout/từ bỏ.
- **Vòng lặp metastable (Metastable failure — vì sao không tự hồi phục sau t=300):**
  Khi provider khôi phục năng lực ở t=300 (phục vụ được ~15–20 rps), hàng đợi của server vẫn đang ứ nghẽn hàng trăm request cũ mà client đã timeout bỏ cuộc từ trước (dead work). Server tiếp tục tiêu tốn năng lượng quý giá để xử lý các request vô ích này. Trong khi đó, hàng nghìn request mới đến gặp hàng đợi nghẽn lại bị timeout và tiếp tục kích hoạt chu kỳ 36 lần retry. Lượng tải retry liên tục đổ vào ở mức 50–176 attempts/s (vượt xa trần xử lý 20 rps của alpha). Năng lực phục vụ bị đè bẹp bởi chính lưu lượng retry do hệ thống tự sinh ra, tạo thành một vòng lặp tự duy trì (self-sustaining loop) khiến hệ thống bị kẹt trong trạng thái sập bền vững suốt 155 giây.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Tiếp tục gửi 100% request vào provider đang nghẽn; retry không giới hạn; retry không giãn cách thời gian ngẫu nhiên (jitter); server cố gắng xử lý các request đã quá hạn client.
- *Giúp ích:* Circuit Breaker sliding window phát hiện lỗi/chậm để ngắt mạch OPEN và chuyển hướng sang deployment dự phòng (`alpha-eu`, `beta`); Retry Budget khống chế trần retry ≤ 10%; Full Jitter xóa bỏ đỉnh tải cộng hưởng; Half-open giới hạn probe requests tránh thundering herd khi provider vừa hồi phục.

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient` tích hợp Circuit Breaker sliding window và Retry Budget (ratio 0.1). | Prevent | Tech Lead (CuongTT) | Chạy kịch bản G2 trong CI: `amplification ≤ 1.2`, `recovery_s ≤ 30 s`, `availability ≥ 95%`. |
| 2 | Cấu hình Deadline Propagation (truyền timeout header tới server) để provider hủy ngay các request trong hàng đợi đã bị client hủy/timeout. | Mitigate | Backend Platform Team | Kiểm chứng trong môi trường staging: số lượng server execution cho request quá hạn bằng 0 khi client ngắt kết nối. |
| 3 | Triển khai Burn-rate Alerting đa cửa sổ (14.4x 1h/5m cho Page khẩn cấp; 1.0x 3d/6h cho Ticket cảnh báo). | Detect | SRE Team | Kiểm chứng trên dữ liệu SLI 30 ngày: 0 false page, phát hiện các sự cố outage lớn trong vòng ≤ 5 phút. |

## 7. Một điểm yếu còn lại

**Tình huống client xử lý kém:**
Khi cả hai deployment chính của họ model A (`alpha` và `alpha-eu`) cùng gặp sự cố lớn (như trong băng G6), client buộc phải failover hàng loạt sang `beta` (model B-large). Tình huống này bộc lộ hai điểm yếu lớn:
1. **Bùng nổ chi phí (Cost spike):** `beta` có chi phí 1,4 / call (đắt hơn 40% so với alpha là 1,0). Khi toàn bộ traffic dồn sang `beta`, chi phí vận hành tăng vọt (chi phí turn ở G6 tăng lên 2,189 so với 2,072 ở G4).
2. **Nguy cơ quá tải thứ cấp (Secondary rate-limit bottleneck):** Tenant rate limit của `beta` chỉ có 16 rps (thấp hơn 20 rps của `alpha`). Khi lưu lượng tổng là 4 turns/s × 2 calls = 8 rps bình thường nhưng khi có đột biến tải hoặc retry, `beta` rất dễ bị 429 hàng loạt.

**Cách khắc phục:**
1. **Phân tầng hạ cấp thông minh (Graceful degradation with tiering):** Tách biệt loại tác vụ; các lượt chat thông thường không cần suy luận phức tạp có thể định tuyến sang `gamma-mini` (chi phí chỉ 0,2, năng lực 30 rps, 40 slot) thay vì dồn hết sang `beta`, dành riêng `beta` cho các tác vụ bắt buộc tool calling.
2. **Client-side Concurrency Limiter (Adaptive Limiter):** Bổ sung bộ điều tiết tải chủ động ở client theo từng provider dựa trên thuật toán TCP Vegas / CoDel để giảm tốc độ gửi request sang `beta` về dưới ngưỡng 16 rps trước khi bị provider trả về lỗi 429.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Khi đặt 0,80 ở cấu hình starter, có tới 175 câu trả lời bị sai (`wrong_answers = 175`, toàn bộ do nhầm sang câu hỏi khác `other_question: 175`). Đặt 0,92 loại bỏ hoàn toàn câu trả lời sai (`wrong_answers = 0`) mà vẫn đạt tỷ lệ trúng cache cao 65,2% (vượt tiêu chí ≥ 50%). Nếu tăng lên 0,97 thì hit ratio giảm xuống 57,9% và cost/turn tăng lên 0,965. |
| ttl_s | 60 | Khi phí vận chuyển thay đổi âm thầm tại t=480 s mà không có webhook, chỉ có TTL mới giúp vô hiệu hóa bản ghi cũ. Nếu đặt TTL = 300 s, có tới 95 câu trả lời lỗi thời bị phục vụ (`outdated_answers = 95`). Đặt `ttl_s = 60 s` khống chế `outdated_answers` chỉ còn 3 (đáp ứng tiêu chuẩn ≤ 15) mà vẫn giữ hit ratio 65,2%. |
| max_stale_s | 300 | Sự cố toàn bộ provider sập ở băng C1 kéo dài 120 s (t=240 đến t=360 s). Nếu đặt `max_stale_s = 0`, client không được dùng cache cũ khi provider sập, `stale_served = 0` và `faq_availability` sụt xuống 87,33% (FAIL). Đặt 300 s cho phép phục vụ 157 lượt trả lời cũ (`stale_served = 157`), kéo `faq_availability` lên 96,45% (vượt tiêu chuẩn ≥ 93%). |

**Thí nghiệm (mỗi cái 2–4 câu + số):**

1. **Ngưỡng.**
   - *Số liệu:*
     * `similarity_threshold = 0.80`: `hit = 0.659`, `wrong_answers = 175` (`{'other_question': 175}`), `outdated = 1`, `cost/turn = 0.978`, `avail = 0.8483`.
     * `similarity_threshold = 0.92`: `hit = 0.652`, `wrong_answers = 0` (`{}`), `outdated = 3`, `cost/turn = 0.915`, `avail = 0.9124`.
     * `similarity_threshold = 0.97`: `hit = 0.579`, `wrong_answers = 0` (`{}`), `outdated = 4`, `cost/turn = 0.965`, `avail = 0.9117`.
   - *Phân tích:* Ở ngưỡng 0,80, các câu hỏi có từ khóa tương tự nhưng mang ý nghĩa khác biệt hoàn toàn (các cặp "gần mà khác", ví dụ: câu hỏi về "Thời gian nhận tiền hoàn" bị nhầm sang câu hỏi về "Chính sách và điều kiện hoàn tiền") bị nhận diện nhầm thành cache hit, dẫn đến 175 câu trả lời sai cho khách hàng. Ngưỡng 0,92 là điểm cân bằng tối ưu giữa việc triệt tiêu câu trả lời sai (0 câu) và duy trì hit ratio cao (65,2%).

2. **TTL.**
   - *Số liệu:*
     * `ttl_s = 30 s`: `outdated = 2`, `hit = 0.592`, `faq_avail = 0.9645`, `cost/turn = 0.955`.
     * `ttl_s = 60 s`: `outdated = 3`, `hit = 0.652`, `faq_avail = 0.9645`, `cost/turn = 0.915`.
     * `ttl_s = 300 s`: `outdated = 95`, `hit = 0.731`, `faq_avail = 0.9640`, `cost/turn = 0.863`.
   - *Phân tích:* Webhook chỉ phát huy tác dụng khi nguồn dữ liệu chủ động gửi thông báo cập nhật (như sự kiện đổi chính sách hoàn tiền ở t=150 s). Khi đơn vị vận chuyển âm thầm thay đổi bảng phí ở t=480 s mà không có thông báo (silent update), hệ thống hoàn toàn không nhận được webhook nào, khiến cơ chế invalidation theo sự kiện bất lực; do đó, TTL là cơ chế phòng vệ duy nhất ép các bản ghi cũ phải hết hạn và buộc client truy vấn lại dữ liệu mới từ provider.

3. **Stale-if-error.**
   - *Số liệu:*
     * `max_stale_s = 0 s`: `faq_availability = 0.8733`, `stale_served = 0`, `hit = 0.561`, `cost/turn = 0.975`.
     * `max_stale_s = 300 s`: `faq_availability = 0.9645`, `stale_served = 157`, `hit = 0.652`, `cost/turn = 0.915`.
   - *Phân tích:* Khi `max_stale_s = 0`, cơ chế stale-if-error bị khóa hoàn toàn; trong suốt 120 giây toàn bộ provider sập, mọi câu hỏi FAQ hết hạn TTL đều bị từ chối phục vụ, khiến `faq_availability` rớt thảm hại xuống 87,33% (dưới ngưỡng yêu cầu 93%). Khi mở `max_stale_s = 300 s`, có 157 lượt khách hàng được phục vụ bằng câu trả lời cũ còn dùng được, nâng tỷ lệ sẵn sàng của FAQ lên 96,45% và duy trì hoạt động ổn định của dịch vụ.

**Trả lời:**

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Nguy cơ rò rỉ dữ liệu định danh và bảo mật (Data Privacy & PII Leakage):* Ngay cả khi định danh theo từng user, bất kỳ sai sót nào trong việc sinh khóa cache, session hijacking hoặc phân quyền đều có thể dẫn đến việc thông tin nhạy cảm của khách hàng (địa chỉ giao hàng, số điện thoại, giá trị đơn) bị phơi bày cho người khác, vi phạm nghiêm trọng quy định bảo mật dữ liệu.
  2. *Bản chất biến động thời gian thực của dữ liệu (Real-time Dynamic State):* Trạng thái đơn hàng thay đổi liên tục theo từng bước xử lý của kho vận (đang đóng gói -> đã xuất kho -> đang giao). Câu trả lời lưu trong cache sẽ bị lỗi thời ngay lập tức, khiến khách hàng nhận thông tin sai lệch về vị trí đơn hàng thực tế của họ.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Không được phép chỉ nhìn vào metric đầu ra `external_availability` (vẫn xanh do người dùng vẫn nhận được câu trả lời từ cache). Đội ngũ on-call bắt buộc phải thiết lập cảnh báo dựa trên các chỉ số nội tại sau:
  1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio` hoặc `stale_served_count`):** Cảnh báo ngay lập tức khi số lượng câu trả lời phục vụ ở trạng thái stale (`stale=True`) tăng vọt, báo hiệu rằng hệ thống đang phải dùng phao cứu sinh stale-if-error do tầng backend không thể phản hồi.
  2. **Tỷ lệ lỗi của nhà cung cấp hạ tầng (`provider_error_rate` / `circuit_breaker_open_state`):** Giám sát trực tiếp tỷ lệ lỗi 5xx, 429 và trạng thái ngắt mạch của Circuit Breaker trên các deployment (`alpha`, `beta`). Khi breaker chuyển sang OPEN, hệ thống phải kích hoạt Page khẩn cấp cho on-call.
  3. **Tỷ lệ lỗi trên các truy vấn không trúng cache (`cache_miss_error_rate`):** Giám sát tỷ lệ thành công của các request bị cache miss. Khi provider sập, 100% request miss cache sẽ thất bại, đây là tín hiệu rõ ràng nhất phản ánh sự cố tầng backend đang bị che giấu bởi cache hit.
