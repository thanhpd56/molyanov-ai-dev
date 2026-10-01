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
Skill tái dùng nguyên vẹn template/script của `user-spec-planning`, chỉ bổ sung các cơ chế vận hành
cần cho môi trường GitHub Actions không có trạng thái phiên liên tục giữa các lần chạy.

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
   hoạt job finalize: cập nhật Project Knowledge, chuyển `work/{feature}/` sang
   `work/completed/{feature}/`, không có bước xác nhận lại ("merge đã là xác nhận"). Finalize commit
   thẳng vào `main` (không có branch riêng); nếu job finalize lỗi trước khi tới bước commit cuối
   cùng, `main` chưa bị đụng tới nên không cần reset gì — job chỉ đơn giản **chạy lại từ đầu** (fresh
   checkout), tự động retry tối đa 2 lần, sau đó comment báo lỗi lên issue.
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
      sạch** (force-push) trước khi làm lại; vẫn lỗi sau 2 lần → comment báo lỗi lên issue và dừng.
- [ ] Trong lúc validate spec hoặc trong lúc implement, nếu reviewer cần người dùng quyết định → bot
      commit phần đã làm xong (nếu đang implement), comment câu hỏi lên issue, dừng — không tính vào
      2 lần retry; trả lời bằng comment → bot tiếp tục đúng từ branch đó, không làm lại từ đầu.
- [ ] PR code merge → job finalize tự động chạy: cập nhật Project Knowledge, chuyển `work/{feature}/`
      sang `work/completed/{feature}/`, không hỏi lại xác nhận.
- [ ] Merge PR spec (lúc approve) **không** tự kích hoạt finalize — chỉ PR code (đúng label/branch)
      mới kích hoạt.
- [ ] Job finalize thất bại trước khi commit → tự động chạy lại từ đầu (không reset gì, vì `main`
      chưa bị đụng tới), tối đa 2 lần, sau đó comment báo lỗi lên issue.
- [ ] Hai issue (hai feature) mở gần nhau trên cùng repo chạy độc lập không chặn nhau; hai job
      finalize trùng thời điểm chạy tuần tự, không ghi đè nhau lên Project Knowledge.
- [ ] Hoạt động cả trên project có sẵn (bật online-pipeline độc lập) và project mới tạo (tích hợp vào
      `project-initialization`).
- [ ] Comment do chính bot tạo ra (câu hỏi, thông báo, báo lỗi, hỏi quyết định) **không** tự kích
      hoạt lại một lần chạy job mới.

