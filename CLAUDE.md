# 內科專科題庫 — 專案操作手冊（Operations Manual）

> 本檔是這個 repo 的**單一權威操作手冊**。舊版「題目灌入規格」已完成階段性任務並過時，本檔取而代之。
> 灌題細節見保留的 `EXPLAIN.md`（詳解生成規格）與 `CLASSIFY.md`（專科分類規格）。

---

## §0 給未來模型的第一段話（先讀這段，這是鐵則）

你（未來接手的 Claude／Opus／Sonnet）若要在此 repo 繼續工作，**務必遵守下列鐵則**，違反任何一條都可能毀掉整批已完成的成果：

1. **序列、逐科處理，禁止並行、禁止 background agents。** 一次只做一個「年度 × 專科」，做完一科、驗證通過、commit + push、確認成功，**才**開始下一科。曾經用並行 background agents 導致整批白跑（見 §11）。
2. **絕不更動保護欄位**：`id`、`year`、`type`、`question`、`choices`、`answer`、`subject`。只能新增／修改 `explanation`。每次寫入後都要用快照 diff 證明保護欄位 byte-identical（見 §7）。
3. **絕不改動、也不暗示既定答案有誤。** 詳解一律與 PDF／題庫既定 `answer` 一致。反向題（為非／不正確／最不適合／何者錯誤）邏輯要對準既定答案，不可寫反。你覺得答案有問題也照寫，頂多在回報時提出，不要自行「更正」。
4. **每次 commit 一定把 `index.html` 的 `questions.json?v=N` 版本號 +1**，與題庫一起 commit，否則使用者抓到舊快取。目前為 **v85**。
5. **撞額度／斷線可接續**：一律停在「已 commit 的科」這個乾淨狀態，不留半成品。恢復後用 §9 統計指令確認進度再往下做。
6. **增補不重寫**：升級既有詳解時，已達標的原內容原封保留，只補缺的段落（多半是 `## 重點整理表`）或補字數，不要整段重寫。

---

## §1 專案總覽

- **產品**：台灣「內科專科醫師」考古題線上題庫，含每題完整詳解。
- **線上網址**：GitHub Pages（`index.html` 部署後的 Pages URL）。
- **repo**：本目錄即 git repo，`main` 為部署分支，push 到 `origin/main` 後 Pages 自動更新。
- **本機路徑**：`/Users/tinrepin/Desktop/im-exam`
- **使用者背景**：醫療／醫學教育相關，重視醫學正確性與繁體中文用語；偏好「一科一科、驗證後才 commit、確認 push 再往下」的穩健節奏；commit message 用英文。

---

## §2 系統架構

- **兩檔制**：
  - `index.html`：單檔 React 應用（含 UI、載入邏輯、樣式）。
  - `questions.json`：1000 題的 JSON 陣列（唯一資料來源）。
- **前端技術**：React 18 UMD（`react@18` / `react-dom@18` production.min）+ **Babel Standalone 釘死 `@babel/standalone@7.26.4`（classic JSX，非 automatic runtime）**。瀏覽器端即時轉譯 JSX。
  - **不要升級 Babel 到 v8**：曾造成整頁白畫面（見 §11）。維持 7.26.x。
- **部署**：GitHub Pages，push `main` 即自動發佈，無 build step。
- **快取破壞**：`index.html` 以 `xhr.open('GET', 'questions.json?v=N', ...)` 載入題庫；每次題庫變動就把 `N` +1，強制使用者抓新檔。

---

## §3 資料結構規格

每題物件：

```json
{
  "id": "114-S8",
  "year": 114,
  "type": "S",
  "subject": "心臟科",
  "question": "題幹完整文字……",
  "choices": [
    { "key": "A", "text": "……" },
    { "key": "B", "text": "……" }
  ],
  "answer": "D",
  "explanation": {
    "summary": "一句粗體命中考點 + 理由……",
    "detail": "## 選項分析 …（Markdown，含 ## 重點整理表）",
    "sources": [ { "text": "2022 ESC/ERS Guidelines" } ]
  }
}
```

