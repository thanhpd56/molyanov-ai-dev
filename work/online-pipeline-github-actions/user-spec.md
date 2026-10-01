---
# Creation date (YYYY-MM-DD)
created: 2026-10-01

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: online-pipeline-github-actions

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building
Một skill mới `online-pipeline` cho framework `molyanov-ai-dev`, cho phép một dự án 1-repo chạy
toàn bộ vòng đời feature (request → interview → approve → implement → PR → finalize) hoàn toàn qua
GitHub (issue, comment, PR), không cần mở máy tính hay chạy `claude` CLI local ở bất kỳ bước nào.
Skill tái dùng phần lớn template/script của `user-spec-planning` nguyên vẹn, chỉ bổ sung các cơ
chế vận hành cần cho môi trường GitHub Actions không có trạng thái phiên liên tục giữa các lần
chạy — và một sửa đổi nhỏ, có kiểm soát (gated bởi tín hiệu rõ ràng) vào `user-spec-planning` và
`documentation-writing` để hỗ trợ đúng hành vi tự động.

## Why
Hiện tại, methodology của `molyanov-ai-dev` bắt buộc người dùng ngồi máy tính, mở một phiên Claude
Code CLI cho mọi bước (plan, interview, approve, implement). Người dùng muốn tạo và theo dõi tiến
độ feature mọi lúc mọi nơi (điện thoại, trình duyệt), không phụ thuộc vào việc máy tính có đang bật
hay không.

## Expected Behavior
1. Người dùng mở một issue mới trên repo đã bật `online-pipeline` — đây là request.
2. Bot comment câu hỏi interview đầu tiên lên issue. Người dùng trả lời bằng comment; mỗi lượt
   hỏi-đáp được commit vào một branch riêng (`userspec/{slug}`) để trạng thái sống sót qua các lần
   chạy job riêng biệt.
3. Khi interview đủ, bot mở một PR (branch `userspec/{slug}`, label `userspec-spec`) chứa
   `user-spec.md` để người dùng xem trước. Trong lúc draft/validate, nếu một reviewer cần người
   dùng quyết định, bot comment câu hỏi đó lên issue và dừng; người dùng trả lời bằng comment, bot
   tiếp tục đúng vòng đó.
4. Người dùng comment `/approve` trên PR spec. Bot merge PR đó vào main (không chỉ đóng), để
   `user-spec.md`/`decisions.md` nằm ở một vị trí cố định trên main.
5. Bot chạy job implement: `claude -p` không tương tác, thực hiện `user-spec.md` đã duyệt, mở một PR
   code (branch `feature/{slug}`, label `userspec-implement`) trong cùng repo.
   - Nếu job implement gặp lỗi kỹ thuật (Claude lỗi, push conflict, hết quota): tự động retry tối đa
     2 lần, mỗi lần reset branch về trạng thái sạch (force-push) trước khi làm lại từ đầu. Sau 2 lần
     vẫn lỗi: comment báo lỗi lên issue và dừng.
   - Nếu quá trình review trong lúc implement (`code-reviewer`/`security-auditor`) phát hiện cần
     người dùng quyết định (không phải lỗi kỹ thuật): bot commit phần đã làm xong, comment câu hỏi
     lên issue, dừng — không tính vào 2 lần retry. Người dùng trả lời, bot tiếp tục từ đúng branch đó
     (không làm lại từ đầu).
6. Người dùng review và merge PR code. Việc merge PR có label `userspec-implement` tự động kích
   hoạt job finalize, được gọi kèm câu tín hiệu cố định `ONLINE_PIPELINE_AUTOMATED` để bỏ qua bước "hỏi
   tiếp tục nếu thấy thiếu sót": cập nhật Project Knowledge, chuyển `work/{feature}/` sang
   `work/completed/{feature}/`. Finalize commit thẳng vào `main` (không có branch riêng); nếu lỗi
   trước khi tới bước commit cuối cùng, `main` chưa bị đụng tới nên chỉ cần **chạy lại từ đầu**
   (fresh checkout, không reset gì); nếu lỗi *sau* khi đã commit/push xong, lần chạy lại tự phát
   hiện `work/{feature}/` đã archive xong và dừng ngay. Auto-retry tối đa 2 lần, sau đó comment báo
   lỗi lên issue.
7. Hai issue (hai feature) mở gần nhau trên cùng repo chạy độc lập, không chặn nhau (concurrency
   group riêng theo từng issue); nếu hai job finalize trùng thời điểm, chúng chạy tuần tự (chung một
   concurrency group cố định), không ghi đè nhau lên Project Knowledge.

