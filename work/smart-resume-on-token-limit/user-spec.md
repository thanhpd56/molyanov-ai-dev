---
# Creation date (YYYY-MM-DD)
created: 2026-10-06

# Status: draft | approved
status: approved

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: smart-resume-on-token-limit

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building

Khi 1 trong 3 stage của `online-pipeline` (`implement`, `knowledge-init`, `userspec-turn`) đang
chạy `claude -p` mà bị chặn giữa lúc làm việc bởi rate-limit Claude thật (session 5 giờ hoặc weekly
7 ngày — 2 chuỗi lỗi đã được nhận diện sẵn từ feature `control-plane-token-switch`), hệ thống đổi
token (cơ chế đã có) rồi dùng `claude -p -c` (`--continue`, tiếp tục đúng hội thoại cũ trong cùng
working directory) để tiếp tục công việc đang làm dở, thay vì `git reset --hard` về trạng thái
trước đó và gửi lại prompt gốc từ đầu như hành vi hiện tại. Lượt rate-limit-resume này không tính
vào giới hạn `ATTEMPTS=3` kỹ thuật hiện có — tách thành 1 counter riêng, không giới hạn cứng, chỉ
dừng khi pool token thật sự hết (`SWITCH_EXHAUSTED`, hành vi đã có, không đổi).

## Why

