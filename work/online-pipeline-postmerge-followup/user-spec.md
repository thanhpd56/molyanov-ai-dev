---
# Creation date (YYYY-MM-DD)
created: 2026-10-09

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: online-pipeline-postmerge-followup

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building
Tách stage `finalize` của `online-pipeline` ra khỏi event merge code PR: khi 1 code PR (round bất
kỳ) merge, 1 job bash thuần (không gọi agent) chỉ set 1 label đánh dấu "đã merge, đang chờ
followup/chốt" trên issue — issue vẫn mở, không tự đóng, không tự chạy finalize. Comment tiếp theo
trên issue được agent tự phân loại: chỉ hỏi-đáp (trả lời, không đổi gì), yêu cầu sửa thêm (tạo 1
branch+PR mới từ `main`, chạy `code-writing` như 1 round implement mới), hoặc tín hiệu người dùng
đã hài lòng/muốn chốt (dispatch job `finalize` thật — cập nhật Project Knowledge, archive, đóng
issue). Số round không giới hạn.

## Why
Hiện tại, ngay khi code PR merge, `online-pipeline-finalize.yml` tự động chạy: cập nhật Project
Knowledge, chuyển `work/{slug}/` vào `work/completed/{slug}/`, đóng issue. Người dùng mất kênh để
hỏi thêm (ví dụ về cách deploy repo đích — nằm ngoài phạm vi pipeline này) hoặc yêu cầu sửa/tối ưu
thêm sau khi đã thấy code thật chạy. Muốn sửa gì sau đó phải tạo 1 issue/feature hoàn toàn mới, mất
hết context (`user-spec.md`, `decisions.md` cũ của feature đó). Feature `online-pipeline-implement-
followup` (đã merge trước đó) đã chủ động để ngoài phạm vi trường hợp này — xem Accepted Decisions
của spec đó: "1 thay đổi sau khi đã merge là 1 feature/fix mới, không phải follow-up của PR cũ."
Feature này đảo lại quyết định đó: biến việc "sửa thêm sau merge" thành 1 phần tự nhiên của cùng
cuộc hội thoại trên issue, thay vì buộc phải mở issue mới mỗi lần.

## Expected Behavior
1. Code PR của round hiện tại (`feature/{slug}` nếu là round 1, `feature/r{N}-{slug}` nếu là round
   N≥2, label `userspec-implement`) merge vào `main`.
2. 1 job bash thuần (không gọi `claude -p`) set label `online-pipeline-merged-awaiting-followup`
   trên issue tương ứng — issue number được suy ra từ tên branch bằng đúng quy tắc hiện có (đoạn
   cuối cùng sau dấu `-` cuối cùng), không bị ảnh hưởng bởi tiền tố `r{N}-` vì tiền tố đó luôn nằm ở
   ĐẦU tên branch. Issue **không** tự đóng, **không** có finalize nào chạy ở bước này.
3. Người dùng xem code đã merge, comment tiếp trên issue (hoặc qua Slack, được relay vào). Route
   job của `online-pipeline-implement.yml`: trước tiên tự check `work/completed/{slug}` đã tồn tại
   trên `main` chưa — nếu đã tồn tại (đã chốt từ trước), `action=skip`. Nếu chưa và issue đang
   mang label ở bước 2, dispatch 1 action mới (post-merge) thay vì `skip` như hiện tại.
4. Stage post-merge: agent tự nhận định comment mới nhất thuộc 1 trong 3 loại:
   - **Hỏi-đáp/xã giao** (ví dụ "cảm ơn", hỏi chung về deploy của repo đích — nằm ngoài phạm vi
     pipeline): trả lời ngắn qua comment, không tạo branch/PR, không đổi label, không chạy
     `code-writing`.
   - **Yêu cầu sửa code thêm**: gỡ label `online-pipeline-merged-awaiting-followup`, tạo branch mới
     từ `main` hiện tại (`feature/r{N}-{slug}`, N = số PR round đã từng mở cho slug này + 1), label
     `userspec-implement`, đọc lại `work/{slug}/user-spec.md` + `decisions.md` gốc để có context,
     coi comment là yêu cầu bổ sung, chạy `code-writing` bình thường (không chạy lại interview
     user-spec-planning).
   - **Tín hiệu hài lòng/muốn chốt**: dispatch (`gh workflow run online-pipeline-finalize.yml`,
     kèm slug/issue number) job `finalize` thật — không tự chạy logic finalize ngay trong job đang
     xử lý comment này.
