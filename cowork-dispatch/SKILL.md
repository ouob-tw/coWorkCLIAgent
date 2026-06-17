---
name: cowork-dispatch
description: "在 Claude Code 中協調 CoWork 協作：規格撰寫、計劃審查、任務派發至 Codex CLI、狀態監控與結果驗收。"
compatibility: "Claude Code 專用。需要 git、zmx、codex、codex-multi-auth。"
---

# CoWork Dispatch

在 Claude Code 端協調 CoWork 協作流程：規格準備、計劃審查、YAML 佇列管理、任務派發、工作階段監控、結果清理。

## 適用時機

使用者提及 CoWork、Claude Code + Codex 協作、規格/計劃撰寫、任務派發、佇列狀態查詢、或結果清理時使用。

## 角色分工

| 角色 | 職責 |
|------|------|
| Claude Code | 撰寫規格、審查計劃、派發已核准的執行任務 |
| Codex 外掛（Claude Code 內） | 協助規格審查、產生實作計劃 |
| 外部 Codex CLI | 僅執行 `tasks.yaml` 中的已核准任務 |

`tasks.yaml` 僅存放已核准的執行工作。規格審查、計劃審查不寫入 YAML。

## 預設執行模式

使用者啟動 CoWork 時，**全程自動執行**，不在階段間暫停：

進度檢查清單 — 每次啟動 CoWork 時建立追蹤，逐項完成後才進入下一階段：

```
- [ ] init：建立 .cowork/ 目錄與佇列檔案
- [ ] spec：腦力激盪 → 草稿 → Codex 審查迴圈 → 規格核准
- [ ] plan：Codex 產生計劃 → Claude 審查迴圈 → 規格與計劃一起提交
- [ ] dispatch：建立任務寫入 tasks.yaml → zmx 啟動 Codex CLI
- [ ] monitor：背景健康檢查 + Monitor 完成監控 → 確認 results.yaml 有結果
- [ ] verify：讀取 results.yaml → git diff → 執行測試套件
- [ ] report：向使用者報告最終狀態
```

- 每個階段轉換以一句話報告進度。
- **不得推斷完成狀態。** 只有實際執行該階段的檢查步驟後，才可勾選完成。
- **不得跳步。** 除非使用者明確要求跳過特定步驟。

**暫停條件：** 設計歧義需使用者決策、Codex 審查發現多解的重大問題、不可恢復的錯誤、執行結果 `failed`。

**不暫停：** 小修正（直接修正後繼續）、計劃缺漏（回饋 Codex 再審）、503 錯誤（自動重試）、單帳號配額耗盡（自動切換）、測試失敗（Codex 自行處理）。

**執行完成後：** 讀取 `results.yaml`、檢視 `git diff`、執行完整測試套件、向使用者報告。

## 指令

### init

1. 在專案根目錄建立 `.cowork/`（如不存在）。
2. 確保 `.gitignore` 排除 `.cowork/`；若無則新增並提交。
3. 建立 `tasks.yaml` 和 `results.yaml`，初始內容為 `[]`。不覆寫已有內容的檔案。

### spec

1. 先使用 Superpowers 腦力激盪工作流。
2. 使用 Superpowers 撰寫計劃工作流產出規格文件。
3. Claude Code 撰寫初稿 → 呼叫 Codex 外掛審查。
4. 小問題由 Codex 直接修正；重大問題由 Claude Code 修正後重新送審。
5. 迴圈直到規格核准。不將審查工作寫入 `tasks.yaml`。

### plan

1. 規格核准後才開始。
2. 呼叫 Codex 外掛產生實作計劃 → Claude Code 對照規格審查。
3. 有缺漏或偏離時，回饋 Codex 修正後再審。
4. 規格與計劃皆核准後一起提交：
   ```bash
   git commit -m "docs: add <feature-name> spec and implementation plan"
   ```
5. 不將計劃撰寫或審查工作寫入 `tasks.yaml`。

### dispatch

1. 確認計劃已通過完整審查迴圈（不是僅存在或看似完成）。
2. 建立**一個摘要任務**指向計劃檔案，不拆分為多個細粒度任務。
3. 附加任務至 `tasks.yaml`，欄位順序與格式參見 `cowork-runner/references/yaml-schema.md`。
   - `task_id`：`task-{unix_ms}-{random_hex_3}`
   - `goal`：一句話描述完整實作範圍
   - `context.plan_file`、`context.spec_file`、`context.related_files`
   - `constraints`：保留計劃與使用者的實作約束
   - `created_by: claude-code`
