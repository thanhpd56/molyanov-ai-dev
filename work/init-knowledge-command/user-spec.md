---
# Creation date (YYYY-MM-DD)
created: 2026-10-04

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: init-knowledge-command

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.
>
> **Lưu ý thực thi đặc biệt:** feature này có 2 phần nằm ở 2 repo khác nhau — xem mục "Accepted
> Decisions" và "Expected Behavior" Phần B. Phần A nằm trong repo hiện tại (`molyanov-ai-dev`).
> Phần B nằm trong repo riêng `github.com/thanhpd56/control-plane` (chưa được clone trong máy này)
> — phải `git clone` repo đó sang 1 thư mục riêng để thực thi Phần B, không thực thi trong thư mục
> của repo này.

## What We Are Building

Thêm `/init-knowledge` — cụm từ kích hoạt khởi tạo Project Knowledge ban đầu
(`create-project-knowledge.md`), dùng được cả local (interactive) và tự động ngay sau khi
`/new-project` bootstrap thành công qua Slack, không cần clone repo về máy. Gồm 2 phần: (A) trong
repo này — thêm trigger phrase `/init-knowledge` cho `documentation-writing`, và 1 stage
`knowledge-init` mới cho `online-pipeline` (chạy qua GitHub Actions, theo turn, commit thẳng
`main`, tự đóng issue khi xong); (B) trong repo `control-plane` — tự tạo issue `knowledge-init`
ngay sau khi `bootstrap-project.yml` vendor secret xong, và 1 Slack slash command thủ công làm
phương án dự phòng/mở rộng cho repo cũ chưa có Project Knowledge.

## Why

Hiện tại sau khi `/new-project` tạo repo mới xong qua Slack, không có cách nào khởi tạo Project
Knowledge cho repo đó mà không phải clone về máy và tự chạy `documentation-writing` — đi ngược lại
lời hứa "0 bước tay ở local" mà `/new-project` đã có cho việc tạo repo. `/init-knowledge` đóng nốt
khoảng trống này: ngay sau khi repo mới sẵn sàng, 1 issue tự xuất hiện và luồng hỏi-đáp bắt đầu
ngay trên Slack/GitHub.

## Expected Behavior

**Phần A — trong `molyanov-ai-dev`:**

1. Interactive local: gõ "/init-knowledge" trong 1 session local trên project đang thiếu/template/
   partial Project Knowledge → `documentation-writing/SKILL.md` nhận trigger phrase mới, chạy
   `create-project-knowledge.md`'s Phase 0 ngay — giống cách `/done` hiện trỏ thẳng tới Feature
   Finalization Mode.
