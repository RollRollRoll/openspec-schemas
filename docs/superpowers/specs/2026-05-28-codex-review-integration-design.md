# Codex review integration — design spec

> Validated design from brainstorming session, 2026-05-28.
> Adds cross-model review gates at every artifact and apply-phase task.
> Source idea: 在現有 superpowers-bridge 流程中,每一步生成都缺少跨模型 review,
> 使用 `/codex:review` 與 `/codex:adversarial-review` 補上。

---

## Context

`superpowers-bridge` v1 既有流程:

```
brainstorm → proposal → (design) → specs → tasks → plan
   → apply (worktree + subagent-driven-development)
   → verify → retrospective → archive → PR
```

**現有 review 覆蓋分析**:

| 階段 | 既有 review | 模型 | 覆蓋盲點 |
|------|----------|------|--------|
| brainstorm | ❌ 無 | — | 設計假設未被挑戰 |
| proposal | ❌ 無 | — | capability 邊界自我證明 |
| design | ❌ 無 | — | 技術決策無對立視角 |
| specs | ❌ 無 | — | scenario 完整性、可測試性 |
| tasks | ❌ 無 | — | 任務粒度、依賴順序 |
| plan | ❌ 無 | — | micro-step 完整性 |
| apply (per task) | ✅ `superpowers:requesting-code-review` | Claude (同模型) | 同模型盲點 |
| apply (final) | ❌ 無 | — | 跨 task 整合風險 |

**核心問題**: 整個流程內所有 review 都由同一模型(Claude)執行,缺少跨模型挑戰,
盲點覆蓋率低。Codex CLI 已透過 `openai-codex` plugin 提供 `/codex:review` 與
`/codex:adversarial-review` 兩個命令,可作為跨模型 review gate。

## Goals

- 在 artifact 生成階段(brainstorm / proposal / design / specs / tasks / plan)各加一道
  Codex `adversarial-review` gate,挑戰設計假設、scope、矛盾
- 在 apply 階段每個 coarse task 完成後加一道 Codex `review` gate,檢查實作正確性
- 在所有 task 完成後加一道 Codex `adversarial-review` gate,挑戰跨 task 整合風險
- review gate 採 hard gate 語意:BLOCK 必須修正才能繼續
- 自動修正失敗超過 2 次 → 升級給人類決定,不無限重試
- 所有 review 結果寫入 `review-log.md`,與其他 artifact 同層保留審計 trail
- 不破壞 PR #970 review 已應對的三大顧慮(capability detection / verify 時序 / 不主動 commit)

## Non-Goals

- 替換 `superpowers:requesting-code-review`。Codex review 與 Superpowers review 並存,
  跨模型互為盲點覆蓋
- 對 verify、retrospective 兩個 artifact 加 review gate(它們本身就是檢查/分析,
  再 review 是套娃)
- 動態 / 條件式 review(如「只 review 大 artifact」)。v1 一律全跑,簡單明確
- 修改 Superpowers 任何 skill 本身。所有變更都在 `superpowers-bridge/schema.yaml` 與
  CLAUDE.md 內

## Decisions

### D1: review 步驟「內聯」進每個 instruction;頂層 review_protocol 只作參考預設值

**選擇**: 每個 artifact 與 apply.instruction 的 review 步驟以**完整可執行文字**內聯
在該 instruction 末尾(POSTCHECK 段)。schema.yaml 頂層**仍可選擇性**定義
`review_protocol` 段,但只作為**參考預設值**(retry 上限、focus template 等),
**不**作為 hard gate 的執行依據。

**為何**: OpenSpec runtime **不會解析未知頂層欄位**,既有 PRECHECK 全部都是
直接內聯在 instruction 文字中,因為這才是模型實際讀到並執行的位置。把 hard gate
邏輯放在 OpenSpec 會忽略的頂層欄位 + 由 instruction 「reference」,等於把可驗證
的 gate 降級為「靠模型自己讀懂頂層欄位」的軟約定。對齊既有 schema 的 PRECHECK
慣例,review POSTCHECK 必須**內聯**才能真正成為 hard gate。

**Trade-off 接受**: review 步驟文字會在 6 個 artifact instruction + apply.instruction
重複出現。這是**可執行性 > DRY** 的有意取捨。修改 review 策略時需要改 7 處,但
每一處都會被 OpenSpec runtime 注入到模型上下文,不會被靜默忽略。

