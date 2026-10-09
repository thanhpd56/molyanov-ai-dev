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
issue). "Branch của round đang mở" được ghi nhớ bằng 1 marker trong issue body, để route
job/implement job luôn biết chính xác đang làm việc trên branch nào dù round nào. Số round không
giới hạn.

## Why
Hiện tại, ngay khi code PR merge, `online-pipeline-finalize.yml` tự động chạy: cập nhật Project
Knowledge, chuyển `work/{slug}/` vào `work/completed/{slug}/`, commit vào `main` (bản thân finalize
hiện tại **không** tự đóng issue — `SKILL.md` ghi rõ "no auto-close is added" cho đường này; issue
chỉ hết được theo dõi về mặt dữ liệu, không có tín hiệu kết thúc rõ ràng nào trên GitHub). Dù vậy,
không có cách nào để người dùng tiếp tục hỏi thêm (ví dụ về cách deploy repo đích — nằm ngoài phạm
vi pipeline này) hoặc yêu cầu sửa/tối ưu thêm sau khi đã thấy code thật chạy, vì route job hiện tại
luôn `skip` mọi comment một khi code PR đã merge. Muốn sửa gì sau đó phải tạo 1 issue/feature hoàn
toàn mới, mất hết context (`user-spec.md`, `decisions.md` cũ của feature đó). Feature
`online-pipeline-implement-followup` (đã merge trước đó) đã chủ động để ngoài phạm vi trường hợp
này — xem Accepted Decisions của spec đó: "1 thay đổi sau khi đã merge là 1 feature/fix mới, không
phải follow-up của PR cũ." Feature này đảo lại quyết định đó: biến việc "sửa thêm sau merge" thành
1 phần tự nhiên của cùng cuộc hội thoại trên issue, và bổ sung luôn 1 tín hiệu đóng issue rõ ràng
(AC11) mà hôm nay chưa có.

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
   - **Yêu cầu sửa code thêm**: tính N = (số PR đã từng mở cho slug này, khớp `feature/{slug}`
     HOẶC `feature/r*-{slug}`) + 1 (round 1 — không tiền tố — luôn được tính vào, nên lần sửa thêm
     đầu tiên luôn ra round 2), tạo branch mới `feature/r{N}-{slug}` từ `main` hiện tại, label
     `userspec-implement`, ghi/cập nhật marker `active_branch` trong issue body (xem Constraints)
     thành branch này. **Chỉ sau khi branch+PR+marker đã tạo xong thành công**, gỡ label
     `online-pipeline-merged-awaiting-followup` (giữ label ổn định cho tới khi chắc chắn, để 1 lỗi
     kỹ thuật giữa đường không làm issue bị kẹt không có label mà cũng chưa có round mới hợp lệ).
     Sau đó đọc lại `work/{slug}/user-spec.md` + `decisions.md` gốc để có context, coi comment là
     yêu cầu bổ sung, chạy `code-writing` bình thường (không chạy lại interview
     user-spec-planning).
   - **Tín hiệu hài lòng/muốn chốt**: dispatch (`gh workflow run online-pipeline-finalize.yml -f
     slug={slug} -f issue_number={issue_number}`) job `finalize` thật — không tự chạy logic
     finalize ngay trong job đang xử lý comment này.
5. Nếu có 1 comment khác tới ngay trong lúc agent đang xử lý bước 4 (đang phân loại, chưa kịp tạo
   round mới hoặc dispatch finalize) — job đang chạy bị huỷ (cancel-in-progress), job mới chạy theo
   đúng comment mới nhất. Áp dụng cùng quy tắc khi 1 round đã có branch/PR đang mở (bước 6).