2. `online-pipeline` có thêm 1 stage mới tên `knowledge-init`, chạy bởi 1 workflow file mới
   `online-pipeline-knowledge-init.yml` (triggered bởi `issues: opened` / `issue_comment: created`
   lọc theo label `knowledge-init`, cùng pattern với `online-pipeline-userspec.yml`):
   - Mỗi lần có comment mới trên issue, 1 lần `claude -p` chạy `create-project-knowledge.md` tiếp
     tục đúng từ trạng thái đã lưu trong `work/project-knowledge/interview.yml`.
   - Mọi điểm skill "hỏi user" — câu hỏi interview, hoặc 1 trong 3 checkpoint "user đồng ý" cuối
     mỗi cycle — post `gh issue comment` + mirror qua `relay_to_slack` (nếu repo có Slack secrets),
     rồi dừng (không có literal signal tự động nào để skip checkpoint).
   - Không dùng branch riêng ở bất kỳ turn nào. Mỗi turn commit `work/project-knowledge/
     interview.yml` thẳng vào `main` để giữ trạng thái qua các lần chạy riêng biệt (mỗi lần chạy
     Actions là 1 checkout mới, không có ổ đĩa liên tục). Turn cuối cùng (sau khi hoàn tất
     "Write the Documentation") thêm commit các file Project Knowledge thật, rồi `gh issue close`.
   - Stage bỏ qua hẳn bước "Return to Documentation Review in the main skill" của
     `create-project-knowledge.md` — tự dừng ngay sau commit + đóng issue, không quay lại menu
     chính của `documentation-writing`.
   - Lớp check THẬT, có tính quyết định (khác với check "rẻ, best-effort" của Worker ở Phần B —
     stage này chỉ chạy khi ĐÃ có 1 issue `knowledge-init` tồn tại): turn đầu tiên luôn chạy
     đúng Phase 0 của `create-project-knowledge.md` — bước này đã có sẵn "Inspect the repository,
     configuration, CLAUDE.md, and current Project Knowledge" để quyết định resume/bắt đầu mới.
     Mở rộng thêm 1 kết luận cho ngữ cảnh automated này: nếu qua inspect thực tế (không chỉ dựa
     vào field `interview_metadata.status`), Project Knowledge đã KHÔNG còn thiếu/template/
     partial — dù do chính flow này hoàn tất trước đó, do mode "full update/audit" của
     `documentation-writing`, hay do sửa tay trực tiếp — stage không chạy interview: chỉ post
     "đã khởi tạo rồi" rồi đóng issue ngay. Đây là lớp chặn chính xác nhất, bắt được MỌI trường
     hợp "đã hoàn chỉnh" chứ không chỉ trường hợp field status đã set (cái mà Worker's check ở
     Phần B chỉ bắt được 1 phần). Cũng áp dụng cho issue bị reopen hoặc 1 comment lạc vào sau khi
     đã đóng.
   - `online-pipeline-userspec.yml`'s route job thêm 1 điều kiện bỏ qua issue/comment mang label
     `knowledge-init`, cùng cách nó đang bỏ qua `local-placeholder`.
   - Lỗi/crash: theo đúng pattern retry đã có (tối đa 2 lần retry, rồi báo lỗi qua `gh issue
     comment` + `relay_to_slack`, không rollback, issue vẫn mở).
   - Workflow mới dùng chung concurrency group cố định với `finalize`
     (`online-pipeline-finalize` — không tạo group riêng cho `knowledge-init`), vì cả hai cùng
     commit thẳng vào `main` và có thể cùng động tới Project Knowledge: nếu 1 feature khác được
     merge (kích hoạt `finalize`) đúng lúc interview `knowledge-init` đang chạy trên cùng repo mới
     bootstrap, GitHub Actions phải serialize 2 run này lại, không cho chạy song song.
3. Workflow mới này vendor vào 2 vị trí trong scaffold của `project-initialization` —
   `.github/workflows/` và `.claude/skills/online-pipeline/assets/workflows/` — giống 3 workflow
   hiện có; `vendor-skills.sh`'s `WORKFLOWS` array thêm file mới. `setup-online-pipeline.sh`'s
   block tạo label trước (eager) cũng thêm `knowledge-init`, đúng pattern đã áp dụng cho 3 label
   hiện có; label còn được tạo inline defensive ngay tại 2 điểm thực sự áp dụng nó —
   `bootstrap-project.yml` cho đường tự động (repo mới), Worker's slash-command handler cho đường
   thủ công (repo cũ, sau khi đã re-vendor).

**Phần B — trong repo riêng `control-plane` (cần clone riêng để thực thi):**

4. `bootstrap-project.yml`: ngay sau bước vendor + set secret (`GH_PAT`, Claude auth,
   `SLACK_RELAY_TOKEN`) cho repo mới thành công (bước đã có sẵn, không đổi) — thêm bước mới:
   - Gọi `/relay` để post 1 message mới (không có `thread_ts`) vào `channel_id` đã biết từ
     `repository_dispatch` payload — message này trở thành gốc của 1 thread Slack mới; lấy `ts`
     của message đó làm `thread_ts` (chi tiết API `/relay` có trả `ts` sẵn hay cần sửa Worker để
     trả — để lại làm quyết định kỹ thuật lúc code bên `control-plane`, không chốt cứng ở đây).
   - Tạo label `knowledge-init` (defensive) rồi `gh issue create` trên repo mới với label đó, body
     nhúng dòng marker `<!-- slack: channel_id=... thread_ts=... -->` dùng `thread_ts` vừa lấy.
   - Issue này được workflow `online-pipeline-knowledge-init.yml` của repo mới (Phần A) tự nhận và
     chạy `claude -p` ngay turn đầu tiên.
   - Nếu lời gọi `/relay` để mở message Slack đầu tiên thất bại: `bootstrap-project.yml` vẫn tạo
     issue `knowledge-init` như thường (phía GitHub không được phép phụ thuộc vào việc Slack có
     thành công hay không) — chỉ là body issue sẽ không có dòng marker Slack. `relay_to_slack` ở
     các turn sau đọc thấy không có marker thì tự no-op, đúng như hành vi hiện tại của 1 repo chưa
     onboard Slack.