**對立方案考慮**:
- *方案 A(原 D1)*: review 邏輯放頂層 `review_protocol`,instruction 只 reference
  → Codex adversarial review 指出:OpenSpec validate 忽略頂層欄位 → hard gate
  名不副實。**否決**。
- *方案 C*: 每個 artifact 後新增獨立 review artifact 節點 → artifact 數量翻倍,
  目錄雜亂,違反 OpenSpec artifact 模型。**否決**。

**驗證**: Migration Plan §1 會在 schema 改完後實作驗證:
`grep -c "POSTCHECK — Codex review gate" superpowers-bridge/schema.yaml` 必須 ≥ 7
(6 個 artifact + 1 個 apply)。CI 加一條檢查確認 hard gate 邏輯確實內聯。

### D2: 分層 review 類型

**選擇**:
- artifact 階段 → `/codex:adversarial-review`(挑戰設計選擇、假設、範圍)
- apply 階段 per-task → `/codex:review`(標準 review,檢查實作)
- apply 階段 final → `/codex:adversarial-review`(挑戰跨 task 整合風險)

**為何**: artifact 是設計決策,需要對抗性挑戰;per-task 是實作,需要正確性檢查;
final 跨 task 又回到設計層問題(整合、coherence),用對抗性。

### D3: Hard gate + 自動修正最多 2 次 → 升級

**選擇**: BLOCK 結果觸發自動修正循環,retry_count 上限 2,超過則停下交人類三選一
(看完整 log 自己改 / 強制通過 + 記 override 原因 / 中止 cycle)。

**為何**:
- 0 次 retry → 等同 soft gate,不符合「Hard gate」需求
- 1 次 retry → 第一輪修正常常只解決部分 findings,第二輪才穩
- 2 次 retry → 給兩次機會(精準修正 + 補漏),失敗就升級
- 3+ 次 → 成本急升,實證上效益遞減,且超過 2 次的 finding 通常需重新設計而非修補

### D4: 雙層 PRECHECK + 顯式 opt_out

**選擇**: 在 `review_protocol.precheck` 段定義:
- Layer 1: 檢查 `/codex:review`、`/codex:adversarial-review` 是否在 available
  slash commands 中
- Layer 2: 檢查 Node.js 是否在 PATH(Codex companion 依賴)
缺失 → STOP,不 silent fall back。
使用者可顯式設定 `review_protocol.enabled: false` opt out,opt-out 事實會
記錄到 verify.md。

**為何**: 對齊 PR #970 顧慮 #1 的核心訴求 — 「不靜默降級」。使用者必須主動選擇
放棄 Codex review,且決策被審計。

### D5: review-log.md 為 append-only 審計檔,**每次嘗試都先寫**

**選擇**: 在 `openspec/changes/<name>/review-log.md` 累積每次 review entry,
**每次 review 嘗試一旦取得 outcome 就立即 append entry**(不等到下一階段才補寫)。
結構為:
- header: timestamp, phase, target, attempt(=retry_count), review type, outcome
- findings: file:line + severity + what + recommendation
- action: 該 entry 收尾時做了什麼(`proceed` / `auto-fix retry` / `escalated_to_human`
  / `deferred_findings`)
- summary(檔尾): total reviews, retries, escalations, avg findings — 從 entries
  重算,不暫存中間狀態

**為何**:
- 永不覆寫 → 即使最後 ALLOW,前面 BLOCK + 修正過程仍可追溯
- **「每次嘗試都先寫」**(Codex adversarial review finding 3 修正):原設計只在 ALLOW
  或 STOP 時寫 log,等於 BLOCK→修正→ALLOW 中間的失敗 entry 被丟棄。最有審計價值的
  「哪些 finding 觸發了修補、第幾次成功、修補是否引入新 finding」反而不可追溯。
- verify.md 與 retrospective.md 都讀取此檔做事後分析
- 結構化欄位方便日後做 review 模式統計(常見 finding 類型可提升為 schema 改進)

### D6: review 觸發時機 = artifact 寫入後 / task 完成後 / 下一階段前

**選擇**: review gate 插在「當前單位完成」與「下一單位開始」之間。
- artifact 階段: 寫完 `<artifact>.md` → review → ALLOW → 進下一個 artifact
- apply 階段: **主 agent** 觀察到當前 task 的 subagent 完成 → review → ALLOW
  → 主 agent dispatch 下一個 task 的 subagent(見 D8)
- apply final: 所有 task done → review → ALLOW → 進 verify

