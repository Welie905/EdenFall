# 《Eden Fall》技術環境與遊戲規格書 (GAME_DESIGN_SPEC)

## 1. 技術環境、代碼規範與常駐架構
- **引擎管線**：Unity 6000.6.8f1 (Supported 鎖定版) / URP (中低規格；3D 探索與 2D 戰鬥相機各自獨立 Renderer Data)。
- **輸入鏡頭**：Input System Package (New，禁用舊 API)；探索鏡頭使用 Cinemachine 3 (`Unity.Cinemachine`)。
- **Unity 6 API**：`linearVelocity`、`linearDamping`、`angularDamping` (3D/2D 通用)；`FindFirstObjectByType`。
- **架構相依**：五大目錄（`Assets/Scripts/` 下 `Core`, `Exploration`, `Combat`, `UI`, `Dialogue`）。跨系統事件放 Core (`BattleEvents`)，戰鬥資料載體（`BattleContext`、`PreBattleSnapshot`、`EnemyData`）屬 Core；Exploration/Combat 僅依賴 Core（防循環引用）。嚴禁散落 `README.md`。類別/關鍵方法附中文 `/// <summary>`，`[SerializeField]` 必加 `[Tooltip]`。
- **常駐架構**：Bootstrap 場景承載跨場景物件（3D 玩家置於此，不使用 `DontDestroyOnLoad`），載入探索場景須 `SetActiveScene`；使用 `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` 支援任意場景直啟。戰鬥場景 (`Battle_*`) 禁放 EventSystem、AudioListener、CinemachineBrain、全域 Volume。
- **Prefab 與計時**：Root-Visuals 分離（Root 掛 Collider/剛體/邏輯；Visuals 放模型/動畫，2D 翻面僅翻 Visuals）。戰鬥與探索機制計時一律使用受時間縮放影響的時間（`Time.deltaTime`，禁 `unscaledDeltaTime`/`WaitForSecondsRealtime`，確保 ESC 暫停全局凍結；禁依賴 Animation Event）；例外：暫停選單等 UI 動畫須使用 unscaled 時間，否則 `timeScale` 凍結時會卡死。
- **Unity MCP 規範**：連線時由 Antigravity 透過 MCP 調整 Inspector/場景並條列已完成設定；無連線時條列手動步驟。操作前先 Commit，單方修改防衝突。

## 2. 關卡拓撲、傳送時序與 3D 凍結
- **探索拓撲**：各場景獨立 Scene，由 `Portal` ↔ `SpawnPoint ID` 雙向切換（無單向門）。`SC002` 為邊緣區中心樞紐對接 SC001/003/004。SpawnPoint 需錯開回程門或加冷卻防重複傳送。
- **第一區域主線拓撲（由上而下）**：
  1. `[SC001 主角家 (起點)]` (持【簽證申請表】) ↕ `Portal_A` (雙向)
  2. `[SC002 古董店 (廢品區)]` (中心樞紐/投擲拾取教學，上限 3) ↔ `Portal_B` ↔ `[SC003 公會]` (繳表取【出境簽證】折返) ↕ `Portal_C`
  3. `[SC004 安檢/反叛幫 (出境口)]` (憑簽證放行；精英單挑：反叛幫組長，載入 `Battle_Edge`，不可撤退) ↕ `Portal_D` (正向抵達自動存檔)
  4. `[外界荒野/廢墟區]` (跳躍地形+隱藏房裝備；小怪群最多 3 隻，載入 `Battle_Wild`) ↕ `Portal_E` (正向抵達自動存檔)
  5. `[汽水工廠 (內部)]` (汽水系+精英，載入 `Battle_Factory`，解鎖水機制) ↕ `Portal_F` (正向抵達自動存檔)
  6. `[工廠深處：汽水怪 Boss]` (Demo 最終戰，載入 `Battle_Factory`，不可撤退；勝後自動存檔並結算)
- **Portal 傳送 7 步時序**：1) 鎖定 `InputReader` 2) 卸載舊場景/載入新場景（`PersistentId` 於 `Awake` 比對自我銷毀）3) `SetActiveScene` 設為新場景 4) 停用物理/位移並對齊目標 `SpawnPoint` 5) 呼叫 Cinemachine 3 瞬移 API (`OnTargetObjectWarped`) 防阻尼飛行 6) 正向抵達 Portal_D/E/F 觸發自動存檔 7) 解除輸入鎖定。
- **3D 凍結機制**：嚴禁 `Time.timeScale=0` 與 `SetActive(false)` 玩家；改為停用控制腳本、剛體轉 `isKinematic`、關閉 `Visuals`，停用探索鏡頭與 AudioListener。2D 戰場錯開座標或以 Culling Mask 隔離。