4. zmx 工作階段命名：`cx-<英文短名>`（如 `cx-user-api`）。
5. `zmx list` 檢查衝突，已存在則停止報告。
6. 以 detached 模式啟動 Codex CLI：

   ```bash
   zmx run cx-<name> -d bash -c 'codex --dangerously-bypass-approvals-and-sandbox "You are the cowork-runner. Read .cowork/tasks.yaml, execute all pending implementation tasks following the cowork-runner skill. Use sub-agents to parallelize independent development work when beneficial. Write .cowork/results.yaml, remove completed tasks, then exit."'
   ```

7. **初始健康檢查** — 以背景任務執行（Bash `run_in_background: true`）：

   ```bash
   sleep 180 && zmx history cx-<name> | tail -20
   ```

   啟動後立即進入步驟 8，健康檢查結果到達時處理錯誤。

8. **完成監控** — 使用 **Monitor** 工具（`timeout_ms: 3600000`，`persistent: false`）：

   ```bash
   EXPECTED_TASK_ID="<task_id>"
   SESSION="cx-<name>"
   ITER=0
   while true; do
     if ! zmx list 2>/dev/null | grep -q "$SESSION"; then
       echo "ALERT: session $SESSION no longer running"
       zmx history "$SESSION" | tail -20
       break
     fi
     if [ -f .cowork/results.yaml ] && uv run --with pyyaml python - "$EXPECTED_TASK_ID" <<'PY'
   import sys; from pathlib import Path; import yaml
   expected = sys.argv[1]
   data = yaml.safe_load(Path(".cowork/results.yaml").read_text()) or []
   sys.exit(0 if isinstance(data, list) and data and isinstance(data[0], dict) and data[0].get("task_id") == expected else 1)
   PY
     then
       echo "DONE: task $EXPECTED_TASK_ID completed"
       zmx history "$SESSION" | tail -20
       break
     fi
     ITER=$((ITER + 1))
     if [ $((ITER % 6)) -eq 0 ]; then
       echo "heartbeat: session $SESSION still running ($((ITER * 30))s elapsed)"
     fi
     sleep 30
   done
   ```

   完成判定：`results.yaml` 為 YAML 列表且首項 `task_id` 匹配已派發任務。不以 `tasks.yaml` 為空、`results.yaml` 存在、或 zmx history 作為完成依據。

9. **錯誤處理：**
   - 503 錯誤：`zmx send cx-<name> "GO"`
   - 帳號配額耗盡：`codex-multi-auth check` → `codex-multi-auth switch <n>` → `zmx send cx-<name> "GO"`
   - 全部帳號無配額：報告需手動處理，不刪除 `tasks.yaml` 中的任務。

10. 告知使用者可用指令：`zmx attach cx-<name>`（即時檢視）、`zmx history cx-<name> | tail -20`（近期輸出）、`zmx list`（所有工作階段）、`Ctrl+\`（脫離 attach 不終止 Codex）。

### status

1. 讀取 `tasks.yaml` 和 `results.yaml`（遺失、空字串、`[]` 均視為空）。
2. 摘要報告：待處理任務、已完成/失敗/部分完成結果。
3. 執行 `zmx list`，顯示 `cx-` 工作階段狀態。

### clean

重設 `results.yaml` 為 `[]`。不修改 `tasks.yaml`。

## 注意事項

- 必須使用 `zmx run -d`（detached）。沒有 `-d` 時 zmx 阻塞呼叫端，呼叫端退出時工作階段被終止。
- 必須用 `bash -c '...'` 包裹 codex 指令。zmx 直接傳遞引數，不加 `bash -c` 時帶旗標的指令被視為單一執行檔名。
- 使用 `codex --dangerously-bypass-approvals-and-sandbox`（非 `codex exec`）。
- 不要從 `zmx list`、`created=...`、`date +%s` 或任何 zmx 時間戳計算經過時間。`created_at`/`completed_at` 僅為 YAML 中繼資料。
- 不要在 Monitor 之外額外執行 `zmx history` 檢查。Monitor 使用 `sleep 30`。
- 佇列檔案規則參見 `cowork-runner/references/yaml-schema.md`。
