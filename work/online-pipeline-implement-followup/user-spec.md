---
# Creation date (YYYY-MM-DD)
created: 2026-10-06

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: online-pipeline-implement-followup

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building
Thêm 1 action mới ("followup") vào stage `implement` của `online-pipeline`: trong lúc code PR
(`feature/{slug}`) đang mở (chưa merge), một issue comment mới (kể cả được Slack relay vào) sẽ được
coi là 1 yêu cầu thay đổi code mới — chạy lại `code-writing` trên đúng nhánh đó, push commit mới vào
PR đang mở (không tạo PR mới), và báo lại qua comment khi xong. Nếu có comment follow-up khác tới khi
1 follow-up đang chạy, job cũ bị huỷ để ưu tiên yêu cầu mới nhất. Lặp lại không giới hạn số lần, cho
đến khi PR được merge.

## Why
Hiện tại, sau khi code PR được tạo xong (`status: ready_for_pr`), mọi issue comment sau đó — kể cả
tin nhắn Slack được relay vào — bị route job của `online-pipeline-implement.yml` bỏ qua
(`action=skip`), vì route job chỉ xử lý comment khi status đang là `awaiting_decision`. Người dùng
không thể yêu cầu sửa/tối ưu tiếp giống như đang chat trực tiếp với Claude Code — phải chờ PR merge
xong, hoặc tự sửa tay trên GitHub. Trong khi đó, stage `userspec` (dành cho spec PR) đã hỗ trợ việc
này từ trước: 1 comment trên issue sau khi spec PR đã mở vẫn tiếp tục kích hoạt `userspec-turn`, vì
route job của `online-pipeline-userspec.yml` không chặn theo status (chỉ chặn khi spec đã
`approved`). Feature này lấp đúng chỗ thiếu đối xứng đó ở phía code PR, để trải nghiệm liền mạch như
1 cuộc hội thoại liên tục để hoàn thiện MR, đúng ý người dùng muốn.

## Expected Behavior
1. Code PR của `feature/{slug}` đang mở, status marker đang là `ready_for_pr`.
2. Người dùng thấy điều gì đó chưa hợp lý trong code, comment thêm trên GitHub issue (hoặc reply
   trong Slack thread, được relay thành issue comment).
3. Route job của `online-pipeline-implement.yml` nhận diện: status vẫn `ready_for_pr` và PR vẫn
   `OPEN` → dispatch action mới `followup` (thay vì `skip` như hiện tại).
4. Stage followup coi comment này là 1 yêu cầu thay đổi code mới (không phải câu trả lời cho câu hỏi
   cũ), chạy lại `code-writing` (kể cả review waves) trên nhánh `feature/{slug}` hiện có.
5. Khi xong: commit mới được push vào đúng `feature/{slug}` (PR đang mở tự cập nhật, không tạo PR
   thứ hai); 1 comment xác nhận được post lên issue (mirror sang Slack nếu đã bật Slack bridge).
6. Người dùng xem lại, có thể comment tiếp để yêu cầu thêm — lặp lại từ bước 2 — không giới hạn số
   lần, cho đến khi hài lòng và merge PR.
7. Nếu 1 comment follow-up khác tới khi follow-up trước đang chạy: job đang chạy bị huỷ, job mới
   chạy theo đúng yêu cầu mới nhất (không cần chờ job cũ xong).
8. Nếu trong lúc follow-up, `code-writing` phát hiện cần người dùng quyết định (ví dụ 1 lựa chọn kỹ
   thuật không rõ): dừng lại, hỏi qua comment (giống implement ban đầu), chờ câu trả lời rồi tiếp
   tục qua đúng đường `resume` đã có — không bị hiểu nhầm là 1 yêu cầu follow-up khác.
9. Nếu comment mới không phải là 1 yêu cầu thay đổi code thật (ví dụ "ok cảm ơn"): agent trả lời
   ngắn mà không sửa code, giữ nguyên trạng thái `ready_for_pr`.
10. Sau khi PR merge (finalize xong): comment muộn trên issue không kích hoạt lại gì — giữ hành vi
    `skip` như hiện tại, không tự mở lại pipeline.

