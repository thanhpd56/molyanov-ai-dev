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
  issue comment mới (không phải Bot, không phải `/switch-online`) khiến route job trả về
  `action=followup` thay vì `skip` như hiện tại.
- [ ] AC2: Stage followup coi comment mới là 1 yêu cầu thay đổi code mới (không phải câu trả lời cho
  câu hỏi cũ) và chạy lại quy trình `code-writing` bình thường (kể cả review waves) trên đúng nhánh
  `feature/{slug}` hiện có.
- [ ] AC3: Khi thành công, commit mới được push vào `feature/{slug}` (verify push thực sự thành
  công, không chỉ dựa vào bước push an toàn `|| true` hiện có — xem Risk 3); PR đang mở tự cập nhật,
  không tạo PR thứ hai; 1 comment xác nhận được post lên issue (mirror sang Slack ở repo đã bật Slack
  bridge). Không post comment xác nhận nếu chưa chắc code đã lên PR thật.
- [ ] AC4: Nếu trong lúc followup, review wave của `code-writing` trả về finding cần quyết định của
  người dùng (`user_decision_required`), xử lý giống implement gốc: ghi `status: awaiting_decision`,
  commit+push, hỏi qua comment, exit 0 — comment tiếp theo được xử lý qua đường `resume` hiện có
  (trả lời câu hỏi), không bị hiểu nhầm là 1 followup request khác.
- [ ] AC5: Nếu 1 comment followup khác tới trong lúc 1 followup đang chạy, job đang chạy bị huỷ
  (concurrency `cancel-in-progress`) và job mới chạy theo yêu cầu mới nhất; không có gì bị push từ
  job bị huỷ (vì chỉ push tại các checkpoint đã định nghĩa, xem Accepted Decisions).