5. Worker thêm 1 slash command Slack mới (tên làm việc: `/init-knowledge`) dùng cho bất kỳ channel
   đã link với 1 repo (mới hoặc cũ), miễn Project Knowledge của repo đó còn thiếu/template/
   partial — cùng pattern với `/new-feature` (ACK <3s, tạo label `knowledge-init` defensively nếu
   repo đó chưa có, mở thread trước để có `thread_ts`, tạo issue `knowledge-init` với marker, relay
   câu hỏi đầu tiên). Trước khi làm bất kỳ hành động nào trên GitHub, Worker làm 2 check rẻ, có
   tính best-effort (không phải nguồn sự thật duy nhất — xem lớp check thật ở Phần A item 2 bên
   dưới):
   - Nếu repo đã có `interview_metadata.status: completed` (đọc qua GitHub API) → **không tạo
     issue nào cả, không mở thread nào cả** — chỉ trả lời ngay qua Slack (response/thread hiện
     có) "đã khởi tạo rồi". Check này chỉ bắt được đúng trường hợp Project Knowledge được hoàn
     tất CHÍNH bằng flow này trước đó; 1 repo cũ có Project Knowledge đầy đủ qua đường khác (mode
     "full update/audit" của `documentation-writing`, hoặc sửa tay) sẽ không có field này —
     trường hợp đó vẫn tạo issue, nhưng lớp check thật ở Phần A item 2 (Phase 0 của
     `create-project-knowledge.md` tự inspect repo) sẽ phát hiện và tự đóng ngay, không chạy
     interview. Worker's check ở đây chỉ là tối ưu để tránh tạo issue/thread không cần thiết cho
     trường hợp phổ biến nhất, không phải điều kiện bắt buộc đúng 100%.
   - Nếu repo đã có 1 issue `knowledge-init` đang mở (interview đang chạy) → **không tạo issue/
     thread thứ 2** — chỉ trả lời ngay qua Slack "đang chạy rồi, xem issue #{number}".
   - Cả 2 check trên là "check rồi mới act", có 1 khoảng hở nhỏ nếu gọi lệnh 2 lần gần như đồng
     thời (race) — chấp nhận như 1 rủi ro hiếm, cùng mức với các race hiếm khác đã được chấp nhận
     trong `slack-pipeline-interaction` (xem Risks).
   Command này KHÔNG phải công cụ audit lại Project Knowledge đã hoàn chỉnh — nó chỉ chạy đúng
   `create-project-knowledge.md`, không chạy mode "full update/audit" của `documentation-writing`.
   **Điều kiện tiên quyết cho repo cũ:** repo đó phải đã re-vendor `online-pipeline` (chạy lại
   `vendor-skills.sh`/`setup-online-pipeline.sh`) SAU KHI feature này merge, để có workflow
   `online-pipeline-knowledge-init.yml` + điều kiện bỏ qua label `knowledge-init` ở
   `online-pipeline-userspec.yml`'s route job. Đây không phải yêu cầu mới — đúng quy ước
   maintenance có sẵn của `online-pipeline` (mọi thay đổi ở `assets/workflows/*.yml` đều cần
   repo cũ tự re-vendor để nhận, xem "Setup Script Maintenance" trong `online-pipeline/SKILL.md`).
   Nếu repo cũ chưa re-vendor: issue vẫn tạo được bên Slack/GitHub, nhưng không có workflow nào
   nhận xử lý nó, và route job cũ (chưa có điều kiện bỏ qua) có thể hiểu nhầm đây là issue
   `/new-feature` rồi tự chạy `user-spec-planning` — command này KHÔNG tự kiểm tra hay cảnh báo
   trước việc repo đã re-vendor hay chưa (ngoài phạm vi v1).