5. Trong lúc PR round mới (bước 4, nhánh "yêu cầu sửa thêm") đang mở: comment tiếp theo xử lý đúng
   y như cơ chế `implement-followup`/`implement-resume` hiện có, bao gồm `cancel-in-progress` khi
   có comment mới tới trong lúc round đang chạy. Toàn bộ (mọi round + job xử lý comment sau merge)
   dùng chung 1 concurrency group `online-pipeline-implement-{slug}`.
6. Khi PR round đó merge: quay lại bước 2 (label được set lại), lặp lại từ bước 3. Không giới hạn
   số round.
7. Job `finalize` (dispatch ở bước 4, nhánh "chốt") chạy dưới đúng concurrency group chung
   `online-pipeline-finalize` hiện có (chia sẻ với `knowledge-init`, không đổi) — không phải group
   riêng theo slug. Nó chạy Feature Finalization Mode như hiện tại (cập nhật Project Knowledge,
   chuyển `work/{slug}/` vào `work/completed/{slug}/`), gỡ label
   `online-pipeline-merged-awaiting-followup`, post 1 comment xác nhận hoàn tất — nói rõ "đã hoàn
   tất; nếu cần sửa thêm, tạo issue/feature mới" — rồi đóng issue.
8. Feature này chỉ áp dụng từ nay về sau (code PR merge sau khi deploy thay đổi này); không xử lý
   ngược cho issue đã merge/đã đóng theo cơ chế cũ.

## Acceptance Criteria
- [ ] AC1: Khi 1 code PR (label `userspec-implement`, branch `feature/{slug}` hoặc
  `feature/r{N}-{slug}`) merge, 1 job bash thuần (không gọi `claude -p`) set label
  `online-pipeline-merged-awaiting-followup` trên đúng issue, suy ra issue number từ tên branch
  bằng quy tắc hiện có (đoạn cuối cùng sau dấu `-` cuối) — không bị ảnh hưởng bởi tiền tố `r{N}-`.
  Job này **không** chạy `claude -p`, không đóng issue, không chạy finalize.
- [ ] AC2: Nếu lệnh `gh api` set label ở AC1 thất bại (lỗi tạm thời), job tự retry tối đa 3 lần
  trong cùng lần chạy; nếu vẫn thất bại sau khi hết retry, post 1 comment báo lỗi kỹ thuật lên issue
  qua `gh issue comment` (không có label, issue tạm không nhận followup tiếp — đây là tín hiệu để
  người dùng biết và có thể báo lại).
- [ ] AC3: Route job của `online-pipeline-implement.yml`, trên 1 `issue_comment` event mới, tự
  check `work/completed/{slug}` đã tồn tại trên `main` chưa TRƯỚC khi xét label — nếu đã tồn tại,
  `action=skip` (đã chốt từ trước, không dispatch gì thêm, bất kể label).
- [ ] AC4: Nếu `work/completed/{slug}` chưa tồn tại và issue đang mang label
  `online-pipeline-merged-awaiting-followup`, route job dispatch action mới (post-merge) thay vì
  `skip` như hiện tại.
- [ ] AC5: Stage post-merge nhận định comment mới nhất là hỏi-đáp/xã giao (không phải yêu cầu thay
  đổi/chốt thật) → trả lời ngắn qua comment, không tạo branch/PR, không đổi label, không chạy
  `code-writing`, không chạy finalize.
