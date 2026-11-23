# 《Eden Fall》技術環境與遊戲規格書 (GAME_DESIGN_SPEC)

## 1. 技術環境與代碼規範
- **引擎與管線**：Unity 6000.6.8f1 (Supported 鎖定版，不升級) / URP (中低規格；3D 探索與 2D 戰鬥相機各自獨立 Renderer Data)。
- **輸入與鏡頭**：Input System Package (New，禁用舊版 API)；探索鏡頭使用 Cinemachine 3 (`Unity.Cinemachine`)。
- **Unity 6 API 規範**：`linearVelocity`、`linearDamping`、`angularDamping` (3D/2D 物理通用)；`FindFirstObjectByType`。
- **代碼註解規範**：嚴禁於 `Assets/Scripts/` 及其子目錄生成零散 `README.md`。類別與關鍵方法須繁體中文 `/// <summary>`，所有 `[SerializeField]` 必加 `[Tooltip("...")]`。實作完成後於對話框條列「Inspector 掛載步驟」。
- **常駐架構**：Bootstrap 常駐場景（放置 GameManager、InputReader、SaveSystem、WorldState 等管理器），探索/戰鬥場景不重複放置。

## 2. 專案定位與雙維度架構（3D 探索 ↔ 2D 戰鬥）
- **專案定位**：PC 2.5D 動作探索；Demo 範圍為第一區域垂直切片（邊緣區 → 外界荒野 → 汽水工廠 → 汽水怪 Boss）。
- **3D 探索**：全 3D 俯視鏡頭與場景；地點為獨立 Scene，透過傳送門（`Portal` ↔ `SpawnPoint ID`）雙向線性切換（無單向捷徑門）。
- **2D 戰鬥**：遇敵時 `LoadSceneMode.Additive` 疊加載入專屬 2D 橫向卷軸戰場（紙片角色、2D 敵人、2D 正交鏡頭），戰後卸載並無縫還原 3D 現場。
- **3D 凍結機制（禁 Time.timeScale=0）**：停用 3D 玩家控制、敵人 AI、NavMeshAgent；Rigidbody 記錄原狀態後轉 `isKinematic`；停用探索鏡頭與其 `AudioListener`（全局維持單一 Listener）。`Time.timeScale = 0` 僅用於 ESC 暫停選單。

## 3. 遇敵資料傳遞（BattleContext）與戰後結算
- **開戰型態**：1) 投擲命中敵：被命中該怪帶【先制暈眩】（同隊最多 3 隻）；2) 正面接觸：正常開戰；3) 被敵背刺：玩家帶【開場硬直】。
- **快照機制**：開戰瞬間建立 `PreBattleSnapshot`（僅存記憶體、不寫檔）：玩家座標、HP、物品欄、該組敵人狀態。
- **戰場載入**：各區獨立 2D 場景（`Battle_Edge`、`Battle_Wild`、`Battle_Factory`，共用玩家與 UI Prefab），由 `BattleContext.SceneId` 決定。
- **強制戰/Boss**：由 Trigger 或對話直接生成 `BattleContext`（不走碰撞），標記「不可撤退」。
- **戰後結算**：
  | 結算型態 | 處理邏輯 |
  |---|---|
  | **勝利** | 卸載 2D 場景；3D 刪除該組敵人並記錄至 `WorldState`（已擊敗）；發放掉落物（不自動回血）。 |
  | **撤退** | 卸載 2D 場景；原 3D 該組敵人獲得 3 秒短暫無敵，防止立刻重複碰撞開戰。 |
  | **敗北** | 卸載 2D 場景；**依 PreBattleSnapshot 還原至開戰前**（位置、HP、物品欄、敵群狀態與血量完全復原，不留殘血）；雙方短暫無敵；**敗北不讀存檔**（Boss 戰同理）。 |

## 4. Input System Action Maps 與鍵位配置（純鍵鼠，無改鍵與手把）
- **Exploration Map**：`WASD` 8 向移動、`Space` 跳躍、`F` 互動/撿取、`LMB` 瞄準投擲（投出即消耗，攜帶上限預設 3，可調參數）、`ESC` 暫停選單。
- **Combat Map（無耐力條、無空中下砍）**：
  - `A` / `D`：左右移動（按住 RMB 格擋時鎖定水平移動）。
  - `Space`：跳躍（短按小跳/長按大跳；可強制中斷普通格擋）。
  - `Shift`：無敵幀閃避（最高優先取消權；按住格擋時按 Shift 可立即取消並閃避）。
  - `Q`：使用物品欄補品回血（動作結束時結算，被取消則不消耗道具）。
  - `LMB`：點按普通攻擊（空地通用）；長按 ≥ 0.25 秒蓄力重擊（空地通用、蓄力不可移動、被打中斷，蓄滿破甲並使非 Boss 敵暈眩）。
  - `RMB`：按住普通格擋（減傷、不可移動）；受擊瞬間點按完美格擋（0 傷害、反彈投射物、敵硬直）。紅光警示攻擊不可格擋，必用 `Shift` 閃避。
  - `ESC`：開啟暫停選單。