## Acceptance Criteria
- [ ] AC1: Khi code PR của `feature/{slug}` đang `OPEN` và status marker đang `ready_for_pr`, 1
  issue comment mới (không phải Bot) khiến route job trả về `action=followup` thay vì `skip` như
  hiện tại.
- [ ] AC2: Stage followup coi comment mới là 1 yêu cầu thay đổi code mới (không phải câu trả lời cho
  câu hỏi cũ) và chạy lại quy trình `code-writing` bình thường (kể cả review waves) trên đúng nhánh
  `feature/{slug}` hiện có.
- [ ] AC3: Khi thành công, commit mới được push vào `feature/{slug}`; PR đang mở tự cập nhật, không
  tạo PR thứ hai; 1 comment xác nhận được post lên issue (mirror sang Slack ở repo đã bật Slack
  bridge) — post bằng bước bash xác định (không phải agent tự post trong session của nó), chỉ sau
  khi bước bash đó TỰ KIỂM TRA exit code của lệnh push và xác nhận push thực sự thành công (không
  dùng `|| true` để bỏ qua lỗi). Nếu push thất bại thật, không post comment xác nhận — coi là lỗi kỹ
  thuật, đi qua ATTEMPTS-retry loop hiện có (xem Risk 4).
- [ ] AC4: Nếu trong lúc followup, review wave của `code-writing` trả về finding cần quyết định của
  người dùng (`user_decision_required`), xử lý giống implement gốc: ghi `status: awaiting_decision`,
  commit+push, hỏi qua comment, exit 0 — comment tiếp theo được xử lý qua đường `resume` hiện có
  (trả lời câu hỏi), không bị hiểu nhầm là 1 followup request khác.
- [ ] AC5: Nếu 1 comment followup khác tới trong lúc 1 followup đang chạy, job đang chạy bị huỷ
  (concurrency `cancel-in-progress`) và job mới chạy theo yêu cầu mới nhất. Vì push chỉ xảy ra ở các
  checkpoint đã định nghĩa (không liên tục), job bị huỷ trước khi tới checkpoint không để lại gì
  trên `origin`; trường hợp hiếm bị huỷ đúng lúc đang thực thi lệnh push xem Risk 2.