- [ ] AC6: Stage post-merge nhận định comment mới nhất là yêu cầu sửa code thêm → gỡ label
  `online-pipeline-merged-awaiting-followup`; tạo branch mới `feature/r{N}-{slug}` từ `main` hiện
  tại (N = số PR round đã từng mở cho slug này, tính qua `gh pr list` khớp pattern
  `feature/r*-{slug}`, cộng thêm 1 — không lưu counter riêng); label `userspec-implement`; đọc lại
  `work/{slug}/user-spec.md` + `decisions.md` gốc; chạy `code-writing` bình thường (không chạy lại
  interview `user-spec-planning`), coi comment là yêu cầu bổ sung — không phải câu trả lời cho câu
  hỏi cũ.
- [ ] AC7: PR round N (AC6) đang mở → mọi comment tiếp theo xử lý đúng y cơ chế
  `implement-followup`/`implement-resume` hiện có không đổi, bao gồm `cancel-in-progress` khi có
  comment mới tới trong lúc round đang chạy. Mọi round + job xử lý comment sau merge dùng chung 1
  concurrency group `online-pipeline-implement-{slug}`.
- [ ] AC8: Stage post-merge nhận định comment mới nhất là tín hiệu hài lòng/muốn chốt → dispatch
  (`gh workflow run online-pipeline-finalize.yml` kèm slug/issue number) job `finalize` riêng —
  KHÔNG tự chạy logic finalize (update Project Knowledge/archive/đóng issue) ngay trong job xử lý
  comment này.
- [ ] AC9: Job `finalize` (dispatch ở AC8) chạy dưới đúng concurrency group chung
  `online-pipeline-finalize` hiện có (chia sẻ với `knowledge-init`, không phải group riêng theo
  slug) — giữ nguyên toàn bộ bảo vệ race-condition giữa các feature khác nhau hiện có.
- [ ] AC10: Job `finalize` (AC9) khi chạy xong: cập nhật Project Knowledge, chuyển `work/{slug}/`
  vào `work/completed/{slug}/`, gỡ label `online-pipeline-merged-awaiting-followup` khỏi issue,
  post 1 comment xác nhận hoàn tất nói rõ "đã hoàn tất; nếu cần sửa thêm, tạo issue/feature mới",
  rồi đóng issue.
- [ ] AC11: Comment muộn tới SAU KHI `work/completed/{slug}` đã tồn tại (đã chốt, issue đã đóng,
  label đã gỡ) → bị `action=skip` ở AC3, không dispatch lại gì — dù issue bị mở lại hay comment vẫn
  cố ý được gửi tới.
- [ ] AC12: Feature này chỉ áp dụng cho code PR merge SAU KHI deploy thay đổi này; không có xử lý
  hồi tố cho issue đã merge/đóng theo cơ chế finalize-tự-động cũ trước đó.

## Constraints
- Việc set label `online-pipeline-merged-awaiting-followup` là bash thuần (có retry), không gọi
  `claude -p`.
- Việc nhận định hỏi-đáp/sửa-thêm/chốt là agent-side judgment (`claude -p`) trong stage post-merge,
  không phải keyword filter cứng trong bash — nhất quán với cách `implement-followup` đã làm.
- Mỗi round PR mới dùng đúng label `userspec-implement` (để job set-label ở AC1 vẫn nhận ra); tên
  branch round N≥2 là `feature/r{N}-{slug}` — tiền tố `r{N}-` đặt ở ĐẦU, không phải cuối, để không
  phá quy tắc hiện có "đoạn cuối cùng sau dấu `-` cuối của branch luôn là issue number". Round 1
  giữ nguyên `feature/{slug}` như hiện tại.
- Job `finalize` KHÔNG tham gia concurrency group riêng theo slug
  (`online-pipeline-implement-{slug}`) — giữ nguyên group chung cố định toàn repo
  `online-pipeline-finalize` (chia sẻ với `knowledge-init`), để 2 feature khác nhau không cùng lúc
  commit đè Project Knowledge trên `main`. Chỉ đổi TRIGGER của job này, từ `pull_request: closed`
  sang `workflow_dispatch`.
