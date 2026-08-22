# 01. 全体設計と責務

[目次へ](../README.md) ｜ [次章：プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md)

## 最初に固定する要件

新しいPlayerは、少なくとも次を満たすものとします。

1. Idle、Walk、Runのうち、ロコモーションStateは常に1つだけ有効である。
2. Outdoor、Indoor、Nakedのうち、衣服は常に1つ選択されている。
3. ロコモーションと衣服は独立して変更できる。
4. 移動入力はカメラ基準だが、Motorが受け取る時点ではワールド方向へ変換済みである。
5. キャラクターの向きはワールド方向として保持し、表示時だけカメラ基準の上下左右へ変換する。
6. カメラが回転したら、PlayerがIdleのままでも表示方向を更新する。
7. CharacterBody3Dの`move_and_slide()`は、1物理フレームに1回だけ呼ぶ。
8. Player専用入力をAI用Intentへ交換しても、State、Motor、Facing、Visualは変更しない。

## 状態軸を分ける

### ロコモーション

Idle、Walk、Runは、同じ時点に1つだけ有効になる排他的な振る舞いです。そのためNode型StateMachineを使います。

~~~mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Walk: 移動入力あり
    Walk --> Idle: 移動入力なし
    Walk --> Run: run入力あり
    Run --> Walk: run入力なし
    Run --> Idle: 移動入力なし
~~~

### 衣服

Outdoor、Indoor、Nakedは、WalkやRunと同時に成立します。`WalkOutdoor`、`RunOutdoor`のような組み合わせStateは作りません。

衣服変更に時間、キャンセル、装着中アニメーションなどのゲームプレイ規則がない初期段階では、衣服はCharacterVisual3Dが保持する永続的な表示状態です。`CharacterOutfit` Resourceを登録し、現在のIDだけを切り替えます。

~~~text
悪い分割
LocomotionStateMachine
├── IdleOutdoor
├── WalkOutdoor
├── RunOutdoor
├── IdleIndoor
└── ...組み合わせが増え続ける

採用する分割
LocomotionStateMachine = Idle / Walk / Run
CharacterVisual3D      = outdoor / indoor / naked
~~~

着替えそのものが数秒かかり、途中でキャンセル可能になる場合は、将来`OutfitActionStateMachine`を別軸として追加します。その場合も、現在装着中の衣服データはCharacterVisual3DまたはEquipmentコンポーネントに残します。

## コンポーネントの責務

| 要素 | 担当すること | 担当しないこと |
|---|---|---|
| Player | 初期化、依存の注入、毎フレームの呼び出し順 | 個別Stateの遷移条件 |
| StateMachine | active_state、exit・切替・enterの順序 | Input読取り、SpriteFrames選択 |
| LocomotionState | Idle・Walk・Run固有の判断、遷移要求、移動速度の選択 | `move_and_slide()`、衣服切替 |
| MovementIntentSource | 移動方向、run・jump・dashの操作意図 | 接地判定、実行可否、速度適用 |
| PlayerMovementIntent | Inputを読み、カメラ基準入力をワールド方向へ変換 | CharacterBody3Dの移動 |
| CharacterMotor3D | 加減速、重力、velocity、`move_and_slide()` | Input、アニメーション、State遷移 |
| CameraRelativeFacing3D | ワールド向きの保持、Camera3Dから見た上下左右の算出 | SpriteFrames、移動速度 |
| CharacterVisual3D | 論理動作、表示方向、衣服から実アニメーションを解決 | ロコモーション遷移、物理移動 |
| CharacterOutfit | 衣服ID、SpriteFrames、フォールバック名 | 現在のState、再生制御 |

## 情報の流れ

物理フレームでは、入力から物理移動までを一方向に流します。

~~~mermaid
flowchart TD
    InputNode[Input] --> IntentNode[PlayerMovementIntent]
    CameraNode[Current Camera3D] --> IntentNode
    IntentNode --> StateNode[Active LocomotionState]
    StateNode -->|world move vector| FacingNode[CameraRelativeFacing3D]
    StateNode -->|move command| MotorNode[CharacterMotor3D]
    MotorNode -->|velocity and move_and_slide| BodyNode[Player CharacterBody3D]
    StateNode -->|logical animation| VisualNode[CharacterVisual3D]
    FacingNode --> VisualNode
    OutfitNode[CharacterOutfit] --> VisualNode
    VisualNode --> SpriteNode[AnimatedSprite3D]
~~~

ここで重要なのは、カメラを知る場所が2つに限定されることです。

