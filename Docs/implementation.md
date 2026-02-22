# UE5実装仕様

このドキュメントは実装者がBlueprintを組み立てる際に参照する詳細仕様です。  
アーキテクチャの方針については [architecture.md](architecture.md) を、ドメインの概念については [domain.md](domain.md) を参照してください。

---

## フォルダ階層

```
Content/
│
├── Domain/                              # ①ドメイン層（UEフレームワーク非依存）
│   ├── Gameplay/
│   │   ├── E_GameState.uasset           # Enum: Playing / Paused / GameOver
│   │   ├── E_BlockState.uasset          # Enum: Falling / Landed / FallenOff
│   │   └── DT_BlockShapes.uasset        # DataTable: ブロック形状定義
│   └── Ranking/
│       └── S_PlayRecord.uasset          # Struct: ユーザ名 / 日時 / スコア
│
├── DomainService/                       # ②ドメインサービス層（Pure関数のみ・状態なし）
│   ├── BFL_ScoreCalculator.uasset       # 着地ブロック群 → 積み上げ高さを算出
│   └── BFL_RankingRule.uasset           # スコア降順ソート・10件トリムのルール
│
├── Application/                         # ③アプリケーション層
│   └── Interfaces/
│       └── BPI_RankingRepository.uasset # Load / Save / GetTopTen の契約（DIP）
│
├── Infrastructure/                      # ④インフラ層（UE5具体実装）
│   ├── Persistence/
│   │   ├── BP_SaveGame_Ranking.uasset   # SaveGame 実体（ディスク永続化）
│   │   └── BP_RankingRepository.uasset  # BPI_RankingRepository を実装
│   └── BP_GameInstance.uasset           # Composition Root・レベル間状態管理
│
├── Presentation/                        # ⑤プレゼンテーション層（最外層）
│   ├── Gameplay/
│   │   ├── Actors/
│   │   │   ├── Plate/
│   │   │   │   ├── BP_Plate.uasset              # 皿（集約ルート相当 Actor）
│   │   │   │   └── BP_FallDetector.uasset       # 落下検知トリガー
│   │   │   ├── Block/
│   │   │   │   ├── BP_Block.uasset              # ブロック（物理有効 Actor）
│   │   │   │   └── BP_BlockSpawner.uasset       # ブロック生成 Actor
│   │   │   └── BP_GenerationMarker.uasset       # 生成場所インジケーター Actor
│   │   └── Framework/
│   │       ├── BP_GM_Gameplay.uasset            # Application Service 兼務 ★中核
│   │       ├── BP_GS_Gameplay.uasset            # ゲーム状態保持
│   │       └── BP_PC_Gameplay.uasset            # 入力処理
│   ├── Top/
│   │   ├── BP_GM_Top.uasset
│   │   └── WBP_TopScreen.uasset                 # トップ画面（スタート・スコアボタン）
│   ├── Score/
│   │   ├── BP_GM_Score.uasset
│   │   ├── WBP_ScoreScreen.uasset               # スコア画面（ランキング一覧）
│   │   └── WBP_RankingRow.uasset                # ランキング1行 Widget
│   └── Shared/
│       ├── WBP_GameHUD.uasset                   # ゲーム中HUD（リアルタイム高さ表示）
│       └── WBP_PauseMenu.uasset                 # Pose中メニュー
│
├── Levels/
│   ├── L_Top.umap                       # トップ画面レベル
│   ├── L_Gameplay.umap                  # ゲームプレイレベル（3D物理ワールド）
│   └── L_Score.umap                     # スコア画面レベル
│
└── Assets/
    ├── Meshes/
    │   ├── SM_Block_A.uasset            # ブロック形状A（凸型コリジョン付き）
    │   ├── SM_Block_B.uasset
    │   ├── SM_Block_C.uasset
    │   └── SM_Plate.uasset              # 皿メッシュ
    ├── Materials/
    └── Sounds/
```

---

## レベル構成と画面遷移

```
L_Top  ────[スタートボタン]────▶  L_Gameplay
  ▲                                    │
  │         [Pose → トップに戻る]       │ ゲームオーバー
  └────────────────────────────────────┘
  ▲                                    │
  └──────────────[戻るボタン]──── L_Score ◀──┘
```

レベル間の状態受け渡しは `BP_GameInstance` 経由で行います。`LastScore: Float` にゲームオーバー時スコアを格納し、L_Score に渡します。L_Top / L_Score は 3D ワールド不要のため最小レベルとし、BeginPlay で対応 Widget を ViewPort に Add するだけの構成にします。

---

## Blueprint 責務詳細

### ② DomainService

#### BFL_ScoreCalculator（Function Library）

```
CalcStackHeight(LandedBlocks: Array<BP_Block>, PlateTopZ: Float) → Float
```

着地済みブロック全件をループして MaxZ を取得し、`MaxZ − PlateTopZ` を返します。  
呼び出し元は着地済み（Landed）ブロックのみを渡す責務を持ちます。