- State "đang chờ followup/chốt" lưu hoàn toàn trên label của issue, không lưu trên file
  status-marker của branch đã merge (branch có thể bị xoá sau merge, tuỳ GitHub setting hoặc hành
  động tay của người dùng — không có cơ chế nào trong repo đảm bảo branch còn tồn tại).
  Status-marker trên branch `feature/r{N}-{slug}` của round N vẫn dùng y như
  `implement`/`implement-followup` hiện có, chỉ trong phạm vi round đó đang chạy.
  "Đã chốt xong" được đọc từ `work/completed/{slug}` tồn tại trên `main`, không từ label hay status
  marker.
- Round mới không chạy lại interview `user-spec-planning`; vẫn đọc `user-spec.md` + `decisions.md`
  gốc của feature.
- Chỉ áp dụng từ nay về sau; không migrate issue đã merge từ trước.
- Thay đổi phải đồng bộ sang các bản vendored (scaffold `project-initialization`, mirror `.codex/`)
  theo đúng "Setup Script Maintenance" đã có trong `SKILL.md`.

## Risks
- **Risk 1:** Số round không giới hạn có thể chạy vô hạn, tốn CI minutes/Claude usage.
  **Mitigation:** Đây là quyết định chủ động của người dùng (mỗi round là 1 comment họ tự gửi); các
  cơ chế có sẵn (ATTEMPTS retry cap, rate-limit auto-switch) vẫn giới hạn chi phí của 1 lần chạy đơn
  lẻ — giống Risk 1 của `implement-followup`.
- **Risk 2:** Agent nhận định sai loại comment — ví dụ hiểu nhầm 1 yêu cầu sửa code thật thành tín
  hiệu "đã hài lòng, chốt lại", dispatch finalize + archive trong khi người dùng còn muốn sửa thêm;
  hoặc ngược lại, hiểu nhầm 1 câu hỏi-đáp thành yêu cầu sửa code và mở 1 PR round không cần thiết.
  **Mitigation:** Chấp nhận rủi ro này (agent-side judgment, không có keyword filter cứng để dựa
  vào). Giảm nhẹ: agent luôn trả lời lại qua comment ở mọi nhánh (không im lặng), và comment xác
  nhận lúc chốt (AC10) luôn nói rõ "đã hoàn tất; nếu cần sửa thêm, tạo issue/feature mới" — nếu bị
  hiểu nhầm, người dùng biết ngay và có đường xử lý tiếp (tạo issue mới), không bị mất thông tin
  hoàn toàn.
- **Risk 3:** `cancel-in-progress` huỷ đúng lúc 1 round đang push (AC7) — rủi ro tương tự Risk 2/4
  của `implement-followup`, cùng mức độ chấp nhận (push chỉ xảy ra ở checkpoint cố định, không liên
  tục; dùng chung concurrency group nên chỉ 1 job chạy/slug tại 1 thời điểm). Không áp dụng cho job
  `finalize` (AC9) vì nó chạy trong group riêng, không cancel-in-progress — nên không có rủi ro
  tương tự cho bước commit vào `main`.
- **Risk 4:** Job set-label (AC1/AC2) thất bại thật sau khi hết retry, VÀ chính bước post comment
  báo lỗi kỹ thuật đó cũng thất bại (ví dụ mất mạng kéo dài) — issue bị kẹt không có label, không
  nhận followup tiếp, không có thông báo nào. **Mitigation:** Chấp nhận ở mức độ tương tự các rủi
  ro "accepted for v1" khác đã có trong `SKILL.md` (ví dụ double-failure case của Hybrid
  Local/Online Switch) — đây là double-failure hiếm, người dùng vẫn luôn thấy code đã merge thật
  trên GitHub dù không có label/comment, có thể tự nhận ra và báo cáo.

