---
# Creation date (YYYY-MM-DD)
created: 2026-10-03

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: hybrid-local-online-pipeline

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building
Mở rộng `skills/online-pipeline/SKILL.md` với khả năng chuyển đổi 2 chiều (local Claude Code CLI
↔ online GitHub Actions) cho mọi feature trong project đã bật `online-pipeline`, ở cả 3 giai đoạn
(userspec, implement, finalize), kể cả giữa chừng 1 giai đoạn. Với mọi project đã bật
`online-pipeline`, Start path của `user-spec-planning` (`skills/user-spec-planning/SKILL.md` Step
1) đổi hẳn: mọi feature local mới đều tạo ngay 1 GitHub issue rỗng (gắn label `local-placeholder`)
và lấy slug = `kebab(issue title) + "-" + issue number` — đúng naming contract online-pipeline đã
dùng — thay cho việc tự chọn slug như hiện tại. Cơ chế switch tái dùng tối đa trigger GitHub
Actions có sẵn (`issues: opened`, `issue_comment: created`, `pull_request: closed`), không sửa
`on:` event của 3 workflow YAML hiện có — chỉ sửa điều kiện nội bộ của route job và bổ sung nhánh
mới trong stage prompt của `online-pipeline/SKILL.md`.

## Why
Hiện tại, một feature phải chọn hẳn 1 trong 2 chế độ (hoàn toàn local hoặc hoàn toàn online) và
không thể đổi giữa đường mà không tự tay dựng lại trạng thái tương đương (branch, PR, label) bằng
git/gh thủ công. Thực tế: người dùng muốn làm nhanh ở local khi đang ngồi máy (dùng tài khoản
Claude subscription, tương tác trực tiếp), rồi khi bận việc (ra khỏi máy) vẫn muốn feature tiếp
tục chạy/theo dõi được qua GitHub (điện thoại/trình duyệt) — và ngược lại — ở bất kỳ giai đoạn, kể
cả giữa chừng 1 giai đoạn.

## Expected Behavior