### 保護欄位（絕不可改）
`id`、`year`、`type`、`question`、`choices`、`answer`、`subject`。只有 `explanation` 可新增／修改。

### 年度題數分布（總計 1000）
| 年度 | 題數 |
|---|---|
| 109 | **200**（注意：唯一 200 題的年度）|
| 110 | 160 |
| 111 | 160 |
| 112 | 160 |
| 113 | 160 |
| 114 | 160 |

`id` 格式 `{year}-S{流水號}`，流水號依 PDF 題序，全域唯一。`type` 一律 `"S"`（單選），`answer` 一律單一字串（非陣列）。

### 9 個專科 `subject` 值（用字要一模一樣）
`心臟科`、`胸腔科`、`腸胃科`、`腎臟科`、`感染科`、`血液腫瘤科`、`新陳代謝科`、`風濕免疫科`、`其他類`

- **「其他類」帶「類」字**（不是「其他」），比對時別漏。
- 全庫專科分布（供估工作量）：血液腫瘤科 121、胸腔科 120、腸胃科 116、心臟科 115、感染科 114、新陳代謝科 113、腎臟科 107、風濕免疫科 103、其他類 91。

### 反向題規則
題幹含「為非／不正確／何者錯誤／不妥／並非／最不適合／下列何者不……」等否定語者為反向題：
- **逐字保留題幹**，不可改寫成正向。
- 選項分析要明確標出「正確敘述（✓）」與「題目要選的那個錯誤敘述＝答案（✗，正確答案）」，邏輯對準既定 `answer`。

---

## §4 檔案清單與職責

| 檔案 | 職責 | 是否進 git |
|---|---|---|
| `index.html` | React 單檔應用 + 快取版本號 | ✅ |
| `questions.json` | 1000 題題庫（唯一資料來源）| ✅ |
| `CLAUDE.md` | 本操作手冊（權威）| ✅ |
| `EXPLAIN.md` | 詳解生成規格（擴充標準、few-shot 範例）| ✅ |
| `CLASSIFY.md` | 專科分類規格 | ✅ |
| `review_needed.md` | 灌題階段低信心清單 | ✅ |
| `subject_review.md` | 專科分類複核清單 | ✅ |
| `append_expl.py` | 增補＋驗證工具（增補式改 detail 並自動驗證）| ❌ gitignored |
| `build_append_*.py` / `append_*.json` | 每批次的臨時建構檔 | ❌ gitignored，用完即刪 |
| `questions.json.bak` | 驗證用備份 | ❌ gitignored，commit 前刪 |
| `resource/`、PDF | 原始考卷 | ❌ **絕不進 git** |

`.gitignore` 已涵蓋上述 ❌ 項目。commit 前務必確認 `git status` 乾淨、無殘留 `.bak` 與臨時建構檔。

---

## §5 核心任務：詳解生成（explanation）

**規格權威在 `EXPLAIN.md`（擴充標準），本節只做摘要。**

- `explanation` 三欄：`summary`（80–200 字，粗體命中考點＋理由）、`detail`（Markdown）、`sources`（1–3 筆指引／試驗／教科書）。
- `detail` **必含**兩段：
  - `## 選項分析`：逐項 A–E（或複選逐項 1–5）分析，正確 ✓、答案 ✗（正確答案）；錯誤選項要補「正確的應該是什麼、為什麼」。
  - `## 重點整理表`：detail 結尾一個 Markdown 表格，把本題考點濃縮成「考前掃一眼」的複習表，**至少 4 個內容列**（不含表頭與分隔線）。慣例最後一列放 `| 本題答案 | X |`。
- **字數**：簡單題 ≥1000 字；整合題／計算題／複選題建議 1500–2200 字。
- **風格**：中英並陳、關鍵詞粗體、cut-off／試驗名稱（如 IABP-SHOCK II、PROSEVA、ART、INCREASE）明確標出。
- **醫學正確性第一**：符合現行國際指引與主流共識；不確定的數據寧可不寫，**不可杜撰**試驗／數據／指引。
- **圖片題**：`question` 提到「如圖／附圖」但 JSON 無影像時，依題幹既有文字資訊與既定答案撰寫詳解，不要臆造看不到的影像細節；必要時在 summary／detail 用文字描述典型影像特徵即可。

