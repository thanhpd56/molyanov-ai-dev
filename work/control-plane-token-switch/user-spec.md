---
# Creation date (YYYY-MM-DD)
created: 2026-10-05

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: control-plane-token-switch

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.
>
> **Lưu ý thực thi đặc biệt:** feature này có 2 phần nằm ở 2 repo khác nhau — xem mục "Expected
> Behavior" Phần A và B. Phần A nằm trong repo hiện tại (`molyanov-ai-dev`). Phần B nằm trong repo
> riêng `github.com/thanhpd56/control-plane` — phải `git clone` repo đó sang 1 thư mục riêng để
> thực thi Phần B, không thực thi trong thư mục của repo này (nếu thư mục đó đã tồn tại từ trước
> trên máy đang chạy, dùng lại, đừng clone đè).

## What We Are Building

Thêm cho `control-plane` 1 pool token OAuth Claude dùng chung (lưu Cloudflare KV, mỗi token 1
alias), lấy cảm hứng từ cơ chế `add-token`/`switch` của `claude-swap` (github.com/realiti4/
claude-swap) nhưng áp dụng vào secret `CLAUDE_CODE_OAUTH_TOKEN` của 1 repo GitHub thay vì file
credentials cục bộ. 3 slash command Slack mới quản lý pool (`/add-token`, `/remove-token`,
`/switch-token`) dùng được bởi bất kỳ ai trong channel đã link với 1 repo. `online-pipeline` (3 nơi
gọi `claude -p`: `userspec-turn`, `implement`, `knowledge-init`) tự nhận diện lỗi rate-limit của
Claude, tự gọi `control-plane` đổi sang 1 token khác trong pool rồi thử lại — không cần người can
thiệp. Repo mới bootstrap qua `/new-project` cũng nhận token từ pool ngay từ đầu, thay vì luôn dùng
đúng 1 token cố định như hiện tại.

## Why

Nhiều repo dùng chung `online-pipeline` để chạy CI, và hiện tại mọi repo mới đều được gán đúng 1
`CLAUDE_CODE_OAUTH_TOKEN` cố định (copy từ secret của `control-plane` lúc bootstrap). Khi nhiều
repo/feature cùng hoạt động, account Claude đó cạn hạn mức 5 giờ hoặc 7 ngày — đã xảy ra thật
trong production (xem log thật ở Acceptance Criteria/Risks), khiến cả 3 lần retry của `implement`
fail liên tiếp, không ai phát hiện ra lý do thật (rate-limit) nếu không đọc kỹ log Actions. Đây là
đúng vấn đề mà `claude-swap` giải quyết cho máy cá nhân (nhiều account, tự chuyển account khi gần
hết hạn mức) — áp dụng tương tự ở đây cho secret GitHub Actions của nhiều repo dùng chung.

## Expected Behavior

### Phần A — trong `molyanov-ai-dev` (3 workflow `online-pipeline`)

1. **Nhận diện rate-limit**: sau mỗi lần gọi `claude -p`, workflow kiểm tra output có khớp 1 trong
   2 mẫu lỗi thật đã xác nhận không:
   - `"You've hit your session limit · resets {time} ({tz})"` (giới hạn 5 giờ)
   - `"You've hit your weekly limit · resets {time} ({tz})"` (giới hạn 7 ngày)

   `{tz}` là tên timezone dạng IANA (đã thấy `UTC` và `Asia/Saigon`). Không khớp cả 2 mẫu → coi là
   lỗi kỹ thuật thường, giữ nguyên hành vi retry-cùng-token hiện tại (auto-switch chỉ cộng thêm,
   không bao giờ làm giảm độ tin cậy hiện có).