**為何**:
- 不打斷 subagent 內部 TDD 循環(避免破壞既有 transitive 行為)
- 在邊界處插 gate,錯誤被早期攔截,不累積到後面 task

### D7: 自動修正不主動 commit

**選擇**: BLOCK 後的修正循環,**全程不主動觸發 git commit**。

- **artifact 階段**: 主 agent 直接 edit `.md` 文件(working tree),**不 commit**。
  artifact 文件最終會在 apply 結束 archive 時與其他 artifact 一起進入 commit
  (這是既有 schema 的行為,本設計不改)。
- **apply 階段**: 主 agent **不**直接 edit source code 並 commit。改為重新 dispatch
  subagent 走 TDD 循環(per-task BLOCK 見 D8;final BLOCK 見 D9)。
  - 這是 Codex adversarial review round 1 finding 1 的修正:原 D7「主 agent 直接
    commit fix-up」與 Goals 第 6 條「不主動 commit」自相矛盾,且繞過 TDD +
    same-model code review 閉環。
- **review-log.md 寫入**: 純 file edit,不觸發 git 操作。最終由 archive 步驟
  與其他 artifact 一起 commit。

### D8: apply 階段主 agent 顯式接管 task loop(取代直接交給 subagent-driven-development)

**選擇**: apply 階段不再直接呼叫 `superpowers:subagent-driven-development` 執行
**整個 plan**。改為主 agent **顯式按 coarse task 一個一個** dispatch:

```
for each coarse task in tasks.md:
  1. 主 agent dispatch 一個新的 subagent 完成 THIS task
     - subagent 內部走 superpowers:test-driven-development(RED-GREEN-REFACTOR)
     - subagent 內部走 superpowers:requesting-code-review(同模型 review)
     - subagent 完成後產生 commits,marks tasks.md `- [x]`
  2. 主 agent run /codex:review --scope working-tree
     focus: 該 task 的 commits + 對應 spec scenario
  3. append review-log entry
  4. 解析 outcome:
     - ALLOW → 進下一個 task
     - BLOCK → 主 agent dispatch SAME task 的新 subagent,把 Codex findings
       作為 input,subagent 重新走 TDD 循環。retry_count += 1
     - retry_count 達 2 仍 BLOCK → STOP 升級給人類
after all tasks done:
  跑 final Codex adversarial review(見 D9)
```

**為何**:
- Codex adversarial review round 2 finding 1 指出:`subagent-driven-development`
  本身是執行**整個 plan** 的 task loop;若把 Codex per-task gate 寫成「追加在
  整個 executor 之後」,等於 gate 只能在所有 task 完成後才 run,task 1 出錯時
  task 2 早就建立在被 BLOCK 的代碼上。**per-task hard gate 名存實亡**。
- 主 agent 顯式接管 task loop 是讓 per-task gate 真正成為 task 間閘門的唯一辦法
  (在不修改 Superpowers skill 本身的前提下)
- 每個 task 仍由 fresh subagent 執行,subagent 仍透過 TDD 與
  requesting-code-review 完成內部驗證,只是 executor 從「Superpowers 跑整個 loop」
  改為「主 agent 跑外層 + Superpowers skills 跑內層」

**Trade-off 接受**:
- 失去 `subagent-driven-development` 整體執行的「自動化便利」,主 agent 需要明確
  按 tasks.md 拆分並逐個 dispatch
- 主 agent 上下文需保留 task 順序與依賴關係,不能讓 token 耗盡(Risks 表新增一條)
- 但換來「per-task hard gate 真正可執行」,值得

**邊界**:
- 如果 subagent 重跑後本身又失敗(連 TDD 都沒走完)→ 不算 Codex retry_count,
  直接升級人類(這是 subagent 機制問題,非 Codex review 問題)
- task 之間如果有並行可能性,v1 一律序列執行(保證 gate 嚴格生效);v1.1
  可考慮獨立 task 並行 + per-task gate 並行檢查

### D9: final gate BLOCK 的修復路徑(獨立於 per-task)

**選擇**: 所有 task 完成後跑 `/codex:adversarial-review --scope branch --base main`
作為 final gate。BLOCK 路徑與 per-task 不同:

```
final review BLOCK
  │
  ▼
┌────────────────────────────────────┐
│ Codex 是否能把 findings 歸因到具體  │
│ task / 文件範圍?                    │
└────┬─────────────────────────┬─────┘
     │                         │
   yes                       no(整體性 finding)
     │                         │
     ▼                         ▼
┌──────────────────────┐  ┌──────────────────────┐
│ 把每個 finding 轉成   │  │ 直接升級人類          │
│ integration-fix task │  │(無從歸因,不嘗試      │
│ append 進 tasks.md    │  │ 自動修補)            │
│ 標記 [integration-fix]│  └──────────────────────┘
└──────┬───────────────┘
       │
       ▼
┌──────────────────────────────┐
│ 主 agent 為每個 integration- │
│ fix task dispatch fresh      │
│ subagent 走 TDD + Superpowers │
│ review                       │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ 重跑 /codex:adversarial-     │
│ review --scope branch        │
│ retry_count += 1             │
└──────┬───────────────────────┘
       │
       ▼
   retry_count < 2?
   ├── yes → 仍 BLOCK 回到上面循環
   └── no  → 升級人類
```

**為何**:
- Codex adversarial review round 2 finding 2 指出:D7 只定義了 per-task BLOCK
  的修復路徑(重新 dispatch **該 task** 的 subagent),但 final review 的 BLOCK
  通常是**跨 task 整合問題**,沒有單一 task 可重派。實作者要麼卡住,要麼
  回退到主 agent 直接改 code(繞開 D7 想保護的 TDD 閉環)。
- 把 findings 轉成 integration-fix task 確保修復仍走 subagent + TDD +
  Superpowers review 閉環,**不**讓主 agent 直接改 source
- 「無從歸因 → 直接升級」這個分支,避免實作者為了套用「轉 task」而強行歸因,
  把整體性的設計問題硬塞進 task 模板裡

**邊界**:
- integration-fix task 寫入 tasks.md 時,前綴標記 `[integration-fix from final review]`,
  retrospective §0 可統計「v1 cycle 中有多少 integration-fix」反推 plan 顆粒度
  問題
- final retry_count 與 per-task retry_count **獨立計數**,不互相累加

## Risks / Trade-offs

| 風險 | 嚴重度 | 緩解 |
|------|------|----|
| Codex CLI 不可用導致全流程卡住 | 🔴 高 | D4 雙層 PRECHECK + 顯式 opt_out |
| 主 agent 接管 task loop 導致主 context token 耗盡(D8)| 🔴 高 | 主 agent 每完成一個 task 就壓縮上下文;只保留 tasks.md 進度與當前 task 的 review 結果;歷史 commits 用 `git log` 重讀,不存記憶體 |
| 11 次以上 Codex 調用導致 token / 額度耗盡 | 🟡 中 | review-log Summary 即時記錄,使用者可中止 |
| apply 階段 BLOCK 重新 dispatch subagent 加重 token / 時間成本 | 🟡 中 | D7+D8 接受此 trade-off 換來「fix-up 與正常 task 同等驗證」;若成本超出可接受值,使用者可走 escalate-to-human 強制通過 |
| final BLOCK 無法歸因時直接升級可能讓使用者卡住(D9)| 🟡 中 | review-log 完整記錄 finding 內容,使用者拿到完整證據可手動拆 task 或選擇 override |
| 自動修正反而引入新 bug | 🟡 中 | D3 max_retries=2 上限,達不到必須升級 |
| review 結果不穩定(同 input 多次結果不同) | 🟡 中 | 在 retrospective §6 累積證據,反覆 false BLOCK 可調 prompt |
| 與 `superpowers:requesting-code-review` 重複 | 📌 低 | 設計上「跨模型 review」是 feature 不是 bug |
| review 階段 Codex 自身有 bug 觸發 retry 死迴圈 | 🟡 中 | review 工具失敗(網路 / CLI 錯誤)不算 retry_count,獨立計入無限重試上限 3 次 |
| review finding 涉及非當前 artifact / task 的舊文件 | 📌 低 | 只在 artifact / per-task gate 記為 deferred,**final gate 不允許 deferral** |
| review 步驟內聯導致 7 處重複文字維護成本 | 📌 低 | D1 接受此 trade-off;改 review 策略需改 7 處,但 CI 加 grep 檢查可確認一致性 |
| 失去 `subagent-driven-development` 的整體執行便利(D8)| 🟡 中 | 主 agent 顯式接管是 per-task hard gate 真正可執行的唯一辦法;trade-off 已知接受 |

## State machine

本設計有**三類 gate**,共用同一套 review-log append 規則,但 BLOCK 修復路徑與
deferred_findings 適用範圍**不同**:

| Gate 類型 | 修復路徑 | deferred_findings 允許? |
|----------|--------|----------------------|
| artifact gate(D6 + D7)| 主 agent 直接 edit `.md` | ✅ 允許(舊 .md 不修) |
| per-task gate(D8)| 重新 dispatch SAME task subagent 走 TDD | ✅ 允許(舊 source 不修) |
| final branch gate(D9)| findings 轉 integration-fix task 重新 dispatch / 無法歸因升級 | ❌ **不允許** |

### 共用流程(任何 gate 都適用)

```
artifact 已生成 / task 已完成 / 所有 task 完成
        │
        ▼
   retry_count = 0
        │
        ▼
  ┌────────────────────────────────────┐
  │ Codex review 執行                   │
  └────────────────┬───────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────┐
  │ 先 append review-log.md entry      │ ← 不論 ALLOW/BLOCK 都先寫
  │ (attempt, outcome, findings, ts)   │
  └────────────────┬───────────────────┘
                   │
                   ▼
   解析 review 結果
   ├── ALLOW ──→ entry.action = "proceed" → 進入下一階段
   └── BLOCK
        │
        ▼
   retry_count < 2 ?
   ├── yes
   │     │
   │     ▼
   │   ┌────────────────────────────────────┐
   │   │ 修正路徑(依 gate 類型分流):         │
   │   │ - artifact gate: 主 agent edit .md │
   │   │ - per-task gate: 重 dispatch SAME  │
   │   │   task subagent (D8)               │
   │   │ - final gate: findings 轉           │
   │   │   integration-fix task → 重新       │
   │   │   dispatch / 無法歸因升級 (D9)      │
   │   └────────────────┬───────────────────┘
   │                    │
   │                    ▼
   │   ┌────────────────────────────────────┐
   │   │ 在當前 entry 末尾 append           │
   │   │ "action taken: <修正描述>"         │
   │   └────────────────┬───────────────────┘
   │                    │
   │                    ▼
   │            retry_count += 1
   │                    │
   │                    └──→ 回到「Codex review 執行」
   │
   └── no  → STOP → entry.action = "escalated_to_human"
                  → 升級給人類
                    ┌──────────────────────────────┐
                    │ [a] 看完整 log 自己改          │
                    │ [b] 強制通過 + 記 override 原因 │
                    │ [c] 中止 cycle                │
                    └──────────────────────────────┘
```

**Append-log 不變條件**(來自 Codex adversarial review round 1 finding 3):

- 每次 review 嘗試**都先 append entry**,再決定下一步動作
- entry 至少包含 `attempt`、`timestamp`、`outcome`、`findings`、`gate_type`
  (artifact / per-task / final)、`action`
- 不論最終 ALLOW、retry、escalate,完整失敗鏈都被保留
- summary(檔尾)從 entries 重新計算,不依賴中間狀態的暫存值

**邊界規則**:

通用:
- review 工具本身失敗(網路 / Codex CLI 錯誤)→ 不算 retry_count,先 append
  一個 `outcome: tool_error` 的 entry,指數退避 3 次後升級
- Codex 回傳非 ALLOW/BLOCK 開頭 → 視為 BLOCK(保守策略),append entry 時標
  `outcome: block_inferred`
- retry 第二次的 fix 引入新 finding → 仍計入 retry_count(避免無限迴圈)
- per-task / integration-fix 重新 dispatch subagent 後本身又失敗 → 不算 Codex
  retry_count,直接升級

deferred_findings 適用範圍(來自 Codex adversarial review round 2 finding 3):
- **artifact gate**: finding 涉及非當前 artifact 的舊 `.md` 文件 → 不修正,
  在當前 entry 標 `deferred_findings`,繼續推進
- **per-task gate**: finding 涉及非當前 task 範圍的舊 source 文件 → 不修正,
  在當前 entry 標 `deferred_findings`,繼續推進
- **final branch gate**: ❌ **不允許 deferred_findings**。final review 的 scope
  本來就是整個 branch,跨 task 與共享文件的問題正是 final gate 要攔的對象;
  套用 deferral 規則等於把 hard gate 旁路。所有 material finding 必須走 D9
  修復或升級 override,**沒有第三條路**

## Migration Plan

**v1 (本設計)**:

1. 在 `superpowers-bridge/schema.yaml` 每個 artifact 的 `instruction` 末尾**完整內聯**
   POSTCHECK — Codex review gate 段(D1)
   - 涵蓋: brainstorm、proposal、design、specs、tasks、plan(共 6 個)
   - 不涵蓋: verify、retrospective(本身即為檢查/分析)
   - 內聯文字必須包含:Layer 1+2 PRECHECK、`/codex:adversarial-review` 呼叫、
     ALLOW/BLOCK 解析、retry 上限 2、escalation 三選項、append review-log 規則
2. (可選)在 `schema.yaml` 頂層新增 `review_protocol` 段作為**參考預設值**
   (retry 上限、focus template 等)。此段**不**作為 hard gate 執行依據,
   只是供未來文件 / 工具讀取的中央化定義。如果這段價值不大,可在實作時直接省略。
3. **重寫 `apply.instruction`**:**主 agent 顯式接管 task loop**(D8),不再
   直接呼叫 `superpowers:subagent-driven-development` 跑整個 plan。
   改為內聯描述以下流程:
   ```
   for each coarse task in tasks.md:
     a. 主 agent dispatch 一個 fresh subagent 完成 THIS task
        (subagent 內部走 superpowers:test-driven-development +
         superpowers:requesting-code-review)
     b. /codex:review --scope working-tree (per-task gate)
     c. append review-log entry
     d. ALLOW → 進下一個 task / BLOCK → retry up to 2 → 升級
   after all tasks done:
     /codex:adversarial-review --scope branch --base main (final gate)
     ALLOW → 進 verify
     BLOCK → D9 修復路徑(integration-fix task / 升級)
   ```
   這段 instruction 必須完整內聯,**不**只 reference 頂層 `review_protocol`
   (D1)。Migration §10 的 CI grep 檢查包含 `apply.instruction` 在內。
4. 新增 `review-log.md` 模板(`templates/review-log.md`),append-only 結構,
   每次 review 嘗試先寫 entry,再決定下一動作(D5)
5. 在 verify.md 模板「Implementation signal」段追加 5b:Codex review trail integrity
6. 在 retrospective.md 模板 §0 Evidence 追加: Codex review stats 一行
7. 更新 `superpowers-bridge/README.md`「六個值得記住的設計觸點」段,新增 #7
   Codex review gate 章節(說明為何選擇內聯而非頂層引用)
8. 更新 CLAUDE.md「修 schema 的紅旗」段,新增第五、第六、第七條:
   - 不可 silent fallback(缺 Codex CLI 偷偷降級)
   - 不可把 review hard gate 邏輯只放在頂層 `review_protocol`,
     必須內聯到 instruction
   - 不可在 apply.instruction 直接呼叫 `subagent-driven-development` 跑整個 plan
     而把 Codex per-task gate 寫成「之後追加」(這會讓 gate 形同虛設,
     見本 spec D8)
9. 同步更新 README.zh-TW.md 與其他繁中翻譯檔
10. **CI 驗證**: 在 `.github/workflows/validate-schemas.yml` 加一條:
    ```bash
    test "$(grep -c 'POSTCHECK — Codex review gate' superpowers-bridge/schema.yaml)" -ge 7
    ```
    確保 6 個 artifact + 1 個 apply.instruction 都內聯了 review gate,防止
    後續修改時意外移除某處(這是 hard gate 名實相符的最後一道防線)。

**Rollback strategy**: 整個變更是 schema 增量,移除每個 instruction 的
POSTCHECK 段 + 移除 review-log 模板 + 移除 CI 驗證即可回到 v1 行為。

**Backward compatibility**: 已採用 v1 schema 的既有 cycle 不受影響(沒有 POSTCHECK
內聯文字等同 enabled=false)。但會被 CI grep 檢查標出,提醒升級。

## Open Questions

1. **Codex CLI 版本要求**: 是否需要在 PRECHECK 加版本檢查(如 `codex --version >= X.Y`)?
   v1 暫不加,等出現相容性問題再補。
2. **review-log.md 是否進入 PR diff**: 預設進。如果使用者覺得 noise 過大,
   可在 v1.1 提供 `.gitignore` 模板選項。
3. **第三方 bridge 是否強制使用 review_protocol**: v1 只在 superpowers-bridge 引入。
   未來新增 bridge 時,review_protocol 設為 opt-in 而非 schema 級強制。
4. **跨 cycle 的 review pattern 累積**: 反覆出現的 false BLOCK 在 retrospective
   §6 提升,但目前沒有跨 cycle 的彙總機制。v1.1 可考慮在 repo 層加
   `docs/review-patterns.md` 累積。