---

## §6 標準工作流程（序列逐科）

對每一個「年度 × 專科」，照下列順序：

1. **統計**：用 §9 指令確認該科題數、哪些缺 `## 重點整理表`、哪些字數 <1000。
2. **生成／增補**：撰寫該科每題要新增的內容。慣例做法：寫一支 `build_append_<year>_<科>.py` 產生 `{id: "要附加到 detail 的 markdown"}` 的 JSON。
3. **寫入 + 驗證**：跑 `append_expl.py`（見 §7），它會備份、增補、並自動驗證。
4. **通過才 commit + push**：`index.html` 版本號 +1、刪 `.bak` 與該科臨時檔（見 §8）。
5. **確認 push 成功**，才開始下一科。撞額度就停在這個乾淨狀態。

**兩種節奏**：
- **手動確認**：每科 commit 前先向使用者回報「本科幾題、字數 min/avg/max、有無疑難」，等確認再 push。首次合作或不確定時用這個。
- **半自動**：使用者已明確授權「整批做完、逐科自動 commit+push」時，可一科接一科連續執行，但**仍是序列**、每科仍要驗證通過才 commit，並在全部完成後總結回報。

一次處理一個年度（除非使用者指定多個）；同一年度內從題數最多的科開始。

---

## §7 驗證機制 Gate

寫入後**每次**都要通過下列七項；任一項失敗 → 不 commit、回報問題：

1. `questions.json` 是合法 JSON，且**仍是 1000 題**。
2. 本科每題都有 `explanation`，且 `summary`、`detail`、`sources` 三欄齊備非空。
3. `detail` 含 `## 選項分析`。
4. `detail` 含 `## 重點整理表`，且表格 **≥4 內容列**。
5. `detail` 字數 **≥1000**。
6. **保護欄位** byte-identical（改動前後快照比對 `id/year/type/question/choices/answer/subject`）。
7. **其他年度、其他科的 `explanation` 完全未動**（非目標題快照比對）。

`append_expl.py` 已實作全部七項。它的用法與核心邏輯：

```
python3 append_expl.py <append_json> <year> <subject>
# append_json 格式：{"114-S8": "要附加到該題 detail 結尾的 markdown", ...}
```

它會：備份 `questions.json` → `questions.json.bak`；快照保護欄位與所有「非目標題」的 explanation；對每個 id 做 `detail = detail.rstrip() + "\n\n" + append.strip()`（**增補、不重寫**）；寫回後驗證上述七項，印出 `applied=N`、字數 min/max/avg，全部通過印 `VALIDATION PASSED`，否則列出錯誤並 `exit 1`。

若不用該工具，也可用等效的獨立驗證腳本（唯讀）：

```python
import json, collections
d = json.load(open('questions.json', encoding='utf-8'))
assert len(d) == 1000, len(d)
ids = [q['id'] for q in d]
assert len(set(ids)) == 1000, 'dup id'
bad = []
YEAR, SUBJ = 114, '心臟科'          # 改成要檢查的範圍
for q in d:
    if q['year'] != YEAR or q['subject'] != SUBJ:
        continue
    e = q.get('explanation') or {}
    det = e.get('detail', '')
    if not (e.get('summary') and det and e.get('sources')):
        bad.append(q['id'] + ' missing-field')
    if '## 選項分析' not in det:
        bad.append(q['id'] + ' no-選項分析')
    if '## 重點整理表' not in det:
        bad.append(q['id'] + ' no-重點整理表')
    else:
        tail = det.split('## 重點整理表', 1)[-1]
        rows = [ln for ln in tail.splitlines() if ln.strip().startswith('|')]
        if max(0, len(rows) - 2) < 4:
            bad.append(q['id'] + ' table<4')
    if len(det) < 1000:
        bad.append(q['id'] + f' short({len(det)})')
print('ISSUES:', bad or 'NONE')
```