- [ ] AC6: Nếu agent nhận định comment mới không phải là 1 yêu cầu thay đổi code thực sự (ví dụ "ok
  cảm ơn"), nó trả lời ngắn qua comment mà không sửa code, và giữ status nguyên `ready_for_pr`.
- [ ] AC7: Sau khi code PR merge (finalize xong), comment muộn trên issue không kích hoạt lại gì —
  giữ hành vi `skip` như hiện tại (route job kiểm tra PR không còn `OPEN`).
- [ ] AC8: Trước khi chạy `code-writing`, job followup re-check (sau khi đã giành được concurrency
  gate — tức đã tới lượt chạy, giống cách resume đã làm) cả 2 điều kiện: status marker vẫn đúng
  `ready_for_pr`, và PR vẫn `OPEN`. Nếu 1 trong 2 sai (ví dụ đã bị 1 followup mới hơn/1 quyết định
  `awaiting_decision` đè lên trong lúc chờ tới lượt, hoặc PR đã merge), job tự dừng (không chạy
  code-writing, không ghi gì).

## Constraints
- Chỉ hoạt động trong khoảng `status == ready_for_pr` và code PR còn `OPEN` (chưa merge); không
  reopen sau khi đã merge/finalize.
- Giữ nguyên kênh giao tiếp hiện có: issue comment (được Slack relay mirror vào) — không thêm cơ chế
  lắng nghe comment trực tiếp trên PR.
- Tái dùng `code-writing` skill nguyên trạng; không viết lại logic implement hiện có.
- Không giới hạn số lượt follow-up (quyết định chủ động của người dùng).
- Khi 1 follow-up đang chạy mà có follow-up mới tới, job cũ bị huỷ để ưu tiên yêu cầu mới nhất —
  chấp nhận việc yêu cầu cũ bị bỏ qua khi bị yêu cầu mới "đè" lên.
- Thay đổi phải đồng bộ sang các bản vendored (scaffold `project-initialization`, mirror `.codex/`)
  theo đúng "Setup Script Maintenance" đã có trong `SKILL.md`.

## Risks
- **Risk 1:** Follow-up không giới hạn số lần có thể chạy vô hạn, tốn CI minutes/Claude usage.
  **Mitigation:** Đây là quyết định chủ động của người dùng; các cơ chế có sẵn (ATTEMPTS retry cap,
  rate-limit auto-switch) vẫn giới hạn chi phí của 1 lần chạy đơn lẻ.
- **Risk 2:** `cancel-in-progress` huỷ đúng lúc job đang thực thi lệnh git push (không chỉ đang tính
  toán) — cancel tới sau khi lệnh push đã bắt đầu nhưng trước khi job kịp ghi nhận kết quả.
  **Mitigation:** Push chỉ xảy ra ở các checkpoint cố định (không phải liên tục), nên rủi ro này
  hiếm; chấp nhận ở mức độ tương tự các rủi ro "accepted for v1" khác đã có trong `SKILL.md`. Vì
  followup dùng chung concurrency group với start/resume (xem Accepted Decisions), tại 1 thời điểm
  chỉ có đúng 1 job chạy cho 1 slug — rủi ro chỉ còn ở phạm vi 1 job tự bị huỷ giữa lúc push của
  chính nó, không phải 2 job khác nhau cùng push đè nhau.
- **Risk 3:** Agent nhận định sai 1 yêu cầu thay đổi thật là "chỉ là lời cảm ơn" nên im lặng bỏ qua.
  **Mitigation:** Agent vẫn luôn post lại 1 comment trả lời (AC3/AC6), nên người dùng luôn thấy phản
  hồi và có thể yêu cầu lại nếu bị hiểu nhầm.
- **Risk 4:** Vì `cancel-in-progress=true` huỷ TOÀN BỘ job đang chạy (không chỉ phần chưa push), 1
  job followup có thể bị 1 comment follow-up mới hơn huỷ ngay SAU KHI code đã push thành công nhưng
  TRƯỚC KHI kịp post comment xác nhận (AC3) — vì push và post-comment là 2 bước riêng trong cùng 1
  session agent. **Mitigation:** Việc post comment xác nhận (AC3) được chuyển ra bước bash xác định
  (deterministic), ngay sau khi xác nhận push đã thành công — để thu hẹp tối đa khoảng hở có thể bị
  huỷ giữa push và comment (còn lại gần như tức thời, không phải cả 1 lượt gọi `claude -p`). Bước
  bash này PHẢI tự kiểm tra exit code của lệnh push (không dùng `|| true` như "Open the code PR"
  hiện tại — pattern đó chỉ phù hợp cho 1 push không-có-gì-mới/vô hại, không phù hợp để quyết định
  có post comment xác nhận hay không): push thành công thật mới post comment; push thất bại thật
  (không phải do bị cancel — nếu bị cancel thì cả job, kể cả bước bash này, đã dừng hẳn, không chạy
  tiếp được nữa) được coi là lỗi kỹ thuật, đi qua ATTEMPTS-retry loop hiện có, không post comment
  xác nhận sai. Phần rủi ro còn sót (huỷ đúng lúc chính bước bash này đang chạy, cực hiếm) được chấp
  nhận vì không mất code: commit đã push thành công trước khi bị huỷ, không bị mất; job follow-up
  mới hơn (chính là lý do gây huỷ) sẽ tự hoàn thành và post comment xác nhận của riêng nó ngay sau
  đó, nên người dùng vẫn luôn nhận được phản hồi cuối cùng — chỉ có thể thiếu xác nhận riêng cho
  đúng yêu cầu đã bị "đè" lên, không phải mất phản hồi hoàn toàn.

## Accepted Decisions
- Thêm route action mới `followup` (khác `resume`) khi `status==ready_for_pr` + PR đang `OPEN`,
  thay vì tái dùng `resume` — vì `resume` đóng khung comment là câu trả lời cho câu hỏi cũ, sai bản
  chất với 1 yêu cầu mới.
- `followup` dùng CHUNG concurrency group `online-pipeline-implement-{slug}` với start/resume
  (không tách group riêng), chỉ khác giá trị `cancel-in-progress` theo action đang dispatch:
  `cancel-in-progress: ${{ action == 'followup' }}` — tức 1 job followup mới sẽ huỷ bất kỳ job nào
  (start/resume/followup khác) đang chạy/đang chờ trong group đó, còn 1 job start/resume mới vẫn
  xếp hàng chờ như hành vi hiện tại (`cancel-in-progress=false`), không đổi gì cho đường đó. Lý do
  chọn 1 group chung thay vì 2 group riêng: GitHub Actions chỉ cho đúng 1 job chạy thật trong 1
  group tại 1 thời điểm (job khác bị huỷ hoặc phải chờ), nên dùng chung group loại bỏ hoàn toàn khả
  năng 2 job (ví dụ 1 resume đang xử lý quyết định và 1 followup khác) cùng chạy và cùng push vào
  `feature/{slug}` — không cần dựng thêm cơ chế verify-push hay chấp nhận rủi ro race giữa 2 job,
  đơn giản hơn so với phương án tách group riêng ban đầu. Cơ chế cancel/queue theo group này đã được
  project tự tin cậy sẵn (resume's "xếp hàng chờ" hiện tại chính là queue-trong-1-group).
- Việc nhận định "comment có phải là yêu cầu thay đổi code thật hay chỉ là lời đáp xã giao" giao cho
  agent tự quyết định trong stage prompt, không viết filter từ khoá cứng trong bash — nhất quán với
  cách routing trong skill này luôn "dumb"/deterministic, mọi nhận định nội dung nằm trong `claude
  -p`.
- Không tạo giá trị status marker mới: followup tái dùng đúng 3 giá trị đã có cho stage implement
  (`in_progress` khi đang chạy — chỉ là bookkeeping local của job đó, không push ngay; `ready_for_pr`
  khi xong thành công; `awaiting_decision` khi cần dừng lại hỏi người dùng, xem AC4). Vì giá trị
  trên `origin` chỉ thực sự đổi khi agent tự commit+push lúc kết thúc (1 trong 2 giá trị terminal
  `ready_for_pr`/`awaiting_decision`), origin vẫn hiển thị giá trị trước đó (`ready_for_pr`) suốt
  lúc 1 followup đang chạy, cho đến khi nó tự commit 1 trong 2 giá trị terminal đó — đây chính là cơ
  chế khiến route job của 1 comment followup thứ 2 (tới trong lúc job đầu chưa commit gì) vẫn
  dispatch đúng `action=followup`, cho phép `cancel-in-progress` huỷ job cũ như AC5 yêu cầu (không
  cần giá trị status mới).
- Post-gate re-check (AC8) kiểm tra cả 2 điều kiện — status vẫn `ready_for_pr` và PR vẫn `OPEN` —
  giống nguyên văn cách resume đã tự re-check `CURRENT_STATUS != "awaiting_decision"` sau khi tới
  lượt chạy (vì snapshot của route job có thể đã cũ nếu job này phải xếp hàng chờ). Cần cả 2 điều
  kiện vì mỗi điều kiện bắt 1 trường hợp khác nhau: check status bắt việc 1 job khác (chạy trước
  trong cùng group) vừa commit `awaiting_decision`/`ready_for_pr` mới trước khi tới lượt job này;
  check PR-open bắt case đã merge trong lúc chờ (vì finalize chạy trên `main`, không đụng tới status
  marker trên `feature/{slug}`, nên check status một mình không phát hiện được việc đã merge).
- Xác nhận follow-up (AC3) là bắt buộc, không im lặng — khác với lần implement đầu tiên (hiện tại
  không post comment khi mở PR lần đầu), vì PR đã tồn tại nên "đã tạo PR" không còn là tín hiệu đủ
  rõ; cần 1 tín hiệu riêng cho mỗi lần follow-up để người dùng biết chắc là đã xử lý xong.
- Không hỗ trợ reopen pipeline sau khi đã merge/finalize — ngoài phạm vi feature này; 1 thay đổi sau
  khi đã merge là 1 feature/fix mới, không phải follow-up của PR cũ.

## Testing

**Unit tests:** không áp dụng được — toàn bộ thay đổi là bash trong workflow YAML (route job,
concurrency, prompt framing) và hướng dẫn agent trong `SKILL.md`, không có logic ứng dụng mới để
unit-test.

**Integration tests:** không áp dụng được theo nghĩa test tự động — không có harness CI nào mô
phỏng GitHub Actions + Slack Worker trong repo này.

**E2E tests:** cần, nhưng thủ công — xem Verification bên dưới (dry-run thực tế trên 1 repo đã bật
online-pipeline + Slack bridge), vì đây là cách duy nhất verify được route job/concurrency/dispatch
thật trên GitHub Actions.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Review diff của `online-pipeline-implement.yml` (route job + concurrency block dùng chung group, chỉ `cancel-in-progress` khác theo action) và `SKILL.md` (mục Stage: implement followup mới) so với pattern đã có của `implement-resume` | Action `followup` được thêm nhất quán: dispatch condition, prompt framing ("yêu cầu mới" không phải "câu trả lời"), post-gate re-check cả status+PR-open (giống stale-guard của resume) — không có đoạn nào copy nguyên `resume`'s "answer to the question" framing; concurrency group giống `online-pipeline-implement-${slug}` hiện có, không tạo group mới. |
| 2. Review đồng bộ vendored copies (`skills/project-initialization/assets/new-project/.github/workflows/online-pipeline-implement.yml`, mirror `.codex/`) | Nội dung khớp với bản nguồn trong `skills/online-pipeline/`, theo đúng "Setup Script Maintenance". |
| 3. Kiểm tra YAML hợp lệ (`actionlint`/`yamllint` nếu có sẵn trong repo, hoặc ít nhất parse YAML) | Không có lỗi cú pháp trong các file workflow đã sửa. |
| 4. Review hướng dẫn agent trong `SKILL.md` cho đường `awaiting_decision` giữa lúc followup (AC4) | Hướng dẫn rõ: followup dừng lại, ghi `awaiting_decision`, commit+push, hỏi qua comment, exit — comment tiếp theo đi qua đúng đường `resume` sẵn có, không có đoạn nào coi nó là 1 followup request khác. |
| 5. Review hướng dẫn agent cho đường "comment không phải yêu cầu đổi code" (AC6) | Hướng dẫn rõ: agent tự nhận định, nếu không phải yêu cầu thật thì chỉ trả lời ngắn, không chạy code-writing, không đổi status. |

### User Verification
- Dry-run thực tế trên 1 repo đã bật online-pipeline + Slack bridge (do route job/concurrency/push
  thật của GitHub Actions không thể verify được bằng code review hay unit test): tạo issue → qua
  spec → implement → code PR mở → comment follow-up (qua GitHub hoặc Slack) → xác nhận (a) commit
  mới vào đúng PR, (b) có comment xác nhận, (c) gửi tiếp 1 comment follow-up khác ngay trong lúc cái
  trước đang chạy → xác nhận job cũ bị huỷ và job mới chạy theo yêu cầu mới nhất, (d) comment sau khi
  merge không kích hoạt gì. Cần verify thủ công vì đây là hành vi thời gian thực của GitHub Actions
  concurrency + Slack relay, không tái tạo được trong môi trường local.
- Thêm 2 case cần verify thủ công cùng đợt dry-run trên (không verify được bằng review tĩnh, vì phụ
  thuộc vào nhận định sống của agent trong lúc chạy `claude -p`):
  (e) comment follow-up yêu cầu 1 thay đổi mà code-writing cần hỏi lại (ví dụ cố tình đưa ra yêu cầu
  mơ hồ về kỹ thuật) → xác nhận pipeline dừng, hỏi qua comment, giữ `awaiting_decision`, và trả lời
  câu hỏi đó tiếp tục đúng qua đường resume (AC4);
  (f) comment chỉ mang tính xã giao (ví dụ "ok cảm ơn") → xác nhận agent trả lời ngắn, không có
  commit mới, status vẫn `ready_for_pr` (AC6).