## Acceptance Criteria

- [ ] Gõ "/init-knowledge" trong session local (project thiếu/template/partial Project Knowledge)
  → `create-project-knowledge.md`'s Phase 0 chạy ngay, không cần thêm bước routing nào.
- [ ] Sau 1 lần `/new-project` thật qua Slack (repo mới + secrets đã vendor xong): 1 issue label
  `knowledge-init` tự xuất hiện trên repo mới; body issue có marker Slack với `thread_ts` thật; câu
  hỏi interview đầu tiên xuất hiện đồng thời ở `gh issue comment` và 1 Slack thread mới.
- [ ] Trả lời trên Slack hoặc comment trên GitHub đều tiếp tục đúng interview, kể cả 3 checkpoint
  cuối mỗi cycle.
- [ ] Hoàn tất interview → Project Knowledge được commit thẳng vào `main` (không qua PR) → issue
  `knowledge-init` tự đóng.
- [ ] `online-pipeline-userspec.yml`'s route job không phản ứng với issue/comment mang label
  `knowledge-init` (không chạy trùng 2 flow).
- [ ] Repo cũ (không phải vừa bootstrap) không bao giờ bị trigger tự động — chỉ trigger qua Slack
  slash command thủ công.
- [ ] Slash command Slack chạy được trên repo cũ còn thiếu/template/partial Project Knowledge,
  đúng luồng như trên.
- [ ] Slash command Slack gọi trên repo mà `interview_metadata.status` đã là `completed` → Worker
  không tạo issue/thread nào, chỉ trả lời ngay qua Slack "đã khởi tạo rồi", không báo lỗi.
- [ ] Slash command Slack gọi trên repo cũ có Project Knowledge đã đầy đủ qua đường KHÁC (mode
  "full update/audit", hoặc sửa tay — không có `interview_metadata.status: completed`) → Worker
  tạo issue như thường (check của Worker không bắt được trường hợp này), nhưng turn đầu tiên của
  stage `knowledge-init` tự inspect (Phase 0 của `create-project-knowledge.md`) phát hiện đã đầy
  đủ, không chạy interview — chỉ post "đã khởi tạo rồi" rồi đóng issue ngay.
- [ ] Slash command Slack gọi lần 2 khi đã có issue `knowledge-init` đang mở cho repo đó → Worker
  không tạo issue/thread thứ 2, chỉ trả lời ngay qua Slack rằng issue đó đang mở.
- [ ] Nếu `/relay` lỗi lúc `bootstrap-project.yml` mở message Slack đầu tiên: issue `knowledge-init`
  vẫn được tạo bình thường trên GitHub (không phụ thuộc Slack thành công), chỉ thiếu marker Slack
  trong body — các lần `relay_to_slack` sau no-op, interview vẫn tiếp tục đúng qua GitHub comment.
- [ ] `claude -p` lỗi/crash trong stage `knowledge-init` → retry tối đa 2 lần, rồi báo lỗi qua
  `gh issue comment` + Slack, issue vẫn mở, không rollback. (Xác nhận bằng cách đọc workflow YAML:
  cấu trúc retry-loop phải khớp với `online-pipeline-implement.yml`/`online-pipeline-finalize.yml`
  hiện có — xem Agent Verification bên dưới; không cần tạo crash thật để kiểm tra việc này.)
- [ ] Workflow `online-pipeline-knowledge-init.yml` dùng chung concurrency group
  `online-pipeline-finalize` với `finalize` — 2 run không chạy song song trên cùng repo.
- [ ] Trên 1 repo cũ ĐÃ re-vendor `online-pipeline` sau khi feature này merge: slash command Slack
  chạy đúng luồng knowledge-init, không bị `online-pipeline-userspec.yml` route nhầm thành feature
  thường.

## Constraints

- Trigger tự động (issue tự tạo ngay sau bootstrap) chỉ áp dụng cho repo mới — gắn liền với sự
  kiện bootstrap, chỉ xảy ra 1 lần/repo.