6. Trong lúc PR round mới (bước 4, nhánh "yêu cầu sửa thêm") đang mở: comment tiếp theo xử lý đúng
   y như cơ chế `implement-followup`/`implement-resume` hiện có, bao gồm `cancel-in-progress`.
   Route job/implement job xác định đúng branch đang mở bằng cách đọc marker `active_branch` trong
   issue body (xem Constraints) — không còn giả định cứng tên branch là `feature/{slug}` ở BẤT KỲ
   nơi nào hiện đang dùng literal đó: kiểm tra branch tồn tại, checkout, mọi lệnh push (lần đầu,
   reset-retry, followup), `gh pr create --head`, `gh pr list --head`, text xác nhận post comment,
   và text mô tả branch trong prompt gửi cho agent. Hướng dẫn agent trong `SKILL.md` cho
   `implement-resume`/`implement-followup` cũng viết lại để không còn hardcode `feature/{slug}`
   trong văn bản stage, mà dẫn chiếu "branch được workflow cung cấp cho lần chạy này". Toàn bộ
   (mọi round + job xử lý comment sau merge) dùng chung 1 concurrency group
   `online-pipeline-implement-{slug}`; biểu thức `cancel-in-progress` của group này mở rộng để
   bao gồm cả action `post-merge` (giai đoạn phân loại, AC8) lẫn `followup` (giai đoạn round đã có
   branch, như hiện có) — không chỉ `followup` như hiện tại.
7. Khi PR round đó merge: quay lại bước 2 (label được set lại), lặp lại từ bước 3. Không giới hạn
   số round.
8. Job `finalize` (dispatch ở bước 4, nhánh "chốt") chạy dưới đúng concurrency group chung
   `online-pipeline-finalize` hiện có (chia sẻ với `knowledge-init`, không đổi) — không phải group
   riêng theo slug. Nó nhận `slug`/`issue_number` từ input `workflow_dispatch` (không còn
   `github.event.pull_request.head.ref` để tự đọc, vì không còn PR nào gắn với event này), chạy
   Feature Finalization Mode như hiện tại (cập nhật Project Knowledge, chuyển `work/{slug}/` vào
   `work/completed/{slug}/`), gỡ label `online-pipeline-merged-awaiting-followup`, post 1 comment
   xác nhận hoàn tất — nói rõ "đã hoàn tất; nếu cần sửa thêm, tạo issue/feature mới" — rồi **đóng
   issue** (hành vi mới; finalize hiện tại không tự đóng issue, xem Why).
9. Feature này chỉ áp dụng từ nay về sau (code PR merge sau khi deploy thay đổi này); không xử lý
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
- [ ] AC6: Stage post-merge nhận định comment mới nhất là yêu cầu sửa code thêm → tính N = (số PR
  đã từng mở cho slug này, khớp `feature/{slug}` HOẶC `feature/r*-{slug}` — liệt kê qua `gh pr list
  --state all --limit 1000 --json headRefName` (có `--limit` tường minh, vì `gh pr list` mặc định
  chỉ trả về 30 kết quả) rồi lọc bằng regex phía client, vì `gh pr list --head` không hỗ trợ glob)
  + 1 (round 1 không tiền tố luôn được tính, nên lần sửa thêm đầu tiên luôn ra round 2); tạo branch
  mới `feature/r{N}-{slug}` từ `main` hiện tại; label `userspec-implement`; ghi/cập nhật marker
  `active_branch` trong issue body (AC13) thành branch mới này. **Chỉ sau khi branch+PR+marker đã
  tạo xong thành công**, gỡ label `online-pipeline-merged-awaiting-followup` (không gỡ trước — nếu
  bước tạo round thất bại thật giữa đường, label vẫn còn, issue không bị kẹt ở trạng thái vô hình).
  Sau đó đọc lại `work/{slug}/user-spec.md` + `decisions.md` gốc; chạy `code-writing` bình thường
  (không chạy lại interview `user-spec-planning`), coi comment là yêu cầu bổ sung — không phải câu
  trả lời cho câu hỏi cũ.
