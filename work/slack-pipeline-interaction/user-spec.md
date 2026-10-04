---
# Creation date (YYYY-MM-DD)
created: 2026-10-04

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: slack-pipeline-interaction

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building

Hai năng lực gộp vào 1 feature:

**Phần A — Tương tác `online-pipeline` qua Slack** cho repo đã bật online-pipeline sẵn: 1 Slack
channel cố định/repo (public, tạo tay 1 lần), 1 thread/feature. Slash command bắt đầu feature mới;
toàn bộ tương tác sau đó (hỏi-đáp interview, `awaiting_decision` lúc implement, approve spec PR,
approve/merge code PR) làm được ngay trong Slack, post song song cả GitHub (như cũ) và Slack. Cầu
nối là 1 Cloudflare Worker (đa-repo, deploy 1 lần/account) — Worker là nơi DUY NHẤT giữ Slack Bot
Token thật; mỗi repo chỉ gọi qua 1 endpoint relay trên Worker đó.

**Phần B — Khởi tạo project GitHub hoàn toàn mới từ Slack** (`/new-project`): 1 repo điều phối
(control-plane) mới, chạy `claude -p project-initialization` (chế độ tự động mới) để tạo repo thật,
rồi tự vendor + set secret cho cả online-pipeline và Slack integration — sẵn sàng dùng ngay, không
cần bước tay nào ở local ngoài việc tạo channel Slack.

## Why

Hiện tại `online-pipeline` chỉ tương tác qua GitHub issue/PR comment — phải mở GitHub (web/app) để
trả lời interview, xem quyết định cần duyệt, hay approve. Người dùng muốn làm toàn bộ việc này ngay
trong Slack (nơi đã quen chat hàng ngày), chỉ ra GitHub khi cần xem diff code thật trước khi
approve. Việc tạo project mới cũng vậy — muốn khởi động 1 dự án hoàn toàn mới chỉ bằng vài dòng lệnh
trong Slack, không cần mở terminal.

## Expected Behavior

### Phần A — Setup 1 lần cho cả account/workspace

1. Tạo Slack App (api.slack.com/apps), cấp scope `chat:write` + `channels:history` +
   `channels:join` (không cần `channels:manage` — channel luôn tạo tay).
2. Lấy Bot Token + Signing Secret, set vào Cloudflare Worker secrets (`wrangler secret put`) —
   Worker là nơi DUY NHẤT trong toàn hệ thống giữ Slack Bot Token thật. Tự sinh 1
   `SLACK_RELAY_TOKEN` (chuỗi bí mật ngẫu nhiên, KHÔNG phải token Slack) dùng để xác thực các repo
   gọi vào Worker — set vào Worker secrets.
3. Deploy Cloudflare Worker — **đúng 1 lần cho cả account**, phục vụ MỌI repo sau này (không
   redeploy mỗi repo). Lấy URL Worker, set làm Request URL cho Slack Slash Command + Event
   Subscriptions trong Slack App config. Worker expose thêm 1 endpoint nội bộ `POST /relay` (nhận
   `channel_id`, `thread_ts` tuỳ chọn, `text`, xác thực bằng header mang `SLACK_RELAY_TOKEN`) —
   endpoint DUY NHẤT mà GitHub Actions job của mọi repo được gọi để post vào Slack; Worker tự dùng
   Slack Bot Token thật của nó để thực hiện `chat.postMessage`.
4. Tạo repo điều phối (control-plane) — source code của Worker đặt NGAY trong repo này (không
   vendor vào `skills/online-pipeline/`, không phải repo thứ 3 — cả Worker và control-plane đều là
   hạ tầng singleton/account-wide). Thêm workflow `bootstrap-project.yml` nhận
   `repository_dispatch`. Set secret riêng cho job Actions này (`GH_PAT` quyền tạo-repo-được,
   Claude auth credential, `SLACK_RELAY_TOKEN` dùng chung) qua `gh secret set` — kho secret này
   tách biệt hoàn toàn với secret của Worker (2 runtime khác nhau, `SLACK_RELAY_TOKEN` dùng cùng
   giá trị ở cả 2 nơi để Worker xác thực được).
5. Cài Claude GitHub App ở mức **"All repositories"** trên account GitHub — 1 lần, để mọi repo tạo
   sau này (kể cả qua `/new-project`) tự động được app này cover, không cần cài lại tay.

### Phần A — Onboard 1 repo đã tồn tại từ trước

6. Chạy `setup-online-pipeline.sh` cục bộ (mở rộng thêm): vendor + set secret `GH_PAT`/Claude-auth
   như cũ, VÀ THÊM `SLACK_RELAY_TOKEN` (giá trị copy từ Worker, KHÔNG phải Slack Bot Token thật)
   làm secret repo — vì mỗi GitHub Actions job (`userspec-turn`, `implement`) gọi `POST /relay`
   trên Worker (không gọi Slack API trực tiếp) để post song song; `SLACK_RELAY_TOKEN` chỉ cho phép
   gọi đúng endpoint relay đó, không có quyền gì khác trên Slack — nếu secret này bị lộ, thiệt hại
   chỉ là "ai đó post được tin nhắn giả vào đúng channel đã link", không lộ token Slack thật.
   NẾU repo chưa chạy bước này (`SLACK_RELAY_TOKEN` không tồn tại): mọi lệnh gọi `/relay` bị bỏ qua
   (có kiểm tra secret trống trước khi gọi), hành vi `online-pipeline` hiện có (chỉ `gh issue
   comment`) không đổi.