- Slash command Slack thủ công bỏ giới hạn thời điểm "phải vừa mới bootstrap", nhưng giữ đúng điều
  kiện gốc của `create-project-knowledge.md` (Project Knowledge còn thiếu/template/partial) — không
  biến thành lệnh "chạy documentation-writing theo yêu cầu" chung, không kích hoạt mode "full
  update/audit".
- Không dùng branch riêng ở bất kỳ đâu trong flow này: mỗi turn commit thẳng `main`; không mở PR.
- Workflow mới dùng chung concurrency group cố định `online-pipeline-finalize` với `finalize` (không
  tạo group riêng cho `knowledge-init`) — cả hai cùng commit thẳng Project Knowledge vào `main`,
  nên phải serialize với nhau, không chỉ serialize với chính mình.
- Mỗi turn: commit+push thẳng `main` phải là hành động CUỐI CÙNG của lần chạy đó, không có bước
  nào chạy sau — đúng luật `userspec-turn`/`finalize` đã áp dụng (viết/commit state trước khi
  exit, không bao giờ sau khi push). Lưu ý: `implement`'s nhánh `awaiting_decision` làm ngược lại
  (post comment/relay SAU khi đã commit+push) — `knowledge-init` cố tình theo đúng thứ tự của
  `userspec-turn`/`finalize`, không theo `implement`, vì thứ tự này mới đảm bảo retry không bao
  giờ xảy ra sau khi 1 turn đã commit xong (tránh nguy cơ hỏi lại/trả lời trùng câu hỏi).
- Slash command Slack trên repo cũ chỉ hoạt động đúng nếu repo đó đã re-vendor `online-pipeline`
  sau khi feature này merge (xem Expected Behavior Phần B item 5) — không phải yêu cầu mới, chỉ là
  quy ước maintenance có sẵn được áp dụng cho đúng.
- Label `knowledge-init` phân biệt issue này với issue `/new-feature`; `online-pipeline-userspec.
  yml`'s route job phải bỏ qua issue/comment mang label này.
- Mọi checkpoint của `create-project-knowledge.md` (kể cả 3 checkpoint cuối cycle) đều round-trip
  qua `gh issue comment`/Slack reply như 1 câu hỏi bình thường — không thêm literal signal tự động
  nào để skip checkpoint.
- `create-project-knowledge.md`'s bước cuối "Return to Documentation Review in the main skill" bị
  bỏ qua trong ngữ cảnh automated — stage tự dừng sau khi commit + đóng issue.
- Marker hoàn tất/idempotency: dùng field có sẵn `interview_metadata.status: completed`, không phát
  sinh marker mới.
- Lỗi theo đúng pattern 2-lần-retry-rồi-báo-lỗi đã có, không rollback, issue vẫn mở khi lỗi.
- Không thêm secret mới nào ngoài các secret `slack-pipeline-interaction` đã vendor sẵn (`GH_PAT`,
  Claude auth, `SLACK_RELAY_TOKEN`/`SLACK_WORKER_URL`).

## Risks

- **Risk 1:** Feature chia 2 repo — phần control-plane không thể implement/test từ working
  directory hiện tại (`control-plane` chưa clone, source chỉ tồn tại ở repo riêng
  `github.com/thanhpd56/control-plane`). **Mitigation:** user-spec mô tả rõ cả 2 phần; lúc thực
  thi phải `git clone` repo đó sang thư mục riêng làm bước tách biệt, không trộn vào thư mục của
  repo này.
- **Risk 2:** `create-project-knowledge.md` chưa từng được thiết kế để chạy theo turn qua GitHub
  Actions (khác hẳn `user-spec-planning` đã có sẵn nhánh `ONLINE_PIPELINE_AUTOMATED`) — đây là địa
  hình mới, không có tiền lệ y hệt để copy. **Mitigation:** không phát sinh signal tự động mới;
  mọi checkpoint chỉ đơn giản trở thành 1 câu hỏi round-trip bình thường — cơ chế generic đang dùng
  cho `userspec-turn` đã đủ xử lý việc này.
- **Risk 3:** `relay_to_slack` hiện không có 1 script chung (bị copy-paste bash ở 4+ nơi) — feature
  này thêm ít nhất 1 bản copy nữa thay vì sửa nợ kỹ thuật đó. **Mitigation:** chấp nhận, nằm ngoài
  phạm vi feature này; không refactor "tiện tay".