Trước feature này, 1 lần rate-limit Claude xảy ra giữa lúc agent đang code dở (ví dụ đang sửa 1
file) làm mất toàn bộ phần việc của attempt đó: hệ thống `git reset --hard` về `$LAST_GOOD`, attempt
kế phải làm lại từ đầu với đúng prompt gốc, không còn nhớ gì đã làm trước đó. Đây là explicit
limitation đã được ghi nhận rõ trong feature `control-plane-token-switch` ("không xây cơ chế resume
giữa chừng mới"). Rate-limit không phải lỗi logic — chỉ là "hết hạn mức tạm thời" — nên việc làm lại
toàn bộ từ đầu vừa tốn thời gian, vừa có thể khiến agent ra kết quả khác với lần trước (không còn
đúng những quyết định/tiến độ đã thực hiện). Feature này giúp pipeline phục hồi mượt hơn: chỉ cần
đổi "chứng chỉ" (token) rồi tiếp tục đúng mạch việc, không phải khởi động lại.

## Expected Behavior

1. Cả 3 workflow (`online-pipeline-implement.yml`, `online-pipeline-knowledge-init.yml`,
   `online-pipeline-userspec.yml`) giữ nguyên `try_rate_limit_switch()` hiện có (match 2 chuỗi rate-
   limit → gọi `POST $SLACK_WORKER_URL/switch-token/auto` → cập nhật `CURRENT_TOKEN` hoặc set
   `SWITCH_EXHAUSTED=true`) — không đổi logic này.

2. Vòng lặp attempt của mỗi workflow tách thành 2 biến đếm độc lập:
   - `ATTEMPT` kỹ thuật: tối đa 3 (`ATTEMPTS=3`, không đổi), chỉ tăng khi lỗi **không phải**
     rate-limit (hoặc rate-limit nhưng `try_rate_limit_switch` thất bại — không khớp mẫu, hoặc gọi
     Worker lỗi khác `no_alias_available`).
   - Số lần "rate-limit resume": không có cap cứng, tăng mỗi lần `try_rate_limit_switch` trả về
     thành công; vòng lặp chỉ dừng khi `SWITCH_EXHAUSTED=true` (pool hết token khả dụng — hành vi
     đã có, không đổi).

3. Khi `try_rate_limit_switch` thành công (token đã đổi):
   - **Không** chạy bước reset-và-dọn hiện tại của stage đó — ở `implement`/`userspec-turn`:
     `git push --force` về `$LAST_GOOD` + `git reset --hard $LAST_GOOD` + `git clean -fd`; ở
     `knowledge-init` (không có commit cục bộ nào để push lại, chỉ có working-tree edits chưa
     commit): `git reset --hard origin/main` + `git clean -fd`, không có bước push nào. Bỏ qua
     đúng bước reset/clean tương ứng của từng stage — giữ nguyên toàn bộ file đã sửa + commit cục
     bộ (nếu có) của attempt đó.
   - Gọi lại `claude -p` với `CLAUDE_CODE_OAUTH_TOKEN="$CURRENT_TOKEN"` (giá trị mới) kèm flag
     `-c`/`--continue` để tiếp tục đúng hội thoại gần nhất trong cùng working directory, cùng 1
     prompt ngắn (khác hẳn `PROMPT_FILE` gốc) thông báo token đã được refresh do rate-limit và yêu
     cầu tiếp tục đúng công việc đang làm, không bắt đầu lại từ đầu.
   - Log rõ 1 dòng phân biệt, ví dụ `"Rate-limit resume #N — continuing previous session with new
     token"`, để không nhầm với dòng log `"Implement attempt X/3"` của attempt kỹ thuật thường.
   - `ATTEMPT` kỹ thuật **không** tăng cho lượt này.

4. Khi switch token thành công nhưng lệnh `claude -p -c ...` resume đó **tự thất bại kỹ thuật** (ví
   dụ "session not found", CLI lỗi khác rate-limit) — coi như 1 lỗi kỹ thuật thường: rơi về đúng
   hành vi cũ (`git reset --hard` về **baseline ban đầu** `$LAST_GOOD`/`origin/main` — chốt 1 lần
   duy nhất trước khi vào loop, không advance sau mỗi lần rate-limit-resume thành công + gửi lại
   prompt gốc ở attempt kế), tính vào `ATTEMPT` kỹ thuật (tối đa 3). **Limitation đã biết** (xem
   Risk 1): nếu điều này xảy ra SAU 1 hoặc nhiều lần rate-limit-resume thành công đã làm thêm việc
   thật, fallback này xoá luôn cả phần việc của các lần resume đó, không chỉ phần việc của lượt
   cuối — chấp nhận cho v1, không advance baseline.

5. Mọi trường hợp khác giữ nguyên hành vi đã có, không đổi:
   - Output không khớp 2 mẫu rate-limit, hoặc gọi Worker lỗi (network/5xx, khác
     `no_alias_available`) → rơi về nhánh technical-failure cũ (reset + tính vào 3 `ATTEMPTS`).
   - `SWITCH_EXHAUSTED=true` (pool hết token khả dụng) → dừng ngay, báo lỗi rõ qua `gh issue
     comment` + relay Slack, giống hệt hành vi hiện tại.

6. Thuật ngữ **"rate-limit resume"** (log, biến bash, `SKILL.md`) được dùng riêng, phân biệt rõ với
   khái niệm **"resume"** đã có từ trước trong codebase (`ROUTE_ACTION == 'resume'`/stage
   `implement-resume` — job GitHub Actions **mới hoàn toàn** chạy khi 1 người trả lời câu hỏi quyết
   định giữa kỳ, không liên quan gì đến session CLI). Hai cơ chế không giao nhau: rate-limit resume
   chỉ xảy ra **trong cùng 1 job/runner** đang chạy; decision-resume luôn là 1 job/runner mới.

7. Cơ chế này **chỉ** áp dụng cho rate-limit Claude thật (2 chuỗi đã match sẵn) — **không** mở rộng
   sang GitHub Actions job timeout (`timeout-minutes` hoặc giới hạn 6 giờ của runner hosted): khi
   job/runner bị huỷ hoàn toàn, không còn gì trên filesystem để `-c`/`--resume`, nên trường hợp đó
   tiếp tục không được xử lý bởi feature này.

8. `skills/online-pipeline/SKILL.md` mô tả rõ hành vi "rate-limit resume" mới cho cả 3 stage, đồng
   bộ cả 4 bản tracked (nguồn, mirror `.codex/` của nguồn, vendor trong scaffold
   `project-initialization`, mirror `.codex/` của vendor) — đúng quy ước đồng bộ đã áp dụng xuyên
   suốt `online-pipeline`. Mỗi workflow YAML có **6** bản tracked cần đồng bộ (nguồn, mirror
   `.codex/` của nguồn, vendor dạng skill dưới `.claude/skills/online-pipeline/assets/workflows/`,
   mirror `.codex/` của vendor đó, vendor dạng workflow **đã cài đặt thật** dưới
   `.github/workflows/` — file thật sự chạy GitHub Actions ở repo scaffold ra — và mirror `.codex/`
   của vendor đó).

9. Không đổi gì ở repo `control-plane` riêng — endpoint `POST /switch-token/auto` giữ nguyên, chỉ
   thay đổi cách phía `molyanov-ai-dev` phản ứng sau khi nhận token mới thành công.

## Acceptance Criteria

- [ ] Giả lập 1 output `claude -p` khớp đúng 1 trong 2 chuỗi rate-limit, `switch-token/auto` trả
  `{ok:true, alias, token}` → không có `git reset`/`git clean` nào chạy; `claude -p` được gọi lại
  với `-c` và token mới; biến `ATTEMPT` kỹ thuật không tăng cho lượt này.
- [ ] Giả lập output không khớp cả 2 mẫu rate-limit → hành vi y hệt trước feature này: `git reset`
  về trạng thái trước, `ATTEMPT` kỹ thuật tăng 1.
- [ ] Giả lập `switch-token/auto` trả lỗi khác `no_alias_available` (ví dụ timeout/5xx) → rơi về
  đúng nhánh technical-failure cũ (reset + tính vào `ATTEMPTS`), không coi là rate-limit resume.
- [ ] Giả lập `switch-token/auto` trả `{ok:false, error:"no_alias_available"}` → dừng ngay, báo lỗi
  rõ qua `gh issue comment` + Slack, y hệt hành vi hiện tại (không đổi).
- [ ] Giả lập switch thành công nhưng lệnh `claude -p -c` resume tự lỗi (ví dụ mock trả về session
  not found) → fallback về reset-và-gửi-lại-prompt-gốc, tính vào `ATTEMPT` kỹ thuật.
- [ ] Giả lập 1 lần rate-limit-resume thành công (có thêm edit/commit mới), rồi 1 lỗi kỹ thuật
  thường ngay sau đó → fallback reset về đúng baseline ban đầu (trước cả lần resume), xoá cả phần
  việc của lần resume đó — đúng limitation đã chấp nhận ở Risk 1, không phải bug.
- [ ] Giả lập 3 lần rate-limit-resume liên tiếp thành công trong 1 job (pool còn nhiều token khả
  dụng) → không bị `ATTEMPTS=3` chặn lại (vì không tính vào counter đó); chỉ dừng khi
  `SWITCH_EXHAUSTED=true`.
- [ ] Cả 3 workflow (`online-pipeline-implement.yml`, `online-pipeline-knowledge-init.yml`,
  `online-pipeline-userspec.yml`) có logic rate-limit-resume giống nhau; `diff` sạch giữa **6**
  bản tracked của mỗi file (nguồn, vendor skill, vendor `.github/workflows/` đã cài đặt thật, và 3
  mirror `.codex/` tương ứng — xem code-research.md).
- [ ] `skills/online-pipeline/SKILL.md` (4 bản tracked) mô tả rõ hành vi "rate-limit resume", dùng
  thuật ngữ khác với "implement-resume"/decision-resume đã có sẵn, không gây nhầm lẫn.
- [ ] Đọc lại log mẫu của 1 lượt rate-limit-resume: có dòng log phân biệt rõ (ví dụ "Rate-limit
  resume #N"), khác dòng log "attempt X/3" của attempt kỹ thuật thường.

## Constraints

- Chỉ xử lý rate-limit Claude thật (2 chuỗi đã có từ `control-plane-token-switch`), không mở rộng
  sang GitHub Actions job timeout — runner bị huỷ hoàn toàn thì không còn gì để resume.
- Rate-limit resume không tính vào `ATTEMPTS=3` kỹ thuật hiện có; không thêm cap cứng cho số lần
  resume liên tiếp — dựa hoàn toàn vào `SWITCH_EXHAUSTED` (pool hết token) làm giới hạn tự nhiên.
- Dùng `-c/--continue` của Claude Code CLI, không tự generate/truyền `--session-id` tường minh.
- Lệnh resume tự thất bại kỹ thuật → fallback đúng hành vi reset-và-làm-lại cũ, tính vào 3
  `ATTEMPTS`.
- Áp dụng đồng nhất cho cả 3 stage; đồng bộ cả 6 bản tracked của mỗi workflow YAML (nguồn, vendor
  skill, vendor `.github/workflows/` đã cài đặt thật, + 3 mirror `.codex/` tương ứng) và 4 bản
  tracked của `SKILL.md` — đúng quy ước sẵn có của `online-pipeline`.
- Không đổi gì ở repo `control-plane` riêng.
- Thuật ngữ "rate-limit resume" phải khác với "implement-resume"/decision-resume đã có sẵn trong
  code/log/`SKILL.md`, tránh nhầm lẫn giữa 2 cơ chế không liên quan nhau.

## Risks

- **Risk 1 (phát hiện lúc validation round 1, 2 reviewer độc lập cùng bắt trúng):** Baseline
  rollback (`$LAST_GOOD`/`origin/main`) chỉ chốt 1 lần duy nhất trước khi vào loop, không advance
  sau mỗi lần rate-limit-resume thành công. Nếu 1 hoặc nhiều lần resume thành công (đã làm thêm
  việc thật) rồi mới gặp 1 lỗi kỹ thuật **thường** (không phải rate-limit) → fallback reset về
  đúng baseline ban đầu, xoá luôn cả phần việc của các lần resume thành công trước đó — compound
  case này mất **nhiều hơn** cả hành vi trước feature. **Mitigation:** chấp nhận là limitation đã
  biết cho v1 (quyết định rõ của người dùng) — không advance checkpoint, vì việc đó cần thêm cơ
  chế "chốt tạm" riêng cho `knowledge-init` (stage này không hề commit tới khi thành công, chỉ có
  uncommitted working-tree edits, nên "advance baseline" không có ref nào sẵn để dùng) — phức tạp
  hơn mức cần cho v1. Feature vẫn bảo vệ đúng trường hợp phổ biến nhất: 1 lần rate-limit giữa lúc
  code, không có lỗi kỹ thuật khác xảy ra sau đó trong cùng job. Để lại cho 1 user-spec riêng nếu
  compound case này gây vấn đề thật trong thực tế.

- **Risk 2:** Nhầm lẫn giữa cơ chế mới và mechanism "resume" có sẵn (`ROUTE_ACTION='resume'`/stage
  `implement-resume` — job hoàn toàn mới khi user trả lời câu hỏi quyết định giữa kỳ, không liên
  quan session CLI). **Mitigation:** dùng thuật ngữ riêng "rate-limit resume" xuyên suốt log/
  `SKILL.md`/biến bash; ghi rõ trong `SKILL.md` rằng 2 cơ chế không giao nhau.
- **Risk 3:** `claude -p -c` continue sai hội thoại nếu có nhiều hơn 1 session trong cùng working
  directory. **Mitigation:** đã xác nhận qua code research — trong loop này chỉ có đúng 1
  `claude -p` active tại 1 thời điểm mỗi job (GitHub-hosted runner là VM mới mỗi job, không có
  session cũ từ job khác), nên "`-c` tiếp tục hội thoại gần nhất" luôn đúng ý; không cần
  `--session-id` tường minh.
- **Risk 3b (phát hiện + verify lúc validation round 3):** `try_rate_limit_switch` luôn đổi
  `CURRENT_TOKEN` sang 1 account khác trong pool trước khi resume — nghi ngờ ban đầu là session
  CLI có thể bị ràng buộc với account đã tạo ra nó, khiến resume dưới account khác thất bại.
  **Đã verify, không còn là rủi ro thật:** đọc trực tiếp cấu trúc file session cục bộ thật
  (`~/.claude/projects/<cwd>/<session-id>.jsonl`) — không có field `token`/`apiKey`/`accountId`
  nào, chỉ có `message: {role, content}` + metadata (cwd, cost, requestId...). Đúng kỳ vọng: API
  `/v1/messages` của Anthropic stateless (mỗi lần gọi gửi lại toàn bộ `messages`, auth theo từng
  request, không có "session" phía server gắn với 1 API key) — `-c`/`-r` chỉ đọc lại transcript
  cục bộ rồi gửi như 1 API call bình thường với credential hiện tại, không có cổng kiểm tra
  account nào có thể fail. Xem code-research.md.
- **Risk 4:** Không cap cứng số lần resume liên tiếp có thể kéo dài thời gian job nếu pool có nhiều
  token khả dụng. Worst case thật (không chỉ "tốn phút CI"): nếu tổng thời gian cộng dồn của nhiều
  lần resume liên tiếp chạm `timeout-minutes`/giới hạn 6 giờ của runner (xem Constraints — đã loại
  khỏi phạm vi feature này), runner bị GitHub huỷ hoàn toàn **ngay lập tức, không qua bất kỳ step
  "Report..." nào** — không `gh issue comment`, không relay Slack, mất dấu vết hoàn toàn, khác hẳn
  mọi nhánh lỗi khác của feature này đều có báo lỗi rõ ràng. **Mitigation:** chấp nhận theo lựa
  chọn rõ của người dùng — dựa vào `SWITCH_EXHAUSTED` làm giới hạn tự nhiên (pool hữu hạn nên số
  lần resume cộng dồn trong thực tế bị chặn bởi đúng số token khả dụng, không thực sự "vô hạn");
  không mitigate thêm cho v1, để lại cho 1 user-spec riêng nếu vấn đề này xảy ra thật.
- **Risk 5:** Rate-limit thật rất khó ép xảy ra theo ý muốn để test end-to-end (đã ghi nhận tương tự
  ở `control-plane-token-switch`). **Mitigation:** dựng sẵn 1 file output giả chứa đúng chuỗi lỗi để
  kiểm tra logic match-pattern + 2-counter cục bộ, không cần `claude -p` thật; xác nhận end-to-end
  thụ động khi rate-limit thật xảy ra tiếp theo trong production (đọc log Actions).

## Accepted Decisions

- Baseline rollback (`$LAST_GOOD`/`origin/main`) KHÔNG advance sau mỗi lần rate-limit-resume thành
  công — phát hiện lúc validation round 1 (2 reviewer độc lập cùng bắt trúng) rằng việc không
  advance có thể làm mất tiến độ của các lần resume trước nếu sau đó gặp 1 lỗi kỹ thuật thường;
  cân nhắc "advance checkpoint" (sửa đúng gốc) nhưng từ chối vì `knowledge-init` không có commit
  ref nào để advance (chỉ có uncommitted working-tree edits) — cần thêm cơ chế chốt tạm riêng,
  phức tạp hơn mức cần cho v1. Chấp nhận là limitation đã biết (xem Risk 1) — feature vẫn bảo vệ
  đúng trường hợp phổ biến nhất (1 lần rate-limit, không có lỗi kỹ thuật khác xảy ra sau đó).
- Phạm vi chỉ giới hạn ở rate-limit Claude thật (2 chuỗi đã match sẵn) — không mở rộng sang GitHub
  Actions job timeout, vì job/runner bị huỷ hoàn toàn thì không còn gì để resume (khác bản chất).
- Áp dụng cho cả 3 stage (`implement`, `knowledge-init`, `userspec-turn`) đồng bộ, dùng chung y hệt
  logic — dù `knowledge-init`/`userspec-turn` không có "code dở" nhiều ý nghĩa như `implement`, vẫn
  chọn đồng bộ thay vì chỉ làm riêng cho `implement`.
- Dùng `-c/--continue` của Claude Code CLI thay vì tự generate + truyền `--session-id` tường minh —
  đơn giản hơn, vì mỗi job chỉ có đúng 1 `claude -p` active tại 1 thời điểm trong cùng working
  directory nên "hội thoại gần nhất" luôn đúng.
- Rate-limit resume tách thành 1 counter riêng, KHÔNG tính vào `ATTEMPTS=3` kỹ thuật hiện có — đảo
  lại quyết định đã chốt ở `control-plane-token-switch` ("reset về `$LAST_GOOD` rồi làm lại từ đầu,
  không xây cơ chế resume giữa chừng mới"), chỉ cho đúng trường hợp rate-limit.
- Không thêm cap cứng cho số lần rate-limit-resume liên tiếp — dựa hoàn toàn vào
  `SWITCH_EXHAUSTED` (pool hết token) làm giới hạn tự nhiên; chấp nhận rủi ro job kéo dài nếu pool
  lớn (xem Risk 3), không mitigate thêm cho v1.
- Lệnh resume tự thất bại kỹ thuật (khác rate-limit) → fallback về đúng hành vi reset-và-làm-lại cũ,
  tính vào `ATTEMPTS=3` — không coi mọi thất bại sau switch-token là rate-limit resume.
- Dùng thuật ngữ riêng "rate-limit resume" (khác "implement-resume"/decision-resume đã có sẵn) để
  tránh nhầm lẫn 2 cơ chế không liên quan nhau trong codebase.
- Verification cho logic bash (không viết unit test JS được) dùng 1 file output giả chứa đúng chuỗi
  rate-limit để chạy thử cục bộ, thay vì chỉ dựa vào xác nhận thụ động khi rate-limit thật xảy ra —
  đúng tinh thần đã áp dụng ở `control-plane-token-switch` cho loại logic bash-trong-YAML này.

## Testing

**Unit tests:** không cần — không có code JS/logic thuần nào mới (toàn bộ thay đổi là bash trong 3
workflow YAML), giống đúng tính chất của phần rate-limit-detection ở `control-plane-token-switch`
("không viết được bằng unit test JS").

**Integration tests:** không cần hạ tầng riêng — tách đoạn bash match-pattern + `try_rate_limit_switch`
+ logic 2-counter ra khỏi lời gọi `claude -p` thật, chạy thử cục bộ với 3 input giả:
(1) `$CLAUDE_OUTPUT_FILE` giả chứa đúng 1 trong 2 chuỗi rate-limit (hoặc không khớp, để test nhánh
ngược); (2) phản hồi giả của `POST $SLACK_WORKER_URL/switch-token/auto` — override hàm gọi `curl`
đó (hoặc trỏ `$SLACK_WORKER_URL` vào 1 local mock HTTP server tối giản) để trả lần lượt
`{ok:true,...}`, `{ok:false,error:"no_alias_available"}`, và 1 lỗi mạng/5xx chung — không cần
gọi Worker thật hay rút cạn pool thật; (3) kết quả giả của chính lời gọi `claude -p -c` resume —
override lệnh `claude` trong harness cục bộ (ví dụ 1 script giả cùng tên, đứng trước `claude` thật
trong `$PATH`) để mô phỏng lần lượt: resume thành công (exit 0, tạo thêm 1 commit test) và resume
tự thất bại kỹ thuật (exit khác 0 / in ra lỗi không phải rate-limit) — không cần `claude -p` thật
chạy. Dùng 3 input giả này để xác nhận: (a) rate-limit resume không tăng `ATTEMPT` kỹ thuật (Agent
Verification bước 1), (b) lỗi không khớp mẫu/Worker lỗi mạng chung vẫn rơi đúng về nhánh reset cũ
tính vào `ATTEMPTS` (bước 2-3), (c) `SWITCH_EXHAUSTED` (từ `no_alias_available`) vẫn dừng loop
đúng như cũ (bước 4), (d) resume tự thất bại kỹ thuật (stub lệnh `claude`) fallback đúng (bước 5),
(e) baseline không advance qua nhiều lần resume thành công liên tiếp, kể cả khi resume đó có tạo
thêm commit thật bằng stub lệnh `claude` (đúng limitation đã chấp nhận ở Risk 1) (bước 6), (f)
không cap cứng số lần resume liên tiếp (bước 7).

**E2E tests:** cần, nhưng không thể ép xảy ra theo ý muốn — rate-limit Claude thật phụ thuộc hạn mức
thật của account. Xác nhận bằng quan sát thụ động lần rate-limit tự nhiên tiếp theo trong production
(đọc log Actions), giống cách verify đã chấp nhận ở `control-plane-token-switch`.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Dựng 1 file output giả chứa đúng chuỗi `"You've hit your session limit"` (và riêng 1 lần với `"...weekly limit"`) + stub `switch-token/auto` trả `{ok:true,...}` (xem Testing), chạy thử đoạn bash match-pattern + logic 2-counter cục bộ (không gọi `claude -p` thật) | Rate-limit resume được kích hoạt đúng (AC1): không có `git reset`/`git clean`; `ATTEMPT` kỹ thuật không tăng; log có dòng phân biệt "Rate-limit resume #N". |
| 2. Dựng 1 file output giả KHÔNG khớp cả 2 mẫu rate-limit, chạy lại cùng đoạn bash | Rơi đúng về nhánh technical-failure cũ (AC2): `git reset` chạy, `ATTEMPT` kỹ thuật tăng 1 — y hệt hành vi trước feature này. |
| 3. Dựng 1 file output giả khớp mẫu rate-limit NHƯNG stub `switch-token/auto` trả 1 lỗi mạng/5xx chung (khác `no_alias_available`), chạy lại cùng đoạn bash | Rơi về đúng nhánh technical-failure cũ (AC3): `git reset` chạy, `ATTEMPT` kỹ thuật tăng 1, không bị coi là rate-limit resume. |
| 4. Dựng cùng file output giả khớp mẫu rate-limit, stub `switch-token/auto` trả `{ok:false, error:"no_alias_available"}` | Dừng ngay (AC4): `SWITCH_EXHAUSTED=true`, không chạy thêm attempt nào, báo lỗi đúng message hiện có — hành vi không đổi so với trước feature này. |
| 5. Stub `switch-token/auto` trả `{ok:true,...}`, VÀ stub chính lệnh `claude` (script giả đứng trước `claude` thật trong `$PATH`) để mô phỏng lần gọi `-c` resume đó tự thất bại kỹ thuật (exit khác 0, output không phải rate-limit) | Fallback về reset-và-gửi-lại-prompt-gốc (AC5): `git reset` chạy, `ATTEMPT` kỹ thuật tăng 1 — không bị coi là rate-limit resume dù token đã đổi thành công. |
| 6. Stub `switch-token/auto` trả `{ok:true,...}` 1 lần, VÀ stub lệnh `claude` để mô phỏng resume đó THÀNH CÔNG (exit 0 + tạo thêm 1 commit test thật) — sau đó giả lập tiếp 1 lỗi kỹ thuật thường (không phải rate-limit) ngay lượt kế | Fallback reset về đúng baseline ban đầu — trước cả lần resume (AC6); xoá luôn commit test vừa thêm bởi lần resume thành công đó — đúng limitation đã chấp nhận ở Risk 1, không phải bug cần sửa. |
| 7. Giả lập 3 lần stub `switch-token/auto` trả `{ok:true,...}` liên tiếp (pool "còn nhiều token") trong 1 lần chạy thử | Cả 3 lần đều không tăng `ATTEMPT` kỹ thuật (AC7); không bị `ATTEMPTS=3` chặn lại; loop chỉ dừng khi stub trả `no_alias_available`. |
| 8. Đọc lại cả 3 workflow sau khi sửa (`online-pipeline-implement.yml`, `online-pipeline-knowledge-init.yml`, `online-pipeline-userspec.yml`) | Logic rate-limit-resume giống nhau cả 3 file; `ATTEMPT` kỹ thuật và counter rate-limit-resume tách biệt rõ; fallback khi lệnh resume tự lỗi kỹ thuật đúng như spec; bước reset/clean bị bỏ qua đúng với từng stage (có/không có `git push --force` tuỳ stage). |
| 9. Kiểm tra `diff` giữa 6 bản tracked của mỗi workflow YAML (nguồn, vendor skill, vendor `.github/workflows/`, + 3 mirror `.codex/`) | `diff` sạch hoàn toàn (không có ngoại lệ nào cho workflow YAML). |
| 10. Kiểm tra `diff` giữa 4 bản tracked của `SKILL.md` (nguồn, vendor, 2 mirror `.codex/`) | `diff` sạch, **trừ đúng 1 dòng header `<!-- Generated by sync-to-codex v1. Do not edit directly. -->`** mà `.codex/skills/online-pipeline/SKILL.md` luôn có thêm (quy ước tự động của `sync-to-codex.py` cho mọi file `.md`, không phải lỗi) — không áp dụng cho các workflow YAML ở bước 9. |
| 11. Đọc lại `skills/online-pipeline/SKILL.md` sau khi sửa | Có mục mô tả rõ "rate-limit resume" cho cả 3 stage, phân biệt rõ với "implement-resume"/decision-resume đã có sẵn, nêu rõ 2 cơ chế không giao nhau. |
| 12. Đọc trực tiếp cấu trúc 1 file session cục bộ thật (`~/.claude/projects/<cwd>/<session-id>.jsonl`) để xác nhận lại Risk 3b trước khi implement | Không có field `token`/`apiKey`/`accountId` nào trong file; `message` chỉ có `{role, content}` — xác nhận resume không bị ràng buộc với account đã tạo session, dùng token khác vẫn resume được bình thường. |

### User Verification

- Theo dõi log GitHub Actions ở lần rate-limit Claude thật xảy ra tiếp theo trong production — xác
  nhận token được đổi, `claude -p -c` được gọi, công việc tiếp tục đúng mạch (không bị reset), và
  attempt đó không làm giảm số `ATTEMPTS` còn lại. Agent không thể ép rate-limit thật xảy ra theo ý
  muốn để verify chủ động.