---

## §8 Git 慣例與版本號

- **確認後才 commit**；commit message 用英文、描述本次內容。
  - 灌題例：`Add 109 questions (80 items)`
  - 詳解例：`Add explanations for 110 cardiology (18 questions)`
  - 升級例：`Upgrade 113 nephrology explanations to expanded standard`
- **每次題庫變動的 commit 一定同時把 `index.html` 的 `?v=N` +1**（一起 add、一起 commit）。
- **9 科英文名對照**（commit message 用）：

| 中文 | 英文 |
|---|---|
| 心臟科 | cardiology |
| 胸腔科 | pulmonology |
| 腸胃科 | gastroenterology |
| 腎臟科 | nephrology |
| 感染科 | infectious disease |
| 血液腫瘤科 | hematology and oncology |
| 新陳代謝科 | endocrinology and metabolism |
| 風濕免疫科 | rheumatology |
| 其他類 | miscellaneous |

- **push 步驟**：`git add questions.json index.html` → `git commit` → `git push origin main`。commit co-author 行照使用者環境慣例附上。
- **push 被拒（non-fast-forward）**：先 `git pull --rebase origin main`，解決後再 push；若有他人／他 session 的變更，先確認不衝突再續。
- **絕不** `git add` `resource/`、PDF、`.bak`、`build_append_*.py`、`append_*.json`。

---

## §9 進度查詢指令

```python
python3 -c "
import json, collections
d = json.load(open('questions.json', encoding='utf-8'))
print('total', len(d))
for y in sorted(set(q['year'] for q in d)):
    qs = [q for q in d if q['year'] == y]
    notab = sum('## 重點整理表' not in (q.get('explanation') or {}).get('detail','') for q in qs)
    short = sum(len((q.get('explanation') or {}).get('detail','')) < 1000 for q in qs)
    noexp = sum(not (q.get('explanation') or {}).get('detail') for q in qs)
    print(f'{y}: n={len(qs)} no_explanation={noexp} no_table={notab} short={short}')
"
```

查某年某科明細：把上面篩選改成 `q['year']==113 and q['subject']=='腸胃科'` 並印每題 `id/len`。

---

## §10 如何調動 Claude Code / Cowork

- **啟動**：在本機終端機 `cd /Users/tinrepin/Desktop/im-exam` 後執行 `claude`（Claude Code CLI）。若用 Cowork／桌面版，開啟對應工作階段並指向本目錄。
- **辨別「Claude Code 提示」vs「zsh 提示」**：Claude Code 的輸入框是給**自然語言指令**用的；zsh（終端機 `%` 提示）是給 **shell 指令**用的。**別把要跟 Claude 說的話貼進 zsh，也別把 shell 指令當對話貼**（見 §11 踩坑）。要 Claude 幫你在本 session 跑 shell，可用 `! <command>` 前綴。
- **指令範本**（貼給 Claude Code）：
  > 讀 CLAUDE.md 和 EXPLAIN.md。執行 §6 標準流程：把 <年> 年 <科> 升級到擴充標準，序列逐科、禁止並行、禁止 background agents。每科：統計→增補→`append_expl.py` 驗證→通過才 commit+push（message 用 §8 格式、`?v=` +1、刪 .bak 與臨時檔）→確認 push 再下一科。撞額度就停在已 commit 的乾淨狀態，完成後總結回報。
- **`/login`**：若 CLI 顯示未登入或額度／驗證失效，執行 `/login` 重新登入。
- **`caffeinate`**：長時間批次時，另開終端機跑 `caffeinate -dimsu` 防止 Mac 休眠中斷 session（做完 Ctrl-C 結束）。

---

## §11 過去採過的誤區（血淚教訓）

