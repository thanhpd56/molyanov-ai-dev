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
   - Nếu issue đang xử lý đã có `interview_metadata.status: completed` (coi như "đã khởi tạo rồi"
     — trường hợp này thực tế chỉ xảy ra khi bị trigger lại ngoài ý muốn, vì issue tự đóng ngay khi
     hoàn tất): post "đã khởi tạo rồi" rồi đóng issue ngay, không chạy lại interview.
   - `online-pipeline-userspec.yml`'s route job thêm 1 điều kiện bỏ qua issue/comment mang label
     `knowledge-init`, cùng cách nó đang bỏ qua `local-placeholder`.
   - Lỗi/crash: theo đúng pattern retry đã có (tối đa 2 lần retry, rồi báo lỗi qua `gh issue
     comment` + `relay_to_slack`, không rollback, issue vẫn mở).
3. Workflow mới này vendor vào 2 vị trí trong scaffold của `project-initialization` —
   `.github/workflows/` và `.claude/skills/online-pipeline/assets/workflows/` — giống 3 workflow
   hiện có; `vendor-skills.sh`'s `WORKFLOWS` array thêm file mới; label `knowledge-init` được tạo
   defensively (check rồi tạo) ngay tại điểm dùng đầu tiên, không qua `setup-online-pipeline.sh`
   (vì repo mới không chạy script đó).

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
5. Worker thêm 1 slash command Slack mới (tên làm việc: `/init-knowledge`) dùng cho bất kỳ channel
   đã link với 1 repo (mới hoặc cũ), miễn Project Knowledge của repo đó còn thiếu/template/
   partial — cùng pattern với `/new-feature` (ACK <3s, mở thread trước để có `thread_ts`, tạo issue
   `knowledge-init` với marker, relay câu hỏi đầu tiên). Trước khi tạo issue mới, kiểm tra:
   - Nếu repo đó đã có `interview_metadata.status: completed` → không tạo issue, báo "đã khởi tạo
     rồi".
   - Nếu repo đó đã có 1 issue `knowledge-init` đang mở (interview đang chạy) → không tạo issue
     thứ 2, báo "đang chạy rồi, xem issue #{number}".
   Command này KHÔNG phải công cụ audit lại Project Knowledge đã hoàn chỉnh — nó chỉ chạy đúng
   `create-project-knowledge.md`, không chạy mode "full update/audit" của `documentation-writing`.

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
- [ ] Slash command Slack gọi trên repo đã có Project Knowledge hoàn chỉnh → báo "đã khởi tạo rồi"
  + đóng issue ngay, không chạy lại interview, không báo lỗi.
- [ ] Slash command Slack gọi lần 2 khi đã có issue `knowledge-init` đang mở cho repo đó → không
  tạo issue/thread thứ 2, báo issue đang mở đó.
- [ ] `claude -p` lỗi/crash trong stage `knowledge-init` → retry tối đa 2 lần, rồi báo lỗi qua
  `gh issue comment` + Slack, issue vẫn mở, không rollback.

## Constraints

- Trigger tự động (issue tự tạo ngay sau bootstrap) chỉ áp dụng cho repo mới — gắn liền với sự
  kiện bootstrap, chỉ xảy ra 1 lần/repo.
- Slash command Slack thủ công bỏ giới hạn thời điểm "phải vừa mới bootstrap", nhưng giữ đúng điều
  kiện gốc của `create-project-knowledge.md` (Project Knowledge còn thiếu/template/partial) — không
  biến thành lệnh "chạy documentation-writing theo yêu cầu" chung, không kích hoạt mode "full
  update/audit".
- Không dùng branch riêng ở bất kỳ đâu trong flow này: mỗi turn commit thẳng `main`; không mở PR.
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
- Label `knowledge-init` được tạo defensively ngay tại điểm dùng đầu tiên (trong
  `bootstrap-project.yml`'s issue-creation call), theo đúng idiom đang dùng cho
  `local-placeholder`/`userspec-spec`/`userspec-implement`.
- Không phát sinh literal signal tự động mới cho `create-project-knowledge.md` — mọi checkpoint
  chỉ round-trip như 1 câu hỏi bình thường; bước "return to main skill" bị stage bỏ qua tường minh.
- Slash command Slack: lúc đầu chốt "chỉ dùng cho project mới" (bản ghi interview Q5/Q16), sau đó
  người dùng chủ động mở rộng sang "dùng được cho repo cũ cũng tốt" (Q17-19) — quyết định cuối: giữ
  đúng điều kiện gốc của `create-project-knowledge.md` (thiếu/template/partial), chỉ bỏ giới hạn
  thời điểm, không biến thành lệnh audit chung. Repo đã hoàn chỉnh → đóng issue ngay, không lỗi;
  repo đang chạy interview → báo đang chạy, không tạo trùng.
- Chi tiết API `/relay` (có trả `ts` sẵn hay cần sửa Worker) để lại làm quyết định kỹ thuật lúc
  thực thi Phần B — không chốt cứng ở user-spec vì source Worker không có trong working directory
  hiện tại.

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
| 3. Đọc workflow mới `online-pipeline-knowledge-init.yml` | Trigger đúng `issues`/`issue_comment` lọc theo label `knowledge-init`; có bước tạo label defensive; không mở branch nào, không gọi `gh pr create`. |
| 4. Đọc `project-initialization/assets/new-project/` sau khi vendor | Workflow mới có mặt ở cả `.github/workflows/` và `.claude/skills/online-pipeline/assets/workflows/`; `vendor-skills.sh`'s `WORKFLOWS` array có tên file mới. |
| 5. Chạy thử local: gõ "/init-knowledge" trên 1 project test thiếu Project Knowledge | `create-project-knowledge.md`'s Phase 0 bắt đầu ngay, không hỏi thêm bước routing nào. |
| 6. `git diff`/`git log` trong `control-plane` sau khi sửa (ở thư mục clone riêng) | `bootstrap-project.yml` có bước mở Slack message mới + tạo issue `knowledge-init` ngay sau bước vendor secret hiện có; Worker có route mới cho slash command, có check "đã completed"/"đã có issue mở" trước khi tạo issue. |

### User Verification

- Chạy 1 lần `/new-project <tên-repo>` thật trong Slack channel đã link, trên account GitHub thật
  của người dùng — xác nhận: repo mới tạo như cũ, sau đó 1 issue `knowledge-init` tự xuất hiện,
  Slack thread mới mở ra với câu hỏi đầu tiên, trả lời vài câu (cả qua Slack và qua GitHub comment)
  để xác nhận interview tiếp tục đúng kể cả qua 1 checkpoint cuối cycle, rồi để interview hoàn tất
  và xác nhận Project Knowledge nằm trên `main`, issue đã đóng. Cần làm tay vì đây là hành vi thật
  trên Slack workspace + GitHub Actions + 1 repo GitHub thật, agent không có quyền truy cập Slack
  workspace hay secrets thật để tự chạy toàn bộ luồng này.
- Gọi slash command Slack thủ công trên 1 repo cũ (không phải vừa bootstrap) còn thiếu Project
  Knowledge — xác nhận luồng chạy giống hệt trường hợp tự động. Gọi lại lần 2 trên repo đã hoàn
  tất — xác nhận báo "đã khởi tạo rồi" và đóng issue ngay, không chạy lại interview.