## 3. 遇敵傳遞 (BattleContext) 與戰後結算
- **開戰型態**：1) 投擲命中敵：被命中該怪帶【先制暈眩】（同隊最多 3 隻）；2) 正面接觸：正常開戰；3) 敵人先碰到玩家：玩家帶【開場硬直】。
- **資料結構與快照**：開戰建立 `PreBattleSnapshot`（僅記憶體不寫檔：玩家座標/HP/物品/敵群狀態）。`BattleContext` 陣列記錄各怪（上限 3）：`EnemyData`、3D `PersistentId`、是否先制暈眩、HP 快照。Boss 戰與劇情強制戰（含 SC004 反叛幫組長）直接生成，並標記「不可撤退」。
- **戰後結算**：
  | 結算型態 | 處理邏輯 |
  |---|---|
  | **勝利** | 卸載 2D 場景；全編隊 `PersistentId` 寫入 `WorldStateRegistry`（已擊敗）；發放掉落物（不自動回血）。 |
  | **撤退** | 卸載 2D 場景；原 3D 該組敵人獲得 3 秒短暫無敵，防止立刻重複碰撞開戰。 |
  | **敗北** | 卸載 2D 場景；**依 PreBattleSnapshot 還原至戰前狀態**（不留殘血、雙方短暫無敵）；**敗北不讀存檔**（Boss 戰同理）。戰鬥中禁止存檔，中途關閉重啟讀取最近一次存檔。 |

## 4. 輸入映射 (Input System) 與戰鬥狀態機
- **Exploration Map**：`WASD` 8 向移動、`Space` 跳躍、`F` 互動/撿取、`LMB` 瞄準投擲（上限 3，投出消耗）、`ESC` 暫停。
- **Combat Map（無耐力/下砍）**：`A/D` 移動（格擋鎖水平）｜`Space` 跳躍（可中斷格擋）｜`Shift` 閃避（最高取消權）｜`Q` 補品（結束結算，取消不耗）｜`LMB` 點按普攻（空地通用）/ 長按 >= 0.25s 蓄力重擊（空地通用/被打中斷/傷害 = Max(1, Atk - Def) × 2.0，先扣護甲再乘倍率；蓄滿對非 Boss 敵永久破甲並暈眩（Boss 免疫暈眩與永久破甲，僅能靠【淋濕】降甲））｜`RMB` 格擋（按住普擋減傷/出招 0.20s 窗口完美格擋反彈 0 傷；紅光必用 Shift）｜`ESC` 暫停。`UI Map` 供選單導航。
- **動作取消優先權**：`Shift` 閃避 > `Space` 跳躍 > `RMB` 格擋 > `LMB` 攻擊/蓄力 > `Q` 補品。不可取消進入 0.15s 輸入緩衝；受擊/硬直/暈眩僅緩衝。
| 當前狀態 | Shift 閃避 | Space 跳躍 | RMB 格擋 | LMB 攻擊/蓄力 | Q 補品 |
|---|---|---|---|---|---|
| **待機 / 地面移動** | 閃避 | 跳躍 | 格擋 | 點按普攻 / 長按蓄力 | 飲用補品 |
| **空中** | 閃避 | 不可（無二段跳） | 不可 | 空中普攻 / 空中蓄力 | 不可 |
| **格擋中** | 中斷並閃避 | 中斷並跳躍 | 放開結束格擋 | 不可 | 不可 |
| **攻擊前搖 / 後搖** | 取消並閃避 | 不可 | 不可 | 緩衝連段 | 不可 |
| **蓄力中** | 中斷並閃避 | 不可 | 不可 | 放開釋放重擊 | 不可 |
| **補品使用中** | 取消並閃避 (不耗道具) | 不可 | 不可 | 不可 | — |
| **閃避 / 硬直 / 暈眩** | 緩衝 | 緩衝 | 緩衝 | 緩衝 | 緩衝 |