## Accepted Decisions
- Tách job set-label (bash thuần, nghe event `pull_request: closed`) khỏi job `finalize` thật
  (dispatch qua `workflow_dispatch`) — thay vì chạy toàn bộ logic finalize ngay khi detect merge
  hoặc ngay trong job xử lý comment. Lý do: `finalize` hiện tại dùng 1 concurrency group CỐ ĐỊNH
  chung toàn repo để đảm bảo không có 2 lần finalize của 2 feature khác nhau cùng commit đè lên
  Project Knowledge trên `main` tại 1 thời điểm; nếu finalize chạy trong group riêng theo slug (như
  mọi round khác), bảo vệ này mất hoàn toàn. Giữ finalize là 1 job riêng dưới group chung hiện có,
  chỉ đổi trigger, bảo toàn được bảo vệ này mà vẫn đạt đúng mục tiêu "finalize do agent tự nhận định
  từ comment, không còn gắn với event merge".
- Branch round N≥2 đặt tên `feature/r{N}-{slug}` (tiền tố `r{N}-` ở ĐẦU), không phải
  `feature/{slug}-{N}` (hậu tố ở cuối) — vì `slug` luôn có dạng `kebab-title-{issueNumber}`, và cơ
  chế hiện có (job set-label ở AC1, cũng như mọi nơi khác trong `SKILL.md`) suy ra issue number
  bằng cách lấy đúng đoạn cuối cùng sau dấu `-` cuối của tên branch. Thêm hậu tố sẽ làm sai lệch
  issue number suy ra được cho mọi round ≥2; thêm tiền tố ở đầu giữ nguyên quy tắc đó vì không đổi
  đoạn cuối cùng.
- Không tạo giá trị status-marker mới hay file state mới cho "đang chờ followup/chốt" — dùng 1
  label trên issue. Lý do: status-marker hiện có sống trên file của chính branch đã merge; không có
  gì trong repo đảm bảo branch đó còn tồn tại sau merge (không có bước `--delete-branch` hay cấu
  hình auto-delete cho code PR, nhưng cũng không có gì đảm bảo ngược lại) — 1 label trên issue thì
  chắc chắn sống sót bất kể branch còn hay mất.
- Route job tự check `work/completed/{slug}` trên `main` trước khi xét label (AC3) — idempotency
  giống cách `finalize` hiện tại tự check thư mục này để tránh chạy lại; phòng hờ cả trường hợp
  label lỡ chưa được gỡ kịp sau khi đã chốt.
- Nhận định hỏi-đáp/sửa-thêm/chốt giao hoàn toàn cho agent trong stage prompt, không viết filter
  từ khoá cứng trong bash — nhất quán với cách routing trong skill này luôn "dumb"/deterministic,
  mọi nhận định nội dung nằm trong `claude -p` (giống lý do đã chọn cho `implement-followup`).
  Không có lệnh tường minh kiểu `/finalize` — người dùng chỉ cần comment tự nhiên.
  Việc nhận định sai được chấp nhận là rủi ro (xem Risk 2), không build thêm bước xác nhận 2 lần.
- Round mới không chạy lại interview `user-spec-planning` — chỉ đọc lại `user-spec.md` +
  `decisions.md` gốc, coi comment là yêu cầu bổ sung trên cùng 1 feature, nhất quán với cách
  `implement-followup` đã xử lý comment trong lúc PR còn mở.
- N (số round) tính qua đếm số PR đã từng mở khớp pattern `feature/r*-{slug}` (+1), không lưu
  counter riêng ở đâu — tránh thêm 1 nguồn state mới có thể lệch khỏi GitHub thật; PR history trên
  GitHub vẫn còn dù branch có bị xoá hay không.
- Chỉ áp dụng từ nay về sau — không migrate issue đã merge theo cơ chế finalize-tự-động cũ. 1 issue
  đã đóng/đã archive trước khi deploy thay đổi này giữ nguyên trạng thái, không được "mở lại" bởi
  feature này.

## Testing

**Unit tests:** không áp dụng được — toàn bộ thay đổi là bash trong workflow YAML (route job, label,
concurrency, trigger của job finalize) và hướng dẫn agent trong `SKILL.md`, không có logic ứng dụng
mới để unit-test.

**Integration tests:** không áp dụng được theo nghĩa test tự động — không có harness CI nào mô
phỏng GitHub Actions + Slack Worker trong repo này.

