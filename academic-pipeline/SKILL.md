---
name: academic-pipeline
description: "Orchestrator for the full academic research pipeline: research -> write -> review -> revise -> finalize. Coordinates deep-research, academic-paper, and academic-paper-reviewer into a seamless 5-stage workflow with state tracking, mid-entry support, and revision loop management. Triggers on: academic pipeline, 學術研究流程, research to paper, 論文 pipeline, 從研究到論文, full paper workflow, 完整論文流程, paper pipeline, 幫我做一篇論文, 從頭到尾寫一篇論文."
metadata:
  version: "1.0"
  last_updated: "2026-02"
  depends_on: "deep-research, academic-paper, academic-paper-reviewer"
---

# Academic Pipeline -- 學術研究全流程調度器

輕量級 orchestrator，管理從研究探索到論文完稿的完整學術 pipeline。不做實質工作，只負責偵測階段、推薦模式、調度 skill、管理轉場和追蹤狀態。

## Quick Start

**完整流程（從零開始）：**
```
我想做一篇關於 AI 對高教品保影響的研究論文
```
--> academic-pipeline 啟動，從 Stage 1 (RESEARCH) 開始

**中途進入（已有論文）：**
```
我已經有一篇論文，幫我審查
```
--> academic-pipeline 偵測 mid-entry，從 Stage 3 (REVIEW) 開始

**修訂模式（收到審稿意見）：**
```
我收到審稿意見了，幫我修改
```
--> academic-pipeline 偵測，從 Stage 4 (REVISE) 開始

**執行結果：**
1. 偵測使用者目前階段與已有材料
2. 推薦每個 stage 的最適 mode
3. 逐 stage 調度對應 skill
4. 每個 stage 完成後主動提示下一步
5. 全程追蹤進度，隨時可查看 Pipeline Status Dashboard

---

## Trigger Conditions

### 觸發關鍵詞

**中文**：學術研究流程, 論文 pipeline, 從研究到論文, 完整論文流程, 幫我做一篇論文, 從頭到尾寫一篇論文, 研究到出版, 全流程論文
**English**：academic pipeline, research to paper, full paper workflow, paper pipeline, end-to-end paper, research-to-publication, complete paper workflow

### 不觸發情境

| 情境 | 應使用的 Skill |
|------|---------------|
| 只需要查資料、做文獻回顧 | `deep-research` |
| 只需要寫論文（不需研究階段） | `academic-paper` |
| 只需要審查一篇論文 | `academic-paper-reviewer` |
| 只需要檢查引用格式 | `academic-paper` (citation-check mode) |
| 只需要轉換論文格式 | `academic-paper` (format-convert mode) |
| 查詢台灣高教數據 | `tw-hei-intelligence` |

### Trigger Exclusions

- 如果使用者只需要單一功能（只查資料、只檢查引用），不需要 pipeline，直接觸發對應的 skill
- 如果使用者已經在使用某個 skill 的特定 mode，不要強制進入 pipeline
- pipeline 是可選的，不是必要的

---

## Pipeline Stages (5+1)

| Stage | 名稱 | 呼叫的 Skill | 可用 Modes | 產出物 |
|-------|------|-------------|-----------|--------|
| 1 | RESEARCH | `deep-research` | socratic, full, quick | RQ Brief, Methodology, Bibliography, Synthesis |
| 2 | WRITE | `academic-paper` | plan, full | Paper Draft |
| 3 | REVIEW | `academic-paper-reviewer` | full, guided, quick | Review Reports, Editorial Decision |
| 4 | REVISE | `academic-paper` | revision | Revised Draft, Response to Reviewers |
| 3' | RE-REVIEW | `academic-paper-reviewer` | full | Final Review, Accept/Reject |
| 5 | FINALIZE | `academic-paper` | format-convert | Final Paper (LaTeX/DOCX/PDF) |

---

## Pipeline State Machine

```
[START]
  |
  v
Stage 1 (RESEARCH)
  |
  +-- [材料齊備？]
  |     +-- Yes --> Stage 2 (WRITE)
  |     +-- No  --> 繼續 Stage 1
  |
  v
Stage 2 (WRITE)
  |
  +-- [論文完成？]
  |     +-- Yes --> Stage 3 (REVIEW)
  |     +-- No  --> 繼續 Stage 2
  |
  v
Stage 3 (REVIEW)
  |
  +-- [Editorial Decision]
  |     +-- Accept         --> Stage 5 (FINALIZE)
  |     +-- Minor Revision --> Stage 4 (REVISE, 標記 minor)
  |     +-- Major Revision --> Stage 4 (REVISE, 標記 major)
  |     +-- Reject         --> [使用者選擇]
  |           +-- 重大修改  --> Stage 2 (WRITE, 重構)
  |           +-- 放棄      --> [END]
  |
  v
Stage 4 (REVISE)
  |
  +-- [修訂完成？]
  |     +-- Yes --> Stage 3' (RE-REVIEW)
  |     +-- No  --> 繼續 Stage 4
  |
  v
Stage 3' (RE-REVIEW, loop_count++)
  |
  +-- [loop_count > 2?]
  |     +-- Yes --> Stage 5 (FINALIZE, 附警告)
  |     +-- No  --> [Decision]
  |           +-- Accept / Minor --> Stage 5 (FINALIZE)
  |           +-- Major          --> Stage 4 (REVISE)
  |
  v
Stage 5 (FINALIZE)
  |
  +-- [格式轉換完成？]
  |     +-- Yes --> [END, 輸出最終論文]
  |
  v
[END]
```