- [ ] AC6: Nếu agent nhận định comment mới không phải là 1 yêu cầu thay đổi code thực sự (ví dụ "ok
  cảm ơn"), nó trả lời ngắn qua comment mà không sửa code, và giữ status nguyên `ready_for_pr`.
- [ ] AC7: Sau khi code PR merge (finalize xong), comment muộn trên issue không kích hoạt lại gì —
  giữ hành vi `skip` như hiện tại (route job kiểm tra PR không còn `OPEN`).
- [ ] AC8: Trước khi chạy `code-writing`, job followup re-check (sau khi đã giành được concurrency
  gate của riêng nó) cả 2 điều kiện: status marker vẫn đúng `ready_for_pr`, và PR vẫn `OPEN`. Nếu 1
  trong 2 sai, job tự dừng (không chạy code-writing, không ghi gì) — tránh chạy chồng lên 1 quyết
  định (`awaiting_decision`) hoặc 1 PR đã merge vừa xảy ra ở nơi khác.

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
- **Risk 2:** `cancel-in-progress` huỷ đúng lúc job đang git push (không chỉ đang tính toán).
  **Mitigation:** Push chỉ xảy ra ở các checkpoint cố định (không phải liên tục), nên rủi ro này
  hiếm; chấp nhận ở mức độ tương tự các rủi ro "accepted for v1" khác đã có trong `SKILL.md`.
- **Risk 3:** Vì followup dùng concurrency group riêng (không chung với group của start/resume, để
  giữ nguyên hành vi xếp-hàng-chờ hiện có cho `awaiting_decision`/`resume`), đây là lần đầu 2 job có
  thể cùng push vào `feature/{slug}`. Nếu push của agent bị reject (non-fast-forward) và bị nuốt lỗi
  bởi bước push an toàn `|| true` hiện có, pipeline có thể báo "thành công" dù code chưa thực sự lên
  PR. **Mitigation:** Push của followup phải được verify thành công (không dựa vào `|| true`); 1
  push bị reject phải được coi là lỗi kỹ thuật, xử lý qua ATTEMPTS-retry loop hiện có — không post
  comment xác nhận khi chưa chắc code đã lên PR.
- **Risk 4:** Race rất hẹp còn sót lại sau các post-gate re-check ở AC8: nếu 1 job resume/start
  (group khác, không bị huỷ) và 1 job followup đều vượt qua re-check của chính nó trong cùng vài
  giây (đọc đúng điều kiện của mình trước khi bên kia commit), cả hai có thể cùng push gần nhau.
  **Mitigation:** Chấp nhận là rủi ro hiếm cho v1, cùng mức độ với các race khác `SKILL.md` đã tự
  nhận là "accepted rare risk" (ví dụ mục "Comment failure and revert", "Push conflicts"); không xây
  cơ chế lock phân tán cho race này.
- **Risk 5:** Agent nhận định sai 1 yêu cầu thay đổi thật là "chỉ là lời cảm ơn" nên im lặng bỏ qua.
  **Mitigation:** Agent vẫn luôn post lại 1 comment trả lời (AC3/AC6), nên người dùng luôn thấy phản
  hồi và có thể yêu cầu lại nếu bị hiểu nhầm.

## Accepted Decisions
- Thêm route action mới `followup` (khác `resume`) khi `status==ready_for_pr` + PR đang `OPEN`,
  thay vì tái dùng `resume` — vì `resume` đóng khung comment là câu trả lời cho câu hỏi cũ, sai bản
  chất với 1 yêu cầu mới.
- `followup` dùng concurrency group riêng (không chung với group của start/resume) với
  `cancel-in-progress=true`, để không ảnh hưởng hành vi xếp-hàng-chờ (`cancel-in-progress=false`)
  đang có cho `awaiting_decision`/`resume`.
- Việc nhận định "comment có phải là yêu cầu thay đổi code thật hay chỉ là lời đáp xã giao" giao cho
  agent tự quyết định trong stage prompt, không viết filter từ khoá cứng trong bash — nhất quán với
  cách routing trong skill này luôn "dumb"/deterministic, mọi nhận định nội dung nằm trong `claude
  -p`.
- Không tạo giá trị status marker mới: followup tái dùng đúng 2 giá trị đã có (`in_progress` khi
  đang chạy — chỉ là bookkeeping local của job đó, không push ngay; `ready_for_pr` khi xong). Vì giá
  trị trên `origin` chỉ thực sự đổi khi agent tự commit+push lúc kết thúc, origin vẫn hiển thị
  `ready_for_pr` suốt lúc 1 followup đang chạy — đây chính là cơ chế khiến route job của 1 comment
  followup thứ 2 vẫn dispatch đúng `action=followup`, cho phép `cancel-in-progress` huỷ job cũ như
  AC5 yêu cầu (không cần giá trị status mới).
- Post-gate re-check (AC8) kiểm tra cả 2 điều kiện — status vẫn `ready_for_pr` và PR vẫn `OPEN` —
  vì mỗi điều kiện bắt 1 loại race khác nhau: check status bắt race với 1 job khác vừa commit
  `awaiting_decision`/`ready_for_pr` mới; check PR-open bắt case đã merge (vì finalize chạy trên
  `main`, không đụng tới status marker trên `feature/{slug}`).
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
| 1. Review diff của `online-pipeline-implement.yml` (route job + concurrency block mới) và `SKILL.md` (mục Stage: implement followup mới) so với pattern đã có của `implement-resume` | Action `followup` được thêm nhất quán: dispatch condition, prompt framing ("yêu cầu mới" không phải "câu trả lời"), post-gate re-check cả status+PR-open, push-verify trước khi báo success — không có đoạn nào copy nguyên `resume`'s "answer to the question" framing. |
| 2. Review đồng bộ vendored copies (`skills/project-initialization/assets/new-project/.github/workflows/online-pipeline-implement.yml`, mirror `.codex/`) | Nội dung khớp với bản nguồn trong `skills/online-pipeline/`, theo đúng "Setup Script Maintenance". |
| 3. Kiểm tra YAML hợp lệ (`actionlint`/`yamllint` nếu có sẵn trong repo, hoặc ít nhất parse YAML) | Không có lỗi cú pháp trong các file workflow đã sửa. |

### User Verification
- Dry-run thực tế trên 1 repo đã bật online-pipeline + Slack bridge (do route job/concurrency/push
  thật của GitHub Actions không thể verify được bằng code review hay unit test): tạo issue → qua
  spec → implement → code PR mở → comment follow-up (qua GitHub hoặc Slack) → xác nhận (a) commit
  mới vào đúng PR, (b) có comment xác nhận, (c) gửi tiếp 1 comment follow-up khác ngay trong lúc cái
  trước đang chạy → xác nhận job cũ bị huỷ và job mới chạy theo yêu cầu mới nhất, (d) comment sau khi
  merge không kích hoạt gì. Cần verify thủ công vì đây là hành vi thời gian thực của GitHub Actions
  concurrency + Slack relay, không tái tạo được trong môi trường local.