**E2E tests:** cần, nhưng thủ công — xem Verification bên dưới (dry-run thực tế trên 1 repo đã bật
online-pipeline), vì đây là cách duy nhất verify được route job/label/concurrency/dispatch thật trên
GitHub Actions.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Review diff của `online-pipeline-finalize.yml`: job set-label (nghe `pull_request: closed`, có retry, không gọi `claude -p`) tách biệt với job finalize thật (trigger đổi sang `workflow_dispatch`, giữ nguyên concurrency group cũ, logic cũ + bước gỡ label) | 2 job tách biệt rõ ràng, không job nào vừa set label vừa chạy finalize; job finalize thật không có thay đổi gì khác ngoài trigger + bước gỡ label; concurrency group của job finalize thật không đổi so với bản gốc. |
| 2. Review diff của `online-pipeline-implement.yml`: route job thêm check `work/completed/{slug}` (AC3) trước khi xét label (AC4), và stage post-merge mới (AC5/AC6/AC8) | Thứ tự check đúng (completed trước label); nhánh "yêu cầu sửa thêm" dùng đúng tên branch `feature/r{N}-{slug}` (không phải `feature/{slug}-{N}`); nhánh "chốt" chỉ gọi `gh workflow run`, không có đoạn nào tự chạy logic finalize/archive/đóng issue trong chính job này. |
| 3. Review hướng dẫn agent trong `SKILL.md` cho stage post-merge (3 nhánh phân loại) | Hướng dẫn rõ: agent tự nhận định, không có keyword filter cứng; nhánh sửa-thêm đọc lại `user-spec.md`/`decisions.md` gốc, không chạy lại interview; nhánh chốt chỉ dispatch, không tự archive. |
| 4. Review đồng bộ vendored copies (`skills/project-initialization/assets/new-project/.github/workflows/`, mirror `.codex/`) | Nội dung khớp với bản nguồn trong `skills/online-pipeline/`, theo đúng "Setup Script Maintenance". |
| 5. Kiểm tra YAML hợp lệ (`actionlint`/`yamllint` nếu có sẵn, hoặc ít nhất parse YAML) | Không có lỗi cú pháp trong các file workflow đã sửa. |

### User Verification
- Dry-run thực tế trên 1 repo đã bật online-pipeline (route job/concurrency/label/dispatch thật của
  GitHub Actions không thể verify được bằng code review hay unit test): tạo issue → qua spec →
  implement → code PR round 1 mở → merge → xác nhận (a) finalize KHÔNG tự chạy, issue vẫn mở, label
  `online-pipeline-merged-awaiting-followup` xuất hiện.
- Comment hỏi-đáp (ví dụ hỏi về deploy repo đích) → xác nhận (b) agent trả lời ngắn, không tạo
  branch/PR mới, label không đổi.
- Comment yêu cầu sửa thêm → xác nhận (c) PR round 2 mở từ `main` với tên `feature/r2-{slug}`, label
  `userspec-implement`, label merged-awaiting-followup đã bị gỡ; gửi tiếp 1 comment khác ngay trong
  lúc round 2 đang chạy → xác nhận (d) job cũ bị huỷ, job mới chạy theo yêu cầu mới nhất
  (cancel-in-progress, giống cơ chế implement-followup).
- Merge PR round 2 → xác nhận (e) label được set lại.
- Comment mang tín hiệu hài lòng ("ok vậy là xong, cảm ơn") → xác nhận (f) job finalize thật được
  dispatch và chạy: Project Knowledge cập nhật, `work/{slug}/` chuyển vào `work/completed/{slug}/`,
  label bị gỡ, issue đóng, có comment xác nhận hoàn tất nói rõ "cần sửa thêm phải tạo issue mới".
- Gửi 1 comment muộn sau khi đã chốt (mở lại issue nếu cần để gửi được comment) → xác nhận (g)
  không có gì được dispatch lại (AC11).
- Cần verify thủ công vì đây là hành vi thời gian thực của GitHub Actions concurrency + label +
  workflow_dispatch, không tái tạo được trong môi trường local/CI của repo này.
