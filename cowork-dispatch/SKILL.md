---
name: cowork-dispatch
description: "協調 CoWork 協作流程：規格撰寫、計劃審查、任務派發、狀態監控與結果驗收。支援多種 Code Agent Client。"
compatibility: "需要 git、zmx。依設定的 client 需要對應的 CLI 工具。"
---

# CoWork Dispatch

協調 CoWork 協作流程：規格準備、計劃審查、YAML 佇列管理、任務派發、工作階段監控、結果清理。

## 適用時機

使用者提及 CoWork、多 agent 協作、規格/計劃撰寫、任務派發、佇列狀態查詢、或結果清理時使用。

## Client 設定

### 可用 Client

| Client ID  | 呼叫方式                         | 適用場景              |
| ---------- | -------------------------------- | --------------------- |
| claude-zmx | zmx run + claude tui（detached） | 長時任務 + 能人工監控 |
| claude-cli | claude -p "\<prompt\>"           | agent 呼叫            |
| codex-exec | codex exec "\<prompt\>"          | agent 呼叫            |
| codex-zmx  | zmx run + codex tui（detached）  | 長時任務 + 能人工監控 |

完整指令範例：

```bash
# claude-cli
claude --model=claude-opus-4-8 --effort xhigh --dangerously-skip-permissions -p "your prompt"
# model: claude-opus-4-8, claude-sonnet-4-6, claude-haiku-4-5
# effort: low, medium, high, xhigh, max

# codex-exec
codex exec --sandbox danger-full-access --model=gpt-5.5 -c model_reasoning_effort="high" "your prompt"
# sandbox: read-only, workspace-write, danger-full-access
# model: gpt-5.5
# model_reasoning_effort: low, medium, high, xhigh
```

### 解析優先序

所有維度（client、model、effort、monitor）皆遵循同一優先序：

1. **使用者 prompt** — 最高優先。例如「spec 用 codex-exec 審查」「code 用 claude-zmx effort max」「不要監控」。
2. **`.cowork/config.yaml`** — 專案預設（選用）。
3. **內建預設** — 兜底。

`.cowork/config.yaml` 格式（僅列出需覆蓋的欄位）：

```yaml
phases:
  spec_reviewer:
    client: codex-exec
    model: gpt-5.5
    effort: xhigh
  plan_writer:
    client: codex-exec
    model: gpt-5.5
    effort: xhigh
  code_executor:
    client: codex-zmx
    model: gpt-5.5
    effort: medium
monitor:                  # 僅影響 Claude Code dispatch（方式 A）
  enabled: true           # false = 不監控
  interval: 300           # 輪詢間隔（秒）
  timeout: 3600000        # Monitor 逾時（毫秒）
```

### 內建預設

| 階段          | 角色     | 預設 Client | 預設 model | 預設 effort |
| ------------- | -------- | ----------- | ---------- | ----------- |
| spec_writer   | 撰寫規格 | self        | —          | —           |
| spec_reviewer | 審查規格 | codex-exec  | gpt-5.5    | xhigh       |
| plan_writer   | 撰寫計劃 | codex-exec  | gpt-5.5    | xhigh       |
| plan_reviewer | 審查計劃 | self        | —          | —           |
| code_executor | 執行實作 | codex-zmx   | gpt-5.5    | medium      |

`self` = 目前執行此 skill 的 agent 自行處理，不委派外部 client。

model 與 effort 會依 client 類型映射到對應 CLI 旗標：
- claude 系列：`--model=<model> --effort <effort>`
- codex 系列：`--model=<model> -c model_reasoning_effort="<effort>"`

## 預設執行模式

使用者啟動 CoWork 時，**全程自動執行**，不在階段間暫停：

進度檢查清單 — 每次啟動 CoWork 時建立追蹤，逐項完成後才進入下一階段：

```
- [ ] init：建立 .cowork/ 目錄與佇列檔案
- [ ] spec：腦力激盪 → 草稿 → 審查迴圈 → 規格核准
- [ ] plan：產生計劃 → 審查迴圈 → 規格與計劃一起提交
- [ ] dispatch：建立任務寫入 tasks.yaml → 啟動 code_executor
- [ ] monitor：健康檢查 + 完成監控 → 確認 results.yaml 有結果
- [ ] verify：讀取 results.yaml → git diff → 執行測試套件 → 失敗則診斷並報告使用者
- [ ] report：向使用者報告最終狀態
```

- 每個階段轉換以一句話報告進度。
- **不得推斷完成狀態。** 只有實際執行該階段的檢查步驟後，才可勾選完成。
- **不得跳步。** 除非使用者明確要求跳過特定步驟。

**暫停條件：** 設計歧義需使用者決策、審查發現多解的重大問題、不可恢復的錯誤、執行結果 `failed`。

**不暫停：** 小修正（直接修正後繼續）、計劃缺漏（回饋再審）、503 錯誤（自動重試）、單帳號配額耗盡（自動切換）、測試失敗（executor 自行處理）。

**執行完成後：** 讀取 `results.yaml`、檢視 `git diff`、執行完整測試套件、向使用者報告。