## Acceptance Criteria
- [ ] Mở issue mới trên repo đã bật `online-pipeline` → nhận được comment câu hỏi interview đầu
      tiên.
- [ ] Trả lời đủ các vòng câu hỏi → nhận được PR (label `userspec-spec`) chứa `user-spec.md` để xem
      trước.
- [ ] Comment `/approve` trên PR đó → PR spec được **merge** vào main (không chỉ đóng), job implement
      bắt đầu.
- [ ] Job implement thành công → PR code (label `userspec-implement`) xuất hiện trong cùng repo.
- [ ] Job implement thất bại do lỗi kỹ thuật → tự động retry tối đa 2 lần, mỗi lần **reset branch về
      commit gần nhất đã lưu** (force-push) — không phải về điểm khởi tạo branch, nên không xoá mất
      checkpoint từ một lượt dừng-hỏi-quyết-định trước đó; vẫn lỗi sau 2 lần → comment báo lỗi lên
      issue và dừng.
- [ ] Trong lúc validate spec, nếu reviewer cần người dùng quyết định → bot comment câu hỏi lên
      issue, lưu lại số vòng đang validate + câu hỏi, dừng (validate vốn không có cơ chế auto-retry
      nên không có gì để "không tính vào"); trả lời bằng comment → bot **chạy lại toàn bộ đúng vòng
      đó** (gọi lại cả 3 reviewer của vòng đó, không phải vòng 1), lần này có câu trả lời.
- [ ] Trong lúc implement, nếu review (`code-reviewer`/`security-auditor`) cần người dùng quyết
      định → bot commit phần đã làm xong, comment câu hỏi lên issue, dừng — **không** tính vào 2
      lần retry dành cho lỗi kỹ thuật của implement; trả lời bằng comment → bot tiếp tục đúng từ
      branch đó, không làm lại từ đầu.
- [ ] PR code merge → job finalize tự động chạy: cập nhật Project Knowledge, chuyển `work/{feature}/`
      sang `work/completed/{feature}/`, không hỏi lại xác nhận.
- [ ] Merge PR spec (lúc approve) **không** tự kích hoạt finalize — chỉ PR code (đúng label/branch)
      mới kích hoạt.
- [ ] Job finalize thất bại trước khi commit → tự động chạy lại từ đầu (không reset gì, vì `main`
      chưa bị đụng tới), tối đa 2 lần, sau đó comment báo lỗi lên issue.
- [ ] Job finalize thất bại **sau khi** đã commit/push xong (vd mất mạng ngay sau đó) → lần retry
      tiếp theo tự phát hiện `work/{feature}/` đã archive xong trên main, dừng ngay, không chạy lại.
- [ ] Hai issue (hai feature) mở gần nhau trên cùng repo chạy độc lập không chặn nhau; hai job
      finalize trùng thời điểm chạy tuần tự, không ghi đè nhau lên Project Knowledge.
- [ ] Hoạt động cả trên project có sẵn (bật online-pipeline độc lập) và project mới tạo (tích hợp vào
      `project-initialization`).
- [ ] Comment do chính bot tạo ra (câu hỏi, thông báo, báo lỗi, hỏi quyết định) **không** tự kích
      hoạt lại một lần chạy job mới.