2. **Gọi `control-plane` để đổi token**: khi khớp mẫu rate-limit, gửi `POST /switch-token/auto` tới
   `$SLACK_WORKER_URL`, xác thực bằng `X-Relay-Token: $SLACK_RELAY_TOKEN` (dùng lại đúng secret
   `/relay` đang dùng, không phát sinh secret mới), body `{owner, repo, rateLimitResetText}` với
   `rateLimitResetText` là đúng đoạn `"resets {time} ({tz})"` vừa khớp được (để Worker tự parse và
   tự biết alias hiện tại của repo qua KV, không cần client gửi alias hiện tại).

   - Thành công (`{ok:true, alias, token}`): export `token` trả về làm giá trị `CLAUDE_CODE_OAUTH_TOKEN`
     cho **lượt thử tiếp theo** (biến môi trường job cố định suốt đời job, không tự refresh — phải
     dùng giá trị Worker trả về trực tiếp, không đọc lại secret gốc), rồi tiếp tục retry như thường.
   - Pool hết token khả dụng (`{ok:false, error:"no_alias_available"}`): **dừng lại ngay, không
     thử thêm** (không tốn hết 3 lượt vô ích vì token cũ CHẮC CHẮN vẫn đang rate-limit) — báo lỗi rõ
     ràng qua `gh issue comment` + relay Slack: "Mọi token trong pool đều đang bị rate-limit, dừng
     tự động — thêm token mới qua `/add-token` hoặc đợi reset", rồi exit khác biệt với "job failed
     after 3 attempts" generic.
   - Gọi Worker thất bại (lỗi mạng/timeout/5xx — khác với phản hồi `no_alias_available`): coi như
     không khớp mẫu rate-limit, rơi về hành vi retry-cùng-token hiện tại cho lượt đó.

3. **Lượt retry do auto-switch tính vào đúng 3 lần `ATTEMPTS` hiện có** của `implement`/
   `knowledge-init` (và loop mới của `userspec-turn`, xem mục 4) — không thêm lượt ngoài giới hạn.
   Tối đa 3 lần gọi `claude -p` thật trong 1 run, mỗi lần có thể trên 1 token khác nhau.

4. **`online-pipeline-userspec.yml`'s "Run userspec-turn" step hiện KHÔNG có retry loop nào** (chỉ
   gọi `claude -p` đúng 1 lần). Thêm 1 loop 3 lần đầy đủ, đúng khuôn với `implement`/
   `knowledge-init` (bao gồm cả nhận diện rate-limit ở mục 1-2) — vừa để hỗ trợ auto-switch ở đây,
   vừa tiện sửa luôn việc hiện tại bất kỳ lỗi kỹ thuật nào cũng làm fail cả run, không retry.

5. **`implement.yml` khi rate-limit xảy ra giữa lúc code đang được implement** (log thật cho thấy
   agent đang "Removing web/vite.config.ts" thì bị limit): lượt retry sau khi đổi token xử lý
   **giống hệt** mọi lỗi kỹ thuật khác hiện tại của loop này — reset về `$LAST_GOOD` (bỏ phần việc
   dở của lượt đó) rồi thử lại từ đầu với token mới. Không có cơ chế "tiếp tục giữa chừng" mới.

6. **`SKILL.md` của `online-pipeline`** ghi lại hành vi mới này cho cả 3 stage.

### Phần B — trong repo riêng `control-plane` (cần clone riêng để thực thi)

7. **Pool lưu ở Cloudflare KV** (namespace `MAPPINGS` đã có sẵn, không cần binding mới):
   - `token-pool:{alias}` → `{token, addedAt}`.
   - `repo-token:{owner}/{repo}` → `{alias}` (alias hiện tại của repo đó).
   - `token-cooldown:{alias}` → ISO timestamp (vắng/đã qua = khả dụng).

8. **3 slash command mới**, dùng chung mức tin tưởng hiện có (ai trong channel đã link cũng gọi
   được, không thêm tầng quyền mới — đúng mô hình `/new-feature` hiện tại):
   - **`/add-token <alias> <token>`**: tách theo khoảng trắng đầu tiên (`alias` = từ đầu, `token` =
     phần còn lại, trim). `alias` chỉ cho chữ/số/`-`/`_` (tái dùng tinh thần charset guard của
     `isValidRepoName` trong `lib.js`, không phát sinh guard mới từ đầu). Thiếu phần token (không
     có khoảng trắng) → báo lỗi cú pháp, không lưu. `alias` đã tồn tại trong pool → **báo lỗi,
     không ghi đè** (phải `/remove-token` trước) — tránh 1 người vô tình đổi token đang được nhiều
     repo khác dùng mà không ai biết. Giá trị token không bao giờ echo lại trong bất kỳ reply nào.
   - **`/remove-token <alias>`**: xoá khỏi pool. Nếu 1 repo đang dùng đúng alias đó: cho xoá bình
     thường — secret `CLAUDE_CODE_OAUTH_TOKEN` của repo đó giữ nguyên giá trị cũ (không bị vô hiệu
     hoá ngược) cho đến lần switch tiếp theo của repo đó.
   - **`/switch-token <alias>`**: đổi secret `CLAUDE_CODE_OAUTH_TOKEN` của repo đang link với
     channel sang token của `alias` đó (mã hoá sealed-box + `PUT` qua GitHub Secrets API — xem mục
     9). `alias` không tồn tại trong pool → báo lỗi, không đổi gì. `alias` tồn tại nhưng đang
     cooldown → **vẫn cho đổi** (lệnh tay luôn tôn trọng lựa chọn rõ ràng của người dùng; cooldown
     chỉ chặn việc tự chọn của auto-switch, không chặn 1 yêu cầu tay nêu rõ alias).
   - **`/switch-token`** (không kèm alias): trả lời alias hiện tại của repo + toàn bộ danh sách
     pool (mỗi alias kèm trạng thái cooldown nếu có). Nếu alias hiện tại của repo đã bị
     `/remove-token` xoá khỏi pool, ghi chú rõ "(đã bị xoá khỏi pool)" thay vì hiện như 1 alias
     bình thường.

