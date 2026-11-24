# Unity 專案開發規範

## 1. 檔案修改邊界（嚴格遵守）
- **僅限修改程式碼：** 只允許新增或編輯 `Assets/` 底下的 `.cs` 腳本與 `.md` 文件。
- **嚴禁修改序列化資源：** 絕對不要直接讀取、編輯或生成 `.unity`（場景）、`.prefab`（預製體）、`.mat`（材質）、`.anim`（動畫）與 `.meta` 檔案。

## 2. 交付與協作格式
- **附上 Unity 編輯器設定清單：** 每次新增或修改 `MonoBehaviour` 腳本後，必須在回覆最後列出清楚的「Unity 編輯器手動操作清單」，明確寫出哪個 GameObject 要掛載哪支腳本，以及 Inspector 中哪些 `[SerializeField]` 欄位需要手動拖曳指定。
- **程式碼風格：** 優先使用 `[SerializeField] private` 暴露需要調整的參數，避免濫用 `public` 欄位；修改既有腳本時保持最小改動，不隨意重構未提及的系統。

# Git 作業規則（Antigravity 專用）

你是《Eden Fall》Unity 專案的實作代理。專案負責人是唯一操作你的人。以下規則每次都要遵守。設計內容以 `@/docs/GAME_DESIGN_SPEC.md` 為準。

## 0. 工作模式（必須先確認）
- 本專案一律使用 **Local 模式**，直接在專案資料夾 `C:/Unity Gamework/EdenFall` 工作。Unity 與 Unity MCP 操作的就是這個資料夾。
- **不使用 Worktree 模式。** Worktree 會在另一個資料夾建立獨立副本，你改的腳本不會出現在 Unity 開啟的專案裡，且未追蹤的檔案（如 `Library/`）不會存在於副本中。
- 開工前檢查：若 `git worktree list` 顯示目前所在資料夾不是 `C:/Unity Gamework/EdenFall`，或 `git switch main` 出現「already used by worktree」，立刻停止並回報，請負責人改用 Local 模式開新對話。

## 1. 每個任務的固定流程
1. **開始前**：執行 `git status`。
   - 工作區乾淨：直接開始。
   - 有未提交變更：先回報變更內容，經負責人同意後 commit `chore: checkpoint before <任務>`，再開始。
2. **執行中**：一個任務只做一件事，不順手重構或修改不相關檔案。
3. **用 Unity MCP 改場景或 Prefab 後**：先儲存場景／Prefab，再 commit。未儲存的變更不會被 Git 記錄。
4. **完成後**：確認 Unity Console 無編譯錯誤，並符合該階段驗收標準，再 commit。
5. **回報**：每次 commit 後告知「分支、commit 編號、改了什麼、如何退回」。退回方式一律給 `git revert <commit>` 或 `git restore <路徑>`。

## 2. commit 規則
- 格式：`<類型>(<階段>): <描述>`，例如 `feat(p1): 建立 Bootstrap 與單場景直啟防呆`。
- 類型：`feat`、`fix`、`chore`、`refactor`、`docs`。
- 一個 commit 只含一件事。
- 在任務分支上工作，例如 `feature/p1-bootstrap`。不直接在 `main` 上 commit。

## 3. 允許與禁止
- **可直接執行**：`git status`、`git fetch`、`git diff`、`git log`、`git add`、`git commit`、`git switch`、`git switch -c`、`git stash`、`git restore <路徑>`、`git revert`。
- **需負責人明確同意才可執行**：`git push`、`git pull`、`git merge`、`git rebase`。
- **絕對不可執行**：`git reset --hard`、`git clean -fd`、`git push --force`、刪除分支、改寫已推送的歷史。
- 不修改 `ProjectSettings/` 與 `Packages/manifest.json`，除非任務明確要求；引入套件前先說明來源、版本、授權。

## 4. 拉取最新版本（負責人要求 pull 時）
1. 執行 `git status` 並回報目前分支。
2. **更新對象是 `main`，不要在任務分支或臨時分支上 pull。** 若目前不在 `main`：
   - 先執行 `git log origin/main..<目前分支> --oneline`。有輸出代表這個分支有 `main` 沒有的 commit，此時停止並回報，不要切換。
   - 沒有輸出才執行 `git switch main`。
3. 沒有未提交修改才執行 `git pull`，僅此一步。若出現 `no tracking information`，改用 `git pull origin main`。
4. 出現衝突或警告：立刻停止並回報，不自行解決。
5. 完成後回報 `git log -1 --oneline`，以及本機 `main` 是否等於 `origin/main`。
6. 之後的新任務，一律從更新後的 `main` 開新分支。

## 5. 遇到下列情況立即停止並回報
- 合併衝突，或 `.unity`、`.prefab` 檔案疑似被他人同時修改。
- 即將提交 `Library/`、`Temp/`、`Logs/`、`UserSettings/`、`*.csproj`、`*.sln`。
- 新增資產缺少對應的 `.meta`。
- 單一檔案超過 50 MB 且未使用 Git LFS（GitHub 對單檔上限為 100 MB）。
- 不確定某個指令是否會丟失未提交內容。

## 6. 協作邊界
- 其他組員會修改 3D 場景與美術資產。你只處理負責人指定的場景與腳本，不主動修改環境美術相關的場景或 Prefab。
- 同一時間只由一方修改同一個場景或 Prefab。開始修改場景前，先向負責人確認該場景目前沒有人在改。