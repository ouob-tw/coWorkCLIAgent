# 監控逾時機制重新設計

## 問題

目前 zmx 監控使用固定的 `monitor.timeout`（預設 3600000ms = 1 小時）控制 Monitor 工具的執行時長。這個做法過於武斷：

- 簡單任務可能 5 分鐘完成，1 小時太長
- 複雜任務可能跑 3 小時，1 小時太短
- timeout 到期只是殺掉 Monitor 腳本，zmx session 繼續跑，dispatch agent 失去自動通知能力
- 需要 agent 手動檢查 `zmx list` 和 `results.yaml`，容易遺漏

## 現行設計

```yaml
# .cowork/config.yaml
monitor:
  enabled: true
  interval: 300        # 輪詢間隔（秒）
  timeout: 3600000     # Monitor 逾時（毫秒）— 問題所在
```

監控腳本本身已有兩個正確的終止條件（DONE / ALERT），timeout 是額外強加的硬性上限。

## 候選方案

| 方案 | 做法 | 優點 | 缺點 |
|------|------|------|------|
| A. 去掉 timeout | 不設 timeout 或設極大值，純靠 DONE/ALERT break | 簡單，不會遺漏 | Monitor 資源一直佔著 |
| B. 停滯偵測 | 監控腳本加 `zmx history` 比對，連續 N 次無新輸出報 STALL | 能偵測卡住的任務 | 實作較複雜，需記錄上次輸出 |
| C. 事件驅動 | `inotifywait -e modify results.yaml` 取代 polling | 零延遲、零浪費 | 依賴 inotify-tools，runner 寫入時機不確定 |
| D. 分段監控 | 前 10 分鐘 60s 間隔，之後 300s，無硬性 timeout | 前期快速回饋，後期不浪費 | 仍需某種上限或手動結束 |

## 待決定

- 選定方案或組合（例如 A+B：無 timeout 但加停滯偵測）
- 確認 Monitor 工具是否支援不設 timeout（或極大值的行為）
- 停滯偵測的閾值（幾次 heartbeat 無變化算 STALL）