9. **Secret GitHub Actions phải mã hoá sealed-box** (yêu cầu bắt buộc của GitHub REST API, không
   có cách khác): `GET /repos/{owner}/{repo}/actions/secrets/public-key` lấy `{key_id, key}`, mã
   hoá giá trị token bằng `crypto_box_seal` với public key đó, base64, rồi
   `PUT /repos/{owner}/{repo}/actions/secrets/CLAUDE_CODE_OAUTH_TOKEN` với
   `{encrypted_value, key_id}`. Thêm dependency JS mới cho Worker (ví dụ `tweetnacl` — thuần JS,
   không cần WASM, rất nhỏ) — dependency runtime **đầu tiên** của `worker/`.

10. **1 endpoint mới cho CI gọi**, `POST /switch-token/auto`, xác thực bằng chính
    `X-Relay-Token`/`SLACK_RELAY_TOKEN` đang dùng cho `/relay` (không phát sinh secret mới). Body
    `{owner, repo, rateLimitResetText?}`:
    - Nếu có `rateLimitResetText`: parse ra timezone + giờ reset (dùng `Intl.DateTimeFormat` với
      zone tên trong chuỗi đó — runtime Workers có đủ dữ liệu ICU, không cần thêm thư viện
      timezone), tra `repo-token:{owner}/{repo}` ra alias hiện tại (nếu có) và đánh dấu
      `token-cooldown:{alias đó}` = giờ reset đã parse.
    - Chọn **ngẫu nhiên** 1 trong các alias "khả dụng" (không đang cooldown) — không theo thứ tự
      cố định, để nhiều request chọn gần như đồng thời có xu hướng rải ra các alias khác nhau thay
      vì luôn trúng đúng 1 alias giống nhau (xem Risk 6: giảm xác suất va chạm, không loại bỏ hoàn
      toàn). Nếu không còn alias khả dụng nào, trả `{ok:false, error:"no_alias_available"}`, không
      set secret gì cả. Không có cơ chế khoá/claim nào giữa bước đọc và bước ghi.
    - Có alias khả dụng: set secret `CLAUDE_CODE_OAUTH_TOKEN` của repo đó (mục 9), ghi
      `repo-token:{owner}/{repo}` = alias mới, trả `{ok:true, alias, token}` (token thật, không chỉ
      alias — vì job CI cần giá trị ngay cho lượt thử tiếp theo trong CÙNG lần chạy).
    - **Nếu set secret GitHub (mục 9) thất bại** (lỗi lấy public key, hoặc `PUT` bị từ chối): KHÔNG
      ghi `repo-token:{owner}/{repo}` = alias mới, trả `{ok:false, error:"set_secret_failed"}` —
      không bao giờ trả `{ok:true}` giả khi secret thật chưa đổi (đúng yêu cầu interview: không
      được âm thầm báo thành công).
    - **Nếu ghi `token-cooldown:{alias}` thất bại** (lỗi KV): bỏ qua, coi alias đó vẫn khả dụng cho
      lần chọn kế tiếp — fail-open, không chặn luồng chính vì đây chỉ là tối ưu chọn lựa, không
      phải nguồn sự thật duy nhất.
    - Không có `rateLimitResetText` (ca bootstrap, hoặc repo chưa có alias nào) — bỏ qua bước đánh
      dấu cooldown, chỉ chọn alias khả dụng + set secret + trả về, dùng chung logic chọn alias với
      ca auto-switch.