- [ ] AC7: PR round N (AC6) đang mở → mọi comment tiếp theo xử lý đúng y cơ chế
  `implement-followup`/`implement-resume` hiện có, bao gồm `cancel-in-progress` khi có comment mới
  tới trong lúc round đang chạy — route job/implement job xác định branch đang hoạt động bằng cách
  đọc marker `active_branch` (AC13) ở TẤT CẢ nơi hiện đang hardcode `feature/$SLUG`: kiểm tra branch
  tồn tại, checkout, mọi lệnh push (lần đầu/reset-retry/followup), `gh pr create --head`, `gh pr
  list --head`, text xác nhận post comment, và text mô tả branch trong prompt gửi cho agent —
  không chỉ ở route job mà cả trong chính implement job. Hướng dẫn agent cho `implement-resume`/
  `implement-followup` trong `SKILL.md` cũng không còn hardcode `feature/{slug}` trong văn bản
  stage. Mọi round + job xử lý comment sau merge dùng chung 1 concurrency group
  `online-pipeline-implement-{slug}`.
- [ ] AC8: Biểu thức `cancel-in-progress` của group `online-pipeline-implement-{slug}` (hiện tại
  chỉ `action == 'followup'`) được mở rộng để bao gồm CẢ action `post-merge` (giai đoạn phân loại ở
  stage post-merge) lẫn `followup` (giai đoạn round đã có branch). Nhờ đó, nếu có 1 comment khác
  tới trong lúc agent đang phân loại comment ở stage post-merge (TRƯỚC khi kịp tạo round mới hoặc
  dispatch finalize) → job phân loại đang chạy bị huỷ, job mới chạy theo đúng comment mới nhất.
- [ ] AC9: Stage post-merge nhận định comment mới nhất là tín hiệu hài lòng/muốn chốt → dispatch
  (`gh workflow run online-pipeline-finalize.yml -f slug={slug} -f issue_number={issue_number}`)
  job `finalize` riêng — KHÔNG tự chạy logic finalize (update Project Knowledge/archive/đóng issue)
  ngay trong job xử lý comment này. Nếu lệnh `gh workflow run` thất bại (lỗi kỹ thuật, không phải bị
  cancel), coi là lỗi kỹ thuật của job xử lý comment, đi qua ATTEMPTS-retry hiện có — label
  `online-pipeline-merged-awaiting-followup` vẫn giữ nguyên, người dùng có thể comment lại để job
  thử lại.
- [ ] AC10: Job `finalize` (dispatch ở AC9) chạy dưới đúng concurrency group chung
  `online-pipeline-finalize` hiện có (chia sẻ với `knowledge-init`, không phải group riêng theo
  slug) — giữ nguyên toàn bộ bảo vệ race-condition giữa các feature khác nhau hiện có. Job này nhận
  `slug` và `issue_number` từ `workflow_dispatch` input (bước "Derive slug and check idempotency"
  hiện tại, đang đọc `github.event.pull_request.head.ref`, được sửa để đọc từ
  `github.event.inputs.*` khi trigger là `workflow_dispatch`). Job trong
  `online-pipeline-implement.yml` thực hiện dispatch (AC9) có thêm quyền `actions: write` trong
  khối `permissions:` của chính nó, dùng `GITHUB_TOKEN` mặc định.
- [ ] AC11: Job `finalize` (AC10) khi chạy xong: cập nhật Project Knowledge, chuyển `work/{slug}/`
  vào `work/completed/{slug}/`, gỡ label `online-pipeline-merged-awaiting-followup` khỏi issue,
  post 1 comment xác nhận hoàn tất nói rõ "đã hoàn tất; nếu cần sửa thêm, tạo issue/feature mới",
  rồi đóng issue (hành vi mới — finalize hiện tại không tự đóng issue).
- [ ] AC12: Comment muộn tới SAU KHI `work/completed/{slug}` đã tồn tại (đã chốt, issue đã đóng,
  label đã gỡ) → bị `action=skip` ở AC3, không dispatch lại gì — dù issue bị mở lại hay comment vẫn
  cố ý được gửi tới.