- **UI Map**：`ESC` 關閉暫停選單、UI 導航與滑鼠點擊。

## 5. 戰鬥動作取消規則（Combat 狀態機）
- **優先權**：`Shift` 閃避 > `Space` 跳躍 > `RMB` 格擋 > `LMB` 攻擊/蓄力 > `Q` 補品。
- **緩衝機制**：不可取消時輸入進入 0.15 秒輸入緩衝；閃避中、受擊硬直、暈眩狀態不可取消，僅接受緩衝。
| 當前狀態 | Shift 閃避 | Space 跳躍 | RMB 格擋 | LMB 攻擊/蓄力 | Q 補品 |
|---|---|---|---|---|---|
| **待機 / 地面移動** | 閃避 | 跳躍 | 格擋 | 點按普攻 / 長按蓄力 | 飲用補品 |
| **空中** | 閃避 | 不可（無二段跳） | 不可 | 空中普攻 / 空中蓄力 | 不可 |
| **格擋中** | 中斷並閃避 | 中斷並跳躍 | 放開結束格擋 | 不可 | 不可 |
| **攻擊前搖 / 後搖** | 取消並閃避 | 不可 | 不可 | 緩衝連段 | 不可 |
| **蓄力中** | 中斷並閃避 | 不可 | 不可 | 放開釋放重擊 | 不可 |
| **補品使用中** | 取消並閃避（不耗道具） | 不可 | 不可 | 不可 | — |
| **閃避 / 硬直 / 暈眩** | 緩衝 | 緩衝 | 緩衝 | 緩衝 | 緩衝 |

- **手感參數設定（皆為 [SerializeField]）**：
  | 參數名稱 | 預設初始值 | 說明 |
  |---|---|---|
  | 完美格擋判定窗口 | 0.2 秒 | 敵人判定幀抵達前的寬容時間 |
  | 閃避冷卻時間 (CD) | 1.0 秒 | 閃避後至下一次可閃避間隔 |
  | 閃避無敵幀長度 | 0.3 秒 | 閃避動作初期的無敵時間 |
  | 撤退後敵人無敵 | 3.0 秒 | 脫離戰場後避免重疊碰撞觸發 |
  | 輸入緩衝窗口 | 0.15 秒 | 動作後搖期間預先註冊指令 |

## 6. 核心戰鬥數值、敵人與水機制
- **傷害公式**：`實際傷害 = Max(1, 攻擊力 - 護甲)`。由 `EnemyTurnScheduler` 統一調度攻擊權，禁止敵群重疊攻擊。
- **數值載體**：`EnemyData (ScriptableObject)` 記錄 HP、攻擊、護甲、類型（近戰/遠程/精英/Boss）、汽水標籤、紅光攻擊參數。
- **敵人配置**：近戰（揮擊/紅光衝撞）、遠程（慢速投射物可被完美格擋反彈）、精英怪（帶格擋姿態需蓄力破甲/繞背，邊緣區出境口首次單挑，後續工廠）、Boss（汽水怪）。
- **汽水工廠水機制**：場景配置可擊破水管/閥門，玩家普攻擊破潑水。僅汽水系與 Boss 受影響：進入【淋濕】狀態（持續 10 秒，護甲削弱/移動減速；重複觸發重置時間不疊加；屬即時狀態，戰後或重置自動清除）。

## 7. UI 動態暫停與存檔世界狀態（WorldState）
- **動態 ESC 暫停選單**：無地圖實體存檔點。探索模式第 3 鈕為【存檔】；戰鬥模式第 3 鈕為【戰鬥撤退】（Boss/劇情強制戰反灰鎖定）。設定僅含音量與解析度。
- **存檔槽位**：手動 1 格（僅探索 ESC）+ 自動 1 格（Portal 切換新場景、Boss 戰勝利）。採原子寫入（先寫 Temp 檔，驗證後覆蓋正式檔）。
- **讀取原則**：「繼續遊戲」讀取手動/自動中時間較新者（閃退重啟同理）。
- **SaveData**：記錄目前場景名、座標或 `SpawnPoint ID`、HP、物品欄、`WorldState`、時間戳。
- **WorldState**：各敵人、拾取物、機關皆具唯一 `PersistentId`，載入場景時比對並銷毀已完成項目，防止刷新。