## Constraints
- Hỗ trợ cả hai cách xác thực: Claude subscription OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`) và
  Anthropic API key (`ANTHROPIC_API_KEY`) — người dùng chọn lúc setup, không cố định cứng.
- Mọi job gọi `claude -p` (interview, draft/validate, implement, finalize) phải chạy ở chế độ
  **bypass permissions hoàn toàn** (`--dangerously-skip-permissions`, tương đương
  `permissions.defaultMode: bypassPermissions`) — thay vì liệt kê tên tool được phép, vì tên tool
  chính xác (vd `Skill`, `Agent` hay `Task`) có thể khác nhau giữa các phiên bản Claude Code CLI và
  đoán sai sẽ tái diễn đúng lỗi "treo chờ xác nhận". Đánh đổi: bypass mode không có lớp bảo vệ khỏi
  prompt injection hay hành động ngoài ý muốn — chấp nhận được vì mỗi job chạy trên runner GitHub
  Actions dùng 1 lần rồi huỷ, trên repo private, không có dữ liệu không tin cậy nào khác trong
  workspace đó.
- Tái dùng `assets/user-spec.md.template`, `assets/decisions.md.template`,
  `scripts/init-feature-folder.sh` của `user-spec-planning` nguyên vẹn, không tạo định dạng riêng,
  để các reviewer agent (`skeptic`, `userspec-quality-validator`, `userspec-adequacy-validator`,
  `interview-completeness-checker`) hoạt động không đổi. Riêng `assets/interview.yml.template` và
  **Step 5 "Validate the User Spec"** của `skills/user-spec-planning/SKILL.md` có một ngoại lệ đã
  thống nhất: thêm field mới lưu trạng thái vòng validate (xem chi tiết ở phần Accepted Decisions)
  — chỉ kích hoạt khi có tín hiệu `ONLINE_PIPELINE_AUTOMATED`, không đổi hành vi/format khi dùng
  local. **Step 3 "Check Interview Completeness"** không cần sửa — nó chạy 1 lượt duy nhất, không
  có khái niệm "vòng", và đã tự lưu trạng thái qua đúng cơ chế commit-mỗi-lượt sẵn có.
  `decisions.md.template` không đổi.
- Mỗi lượt interview phải commit+push `interview.yml` lên branch riêng (`userspec/{slug}`) ngay sau
  lượt đó — vì mỗi lần GitHub Actions chạy là một checkout hoàn toàn mới, không có trạng thái phiên
  giữa các lần chạy (đã xác nhận qua tài liệu `claude-code-action`: "continuing conversations" chưa
  phải tính năng có sẵn).
- Mọi job được trigger bởi `issue_comment: created` phải lọc bỏ comment do chính bot tạo ra (vd
  `github.event.comment.user.type != 'Bot'`), để tránh bot tự kích hoạt lại chính nó.
- Không kiểm tra actor cho lệnh `/approve` — rủi ro được chấp nhận với điều kiện repo giữ ở chế độ
  private (solo dev).
- `/approve` merge PR spec vào main (không chỉ đóng), để `user-spec.md`/`decisions.md` nằm ở vị trí
  cố định cho implement và finalize đọc.
- PR spec (branch `userspec/{slug}`, label `userspec-spec`) và PR code (branch `feature/{slug}`,
  label `userspec-implement`) phải phân biệt được bằng label/branch, vì cả hai đều merge vào main
  qua cùng loại sự kiện (`pull_request: closed`, `merged=true`) — job finalize chỉ trigger theo
  label/branch của PR code.
- Job implement auto-retry tối đa 2 lần khi gặp lỗi kỹ thuật, **reset branch riêng về commit gần
  nhất đã lưu trên branch đó (force-push)** trước mỗi lần retry — không phải reset về điểm khởi tạo
  branch. "Commit gần nhất" có thể là điểm bắt đầu implement (nếu chưa có gì được commit) hoặc một
  checkpoint đã commit từ một lượt dừng-hỏi-quyết-định trước đó trong cùng job (xem mục dưới) —
  cách này chỉ xoá đúng phần dở/hỏng do lỗi gây ra, không xoá mất quyết định người dùng đã trả lời
  trước đó. Áp dụng đồng nhất cho mọi nguyên nhân lỗi (Claude error, push conflict, hết quota),
  không phân nhánh theo loại lỗi (đánh đổi: tốn thêm token khi lỗi do push conflict, đổi lại đơn
  giản hơn trong v1).
- Job finalize auto-retry tối đa 2 lần khi gặp lỗi kỹ thuật, nhưng **không** force-push/reset gì —
  vì finalize commit thẳng vào `main` (không có branch riêng); force-push `main` sẽ xoá mất commit
  hợp lệ của feature khác đang chạy song song. Một lỗi trước bước commit cuối cùng không để lại dấu
  vết trên `main`, nên retry chỉ cần chạy lại job từ một checkout mới. Trước mỗi lần retry, job tự
  kiểm tra `work/{feature}/` đã chuyển sang `work/completed/{feature}/` trên main chưa — nếu rồi
  (nghĩa là lần chạy trước thực ra đã thành công, chỉ lỗi sau khi commit/push xong), coi như đã
  hoàn tất và dừng, không chạy lại.
- Dùng **một tín hiệu cố định chung** cho mọi lệnh gọi `claude -p` trong ngữ cảnh tự động, không có
  người trả lời: `ONLINE_PIPELINE_AUTOMATED`. Cả 2 skill dùng chung (`documentation-writing` cho
  finalize, `user-spec-planning` cho validate) chỉ bật nhánh hành vi tự động khi nhận diện đúng
  literal chuỗi này trong lệnh gọi, không tự suy luận "có phải đang chạy tự động hay không". Lúc
  chạy tương tác (local, không có tín hiệu này), hành vi của cả 2 skill giữ nguyên như cũ.
- Scope bao gồm thêm một sửa đổi nhỏ vào `skills/user-spec-planning/SKILL.md` **Step 5 "Validate
  the User Spec"** (không đụng Step 3): khi nhận diện tín hiệu `ONLINE_PIPELINE_AUTOMATED` và một
  reviewer phát hiện `user_decision_required`, trước khi comment hỏi và dừng, lưu lại **số vòng
  validate hiện tại** (vd vòng 2) + **finding đang chờ** vào `interview.yml` (field mới, không có
  trong template gốc) — không cố lưu lại các sửa dở/chưa commit của 2 reviewer khác trong cùng
  vòng đó, vì `user-spec-planning` chỉ commit 1 lần sau khi cả vòng xong, không commit riêng từng
  finding. Lần chạy sau, khi người dùng đã trả lời, job đọc field đó và **chạy lại toàn bộ đúng
  vòng N đang dở** (gọi lại đủ cả 3 reviewer của vòng N, lần này có câu trả lời) — không chạy lại
  từ vòng 1, nhưng cũng không cố "nhớ" phần việc dở của 2 reviewer đã qua trong vòng đó; rẻ hơn
  restart-từ-vòng-1 nhưng đơn giản hơn so với cố giữ trạng thái chưa commit. Hành vi tương tác
  (local) giữ nguyên không đổi — cơ chế này chỉ áp dụng khi có tín hiệu tự động.
- Interview-turn và draft/validate-round khi gặp **lỗi kỹ thuật** **không** auto-retry — job fail
  thẳng, người dùng tự re-trigger bằng cách comment lại (vì mỗi lượt đã sẵn comment-triggered).
- Một finding `user_decision_required` (không phải lỗi kỹ thuật) được xử lý khác với lỗi, ở cả 2
  nơi có thể xảy ra: lúc validate (vốn không có auto-retry nào cả — chỉ đơn giản comment hỏi và
  dừng, rồi tiếp tục đúng vòng đang dở nhờ field mới nói trên) và lúc implement (có auto-retry cho
  lỗi kỹ thuật, nên cần nói rõ: bot commit phần đã làm xong, comment câu hỏi, dừng — không tính vào
  2 lần retry dành cho lỗi kỹ thuật của implement, tiếp tục từ branch đã commit khi người dùng trả
  lời).
- Mỗi feature chạy trên concurrency group riêng theo số issue (song song không giới hạn số lượng);
  mọi job finalize chia sẻ một concurrency group cố định để chạy tuần tự khi trùng thời điểm.
  Concurrency group theo issue dùng chế độ **chờ** (`cancel-in-progress: false`), không hủy job
  đang chạy — nếu 2 comment chồng lên cùng 1 issue, comment sau phải đợi job của comment trước chạy
  xong (đã commit/push `interview.yml` an toàn) rồi mới xử lý, tránh ngắt job giữa lúc đang commit.
- Dùng OAuth subscription cho nhiều feature song song có trần rate/concurrency phía Anthropic —
  chấp nhận là giới hạn đã biết của v1, không có cơ chế queue tự động; người dùng tự chuyển sang API
  key nếu gặp giới hạn này.
- Scope bao gồm một sửa đổi nhỏ, tương thích ngược vào `skills/documentation-writing/SKILL.md`
  (Feature Finalization Mode): thêm một nhánh rõ ràng cho ngữ cảnh gọi tự động, không có người trả
  lời, bỏ qua bước "hỏi tiếp tục nếu thấy thiếu sót" và luôn tiếp tục finalize. Hành vi hiện có khi
  gọi tương tác (local, có người trả lời) giữ nguyên không đổi.
- Setup ban đầu cho 1 repo gồm 2 bước không thể tự động hoá (đòi hỏi xác thực qua trình duyệt của
  chính người dùng): cài Claude GitHub App vào repo, và tạo OAuth token (`claude setup-token`) hoặc
  API key. Phần còn lại (thêm secret, copy workflow YAML) được tự động hoá bằng một script setup.

## Risks
- **Risk 1:** Chi phí API/quota tăng vì chạy nhiều feature song song cộng với tối đa 2 lần retry mỗi
  job lỗi. **Mitigation:** Giữ trần retry = 2; không giới hạn số feature song song theo lựa chọn của
  người dùng.
- **Risk 2:** `/approve` không kiểm tra actor — nếu lỡ để repo public, bất kỳ ai cũng approve được.
  **Mitigation:** Khuyến cáo/yêu cầu giữ repo ở chế độ private; không có enforcement tự động trong
  workflow.
- **Risk 3:** Dùng OAuth subscription cho nhiều feature song song có thể chạm trần rate/concurrency
  của Anthropic. **Mitigation:** Ghi nhận là giới hạn đã biết của v1; người dùng tự chuyển sang API
  key nếu gặp giới hạn, không có cơ chế queue tự động.
- **Risk 4:** Chạy mọi job ở chế độ bypass permissions mất lớp bảo vệ khỏi prompt injection hoặc
  hành động ngoài ý muốn trong job đó. **Mitigation:** Mỗi job chạy trên runner GitHub Actions dùng
  1 lần rồi huỷ, trên repo private, không xử lý dữ liệu/code không tin cậy từ bên ngoài.

## Accepted Decisions
- Chọn tái dùng nguyên vẹn template/script của `user-spec-planning` thay vì tạo định dạng riêng cho
  luồng online, vì giữ tương thích với 4 reviewer agent hiện có và tránh lệch dần giữa luồng local và
  online theo thời gian.
- Chọn hỗ trợ cả OAuth subscription token và API key, để người dùng tự chọn theo nhu cầu, thay vì cố
  định một cách.
- Từ chối phương án xoay vòng nhiều tài khoản Claude cá nhân để tăng quota tự động hoá, vì nằm ở vùng
  xám so với điều khoản sử dụng cá nhân của Claude.ai; khuyến nghị dùng API key nếu cần thêm
  throughput.
- Chọn skill ship cả hai dạng: standalone (bật cho project có sẵn) và tích hợp vào
  `project-initialization` (project mới), theo đúng yêu cầu dùng cho cả hai trường hợp.
- Chọn không kiểm tra actor cho `/approve`, chấp nhận rủi ro vì người dùng là solo dev và repo dự
  kiến private.
- Chọn `/approve` merge PR spec vào main (thay vì chỉ đóng), để implement và finalize có một vị trí
  cố định đọc `user-spec.md`/`decisions.md`.
- Chọn coi finding `user_decision_required` giữa lúc implement giống hệt cơ chế đã có cho lúc
  validate (comment hỏi, dừng, tiếp tục từ branch đã commit) thay vì để job tự quyết định hoặc treo
  vô ích trong 2 lần retry, vì retry không giải quyết được một câu hỏi cần quyết định của người dùng.
- Chọn reset-and-restart đồng nhất cho mọi nguyên nhân lỗi của job implement (không phân nhánh theo
  loại lỗi như "chỉ push conflict mới rebase-retry-push"), đánh đổi lấy đơn giản trong v1 — người
  dùng xác nhận chấp nhận tốn thêm token ở trường hợp push conflict.
- Sửa lại (sau validate round 1 — phát hiện lỗi khi draft): "reset branch bằng force-push" **chỉ**
  áp dụng cho job implement (có branch riêng, an toàn để reset). Finalize commit thẳng vào `main`,
  nên áp dụng force-push cho finalize sẽ xoá mất commit hợp lệ của feature khác — finalize retry chỉ
  chạy lại job, không reset gì.
- Chọn hỗ trợ nhiều feature song song ngay trong lần nâng cấp này (không để ngoài phạm vi), với
  finalize được serialize qua một concurrency group chung để tránh ghi đè Project Knowledge.
- Chọn finalize luôn chạy khi PR code merge, không giữ lại bước "hỏi tiếp tục finalize không nếu
  phát hiện thiếu sót" của `documentation-writing`, vì việc merge đã là một xác nhận ngầm của người
  dùng. Vì bước đó nằm trong `documentation-writing` (skill dùng chung cho nhiều việc khác, không
  riêng online-pipeline), cần một cơ chế rõ ràng thay vì chỉ dựa vào chỉ dẫn prompt để "ghi đè" hành
  vi đã viết sẵn — chọn phương án: thêm một nhánh rõ ràng vào Feature Finalization Mode của
  `documentation-writing/SKILL.md`, thay vì để online-pipeline tự viết logic finalize rút gọn
  riêng. Minh bạch và verify được, đổi lại là một thay đổi nhỏ vào skill dùng chung — chấp nhận vì
  nhánh mới không đổi hành vi hiện có khi gọi tương tác như trước.
- Sửa lại (sau validate round 2 — phát hiện rủi ro suy luận sai): nhánh tự động trong
  `documentation-writing` không dựa vào việc model tự suy luận "đây có phải ngữ cảnh tự động
  không", vì nếu suy luận sai sẽ rơi lại đúng tình trạng treo máy mà bản sửa này muốn tránh. Thay
  vào đó dùng một câu tín hiệu cố định, rõ ràng (`ONLINE_PIPELINE_AUTOMATED`) mà online-pipeline
  luôn kèm theo khi gọi tự động — giống bật/tắt một cờ (flag), không mơ hồ.
- Sửa lại (sau validate round 4 — phát hiện mâu thuẫn với hành vi thật của `user-spec-planning`):
  lời hứa "tiếp tục đúng vòng validate đang dở" không thể đạt được nếu chỉ tái dùng
  `user-spec-planning/SKILL.md` nguyên vẹn, vì chính skill đó quy định một phiên mới sau khi bị
  dừng sẽ validate lại từ vòng 1, không khôi phục phản hồi reviewer cũ — mà mỗi lần GitHub Actions
  chạy luôn là một phiên mới. Giữa 2 hướng (chấp nhận chạy lại từ vòng 1 — giữ "tái dùng nguyên
  vẹn", hay sửa thêm `user-spec-planning` để lưu/khôi phục đúng vòng — giữ đúng lời hứa ban đầu),
  người dùng chọn **sửa thêm `user-spec-planning`**, dùng chung tín hiệu `ONLINE_PIPELINE_AUTOMATED`
  với nhánh tự động của `documentation-writing` thay vì tạo 2 tín hiệu riêng cho cùng một khái niệm
  ("được gọi tự động, không ai trả lời").
- Thêm cơ chế idempotency cho finalize retry (sau validate round 2): trước khi retry, job tự kiểm
  tra `work/{feature}/` đã archive xong trên main chưa; nếu rồi thì dừng, không chạy lại — tránh
  trường hợp lỗi xảy ra *sau* khi commit/push đã thành công (job chỉ lỗi lúc đang thoát) khiến retry
  chạy nhầm trên trạng thái đã hoàn tất.
- Thêm yêu cầu permission mode không tương tác cho mọi job `claude -p` (sau validate round 2): nếu
  không cấu hình, job có thể treo chờ xác nhận mà không ai trả lời được trong môi trường tự động.
- Sửa lại (sau validate round 4 — phát hiện rủi ro đoán sai tên tool): dùng **bypass permissions
  hoàn toàn** cho mỗi job, thay vì liệt kê tên tool cụ thể (vì tên tool thật có thể đã đổi/khác
  phiên bản CLI, đoán sai sẽ tái diễn đúng lỗi treo máy). Đánh đổi mất lớp bảo vệ injection, chấp
  nhận vì mỗi job chạy trên runner dùng 1 lần, repo private.
- Concurrency theo issue dùng chế độ chờ (`cancel-in-progress: false`), không hủy job đang chạy
  (sau validate round 2): tránh 2 comment chồng lên cùng issue làm ngắt job đang commit
  `interview.yml` giữa chừng, có thể làm hỏng file đó.
- Chọn phân biệt PR spec và PR code bằng branch (`userspec/{slug}` vs `feature/{slug}`) và label
  (`userspec-spec` vs `userspec-implement`), vì cả hai merge vào main qua cùng loại sự kiện GitHub
  và finalize cần biết chỉ nên trigger theo PR code.
- Chọn lọc comment theo tác giả (loại trừ comment của chính bot) trên mọi job nghe `issue_comment`,
  để tránh bot tự kích hoạt lại chính mình.
- Chọn chấp nhận giới hạn rate/concurrency của OAuth subscription khi chạy nhiều feature song song
  là một hạn chế đã biết của v1, không xây cơ chế queue tự động.
- Chọn tiêu chí "xong" là chạy thử toàn bộ vòng đời trên một repo demo nhỏ thật, không viết unit test
  cho các bash script/workflow YAML.
- Sửa lại (sau validate round 5 — phát hiện của skeptic): chỉ sửa **Step 5** của
  `user-spec-planning/SKILL.md`, không đụng Step 3 — Step 3 chạy 1 lượt duy nhất, không có khái
  niệm "vòng" nên không có gì bị mất khi phiên kết thúc; `decisions.md.template` không đổi.
- Sửa lại (sau validate round 5 — phát hiện của adequacy): đơn giản hoá cơ chế "tiếp tục đúng vòng"
  — chỉ lưu số vòng + câu trả lời của người dùng, không cố lưu các sửa chưa commit của reviewer
  khác trong cùng vòng (vì `user-spec-planning` chỉ commit 1 lần/vòng, không commit riêng từng
  finding). Khi resume, chạy lại **toàn bộ vòng đang dở** (không phải vòng 1) với câu trả lời đã có
  — rẻ hơn restart-từ-vòng-1, đơn giản hơn so với cố giữ trạng thái chưa commit.
- Sửa lại (sau validate round 5 — phát hiện của adequacy): "reset về sạch" khi implement lỗi nghĩa
  là reset về **commit gần nhất trên branch**, không phải điểm khởi tạo branch — để không xoá mất
  một quyết định người dùng đã trả lời và được commit trước đó trong cùng job.

## Testing

**Unit tests:** không cần — đây là workflow YAML và bash script điều phối (copy file, gọi `gh`
  CLI, `git`), không có logic nghiệp vụ mới cần unit test.

**Integration tests:** không cần dạng tự động — xác minh bằng cách chạy thử thật trên 1 repo demo
  (xem Verification), vì phần cần kiểm tra là tương tác giữa GitHub Actions, GitHub API, và các skill
  Claude Code hiện có, không phải logic có thể cô lập bằng mock hợp lý.

**E2E tests:** cần, thực hiện bằng chạy thử thủ công trên repo demo thật (không phải một bộ test tự
  động trong CI) — xem chi tiết ở phần Verification.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Chạy `setup-online-pipeline.sh` trên một repo demo nhỏ (không cần GitHub App/token thật) | Workflow YAML được copy đúng vào `.github/workflows/`, cú pháp YAML hợp lệ (lint), `gh secret set` được gọi đúng cú pháp, script in ra đúng 2 bước thủ công còn lại (cài GitHub App, tạo token) |
| 2. Đọc lại các workflow file đã sinh ra | Tên branch/label dùng đúng theo spec (`userspec/{slug}`+`userspec-spec`, `feature/{slug}`+`userspec-implement`); mọi job nghe `issue_comment` có điều kiện lọc tác giả bot |
| 3. Kiểm tra sửa đổi trong `documentation-writing/SKILL.md` | Nhánh tự động mới không đổi hành vi nhánh tương tác hiện có; nhánh mới nhận diện đúng literal tín hiệu `ONLINE_PIPELINE_AUTOMATED` |
| 4. Tạo project mới qua `project-initialization` với online-pipeline được bật | Scaffold project mới có đủ file workflow + hướng dẫn setup (kiểm tra tĩnh, không chạy vòng đời thật) |
| 5. Đọc workflow file của **cả 4 loại job** (interview, draft/validate, implement, finalize) | Mỗi loại job đều khai báo chế độ bypass permissions hoàn toàn (`--dangerously-skip-permissions` hoặc tương đương), không riêng implement/finalize |
| 6. Đọc khối `concurrency:` của workflow theo issue | Có `cancel-in-progress: false` (không phải mặc định hủy job) |
| 7. Đọc prompt/script gọi finalize | Có chứa đúng literal tín hiệu `ONLINE_PIPELINE_AUTOMATED` được truyền vào lệnh gọi `claude -p` |
| 8. Đọc workflow file của **cả 4 loại job** (interview-turn, draft/validate, implement, finalize) | Không có cấu hình retry nào gắn vào job interview-turn hoặc draft/validate-round (chỉ implement và finalize mới có retry) |
| 9. Đọc workflow file | Có branch rõ ràng cho cả 2 secret `CLAUDE_CODE_OAUTH_TOKEN` và `ANTHROPIC_API_KEY` khi gọi `claude -p`, không hardcode chỉ 1 trong 2 |
| 10. Kiểm tra sửa đổi trong `user-spec-planning/SKILL.md` Step 5 và template `interview.yml` | Có nhánh mới nhận diện literal tín hiệu `ONLINE_PIPELINE_AUTOMATED`, có field lưu số vòng validate đang dở + finding đang chờ quyết định; Step 3 và `decisions.md.template` không bị đổi |

Agent chỉ kiểm tra được các mục tĩnh trên (file đúng chỗ, cú pháp hợp lệ, cấu hình đúng tên). Agent
không thể tự tạo issue GitHub thật, chờ nhiều ngày, hay tự trả lời comment hộ người dùng — nên toàn
bộ hành vi vận hành thật của pipeline (interview, approve, implement, retry, finalize, song song)
chỉ được xác minh ở User Verification dưới đây, không lặp lại ở bảng trên.

### User Verification
Người dùng tự thực hiện trên một repo demo thật (điện thoại hoặc trình duyệt, không cần mở
terminal) — lý do cần xác minh thủ công: đây là luồng tương tác người-máy qua nhiều lượt comment
thật, không thể mô phỏng bằng agent chạy một lần.

1. Mở issue mới trên repo demo → nhận comment câu hỏi interview đầu tiên.
2. Trả lời đủ các vòng câu hỏi → nhận PR (label `userspec-spec`) chứa `user-spec.md`.
3. Comment `/approve` → PR spec được **merge** vào main (kiểm tra main có `user-spec.md`, không chỉ
   đóng PR); job implement bắt đầu.
4. Theo dõi job implement chạy thành công → PR code (label `userspec-implement`) xuất hiện.
5. Cố ý làm job implement lỗi 1 lần (vd gây push conflict) → job tự động retry, branch
   `feature/{slug}` bị force-push reset sạch (kiểm tra lịch sử commit trước/sau retry).
6. Cố ý làm job implement lỗi liên tiếp đủ 2 lần → sau lần retry thứ 2 vẫn lỗi, job dừng hẳn (không
   retry lần 3) và comment báo lỗi lên issue.
7. Cố ý tạo tình huống reviewer cần quyết định **lúc implement** → bot commit phần đã làm xong,
   comment câu hỏi lên issue, job dừng; trả lời comment → job tiếp tục đúng từ branch đó, không
   tính vào số lần retry.
7b. Cố ý tạo tình huống reviewer cần quyết định **lúc validate ở vòng 2 hoặc 3** → bot comment câu
    hỏi lên issue, job dừng; trả lời comment → job **chạy lại toàn bộ đúng vòng đang dở** (vòng 2
    hoặc 3, gọi lại cả 3 reviewer của vòng đó), không chạy lại từ vòng 1 — xác nhận bằng cách kiểm
    tra log: không có lệnh gọi lại của vòng 1 (hoặc vòng trước vòng đang dở).
7c. Cố ý tạo tình huống (a) dừng hỏi quyết định lúc implement, trả lời, bot commit tiếp — rồi (b)
    ngay sau đó ép job implement gặp lỗi kỹ thuật thật → job retry, branch reset về đúng commit của
    bước (a) (không mất quyết định vừa trả lời), không hỏi lại câu đã trả lời.
8. Merge PR code → job finalize tự chạy: Project Knowledge được cập nhật, `work/{feature}/` chuyển
   sang `work/completed/{feature}/`, không có commit nào reset `main`.
9. Xác nhận việc merge PR spec ở bước 3 không kích hoạt finalize — chỉ bước 8 mới kích hoạt.
10. Cố ý làm job finalize lỗi trước bước commit cuối → job chạy lại từ đầu (không reset gì), tối đa
    2 lần, sau đó comment báo lỗi nếu vẫn lỗi.
11. Cố ý làm job finalize "lỗi" ngay sau khi đã commit/push xong (vd ngắt kết nối giả lập sau bước
    push) → lần chạy lại tự phát hiện `work/{feature}/` đã archive xong, dừng ngay không làm lại.
12. Mở 2 issue (2 feature) gần nhau trên repo demo → cả 2 chạy độc lập không chặn nhau; nếu 2 job
    finalize trùng thời điểm, chúng chạy tuần tự, không ghi đè Project Knowledge.
13. Gửi 2 comment liên tiếp nhanh trên cùng 1 issue trong lúc job đầu còn đang chạy → job thứ 2 chờ
    job đầu chạy xong (không bị hủy giữa lúc đang commit `interview.yml`).
14. Theo dõi log Actions trong suốt quá trình trên → không có lần chạy nào bị tự kích hoạt lại bởi
    comment do chính bot tạo ra; mọi job `claude -p` chạy xong không bị treo chờ xác nhận tool nào.
15. Dùng một feature demo mà `documentation-writing` sẽ đánh giá là "chưa hoàn chỉnh" (vd cố ý chỉ
    implement một phần user-spec trước khi merge) rồi merge PR code → finalize vẫn chạy thẳng, không
    hỏi lại xác nhận, vì nhận diện đúng tín hiệu `ONLINE_PIPELINE_AUTOMATED` — xác nhận nhánh tự động
    mới thực sự chặn được bước "hỏi tiếp tục" chứ không chỉ lý thuyết.
16. Cố ý làm lỗi kỹ thuật ở một lượt interview hoặc một vòng validate (vd ngắt job giữa chừng) → job
    đó fail thẳng, không tự retry; comment lại trên issue → job chạy lại bình thường từ comment mới.
17. Chạy toàn bộ vòng đời một lần với secret là `CLAUDE_CODE_OAUTH_TOKEN` (OAuth) và một lần khác
    (hoặc 1 repo demo khác) với secret là `ANTHROPIC_API_KEY` → cả hai đường xác thực đều chạy được
    hết vòng đời, không chỉ đường thuận tiện hơn (OAuth) được thử.