- PlayerMovementIntentは、画面基準の入力をワールド移動方向へ変換するためにCamera3Dを見る。
- CameraRelativeFacing3Dは、ワールド向きを画面上の上下左右へ変換するためにCamera3Dを見る。

State、Motor、CharacterOutfitはCamera3Dを知りません。

## Scene Tree

最初の完成形は、次のNode構成にします。

~~~text
Player (CharacterBody3D) [PlayerCharacter]
├── CollisionShape3D
├── AnimatedSprite3D
├── PlayerMovementIntent (Node) [PlayerMovementIntent]
├── CharacterMotor3D (Node) [CharacterMotor3D]
├── CameraRelativeFacing3D (Node) [CameraRelativeFacing3D]
├── CharacterVisual3D (Node) [CharacterVisual3D]
└── LocomotionStateMachine (Node) [StateMachine]
    ├── Idle (Node) [CharacterIdleState]
    ├── Walk (Node) [CharacterWalkState]
    └── Run (Node) [CharacterRunState]
~~~

`LocomotionStateMachine`直下にはStateだけを置きます。MotorやVisualをStateMachineの子に混在させません。

## 推奨フォルダ構成

実装時は、Player固有部分と共有部分をファイル配置でも区別します。

~~~text
res://entities/character/
├── shared/
│   ├── state_machine/
│   │   ├── state.gd
│   │   └── state_machine.gd
│   ├── movement/
│   │   ├── movement_intent_source.gd
│   │   └── character_motor_3d.gd
│   ├── locomotion/
│   │   ├── locomotion_state.gd
│   │   ├── character_idle_state.gd
│   │   ├── character_walk_state.gd
│   │   └── character_run_state.gd
│   └── visual/
│       ├── camera_relative_facing_3d.gd
│       ├── character_outfit.gd
│       └── character_visual_3d.gd
├── player/
│   ├── player.gd
│   ├── player.tscn
│   ├── player_movement_intent.gd
│   └── outfits/
│       ├── outdoor.tres
│       ├── indoor.tres
│       └── naked.tres
└── npc/
    └── navigation_movement_intent.gd
~~~

`shared`内のクラス名へ`Player`を付けないことが再利用の第一歩です。Player固有なのは、Inputを読むIntentとPlayerシーンの組み立てだけです。

## Playerが決める更新順

StateMachine自身には`_process()`や`_physics_process()`を実装せず、Playerから明示的に呼びます。

~~~mermaid
sequenceDiagram
    participant GodotEngine as Godot
    participant PlayerNode as Player
    participant MachineNode as Locomotion
    participant StateNode as ActiveState
    participant MotorNode as Motor
    participant VisualNode as Visual

    GodotEngine->>PlayerNode: _physics_process(delta)
    PlayerNode->>MachineNode: physics_update(delta)
    MachineNode->>StateNode: physics_update(delta)
    StateNode->>MotorNode: command_move or command_stop
    PlayerNode->>MotorNode: physics_step(delta)
    MotorNode-->>PlayerNode: move_and_slide完了
    GodotEngine->>PlayerNode: _process(delta)
    PlayerNode->>MachineNode: update(delta)
    PlayerNode->>VisualNode: refresh()
~~~

この順序により、Stateが決めた移動方針を同じ物理フレームの最後にMotorが1回だけ実行します。将来Action用StateMachineを追加しても、最終的な`move_and_slide()`の所有者はMotorのままです。

## 依存性注入

PlayerはLocomotionStateMachineの初期化時に次を渡します。

~~~gdscript
locomotion.initialize(
    self,
    {
        &"movement_intent": movement_intent,
        &"motor": motor,
        &"facing": facing,
        &"visual": visual,
    }
)
~~~

NPCでは`movement_intent`だけをNavigationMovementIntentへ差し替えます。Stateから固定NodePathでPlayerを探さないため、共有Stateのコード変更は不要です。

## 所有権の規則

実装中に迷ったら、次の規則へ戻ります。

- active_stateを書き換えるのはStateMachineだけ。
- `move_and_slide()`を呼ぶのはCharacterMotor3Dだけ。
- ワールド上の最後の向きを持つのはCameraRelativeFacing3Dだけ。
- AnimatedSprite3Dの`animation`と`sprite_frames`を変更するのはCharacterVisual3Dだけ。
- 現在の衣服IDを持つのはCharacterVisual3Dだけ。
- Inputを直接読むのはPlayerMovementIntentだけ。
- Camera3Dを参照するのはPlayerMovementIntentとCameraRelativeFacing3Dだけ。

---

[目次へ](../README.md) ｜ **次のページ:** [02. プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md)