## 5. 戰鬥數值、機制與手感參數
- **基礎數值表（ScriptableObject 與 PlayerStats 初值）**：
  | 單位 | HP | 攻擊 | 護甲 | 特性與數值備註 |
  |---|---|---|---|---|
  | **玩家 (初始)** | 100 | 15 | 3 | 補品每次回復 40 HP；隱藏房拾取裝備永久 +5 攻、+3 甲（無裝備欄，依 WorldState 計算） |
  | **普通小怪** | 45 | 12 | 2 | 近戰揮擊/紅光衝撞、遠程慢速可彈反；玩家普攻 13 傷，4 刀擊殺 |
  | **精英怪** | 120 | 18 | 10 | 帶格擋姿態（未破甲傷 5；破甲後傷 15）；配置轉身延遲供繞背 |
  | **汽水怪 Boss** | 300 | 22 | 12 | 免疫暈眩與永久破甲；僅能靠【淋濕】削弱護甲 |
- **破甲、繞背與調度**：`傷害 = Max(1, Atk - Def)`，`蓄力重擊傷害 = Max(1, Atk - Def) × 2.0`（先扣護甲，再乘倍率；例：Boss 未淋濕 6 傷，淋濕後 18 傷）。蓄滿對非 Boss 敵永久破甲並暈眩（精英暈眩時間為一般的一半；Boss 免疫暈眩與永久破甲，僅能靠【淋濕】降甲）。2D X 軸 `(玩家.X - 精英怪.X)` 與怪面朝相反視為繞背，以命中幀判定無視正面減傷，蓄力重擊直接破甲。`EnemyTurnScheduler` 輪流行動禁同幀重疊。
- **閃避穿身與重疊**：無敵幀 (0.3s) 期間以 `Collider2D.excludeLayers` 穿身；結束若重疊延後恢復碰撞（上限 0.3s，超時速度微推開；Boss 允許穿身）。
- **汽水工廠水機制**：配置可擊破閥門（冷卻 8s，場景 >= 2 個），打碎潑水。僅汽水系與 Boss 獲【淋濕】10s（護甲 -50%、移速 -25%、前搖速率 × 0.75；重觸發重置時間不疊加，戰後清除）。
- **手感參數 (`[SerializeField]`)**：完美格擋窗口 `0.20s`、閃避 CD `1.0s`、閃避無敵 `0.30s`、蓄力門檻 `0.25s`、撤退無敵 `3.0s`、輸入緩衝 `0.15s`。

## 6. UI 動態暫停與存檔世界狀態 (WorldState)
- **ESC 選單**：探索＝`繼續遊戲` → `設定` (音量/解析度) → `存檔` → `退出遊戲`；戰鬥＝`繼續遊戲` → `設定` → `戰鬥撤退` (強制戰反灰) → `退出遊戲`。
- **存檔槽位**：手動 1 格 (僅探索 ESC，戰鬥封鎖) + 自動 1 格 (正向抵達 Portal_D/E/F、Boss 勝後)；原子寫入 (Temp 覆蓋)。
- **讀取原則**：讀取手動/自動較新者（取代現有狀態，不合併）；皆無時清空 `WorldStateRegistry` 與 `StoryFlags` 重置為開新局狀態。
- **資料載體**：`SaveData` 存場景、座標/SpawnPoint ID、HP、物品欄、`WorldState`、`StoryFlags`。`PersistentId` 編輯器產生 GUID 且 `OnValidate` 警告防呆。

## 7. 對話系統架構與排程
- **排程與分期**：完整對話模組列為 Phase 4。Phase 1 建立 `StoryFlags`；Phase 2 建立 `IDialogueService` 介面（`StartDialogue`、`IsRunning`、`OnDialogueStarted`、`OnDialogueEnded`）與 `PlaceholderDialogueService`（簡易文字框，供 SC003/SC004 流程驗收）。
- **實作決策**：Phase 4 二選一（方案 A 自製 / 備案 B Yarn Spinner）。外部系統僅依賴 `IDialogueService`；對話時切換 `UI` Map、鎖定探索、禁 `timeScale=0`。

