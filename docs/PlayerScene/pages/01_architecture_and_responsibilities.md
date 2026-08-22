# 01. 全体設計と責務

[目次へ](../README.md) ｜ [次章：プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md)

## 最初に固定する要件

新しいPlayerは、少なくとも次を満たすものとします。

1. Idle、Walk、Runのうち、ロコモーションStateは常に1つだけ有効である。
2. AnimatedSprite3Dは、Inspectorで設定した1つのSpriteFramesだけを使用する。
3. 移動入力はカメラ基準だが、Motorが受け取る時点ではワールド方向へ変換済みである。
4. キャラクターの向きはワールド方向として保持し、表示時だけカメラ基準の上下左右へ変換する。
5. カメラが回転したら、PlayerがIdleのままでも表示方向を更新する。
6. CharacterBody3Dの`move_and_slide()`は、1物理フレームに1回だけ呼ぶ。
7. Player専用入力をAI用Intentへ交換しても、State、Motor、Facing、方向別Animatorは変更しない。

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

### 表示はStateにしない

画面上のDown、Left、Right、Upは、ロコモーションのような振る舞いではなく、ワールド上の向きとCamera3Dから求める表示結果です。そのため、`IdleDown`や`WalkLeft`のようなStateは作りません。

LocomotionStateは`idle`、`walk`、`run`という論理アニメーション名だけを要求します。DirectionalSpriteAnimator3DがCameraRelativeFacing3Dの結果を組み合わせ、単一のSpriteFramesから`idle_down`や`walk_left`を選びます。

~~~text
避ける分割
LocomotionStateMachine
├── IdleDown
├── IdleUp
├── WalkDown
├── WalkUp
└── ...動作と方向の組み合わせが増え続ける

採用する分割
LocomotionStateMachine = Idle / Walk / Run
Facing                = Down / Left / Right / Up
Animator              = logical_animation + facing
~~~

SpriteFramesはAnimatedSprite3DのInspectorへ1つだけ割り当て、実行中には差し替えません。見た目を切り替えるためのID、Resource一覧、シグナル、フォールバック規則は初期実装へ含めません。

## コンポーネントの責務

| 要素 | 担当すること | 担当しないこと |
|---|---|---|
| Player | 初期化、依存の注入、毎フレームの呼び出し順 | 個別Stateの遷移条件 |
| StateMachine | active_state、exit・切替・enterの順序 | Input読取り、アニメーション選択 |
| LocomotionState | Idle・Walk・Run固有の判断、遷移要求、移動速度と論理アニメーションの選択 | `move_and_slide()`、方向判定 |
| MovementIntentSource | 移動方向、run・jump・dashの操作意図 | 接地判定、実行可否、速度適用 |
| PlayerMovementIntent | Inputを読み、カメラ基準入力をワールド方向へ変換 | CharacterBody3Dの移動 |
| CharacterMotor3D | 加減速、重力、velocity、`move_and_slide()` | Input、アニメーション、State遷移 |
| CameraRelativeFacing3D | ワールド向きの保持、Camera3Dから見た上下左右の算出 | SpriteFrames、移動速度 |
| DirectionalSpriteAnimator3D | 論理動作と表示方向から実アニメーション名を解決・再生 | SpriteFramesの差替え、ロコモーション遷移、物理移動 |

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
    StateNode -->|logical animation| AnimatorNode[DirectionalSpriteAnimator3D]
    FacingNode --> AnimatorNode
    AnimatorNode -->|play animation| SpriteNode[AnimatedSprite3D]
    FramesNode[Single SpriteFrames] --> SpriteNode
~~~

ここで重要なのは、カメラを知る場所が2つに限定されることです。

- PlayerMovementIntentは、画面基準の入力をワールド移動方向へ変換するためにCamera3Dを見る。
- CameraRelativeFacing3Dは、ワールド向きを画面上の上下左右へ変換するためにCamera3Dを見る。

State、Motor、AnimatedSprite3DはCamera3Dを知りません。

## Scene Tree

最初の完成形は、次のNode構成にします。

~~~text
Player (CharacterBody3D) [PlayerCharacter]
├── CollisionShape3D
├── AnimatedSprite3D
├── PlayerMovementIntent (Node) [PlayerMovementIntent]
├── CharacterMotor3D (Node) [CharacterMotor3D]
├── CameraRelativeFacing3D (Node) [CameraRelativeFacing3D]
├── DirectionalSpriteAnimator3D (Node) [DirectionalSpriteAnimator3D]
└── LocomotionStateMachine (Node) [StateMachine]
    ├── Idle (Node) [CharacterIdleState]
    ├── Walk (Node) [CharacterWalkState]
    └── Run (Node) [CharacterRunState]
~~~

`LocomotionStateMachine`直下にはStateだけを置きます。Motorや方向別AnimatorをStateMachineの子に混在させません。

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
│       └── directional_sprite_animator_3d.gd
├── player/
│   ├── player.gd
│   ├── player.tscn
│   └── player_movement_intent.gd
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
    participant AnimatorNode as DirectionalAnimator

    GodotEngine->>PlayerNode: _physics_process(delta)
    PlayerNode->>MachineNode: physics_update(delta)
    MachineNode->>StateNode: physics_update(delta)
    StateNode->>MotorNode: command_move or command_stop
    PlayerNode->>MotorNode: physics_step(delta)
    MotorNode-->>PlayerNode: move_and_slide完了
    GodotEngine->>PlayerNode: _process(delta)
    PlayerNode->>MachineNode: update(delta)
    PlayerNode->>AnimatorNode: refresh()
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
        &"animator": directional_animator,
    }
)
~~~

NPCでは`movement_intent`だけをNavigationMovementIntentへ差し替えます。Stateから固定NodePathでPlayerを探さないため、共有Stateのコード変更は不要です。

## 所有権の規則

実装中に迷ったら、次の規則へ戻ります。

- active_stateを書き換えるのはStateMachineだけ。
- `move_and_slide()`を呼ぶのはCharacterMotor3Dだけ。
- ワールド上の最後の向きを持つのはCameraRelativeFacing3Dだけ。
- AnimatedSprite3Dの`animation`を変更するのはDirectionalSpriteAnimator3Dだけ。
- AnimatedSprite3Dの`sprite_frames`はInspectorで1つだけ設定し、実行時コードから変更しない。
- Inputを直接読むのはPlayerMovementIntentだけ。
- Camera3Dを参照するのはPlayerMovementIntentとCameraRelativeFacing3Dだけ。

---

[目次へ](../README.md) ｜ **次のページ:** [02. プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md)