- **Risk 4:** Thiếu rõ hành vi của API `/relay` (có trả `ts` của message mới hay không) khi chưa
  clone được `control-plane` để xem source thật. **Mitigation:** ghi nhận là 1 quyết định kỹ thuật
  cần làm rõ ngay khi bắt đầu thực thi Phần B (đọc source Worker thật lúc đó), không chốt cứng
  trước.
- **Risk 5:** Stage `knowledge-init` commit thẳng `main` giống `finalize` — nếu 1 feature khác
  được merge (kích hoạt `finalize`) đúng lúc interview `knowledge-init` đang chạy trên cùng repo
  mới bootstrap, 2 run có thể cùng động tới Project Knowledge. **Mitigation:** dùng chung
  concurrency group cố định `online-pipeline-finalize` cho cả 2 workflow để GitHub Actions tự
  serialize, không chạy song song.
- **Risk 6:** Check "đã completed"/"đã có issue mở" trong slash command Worker là kiểu
  check-rồi-mới-act — gọi lệnh 2 lần gần như đồng thời (race hiếm) có thể lọt qua check cả 2 lần,
  tạo 2 issue/thread cho cùng 1 repo. **Mitigation:** chấp nhận như rủi ro hiếm, cùng mức với các
  race hiếm khác đã được chấp nhận trong `slack-pipeline-interaction` (không xây thêm cơ chế lock
  mới cho v1).
- **Risk 7:** Nếu repo cũ chưa re-vendor `online-pipeline` mà vẫn gọi slash command, issue
  `knowledge-init` vẫn được tạo nhưng route job cũ (chưa có điều kiện bỏ qua label mới) có thể
  hiểu nhầm đây là issue `/new-feature` và tự chạy `user-spec-planning` trên nó. **Mitigation:**
  chấp nhận cho v1 — đây là hệ quả tất yếu của quy ước maintenance có sẵn (mọi update
  `online-pipeline` đều cần repo cũ tự re-vendor), không phải lỗi riêng của feature này; command
  không tự kiểm tra/cảnh báo trước, xem Constraints.

## Accepted Decisions

- Dùng đúng pattern "trigger phrase" đã có (`/done` → Feature Finalization Mode) cho
  `/init-knowledge` → `create-project-knowledge.md`, thay vì phát sinh cơ chế slash-command thật
  (methodology này không dùng slash-command wrapper file nào — xem `skills/methodology/SKILL.md`).
- Thêm 1 stage GitHub-Actions mới (`knowledge-init`) theo đúng khuôn các stage hiện có của
  `online-pipeline`, tái dùng toàn bộ hạ tầng đã có (Naming Contract, `relay_to_slack`, label
  filtering, retry pattern) — không phát sinh cơ chế điều phối riêng.
- Mỗi turn commit thẳng `main`, không dùng branch — khác với `finalize` (chạy 1 lần, commit 1 lần)
  vì `knowledge-init` chạy nhiều turn qua nhiều lần Actions riêng biệt không có ổ đĩa liên tục;
  phải có cách giữ trạng thái `interview.yml` giữa các lần chạy, và cách đơn giản nhất không vi
  phạm "không PR" là commit từng turn thẳng vào `main`.
- Tái dùng `interview_metadata.status: completed` làm marker hoàn tất/idempotency, không phát sinh
  marker mới.