## 8. C# 腳本架構與 Phase 1 ~ Phase 4 開發排程藍圖
| 腳本名稱 | 資料夾目錄 | 核心職責 | 通訊方式 | 開發階段 |
|---|---|---|---|---|
| `GameBootstrap` | `Assets/Scripts/Core/` | 檢查並加載常駐單例、初始化存檔與加載啟動場景（支援任意場景直啟） | Direct Call / Unity Event | Phase 1 |
| `GameStateManager` | `Assets/Scripts/Core/` | 維護遊戲主狀態（探索 / 戰鬥 / 暫停），派發狀態切換事件 | C# Action Event | Phase 1 |
| `InputReader` | `Assets/Scripts/Core/` | 封裝新版 Input System，集中處理 Action Map 啟用與事件派發 | C# Action Event | Phase 1 |
| `SceneFlowController` | `Assets/Scripts/Core/` | 非同步場景流轉、7 步傳送時序、Additive 疊加/卸載 2D 戰鬥場景 | C# Action Event | Phase 1 |
| `SaveSystem` / `SaveData` | `Assets/Scripts/Core/` | 雙槽位 JSON 序列化、原子寫入檔案、提供自動與手動存檔 API | Direct Call | Phase 1 |
| `WorldStateRegistry` / `PersistentId` | `Assets/Scripts/Core/` | 註冊與比對場景實體存續狀態（GUID 防呆），開局重置與讀檔取代 | Direct Call / C# Action | Phase 1 |
| `PlayerStats` / `InventoryManager` | `Assets/Scripts/Core/` | 管理玩家 HP、補品數量、投擲物計數（上限 3）、裝備加成與跨場景數據 | C# Action / Direct Call | Phase 1 |
| `StoryFlags` | `Assets/Scripts/Core/` | 管理關鍵劇情道具（申請表、簽證）與進度旗標，納入 SaveData | C# Action / Direct Call | Phase 1 |
| `BattleEvents` | `Assets/Scripts/Core/` | 定義戰鬥開始、結束、撤退等跨模組解耦事件 | C# Action Event | Phase 1 |
| `Portal` / `SpawnPoint` | `Assets/Scripts/Exploration/` | 3D 場景雙向門與出生點 ID 綁定，觸發傳送與自動存檔 | C# Action (`SceneFlowController`) | Phase 1 |
| `ExplorationCameraRig` | `Assets/Scripts/Exploration/` | Cinemachine 3 俯視跟隨，處理瞬移通知 (`OnTargetObjectWarped`) 防飛行 | Direct Reference / C# Action | Phase 1 |
| `ExplorationController` (精簡版) | `Assets/Scripts/Exploration/` | 最小可行 3D WASD 移動與跳躍，供 Portal 與場景切換測試通道 | Listen `InputReader` | Phase 1 |
| `IDialogueService` | `Assets/Scripts/Core/` | 定義對話啟動、結束與狀態查詢之通用解耦介面 | C# Interface | Phase 2 |
| `PlaceholderDialogueService` | `Assets/Scripts/Dialogue/` | 簡易暫代文字框，供 SC003 葵妮與 SC004 守衛交表及開戰測試 | Implements `IDialogueService` | Phase 2 |
| `ExplorationController` (完整版) | `Assets/Scripts/Exploration/` | 完整 3D 移動手感、跳躍調校、物品拾取互動與 LMB 瞄準投擲邏輯 | Listen `InputReader` | Phase 2 |
| `Interactable` / `PickupItem` | `Assets/Scripts/Exploration/` | 3D 場景 F 鍵互動、物品/補品拾取判定與 WorldState 登記 | Direct Call / C# Action | Phase 2 |
| `ThrowableProjectile` | `Assets/Scripts/Exploration/` | 投擲物拋物線飛行、碰撞判定，命中敵人時觸發先制開戰 | Direct Call / Unity Event | Phase 2 |
| `EncounterTrigger` | `Assets/Scripts/Exploration/` | 偵測接觸與投擲命中，判定 3 種開戰型態並建立 `BattleContext` | Direct Call / C# Action | Phase 2 |
| `ExplorationFreezer` | `Assets/Scripts/Exploration/` | 進入戰鬥時凍結 3D 角色與敵實體，停用鏡頭與 AudioListener | Listen `BattleEvents` (Core) | Phase 2 |
| `CombatStateMachine` | `Assets/Scripts/Combat/` | 2D 橫向戰鬥狀態機（移動/跳躍/格擋/閃避/蓄力/輸入緩衝） | Listen `InputReader` | Phase 2 |
| `CombatDamageCalculator` | `Assets/Scripts/Combat/` | 傷害計算 `Max(1, Atk - Def)`、護甲削減、破甲狀態判定 | Direct Call / C# Action | Phase 2 |
| `BattleTransitionManager` | `Assets/Scripts/Combat/` | 控管進出戰鬥流程、建立 `PreBattleSnapshot`、結算與敗北還原 | Listen Events, Dispatch Scenes | Phase 2 |
| `BattleContext` / `PreBattleSnapshot`| `Assets/Scripts/Core/` | 資料載體：遇敵觸發類型、敵編隊陣列（上限 3）、戰前記憶體快照 | Pure C# Data Class | Phase 2 |
| `EnemyData` | `Assets/Scripts/Core/` | 敵人數值 ScriptableObject（HP、攻擊力、護甲、汽水標記、紅光攻擊） | ScriptableObject Asset | Phase 2 |
| `LootDropper` | `Assets/Scripts/Combat/` | 戰鬥勝利時計算並發放掉落物，寫入背包（不自動回血） | Direct Call / C# Action | Phase 2 |
| `EnemyTurnScheduler` | `Assets/Scripts/Combat/` | 2D 戰場輪流發放攻擊權，協調敵群攻擊順序以避免同幀重疊出手 | C# Action / Direct Call | Phase 3 |
| `BaseEnemyAI` / `EliteEnemyAI` | `Assets/Scripts/Combat/` | 小怪與精英怪行為邏輯（近戰揮擊、遠程投射物、格擋姿態、紅光衝刺） | Read `EnemyData`, Turn Scheduler | Phase 3 |
| `BossController` | `Assets/Scripts/Combat/` | 汽水怪 Boss 階段轉換、特殊紅光招式調度與不可撤退鎖定 | Listen `CombatDamageCalculator` | Phase 3 |
| `WaterValveMechanism` | `Assets/Scripts/Combat/` | 工廠可擊破水管閥門（冷卻 8s），潑水賦予汽水系 10 秒淋濕 Debuff | Direct Call / C# Action | Phase 3 |
| `PauseMenuController` | `Assets/Scripts/UI/` | ESC 暫停選單（動態切換存檔/撤退按鈕，音量與解析度設定） | Listen `InputReader`, Call Core | Phase 3 |
| `BattleHUDController` | `Assets/Scripts/UI/` | 2D 戰鬥 HUD（血條、補品數量、淋濕狀態、按鍵提示） | Listen Combat Events | Phase 3 |
| `DialogueModule` (方案 A 或 B) | `Assets/Scripts/Dialogue/` | 完整對話系統（分支選項、條件判斷、UI 表現；自製或 Yarn Spinner） | Implements `IDialogueService` | Phase 4 |