完整狀態轉換定義見 `references/pipeline_state_machine.md`。

---

## Agent Team (2 Agents)

| # | Agent | 角色 | 檔案 |
|---|-------|------|------|
| 1 | `pipeline_orchestrator_agent` | 主調度器：偵測階段、建議 mode、觸發 skill、管理轉場 | `agents/pipeline_orchestrator_agent.md` |
| 2 | `state_tracker_agent` | 狀態追蹤器：記錄已完成階段、已產出材料、revision 循環次數 | `agents/state_tracker_agent.md` |

---

## Orchestrator Workflow

### Step 1: INTAKE & DETECTION

```
pipeline_orchestrator_agent 分析使用者的輸入：

1. 使用者有什麼材料？
   - 無材料           --> Stage 1 (RESEARCH)
   - 有研究資料       --> Stage 2 (WRITE)
   - 有論文草稿       --> Stage 3 (REVIEW)
   - 有審查意見       --> Stage 4 (REVISE)
   - 有修訂稿         --> Stage 3' (RE-REVIEW)
   - 有最終稿要轉格式 --> Stage 5 (FINALIZE)

2. 使用者的目標？
   - 完整流程（從研究到出版）
   - 部分流程（只需要某幾個 stage）

3. 判斷進入點，向使用者確認
```

### Step 2: MODE RECOMMENDATION

```
根據進入點和使用者偏好，推薦每個 stage 的 mode：

使用者類型判斷：
- 新手 / 想被引導 --> socratic (Stage 1) + plan (Stage 2) + guided (Stage 3)
- 老手 / 要直接產出 --> full (Stage 1) + full (Stage 2) + full (Stage 3)
- 時間有限 --> quick (Stage 1) + full (Stage 2) + quick (Stage 3)

推薦時說明每個 mode 的差異，讓使用者選擇
```

### Step 3: STAGE EXECUTION

```
呼叫對應的 skill（不自己做事，純粹調度）：

1. 告知使用者即將進入哪個 Stage
2. 載入對應 skill 的 SKILL.md
3. 以推薦的 mode 啟動 skill
4. 監控 stage 完成狀態

完成後：
1. 彙整產出物清單
2. 更新 pipeline state（呼叫 state_tracker_agent）
3. 主動提示：「Stage X 完成，產出了 [材料清單]。下一步是 Stage Y，要繼續嗎？」
```

### Step 4: TRANSITION

```
使用者確認後：

1. 將上一 stage 的產出作為下一 stage 的輸入
2. 觸發 handoff protocol（已定義在各 skill 的 SKILL.md 中）：
   - Stage 1 --> 2：deep-research handoff（RQ Brief + Bibliography + Synthesis）
   - Stage 2 --> 3：將完整論文交給 reviewer
   - Stage 3 --> 4：將 Revision Roadmap 交給 academic-paper revision mode
   - Stage 4 --> 3'：將修訂稿和 Response to Reviewers 交給 reviewer
   - Stage 3'/5：將最終稿交給 format-convert mode
3. 開始下一 stage
```

---

## Mid-Entry Protocol

使用者可以從任何 stage 進入。orchestrator 會：

1. **偵測材料**：分析使用者提供的內容，判斷已有什麼
2. **確認缺口**：檢查進入該 stage 需要什麼前置材料
3. **建議補做**：如果缺少關鍵材料，建議是否需要回到前面的 stage
4. **直接進入**：如果材料足夠，直接開始指定 stage

**範例：**
```
使用者：我已經寫好一篇論文，想審查一下

orchestrator：
  偵測到你有完整論文，可以直接進入 Stage 3 (REVIEW)。

  在開始之前，確認一下：
  1. 論文主題/領域是什麼？
  2. 你想要完整審查 (full) 還是引導式審查 (guided)？
  3. 論文是中文還是英文？

  如果你之後想根據審查意見修改，我會自動引導你進入 Stage 4。
```

---

## Progress Dashboard

使用者隨時可以說「進度」「status」「pipeline 狀態」查看：

```
+=========================================+
|   Academic Pipeline Status              |
+=========================================+
| Topic: AI 對高等教育品質保證的影響       |
+-----------------------------------------+

  Stage 1 RESEARCH    [v] Completed
    Mode: socratic
    Outputs: RQ Brief, Methodology,
             Bibliography (22 sources),
             Synthesis

  Stage 2 WRITE       [v] Completed
    Mode: plan -> full
    Outputs: Paper Draft
             (5,200 words, IMRaD)

  Stage 3 REVIEW      [v] Completed
    Mode: full
    Decision: Major Revision
    Required Revisions: 5 items

  Stage 4 REVISE      [..] In Progress
    Revision Round: 1/2
    Addressed: 3/5 required revisions

  Stage 3' RE-REVIEW  [ ] Pending
  Stage 5 FINALIZE    [ ] Pending

+-----------------------------------------+
| Materials:                              |
|   [v] RQ Brief                          |
|   [v] Methodology Blueprint             |
|   [v] Bibliography                      |
|   [v] Synthesis Report                  |
|   [v] Paper Draft                       |
|   [v] Review Reports                    |
|   [v] Revision Roadmap                  |
|   [ ] Revised Draft                     |
|   [ ] Response to Reviewers             |
|   [ ] Final Paper                       |
+=========================================+
```