- [ ] AC13: "Branch của round đang mở" được ghi nhớ bằng 1 dòng marker trong issue body
  (`<!-- online-pipeline: active_branch=feature/r{N}-{slug} -->`, giống cách Slack marker
  channel_id/thread_ts đã lưu metadata trong issue body — 2 marker không xung đột vì mỗi dòng được
  parse độc lập bằng pattern riêng của chính nó). Marker **vắng mặt** = ngầm định round 1
  (`feature/{slug}`) — stage `implement` gốc (round 1) KHÔNG cần sửa để viết marker này. Chỉ stage
  post-merge (AC6, khi tạo round N≥2) viết/cập nhật marker này qua `gh issue edit`. Route
  job/implement job đọc marker này (hoặc dùng `feature/{slug}` nếu vắng mặt) ở TẤT CẢ nơi hiện đang
  hardcode `feature/$SLUG` cho mục đích xác định branch: kiểm tra branch tồn tại (`git ls-remote`),
  `git fetch`/`checkout`, mọi lệnh `git push` (lần đầu/reset-retry/followup), `gh pr create
  --head`, `gh pr list --head` (ở cả route job và implement job), text xác nhận post comment, và
  text mô tả branch trong prompt gửi cho agent. Đường dẫn status-file
  (`work/$SLUG/logs/working/online-pipeline-status.yml`) không đổi — nó phụ thuộc `$SLUG` (work
  folder), không phụ thuộc tên branch; chỉ cần đảm bảo đã checkout đúng branch (qua marker) trước
  khi đọc/ghi file này.
- [ ] AC14: Feature này chỉ áp dụng cho code PR merge SAU KHI deploy thay đổi này; không có xử lý
  hồi tố cho issue đã merge/đóng theo cơ chế finalize-tự-động cũ trước đó.

## Constraints
- Việc set label `online-pipeline-merged-awaiting-followup` là bash thuần (có retry), không gọi
  `claude -p`.
- Việc nhận định hỏi-đáp/sửa-thêm/chốt là agent-side judgment (`claude -p`) trong stage post-merge,
  không phải keyword filter cứng trong bash — nhất quán với cách `implement-followup` đã làm.
- Mỗi round PR mới dùng đúng label `userspec-implement` (để job set-label ở AC1 vẫn nhận ra); tên
  branch round N≥2 là `feature/r{N}-{slug}` — tiền tố `r{N}-` đặt ở ĐẦU, không phải cuối, để không
  phá quy tắc hiện có "đoạn cuối cùng sau dấu `-` cuối của branch luôn là issue number". **Round 1
  giữ nguyên `feature/{slug}` như hiện tại — không đổi Naming Contract của stage `implement` gốc đã
  chạy production.**
- "Branch của round đang mở" không được tra bằng cách query động (`gh pr list` mỗi lần) — lưu 1
  marker cố định trong issue body (AC13), chỉ stage post-merge cập nhật khi tạo round mới. Route
  job/implement job luôn đọc marker này thay cho literal `feature/$SLUG` ở MỌI nơi dùng để xác định
  branch (không chỉ route job) — xem danh sách đầy đủ ở AC13.
- Label `online-pipeline-merged-awaiting-followup` chỉ bị gỡ SAU KHI round mới (branch+PR+marker)
  đã tạo xong thành công (AC6) — không gỡ trước, để 1 lỗi kỹ thuật giữa đường không làm issue bị
  kẹt ở trạng thái không có label mà cũng chưa có round mới hợp lệ.
- Biểu thức `cancel-in-progress` của concurrency group `online-pipeline-implement-{slug}` phải bao
  gồm cả action `post-merge` lẫn `followup` (AC8) — không chỉ `followup` như hiện tại.
- Lệnh đếm round (AC6) phải có `--limit` tường minh (ví dụ 1000) khi gọi `gh pr list --state all`,
  vì `gh pr list` mặc định chỉ trả về 30 kết quả.
- Job `finalize` KHÔNG tham gia concurrency group riêng theo slug
  (`online-pipeline-implement-{slug}`) — giữ nguyên group chung cố định toàn repo
  `online-pipeline-finalize` (chia sẻ với `knowledge-init`), để 2 feature khác nhau không cùng lúc
  commit đè Project Knowledge trên `main`. Đổi TRIGGER của job này từ `pull_request: closed` sang
  `workflow_dispatch`, kèm sửa bước đọc slug/issue_number từ input (không còn PR event để tự parse).