---

#### BFL_RankingRule（Function Library）

```
ApplyRule(Records: Array<S_PlayRecord>, NewRecord: S_PlayRecord) → Array<S_PlayRecord>
```

Records に NewRecord を追加 → Score 降順ソート → 先頭10件にトリム → 返します。

---

### ③ Application / Interfaces

#### BPI_RankingRepository（Blueprint Interface）

```
LoadRanking()
SaveRanking(Records: Array<S_PlayRecord>)
AddRecord(UserName: String, Score: Float)
GetTopTen() → Array<S_PlayRecord>
```

---

### ④ Infrastructure

#### BP_GameInstance（Composition Root）

変数

| 変数名 | 型 | 用途 |
|---|---|---|
| Repository | Object（BPI_RankingRepository として参照） | リポジトリDI |
| LastScore | Float | レベル間スコア受け渡し用 |

BeginPlay で `BP_RankingRepository` を CreateObject して `Repository` にセットし、`Repository.LoadRanking()` を呼び出します。

---

#### BP_RankingRepository（BPI_RankingRepository を実装）

変数: `Records: Array<S_PlayRecord>`

`LoadRanking()` — `LoadGameFromSlot("Ranking", 0)` で Records を読み込みます。

`AddRecord(UserName: String, Score: Float)` — `Now()` で日時を取得して `S_PlayRecord` を作成し、`BFL_RankingRule.ApplyRule` で Records を更新後、`SaveRanking()` を呼び出します。

`SaveRanking()` — `CreateSaveGameObject(BP_SaveGame_Ranking)` でオブジェクトを作成し、`SaveGameToSlot("Ranking", 0)` で書き込みます。

`GetTopTen() → Array<S_PlayRecord>` — Records をそのまま返します（最大10件）。

---

#### BP_SaveGame_Ranking（SaveGame）

変数: `Records: Array<S_PlayRecord>`

---

### ⑤ Presentation / Framework

#### BP_GM_Gameplay ★ Application Service 兼務・ゲームプレイコンテキストの不変条件を守る中核

**BeginPlay**
- BP_Plate / BP_FallDetector / BP_BlockSpawner / BP_GenerationMarker を Spawn・配置
- WBP_GameHUD を Viewport に追加

**入力: `DropBlock(X)`**
- BlockSpawner.SpawnBlock(X) 呼び出し

**入力: `TogglePause`**
- SetGamePaused(true) → WBP_PauseMenu を表示

**イベント: `OnBlockFallenOff(Block)`** ← BP_FallDetector から通知
1. `BFL_ScoreCalculator.CalcStackHeight(LandedBlocks, Plate.GetTopZ())` でスコア取得
2. `GameInstance.LastScore = Score`
3. `GS_Gameplay.GameState = GameOver`
4. 名前入力ダイアログ（Widget）を表示
5. 入力完了後 `GameInstance.Repository.AddRecord(UserName, Score)`
6. `OpenLevel(L_Score)`

> ドメインロジック（スコア算出・ランキングルール）は自身に持たず、DomainService（BFL_*）と Repository（BPI_*）へ委譲します。

---

#### BP_GS_Gameplay

変数

| 変数名 | 型 |
|---|---|
| GameState | E_GameState（Playing / Paused / GameOver） |
| CurrentHeight | Float（リアルタイム積み上げ高さ） |

EventDispatcher: `OnGameStateChanged(NewState: E_GameState)`

---

#### BP_PC_Gameplay

マウス左クリック / タッチ — Pose中でないことを確認したうえで、`GetHitResultUnderCursor`（Channel: Visibility）で HitLocation.X を取得し、`GM_Gameplay.DropBlock(X)` に渡します。Pose中の場合は入力を無視します。

Pキー / 一時停止ボタン — `GM_Gameplay.TogglePause` を呼び出します。

---

### ⑤ Presentation / Actors

#### BP_Plate

コンポーネント: `StaticMeshComponent`（SM_Plate）、`BoxComponent`（有効範囲）

関数

| 関数名 | 戻り値 | 説明 |
|---|---|---|
| GetTopZ() | Float | 皿の表面Z座標を返す |
| GetBounds() | FBox | 水平方向の有効範囲を返す |

---

#### BP_FallDetector

コンポーネント: `BoxTriggerComponent`（皿の真下・画面外に配置）

`OnComponentBeginOverlap(OtherActor)` — OtherActor が BP_Block であれば `GM_Gameplay.OnBlockFallenOff(Block)` を通知します。

---

#### BP_Block

コンポーネント: `StaticMeshComponent`（DT_BlockShapes からランダム選択した SM_Block_\*）。BeginPlay で SetStaticMesh / SetSimulatePhysics(true) を呼び出します。

変数: `BlockState: E_BlockState（Falling / Landed / FallenOff）`

`OnComponentHit(HitResult)` — `BlockState = Landed` に更新し、GM_Gameplay に着地を通知します。