## 指令

### init

1. 在專案根目錄建立 `.cowork/`（如不存在）。
2. 確保 `.gitignore` 排除 `.cowork/`；若無則新增並提交。
3. 建立 `tasks.yaml` 和 `results.yaml`，初始內容為 `[]`。不覆寫已有內容的檔案。
4. 建立 `logs/` 子目錄（如不存在）。

### spec

1. 先使用 Superpowers 腦力激盪工作流。
2. 使用 Superpowers 撰寫計劃工作流產出規格文件。
3. `spec_writer` 撰寫初稿 → 依 `spec_reviewer` 的 client 類型呼叫審查。
4. CLI client 執行時 stdout 回傳給 AI，同時將完整輸出（stdout + stderr）寫入 log 檔：
   ```bash
   LOG=.cowork/logs/spec-review-$(date -u +%Y%m%dT%H%M%SZ).log
   codex exec ... "review prompt" 2>>"$LOG" | tee -a "$LOG"
   ```
   log 命名格式：`spec-review-<ISO8601>.log`。多輪審查產生多個 log 檔。
5. 小問題由 reviewer 直接修正；重大問題由 writer 修正後重新送審。
6. 迴圈直到規格核准。不將審查工作寫入 `tasks.yaml`。

### plan

1. 規格核准後才開始。
2. 依 `plan_writer` 的 client 類型呼叫產生實作計劃 → `plan_reviewer` 對照規格審查。
3. CLI client 執行時 stdout 回傳給 AI，同時將完整輸出（stdout + stderr）寫入 log 檔：
   ```bash
   LOG=.cowork/logs/plan-write-$(date -u +%Y%m%dT%H%M%SZ).log
   codex exec ... "plan prompt" 2>>"$LOG" | tee -a "$LOG"
   ```
   log 命名格式：`plan-write-<ISO8601>.log`。審查迴圈中每次呼叫產生獨立 log。
4. 有缺漏或偏離時，回饋 writer 修正後再審。
5. 規格與計劃皆核准後一起提交：
   ```bash
   git commit -m "docs: add <feature-name> spec and implementation plan"
   ```
6. 不將計劃撰寫或審查工作寫入 `tasks.yaml`。

### dispatch

1. 確認計劃已通過完整審查迴圈（不是僅存在或看似完成）。
2. 建立**一個摘要任務**指向計劃檔案，不拆分為多個細粒度任務。
3. 附加任務至 `tasks.yaml`，欄位順序與格式參見 `cowork-runner/references/yaml-schema.md`。
   - `task_id`：`task-{unix_ms}-{random_hex_3}`
   - `goal`：一句話描述完整實作範圍
   - `context.plan_file`、`context.spec_file`、`context.related_files`
   - `constraints`：保留計劃與使用者的實作約束
   - `created_by`：目前 agent 的識別名稱
4. 依解析出的 `code_executor` client 啟動執行：

   **zmx client（codex-zmx / claude-zmx）：**

   客戶端短名: claude: cc, codex: cx。

   工作階段命名 `<客戶端短名>-<英文短名>`（如 `cx-user-api`）。先 `zmx list` 檢查同名 session 是否存在，已存在則加數字後綴（如 `cx-user-api-2`）。

   codex-zmx：

   ```bash
   zmx run cx-<name> -d bash -c 'codex --dangerously-bypass-approvals-and-sandbox --model=<model> -c model_reasoning_effort="<effort>" "[<task_id>] You are the cowork-runner. Read .cowork/tasks.yaml, execute all pending implementation tasks following the cowork-runner skill. Use sub-agents to parallelize independent development work when beneficial. Write .cowork/results.yaml, remove completed tasks, then exit."'
   ```

   claude-zmx：

   ```bash
   zmx run cc-<name> -d bash -c 'claude --name <task_id> --model=<model> --effort <effort> --dangerously-skip-permissions -p "[<task_id>] You are the cowork-runner. Read .cowork/tasks.yaml, execute all pending implementation tasks following the cowork-runner skill. Use sub-agents to parallelize independent development work when beneficial. Write .cowork/results.yaml, remove completed tasks, then exit."'
   ```

5. **zmx 監控** — 依 dispatch agent 的能力與 `monitor` 設定選擇方式。`monitor.enabled: false` 或使用者說「不要監控」時跳過監控，直接告知使用者手動檢查。

   監控腳本（兩種方式共用）：

   ```bash
   SESSION="<session>"
   TID="<task_id>"
   while sleep ${INTERVAL:-300}; do
     zmx list 2>/dev/null | grep -q "$SESSION" || {
       echo "ALERT: session gone"; zmx history "$SESSION" | tail -20; break
     }
     head -3 .cowork/results.yaml 2>/dev/null | grep -q "$TID" && {
       echo "DONE: $TID"; zmx history "$SESSION" | tail -20; break
     }
     echo "heartbeat: waiting (${SECONDS}s elapsed)"
   done
   ```

   完成判定：`results.yaml` 前幾行包含已派發的 `task_id`。

   **方式 A：Claude Code dispatch（有 Monitor 工具）**

   初始健康檢查 — Bash `run_in_background: true`：
   ```bash
   sleep 180 && zmx history <session> | tail -20
   ```
   啟動後立即進入完成監控，健康檢查結果到達時處理錯誤。

   使用 **Monitor** 工具執行監控腳本（`timeout_ms` 取自 `monitor.timeout`，預設 `3600000`；`persistent: false`）。非阻塞，每行 `echo` 成為 agent 通知。

   逾時行為：`timeout_ms` 到期時 Monitor 終止腳本，但 zmx session 繼續運行。agent 須手動檢查 `zmx list` 和 `results.yaml`。

   **方式 B：Codex dispatch 或其他無 Monitor 的 agent**

   直接以 bash 執行監控腳本，阻塞直到完成或 session 消失。