11. **`bootstrap-project.yml`** gọi đúng endpoint ở mục 10 (không kèm `rateLimitResetText`) ngay
    sau bước vendor+set-secret hiện có, để lấy 1 alias+token từ pool thay vì copy secret
    `CLAUDE_CODE_OAUTH_TOKEN` cố định của `control-plane`. Nếu pool rỗng (`no_alias_available`
    ngay từ repo đầu tiên): fallback về hành vi hiện tại (copy secret cố định của `control-plane`)
    — không chặn bootstrap chỉ vì pool chưa được seed.

## Acceptance Criteria

- [ ] `/add-token work2 sk-ant-oat01-xxx` trong channel đã link → token lưu vào pool dưới alias
  `work2`; reply không chứa lại giá trị token.
- [ ] `/add-token work2 <token khác>` khi `work2` đã tồn tại → báo lỗi, token cũ trong pool không
  đổi.
- [ ] `/add-token onlyalias` (không có phần token) → báo lỗi cú pháp, không lưu gì.
- [ ] `/switch-token work2` → secret `CLAUDE_CODE_OAUTH_TOKEN` của repo link với channel đó đổi
  đúng giá trị token của `work2`, mã hoá đúng sealed-box (xác minh bằng 1 lần `claude -p` thật chạy
  trên repo đó sau đổi — xác nhận secret mới thật sự có hiệu lực, không chỉ API trả 2xx).
- [ ] `/switch-token khong-ton-tai` → báo lỗi, secret không đổi.
- [ ] Giả lập set secret GitHub thất bại (ví dụ public key fetch lỗi) trong `/switch-token/auto` →
  trả `{ok:false, error:"set_secret_failed"}`, KHÔNG ghi `repo-token` sang alias mới, không báo
  thành công giả.
- [ ] Giả lập ghi `token-cooldown:{alias}` thất bại (lỗi KV) trong `/switch-token/auto` → không
  chặn luồng chính, vẫn chọn+set secret alias mới bình thường (fail-open, chỉ mất tối ưu chọn lựa
  cho lần sau, không làm cả request fail).
- [ ] 1 alias đang cooldown, sau khi qua đúng giờ reset đã lưu → được chọn lại bình thường ở lần
  auto-switch/`/switch-token` tiếp theo (không bị kẹt "không khả dụng" vĩnh viễn).
- [ ] `/switch-token` (không alias) → trả lời đúng alias hiện tại của repo + danh sách toàn bộ pool.
- [ ] `/remove-token work2` khi đang là alias hiện tại của 1 repo → xoá khỏi pool thành công; secret
  `CLAUDE_CODE_OAUTH_TOKEN` của repo đó KHÔNG đổi ngay; `/switch-token` (status) sau đó cho repo đó
  hiện "work2 (đã bị xoá khỏi pool)".
- [ ] 1 lần `claude -p` thật trong `implement`/`knowledge-init`/`userspec-turn` gặp đúng lỗi
  `"You've hit your session limit"` hoặc `"...weekly limit"` → workflow tự gọi `control-plane`,
  đổi sang token khác trong pool, thử lại trong CÙNG lần chạy, không cần người can thiệp; lượt đó
  tính vào đúng 3 lần `ATTEMPTS` hiện có.
- [ ] Khi pool chỉ còn đúng 1 token và nó đang bị rate-limit (test giả lập: đặt cooldown tay qua
  KV hoặc để auto-switch tự đánh dấu) → lần `claude -p` tiếp theo gặp rate-limit dừng ngay, báo lỗi
  rõ ràng qua `gh issue comment` + Slack, không tốn hết 3 lượt.
- [ ] `userspec-turn` hiện có retry loop 3 lần (trước đây không có) — 1 lỗi kỹ thuật thường (không
  phải rate-limit) khi chạy `userspec-turn` giờ được retry tối đa 3 lần, không fail ngay từ lỗi đầu.
- [ ] 1 lần `/new-project` thật mới → repo mới được gán 1 alias từ pool (không phải luôn cùng 1
  token cố định như trước), xác nhận qua `repo-token:{owner}/{repo}` trong KV hoặc `/switch-token`
  status ngay sau bootstrap.
- [ ] 1 chuỗi lỗi output của `claude -p` KHÔNG khớp cả 2 mẫu rate-limit → không gọi Worker, không
  đổi token gì cả; lượt đó retry với đúng token cũ, giống hệt hành vi trước khi có feature này.
