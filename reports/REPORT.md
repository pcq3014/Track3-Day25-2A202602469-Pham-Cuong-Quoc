# Lab 25B — Report

**Họ tên / nhóm:** Phạm Cường Quốc — 2A202602469  **Commit:** xem `git log` của bài nộp (base: `18cc181`)  **Điểm tự động (`make grade`):** 84 / 88
*(4 điểm còn lại là phần "dữ liệu ẩn" của Part E, chỉ chấm trên máy giảng viên; dữ liệu public của Part E: PASS)*

> Mọi số trong report lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`
> (seed 25, trừ khi ghi khác). Các thí nghiệm chạy bằng cách đổi `cfg.client` / `cfg.cache` trong bộ nhớ, không sửa simulator.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0,7194 | 7,157 | 7,738 | 6,738 | 2,163 | 4 | ✗ (availability, retry-after, amplification) |
| G1 | yours | 1,0 | 4,817 | 1,060 | 0,001 | 2,094 | 0 | ✓ |
| G2 | naive | 0,456 | 15,034 | 9,904 | 8,904 | 4,627 | 155 | ✗ (availability, amplification, recovery) |
| G2 | yours | 1,0 | 11,874 | 1,058 | 0,007 | 2,164 | 0 | ✓ |
| G3 | naive | 1,0 | 4,768 | 1,164 | 0,164 | 2,328 | 0 | ✗ (retry_ratio) |
| G3 | yours | 1,0 | 4,621 | 1,058 | 0,030 | 2,140 | 0 | ✓ |
| G4 | naive | 1,0 | 13,764 | 1,004 | 0,004 | 2,008 | 0 | ✗ (p95) |
| G4 | yours | 1,0 | 5,626 | 1,014 | 0,002 | 2,072 | 0 | ✓ |
| G5 | naive | 0,3699 | 5,359 | 17,03 | 16,03 | 2,124 | — | ✗ (client_error_retries, availability) |
| G5 | yours | 0,8179 | 5,482 | 1,003 | 0,002 | 2,504 | — | ✓ |
| G6 | naive | 0,517 | 4,836 | 12,344 | 11,344 | 2,009 | 0 | ✗ (availability) |
| G6 | yours | 1,0 | 6,149 | 1,058 | 0,023 | 2,183 | 0 | ✓ |

Client của tôi đạt 6/6 băng trên seed 25 và cả seed 1, 2, 3, 4. Cùng 2.362 lượt user ở G2, naive gửi 34.475 attempt,
client của tôi chỉ gửi 4.998 attempt.

## 2. Cấu hình client và lý do

Tôi giữ nguyên giá trị khởi đầu vì đã đạt 6/6 băng. Mỗi tham số được kiểm chứng bằng cách đổi **một** giá trị và chạy lại băng nhạy nhất với nó:

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8 | G6 (`alpha-eu` treo): timeout 30 → availability 0,9488 (**FAIL**); 15 → 0,9958; 8 → 1,0. Timeout 4 cắt cả call chậm nhưng hợp lệ ở G4 (alpha +5 s): availability còn 0,9995. 8 s cao hơn độ trễ "chậm nhưng sống" và nhỏ hơn 1/3 deadline 30 s, nên vẫn còn chỗ cho 2 lần failover. |
| max_attempts_per_provider | 2 | Thêm đúng 1 retry trên cùng model (cost 1,0, cùng chất lượng) trước khi sang `beta` (cost 1,4). G3: retry_ratio 0,030, dưới ngưỡng 0,10. Với 3 attempt thì G2 p95 = 10,0 s; với 1 attempt thì G2 p95 = 6,5 s (xem mục 7). |
| backoff_base_s / cap_s | 0,25 / 4 | Full jitter làm lệch thời điểm retry, nên không có "đợt sóng" như naive (delay cố định 0,2 s). Thu hẹp còn 0,05/0,5: G2 ampl 1,040, p95 11,08 s, khác biệt không đáng kể. cap 4 s < timeout nên retry vẫn vừa trong deadline. |
| retry_budget_ratio | 0,1 | Thí nghiệm 1: khi breaker hỏng, budget giữ G3 retry_ratio ở 0,068 (PASS), tắt budget lên 0,158 (**FAIL**). Hạ xuống 0,02 thì G3 retry_ratio = 0,007, vẫn PASS, nhưng mất bớt retry cho lỗi lẻ tẻ. |
| breaker.window_size / min_calls | 20 / 10 | alpha nhận khoảng 8 call/s, nên 10 kết quả ≈ 1,3 s bằng chứng: đủ nhanh để trip nhưng không trip vì 2 lỗi lúc ít traffic. 5/3 cũng PASS ở G3/G5 trên băng này, nhưng dễ trip vì nhiễu. |
| breaker.failure_rate_threshold | 0,5 | G3 có 30% lỗi 500: ở 0,5 breaker thường không trip, retry 1 lần xử lý được (retry_ratio 0,030). Ở 0,9 breaker gần như không bao giờ trip: G3 ampl 1,136, retry_ratio 0,068. Ở 0,2: G3 cost 2,082 (rẻ hơn) nhưng breaker trip vì vài lỗi lẻ khi window nhỏ. |
| breaker.slow_call_s | 5 | G4 không có lỗi nào, chỉ chậm thêm 5 s. Đặt slow_call_s = 10 → G4 p95 13,77 s (**FAIL**, giống naive). Đặt 5 → p95 5,63 s, vì call ≥ 5 s mở breaker và chuyển traffic sang `alpha-eu`/`beta`. |
| breaker.open_duration_s | 10 | Ngắn để quay lại `alpha` nhanh sau sự cố: G2 recovery 0 s, `primary_share_after_restore` 1,0. Đặt 60 → G2 p95 5,75 s (tốt hơn), nhưng sau khi provider hồi phục, traffic còn bị giữ ở beta (đắt hơn 40%) tới 60 s. |
| breaker.half_open_max_probes | 2 | Đặt 50 (gần như không giới hạn): G2 p95 14,6 s, ampl 1,111; G4 p95 9,0 s. Mỗi lần HALF-OPEN, 50 request lao vào provider còn ốm. Đặt 2: G2 ampl 1,058, G4 p95 5,63 s. |

Sơ đồ quyết định của `on_error`:

```
classify(err)
 ├─ CLIENT_ERROR (400/401/422)   breaker.release  → GiveUp("client_error")      (không retry, không failover)
 ├─ QUOTA_EXHAUSTED (429 quota)  breaker.release  → cooldown provider 300 s → failover ngay (delay 0)
 ├─ RATE_LIMITED (429 + r-a)     breaker.release  → cooldown provider tới now+retry_after → failover ngay
 └─ OVERLOADED / SERVER_ERROR / TIMEOUT
                                 breaker.on_failure
                                 tries < max_attempts AND kịp deadline AND budget còn token?
                                   ├─ có  → retry CÙNG provider sau FullJitter(tries)
                                   └─ không → failover sang provider kế tiếp trong ladder