6. **zmx 錯誤處理：**
   - 503 錯誤：`zmx send <session> "GO"`
   - codex 帳號配額耗盡：`codex-multi-auth check` → `codex-multi-auth switch <n>` → `zmx kill <session>` → 依「Session 恢復」流程查詢 session ID 並 resume
   - 全部帳號無配額：報告需手動處理，不刪除 `tasks.yaml` 中的任務。

7. 告知使用者可用指令：`zmx attach <session>`（即時檢視）、`zmx tail <session>`（即時跟蹤輸出）、`zmx history <session> | tail -20`（近期輸出）、`zmx list`（所有工作階段）、`Ctrl+\`（脫離 attach 不終止）。

   **cli client（codex-exec / claude-cli）：** 將輸出導向 log 檔後等待完成，再讀取 `results.yaml`：
   ```bash
   LOG=.cowork/logs/code-exec-$(date -u +%Y%m%dT%H%M%SZ).log
   codex exec ... "runner prompt" 2>>"$LOG" | tee -a "$LOG"
   ```
   不需 zmx session 管理與監控。

### status

1. 讀取 `tasks.yaml` 和 `results.yaml`（遺失、空字串、`[]` 均視為空）。
2. 摘要報告：待處理任務、已完成/失敗/部分完成結果。
3. 執行 `zmx list`，顯示 `cx-` 工作階段狀態。

### clean

重設 `results.yaml` 為 `[]`。不修改 `tasks.yaml`。

## Session 恢復

zmx session 被 kill 或意外中斷後，底層的 Codex/Claude 對話仍保存在磁碟上。恢復流程：

1. `zmx kill <session>`（如 session 仍存在）。
2. 查詢 session ID：

   codex — 從 session 檔案中搜尋 prompt 裡的 task_id，UUID 直接從檔名取：
   ```bash
   grep -l "<task_id>" ~/.codex/sessions/$(date -u +%Y/%m/%d)/*.jsonl | head -1 | grep -oP '[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}'
   ```
   若跨日則搜前一天：`$(date -u -d yesterday +%Y/%m/%d)`。

   claude — 直接用啟動時 `--name` 指定的 `<task_id>`。

3. 開新 zmx session 恢復對話：

   codex-zmx：
   ```bash
   zmx run cx-<name> -d bash -c 'codex resume --include-non-interactive <session_uuid> "繼續執行未完成的任務"'
   ```

   claude-zmx：
   ```bash
   zmx run cc-<name> -d bash -c 'claude --resume <task_id> --dangerously-skip-permissions -p "繼續執行未完成的任務"'
   ```

**不要使用 `codex resume --last` 或 `claude --continue`。** 這些指令按工作目錄取最新 session，若中間有人類手動開過 session 會恢復到錯誤的對話。

## Log 規則

- 所有 CLI client（codex-exec / claude-cli）執行時使用 `2>>"$LOG" | tee -a "$LOG"` 模式：
  - stdout（fd1）→ terminal（AI 可見）+ log 檔
  - stderr（fd2）→ 僅 log 檔
- 命名格式：`<phase>-<ISO8601>.log`，例如 `spec-review-20260623T100500Z.log`。
- **僅在超時或異常時**才主動讀取 log 檔排查：`tail -20 .cowork/logs/<對應 log>`。
- zmx client 不受影響，仍用 `zmx history` 查看輸出。

## 注意事項

- zmx client 必須使用 `zmx run -d`（detached）。沒有 `-d` 時 zmx 阻塞呼叫端，呼叫端退出時工作階段被終止。
- zmx 必須用 `bash -c '...'` 包裹指令。不加時帶旗標的指令被視為單一執行檔名。
- codex-zmx 用 `codex --dangerously-bypass-approvals-and-sandbox`（非 `codex exec`）。
- claude-zmx 用 `claude --dangerously-skip-permissions -p`。
- `codex-multi-auth` 僅適用於 codex 系列 client。
- 不要從 `zmx list`、`created=...`、`date +%s` 或任何 zmx 時間戳計算經過時間。
- 不要在監控腳本之外額外執行 `zmx history` 檢查。
- 佇列檔案規則參見 `cowork-runner/references/yaml-schema.md`。