| 嚴重度 | 誤區 | 後果 | 正解 |
|---|---|---|---|
| ★★★ | 用**並行 background agents** 同時生成多科 | 多 agent 互相覆寫 `questions.json`、整批白跑、難以復原 | **序列逐科**，一次一科，`append_expl.py` 內建非目標題保護 |
| ★★★ | 詳解**擅自「更正」或暗示既定答案有誤** | 破壞題庫可信度、與使用者信任 | 一律照既定 `answer` 寫，反向題邏輯對準答案；有疑慮只在回報時提 |
| ★★ | **Babel 升到 v8** | 前端整頁白畫面、題庫打不開 | 釘死 `@babel/standalone@7.26.4`（classic） |
| ★★ | commit 題庫但**忘記 `?v=N` +1** | 使用者一直抓到舊快取、看不到更新 | 每次與題庫一起 +1 |
| ★★ | 把要跟 Claude 說的話**貼到 zsh**（或 shell 指令貼到對話框） | 指令亂噴、無效或報錯 | 分清 Claude 輸入框 vs zsh；本 session 跑 shell 用 `! cmd` |
| ★ | 詳解**字數偏短（900–999）硬 commit** | 未達 ≥1000 標準、Gate 應擋下 | 驗證未過不 commit；補「## 重點整理表」或有料內容到達標 |

---

## §12 偵錯方式（症狀 → 解法）

| 症狀 | 可能原因 | 解法 |
|---|---|---|
| 打開網頁**整頁白** | Babel/React 版本問題、JSON 壞掉 | 檢查 `index.html` Babel 釘 7.26.4；用 `python3 -m json.tool questions.json` 驗 JSON |
| 更新後**看不到新內容** | 忘了 `?v=N` +1 或瀏覽器快取 | 確認版本號已 +1、已 push；hard reload |
| `append_expl.py` 印 **`OTHER explanation changed!`** | 誤動到非目標題 | 還原 `.bak`，檢查 append_json 的 id 範圍，只放本科 id |
| 驗證 **`table<4`** | 重點整理表內容列不足 4 | 補到 ≥4 個內容列（`\|...\|` 行，不含表頭/分隔線）|
| 驗證 **`short(NNN)`** | detail <1000 字 | 補機轉／鑑別／試驗／cut-off／易錯陷阱或加表格列 |
| `git push` **被拒** | 遠端有新 commit（non-fast-forward）| `git pull --rebase origin main` 後再 push |
| 某科**題數對不上** | subject 用字錯（如「其他」漏「類」）| 用 §3 的精確 subject 值比對 |

---

## §13 進度追蹤

- **狀態：1000 / 1000 題全部完成，全庫已統一至「擴充標準」。**
- **涵蓋**：109–114 **六個年度 × 9 專科**，每題皆含 `summary`／`detail`／`sources`，`detail` 皆含 `## 選項分析` 與 `## 重點整理表`（≥4 列）、字數 ≥1000。
- **113 / 114 於「階段 D」升級**：這兩年原本缺 `## 重點整理表`、多題字數不足，已逐科增補至擴充標準（增補不重寫）。
- **最新快取版本號：`v=85`**
- **完成日期：2026-07-17**

### EXPLAIN.md 標準演進史
1. **初版**：只要求 `summary` + 基本 `detail` + `sources`。
2. **加詳細版**：`detail` 必含 `## 選項分析`，逐項講透對錯與正解理由。
3. **擴充標準（現行）**：再加 **`## 重點整理表`（≥4 列）必含**、字數下限（簡單題 ≥1000、整合／計算／複選 1500–2200）、中英並陳、標明試驗與 cut-off。109–112 於此標準生成；113／114 以「階段 D」補齊到此標準，全庫一致。

---

## §14 優化建議（做下批前值得記住）