### Tạo issue placeholder ngay từ đầu
1. Người dùng bắt đầu 1 feature mới ở local trong project đã bật `online-pipeline`.
2. Start path của `user-spec-planning` kích hoạt nhánh tạo issue mới CHỈ KHI cả 2 điều kiện đúng:
   (a) project có bật online-pipeline (dựa trên sự tồn tại của
   `.github/workflows/online-pipeline-userspec.yml` đã vendor), VÀ (b) Start path được gọi MÀ
   KHÔNG có slug nào truyền sẵn — tức người dùng gõ tương tác ở local, chưa có slug. Khi đúng cả
   2: tạo 1 GitHub issue rỗng (title = tên feature), gắn label `local-placeholder`, tính slug =
   `kebab(issue-title)-{issue-number}`, dùng slug này cho `work/{slug}` (gọi
   `init-feature-folder.sh` như hiện tại, không đổi chữ ký script). Nếu `gh` CLI chưa cài/chưa
   auth: báo lỗi rõ, dừng ngay, không tạo feature folder dở. Nếu project không bật
   online-pipeline: hành vi Start path giữ nguyên như hiện tại (tự chọn slug, không tạo issue).
   **Quan trọng:** `online-pipeline/SKILL.md` stage `userspec-turn` đã có sẵn hợp đồng gọi Start
   path TỪ BÊN TRONG GitHub Actions job với slug cho sẵn ("call Start path directly with the given
   slug, do not let it choose its own slug") — mỗi lần đó, điều kiện (b) sai nên nhánh tạo issue
   mới KHÔNG được kích hoạt, tránh tạo thêm 1 issue thừa mỗi khi online-pipeline tự chạy bình
   thường (kể cả lần đầu hoàn toàn không liên quan tới hybrid switch).
3. Route job của `online-pipeline-userspec.yml` bỏ qua xử lý cho MỌI event (`issues: opened` và
   `issue_comment: created`) trong khi issue còn label `local-placeholder` — không có comment
   interview nào bị tự động post lên issue trong lúc người dùng đang làm việc ở local (tránh race
   giữa automation và phiên đang chạy ở local).
4. Người dùng tiếp tục interview/implement/finalize hoàn toàn ở local như bình thường.

### Switch-to-online (local → online)
5. Bất kỳ lúc nào (kể cả giữa chừng 1 batch câu hỏi interview, hoặc giữa chừng 1 file đang
   implement), người dùng gọi hành động switch-to-online trong `online-pipeline/SKILL.md`:
   - **Giữa chừng userspec:** push branch `userspec/{slug}` (tạo mới nếu chưa có) với
     `interview.yml` ở trạng thái hiện tại; gỡ label `local-placeholder` khỏi issue; đăng comment
     `/switch-online` lên issue (không kèm câu trả lời thật — câu trả lời thật, nếu có, là 1
     comment riêng, đến sau). Route job của `online-pipeline-userspec.yml` dispatch "continue" như
     bình thường qua `issue_comment`; stage `userspec-turn` nhận diện comment kích hoạt chính là
     `/switch-online` → KHÔNG append nó làm câu trả lời, chỉ tiếp tục interview loop bình thường
     (tìm gap đang dở, đăng câu hỏi tiếp theo như 1 comment mới).
   - **Ranh giới userspec (spec đã approve local, chưa từng lên GitHub):** switch-to-online đẩy
     `userspec/{slug}` với `user-spec.md`/`decisions.md` đã approved, gỡ label, mở PR (label
     `userspec-spec`) — người dùng dùng đúng luồng `/approve` có sẵn để merge và kích hoạt
     implement online, không cần cơ chế mới.
   - **Giữa chừng implement (code đang viết dở):** push branch `feature/{slug}` (tạo mới nếu chưa
     có, dựa trên `main` hiện tại) với code hiện tại. Nếu đây là lần đầu `feature/{slug}` tồn tại
     online: khởi tạo file `work/{slug}/logs/working/online-pipeline-status.yml` với `status:
     awaiting_decision` (giá trị chính xác route job thật của `online-pipeline-implement.yml` cần
     để dispatch qua comment — không ghi đè nếu branch đã có marker từ trước). Gỡ label
     `local-placeholder` khỏi issue TRƯỚC khi đăng comment `/switch-online` (route job đọc trạng
     thái label từ snapshot payload của chính event comment, nên thứ tự này bắt buộc). Stage
     `implement-resume` nhận diện comment kích hoạt chính là `/switch-online` → chạy như bước 1 của
     stage `implement` gốc (đọc `user-spec.md`/`decisions.md` đã duyệt, implement bằng
     `code-writing` trên repo hiện tại), không ép buộc coi comment là câu trả lời cho câu hỏi
     không tồn tại.
     - Nếu feature làm hoàn toàn local tới thời điểm này (userspec chưa từng có branch/PR online,
       bỏ qua bước PR-spec-review hình thức): route job của `online-pipeline-userspec.yml` không
       hiểu nhầm comment `/switch-online` (hay bất kỳ comment sau đó trong lúc implement) thành
       yêu cầu bắt đầu interview mới. Logic phát hiện "userspec đã xong" thật của route job là
       nested if/else (không phải 2 dấu hiệu OR phẳng): nếu branch `userspec/{slug}` KHÔNG tồn tại
       → đọc `status: approved` từ `user-spec.md` trên main; nếu branch TỒN TẠI → đọc `status:
       approved` từ chính branch đó (branch tồn tại mà chưa approved nghĩa là "đang dở", không
       phải "đã xong"). Dấu hiệu thứ 3 — branch `feature/{slug}` tồn tại trên remote — được chèn
       vào ĐÚNG nhánh "branch `userspec/{slug}` KHÔNG tồn tại" (song song với check approved trên
       main), đúng case bypass này, không gộp OR phẳng với nhánh "branch tồn tại" để tránh phá
       luồng continue bình thường.
   - **Ranh giới implement (code xong local, chưa từng lên GitHub):** switch-to-online mở PR
     `feature/{slug}` (label `userspec-implement`) — người dùng review/merge như luồng online bình
     thường để kích hoạt finalize.
6. Lỗi push/conflict khi switch (remote có commit mới mà local chưa có): tự `fetch` + thử
   `merge`/`rebase`; nếu merge sạch thì tự tiếp tục push; nếu có conflict thật, dừng và hiển thị
   đúng conflict đó cho người dùng tự xử lý trong chat (không tự chọn bên thắng).
   - Nếu branch push thành công nhưng bước đăng comment `/switch-online` lỗi kỹ thuật (mất mạng,
     quyền `gh` không đủ): dừng, báo lỗi rõ, KHÔNG coi switch là đã thành công — trạng thái local
     vẫn là nguồn đáng tin cho tới khi comment thật sự lên được issue (branch đã push không đồng
     nghĩa automation đã được "đánh thức").
7. Giai đoạn finalize chạy nhanh, atomic (commit thẳng vào `main`, không có checkpoint giữa
   chừng) — switch ở giai đoạn này chỉ có nghĩa là chọn NƠI finalize chạy (local hay online) trước
   khi bắt đầu, không có khái niệm "giữa chừng finalize" để switch, vì bản chất atomic của bước
   này không tạo ra điểm dừng nào. Đây là giới hạn đã xác nhận, không phải thiếu sót.

### Switch-to-local (online → local)
8. Khi đang ở GitHub (dở interview qua comment, hoặc dở implement, có thể đang
   `awaiting_decision`), người dùng gọi hành động switch-to-local: kiểm tra working tree local có
   thay đổi chưa commit — nếu có, dừng ngay, báo lỗi rõ, yêu cầu người dùng tự commit/stash trước
   (không tự động stash hộ, không checkout đè lên thay đổi chưa lưu). Nếu sạch: `fetch` + checkout
   đúng branch (`userspec/{slug}` hoặc `feature/{slug}`), đọc status marker/`interview.yml`, báo rõ
   "đang dở ở đâu, câu hỏi/finding gì chưa trả lời" để người dùng tiếp tục ngay trong chat local
   (dùng lại đúng `user-spec-planning` Resume path hoặc `code-writing` như bình thường).

### Finalize và dọn issue
9. Feature hoàn tất ở local (bất kỳ lịch sử switch, kể cả chưa từng switch) → khi finalize chạy ở
   local xong, issue placeholder tự đóng (`gh issue close`). Nếu `gh issue close` thất bại: finalize
   local vẫn coi là thành công (archive `work/{feature}` vẫn xảy ra), chỉ log/báo không đóng được
   issue, không rollback finalize. Finalize chạy online giữ nguyên hành vi hiện có (ngoài phạm vi,
   không thêm auto-close ở đó).

## Acceptance Criteria
- [ ] Start feature mới (local, project bật online-pipeline) → issue GitHub mới xuất hiện với
      label `local-placeholder`, slug = `kebab(issue-title)-{issue-number}`, `work/{slug}` dùng
      đúng slug này.
- [ ] Start feature mới (local, project KHÔNG bật online-pipeline) → hành vi Start path không đổi
      (tự chọn slug, không tạo issue).
- [ ] `online-pipeline/SKILL.md` stage `userspec-turn` tự gọi Start path từ bên trong GitHub
      Actions job (kèm slug có sẵn, kể cả lần đầu hoàn toàn bình thường không liên quan hybrid
      switch) → KHÔNG tạo thêm issue mới nào, dùng đúng slug được cho.
- [ ] `gh issue create --label local-placeholder` trên 1 project đã bật online-pipeline từ TRƯỚC
      khi feature này tồn tại (label chưa có sẵn) → label được tự tạo defensively trước khi dùng,
      không lỗi "label not found".
- [ ] Máy local chưa auth `gh` CLI → Start feature mới trong project bật online-pipeline báo lỗi
      rõ, dừng, không tạo feature folder dở.
- [ ] Issue còn label `local-placeholder` → route job của `online-pipeline-userspec.yml` bỏ qua
      MỌI event (`issues: opened` và `issue_comment: created`) trên issue đó; không comment
      interview nào được tự động post trong lúc người dùng làm interview/implement ở local.
- [ ] Switch-to-online (ở userspec hoặc implement) gỡ label `local-placeholder` khỏi issue TRƯỚC
      khi đăng comment `/switch-online` — kể cả khi userspec làm hoàn toàn local và switch chỉ
      xảy ra lần đầu ở giữa implement.
- [ ] Switch-to-online giữa chừng interview (push `userspec/{slug}` + đăng `/switch-online` không
      kèm câu trả lời thật) → stage `userspec-turn` không append `/switch-online` làm câu trả lời
      cho gap đang dở; chỉ đăng lại đúng câu hỏi tiếp theo như 1 comment mới.
- [ ] Switch-to-online ở ranh giới userspec (spec approved local, chưa từng lên GitHub) → PR
      `userspec/{slug}` (label `userspec-spec`) được mở đúng, chờ `/approve` như luồng online bình
      thường.
- [ ] Switch-to-online giữa chừng implement → branch `feature/{slug}` có code hiện tại; nếu lần
      đầu tồn tại online, status marker được khởi tạo `awaiting_decision`; comment `/switch-online`
      kích hoạt đúng `implement-resume`, tiếp tục từ đúng commit, không làm lại từ đầu.
- [ ] Switch-to-online giữa implement cho feature làm hoàn toàn local (bỏ qua PR-spec-review,
      `feature/{slug}` chưa từng tồn tại online) → route job thật của
      `online-pipeline-implement.yml` dispatch đúng (không rơi vào `action=skip`); `implement-resume`
      nhận diện comment là `/switch-online` → chạy như bước 1 của stage `implement` gốc, không hiểu
      nhầm là câu trả lời cho câu hỏi cũ.
- [ ] Feature làm hoàn toàn local qua hết userspec rồi switch-to-online giữa implement → mọi
      comment SAU comment switch đầu tiên (vd câu trả lời cho 1 `awaiting_decision` lúc implement
      online) không bị workflow userspec xử lý nhầm thành yêu cầu bắt đầu interview mới, vì route
      job coi "userspec đã xong" dựa trên dấu hiệu thứ 3 (branch `feature/{slug}` tồn tại) được
      chèn đúng vào nhánh "branch `userspec/{slug}` không tồn tại" của logic nested if/else có sẵn
      — không phá luồng continue bình thường (branch `userspec/{slug}` tồn tại, chưa approved) của
      các feature khác không liên quan tới case bypass này.
- [ ] Branch push thành công nhưng đăng comment `/switch-online` lỗi (mất mạng, quyền `gh` không
      đủ) → dừng, báo lỗi rõ, không coi switch là đã thành công.
- [ ] Switch-to-online ở ranh giới implement (code xong local, chưa từng lên GitHub) → PR
      `feature/{slug}` (label `userspec-implement`) được mở đúng.
- [ ] Switch-to-local (đang dở interview qua comment GitHub) → fetch/checkout đúng branch, báo
      đúng gap/câu hỏi còn thiếu, tiếp tục interview loop bình thường.
- [ ] Switch-to-local (đang `awaiting_decision` giữa implement online) → fetch/checkout đúng
      `feature/{slug}`, đọc status marker, báo đúng finding cần quyết định, tiếp tục `code-writing`
      từ đúng trạng thái code hiện tại.
- [ ] Switch-to-local khi working tree đang có thay đổi chưa commit → dừng trước khi checkout, báo
      lỗi rõ, không mất thay đổi local.
- [ ] Conflict push khi switch (remote có commit mới) → tự fetch+merge/rebase; merge sạch thì tự
      push tiếp; conflict thật thì dừng, hiển thị đúng conflict cho người dùng xử lý trong chat.
- [ ] Feature hoàn tất ở local (bất kỳ lịch sử switch) → finalize local xong, issue placeholder tự
      đóng (`gh issue close`); nếu đóng thất bại, finalize vẫn coi là thành công, chỉ log cảnh báo.
- [ ] Finalize chạy online không bị ảnh hưởng — không thêm auto-close issue ở đó.
- [ ] Không có sửa đổi nào vào `on:` event trigger của 3 workflow YAML hiện có.

## Constraints
- Chỉ áp dụng hành vi "tạo issue ngay từ đầu + slug theo issue" cho project đã bật
  online-pipeline (phát hiện qua sự tồn tại của `.github/workflows/online-pipeline-userspec.yml`
  đã vendor); project không bật giữ nguyên Start path hiện tại.
- Hành vi "tự tạo issue" CHỈ kích hoạt khi Start path được gọi MÀ KHÔNG có slug nào truyền sẵn
  (người dùng gõ tương tác ở local). Khi `online-pipeline/SKILL.md` stage `userspec-turn` tự gọi
  Start path từ bên trong GitHub Actions job (luôn kèm slug có sẵn, theo đúng hợp đồng hiện có
  không đổi: "call Start path directly with the given slug, do not let it choose its own slug"),
  bỏ qua hoàn toàn nhánh tạo issue mới — nếu không, mỗi lần online-pipeline tự gọi Start path (kể
  cả lần đầu hoàn toàn bình thường, không liên quan hybrid switch) sẽ tự tạo thêm 1 issue thừa,
  phá vỡ luồng online-pipeline gốc.
- Label `local-placeholder` (mới, chưa tồn tại trên project đã bật online-pipeline từ trước) phải
  được tạo defensively giống đúng cách 2 label hiện có (`userspec-spec`, `userspec-implement`) đã
  làm: thêm vào `setup-online-pipeline.sh` (tạo 1 lần lúc setup) VÀ kiểm tra/tạo inline ngay trước
  khi dùng (`gh label list ... | grep -qx local-placeholder || gh label create local-placeholder`)
  ở chỗ gọi `gh issue create --label local-placeholder`.
- Không sửa `on:` event trigger của 3 workflow YAML hiện có — chỉ sửa điều kiện nội bộ của route
  job và bổ sung nhánh mới trong stage prompt của `online-pipeline/SKILL.md`:
  1. `online-pipeline-userspec.yml` route bỏ qua xử lý cho MỌI event khi issue có label
     `local-placeholder` (label được gỡ TRƯỚC khi đăng comment `/switch-online`, bất kể switch xảy
     ra ở giai đoạn nào).
  2. Logic phát hiện "userspec đã xong" có sẵn là nested if/else, KHÔNG phải 2 dấu hiệu OR phẳng:
     nếu branch `userspec/{slug}` KHÔNG tồn tại → đọc `status: approved` từ `user-spec.md` trên
     main; nếu branch TỒN TẠI → đọc `status: approved` từ chính branch đó (branch tồn tại mà chưa
     approved = "đang dở", phải trả continue, KHÔNG phải "đã xong"). Dấu hiệu thứ 3 — branch
     `feature/{slug}` tồn tại trên remote — được chèn vào ĐÚNG nhánh "branch `userspec/{slug}`
     KHÔNG tồn tại" (song song với check approved trên main), KHÔNG gộp OR phẳng với nhánh "branch
     tồn tại" để tránh phá luồng continue bình thường của các feature không liên quan.
  3. Stage prompt (không phải YAML, không phải route job bash) ở cả `userspec-turn` và
     `implement-resume` thêm nhánh nhận diện "comment kích hoạt chính là `/switch-online`" khỏi
     "comment là trả lời câu hỏi cụ thể" — `userspec-turn` thì không append làm câu trả lời,
     `implement-resume` thì chạy như bước 1 của `implement` gốc. Route job của cả 2 workflow
     **không** cần đọc nội dung comment để nhận diện điều này — dispatch của chúng vẫn dựa đúng
     vào branch-existence/label/status-marker như hiện có.
  4. Switch-to-online, lần đầu push `feature/{slug}` lên remote (branch chưa từng tồn tại online),
     tự khởi tạo file status marker với `status: awaiting_decision` — đúng giá trị route job thật
     của `online-pipeline-implement.yml` cần (`grep -q '^status: awaiting_decision'`) để dispatch
     qua `issue_comment`; không ghi đè nếu branch đã có marker từ trước.
- `code-writing` không có git/commit discipline nội tại — switch-to-online giữa implement phải tự
  áp đặt kỷ luật commit/push từ ngoài (switch action làm, giống cách `online-pipeline/SKILL.md` đã
  áp đặt cho automated runs); local tương tác bình thường thì không cần.
- Feature làm hoàn toàn local tới giữa implement (chưa từng có PR/branch thật trên GitHub) khi
  switch: bỏ qua bước dựng lại PR-spec-review hình thức, đẩy thẳng `feature/{slug}` kèm code dở.
- Không xử lý race condition giữa 2 lần switch liên tiếp nhanh (switch online rồi đổi ý switch
  ngay về local trước khi job GitHub Actions kịp chạy xong) — last-write-wins, chấp nhận rủi ro
  hãn hữu cho v1.
- Conflict push khi switch: tự `fetch` + thử `merge`/`rebase`; nếu merge sạch thì tự push tiếp;
  nếu conflict thật, dừng và đưa conflict ra cho người dùng tự xử lý trong chat.
- Switch-to-local: trước khi checkout branch remote, kiểm tra working tree local có thay đổi chưa
  commit; nếu có → dừng, báo lỗi rõ, bắt người dùng tự commit/stash trước (không tự động stash hộ).
- Máy local chưa cài/chưa auth `gh` CLI khi bắt đầu feature mới (project bật online-pipeline) → báo
  lỗi dừng, không fallback âm thầm về hành vi tự chọn slug.
- Feature hoàn tất hoàn toàn local → issue placeholder tự đóng khi finalize xong; lỗi đóng issue
  không rollback finalize.
- Gắn thẳng vào `online-pipeline/SKILL.md` (section mới cho audience "local user muốn switch"),
  không tạo skill riêng.
- `scripts/init-feature-folder.sh` không đổi chữ ký (vẫn nhận slug đã tính xong từ bên ngoài) —
  việc tạo issue + tính slug xảy ra TRƯỚC khi gọi script này.
- Không đụng `code-writing/SKILL.md` hay `documentation-writing/SKILL.md`.

## Risks
- **Risk 1:** Tạo issue ngay từ đầu cho MỌI feature local (kể cả feature chắc chắn chỉ làm local)
  gây nhiễu issue tracker nếu project có nhiều feature nhỏ. **Mitigation:** issue tự đóng khi
  finalize local hoàn tất; người dùng đã xác nhận chấp nhận đánh đổi này để đổi lấy khả năng switch
  tức thời bất kỳ lúc nào.
- **Risk 2:** `issue_comment` là trigger nghe mọi comment không phải bot vốn đã có sẵn — thêm token
  `/switch-online` giảm nguy cơ 1 comment thường bị hiểu nhầm thành lệnh switch, nhưng không loại
  trừ hoàn toàn (người dùng tự gõ nhầm chuỗi đó trong 1 comment thường). **Mitigation:** chấp nhận
  rủi ro nhỏ này, nhất quán với cách `ONLINE_PIPELINE_AUTOMATED` đã được xử lý (so khớp literal
  string, không suy luận ý định).
- **Risk 3:** Thêm 1 dấu hiệu thật (`local-placeholder` label) vào route job hiện có — nếu sai thứ
  tự gỡ label/đăng comment, có thể vô tình chặn luôn switch-to-online. **Mitigation:** quy tắc thứ
  tự (gỡ label trước, đăng comment sau) được ghi rõ thành constraint cứng, kiểm tra tĩnh được ở
  Agent Verification.
- **Risk 4:** Switch giữa implement khi feature làm hoàn toàn local (bỏ qua PR-spec-review hình
  thức) đồng nghĩa online không có bước review spec nào trước khi implement tiếp. **Mitigation:**
  chấp nhận vì spec đã được tự duyệt ở local qua đúng luồng `user-spec-planning` Step 6 trước đó.
- **Risk 5:** Race condition khi switch 2 lần liên tiếp nhanh có thể làm mất 1 phần cập nhật
  (last-write-wins). **Mitigation:** chấp nhận là hạn chế v1, hãn hữu trong thực tế dùng solo.

## Accepted Decisions
- Chọn gắn thẳng khả năng switch vào `online-pipeline/SKILL.md` thay vì tạo skill riêng, vì skill
  đó đã phục vụ 2 audience (human setup, automated stage) rồi — giảm số skill phải nhớ.
- Chọn tái dùng tối đa trigger GitHub Actions có sẵn (`issues: opened`, `issue_comment: created`,
  `pull_request: closed`) thay vì thêm trigger `push` mới, để giảm thay đổi vào workflow YAML và
  giảm rủi ro phá vỡ hành vi hiện có của online-pipeline gốc.
- Chọn tạo issue GitHub ngay từ đầu Start path cho MỌI feature local (project bật online-pipeline),
  không phải lựa chọn riêng theo feature, để switch có thể xảy ra tức thời bất kỳ lúc nào mà không
  cần dựng issue giữa đường.
- Chọn dùng 1 label (`local-placeholder`) làm cơ chế chặn race tạm thời, tách biệt hoàn toàn khỏi
  tín hiệu "userspec đã xong" (vốn vẫn dựa 100% vào các dấu hiệu thật có sẵn: branch tồn tại,
  spec approved trên main, hoặc — bổ sung mới — branch `feature/{slug}` tồn tại) — quyết định này
  được sửa lại 2 lần qua các vòng kiểm tra sau khi phát hiện gắn 2 ý nghĩa vào 1 label gây race
  vĩnh viễn cho case bypass.
- Chọn dùng chính giá trị `status: awaiting_decision` (không phải `in_progress` hay 1 marker mới)
  để khởi tạo status marker khi switch-to-online lần đầu giữa implement, vì đó là giá trị chính
  xác route job thật của `online-pipeline-implement.yml` đã luôn kiểm tra (`grep -q '^status:
  awaiting_decision'`) — tái dùng đúng gate có sẵn, không cần sửa YAML. Quyết định này được sửa lại
  sau khi phát hiện giá trị `in_progress` đề xuất ban đầu sẽ làm route job luôn trả về
  `action=skip`, khiến use case chủ lực (switch giữa implement) không bao giờ chạy được.
- Chọn dùng nội dung comment (`/switch-online` hay không) làm dấu hiệu DUY NHẤT để `implement-resume`
  phân biệt "switch-handoff" khỏi "trả lời câu hỏi cụ thể", bỏ hẳn điều kiện "không có status
  marker" đã đề xuất trước đó — vì switch-to-online luôn tự tạo marker trước khi comment, nên điều
  kiện "không tồn tại marker" không bao giờ đúng trong thực tế, tự mâu thuẫn với chính cơ chế.
- Chọn bỏ qua bước dựng lại PR-spec-review hình thức khi switch giữa implement cho feature làm
  hoàn toàn local (đã tự duyệt spec qua `user-spec-planning` Step 6), đổi lại phải bổ sung dấu hiệu
  thứ 3 (branch `feature/{slug}` tồn tại) vào logic phát hiện "userspec đã xong" của route job
  userspec, vì 2 dấu hiệu cũ (branch `userspec/{slug}`, `status: approved` trên main) không bao
  giờ được tạo ra cho đường đi này.
- Chọn dừng và báo lỗi (không tự động stash) khi switch-to-local gặp working tree local có thay
  đổi chưa commit, ưu tiên an toàn hơn tiện lợi — tránh mất việc đang làm do checkout đè lên.
- Chọn tự đóng issue placeholder khi finalize chạy ở local xong, áp dụng cho MỌI lịch sử switch
  (không chỉ case chưa từng switch) — tiêu chí phân biệt là NƠI finalize chạy (local), không phải
  lịch sử switch của feature; finalize online giữ nguyên hành vi cũ, ngoài phạm vi.
- Từ chối cơ chế xử lý race condition giữa 2 lần switch liên tiếp (khóa status marker, phát hiện
  xung đột) — chấp nhận last-write-wins cho v1, vì rủi ro hãn hữu trong thực tế dùng solo và thêm
  cơ chế sẽ phức tạp không tương xứng.
- Sửa lại (sau validation round 1 — phát hiện của adequacy): hành vi "tự tạo issue" trong Start
  path CHỈ kích hoạt khi KHÔNG có slug truyền sẵn, thay vì áp dụng không điều kiện cho mọi lần gọi
  Start path — vì `online-pipeline/SKILL.md` đã có sẵn hợp đồng tự gọi Start path từ bên trong
  GitHub Actions job với slug cho sẵn, và áp dụng không điều kiện sẽ tự tạo issue thừa mỗi lần đó,
  phá vỡ luồng online-pipeline gốc ngay cả khi không liên quan tới hybrid switch.
- Thêm yêu cầu tạo label `local-placeholder` defensively (sau validation round 1 — phát hiện của
  adequacy): giống đúng cách 2 label hiện có (`userspec-spec`, `userspec-implement`) đã được tạo
  trong `setup-online-pipeline.sh` và inline trong workflow YAML — tránh lỗi "label not found"
  trên project đã bật online-pipeline từ trước khi feature này tồn tại.
- Sửa lại (sau validation round 1 — phát hiện của skeptic): mô tả chính xác logic phát hiện
  "userspec đã xong" là nested if/else (không phải 2 dấu hiệu OR phẳng) — dấu hiệu thứ 3 phải chèn
  vào đúng nhánh "branch `userspec/{slug}` không tồn tại", không gộp OR phẳng với nhánh "branch
  tồn tại", để không phá luồng continue bình thường của các feature không liên quan tới case
  bypass.

## Testing

**Unit tests:** không cần — đây là workflow YAML (sửa điều kiện route job) và stage prompt trong
`online-pipeline/SKILL.md`, không có logic nghiệp vụ mới cần unit test, giống tiền lệ
`online-pipeline-github-actions` gốc.

**Integration tests:** không cần dạng tự động — phần cần kiểm tra là tương tác giữa GitHub Actions,
GitHub API, git, và các skill Claude Code hiện có, không phải logic có thể cô lập bằng mock hợp lý.

**E2E tests:** cần, thực hiện bằng chạy thử thủ công trên 1 repo demo có bật online-pipeline thật
(không phải bộ test tự động trong CI) — xem chi tiết ở Verification.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Đọc `online-pipeline/SKILL.md` sau khi sửa | Section mới cho switch tồn tại; token `/switch-online` được định nghĩa và dùng đúng chỗ trong cả 2 stage `userspec-turn` và `implement-resume` (lưu ý: đây là cơ chế mới riêng của feature này, KHÁC phạm vi `ONLINE_PIPELINE_AUTOMATED` hiện có — tín hiệu đó chỉ scope cho `userspec-turn`+`finalize`, không cho `implement`/`implement-resume`) |
| 2. Đọc `user-spec-planning/SKILL.md` Start path đã sửa | Nhánh tạo issue mới CHỈ áp dụng khi (a) phát hiện project có online-pipeline VÀ (b) Start path được gọi không kèm slug sẵn; khi `online-pipeline/SKILL.md` tự gọi Start path với slug cho sẵn, nhánh này không kích hoạt; không đổi hành vi khi project không bật online-pipeline |
| 3. Đọc route job của `online-pipeline-userspec.yml` | Điều kiện bỏ qua xử lý khi có label `local-placeholder` áp dụng cho CẢ `issues: opened` và `issue_comment: created`; logic "userspec đã xong" vẫn là nested if/else đúng cấu trúc thật, dấu hiệu thứ 3 (branch `feature/{slug}`) nằm đúng trong nhánh "branch userspec/{slug} không tồn tại", không gộp OR phẳng với nhánh "branch tồn tại" |
| 3b. Đọc `scripts/setup-online-pipeline.sh` và route job của `online-pipeline-userspec.yml`/`online-pipeline-implement.yml` | Label `local-placeholder` được tạo defensively (kiểm tra tồn tại trước, tạo nếu chưa có) giống đúng cách `userspec-spec`/`userspec-implement` đã làm |
| 4. Đọc route job của `online-pipeline-implement.yml` | Không có thay đổi nào vào gate `grep -q '^status: awaiting_decision'` hiện có — switch-to-online tự tạo marker đúng giá trị này, không cần sửa YAML |
| 5. Đọc đoạn switch-to-online trong `online-pipeline/SKILL.md` | Thứ tự thao tác đúng: gỡ label `local-placeholder` TRƯỚC, đăng comment `/switch-online` SAU; khởi tạo status marker `awaiting_decision` trước khi push lần đầu `feature/{slug}` |
| 6. Đọc đoạn switch-to-local | Có bước kiểm tra working tree sạch trước khi checkout; dừng và báo lỗi nếu có thay đổi chưa commit, không tự stash |
| 7. Đọc đoạn xử lý conflict push khi switch | Có bước `fetch`+`merge`/`rebase` trước khi báo conflict thật cho người dùng |
| 8. Đọc đoạn finalize local | Có lệnh `gh issue close`, lỗi đóng issue không chặn việc archive `work/{feature}` |
| 9. Kiểm tra `scripts/init-feature-folder.sh` | Chữ ký không đổi |
| 10. Đọc đoạn xử lý lỗi đăng comment `/switch-online` trong `online-pipeline/SKILL.md` | Có nhánh: branch push thành công nhưng comment lỗi → dừng, báo lỗi, không coi là switch thành công |

Agent chỉ kiểm tra được các mục tĩnh trên (file đúng chỗ, cú pháp hợp lệ, điều kiện route job đúng
tên biến/giá trị). Agent không thể tự tạo issue GitHub thật, chờ GitHub Actions chạy, hay tự đóng
vai cả 2 phía local/online trong 1 lần chạy — nên toàn bộ hành vi vận hành thật của switch (2
chiều, giữa chừng, ranh giới, cho cả 3 giai đoạn) chỉ được xác minh ở User Verification dưới đây.

### User Verification
Người dùng tự thực hiện trên 1 repo demo thật đã bật online-pipeline — lý do cần xác minh thủ
công: đây là luồng tương tác người-máy qua nhiều lượt comment/push thật, không thể mô phỏng bằng
agent chạy một lần.

1. Start feature mới ở local trong repo demo → issue GitHub mới xuất hiện với label
   `local-placeholder`, slug đúng `kebab(title)-{number}`.
2. Trả lời vài câu hỏi interview ở local, KHÔNG switch → xác nhận không có comment nào tự động
   xuất hiện trên issue (label vẫn chặn đúng).
3. Giữa lúc đang trả lời dở 1 batch câu hỏi, gọi switch-to-online → kiểm tra: label bị gỡ, branch
   `userspec/{slug}` xuất hiện trên remote, comment `/switch-online` xuất hiện trên issue, bot trả
   lời đúng câu hỏi tiếp theo (không lặp lại câu cũ, không coi `/switch-online` là câu trả lời).
4. Trả lời tiếp vài câu bằng comment GitHub (hoàn toàn online) → interview tiến triển đúng.
5. Giữa lúc interview còn dở trên GitHub, gọi switch-to-local → Claude Code CLI fetch/checkout
   đúng branch, báo đúng câu hỏi còn thiếu, tiếp tục trả lời ngay trong chat.
6. Hoàn tất interview, approve spec hoàn toàn ở local (KHÔNG mở PR online) → tiếp tục implement ở
   local vài file.
7. Giữa lúc implement dở, gọi switch-to-online → kiểm tra: branch `feature/{slug}` xuất hiện lần
   đầu trên remote với status marker `awaiting_decision`, label gỡ trước khi comment
   `/switch-online` xuất hiện, job GitHub Actions implement-resume chạy đúng (đọc spec/decisions,
   tiếp tục code hiện tại, KHÔNG hỏi lại gì, KHÔNG báo lỗi `action=skip`).
8. Trong lúc đó, đăng tiếp 1 comment bất kỳ khác trên issue (không phải `/switch-online`) → xác
   nhận route job userspec KHÔNG xử lý nhầm (vì branch `feature/{slug}` đã tồn tại).
9. Cố ý tạo conflict push khi switch (branch remote có commit mới local chưa có) → xác nhận tự
   `fetch`+`merge` nếu sạch; cố ý tạo conflict thật → xác nhận bị đưa ra cho người dùng tự xử lý.
10. Cố ý để working tree local có thay đổi chưa commit rồi gọi switch-to-local → xác nhận dừng,
    báo lỗi, không mất thay đổi.
11. Để implement hoàn tất online, merge PR code → finalize chạy online bình thường (không bị ảnh
    hưởng bởi feature này).
12. Lặp lại toàn bộ vòng đời cho 1 feature khác, lần này hoàn tất **finalize ở local** → xác nhận
    issue placeholder tự đóng.
13. Cố ý làm `gh issue close` lỗi lúc finalize local (vd mất quyền tạm thời) → xác nhận finalize
    vẫn hoàn tất (archive xảy ra), chỉ log cảnh báo không đóng được issue.
14. Thử Start feature mới khi máy chưa auth `gh` CLI → xác nhận báo lỗi rõ, dừng, không tạo feature
    folder dở.
15. Start 1 feature trong project KHÔNG bật online-pipeline → xác nhận hành vi Start path không
    đổi (tự chọn slug, không tạo issue).
16. Mở 1 issue trực tiếp trên GitHub (luồng online-pipeline gốc, không qua Start path local) →
    `userspec-turn` tự gọi Start path với slug tính từ issue này → xác nhận KHÔNG có issue thứ 2
    nào được tạo thêm (nhánh "tự tạo issue" không kích hoạt vì slug đã được cho sẵn).
17. Trên project đã bật online-pipeline từ TRƯỚC khi feature này tồn tại (label `local-placeholder`
    chưa có sẵn trong repo) → Start feature mới ở local → xác nhận label được tự tạo, không lỗi
    "label not found".
18. Cố ý làm bước đăng comment `/switch-online` lỗi ngay sau khi branch đã push thành công (vd cắt
    mạng ngay sau push) → xác nhận switch không bị coi là thành công, trạng thái local vẫn được
    giữ làm nguồn đáng tin.