## Constraints
- Hỗ trợ cả hai cách xác thực: Claude subscription OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`) và
  Anthropic API key (`ANTHROPIC_API_KEY`) — người dùng chọn lúc setup, không cố định cứng.
- Tái dùng nguyên vẹn `assets/user-spec.md.template`, `assets/interview.yml.template`,
  `assets/decisions.md.template`, `scripts/init-feature-folder.sh` của `user-spec-planning`, không
  tạo định dạng riêng, để các reviewer agent (`skeptic`, `userspec-quality-validator`,
  `userspec-adequacy-validator`, `interview-completeness-checker`) hoạt động không đổi.
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
- Job implement auto-retry tối đa 2 lần khi gặp lỗi kỹ thuật, **reset branch riêng về sạch
  (force-push) trước mỗi lần retry** — vì implement ghi vào branch riêng `feature/{slug}`, reset
  branch đó không ảnh hưởng gì khác. Áp dụng đồng nhất cho mọi nguyên nhân lỗi (Claude error, push
  conflict, hết quota), không phân nhánh theo loại lỗi (đánh đổi: tốn thêm token khi lỗi do push
  conflict, đổi lại đơn giản hơn trong v1).
- Job finalize auto-retry tối đa 2 lần khi gặp lỗi kỹ thuật, nhưng **không** force-push/reset gì —
  vì finalize commit thẳng vào `main` (không có branch riêng); force-push `main` sẽ xoá mất commit
  hợp lệ của feature khác đang chạy song song. Một lỗi trước bước commit cuối cùng không để lại dấu
  vết trên `main`, nên retry chỉ cần chạy lại job từ một checkout mới.
- Interview-turn và draft/validate-round khi gặp lỗi kỹ thuật **không** auto-retry — job fail thẳng,
  người dùng tự re-trigger bằng cách comment lại (vì mỗi lượt đã sẵn comment-triggered).
- Một finding `user_decision_required` (không phải lỗi kỹ thuật) trong lúc validate hoặc trong lúc
  implement được xử lý khác với lỗi: commit phần đã làm xong, comment câu hỏi lên issue, dừng —
  không tính vào 2 lần retry; tiếp tục từ branch đã commit khi người dùng trả lời.
- Mỗi feature chạy trên concurrency group riêng theo số issue (song song không giới hạn số lượng);
  mọi job finalize chia sẻ một concurrency group cố định để chạy tuần tự khi trùng thời điểm.
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
  `documentation-writing/SKILL.md` ("nếu được gọi trong ngữ cảnh tự động, không có người trả lời →
  luôn tiếp tục finalize, không hỏi lại"), thay vì để online-pipeline tự viết logic finalize rút gọn
  riêng. Minh bạch và verify được, đổi lại là một thay đổi nhỏ vào skill dùng chung — chấp nhận vì
  nhánh mới không đổi hành vi hiện có khi gọi tương tác như trước.
- Chọn phân biệt PR spec và PR code bằng branch (`userspec/{slug}` vs `feature/{slug}`) và label
  (`userspec-spec` vs `userspec-implement`), vì cả hai merge vào main qua cùng loại sự kiện GitHub
  và finalize cần biết chỉ nên trigger theo PR code.
- Chọn lọc comment theo tác giả (loại trừ comment của chính bot) trên mọi job nghe `issue_comment`,
  để tránh bot tự kích hoạt lại chính mình.
- Chọn chấp nhận giới hạn rate/concurrency của OAuth subscription khi chạy nhiều feature song song
  là một hạn chế đã biết của v1, không xây cơ chế queue tự động.
- Chọn tiêu chí "xong" là chạy thử toàn bộ vòng đời trên một repo demo nhỏ thật, không viết unit test
  cho các bash script/workflow YAML.

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
| 3. Kiểm tra sửa đổi trong `documentation-writing/SKILL.md` | Nhánh tự động mới không đổi hành vi nhánh tương tác hiện có |
| 4. Tạo project mới qua `project-initialization` với online-pipeline được bật | Scaffold project mới có đủ file workflow + hướng dẫn setup (kiểm tra tĩnh, không chạy vòng đời thật) |

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
6. Cố ý tạo tình huống reviewer cần quyết định (lúc validate và lúc implement) → bot commit phần đã
   làm xong (nếu đang implement), comment câu hỏi lên issue, job dừng; trả lời comment → job tiếp
   tục đúng từ branch đó, không tính vào số lần retry.
7. Merge PR code → job finalize tự chạy: Project Knowledge được cập nhật, `work/{feature}/` chuyển
   sang `work/completed/{feature}/`, không có commit nào reset `main`.
8. Xác nhận việc merge PR spec ở bước 3 không kích hoạt finalize — chỉ bước 7 mới kích hoạt.
9. Cố ý làm job finalize lỗi trước bước commit cuối → job chạy lại từ đầu (không reset gì), tối đa 2
   lần, sau đó comment báo lỗi nếu vẫn lỗi.
10. Mở 2 issue (2 feature) gần nhau trên repo demo → cả 2 chạy độc lập không chặn nhau; nếu 2 job
    finalize trùng thời điểm, chúng chạy tuần tự, không ghi đè Project Knowledge.
11. Theo dõi log Actions trong suốt quá trình trên → không có lần chạy nào bị tự kích hoạt lại bởi
    comment do chính bot tạo ra.
