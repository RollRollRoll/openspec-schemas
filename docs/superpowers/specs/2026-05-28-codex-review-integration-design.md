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

### D1: review_protocol 集中定義在 schema.yaml 頂層

**選擇**: 在 `schema.yaml` 新增頂層 `review_protocol` 欄位,artifact_phase / apply_phase
兩個子段各自定義 review type、gate、retry 策略、focus template。每個 artifact 的
`instruction` 末尾統一 reference 同一個 protocol。

**為何**: 避免 review 邏輯散落在 7 個 artifact instruction 中,改一處全局生效。
OpenSpec validate 會忽略未知頂層欄位,不影響 schema 合法性。

**對立方案考慮**:
- *方案 A*: 每個 artifact 的 instruction 內聯 review 邏輯 → 重複嚴重,改 review 策略
  要改 7 處
- *方案 C*: 每個 artifact 後新增獨立 review artifact 節點 → artifact 數量翻倍,
  目錄雜亂,違反 OpenSpec artifact 模型

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

### D5: review-log.md 為 append-only 審計檔

**選擇**: 在 `openspec/changes/<name>/review-log.md` 累積每次 review entry,結構為:
- header: timestamp, phase, target, review type, outcome, findings count
- findings: file:line + severity + what + recommendation
- action taken: 修正了哪些行
- summary(檔尾): total reviews, retries, escalations, avg findings

**為何**:
- 永不覆寫 → 即使最後 ALLOW,前面 BLOCK + 修正過程仍可追溯
- verify.md 與 retrospective.md 都讀取此檔做事後分析
- 結構化欄位方便日後做 review 模式統計(常見 finding 類型可提升為 schema 改進)

### D6: review 觸發時機 = artifact 寫入後 / task 完成後 / 下一階段前

**選擇**: review gate 插在「當前單位完成」與「下一單位開始」之間。
- artifact 階段: 寫完 `<artifact>.md` → review → ALLOW → 進下一個 artifact
- apply 階段: subagent 標記 task done → review → ALLOW → 進下一個 task
- apply final: 所有 task done → review → ALLOW → 進 verify

**為何**:
- 不打斷 subagent 內部 TDD 循環(避免破壞既有 transitive 行為)
- 在邊界處插 gate,錯誤被早期攔截,不累積到後面 task

### D7: 自動修正由主 agent (Claude) 執行,非重新 dispatch subagent

**選擇**: BLOCK 後的修正循環,artifact 階段直接 edit `.md` 文件;apply 階段
直接 edit source code 並產生新的 fix-up commit(不 amend)。

**為何**:
- subagent 已完成,retry 修正屬於「補丁」性質,不需重新 dispatch
- 不 amend 符合 git_safety 與既有 schema 規範(「Prefer new commits over --amend」)

## Risks / Trade-offs

| 風險 | 嚴重度 | 緩解 |
|------|------|----|
| Codex CLI 不可用導致全流程卡住 | 🔴 高 | D4 雙層 PRECHECK + 顯式 opt_out |
| 11 次以上 Codex 調用導致 token / 額度耗盡 | 🟡 中 | review-log Summary 即時記錄,使用者可中止 |
| 自動修正反而引入新 bug | 🟡 中 | D3 max_retries=2 上限,達不到必須升級 |
| review 結果不穩定(同 input 多次結果不同) | 🟡 中 | 在 retrospective §6 累積證據,反覆 false BLOCK 可調 prompt |
| 與 `superpowers:requesting-code-review` 重複 | 📌 低 | 設計上「跨模型 review」是 feature 不是 bug |
| review 階段 Codex 自身有 bug 觸發 retry 死迴圈 | 🟡 中 | review 工具失敗(網路 / CLI 錯誤)不算 retry_count,獨立計入無限重試上限 3 次 |
| review finding 涉及非當前 artifact 的舊文件 | 📌 低 | 只記入 review-log 為 deferred,**不**修正,避免 review scope 失控 |

## State machine

```
artifact 已生成 / task 已完成
        │
        ▼
   retry_count = 0
   Codex review 執行
        │
        ▼
   解析 review 結果
   ├── ALLOW ──→ 寫 review-log.md → 進入下一階段
   └── BLOCK
        │
        ▼
   retry_count < 2 ?
   ├── yes → 自動修正 + retry_count += 1 → 重新 review
   └── no  → STOP → 寫 review-log.md → 升級給人類
                    ┌──────────────────────────────┐
                    │ [a] 看完整 log 自己改          │
                    │ [b] 強制通過 + 記 override 原因 │
                    │ [c] 中止 cycle                │
                    └──────────────────────────────┘
```

**邊界規則**:
- review 工具本身失敗(網路 / Codex CLI 錯誤)→ 不算 retry_count,指數退避 3 次後升級
- Codex 回傳非 ALLOW/BLOCK 開頭 → 視為 BLOCK(保守策略)
- retry 第二次的 fix 引入新 finding → 仍計入 retry_count(避免無限迴圈)
- finding 涉及非當前 artifact / task 的舊文件 → 不修正,僅 review-log 記錄為 deferred

## Migration Plan

**v1 (本設計)**:

1. 在 `superpowers-bridge/schema.yaml` 新增頂層 `review_protocol` 段(D1)
2. 在每個 artifact instruction 末尾追加 POSTCHECK 段,引用 review_protocol.artifact_phase
   - 涵蓋: brainstorm、proposal、design、specs、tasks、plan(共 6 個)
   - 不涵蓋: verify、retrospective(本身即為檢查/分析)
3. 在 `apply.instruction` 步驟 2「subagent-driven-development」之後追加:
   - 2a: 每個 coarse task 完成後 → `/codex:review`(per-task gate)
   - 2.5: 所有 task 完成後 → `/codex:adversarial-review --scope branch`(final gate)
4. 新增 `review-log.md` 模板(`templates/review-log.md`),append-only 結構
5. 在 verify.md 模板「Implementation signal」段追加 5b:Codex review trail integrity
6. 在 retrospective.md 模板 §0 Evidence 追加: Codex review stats 一行
7. 更新 `superpowers-bridge/README.md`「六個值得記住的設計觸點」段,新增 #7
   Codex review gate 章節
8. 更新 CLAUDE.md「修 schema 的紅旗」段,新增第五條:不可 silent fallback
9. 同步更新 README.zh-TW.md 與其他繁中翻譯檔

**Rollback strategy**: 整個 review_protocol 是 schema 增量,移除頂層
`review_protocol` 段 + 還原各 artifact instruction 的 POSTCHECK 即可回到 v1 行為。

**Backward compatibility**: 已採用 v1 schema 的既有 cycle 不受影響(沒有
review_protocol 欄位等同 enabled=false 的 opt out 路徑,但會被 PRECHECK 提示)。

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