設定上の注意点として、`GenerateHitEvents = true`、`Collision Preset = PhysicsActor` にしてください。

---

#### BP_BlockSpawner

関数: `SpawnBlock(X: Float)`
1. DT_BlockShapes から PickRandomRow
2. 位置 `(X, 0, SpawnHeight)` に BP_Block を SpawnActor
3. Block.SetStaticMesh(Row.Mesh)
4. Block.SetSimulatePhysics(true)

---

#### BP_GenerationMarker

Tick で `PC.GetHitResultUnderCursor` の HitLocation.X を取得し、`SetActorLocation(X, 0, SpawnHeight)` でマーカーをカーソルに追従させます。次のブロック落下位置をプレイヤーに視覚的に提示する役割を持ちます。

---

### ⑤ Presentation / UI

#### WBP_TopScreen
- スタートボタン OnClicked → `OpenLevel(L_Gameplay)`
- スコアボタン OnClicked → `OpenLevel(L_Score)`

#### WBP_PauseMenu
- 再開ボタン OnClicked → `SetGamePaused(false)` → 自身を RemoveFromParent
- トップに戻るボタン OnClicked → `SetGamePaused(false)` → `OpenLevel(L_Top)`

#### WBP_GameHUD
- Bind: Text_Height → `GS_Gameplay.CurrentHeight` をリアルタイム表示

#### WBP_ScoreScreen

BeginPlay で `GameInstance.Repository.GetTopTen()` を呼び出し、ForEach で WBP_RankingRow を生成して ScrollBox に追加します。戻るボタンは `OpenLevel(L_Top)` を呼び出します。

#### WBP_RankingRow

外部からセットされる変数: `Rank: Integer`、`UserName: String`、`DateTime: String`、`Score: Float`

---

## 主要データフロー

### ブロック生成フロー

```
PlayerController
  → GetHitResultUnderCursor → X座標取得
  → GM_Gameplay.DropBlock(X)
  → BlockSpawner.SpawnBlock(X)
      DT_BlockShapes PickRandomRow
      SpawnActor BP_Block at (X, 0, SpawnHeight)
      SetSimulatePhysics(true)
  → Chaos Physics が落下・衝突を計算（自動）
  → BP_Block.OnComponentHit → BlockState = Landed
```

### ゲームオーバー → スコア保存フロー

```
BP_Block（落下済み）
  → BP_FallDetector.OnOverlap
  → GM_Gameplay.OnBlockFallenOff
      BFL_ScoreCalculator.CalcStackHeight(LandedBlocks, Plate.GetTopZ())
                                         … ②DomainService呼び出し
      GameInstance.LastScore = Score
      GS_Gameplay.GameState = GameOver
      名前入力ダイアログ表示（Widget）
  → ユーザが UserName 入力・確定
  → GameInstance.Repository.AddRecord(UserName, Score)
                                         … ③BPI経由でInfraへ
      BFL_RankingRule.ApplyRule(Records, NewRecord)
                                         … ②DomainService呼び出し
      SaveGameToSlot("Ranking", 0)
  → OpenLevel(L_Score)
```

### スコア画面表示フロー

```
L_Score BeginPlay
  → WBP_ScoreScreen を Viewport に追加
  → GameInstance.Repository.GetTopTen()  … ③BPI経由でInfraへ
  → 最大10件の S_PlayRecord を WBP_RankingRow に渡して表示
```

---

## DT_BlockShapes 定義（DataTable）

Row Struct: `FBlockShapeRow`（FTableRowBase 継承）

| フィールド | 型 | 説明 |
|---|---|---|
| Mesh | StaticMesh参照 | ブロックのメッシュ |
| Mass | Float | 重量（kg）物理挙動の調整用 |
| Label | String | デバッグ用ラベル |

行を追加するだけで形状を増やせます。BP_Block / BP_BlockSpawner のロジック変更は不要です。

---

## 設計上の注意点

**依存の向きを守る**  
`Domain/` と `DomainService/` は `Presentation/` や `Infrastructure/` を参照しません。BFL_* は引数で値を受け取り結果を返すだけにとどめます。

**BPI_RankingRepository 経由を徹底する**  
`BP_GM_Gameplay` は `BP_RankingRepository` を直接参照しません。`GameInstance.Repository`（BPI_RankingRepository）として扱います。これにより Infrastructure の差し替え（例: オンラインサービスへの移行）が可能になります。

**Pause と Chaos Physics**  
`SetGamePaused(true)` はグローバル一時停止のため、ブロック落下・積み上げ物理が自動で停止します。再開は `SetGamePaused(false)` で復帰します。

**スコア計算タイミング**  
ゲームオーバー直後に `BFL_ScoreCalculator` を呼びます。落下中ブロックは除外し、着地済み（Landed）ブロックのみを渡す責務は GM_Gameplay が持ちます。

**コンテキスト境界の維持**  
BP_Block や BP_Plate は `Ranking/` 配下を直接参照しません。GameMode を経由して GameInstance に渡します。