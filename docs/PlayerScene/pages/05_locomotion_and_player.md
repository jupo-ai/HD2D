# 05. Idle・Walk・RunとPlayer本体

[前章：カメラ基準のFacingと方向別アニメーション](04_camera_relative_facing.md) ｜ [目次へ](../README.md) ｜ [次章：拡張・NPC化・テスト](06_extensions_npc_and_tests.md)

## この章で接続するもの

ここまでに作ったIntent、Motor、Facing、DirectionalSpriteAnimatorを、共有ロコモーションStateとPlayer本体から接続します。

~~~text
PlayerMovementIntent ─┐
CharacterMotor3D     ─┼─> LocomotionStateMachine ─> Idle / Walk / Run
Facing               ─┤
DirectionalAnimator  ─┘
~~~

StateはPlayer専用にせず、`LocomotionState`という共有基底クラスを使用します。

## 1. LocomotionStateを作る

作成先:

~~~text
res://entities/character/shared/locomotion/locomotion_state.gd
~~~

~~~gdscript
class_name LocomotionState
extends State

@export var logical_animation: StringName

var body: CharacterBody3D
var movement_intent: MovementIntentSource
var motor: CharacterMotor3D
var facing: CameraRelativeFacing3D
var animator: DirectionalSpriteAnimator3D


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
    facing = injected_dependencies.get(
        &"facing"
    ) as CameraRelativeFacing3D
    animator = injected_dependencies.get(
        &"animator"
    ) as DirectionalSpriteAnimator3D

    assert(body != null, "CharacterBody3Dが必要です")
    assert(
        movement_intent != null,
        "MovementIntentSourceが必要です"
    )
    assert(motor != null, "CharacterMotor3Dが必要です")
    assert(facing != null, "CameraRelativeFacing3Dが必要です")
    assert(
        animator != null,
        "DirectionalSpriteAnimator3Dが必要です"
    )


func enter(
    _previous_state: State,
    _data: Dictionary = {}
) -> void:
    animator.set_logical_animation(logical_animation)


func get_move_vector_world() -> Vector3:
    var move_vector: Vector3 = (
        movement_intent.get_move_vector_world()
    )
    move_vector.y = 0.0

    if move_vector.length_squared() > 1.0:
        move_vector = move_vector.normalized()

    return move_vector


func command_motion(
    move_vector_world: Vector3,
    max_speed: float
) -> void:
    if not move_vector_world.is_zero_approx():
        facing.set_world_facing(move_vector_world)

    motor.command_move(move_vector_world, max_speed)
~~~

具体StateはInput、Camera3D、AnimatedSprite3D、SpriteFramesを直接参照しません。SpriteFramesはAnimatedSprite3Dへ設定済みの1つだけを使用します。

## 2. IdleStateを作る

作成先:

~~~text
res://entities/character/shared/locomotion/character_idle_state.gd
~~~

~~~gdscript
class_name CharacterIdleState
extends LocomotionState

@export var walk_state: State


func physics_update(_delta: float) -> void:
    var move_vector: Vector3 = get_move_vector_world()

    motor.command_stop()

    if not move_vector.is_zero_approx():
        request_transition(
            walk_state,
            {&"move_vector": move_vector}
        )
        return
~~~

Idleは水平速度を減速させます。world_facingは更新しないため、最後の向きを維持します。

## 3. WalkStateを作る

作成先:

~~~text
res://entities/character/shared/locomotion/character_walk_state.gd
~~~

~~~gdscript
class_name CharacterWalkState
extends LocomotionState

@export var idle_state: State
@export var run_state: State
@export var walk_speed: float = 2.5


func physics_update(_delta: float) -> void:
    var move_vector: Vector3 = get_move_vector_world()

    if move_vector.is_zero_approx():
        motor.command_stop()
        request_transition(idle_state)
        return

    command_motion(move_vector, walk_speed)

    if movement_intent.wants_run():
        request_transition(
            run_state,
            {&"move_vector": move_vector}
        )
        return
~~~

## 4. RunStateを作る

作成先:

~~~text
res://entities/character/shared/locomotion/character_run_state.gd
~~~

~~~gdscript
class_name CharacterRunState
extends LocomotionState

@export var idle_state: State
@export var walk_state: State
@export var run_speed: float = 5.0


func physics_update(_delta: float) -> void:
    var move_vector: Vector3 = get_move_vector_world()

    if move_vector.is_zero_approx():
        motor.command_stop()
        request_transition(idle_state)
        return

    command_motion(move_vector, run_speed)

    if not movement_intent.wants_run():
        request_transition(
            walk_state,
            {&"move_vector": move_vector}
        )
        return
~~~

### 遷移フレームの速度

このStateMachineは遷移先の`enter()`までを同じ物理フレームに実行しますが、遷移先の`physics_update()`は次の物理フレームから呼びます。

したがってWalkからRunへ遷移するフレームではWalkの速度命令、次の物理フレームからRunの速度命令になります。RunからWalkも同様に、現在Stateがそのフレームの移動方針を最後まで所有します。