- State "đang chờ followup/chốt" lưu hoàn toàn trên label của issue, không lưu trên file
  status-marker của branch đã merge (branch có thể bị xoá sau merge, tuỳ GitHub setting hoặc hành
  động tay của người dùng — không có cơ chế nào trong repo đảm bảo branch còn tồn tại).
  Status-marker trên branch của round đang chạy vẫn dùng y như `implement`/`implement-followup`
  hiện có, chỉ trong phạm vi round đó đang chạy. "Đã chốt xong" được đọc từ `work/completed/{slug}`
  tồn tại trên `main`, không từ label hay status marker.
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
  nhận lúc chốt (AC11) luôn nói rõ "đã hoàn tất; nếu cần sửa thêm, tạo issue/feature mới" — nếu bị
  hiểu nhầm, người dùng biết ngay và có đường xử lý tiếp (tạo issue mới), không bị mất thông tin
  hoàn toàn.
- **Risk 3:** `cancel-in-progress` huỷ đúng lúc 1 round đang push (AC7), hoặc đúng lúc agent đang
  phân loại/vừa quyết định xong (AC8) — rủi ro tương tự Risk 2/4 của `implement-followup`, cùng mức
  độ chấp nhận (push chỉ xảy ra ở checkpoint cố định, không liên tục; dùng chung concurrency group
  nên chỉ 1 job chạy/slug tại 1 thời điểm). Không áp dụng cho job `finalize` (AC10) vì nó chạy trong
  group riêng, không cancel-in-progress — nên không có rủi ro tương tự cho bước commit vào `main`.
- **Risk 4:** Job set-label (AC1/AC2) thất bại thật sau khi hết retry, VÀ chính bước post comment
  báo lỗi kỹ thuật đó cũng thất bại (ví dụ mất mạng kéo dài) — issue bị kẹt không có label, không
  nhận followup tiếp, không có thông báo nào. **Mitigation:** Chấp nhận ở mức độ tương tự các rủi
  ro "accepted for v1" khác đã có trong `SKILL.md` (ví dụ double-failure case của Hybrid
  Local/Online Switch) — đây là double-failure hiếm, người dùng vẫn luôn thấy code đã merge thật
  trên GitHub dù không có label/comment, có thể tự nhận ra và báo cáo.
- **Risk 5:** `gh workflow run` dispatch finalize (AC9) thất bại thật (không phải do cancel) — issue
  vẫn giữ label `online-pipeline-merged-awaiting-followup`, không bị mất trạng thái.
  **Mitigation:** Coi là lỗi kỹ thuật thông thường của job xử lý comment, đi qua ATTEMPTS-retry hiện
  có; nếu vẫn thất bại sau khi hết retry, job báo lỗi qua comment giống các lỗi kỹ thuật khác của
  stage này — người dùng có thể comment lại (ví dụ lặp lại tín hiệu "chốt") để thử dispatch lại.

## Accepted Decisions
- Tách job set-label (bash thuần, nghe event `pull_request: closed`) khỏi job `finalize` thật
  (dispatch qua `workflow_dispatch`) — thay vì chạy toàn bộ logic finalize ngay khi detect merge
  hoặc ngay trong job xử lý comment. Lý do: `finalize` hiện tại dùng 1 concurrency group CỐ ĐỊNH
  chung toàn repo để đảm bảo không có 2 lần finalize của 2 feature khác nhau cùng commit đè lên
  Project Knowledge trên `main` tại 1 thời điểm; nếu finalize chạy trong group riêng theo slug (như
  mọi round khác), bảo vệ này mất hoàn toàn. Giữ finalize là 1 job riêng dưới group chung hiện có,
  chỉ đổi trigger (kèm sửa bước đọc slug/issue_number từ `workflow_dispatch` input, vì
  `github.event.pull_request.head.ref` không còn tồn tại dưới trigger mới), bảo toàn được bảo vệ
  này mà vẫn đạt đúng mục tiêu "finalize do agent tự nhận định từ comment, không còn gắn với event
  merge". Job dispatch cần thêm quyền `actions: write`, dùng `GITHUB_TOKEN` mặc định (không cần PAT
  mới).