_route: bỏ qua provider đang cooldown / breaker OPEN; ladder lọc theo supports_tools.
Hết ladder → chờ cooldown rate-limit ngắn nhất (≤ 5 s, + jitter) nếu kịp deadline, ngược lại GiveUp.
```

## 3. Ba thí nghiệm

**1. Retry budget là lớp phòng thủ cuối.** Breaker hỏng (`min_calls: 1e9`) và `max_attempts_per_provider: 5`:

| Băng | Cấu hình | amplification | retry_ratio | recovery (s) | availability | PASS? |
|---|---|---:|---:|---:|---:|:---:|
| G2 | (a) có budget | 1,378 | 0,058 | 0 | 0,9983 | ✓ |
| G2 | (b) tắt budget | 1,597 | 0,400 | 32 | 0,967 | ✗ (recovery) |
| G3 | (a) có budget | 1,136 | 0,068 | 0 | 1,0 | ✓ |
| G3 | (b) tắt budget | 1,162 | 0,158 | 0 | 1,0 | ✗ (retry_ratio) |
| G2 | breaker đúng, tắt budget | 1,058 | 0,007 | 0 | 1,0 | ✓ |

Khi breaker hỏng, budget giới hạn retry ở khoảng 10% traffic nên hệ thống vẫn qua được G2. Tắt cả hai thì retry_ratio
G2 tăng gần 7 lần (0,058 → 0,400), và vì lượng retry đè lên alpha đang hồi phục nên G2 mất 32 s mới hồi phục.
Khi breaker hoạt động đúng, tắt budget **không đổi gì** trên G2: breaker đã chặn retry trước khi budget phải can thiệp.
Vẫn nên có cả hai vì chúng bảo vệ ở hai tầng khác nhau. Breaker là trạng thái *theo từng provider*, có thể cấu hình sai
hoặc không trip (lỗi 30% ở G3 nằm dưới ngưỡng 50%). Budget là giới hạn *toàn cục* trên tổng retry, không phụ thuộc
cách phân loại lỗi. Đó là chốt chặn cuối khi những lớp khác hỏng.

**2. `attempt_timeout_s: 30`.**

| Băng | timeout | availability | p95 (s) | PASS? |
|---|---:|---:|---:|:---:|
| G4 | 8 | 1,0 | 5,626 | ✓ |
| G4 | 30 | 1,0 | 5,608 | ✓ |
| G6 | 8 | 1,0 | 6,149 | ✓ |
| G6 | 30 | 0,9488 | 5,502 | ✗ |

G4 gần như không đổi, vì ở đó thứ cứu p95 là **slow-call breaker** (call vẫn trả lời sau khoảng 6 s, dưới cả 8 s lẫn
30 s), không phải timeout. G6 thì khác: `alpha-eu` *treo* chứ không trả lỗi, nên với timeout 30 s một call ngồi chờ
`alpha-eu` hết sạch 30 s deadline của user, không còn thời gian để sang `beta`. Availability rơi từ 1,0 xuống 0,9488.
p95 trông "đẹp hơn" (5,50 s) chỉ vì p95 tính trên lượt *thành công*: những lượt chậm nhất đã thất bại và bị loại khỏi
mẫu (survivorship bias). Timeout đúng nên dựa vào phân phối độ trễ thật của provider (vài lần p99 lúc khỏe, ở đây khoảng
1,2–1,8 s → 8 s) **và** ngân sách deadline: timeout × số bậc fallback cần thử phải nhỏ hơn deadline của user.

**3. Bỏ lọc tool calling khỏi `ladder()`, chạy G6.** `incompatible_routes` = **257**, availability 1,0 → **0,8922**
(**FAIL**). Diễn giải cho product manager: khi nhà cung cấp chính sập, hệ thống đã chuyển 257 yêu cầu cần "bấm nút hành
động" (tra đơn, hoàn tiền) sang một model rẻ **không biết dùng công cụ**. Model đó không thể hoàn tiền, nên hơn 1/10
khách trong sự cố nhận câu trả lời vô dụng hoặc lỗi, dù hệ thống vẫn còn một model đắt hơn (`beta`) làm được việc này.
Rẻ hơn 7 lần chẳng có ý nghĩa nếu không giải quyết được việc của khách.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1,0 | 81 | 128 | 81 | 0 | 15.742,00 (1.574.200 cent) | 1,123 |
| yours | 1,0 | 66 | 0 | 0 | 0 | 0 | 0,303 |

Thí nghiệm bỏ từng thành phần (500 workflow, seed 25):

| Biến thể | completion | duplicate refunds | duplicate emails | lost refunds | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|
| Chỉ idempotency key (không checkpoint) | 1,0 | 0 | 0 | 0 | 1,132 |
| Chỉ checkpoint (key = `uuid4()` mỗi lần) | 1,0 | 14 | 22 | 0 | 0,299 |
| Checkpoint **trước** khi chạy step | 0,914 | 0 | 0 | **11** | — |

- **Chỉ idempotency key là chưa đủ.** Tiền không bị trả trùng vì payment provider de-duplicate theo key. Nhưng mỗi worker
  mới phải chạy lại từ đầu, kể cả các LLM call (`classify_request`, `write_summary`), nên wasted LLM/crash = 1,13, vượt
  ngưỡng 0,6. Ngoài ra ta phụ thuộc hoàn toàn vào việc *mọi* dịch vụ bên ngoài đều hỗ trợ idempotency.
- **Chỉ checkpoint là chưa đủ.** Checkpoint chỉ ghi *sau* khi step trả về. Nếu worker chết sau khi tiền đã đi nhưng trước
  khi ghi checkpoint (`WorkerCrash`), hoặc response bị mất (`ServiceTimeout`), step sẽ chạy lại. Với key ngẫu nhiên,
  provider coi đó là giao dịch mới: 14 lần hoàn tiền trùng, trả thừa 1.737,00.
- **Ghi checkpoint trước khi gọi payment** đánh dấu "đã xong" cho một việc chưa chắc đã xảy ra. Nếu worker chết giữa
  chừng, worker mới thấy checkpoint và **bỏ qua** bước hoàn tiền: 11 khách không bao giờ nhận được tiền, 8,6% workflow
  không hoàn tất. Thứ tự đúng là: chạy step với key ổn định `(session, step)` → **sau đó** mới ghi checkpoint.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
|---|---|---|---|---:|
| checkout_bug | page | page (1 phút) | page ✓ | 27 phút |
| provider_outage | page | page (0 phút) | page ✓ | 4 phút |
| deploy_blip | none | **page (false)** | không ✓ | — |
| slow_leak | ticket | **không phát hiện** | ticket ✓ | 1.337 phút |
| degraded_evening | page | page (214 phút) | page ✓ | 291 phút |
| night_spike | none | **page (false)** | không ✓ | — |
| small_bump | none | **page (false)** | không ✓ | — |

Naive: 7 page, 3 false page, bỏ sót slow_leak. Burn-rate: 3 page, 1 ticket, **0 false page**. Đổi lại, burn-rate phát
hiện chậm hơn với sự cố nhỏ hoặc vừa (checkout_bug 27 phút so với 1 phút), vì nó chờ đủ bằng chứng là budget đang thực
sự bị đốt.

**Error budget 30 ngày:** 55.618 request lỗi / 25.901.755 request (availability 0,997853). Budget cho phép 129.508,8 →
**đã tiêu 42,95%, còn 57,05%**. **Không đóng băng release** (`freeze_releases = False`): budget còn hơn một nửa.

**Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?** Burn rate tức thời là 0,12 / 0,005
= 24×, nên cửa sổ 5 phút vượt 14,4×. Nhưng cửa sổ dài 1 giờ chỉ thấy 0,12 × 20/60 / 0,005 = **8×** < 14,4×, và cửa sổ
6 giờ chỉ thấy 1,3× < 6×. Toàn bộ sự cố tiêu khoảng 0,12 × 20 / (43.200 × 0,005) ≈ **1,1% budget tháng**: không đe dọa
SLO, và đã tự hết trước khi người được page kịp mở laptop. Page nghĩa là "phải hành động ngay, nếu không sẽ vỡ SLO".
Page cho sự cố như vậy chỉ làm on-call mất niềm tin vào alert (alert fatigue). Nếu cần, nó thuộc về ticket hoặc review
trong giờ làm việc.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 naive vs resilient](figures/G2_overload_storm.png)

*Nguồn: `reports/gameday/naive_G2_overload_storm_timeseries.csv` (gộp theo cửa sổ 30 s), `reports/gameday/naive.json`.*

**Tóm tắt.** Lúc t=120 s, deployment `alpha` mất 75% năng lực GPU trong 3 phút. Lớp retry của ShopAgent v1 (3 tầng
chồng nhau, delay cố định, không breaker hay fallback) nhân tải lên `alpha` khoảng 20 lần, đẩy nó vượt cả rate limit của
tenant. Hệ thống ngừng trả lời hoàn toàn trong khoảng 5 phút 18 giây và chỉ hồi phục ở t≈455 s, **155 s sau khi provider
đã khỏe lại**.

**Ảnh hưởng.**
- **1.285 / 2.362 lượt user thất bại** (availability 45,6%).
- Tỷ lệ call thành công bằng 0 từ khoảng t=137 s đến t≈455 s: khoảng **318 s**, so với 180 s sự cố của provider.
- 34.475 attempt cho 3.481 logical call (amplification 9,9). Chi phí mỗi lượt thành công 4,627, gấp 2,1 lần bình thường.
- Error budget: với 4 lượt/s, một tháng có 10.368.000 lượt, SLO 99,5% cho phép 51.840 lượt lỗi. Riêng sự cố này tiêu
  **≈ 2,5% budget tháng** trong 10 phút. Nếu lặp lại mỗi ngày, budget sẽ cạn sau khoảng 40 ngày.

**Timeline** (giây ảo):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–120 | Bình thường | ~7,5 attempt/s lên alpha, 100% call thành công (895/895 trong 0–120 s) |
| 120 | alpha mất 75% năng lực (`capacity_factor 0,25`) | Hàng đợi bắt đầu dài, xuất hiện 529 `overloaded_error` |
| 120–150 | Retry storm bắt đầu | Attempt tăng lên 90,8/s (×12), đỉnh **254 attempt/s ở t=143**; success 51/98 = 52% |
| ~137 | Call thành công cuối cùng | Sau t=136 không còn call nào thành công |
| 150–300 | Sập hoàn toàn | 140–156 attempt/s, vượt xa rate limit 20 rps của tenant; **0/631 call thành công**. Lỗi chính: 26.298 × `429 rate_limit_exceeded` |
| 300 | alpha hồi phục hoàn toàn | Năng lực trở lại 100% |
| 300–360 | Hệ thống **vẫn sập** | Attempt giảm còn 15–23/s nhưng success 0/169: hàng đợi vẫn đầy request mà client đã bỏ |
| 360–420 | Đợt sóng retry thứ hai | Attempt tăng lại 113/s, rồi 78/s; success 0/252 |
| ~455 | Hồi phục (`recovery_s` = 155) | 450–480 s: success 299/332 = 90% |
| 480–600 | Bình thường | ~7–10 attempt/s, success 100% |

**Trigger vs root cause.**
- **Trigger:** `alpha` mất 75% GPU trong 180 s. Đây là sự kiện bên ngoài, không tránh được, và chỉ nên làm chậm hệ thống
  trong 3 phút.
- **Root cause:** lớp gọi LLM của chúng ta khuếch đại tải thay vì giảm tải. Có 3 tầng retry độc lập (app 4 × gateway 3 ×
  SDK 3 = tối đa 36 attempt cho một call), delay cố định 0,2 s không jitter, không tôn trọng `retry-after`, không retry
  budget, không circuit breaker, không fallback sang `alpha-eu`/`beta`. Mọi lỗi đều được "retry tiếp", kể cả 429.

**Vì sao không tự hồi phục sau t=300 (metastable failure).** Có hai vòng lặp tự duy trì:
1. **Hàng đợi đầy việc vô ích.** Khi client hết timeout 8 s và bỏ request, provider *vẫn* xử lý request đó. Hàng đợi của
   alpha đầy request mà không ai còn chờ. Request mới vào sau phải chờ quá 10 s → 529 → lại bị retry. Năng lực đã về
   100% nhưng bị tiêu cho việc vô ích (2.737 `client_timeout`, 3.194 `overloaded_error`).
2. **Retry vượt rate limit.** Tải retry (100–150 attempt/s) gấp 5–7 lần rate limit 20 rps. Provider trả 429 kèm
   `retry_after`, client bỏ qua và retry sau 0,2 s, nên tải vẫn ở trên mức limit, và 429 lại sinh thêm retry. Vòng lặp này
   không cần provider ốm để tồn tại.

Hệ thống chỉ thoát ra khi các logical call cũ lần lượt hết 36 attempt hoặc hết deadline 30 s (655 `retries_exhausted`,
585 `deadline`), tức là khi tải retry *tự cạn* chứ không phải nhờ provider hồi phục.

**Điều gì giúp / điều gì làm tệ hơn.**
- *Giúp:* deadline 30 s của user cuối cùng cũng chặn được vòng đời của mỗi call, nên hệ thống tự thoát sau khoảng 155 s.
  Provider có rate limit và hàng đợi giới hạn 10 s nên không bị sập hẳn.
- *Làm tệ hơn:*
  - Retry nhân 3 tầng.
  - Delay cố định khiến retry đồng bộ thành từng đợt sóng (đỉnh 254/s, rồi đợt hai ở t=360).
  - Coi 429 như lỗi tạm thời và bỏ qua `retry-after`.
  - Không fallback: `alpha-eu` và `beta` có tổng 46 slot rảnh suốt sự cố nhưng nhận 0 attempt.
  - Không có alert nào về amplification: chỉ số này vọt từ 1,0 lên 9,9 mà không ai biết.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Gom retry về **đúng 1 tầng** (`ResilientClient`): tắt retry ở gateway và SDK, chỉ retry lỗi transient, full jitter, tôn trọng `retry-after`, retry budget 10% | prevent | Team LLM Platform | CI chạy G1–G3: amplification ≤ 1,2, `retry_after_violations` = 0, `retry_ratio` ≤ 0,10 (hiện tại: 1,058 / 0 / 0,030) |
| 2 | Circuit breaker sliding-window theo từng deployment + fallback ladder có kiểm tra tool calling (`alpha → alpha-eu → beta → gamma-mini` chỉ cho chat) | mitigate | Team LLM Platform | CI chạy G2: availability ≥ 95%, `recovery_s` ≤ 30 (hiện tại: 1,0 / 0 s); G6: `incompatible_routes` = 0 |
| 3 | Dashboard + alert cho **amplification** (attempt / logical call) và tỷ lệ 429 theo provider: ticket khi > 1,5 trong 5 phút, page khi > 3 kèm burn-rate SLO | detect | SRE on-call rotation | Replay CSV G2 naive: alert phải bắn trong ≤ 60 s sau t=120 (amplification đã vượt 3 ở cửa sổ 120–150 s) |
| 4 | Thêm băng G2 (và biến thể ẩn: lệch thời điểm, cường độ 50–90%) vào Game Day hằng quý và chặn release khi FAIL | prevent | Team ShopAgent + SRE | `make grade` trong pipeline release: Part C = 30/30 |

## 7. Một điểm yếu còn lại

**G2 p95 = 11,9 s** là cao nhất trong 6 băng (vẫn PASS vì G2 không kiểm p95). Nguyên nhân: khi alpha bắt đầu quá tải,
nhiều call phải chờ hết **timeout 8 s** rồi breaker mới đủ `min_calls` để trip. Sau đó, mỗi lần breaker sang HALF-OPEN
(sau 10 s), probe có thể lại mất trọn timeout. Thí nghiệm một tham số cho thấy cái giá:
`attempt_timeout_s: 4` → p95 5,2 s; `open_duration_s: 60` → 5,75 s; `max_attempts_per_provider: 1` → 6,5 s.
Tôi **chưa** áp dụng các giá trị này. Timeout 4 s làm G4 mất 0,05% availability (cắt call chậm mà hợp lệ), và
`open_duration` 60 s giữ traffic ở `beta` (đắt hơn 40%) tới 1 phút sau khi provider đã hồi phục. Băng ẩn lại đổi cường
độ và thời điểm, nên tôi không muốn tinh chỉnh khít theo băng public.

Cách sửa đúng hơn:
1. **Timeout thích nghi** theo phân phối độ trễ gần đây của từng provider (vd. `max(2 s, 3 × p99 của 60 s gần nhất)`,
   kẹp trong khoảng 2–8 s), thay vì một hằng số.
2. **open_duration tăng dần** (10 s → 20 s → 40 s) khi breaker re-trip liên tiếp từ HALF-OPEN, và quay về 10 s sau khi
   CLOSED ổn định.
3. Có thể thêm *hedged request* sang `alpha-eu` cho call đã chờ quá p95, đi kèm retry budget để không tạo thêm tải.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (băng C1, seed 25).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0,8373 | 0,8424 | 0,0 | 0 | 0 | 0 | 1,386 | ✗ (faq_availability, hit ratio) |
| cached (của bạn) | 0,9124 | 0,9645 | 0,652 | 157 | 0 | 3 | 0,915 | ✓ |

Cache giảm chi phí mỗi lượt thành công 34% (1,386 → 0,915) và nâng `faq_availability` từ 84,2% lên 96,5%. Kết quả PASS
trên seed 1–4 và 25 (`wrong_answers` = 0, `outdated_answers` từ 1 đến 8).

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0,90 | Ở 0,80 (starter, TTL 600): 181 `wrong_answers`, toàn bộ là `other_question`. Ở 0,85 đã về 0, nên 0,90 chừa biên an toàn cho băng ẩn. Ở 0,97 hit ratio giảm từ 0,652 xuống 0,581 mà không bớt lỗi nào. |
| ttl_s | 60 | Thay đổi không báo trước (bảng phí ship ở t=480) chỉ có TTL giới hạn được. TTL 600 → 80 `outdated_answers` (ngưỡng ≤ 15); 300 → 95; 60 → 3. Giảm xuống 30 chỉ còn 2 outdated nhưng hit ratio rơi từ 0,652 xuống 0,592 và cost tăng lên 0,955. |
| max_stale_s | 300 | Phải dài hơn cả đợt sập (C1: 120 s, từ t=240 đến t=360) cộng TTL. Đặt 0 → `faq_availability` 0,8733 (**FAIL**); 60 → 0,9233 (**FAIL**); 300 → 0,9645. |
| max_entries | 500 | Giữ nguyên: số câu FAQ khác nhau trong băng nhỏ hơn nhiều, LRU không phải evict. |

**Thí nghiệm:**

1. **Ngưỡng** (TTL 60, stale 300):

   | threshold | hit ratio | wrong_answers | cost/turn |
   |---:|---:|---:|---:|
   | 0,80 | 0,666 | 157 (`other_question`) | 0,965 |
   | **0,90** | 0,652 | **0** | **0,915** |
   | 0,97 | 0,581 | 0 | 0,964 |

   Ở 0,80, các cặp intent "gần mà khác" bị coi là một. Theo `sim/faq.py`, cosine giữa chúng khoảng 0,80–0,86: chính
   sách hoàn tiền ↔ thời gian hoàn tiền ("Chính sách hoàn tiền của shop thế nào?" ↔ "Bao lâu thì nhận được tiền
   hoàn?"), đổi size ↔ đổi màu, phí ship nội thành ↔ đi tỉnh. Cùng chủ đề nên embedding gần nhau, nhưng câu trả lời
   khác hẳn. Ngưỡng 0,90 nằm trên dải đó. Một ngưỡng quá chặt (0,97) không
   an toàn hơn mà chỉ đắt hơn, vì các cách diễn đạt khác của cùng một câu không còn hit.
2. **TTL:** 30 s → 2 outdated, hit 0,592; 60 s → 3 outdated, hit 0,652; 300 s → **95** outdated, hit 0,731. Webhook
   chỉ giúp khi nguồn *báo* là đã đổi (policy/refund ở t=150 được invalidate, nên không có `ignored_invalidation`). Bảng
   phí ship đổi **lặng lẽ** ở t=480, không có sự kiện nào để invalidate. Cache chỉ biết entry đã cũ khi tuổi của nó vượt
   TTL, nên TTL là giới hạn trên duy nhất cho thời gian trả lời sai giá.
3. **Stale-if-error:** `max_stale_s: 0` → `faq_availability` 0,9645 → **0,8733**, stale_served 157 → 0. Trong 120 s
   mọi provider sập, câu FAQ đã có trong cache vẫn được trả lời nhờ bản stale. Đây là bậc 3 ("cached response") của
   fallback ladder.

**Trả lời:**

- **Vì sao không cache câu hỏi cá nhân, kể cả cache theo từng user?**
  - Câu trả lời ("Đơn #48213 đang ở đâu?", "tiền hoàn của tôi") phụ thuộc vào **trạng thái sống** của dữ liệu user và
    đổi theo từng phút: đơn chuyển kho, refund được duyệt. Không có webhook nào báo cho cache, nên trả từ cache là trả
    thông tin sai về *chính tiền và hàng của khách*.
  - Embedding của "đơn #48213 của tôi" và "đơn #48231 của tôi" gần như trùng nhau. Chỉ cần một lỗi scope (thiếu
    `user_id` trong key, một bug khi refactor) là cache lộ dữ liệu cá nhân của người này cho người khác: sự cố quyền
    riêng tư, không chỉ là trả lời sai.
  - Lợi ích lại gần như bằng 0, vì mỗi câu hỏi cá nhân hầu như chỉ được hỏi một lần. Vì vậy chúng không nằm trong
    allowlist, và lúc provider sập chúng thất bại có chủ đích (đó là lý do `faq_availability` không lên được 100%).
- **Dashboard xanh nhờ cache, alert dựa trên số nào?** Alert trên **sức khỏe phía sau cache**, không phải availability
  mà user thấy:
  - (1) **Tỷ lệ stale** = `Serve(stale=True)` / lượt FAQ. Lúc bình thường gần 0, nên > 5% trong 5 phút là page: "đang
    sống nhờ dự trữ, còn tối đa `max_stale_s` trước khi rơi".
  - (2) **Tỷ lệ GiveUp của client bên trong**, tức availability *trước* cache, đưa vào chính burn-rate alert ở Part E.
  - (3) Tỷ lệ lỗi theo từng provider và số breaker đang OPEN.

  Trong C1, availability tổng chỉ giảm nhẹ (0,9124) trong khi 100% provider sập. Nếu chỉ nhìn availability user thấy,
  on-call sẽ biết có sự cố muộn nhất 300 s sau, đúng lúc cache stale hết hạn và mọi thứ rơi cùng lúc.