輸出模板見 `templates/pipeline_status_template.md`。

---

## Revision Loop Management

- 最多 **2 輪** review-revise 循環（Stage 3/4 --> Stage 3'/4）
- 每輪追蹤：哪些審查意見已處理、哪些未處理
- 第 2 輪後仍有 major issues：
  - 提醒使用者「已達最大修訂輪數」
  - 建議直接進入 Stage 5 (FINALIZE)
  - 將未解決問題標記為 Acknowledged Limitations
  - 不阻擋 finalize
- 提供累計的 revision history（每輪的 decision、處理項目數、未處理項目）

---

## Quality Standards

| 維度 | 要求 |
|------|------|
| 階段偵測 | 正確識別使用者目前所在階段和已有材料 |
| Mode 推薦 | 根據使用者偏好和材料狀態推薦合適的 mode |
| 材料傳遞 | Stage 間的 handoff 材料完整、格式正確 |
| 狀態追蹤 | Pipeline state 即時更新、Progress Dashboard 準確 |
| 不越權 | orchestrator 不做實質研究/寫作/審查，只做調度 |
| 不強制 | 使用者可以隨時跳過某個 stage 或退出 pipeline |

---

## Error Recovery

| 階段 | 錯誤 | 處理 |
|------|------|------|
| Intake | 無法判斷進入點 | 詢問使用者已有什麼材料和目標 |
| Stage 1 | deep-research 未收斂 | 建議切換 mode（socratic --> full）或縮小範圍 |
| Stage 2 | 缺少研究基礎 | 建議回到 Stage 1 補做研究 |
| Stage 3 | Review 結果為 Reject | 提供選項：重大重構 (Stage 2) 或放棄 |
| Stage 4 | 修訂未完成所有 items | 列出未處理項目，詢問是否繼續 |
| Stage 3' | 超過 2 輪修訂 | 提醒並建議直接 finalize |
| Any | 使用者中途離開 | 儲存 pipeline state，下次可從斷點續行 |
| Any | Skill 執行失敗 | 報告錯誤，建議重試或跳過 |

---

## Agent File References

| Agent | Definition File |
|-------|----------------|
| pipeline_orchestrator_agent | `agents/pipeline_orchestrator_agent.md` |
| state_tracker_agent | `agents/state_tracker_agent.md` |

---

## Reference Files

| Reference | Purpose |
|-----------|---------|
| `references/pipeline_state_machine.md` | 完整狀態機定義：所有合法轉換、前置條件、動作 |

---

## Templates

| Template | Purpose |
|----------|---------|
| `templates/pipeline_status_template.md` | Progress Dashboard 輸出模板 |

---

## Examples

| Example | Demonstrates |
|---------|-------------|
| `examples/full_pipeline_example.md` | 完整 pipeline 對話紀錄（Stage 1-5，含 revision loop） |
| `examples/mid_entry_example.md` | 從 Stage 3 中途進入的範例（已有論文 --> 審查 --> 修訂 --> 完稿） |

---

## Output Language

- 跟隨使用者語言（中文輸入用中文回應，英文輸入用英文回應）
- Pipeline Status Dashboard 固定使用使用者語言
- 學術術語保留英文（如 IMRaD, APA 7.0, peer review 等）

---

## Integration with Other Skills

```
academic-pipeline 調度以下 skills（不自己做事）：

Stage 1: deep-research
  - socratic mode: 引導式研究探索
  - full mode: 完整研究報告
  - quick mode: 快速研究摘要

Stage 2: academic-paper
  - plan mode: 蘇格拉底式逐章引導
  - full mode: 完整論文撰寫

Stage 3/3': academic-paper-reviewer
  - full mode: 完整 4 人審查
  - guided mode: 蘇格拉底式引導審查
  - quick mode: EIC 快速評估

Stage 4: academic-paper (revision mode)
Stage 5: academic-paper (format-convert mode)
```

---

## Related Skills

| Skill | 關係 |
|-------|------|
| `deep-research` | 被調度（Stage 1 研究階段） |
| `academic-paper` | 被調度（Stage 2 撰寫、Stage 4 修訂、Stage 5 格式化） |
| `academic-paper-reviewer` | 被調度（Stage 3/3' 審查階段） |

---

## Version Info

| 項目 | 內容 |
|------|------|
| Skill 版本 | 1.0 |
| 最後更新 | 2026-02 |
| 維護者 | HEEACT |
| 相依 Skills | deep-research v2.0+, academic-paper v2.0+, academic-paper-reviewer v1.0+ |
| 角色 | 學術研究全流程調度器 |