- [ ] Giả lập `POST /switch-token/auto` trả lỗi (timeout/5xx/network error, không phải
  `{ok:false, error:"no_alias_available"}`) → lượt đó rơi về retry-cùng-token như cũ, không làm
  run fail khác hay dừng sớm so với hành vi trước feature này.
- [ ] `skills/online-pipeline/SKILL.md` được cập nhật mô tả rõ hành vi rate-limit-detection +
  auto-switch cho cả 3 stage (`userspec-turn`, `implement`, `knowledge-init`) — đồng bộ cả 4 file
  đang track (1 nguồn, 1 bản vendor trong scaffold `project-initialization` giống byte-for-byte
  bản nguồn, và 2 bản mirror `.codex/` tương ứng của 2 file trên).

## Constraints

- Pool lưu Cloudflare KV, dùng namespace `MAPPINGS` đã có — không thêm KV binding mới.
- Không thêm tầng quyền mới: mọi lệnh pool (switch/add/remove/status) dùng đúng mức tin tưởng hiện
  tại (ai trong channel đã link cũng gọi được), giống `/new-feature`.
- Endpoint CI-facing mới dùng lại đúng `X-Relay-Token`/`SLACK_RELAY_TOKEN` đang có cho `/relay` —
  không phát sinh secret mới.
- Mã hoá secret GitHub Actions bắt buộc theo đúng sealed-box scheme của GitHub REST API — không
  có cách khác, phải thêm 1 dependency JS runtime mới cho `worker/` (dependency đầu tiên).
- Auto-switch chỉ cộng thêm vào hành vi hiện có — mọi lỗi/trường hợp không nhận diện được rate-limit
  (sai mẫu, lỗi gọi Worker) phải rơi về đúng hành vi retry-cùng-token hiện tại, không làm hệ thống
  kém tin cậy hơn trước feature này.
- Lượt retry do đổi token tính vào đúng giới hạn `ATTEMPTS=3` hiện có của mỗi workflow — không nới
  giới hạn tổng số lần thử.
- Cả 3 workflow YAML (và bản vendor trong scaffold `project-initialization` + mirror `.codex/`)
  phải được cập nhật đồng bộ, đúng quy ước 3-bản-giống-nhau đã áp dụng xuyên suốt `online-pipeline`.
- Mọi repo đã onboard cần tự `vendor-skills.sh` lại sau khi feature này merge mới nhận được
  auto-switch — không phải lỗi riêng của feature này, đúng quy ước maintenance có sẵn.

## Risks

- **Risk 1:** Nhận diện sai mẫu lỗi rate-limit (ví dụ viết sai phần cố định của chuỗi, hoặc GitHub
  đổi câu chữ theo thời gian) khiến auto-switch không bao giờ kích hoạt — đúng loại lỗi đã xảy ra
  thật trong session này (nhầm `"github-actions[bot]"` với `"github-actions"` ở 1 feature khác).
  **Mitigation:** chỉ dùng đúng phần cố định của 2 chuỗi đã xác nhận thật từ log production trong
  interview này (`"You've hit your session limit"` / `"...weekly limit"`, phần `resets {time}
  ({tz})` luôn thay đổi — xem Expected Behavior Phần A mục 1), review kỹ bash khớp mẫu này ở
  Agent Verification (bước 7) vì đây là bash trong workflow YAML, không viết được bằng unit test
  JS; không suy đoán biến thể khác chưa có bằng chứng.
- **Risk 2:** Trả token thật (plaintext) qua response HTTP của endpoint mới cho CI job — mặt lộ
  diện mới. **Mitigation:** cùng mức tin tưởng với secret GitHub Actions bản thân nó (vốn cũng
  truyền plaintext tới runner qua kênh riêng của GitHub), truyền qua HTTPS, xác thực bằng đúng
  shared secret `SLACK_RELAY_TOKEN` đã được tin tưởng cho `/relay`.
- **Risk 3:** Dependency mã hoá sealed-box mới làm tăng kích thước bundle Worker, có thể chạm giới
  hạn kích thước script của Cloudflare Workers. **Mitigation:** chọn thư viện thuần JS, không WASM,
  kích thước nhỏ (ví dụ `tweetnacl`); kiểm tra kích thước build trước khi merge.