## 9. 附錄：Phase 1 驗收標準與待確認預設值
- **Phase 1 驗收標準**：
  1. `SC001`～`SC004` 任一場景直接按 Play 皆可自動加載 Bootstrap 並將玩家對齊預設 `SpawnPoint`。
  2. `SC001` ↔ `SC002` 往返傳送不重複觸發，Cinemachine 鏡頭無阻尼飛行瞬移。
  3. 手動存檔、自動存檔、「繼續遊戲」讀取較新者雙向驗證無誤。
  4. 開新局狀態正確（背包含【簽證申請表】、投擲上限 3、旗標與世界狀態皆為初始值）。
- **待確認事項預設值彙整**：
  | 項目 | 預設決策與數值 |
  |---|---|
  | 戰鬥中途關閉遊戲之存檔行為 | 接受回到最近一次有效存檔（記憶體快照不落盤） |
  | 蓄力重擊傷害倍率 | 蓄力重擊傷害 = Max(1, Atk - Def) × 2.0（先扣護甲，再乘倍率） |
  | Boss 是否允許閃避穿身 | 允許，若無敵結束重疊則延後恢復碰撞（上限 0.3s 後速度微推開） |
  | 閥門冷卻與戰場數量 | 冷卻 8 秒，每個 Boss 戰場至少配置 2 個閥門 |
  | 隱藏房裝備數量與數值 | 兩件獨立道具：攻擊 +5、護甲 +3（拾取即生效，無額外 UI） |
  | 暈眩持續秒數 | 待負責人決定（精英暈眩時間為一般的一半） |
  | Boss 戰時長實測 | 淋濕後每刀 9 傷（15 - 12×50% = 9），約需 34 刀；以每個淋濕窗口 6～8 刀估算，需 4～5 次淋濕窗口；未計入裝備加成，需實機測試後調整閥門數量與冷卻 |