- Branch round N≥2 đặt tên `feature/r{N}-{slug}` (tiền tố `r{N}-` ở ĐẦU), không phải
  `feature/{slug}-{N}` (hậu tố ở cuối) — vì `slug` luôn có dạng `kebab-title-{issueNumber}`, và cơ
  chế hiện có suy ra issue number bằng cách lấy đúng đoạn cuối cùng sau dấu `-` cuối của tên branch.
  Thêm hậu tố sẽ làm sai lệch issue number suy ra được cho mọi round ≥2; thêm tiền tố ở đầu giữ
  nguyên quy tắc đó.
- **Round 1 giữ nguyên tên `feature/{slug}` (không tiền tố), không đổi thành `feature/r1-{slug}`** —
  dù việc đồng nhất mọi round cùng 1 công thức tên có vẻ gọn hơn, làm vậy đồng nghĩa phải sửa cả
  bước tạo branch của stage `implement` gốc đã chạy production (Naming Contract hiện có trong
  `SKILL.md`, dùng xuyên suốt 3 stage `implement`/`implement-resume`/`implement-followup` + các bản
  vendored) — vượt phạm vi "chỉ thêm followup sau merge" của feature này. Thay vào đó, công thức
  tính N (AC6) tự tính cả round 1 (khớp `feature/{slug}` KHÔNG tiền tố, HOẶC `feature/r*-{slug}`)
  nên lần sửa thêm đầu tiên vẫn luôn ra round 2 đúng như mong đợi, mà không cần đổi gì ở stage gốc.
- "Branch của round đang mở" lưu bằng 1 marker trong issue BODY (không phải query `gh pr list` động
  mỗi lần) — vì (a) `gh pr list --head` không hỗ trợ glob nên không thể query trực tiếp theo pattern
  `feature/r*-{slug}`, phải liệt kê hết rồi lọc phía client, tốn thêm 1 lệnh mỗi lần route job chạy;
  (b) lưu trực tiếp trên issue (giống cách Slack marker đã lưu channel_id/thread_ts) đơn giản hơn,
  chỉ cần đọc issue body có sẵn trong mọi context. Marker vắng mặt ngầm định round 1 — nên KHÔNG cần
  sửa gì ở stage `implement` gốc; chỉ stage post-merge (tạo round N≥2) mới cần viết marker này.
  Marker này không xung đột với Slack marker hiện có (mỗi dòng parse độc lập bằng pattern riêng).
  Phạm vi thay thế literal `feature/$SLUG` bằng marker này áp dụng cho MỌI nơi dùng để xác định
  branch trong cả route job VÀ implement job (không chỉ 4 chỗ ban đầu nêu ra ở vòng validate trước:
  ls-remote/checkout/gh pr list), bao gồm cả mọi lệnh push, `gh pr create --head`, text xác nhận,
  và text mô tả branch trong prompt gửi cho agent — cũng như văn bản stage `implement-resume`/
  `implement-followup` trong `SKILL.md`, vì các stage này chính là cơ chế được round N≥2 tái dùng
  (AC7). Đường dẫn status-file (`work/$SLUG/...`) không cần đổi vì nó phụ thuộc work-folder theo
  slug, không phải tên branch.
- Label `online-pipeline-merged-awaiting-followup` chỉ bị gỡ SAU KHI round mới tạo xong thành công
  (không gỡ ngay từ đầu AC6 như bản nháp trước) — nhất quán với cách nhánh "chốt" (AC9) giữ label
  ổn định cho tới khi chắc chắn dispatch thành công; tránh issue bị kẹt ở trạng thái không có label
  mà cũng chưa có round mới hợp lệ nếu có lỗi kỹ thuật giữa đường.
- Biểu thức `cancel-in-progress` hiện tại (`action == 'followup'`) được mở rộng thành bao gồm cả
  `post-merge` — vì AC8 (huỷ job đang phân loại khi có comment mới tới) không thể đạt được chỉ bằng
  cách "dùng chung concurrency group" nếu biểu thức huỷ không nhận diện action mới này.