- **Risk 4:** KV không cách ly bằng mã hoá như Worker secret — bất kỳ ai có quyền truy cập binding
  Worker đều đọc được token trong pool. **Mitigation:** chấp nhận theo đúng lựa chọn rõ ràng của
  người dùng (ưu tiên không cần redeploy khi thêm/xoá token hơn mức cách ly cao nhất).
- **Risk 5:** `/add-token` không giới hạn quyền — bất kỳ ai trong channel đã link cũng thêm được
  token mới vào pool chung (dùng được cho MỌI repo khác, không chỉ repo của channel đó).
  **Mitigation:** chấp nhận theo đúng lựa chọn rõ ràng của người dùng (câu 15 interview); đã có
  lớp chặn riêng cho ghi đè (mục 8) để giảm rủi ro cụ thể nhất (đổi ngầm token đang dùng chung).
- **Risk 6:** 2+ repo cùng bị rate-limit gần như đồng thời gọi `/switch-token/auto` cùng lúc —
  **đây không phải trường hợp hiếm, mà đúng là kịch bản chính feature này nhắm tới** (nhiều repo
  chia sẻ 1 token, cùng cạn hạn mức cùng lúc). Cloudflare KV không có compare-and-swap thật (và là
  eventually-consistent giữa các edge location, có thể mất tới hàng chục giây để 1 lần ghi hiển
  thị ở nơi khác — đúng điều kiện mà 2 request CI ở 2 vùng địa lý khác nhau dễ gặp phải), nên bước
  đọc-rồi-ghi có thể đọc trùng danh sách "alias khả dụng". **Mitigation:** chọn alias **ngẫu
  nhiên** trong số khả dụng (mục 10) thay vì theo thứ tự cố định — với N alias khả dụng, 2 request
  va nhau chỉ còn khoảng 1/N xác suất chọn trúng cùng 1 alias, thay vì chắc chắn trùng như thuật
  toán thứ-tự-cố-định ban đầu. Không loại bỏ hoàn toàn race (vẫn có xác suất trùng, không có
  khoá/claim thật), nhưng không cần thêm hạ tầng mới (Durable Objects) cho v1 — để lại cho 1
  user-spec riêng nếu va chạm xác suất này vẫn gây vấn đề thật trong thực tế.
- **Risk 7:** 2 người gọi `/add-token` với **cùng 1 alias mới** (chưa tồn tại trong pool) gần như
  đồng thời — cùng loại race check-rồi-ghi trên KV, nhưng điều kiện xảy ra (2 người độc lập chọn
  đúng cùng 1 tên alias chưa từng dùng, cùng lúc) thực sự hiếm, không tương quan với kịch bản bận
  rộn của Risk 6. **Mitigation:** chấp nhận, không mitigate cho v1 — ngẫu nhiên hoá không áp dụng
  được cho trường hợp này (không có "nhiều lựa chọn" để rải ra, chỉ có đúng 1 alias cả 2 cùng nhắm
  tới).

## Accepted Decisions

- Pool token lưu Cloudflare KV, không dùng Worker secret riêng từng slot — ưu tiên thêm/xoá token
  không cần redeploy, đánh đổi mức cách ly thấp hơn secret thật (đã cân nhắc, chấp nhận).
- Mọi lệnh (switch/add/remove/status) dùng đúng mức tin tưởng hiện tại của channel đã link, không
  thêm tầng quyền riêng cho `/add-token` dù rủi ro cao hơn `/switch-token` — người dùng chủ động
  chọn không giới hạn thêm.
- Auto-switch chỉ khả thi dưới dạng phản ứng (detect lỗi rate-limit trong `claude -p`'s output rồi
  gọi Worker đổi + retry), không có cơ chế polling nền như `cswap auto` — vì mỗi GitHub Actions run
  là 1 process độc lập, không có nơi nào để chạy vòng lặp polling liên tục.
- Né cooldown "thông minh" (lưu giờ reset thật, bỏ qua token đó đến khi hết giờ) thay vì xoay vòng
  mù — vì pool có thể chỉ có 1-2 token, xoay vòng mù dễ thử lại đúng token vừa bị limit.
- Khi pool hết token khả dụng: dừng lại ngay, báo lỗi rõ ràng — không cố thử "best-effort" với
  token biết chắc vẫn đang rate-limit, tránh lãng phí lượt retry còn lại 1 cách vô ích.