Idleでrunを押しながら移動を始めた場合は、IdleからWalk、次の物理フレームにWalkからRunへ進みます。即座にRunへ入りたい操作感が必要なら、Idleへ`run_state`参照を追加し、`wants_run()`時だけ直接Runを要求します。

## 5. Player本体を作る

作成先:

~~~text
res://entities/character/player/player.gd
~~~

~~~gdscript
class_name PlayerCharacter
extends CharacterBody3D

@onready var movement_intent: MovementIntentSource = (
    $PlayerMovementIntent as MovementIntentSource
)
@onready var motor: CharacterMotor3D = (
    $CharacterMotor3D as CharacterMotor3D
)
@onready var facing: CameraRelativeFacing3D = (
    $CameraRelativeFacing3D as CameraRelativeFacing3D
)
@onready var directional_animator: DirectionalSpriteAnimator3D = (
    $DirectionalSpriteAnimator3D as DirectionalSpriteAnimator3D
)
@onready var locomotion: StateMachine = (
    $LocomotionStateMachine as StateMachine
)


func _ready() -> void:
    directional_animator.initialize()

    locomotion.initialize(
        self,
        {
            &"movement_intent": movement_intent,
            &"motor": motor,
            &"facing": facing,
            &"animator": directional_animator,
        }
    )


func _unhandled_input(event: InputEvent) -> void:
    locomotion.handle_input(event)


func _process(delta: float) -> void:
    locomotion.update(delta)
    directional_animator.refresh()


func _physics_process(delta: float) -> void:
    locomotion.physics_update(delta)
    motor.physics_step(delta)
~~~

### 初期化順

DirectionalSpriteAnimatorを先に初期化し、その後にStateMachineを初期化します。StateMachineの初期StateであるIdleの`enter()`がAnimatorへ`idle`を要求するとき、AnimatedSprite3Dの単一SpriteFramesとFacingがすでに利用可能である必要があるためです。

## 6. Inspectorを設定する

### LocomotionStateMachine

| プロパティ | 値 |
|---|---|
| initial_state | Idle |

### Idle

| プロパティ | 値 |
|---|---|
| logical_animation | `idle` |
| walk_state | Walk |

### Walk

| プロパティ | 値 |
|---|---|
| logical_animation | `walk` |
| idle_state | Idle |
| run_state | Run |
| walk_speed | 2.5 |

### Run

| プロパティ | 値 |
|---|---|
| logical_animation | `run` |
| idle_state | Idle |
| walk_state | Walk |
| run_speed | 5.0 |

速度はアートのフレームレートではなく、ワールド単位の移動速度として調整します。SpriteFrames側のFPSは足滑りを見ながら別に調整できます。

## 7. 実行時の命令順

Walk中にrunを押した物理フレームを追います。

~~~mermaid
sequenceDiagram
    participant GodotEngine as Godot
    participant PlayerNode as Player
    participant MachineNode as Locomotion
    participant WalkStateNode as Walk
    participant RunStateNode as Run
    participant FacingNode as Facing
    participant MotorNode as Motor
    participant AnimatorNode as DirectionalAnimator

    GodotEngine->>PlayerNode: _physics_process(delta)
    PlayerNode->>MachineNode: physics_update(delta)
    MachineNode->>WalkStateNode: physics_update(delta)
    WalkStateNode->>FacingNode: set_world_facing(move_vector)
    WalkStateNode->>MotorNode: command_move(move_vector, walk_speed)
    WalkStateNode->>MachineNode: transition_requested(Walk, Run)
    MachineNode->>WalkStateNode: exit(Run)
    MachineNode->>RunStateNode: enter(Walk, data)
    RunStateNode->>AnimatorNode: set_logical_animation(run)
    WalkStateNode-->>MachineNode: return
    MachineNode-->>PlayerNode: return
    PlayerNode->>MotorNode: physics_step(delta)
    MotorNode-->>PlayerNode: move_and_slide完了
~~~

`request_transition()`後に旧Stateの処理を続けないため、各遷移要求の直後に`return`しています。

## 8. 最初の動作確認

1. currentなCamera3Dを持つテストMapへPlayerを配置する。
2. 起動直後のactive_stateがIdleであることを確認する。
3. W入力でWalkへ遷移し、画面上方向へ動くことを確認する。
4. runを押してRunへ遷移し、速度とアニメーションが変わることを確認する。
5. 入力を離し、Idleへ戻って減速することを確認する。
6. Idle中にCamera3Dを回転し、`idle_*`が切り替わることを確認する。
7. AnimatedSprite3DのSpriteFramesが実行中に変化しないことを確認する。
8. `move_and_slide()`がMotorから1物理フレームに1回だけ呼ばれることを確認する。

---

[前章：カメラ基準のFacingと方向別アニメーション](04_camera_relative_facing.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [06. 拡張・NPC化・テスト](06_extensions_npc_and_tests.md)