- Lệnh đếm round (AC6) thêm `--limit` tường minh (ví dụ 1000) cho `gh pr list --state all` — vì
  `gh pr list` mặc định chỉ trả về 30 kết quả, có thể đếm thiếu round cho 1 feature có nhiều round
  hoặc 1 repo có nhiều PR khác.
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
- `cancel-in-progress` áp dụng cho CẢ giai đoạn agent đang phân loại comment (trước khi tạo round
  mới hoặc dispatch finalize) VÀ giai đoạn round đã có branch/PR đang mở — nhất quán nguyên tắc
  "ưu tiên comment mới nhất" áp dụng cho toàn bộ feature, không chỉ 1 giai đoạn.
- Round mới không chạy lại interview `user-spec-planning` — chỉ đọc lại `user-spec.md` +
  `decisions.md` gốc, coi comment là yêu cầu bổ sung trên cùng 1 feature, nhất quán với cách
  `implement-followup` đã xử lý comment trong lúc PR còn mở.
- N (số round) tính qua liệt kê (`gh pr list --state all --json headRefName`, không dùng `--head`
  vì không hỗ trợ glob) rồi lọc client-side theo pattern, không lưu counter riêng ở đâu — tránh thêm
  1 nguồn state mới có thể lệch khỏi GitHub thật; PR history trên GitHub vẫn còn dù branch có bị
  xoá hay không.
- Finalize (AC11) tự đóng issue khi hoàn tất — khác với hành vi hiện tại của `online-pipeline-
  finalize.yml` (không tự đóng issue, xem Why/Expected Behavior bước 8). Đây là 1 cải tiến đi kèm,
  không phải giữ nguyên hành vi cũ, vì feature này cần 1 tín hiệu kết thúc rõ ràng để AC12 (chặn
  comment muộn) có ý nghĩa quan sát được trên GitHub, không chỉ ở tầng dữ liệu nội bộ.
- Chỉ áp dụng từ nay về sau — không migrate issue đã merge theo cơ chế finalize-tự-động cũ. 1 issue
  đã đóng/đã archive trước khi deploy thay đổi này giữ nguyên trạng thái, không được "mở lại" bởi
  feature này.

## Testing

**Unit tests:** không áp dụng được — toàn bộ thay đổi là bash trong workflow YAML (route job, label,
marker trong issue body, concurrency, trigger của job finalize) và hướng dẫn agent trong `SKILL.md`,
không có logic ứng dụng mới để unit-test.

**Integration tests:** không áp dụng được theo nghĩa test tự động — không có harness CI nào mô
phỏng GitHub Actions + Slack Worker trong repo này.