- `userspec-turn` được thêm hẳn 1 retry loop 3 lần đầy đủ (không chỉ 1 lượt tối giản) — vừa đồng bộ
  với `implement`/`knowledge-init`, vừa tiện sửa luôn hành vi "không retry gì cả" hiện tại của nó.
- Rate-limit giữa lúc `implement` đang code: xử lý giống mọi lỗi kỹ thuật khác của loop đó — reset
  về `$LAST_GOOD` rồi làm lại từ đầu với token mới, không xây cơ chế resume giữa chừng mới.
- `/add-token` trên alias đã tồn tại: báo lỗi, không ghi đè — phải `/remove-token` trước. Tránh 1
  người vô tình đổi ngầm token đang được nhiều repo khác phụ thuộc.
- `/remove-token` trên alias 1 repo đang dùng: cho xoá, secret hiện tại của repo đó giữ nguyên giá
  trị cũ (không rollback ngược) — đơn giản hơn truy vết/cập nhật ngược mọi repo đang dùng alias đó.
- Repo mới bootstrap qua `/new-project` cũng lấy token từ pool (không giữ nguyên hành vi copy 1
  token cố định) — đúng mục đích ban đầu của pool (phân tải nhiều repo qua nhiều account), có
  fallback về hành vi cũ nếu pool rỗng lúc bootstrap.
- Endpoint CI-facing mới tái dùng `SLACK_RELAY_TOKEN` hiện có, không phát sinh secret riêng — nhất
  quán với kiến trúc `/relay` đã có.
- Reset-time parse dùng `Intl.DateTimeFormat` với tên zone bắt được trong chuỗi lỗi thật (`UTC`,
  `Asia/Saigon`) — Workers runtime có đủ dữ liệu ICU, không cần thêm thư viện timezone riêng.
- Race condition khi chọn alias cho auto-switch (Risk 6) — đây KHÔNG phải rủi ro hiếm, mà tương
  quan trực tiếp với kịch bản chính feature này nhắm tới (nhiều repo cùng rate-limit). Đã xem xét
  và loại 2 phương án: claim TTL ngắn trên KV (không hiệu quả — TTL tối thiểu bắt buộc 60s, KV
  eventually-consistent giữa vùng) và Durable Objects (đúng nhưng phức tạp hơn mức cần cho v1).
  Chọn **thuật toán chọn alias ngẫu nhiên** thay vì thứ tự cố định — sửa nhỏ, không thêm hạ tầng,
  giảm xác suất va chạm từ "chắc chắn trùng" xuống "~1/N xác suất trùng" (N = số alias khả dụng).
  Chấp nhận phần xác suất còn lại cho v1.
- Race condition khi `/add-token` cùng alias mới (Risk 7): chấp nhận là rủi ro hiếm thật (khác
  Risk 6 — không tương quan với kịch bản bận rộn), không mitigate — ngẫu nhiên hoá không áp dụng
  được cho trường hợp chỉ có đúng 1 lựa chọn.

## Testing

