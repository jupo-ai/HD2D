# 05. Idle・Walk・Runの実装例

[前章：移動コンポーネントの設計](04_movement_components.md) ｜ [目次へ](../README.md) ｜ [次章：拡張パターン](06_extensions_and_composition.md)

**関連する参考動画:** [HowToMakeAStateMachine.mp4](../HowToMakeAStateMachine.mp4)、[AdvancedStateMachine.mp4](../AdvancedStateMachine.mp4)

## この例のStateグラフ

~~~mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Walk: move_vector != ZERO
    Walk --> Idle: move_vector == ZERO
    Walk --> Run: wants_run
    Run --> Idle: move_vector == ZERO
    Run --> Walk: not wants_run
~~~

この章では、第2章のStateとStateMachine、第4章のMovementIntentSourceとCharacterMotor3Dを組み合わせます。

## 1. Player専用の型付き基底State

汎用StateはactorをNode、依存をDictionaryとして受け取ります。具体Stateで毎フレームDictionaryを検索しないよう、PlayerStateで型付き参照へ変換します。

~~~gdscript
class_name PlayerState
extends State

@export var animation_name: StringName

var body: CharacterBody3D
var movement_intent: MovementIntentSource
var motor: CharacterMotor3D
var visual: AnimatedSprite3D


func setup(
    target_actor: Node,
    injected_dependencies: Dictionary
) -> void:
    super(target_actor, injected_dependencies)

    body = target_actor as CharacterBody3D
    movement_intent = injected_dependencies.get(
        &"movement_intent"
    ) as MovementIntentSource
    motor = injected_dependencies.get(
        &"motor"
    ) as CharacterMotor3D
    visual = injected_dependencies.get(
        &"visual"
    ) as AnimatedSprite3D

    assert(body != null, "CharacterBody3Dが必要です")
    assert(
        movement_intent != null,
        "MovementIntentSourceが必要です"
    )
    assert(motor != null, "CharacterMotor3Dが必要です")


func enter(
    _previous_state: State,
    _data: Dictionary = {}
) -> void:
    if (
        visual != null
        and visual.sprite_frames != null
        and visual.sprite_frames.has_animation(animation_name)
    ):
        visual.play(animation_name)
~~~

服装や向きから実アニメーション名を決めるVisualコンポーネントがある場合は、AnimatedSprite3Dを直接注入せず、そのVisual型を注入します。

## 2. IdleState

~~~gdscript
class_name PlayerIdleState
extends PlayerState

@export var walk_state: State


func physics_update(delta: float) -> void:
    var move_vector: Vector2 = (
        movement_intent.get_move_vector()
    )

    if not move_vector.is_zero_approx():
        request_transition(
            walk_state,
            {&"move_vector": move_vector}
        )
        return

    motor.move(Vector2.ZERO, 0.0, delta)
~~~

入力があればWalkを要求し、このStateの処理を終了します。入力がなければMotorを減速させ、必要な衝突更新を行います。

## 3. WalkState

~~~gdscript
class_name PlayerWalkState
extends PlayerState

@export var idle_state: State
@export var run_state: State
@export var walk_speed: float = 2.5


func physics_update(delta: float) -> void:
    var move_vector: Vector2 = (
        movement_intent.get_move_vector()
    )

    if move_vector.is_zero_approx():
        motor.move(Vector2.ZERO, 0.0, delta)
        request_transition(idle_state)
        return

    if movement_intent.wants_run():
        request_transition(
            run_state,
            {&"move_vector": move_vector}
        )
        return

    motor.move(move_vector, walk_speed, delta)
~~~

遷移条件を先に判定し、遷移要求後は必ずreturnします。停止遷移では、そのフレームから減速を開始してからIdleへ入れています。

## 4. RunState

~~~gdscript
class_name PlayerRunState
extends PlayerState

@export var idle_state: State
@export var walk_state: State
@export var run_speed: float = 5.0


func physics_update(delta: float) -> void:
    var move_vector: Vector2 = (
        movement_intent.get_move_vector()
    )

    if move_vector.is_zero_approx():
        motor.move(Vector2.ZERO, 0.0, delta)
        request_transition(idle_state)
        return

    if not movement_intent.wants_run():
        request_transition(
            walk_state,
            {&"move_vector": move_vector}
        )
        return

    motor.move(move_vector, run_speed, delta)