7. Tự tạo 1 channel Slack cho project đó (tay) — **phải là public channel**
   (`conversations.join` chỉ hoạt động với channel public, Worker không tự join được private
   channel).
8. Gõ `/link-repo <owner>/<name>` trong channel đó 1 lần — Worker tự `conversations.join` channel
   này (dùng scope `channels:join`), rồi ghi mapping `channel_id <-> repo` vào Cloudflare KV.

### Phần A — Vòng đời 1 feature qua Slack

9. User gõ `/new-feature <mô tả>` trong channel đã link — Worker ACK ngay (<3s, theo giới hạn
   Slack), xử lý thật ở background (`ctx.waitUntil`): post message mở thread trước (lấy
   `thread_ts` của chính message đó), tạo GitHub issue mới (title từ mô tả, body nhúng `channel_id`
   + `thread_ts` vừa có), ghi `thread_ts -> issue` vào Cloudflare KV, gửi kết quả qua
   `response_url`. `userspec-turn` chạy, câu hỏi đầu tiên post song song vào GitHub issue comment
   (như cũ) VÀ gọi `POST /relay` trên Worker để post vào Slack thread.
10. User trả lời bằng tin nhắn thường trong thread (qua Slack Events API — Worker ACK 200 ngay rồi
    xử lý background; đường này không có `response_url`, Worker tự `chat.postMessage` bằng token
    thật của nó để báo kết quả) — Worker relay thành `gh issue comment`, interview tiếp tục, câu
    hỏi kế post song song cả 2 nơi như bước 9.
11. Validate xong, spec PR mở — message trong thread (qua `/relay`) có link PR + hướng dẫn gõ
    `!approve`. User mở link, xem diff trên GitHub, quay lại Slack gõ `!approve` trong thread —
    Worker đọc status marker biết đang chờ spec-approve, GIẢ LẬP đúng hành vi người dùng: post
    comment `/approve` lên PR đó — route job cũ của `online-pipeline-userspec.yml` xử lý y nguyên
    (merge spec PR, implement bắt đầu).
12. Lúc implement gặp quyết định cần hỏi (`awaiting_decision`) — câu hỏi/finding post song song vào
    GitHub VÀ thread (qua `/relay`), user trả lời ngay trong Slack như bước 10.
13. Implement xong, code PR mở — message (qua `/relay`) có link PR + hướng dẫn `!approve`. User xem
    diff, quay lại Slack gõ `!approve` — Worker đọc status marker biết đang chờ code-approve, gọi
    GitHub REST API `PUT .../pulls/{n}/merge` TRỰC TIẾP (năng lực mới — không có listener comment
    nào cho code PR hiện nay; không cần workflow/trigger GitHub Actions mới vì `pull_request:
    closed` không phân biệt actor/phương thức merge) — finalize chạy.
14. Finalize xong — message "✅ Feature xong" post vào thread qua `/relay` (năng lực mới — hiện tại
    `finalize` không post gì cả, kể cả lên GitHub).

### Phần B — Khởi tạo project mới

15. User tự tạo 1 channel Slack mới (public) cho project sắp tạo.
16. Gõ `/new-project <tên-repo>` NGAY TRONG channel đó (top-level, không phải reply vào thread nào
    — chưa có feature/thread nào tồn tại ở bước này). Worker ACK ngay, tự `conversations.join`
    channel này, ghi mapping `channel_id -> <tên-repo>` vào KV (biết sẵn 2 giá trị, không cần
    round-trip), rồi bắn `repository_dispatch` tới repo điều phối, kèm `channel_id` trong payload
    (KHÔNG kèm `thread_ts` — không có thread ở bước này).