**Unit tests:** cần — mọi logic thuần sống trong `control-plane/worker` có thể test độc lập (parse
giờ reset + timezone ra timestamp tuyệt đối, logic chọn alias khả dụng ngẫu nhiên — test rằng kết
quả luôn thuộc tập khả dụng và không bao giờ là alias đang cooldown, không cần test tính "ngẫu
nhiên" thật của nó — /bỏ qua cooldown/phát hiện hết pool kể cả nhánh "cooldown hết hạn → khả dụng
lại", các KV key builder mới, helper mã hoá sealed-box nếu tách được thành hàm thuần) — theo đúng
quy ước `worker/src/*.test.js` (vitest) đã
có. `index.js`'s wiring (các handler slash-command mới, endpoint mới) không cần test riêng — đúng
quy ước sẵn có của file này (business logic sống ở `lib.js`/`github.js` để test được).

Việc khớp 2 mẫu lỗi rate-limit (và nhánh "không khớp → giữ hành vi cũ", "gọi Worker thất bại → rơi
về retry-cùng-token") sống trong bash của cả 3 workflow YAML ở `molyanov-ai-dev` (xem Expected
Behavior Phần A mục 1-2), KHÔNG phải JS của Worker — không viết được bằng vitest ở `worker/src/`.
Xác nhận bằng đọc lại logic (Agent Verification bước 7), không phải unit test.

**Integration tests:** không cần tự động riêng — không có hạ tầng integration-test hiện có cho
Worker này ngoài unit test + E2E thật; không phát sinh hạ tầng mới ngoài phạm vi.

**E2E tests:** cần, làm tay bởi người dùng — đây là hành vi thật trên Slack workspace + GitHub
Secrets API + Cloudflare KV/Worker deploy thật, không thể giả lập đầy đủ bằng unit test (đặc biệt
việc secret mới thật sự có hiệu lực ở lần chạy CI kế tiếp, và việc tái hiện 1 lần rate-limit thật
để xác nhận auto-switch — khó ép xảy ra theo ý muốn, xem Verification).

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Chạy `npm test` trong `control-plane/worker` sau khi thêm code | Toàn bộ test cũ + mới pass: parse giờ reset + timezone, logic chọn alias/cooldown (gồm nhánh hết hạn → khả dụng lại), KV key builder mới, helper sealed-box nếu tách được thành hàm thuần. (Việc khớp 2 chuỗi mẫu rate-limit KHÔNG nằm trong test này — nó sống trong bash của 3 workflow YAML ở `molyanov-ai-dev`, xem bước 7.) |
| 2. Đọc lại `online-pipeline-userspec.yml` sau khi sửa | Có retry loop 3 lần mới (trước đây không có), có bước nhận diện rate-limit + gọi Worker trước khi phân loại thành công/thất bại. |
| 3. Đọc lại `online-pipeline-implement.yml`/`online-pipeline-knowledge-init.yml` sau khi sửa | Loop 3 lần hiện có được thêm bước nhận diện rate-limit + gọi Worker, không đổi cấu trúc reset-on-failure hiện tại; lượt đổi token vẫn tính vào đúng 3 `ATTEMPTS`. |
| 4. Đọc code mới trong `control-plane/worker/src/index.js`/`github.js`/`lib.js` | 3 slash command mới + endpoint `/switch-token/auto` đúng hành vi đã chốt (không ghi đè alias trùng, không echo token, cooldown chỉ chặn tự chọn không chặn tay). |
| 5. Đọc `bootstrap-project.yml` sau khi sửa | Gọi đúng endpoint mới để lấy alias ban đầu, có fallback về secret cố định khi pool rỗng. |
| 6. Kiểm tra cả 3 bản vendor (`.claude/skills/online-pipeline/assets/workflows/`, `.github/workflows/`, mirror `.codex/`) giống nhau | `diff` sạch giữa 3 bản, đúng quy ước đã áp dụng xuyên suốt `online-pipeline`. |
| 7. Đọc lại bash logic nhận diện rate-limit trong cả 3 workflow | Chuỗi KHÔNG khớp 2 mẫu rate-limit, hoặc lời gọi Worker bị lỗi (không phải phản hồi `no_alias_available`), đều rơi về đúng nhánh retry-cùng-token hiện có — không có đường nào khiến lượt đó fail khác hay dừng sớm hơn hành vi trước feature này. |
| 8. Đọc lại `skills/online-pipeline/SKILL.md` sau khi sửa | Có mục mô tả rõ hành vi rate-limit-detection + auto-switch cho cả 3 stage (`userspec-turn`, `implement`, `knowledge-init`); cả 4 file đang track (nguồn, bản vendor, 2 mirror `.codex/`) giống nhau theo đúng cặp (nguồn = mirror của nguồn, vendor = mirror của vendor). |

### User Verification

- Deploy lại Worker (`cd control-plane/worker && npm run deploy`) và re-vendor `online-pipeline`
  trên ít nhất 1 repo thật — agent không có quyền deploy Cloudflare hay push lên repo thật của
  người dùng mà không được xin phép rõ.
- Chạy tay `/add-token`, `/switch-token`, `/remove-token` thật trên Slack workspace thật, xác nhận
  secret GitHub thật đổi đúng và có hiệu lực ở 1 lần `claude -p` thật tiếp theo — cần Slack
  workspace + GitHub thật, agent không truy cập được.
- Xác nhận auto-switch hoạt động khi gặp rate-limit thật: khó ép xảy ra theo ý muốn (phụ thuộc hạn
  mức thật của account) — chấp nhận xác nhận thụ động (theo dõi lần rate-limit tự nhiên tiếp theo
  xảy ra, kiểm tra log Actions thấy đúng hành vi đổi token + retry).
- Chạy 1 lần `/new-project` thật, xác nhận repo mới nhận alias từ pool (không phải luôn cùng 1
  token cố định).