**E2E tests:** cần, nhưng thủ công — xem Verification bên dưới (dry-run thực tế trên 1 repo đã bật
online-pipeline), vì đây là cách duy nhất verify được route job/label/marker/concurrency/dispatch
thật trên GitHub Actions.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Review diff của `online-pipeline-finalize.yml`: job set-label (nghe `pull_request: closed`, có retry, không gọi `claude -p`) tách biệt với job finalize thật (trigger đổi sang `workflow_dispatch`, bước đọc slug/issue_number đổi sang đọc từ input, giữ nguyên concurrency group cũ, logic cũ + bước gỡ label + bước đóng issue) | 2 job tách biệt rõ ràng; job finalize thật không còn đọc `github.event.pull_request.head.ref`; concurrency group của job finalize thật không đổi so với bản gốc. |
| 2. Review diff của `online-pipeline-implement.yml`: route job thêm check `work/completed/{slug}` (AC3), đọc marker `active_branch` từ issue body thay cho literal `feature/$SLUG` (AC13) ở TẤT CẢ call site (route job VÀ implement job: ls-remote, checkout, mọi lệnh push, `gh pr create`/`gh pr list --head`, text xác nhận, text prompt cho agent), thêm quyền `actions: write` cho job dispatch (AC10), mở rộng biểu thức `cancel-in-progress` để bao gồm action `post-merge` (AC8), và stage post-merge mới (AC5/AC6/AC8/AC9) | Thứ tự check đúng (completed trước label); nhánh "yêu cầu sửa thêm" dùng đúng công thức tính N (tính cả round 1, có `--limit`) và tên branch `feature/r{N}-{slug}`, gỡ label chỉ SAU KHI round mới tạo xong (không gỡ trước); nhánh "chốt" chỉ gọi `gh workflow run` kèm slug/issue_number, không có đoạn nào tự chạy logic finalize/archive/đóng issue trong chính job này; rà soát không còn SÓT bất kỳ literal `feature/$SLUG` nào trong implement job dùng cho mục đích xác định branch (đây là lỗi round validate trước đã tìm ra — chỉ sửa route job là chưa đủ). |
| 3. Review hướng dẫn agent trong `SKILL.md` cho stage post-merge (3 nhánh phân loại) và cập nhật Naming Contract (marker `active_branch`, branch pattern `feature/r{N}-{slug}`) | Hướng dẫn rõ: agent tự nhận định, không có keyword filter cứng; nhánh sửa-thêm đọc lại `user-spec.md`/`decisions.md` gốc, không chạy lại interview, tự ghi marker; nhánh chốt chỉ dispatch, không tự archive; `implement-resume`/`implement-followup` trong `SKILL.md` không còn giả định cứng `feature/{slug}` mà dẫn chiếu marker. |
| 4. Review đồng bộ vendored copies (`skills/project-initialization/assets/new-project/.github/workflows/`, mirror `.codex/`) | Nội dung khớp với bản nguồn trong `skills/online-pipeline/`, theo đúng "Setup Script Maintenance". |
| 5. Kiểm tra YAML hợp lệ (`actionlint`/`yamllint` nếu có sẵn, hoặc ít nhất parse YAML) | Không có lỗi cú pháp trong các file workflow đã sửa. |

### User Verification
- Dry-run thực tế trên 1 repo đã bật online-pipeline (route job/concurrency/label/marker/dispatch
  thật của GitHub Actions không thể verify được bằng code review hay unit test): tạo issue → qua
  spec → implement → code PR round 1 mở (`feature/{slug}`) → merge → xác nhận (a) finalize KHÔNG tự
  chạy, issue vẫn mở, label `online-pipeline-merged-awaiting-followup` xuất hiện.
- Comment hỏi-đáp (ví dụ hỏi về deploy repo đích) → xác nhận (b) agent trả lời ngắn, không tạo
  branch/PR mới, label không đổi.
- Comment yêu cầu sửa thêm → xác nhận (c) PR round 2 mở từ `main` với tên `feature/r2-{slug}`,
  label `userspec-implement`, label merged-awaiting-followup đã bị gỡ, marker `active_branch` trong
  issue body được cập nhật thành branch mới; gửi tiếp 1 comment khác ngay trong lúc round 2 đang
  chạy → xác nhận (d) job cũ bị huỷ, job mới chạy theo yêu cầu mới nhất (cancel-in-progress).
- Merge PR round 2 → xác nhận (e) label được set lại đúng issue (vẫn suy ra đúng issue number dù
  branch có tiền tố `r2-`).
- Gửi 2 comment liên tiếp rất nhanh ngay sau khi merge (trước khi job phân loại đầu kịp xong) → xác
  nhận (f) job phân loại đầu bị huỷ, chỉ job theo comment thứ 2 chạy tiếp (AC8).
- Comment mang tín hiệu hài lòng ("ok vậy là xong, cảm ơn") → xác nhận (g) job finalize thật được
  dispatch (`gh workflow run`) và chạy: Project Knowledge cập nhật, `work/{slug}/` chuyển vào
  `work/completed/{slug}/`, label bị gỡ, **issue đóng**, có comment xác nhận hoàn tất nói rõ "cần
  sửa thêm phải tạo issue mới".
- Gửi 1 comment muộn sau khi đã chốt (mở lại issue nếu cần để gửi được comment) → xác nhận (h)
  không có gì được dispatch lại (AC12).
- Cần verify thủ công vì đây là hành vi thời gian thực của GitHub Actions concurrency + label +
  issue-body marker + `workflow_dispatch`, không tái tạo được trong môi trường local/CI của repo
  này.