17. Workflow `bootstrap-project.yml` trong repo điều phối nhận dispatch, chạy trên 1 working
    directory HOÀN TOÀN TRỐNG (không phải checkout của chính repo điều phối) rồi gọi `claude -p`
    thật với skill `project-initialization`, kèm literal tự động `PROJECT_BOOTSTRAP_AUTOMATED` VÀ
    tên repo đã có sẵn từ lệnh Slack (đúng hợp đồng có sẵn của skill này: "ask for the GitHub
    repository name unless already supplied" — tên đã được cung cấp nên bước hỏi này tự nhiên
    không kích hoạt, không cần signal mới). Để skill TỰ chạy `gh repo create` ở Step 4 của chính nó
    (KHÔNG tự `gh repo create` trước rồi mới gọi `claude -p` — nếu làm vậy, `origin` đã tồn tại khi
    Step 4 chạy, rơi vào nhánh "origin đã có, hỏi xác nhận đây có phải đúng repo" thay vì nhánh
    "origin chưa có, tạo mới" — không ai trả lời được câu hỏi xác nhận đó trong 1 job tự động). Vì
    thư mục luôn trống/không có `.git` khi bắt đầu, Step 1 của skill (ask chỉ khi "đã là git repo
    VÀ có uncommitted changes") không bao giờ kích hoạt — không cần signal bỏ qua riêng cho bước
    này. Dev branch cũng luôn chưa tồn tại (repo mới hoàn toàn) nên nhánh "ask có nên tái dùng dev"
    của Step 4 không kích hoạt. Signal `PROJECT_BOOTSTRAP_AUTOMATED` CHỈ cần bỏ qua ĐÚNG 1 điểm
    thật sự luôn có mặt bất kể hoàn cảnh: "Ask explicitly before pushing main" ở cuối Step 4 — tự
    push ngay vì chính lệnh Slack là xác nhận. Step 5 (Report) không có "ask" nào — signal này chỉ
    đổi ĐÍCH của report từ chat sang gọi `POST /relay` trên Worker, không phải bỏ qua gì cả. Sau khi
    `project-initialization` xong, job tự vendor + set secret cho CẢ `online-pipeline` VÀ Slack
    integration trên repo mới (copy `GH_PAT`, Claude auth credential, `SLACK_RELAY_TOKEN` dùng
    chung từ control-plane) — không cần chạy script tay ở local.
18. Job gửi kết quả về đúng `channel_id` (top-level message, không phải reply thread) qua `POST
    /relay`: link repo mới, xác nhận đã sẵn sàng. User gõ `/new-feature ...` ngay trong channel đó
    như bước 9 — không cần làm gì thêm ở local, vì Claude GitHub App đã cover repo này nhờ cài ở
    mức "All repositories" từ bước 5.

### Trường hợp biên đã đồng ý (cả Phần A và Phần B)

- Gõ `/new-feature` hoặc `!approve` trong 1 channel CHƯA từng `/link-repo` hay `/new-project` →
  Worker báo lỗi rõ "channel chưa được liên kết với repo nào, gõ /link-repo hoặc /new-project
  trước", không làm gì trên GitHub.
- Tin nhắn thường gửi vào channel nhưng KHÔNG phải reply vào thread nào (tin nhắn rời ở top-level)
  → Worker bỏ qua hoàn toàn, không relay, không báo lỗi (tránh spam GitHub với comment rác từ chat
  thường không liên quan tới feature nào).
- Tin nhắn đến cho 1 thread mà feature tương ứng đã finalize/archive xong (hoặc issue đã closed) →
  Worker không tìm thấy issue đang mở tương ứng → báo lỗi rõ trong thread, không tạo hành động gì
  trên GitHub.
- `!approve` gõ nhiều lần liên tiếp (double-submit) trong lúc route job đầu đang xử lý → chấp nhận
  rủi ro race nhỏ (last-write-wins, nhất quán với cách `hybrid-local-online-pipeline` đã chấp nhận
  cho `/switch-online`), không xây lock.
- Slash command gõ 2 lần cho cùng 1 mô tả feature (tạo nhầm trùng) → mỗi lần tạo 1 issue/thread
  riêng, không de-dup — hạn chế v1, người dùng tự đóng issue thừa.
- `/link-repo` gõ 2 lần cho cùng 1 channel với 2 repo khác nhau → ghi đè mapping cũ bằng mapping
  mới (last-write-wins), không cảnh báo "đang ghi đè" — hạn chế v1.
- `/new-project` gõ trong channel đã map với 1 project khác rồi → cho phép về mặt kỹ thuật (không
  chặn), nhưng là dùng sai quy ước UX đã thống nhất (1 project = 1 channel) — hạn chế v1, không
  validate/chặn phía hệ thống.

## Acceptance Criteria

**Phần A:**
- [ ] `/link-repo <owner>/<name>` gõ trong 1 channel public → Worker tự join channel (qua
      `conversations.join`), ghi mapping `channel_id <-> repo` vào KV; gõ với repo không tồn tại
      hoặc không có quyền → báo lỗi rõ, không ghi mapping.
- [ ] `/new-feature <mô tả>` trong channel đã link → GitHub issue mới xuất hiện (body nhúng
      `channel_id`+`thread_ts`), thread mới xuất hiện trong channel đó, `userspec-turn` chạy và
      post câu hỏi đầu tiên vào ĐÚNG thread (qua `POST /relay`) VÀ đồng thời `gh issue comment`
      như cũ.
- [ ] Mọi câu hỏi/thông báo bot ở `userspec-turn`, `implement`, `awaiting_decision` xuất hiện ĐỒNG
      THỜI ở `gh issue comment` VÀ Slack thread, nội dung khớp nhau — ngoại trừ thông báo
      "Feature xong" lúc finalize là Slack-only (finalize hiện không gọi `gh issue comment` ở bước
      nào).
- [ ] Tin nhắn thường trong thread được relay đúng thành `gh issue comment`, kích hoạt route job
      hiện có (`issue_comment`) y như người dùng tự comment trên GitHub.
- [ ] `!approve` lúc spec PR đang mở → Worker post `/approve` lên đúng PR đó → PR merge qua route
      job cũ, không gọi merge trực tiếp.
- [ ] `!approve` lúc code PR đang mở → Worker gọi merge REST API trực tiếp → code PR merge thật,
      `finalize` trigger đúng (`pull_request: closed` không phân biệt actor).
- [ ] `!approve` gõ sai thời điểm (không có PR nào đang mở chờ approve) → Worker báo lỗi rõ trong
      thread, không làm gì trên GitHub.
- [ ] Finalize hoàn tất → message xác nhận xuất hiện trong thread.
- [ ] Signature Slack (signing secret) được verify cho MỌI request tới Worker; request không hợp
      lệ bị từ chối (401), không xử lý. `POST /relay` chỉ nhận request mang đúng
      `SLACK_RELAY_TOKEN`; sai token → 401, không post gì vào Slack.
- [ ] Mọi slash-command (`/new-feature`, `/new-project`, `/link-repo`) trả response trong <3 giây
      (ACK ngay, xử lý thật ở background qua `ctx.waitUntil`, gửi kết quả qua `response_url`); mọi
      tin nhắn Events API (relay, `!approve`) cũng ACK (200) trong <3 giây trước khi xử lý GitHub
      API ở background.
- [ ] Channel Slack private (sai yêu cầu) → `conversations.join` thất bại → Worker báo lỗi rõ ngay
      lần tương tác đầu, không âm thầm bỏ qua.
- [ ] Repo KHÔNG bật Slack integration (chưa chạy bước onboard, không có `SLACK_RELAY_TOKEN`) →
      mọi lời gọi `POST /relay` từ repo đó bị bỏ qua (secret trống → không gọi), toàn bộ hành vi
      `online-pipeline` hiện có không đổi (chỉ `gh issue comment` như cũ).
- [ ] Mọi edge case ở mục "Trường hợp biên đã đồng ý" trên hoạt động đúng như mô tả.

**Phần B:**
- [ ] `/new-project <tên-repo>` trong 1 channel public mới tạo → GitHub repo mới xuất hiện (private,
      account hiện tại), đã chạy `project-initialization` đầy đủ (main+dev, hooks), đã tự
      vendor+set-secret `online-pipeline` VÀ Slack (gồm `SLACK_RELAY_TOKEN`), Claude GitHub App đã
      cover (nhờ All-repositories), message xác nhận xuất hiện ĐÚNG channel (top-level, không phải
      reply thread) — sẵn sàng `/new-feature` ngay, không cần bước tay nào ở local.
- [ ] `/new-project` với tên repo đã tồn tại → `gh repo create` lỗi, job báo lỗi rõ trong channel,
      không tạo đè/tạo trùng.
- [ ] Bootstrap lỗi giữa chừng (sau khi repo đã tạo nhưng trước khi set secret xong) → báo lỗi rõ
      trong channel kèm bước nào đã xong/chưa, KHÔNG tự rollback (không xoá repo đã tạo).
- [ ] Job bootstrap chạy `project-initialization` không bao giờ bị kẹt chờ trả lời cho câu hỏi
      "origin đã tồn tại, có đúng repo không?" (vì không tự `gh repo create` trước khi gọi `claude
      -p` — để skill tự tạo repo ở Step 4 của chính nó khi `origin` chưa tồn tại).

## Constraints

- Cầu nối bắt buộc dùng Cloudflare Workers (free tier, đã có account) + Cloudflare KV — không dùng
  server luôn chạy khác. Worker là 1 instance DUY NHẤT phục vụ MỌI repo, deploy 1 lần/account,
  KHÔNG redeploy/vendor lại mỗi repo. Worker là nơi DUY NHẤT giữ Slack Bot Token thật; mọi repo
  khác chỉ giữ `SLACK_RELAY_TOKEN` (quyền hẹp, chỉ gọi được `POST /relay`).
- Bật Slack integration theo đơn vị: 1 lần cho hạ tầng account-wide (Worker, control-plane, Claude
  GitHub App), + 1 lần ngắn cho MỖI repo (onboard qua setup script + `/link-repo`, hoặc tự động qua
  `/new-project`).
- Không giới hạn user nào trong Slack workspace được trả lời/`!approve`/`/new-project` — quyết định
  tường minh của user, kể cả sau khi biết rõ `!approve` merge code trực tiếp và `/new-project` tiêu
  tốn credential quyền rộng toàn account. Rủi ro chỉ thực tế hoá nếu sau này có thêm người khác vào
  workspace.
- Không đổi `on:` event hay thêm workflow YAML mới trong 3 workflow hiện có của `online-pipeline`.
  Approve spec PR qua Slack giả lập đúng comment `/approve` hiện có; approve code PR qua Slack là
  Worker gọi REST API merge trực tiếp — cả 2 không cần sửa trigger GitHub Actions nào.
- Approve dùng literal `!approve` (không phải `/approve`, để không trùng cú pháp slash-command thật
  của Slack) — dùng CHUNG 1 từ khoá cho cả spec-approve và code-approve, Worker tự route theo status
  marker/PR đang mở của feature đó.
- Post song song GitHub + Slack ở MỌI điểm hiện tại đã có `gh issue comment` — ngoại lệ duy nhất:
  thông báo finalize là Slack-only (năng lực mới, không có điểm tương đương trên GitHub).
- Channel Slack cho mỗi project PHẢI là public channel (giới hạn kỹ thuật của
  `conversations.join`).
- Mọi slash-command và mọi message Events API phải ACK trong <3 giây (giới hạn cứng của Slack);
  xử lý GitHub API thật luôn ở background sau khi đã ACK.
- Repo điều phối (control-plane) + Worker dùng CHUNG 1 credential quyền rộng (GH_PAT tài khoản,
  Claude auth, Claude GitHub App "All repositories") cho MỌI repo hiện tại và tương lai — quyết
  định tường minh của user, đổi lấy "0 bước tay" cho mỗi project mới. Slack Bot Token thật KHÔNG
  nằm trong nhóm "dùng chung" này — chỉ Worker giữ, mọi repo khác chỉ có `SLACK_RELAY_TOKEN`.
- `project-initialization/SKILL.md` thêm 1 chế độ tự động gated bằng literal CỐ ĐỊNH
  `PROJECT_BOOTSTRAP_AUTOMATED` (không phải placeholder) — CHỈ bỏ "ask" ở ĐÚNG 1 điểm luôn có mặt
  bất kể hoàn cảnh (Step 4: "ask explicitly before pushing main"); không đổi gì khác trong skill,
  không áp dụng, không thay đổi gì khi gọi tương tác (local) như hiện tại.
- Job bootstrap PHẢI chạy trên working directory hoàn toàn trống (không checkout nội dung của
  chính repo điều phối) và KHÔNG tự `gh repo create` trước khi gọi `claude -p` — để
  `project-initialization` tự tạo repo ở Step 4 của chính nó (nhánh "origin chưa tồn tại"), tránh
  rơi vào nhánh "origin đã tồn tại, hỏi xác nhận" mà không ai trả lời được trong job tự động.
- Không đụng `code-writing/SKILL.md` hay `documentation-writing/SKILL.md`.

## Risks

- **Risk 1:** Literal `!approve` trong tin nhắn thường (không phải lệnh Slack thật) có thể bị gõ
  nhầm trong hội thoại thường, kích hoạt approve ngoài ý muốn. **Mitigation:** chấp nhận (nhất quán
  với `/switch-online` đã chấp nhận ở `hybrid-local-online-pipeline`), rủi ro thấp vì chỉ xảy ra
  trong thread riêng của 1 feature.
- **Risk 2:** Worker là thành phần public-facing DUY NHẤT (nhận webhook Slack) nhưng dùng CHUNG
  GH_PAT/Claude-GitHub-App "All repositories" cho MỌI repo hiện tại+tương lai. **Mitigation:** chấp
  nhận theo quyết định tường minh của user (đổi lấy đơn giản + "0 bước tay"), không xây cơ chế phân
  quyền hẹp hơn cho v1.
- **Risk 3:** Cloudflare Workers KV là nguồn sự thật DUY NHẤT cho mapping `channel_id<->repo` và
  `thread_ts<->issue`; nếu KV mất dữ liệu, toàn bộ feature/project đang chạy qua Slack mất khả năng
  relay — phải tương tác lại qua GitHub trực tiếp. **Mitigation:** chấp nhận hạn chế v1 (rủi ro hãn
  hữu).
- **Risk 3b:** Cloudflare Workers KV là eventually-consistent (ghi có thể mất tới ~60s để lan truyền
  khắp các edge location) — `/link-repo`/`/new-project` ghi mapping rồi `/new-feature` hoặc 1 tin
  nhắn trả lời gõ ngay sau đó (vài giây) có thể đọc thấy dữ liệu cũ/trống nếu rơi vào edge khác.
  **Mitigation:** chấp nhận là rủi ro hãn hữu v1 (đa số request của 1 user solo rơi vào cùng edge
  gần nhất); khi gặp lỗi "mapping not found" ngay sau khi vừa ghi, Worker báo lỗi kèm gợi ý "thử lại
  sau vài giây" — không xây cơ chế đợi/polling phức tạp hơn.
- **Risk 4:** Repo điều phối là 1 điểm tập trung quyền lực (giữ credential tạo repo cho toàn bộ
  account) — bị compromise tương đương compromise toàn bộ account GitHub. **Mitigation:** giữ repo
  điều phối private, không thêm collaborator ngoài owner.
- **Risk 5:** KHÔNG giới hạn ai trong Slack workspace được `!approve`/`/new-project`. **Mitigation:**
  chấp nhận v1 vì workspace hiện chỉ có user; rủi ro chỉ thực tế hoá nếu có thêm người khác.
- **Risk 6:** `project-initialization/SKILL.md` vốn có "ask before push main" làm lưới an toàn cuối
  cùng trước 1 hành động không thể hoàn tác dễ dàng (push lên GitHub công khai). Chế độ tự động bỏ
  qua đúng điểm này. **Mitigation:** job bootstrap LUÔN chạy trên working directory hoàn toàn mới,
  không tự `gh repo create` trước (xem Constraints) — tự thoả mọi điều kiện tiên quyết khác của
  skill (Step 1, Step 4 nhánh dev) nên các "ask" đó tự nhiên không kích hoạt, không cần bỏ qua gì
  thêm ngoài đúng 1 điểm đã nêu; bản thân lệnh Slack `/new-project` đã là sự xác nhận ý định của
  user trước khi job này chạy.
- **Risk 7:** Slack Events API có thể gửi lại (retry) 1 message nếu Worker không ACK kịp, có thể
  gây xử lý trùng 1 tin nhắn 2 lần (post lặp 1 `gh issue comment`). **Mitigation:** ACK ngay (200
  trống) TRƯỚC khi xử lý GitHub API giảm hầu hết nguy cơ; retry hiếm vẫn xảy ra (vd Worker crash
  giữa xử lý) được chấp nhận là rủi ro hãn hữu v1, không xây de-dup theo `event_id`.
- **Risk 8:** Đây là lần đầu repo có Cloudflare Worker/KV/serverless — thêm 1 hệ thống vận hành mới
  (deploy, secrets, theo dõi lỗi) ngoài phạm vi GitHub Actions quen thuộc. **Mitigation:** chấp nhận,
  Worker source đặt trong repo điều phối để dễ theo dõi, không rải ra nhiều nơi.

## Accepted Decisions

- Mapping `thread<->feature`: nhúng `channel_id`+`thread_ts` vào BODY issue lúc tạo (Actions đọc
  trực tiếp, không round-trip); chiều ngược (Slack event → issue) lưu Cloudflare KV do Worker tự
  ghi lúc tạo issue. Từ chối phương án file commit riêng (sidecar) vì tạo thêm round-trip không
  cần thiết cho phía Worker.
- Mapping `channel<->repo` (phát hiện thiếu ở vòng kiểm tra sau): lưu trong CHÍNH Cloudflare KV đã
  dùng. Ghi bởi Worker tự động lúc `/new-project`, hoặc qua lệnh mới `/link-repo <owner>/<name>`
  cho repo đã tồn tại từ trước — không cần endpoint/HTTP mới, tái dùng đúng pathway slash-command.
- Approve dùng literal `!approve` (không `/approve`) để tránh trùng cú pháp slash-command thật của
  Slack — dùng CHUNG 1 từ khoá cho cả spec-approve và code-approve, Worker tự route theo status
  marker/PR đang mở (không cần 2 từ khoá riêng).
- Approve spec PR qua Slack: Worker GIẢ LẬP lại đúng hành vi người dùng (post comment `/approve`
  lên PR) để tái dùng 100% route job cũ, không gọi merge trực tiếp.
- Approve code PR qua Slack: Worker gọi merge REST API TRỰC TIẾP (năng lực mới) — xác nhận qua code
  research không cần workflow/trigger mới vì `pull_request: closed` không phân biệt actor.
- Post song song GitHub + Slack luôn luôn (không có cờ chọn 1 trong 2), để giữ GitHub làm nguồn sự
  thật không đổi — ngoại lệ duy nhất là thông báo finalize (Slack-only, không có điểm GitHub tương
  đương).
- Worker dùng chung `GH_PAT`/Claude-auth/Claude-GitHub-App với toàn bộ account (không tạo
  token/App riêng theo repo) — quyết định của user, đổi lại đơn giản hơn, đánh đổi blast-radius cao
  hơn nếu secret Worker lộ (xem Risks).
- Worker là 1 instance DUY NHẤT phục vụ MỌI repo, deploy 1 lần/account (sửa lại sau khi phát hiện
  mâu thuẫn với lời hứa "0 bước tay" của `/new-project` — nếu Worker phải deploy lại mỗi repo thì
  lời hứa đó không thể giữ được) — source đặt trong repo điều phối, không vendor vào
  `skills/online-pipeline/assets/`.
- Bootstrap project mới KHÔNG tự chế logic tạo repo bằng Worker — Worker chỉ bắn
  `repository_dispatch` tới 1 workflow Actions THẬT có `gh` CLI sẵn trong repo điều phối. Lý do:
  Cloudflare Worker không có git/gh CLI; dùng `gh` CLI qua Actions tái dùng đúng công cụ đã chứng
  minh hoạt động trong `online-pipeline`, tránh phải tự hiện thực mã hoá secret (sealed-box) trong
  Worker.
- Chế độ tự động của `project-initialization` CHỈ bỏ "ask" ở ĐÚNG 1 điểm (push main) — sửa lại sau
  validation round 1 (phát hiện của skeptic): mô tả ban đầu sai cơ chế thật của Step 1/4/5 (Step 1
  ask chỉ kích hoạt khi "đã là git repo + có uncommitted changes", không phải "thư mục không
  trống"; Step 4 có 2 nhánh loại trừ nhau theo origin tồn tại hay không, bootstrap phải đi đúng
  nhánh "origin chưa tồn tại" bằng cách không tự tạo repo trước; Step 5 không có ask nào, chỉ cần
  đổi đích report). Sửa bằng cách đổi thứ tự: để skill tự `gh repo create` ở Step 4 của chính nó
  (name đã cung cấp sẵn), không tự tạo repo trước khi gọi `claude -p`.
- Bot Slack chỉ cần `chat:write`+`channels:history`+`channels:join` (sửa lại sau khi phát hiện
  `chat:write` một mình không đủ để NHẬN tin nhắn) — Worker tự `conversations.join` channel ngay
  khi nhận slash-command đầu tiên, không cần user tự tay invite bot; không cần `channels:manage`
  vì channel luôn tạo tay (không bao giờ do bot tạo).
- Channel Slack cho mỗi project PHẢI là public (sửa lại sau khi phát hiện `conversations.join`
  không hoạt động với private channel) — ghi rõ thành constraint, không xây fallback cho private
  channel ở v1.
- Mọi slash-command và Events API message đều ACK ngay (<3s) rồi xử lý thật ở background (sửa lại
  sau khi phát hiện công việc thật — gọi GitHub API, ghi KV — vượt quá giới hạn 3 giây cứng của
  Slack); kết quả thật gửi qua `response_url` (slash-command) hoặc `chat.postMessage`/`POST /relay`
  (Events API).
- GitHub Actions job của mỗi repo gọi `POST /relay` trên Worker (KHÔNG gọi Slack API trực tiếp) —
  sửa lại sau validation round 1 (phát hiện của adequacy): phương án ban đầu để mỗi repo tự giữ
  Slack Bot Token thật đã rải token quyền đầy đủ ra mọi repo, không tận dụng đúng vai trò "cầu nối
  duy nhất" đã định cho Worker. Đổi sang: Worker là nơi DUY NHẤT giữ token Slack thật; mỗi repo chỉ
  giữ `SLACK_RELAY_TOKEN` (quyền hẹp, chỉ gọi được 1 endpoint relay) — giảm thiệt hại nếu secret
  của 1 repo con bị lộ.
- Giữ CHUNG 1 user-spec cho cả Phần A và Phần B (user xác nhận không tách), dù về nguyên tắc 2 mảng
  tách được thành 2 giá trị độc lập — vì user ưu tiên làm 1 lần.
- Không giới hạn ai trong Slack workspace được `!approve`/`/new-project` — user xác nhận lại quyết
  định này sau khi được nhắc rõ 2 năng lực mạnh này tồn tại, không chỉ là quyết định ban đầu lúc
  phạm vi còn nhẹ.
- Test `/new-project` lần đầu ngay trên account GitHub chính của user (không dùng account/workspace
  demo riêng) — user tự chấp nhận rủi ro tạo repo thật nếu có lỗi lúc test lần đầu.

## Testing

**Unit tests:** Cloudflare Worker LÀ code mới (JS/TS) — các hàm thuần (verify Slack signing secret,
verify `SLACK_RELAY_TOKEN` cho `/relay`, parse payload, xác định route `!approve` theo status
marker/PR đang mở, build GitHub API request, đọc/ghi mapping KV) nên có unit test. Phần còn lại
(mở rộng `SKILL.md`/workflow YAML, chế độ tự động của `project-initialization`) là văn bản điều
phối thuần, không có logic cô lập được — không cần unit test, giống tiền lệ
`hybrid-local-online-pipeline`.

**Integration tests:** không cần dạng tự động — phần cần kiểm tra là tương tác giữa Slack API,
Cloudflare Workers/KV, GitHub Actions, GitHub API, và các skill Claude Code hiện có, không phải
logic có thể cô lập bằng mock hợp lý.

**E2E tests:** cần, thực hiện bằng chạy thử thủ công trên workspace Slack + repo/account GitHub
thật (không phải bộ test tự động trong CI) — xem chi tiết ở Verification.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Đọc `online-pipeline/SKILL.md` sau khi sửa | Có điểm chèn gọi `POST /relay` song song với mọi `gh issue comment` hiện có (trừ finalize); literal `!approve` định nghĩa rõ, route theo status marker/PR đang mở; không đổi `on:` event của 3 workflow YAML hiện có |
| 2. Đọc `project-initialization/SKILL.md` sau khi sửa | Chế độ tự động gated bằng literal cố định `PROJECT_BOOTSTRAP_AUTOMATED`; CHỈ bỏ ĐÚNG 1 "ask" ("ask explicitly before pushing main" cuối Step 4) — Step 1/Step 4-dev-branch không bị sửa gì (ask của chúng tự nhiên không kích hoạt trong bối cảnh bootstrap); Step 5 chỉ đổi đích report sang `POST /relay`, không phải bỏ "ask"; không đổi hành vi khi KHÔNG có signal này |
| 3. Đọc workflow `bootstrap-project.yml` | KHÔNG có bước `gh repo create` TRƯỚC khi gọi `claude -p` — việc tạo repo để nguyên cho `project-initialization` Step 4 tự làm; working directory cho `claude -p` hoàn toàn trống (không checkout nội dung repo điều phối) |
| 4. Đọc Worker source trong repo điều phối | Có verify Slack signing secret cho request từ Slack; có `POST /relay` riêng xác thực bằng `SLACK_RELAY_TOKEN` (không phải signing secret Slack); chỉ Worker gọi `chat.postMessage` bằng Slack Bot Token thật, không repo nào khác có token này; có xử lý ACK-ngay-rồi-background cho cả slash-command (`response_url`) và Events API; có `conversations.join` lúc nhận lệnh đầu tiên trong channel; đọc/ghi đúng 2 mapping KV (`channel_id<->repo`, `thread_ts<->issue`) |
| 5. Đọc `bootstrap-project.yml` tiếp | Nhận `repository_dispatch` đúng payload (`channel_id`, tên repo, KHÔNG có `thread_ts`); gọi `claude -p` với signal `PROJECT_BOOTSTRAP_AUTOMATED`; sau đó tự vendor+set-secret (`GH_PAT`, Claude auth, `SLACK_RELAY_TOKEN`) cho repo mới; không tự rollback khi lỗi giữa chừng |
| 6. Đọc `setup-online-pipeline.sh` sau khi mở rộng | Có thêm bước set secret `SLACK_RELAY_TOKEN` (không phải Slack Bot Token thật) cho repo được onboard, không chỉ `GH_PAT`/Claude-auth như cũ; online-pipeline/SKILL.md có kiểm tra secret trống trước khi gọi `/relay`, không lỗi nếu repo chưa onboard Slack |
| 7. Kiểm tra `scripts/init-feature-folder.sh` | Chữ ký không đổi |
| 8. Đọc đoạn approve code PR trong Worker/SKILL.md | Gọi REST API merge trực tiếp, không qua comment; xác nhận tài liệu ghi rõ không cần workflow/trigger GitHub Actions mới |
| 9. Đọc đoạn xử lý edge case trong Worker/SKILL.md | Có đủ các nhánh: channel chưa link, tin nhắn rời không thuộc thread nào, tin nhắn cho thread đã finalize, double-submit `!approve`, slash command trùng lặp, `/link-repo` ghi đè |

Agent chỉ kiểm tra được các mục tĩnh trên (file đúng chỗ, cú pháp hợp lệ, điểm chèn đúng). Agent
không thể tự tạo Slack App/channel thật, tạo GitHub repo thật, hay chờ Cloudflare Worker/GitHub
Actions chạy thật — nên toàn bộ hành vi vận hành thật (Phần A và Phần B) chỉ được xác minh ở User
Verification dưới đây.

### User Verification

Người dùng tự thực hiện trên workspace Slack + repo/account GitHub thật — lý do cần xác minh thủ
công: đây là luồng tương tác người-Slack-GitHub-Cloudflare nhiều lượt thật, không mô phỏng được
bằng agent chạy 1 lần.

**Phần A (trên 1 repo đã bật online-pipeline từ trước):**
1. Setup 1 lần cho account: tạo Slack App, deploy Worker (kèm `/relay` endpoint +
   `SLACK_RELAY_TOKEN`), tạo repo điều phối, cài Claude GitHub App "All repositories".
2. Onboard 1 repo có sẵn: chạy setup script mở rộng, tạo channel public, gõ `/link-repo` → xác
   nhận Worker tự join channel, mapping ghi đúng vào KV.
3. Gõ `/new-feature ...` → issue + thread mới xuất hiện, câu hỏi đầu tiên post cả 2 nơi.
4. Trả lời vài câu bằng tin nhắn thường trong thread → interview tiến triển đúng, không trùng lặp
   do retry Events API.
5. Spec PR mở, gõ `!approve` trong thread → spec PR merge thật, implement bắt đầu.
6. Lúc `awaiting_decision`, trả lời trong thread → implement tiếp tục đúng.
7. Code PR mở, gõ `!approve` trong thread → code PR merge thật qua REST API trực tiếp, finalize
   chạy.
8. Finalize xong → message "Feature xong" xuất hiện trong thread.
9. Cố ý gõ `!approve` khi không có PR nào chờ → báo lỗi rõ, không làm gì trên GitHub.
10. Cố ý tạo 1 channel PRIVATE rồi `/link-repo` → xác nhận báo lỗi rõ (`conversations.join` thất
    bại), không âm thầm treo.
11. Cố ý gửi request Slack với signature sai tới Worker → xác nhận bị từ chối (401); cố ý gọi
    `POST /relay` với `SLACK_RELAY_TOKEN` sai → xác nhận bị từ chối (401), không post vào Slack.
12. Gõ `/new-feature` hoặc nhắn thường trong 1 channel CHƯA `/link-repo` → xác nhận báo lỗi rõ.
13. Nhắn 1 tin nhắn thường KHÔNG reply vào thread nào trong channel đã link → xác nhận bị bỏ qua
    im lặng, không có comment rác nào xuất hiện trên GitHub.
14. Gõ `!approve` 2 lần liên tiếp thật nhanh → xác nhận không crash, chấp nhận hành vi
    last-write-wins (không cần merge 2 lần thành công riêng biệt).
15. Trên repo CHƯA chạy bước onboard (không có `SLACK_RELAY_TOKEN`) → xác nhận `online-pipeline`
    vẫn hoạt động như trước khi có feature này (chỉ `gh issue comment`, không lỗi gì liên quan
    Slack).

**Phần B (khởi tạo project mới):**
16. Tạo 1 channel Slack public mới, gõ `/new-project <tên-repo>` → xác nhận: repo GitHub mới xuất
    hiện đúng tên/private/account, có main+dev+hooks, đã vendor online-pipeline+Slack, secret đã
    set đủ (`GH_PAT`, Claude auth, `SLACK_RELAY_TOKEN`), Claude GitHub App đã cover, message xác
    nhận xuất hiện ĐÚNG channel (top-level), job KHÔNG bị kẹt chờ câu hỏi xác nhận nào.
17. Gõ `/new-feature ...` ngay trong channel vừa tạo ở bước 16 → chạy được ngay, không cần thêm
    bước tay nào ở local (lặp lại toàn bộ chu trình Phần A từ bước 3).
18. Gõ `/new-project` với tên repo đã tồn tại → báo lỗi rõ trong channel, không tạo đè.
19. Cố ý làm lỗi job bootstrap giữa chừng (vd revoke tạm thời quyền secret) → xác nhận báo lỗi rõ
    kèm bước đã xong/chưa, không tự rollback repo đã tạo.
20. Gõ `/new-project` lần 2 trong channel đã map với project A (vi phạm quy ước 1-channel-1-project)
    → xác nhận hệ thống không chặn (chạy vẫn được), chỉ là người dùng tự gây nhiễu quy ước.