- Label `knowledge-init` theo đúng pattern 2-điểm-tạo đã có sẵn cho CẢ 3 label hiện tại
  (`local-placeholder`, `userspec-spec`, `userspec-implement`): mỗi label đều được tạo defensively
  2 lần — 1 lần tạo trước (eager) trong `setup-online-pipeline.sh` lúc onboard repo, và 1 lần tạo
  inline ngay tại điểm thực sự áp dụng label đó (`online-pipeline-userspec.yml:275`,
  `online-pipeline-implement.yml:203`, `user-spec-planning/SKILL.md`'s Start path tương ứng).
  `knowledge-init` áp dụng cùng pattern: thêm vào block eager-create của
  `setup-online-pipeline.sh` (để repo onboard/re-vendor sau này đã có sẵn label), VÀ tạo inline
  defensive ngay tại 2 điểm thực sự áp dụng nó — `bootstrap-project.yml`'s issue-creation call
  (đường tự động) và Worker's slash-command handler (đường thủ công).
- Không phát sinh literal signal tự động mới cho `create-project-knowledge.md` — mọi checkpoint
  chỉ round-trip như 1 câu hỏi bình thường; bước "return to main skill" bị stage bỏ qua tường minh.
- Slash command Slack: lúc đầu chốt "chỉ dùng cho project mới" (bản ghi interview Q5/Q16), sau đó
  người dùng chủ động mở rộng sang "dùng được cho repo cũ cũng tốt" (Q17-19) — quyết định cuối: giữ
  đúng điều kiện gốc của `create-project-knowledge.md` (thiếu/template/partial), chỉ bỏ giới hạn
  thời điểm, không biến thành lệnh audit chung. Worker check trước khi tạo bất kỳ thứ gì trên
  GitHub, nhưng chỉ là check RẺ, best-effort (dựa trên field `interview_metadata.status`, không
  tự inspect nội dung thật): nếu field đó đã `completed` → không tạo issue, chỉ báo qua Slack,
  không lỗi; nếu đã có issue đang mở → không tạo issue thứ 2, chỉ báo đang chạy. Lớp check THẬT,
  có tính quyết định, nằm ở turn đầu tiên của stage `knowledge-init` (Phase 0 của
  `create-project-knowledge.md` tự inspect repo) — bắt được cả trường hợp Worker's check bỏ lọt
  (Project Knowledge hoàn chỉnh qua đường khác, không có field status).
- Chi tiết API `/relay` (có trả `ts` sẵn hay cần sửa Worker) để lại làm quyết định kỹ thuật lúc
  thực thi Phần B — không chốt cứng ở user-spec vì source Worker không có trong working directory
  hiện tại. Nếu `/relay` lỗi, `bootstrap-project.yml` vẫn tạo issue (không phụ thuộc Slack).

## Testing

**Unit tests:** không cần — đây là thay đổi hạ tầng (SKILL.md, workflow YAML, scripts vendor,
Worker code), không có logic đơn vị độc lập để unit-test theo nghĩa thông thường.

**Integration tests:** không cần test tự động riêng — các script vendor (`vendor-skills.sh`,
`setup-online-pipeline.sh`) đã có cơ chế validate-before-apply sẵn (all-or-nothing), chạy thử bằng
cách vendor thật vào 1 scaffold rồi kiểm tra file xuất hiện đúng vị trí là đủ.

**E2E tests:** cần — đây là thay đổi hành vi thật trên GitHub Actions + Slack + 1 repo GitHub mới
thật. Không thể giả lập bằng unit test; phải chạy tay theo đúng kịch bản ở mục Verification dưới,
tương tự cách `slack-pipeline-interaction` đã được verify.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Đọc lại `documentation-writing/SKILL.md` sau khi sửa | Có dòng trigger "/init-knowledge" trỏ thẳng tới `create-project-knowledge.md`, đúng vị trí/văn phong như dòng "/done" hiện có; không đổi logic routing khác. |
| 2. Đọc lại `online-pipeline/SKILL.md` sau khi sửa | Naming Contract có thêm label `knowledge-init`; có mục Stage `knowledge-init` mô tả đủ: mỗi turn commit `interview.yml` thẳng `main`, turn cuối commit Project Knowledge + đóng issue, bỏ qua bước "return to main skill", retry 2 lần rồi báo lỗi; `online-pipeline-userspec.yml`'s route job có thêm check bỏ qua label `knowledge-init`. |
| 3. Đọc workflow mới `online-pipeline-knowledge-init.yml` | Trigger đúng `issues`/`issue_comment` lọc theo label `knowledge-init`; KHÔNG có bước tạo label (label do `bootstrap-project.yml`/Worker tạo trước khi issue tồn tại, workflow này chỉ lọc theo label có sẵn); không mở branch nào, không gọi `gh pr create`; dùng `concurrency: group: online-pipeline-finalize` (chung với `finalize`, không phải group riêng); mỗi turn commit+push thẳng `main` là hành động CUỐI CÙNG trước khi job kết thúc — không có bước nào chạy sau đó. |
| 3b. Đọc retry-loop của workflow mới | Cấu trúc retry (đếm lần thử, reset về `origin/main` khi lỗi, báo lỗi qua `gh issue comment` + `relay_to_slack` sau khi hết lần thử, không rollback) khớp với retry-loop hiện có của `online-pipeline-implement.yml`/`online-pipeline-finalize.yml`. |
| 4. Đọc `project-initialization/assets/new-project/` sau khi vendor | Workflow mới có mặt ở cả `.github/workflows/` và `.claude/skills/online-pipeline/assets/workflows/`; `vendor-skills.sh`'s `WORKFLOWS` array có tên file mới. |
| 5. Chạy thử local: gõ "/init-knowledge" trên 1 project test thiếu Project Knowledge | `create-project-knowledge.md`'s Phase 0 bắt đầu ngay, không hỏi thêm bước routing nào. |
| 6. `git diff`/`git log` trong `control-plane` sau khi sửa (ở thư mục clone riêng) | `bootstrap-project.yml` có bước mở Slack message mới + tạo issue `knowledge-init` ngay sau bước vendor secret hiện có, VÀ vẫn tạo issue (chỉ thiếu marker) nếu `/relay` lỗi; Worker có route mới cho slash command, có check "đã completed"/"đã có issue mở" trước khi tạo issue (2 check này chỉ là best-effort, không cần hoàn hảo). |
| 7. Đọc Phase 0 phần mở rộng trong `online-pipeline/SKILL.md`'s mục Stage `knowledge-init` | Có mô tả rõ: turn đầu tiên luôn chạy Phase 0 của `create-project-knowledge.md` để tự inspect; nếu Project Knowledge thực tế đã không còn thiếu/template/partial (bất kể lý do, bất kể field `interview_metadata.status`), stage post "đã khởi tạo rồi" rồi đóng issue ngay, không chạy interview — đây là lớp check có tính quyết định, khác với check rẻ của Worker ở bước 6. |

### User Verification

- Chạy 1 lần `/new-project <tên-repo>` thật trong Slack channel đã link, trên account GitHub thật
  của người dùng — xác nhận: repo mới tạo như cũ, sau đó 1 issue `knowledge-init` tự xuất hiện,
  Slack thread mới mở ra với câu hỏi đầu tiên, trả lời vài câu (cả qua Slack và qua GitHub comment)
  để xác nhận interview tiếp tục đúng kể cả qua 1 checkpoint cuối cycle, rồi để interview hoàn tất
  và xác nhận Project Knowledge nằm trên `main`, issue đã đóng. Cần làm tay vì đây là hành vi thật
  trên Slack workspace + GitHub Actions + 1 repo GitHub thật, agent không có quyền truy cập Slack
  workspace hay secrets thật để tự chạy toàn bộ luồng này.
- Re-vendor `online-pipeline` trên 1 repo cũ (chạy lại `vendor-skills.sh`/
  `setup-online-pipeline.sh` sau khi feature này merge), rồi gọi slash command Slack thủ công trên
  repo đó còn thiếu Project Knowledge — xác nhận luồng chạy giống hệt trường hợp tự động (kể cả
  việc label `knowledge-init` tự được tạo trên repo đó nếu chưa có). Gọi lại trên repo đã hoàn tất
  (có `interview_metadata.status: completed`) — xác nhận Worker chỉ trả lời "đã khởi tạo rồi" qua
  Slack, không tạo issue/thread mới, không chạy lại interview.
- Gọi slash command trên 1 repo cũ có Project Knowledge đã đầy đủ nhưng KHÔNG qua
  `create-project-knowledge.md` (ví dụ xoá/không có `work/project-knowledge/interview.yml`) —
  xác nhận Worker's check rẻ không bắt được (vẫn tạo issue), nhưng turn đầu tiên của stage tự
  inspect và phát hiện đã đầy đủ, post "đã khởi tạo rồi" rồi đóng issue ngay, không chạy interview.