## 8. C# 腳本架構與 Phase 1 ~ Phase 3 開發排程藍圖
| 腳本名稱 | 資料夾目錄 | 核心職責 | 通訊方式 | 開發階段 |
|---|---|---|---|---|
| `GameBootstrap` | `Assets/Scripts/Core/` | 檢查並加載常駐單例、初始化存檔與加載啟動場景 | Direct Call / Unity Event | Phase 1 |
| `GameStateManager` | `Assets/Scripts/Core/` | 維護遊戲宏觀狀態（探索 / 戰鬥 / 暫停），派發狀態切換 | C# Action Event | Phase 1 |
| `InputReader` | `Assets/Scripts/Core/` | 封裝新版 Input System，集中處理 Action Map 啟用與事件派發 | C# Action Event | Phase 1 |
| `SceneFlowController` | `Assets/Scripts/Core/` | 非同步場景流轉、Additive 疊加/卸載 2D 戰鬥場景 | C# Action Event | Phase 1 |
| `SaveSystem` / `SaveData` | `Assets/Scripts/Core/` | 雙槽位 JSON 序列化、原子寫入檔案、提供自動與手動存檔 API | Direct Call | Phase 1 |
| `WorldStateRegistry` / `PersistentId` | `Assets/Scripts/Core/` | 註冊與比對場景實體存續狀態，避免離開後重複生成 | Direct Call / C# Action | Phase 1 |
| `Portal` / `SpawnPoint` | `Assets/Scripts/Exploration/` | 3D 場景雙向門與出生點 ID 綁定，觸發傳送與自動存檔 | C# Action (`SceneFlowController`) | Phase 1 |
| `ExplorationController` | `Assets/Scripts/Exploration/` | 3D 角色 WASD 移動、跳躍、物品互動檢視與投擲計數 (上限 3) | Listen `InputReader` | Phase 2 |
| `ExplorationFreezer` | `Assets/Scripts/Exploration/` | 進戰鬥時凍結 3D 角色、敵 AI、NavMesh、鏡頭與 AudioListener | Listen `BattleTransitionManager` | Phase 2 |
| `CombatStateMachine` | `Assets/Scripts/Combat/` | 2D 橫向戰鬥狀態機（移動/跳躍/格擋/閃避/蓄力重擊/輸入緩衝） | Listen `InputReader` | Phase 2 |
| `CombatDamageCalculator` | `Assets/Scripts/Combat/` | 傷害計算 `Max(1, Atk - Def)`、護甲削減、破甲狀態判定 | Direct Call / C# Action | Phase 2 |
| `BattleTransitionManager` | `Assets/Scripts/Combat/` | 控管進出戰鬥流程、建立 `PreBattleSnapshot`、結算與敗北還原 | Listen Events, Dispatch Scenes | Phase 2 |
| `BattleContext` / `PreBattleSnapshot`| `Assets/Scripts/Combat/` | 資料載體：遇敵觸發類型、敵編隊、開戰前玩家與敵人記憶體快照 | Pure C# Data Class | Phase 2 |
| `EnemyData` | `Assets/Scripts/Combat/` | 敵人數值 ScriptableObject（HP、攻擊力、護甲、汽水標記、紅光攻擊） | ScriptableObject Asset | Phase 2 |
| `EnemyTurnScheduler` | `Assets/Scripts/Combat/` | 2D 戰場輪流發放攻擊權，協調敵群攻擊順序以避免同幀重疊出手 | C# Action / Direct Call | Phase 3 |
| `BaseEnemyAI` / `EliteEnemyAI` | `Assets/Scripts/Combat/` | 小怪與精英怪行為邏輯（近戰揮擊、遠程投射物、格擋姿態、紅光衝刺） | Read `EnemyData`, Turn Scheduler | Phase 3 |
| `BossController` | `Assets/Scripts/Combat/` | 汽水怪 Boss 階段轉換、特殊紅光招式調度與不可撤退鎖定 | Listen `CombatDamageCalculator` | Phase 3 |
| `WaterValveMechanism` | `Assets/Scripts/Combat/` | 工廠可擊破水管閥門，遭打碎時潑水賦予汽水系敵人 10 秒淋濕 Debuff | Direct Call / C# Action | Phase 3 |
| `PauseMenuController` | `Assets/Scripts/UI/` | ESC 暫停選單（動態切換存檔/撤退按鈕，音量與解析度設定） | Listen `InputReader`, Call Core | Phase 3 |
| `BattleHUDController` | `Assets/Scripts/UI/` | 2D 戰鬥 HUD（血條、補品數量、淋濕狀態、按鍵提示） | Listen Combat Events | Phase 3 |