~~~

## 5. Scene TreeとInspectorの設定

~~~text
Player
├── Visual
├── MovementIntent
├── CharacterMotor3D
└── LocomotionStateMachine
    ├── Idle
    ├── Walk
    └── Run
~~~

設定例:

| Node | プロパティ | 値 |
|---|---|---|
| LocomotionStateMachine | initial_state | Idle |
| Idle | walk_state | Walk |
| Idle | animation_name | idle |
| Walk | idle_state | Idle |
| Walk | run_state | Run |
| Walk | walk_speed | 2.5 |
| Walk | animation_name | walk |
| Run | idle_state | Idle |
| Run | walk_state | Walk |
| Run | run_speed | 5.0 |
| Run | animation_name | run |
| CharacterMotor3D | body | Player |

## 6. Actorから初期化する

~~~gdscript
extends CharacterBody3D

@onready var locomotion: StateMachine = (
    $LocomotionStateMachine as StateMachine
)
@onready var movement_intent: MovementIntentSource = (
    $MovementIntent as MovementIntentSource
)
@onready var motor: CharacterMotor3D = (
    $CharacterMotor3D as CharacterMotor3D
)
@onready var visual: AnimatedSprite3D = (
    $Visual as AnimatedSprite3D
)


func _ready() -> void:
    locomotion.initialize(
        self,
        {
            &"movement_intent": movement_intent,
            &"motor": motor,
            &"visual": visual,
        }
    )


func _unhandled_input(event: InputEvent) -> void:
    locomotion.handle_input(event)


func _process(delta: float) -> void:
    locomotion.update(delta)


func _physics_process(delta: float) -> void:
    locomotion.physics_update(delta)
~~~

## 1物理フレームでの具体的な流れ

Walk中にrunが押された場合を追います。

1. GodotがPlayer._physics_processを呼ぶ。
2. Playerがlocomotion.physics_updateを呼ぶ。
3. StateMachineがWalk.physics_updateを呼ぶ。
4. WalkがMovementIntentからmove_vectorとwants_runを取得する。
5. Walkがrequest_transition(Run, data)を呼ぶ。
6. StateMachineが遷移元Walkを検証する。
7. Walk.exit(Run)が呼ばれる。
8. active_stateがRunへ変わる。
9. Run.enter(Walk, data)が呼ばれ、runアニメーションが始まる。
10. request_transitionの呼び出しへ戻る。
11. Walkは直後のreturnで処理を終了する。
12. 次の物理フレームからRun.physics_updateが走行速度を適用する。

この実装では、Stateが切り替わった物理フレームに新Stateのphysics_updateを再実行しません。制御フローを1回で終わらせ、連鎖遷移を防ぐためです。

## アニメーション名の扱い

この例では簡略化のため、Stateがidle、walk、runという論理名を持っています。

実際に次の条件がある場合は、Visualコンポーネントへ解決を移します。

- idle_down、idle_upのように向き別アニメーションがある。
- 衣服状態によってSpriteFramesが変わる。
- 武器や状態異常で表示が変わる。
- AnimationTreeや上半身・下半身レイヤーを使う。

~~~gdscript
visual.play_locomotion(&"walk", facing)
~~~

Stateは「walkを表示したい」とだけ伝え、Visualが現在の向きと衣服に対応する実アニメーションを選びます。

## この例をそのまま使わない方がよいケース

- WalkとRunで違うのが速度だけで、State固有のenter、exit、遷移が今後も増えない。
- アナログ入力の強さだけで連続的に速度を変えたい。
- 速度ブレンドをAnimationTreeだけで表現したい。

その場合はMoveという1つのStateにまとめ、入力強度やrunフラグから速度を計算する方が簡潔です。State数はアニメーション数ではなく、振る舞いとライフサイクルの違いで決めます。

---

[前章：移動コンポーネントの設計](04_movement_components.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [06. 拡張パターン](06_extensions_and_composition.md)