- **序列 > 並行**：本任務的瓶頸不是速度而是正確性與不互相覆寫；序列逐科最安全（見 §11 ★★★）。
- **Gate 只顧結構、不顧醫學**：§7 自動驗證能保證「有表格、夠長、沒動到別題」，但**保證不了醫學正確**。醫學正確要靠生成時的嚴謹＋完工後人工抽查（§16 階段 C）。
- **用 Python 建構、避免手打跳脫**：詳解含大量 Markdown 表格與符號，直接手寫進 JSON 容易跳脫錯誤；用 `build_append_*.py` 以字串字面量組裝再 `json.dump(..., ensure_ascii=False)` 最穩。
- **備份 + diff**：每次改動前備份、改動後用快照比對保護欄位與非目標題（`append_expl.py` 已內建）。
- **額度管理**：長批次前用 `caffeinate` 防休眠；撞額度停在乾淨狀態、記下做到哪一科；恢復用 §9 統計確認再續。
- **圖片題**：JSON 無影像，依題幹文字與既定答案寫，別臆造影像細節。
- **檔案漸大**：`questions.json` 已達 1000 題、體積不小；避免整檔反覆貼進對話，改用腳本讀寫與 §9 統計。
- **規格單一來源**：操作看本檔，詳解格式看 `EXPLAIN.md`；兩者衝突時以實際驗證 Gate（§7）為準，並回報使用者更新規格。
- **完工後抽查**：整年度做完後跑一次全年度 §7 驗證 + 抽幾題人工看醫學內容。

---

## §15 一頁速查（TL;DR）

- **現況**：1000/1000 全部完成，全庫擴充標準，快取 **v85**，完成日 **2026-07-17**。
- **鐵則**：序列逐科（禁並行/禁 background agents）｜不動保護欄位｜不改/不暗示答案有誤｜每次 `?v=N` +1｜增補不重寫｜撞額度停在乾淨狀態。
- **一科流程**：§9 統計 → `build_append_*.py` 生成 → `python3 append_expl.py <json> <year> <科>` 驗證 → 過了才 `git add questions.json index.html && commit && push`（`?v=` +1、刪 .bak/臨時檔）→ 確認 → 下一科。
- **詳解必含**：`## 選項分析` + `## 重點整理表`（≥4 列）＋字數 ≥1000。
- **9 科英文名**：cardiology / pulmonology / gastroenterology / nephrology / infectious disease / hematology and oncology / endocrinology and metabolism / rheumatology / miscellaneous。
- **別踩**：並行白跑★★★｜暗示答案錯★★★｜Babel v8 白畫面★★｜忘記版本號★★。
- **下一步（待使用者執行）**：§16 階段 C（醫學正確性人工抽查）與階段 E3（手機排版實測）。

---

## §16 收尾檢查清單

### 階段 A — 結構與完整性驗證　✅ 已完成
- 全 1000 題 JSON 合法、題數正確、`id` 唯一。
- 每題 `explanation` 三欄齊備；`detail` 含 `## 選項分析` 與 `## 重點整理表`（≥4 列）；字數 ≥1000。
- **發現與處理**：113/114 共 **320 題**原缺 `## 重點整理表`，已逐科增補；其中 **126 題**字數 <1000，已補有料內容到達標。

### 階段 B — 一致性檢查　✅ 已完成
- 保護欄位在所有升級批次前後 byte-identical；非目標題 explanation 未被動到。
- 各年各科 `detail` 字數 min/mean/max 已掃描，無異常離群（全庫 ≥1000）。

### 階段 C — 醫學正確性人工抽查　⬜ 待使用者執行
- 自動 Gate 保證不了醫學正確。建議由具醫學背景者抽查各科數題，重點看：反向題邏輯是否對準答案、試驗名稱與結論、cut-off 數值、藥物適應症/禁忌。發現錯誤回報後定點修正（僅動該題 `explanation`）。

### 階段 D — 升級舊批次至擴充標準　✅ 已完成
- 114 全 9 科、113 全 9 科皆已由「有選項分析但缺重點整理表／偏短」升級至擴充標準。
- **零星修補**：**114-S8** 補上原缺的 `## 選項分析`；**111-S149** 的重點整理表由 3 列補到 ≥4 列。

### 階段 E — 部署收尾
- **E1** 快取版本號 `?v=N`：每次 push 已 +1，現為 **v85**。✅
- **E2** push `main`、Pages 自動發佈：各批次已 push 成功。✅
- **E3** 手機／不同瀏覽器實機排版測試（表格是否橫向溢出、詳解可讀性）：⬜ **待使用者執行**。
