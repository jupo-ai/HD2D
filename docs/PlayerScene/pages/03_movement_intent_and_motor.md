# 03. 入力IntentとMotor

[前章：プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md) ｜ [目次へ](../README.md) ｜ [次章：カメラ基準のFacing](04_camera_relative_facing.md)

## この章で分けるもの

移動処理を次の3段階へ分けます。

1. MovementIntentSourceが「どちらへ、何をしたいか」を返す。
2. LocomotionStateがIdle、Walk、Runの規則と速度を選ぶ。
3. CharacterMotor3Dがvelocityを更新し、`move_and_slide()`を実行する。

PlayerMovementIntentだけがInputとCamera3Dを知ります。Motorが受け取る方向は、すでにワールドXZ平面のVector3です。

## 1. MovementIntentSourceを作る

作成先:

~~~text
res://entities/character/shared/movement/movement_intent_source.gd
~~~

~~~gdscript
class_name MovementIntentSource
extends Node


## 長さ0～1のワールドXZ平面の移動意図を返す。
func get_move_vector_world() -> Vector3:
    return Vector3.ZERO


## 走りたいという意図だけを返す。
## スタミナや地形による実行可否はStateまたはAbilityが判断する。
func wants_run() -> bool:
    return false


## 将来のJumpState用。現在は常にfalseでも契約を先に定義できる。
func wants_jump() -> bool:
    return false


## 将来のDashState用。
func wants_dash() -> bool:
    return false
~~~

`get_move_vector_world()`の契約は必ず守ります。

- XはワールドX方向である。
- Yは常に0である。
- ZはワールドZ方向である。
- 長さは0～1で、アナログ入力の強さを保持してよい。

StateとMotorは、元の入力がキーボード、スティック、AI、リプレイのどれかを知りません。

## 2. PlayerMovementIntentを作る

作成先:

~~~text
res://entities/character/player/player_movement_intent.gd
~~~

~~~gdscript
class_name PlayerMovementIntent
extends MovementIntentSource

@export_group("Camera")
@export var camera_override: Camera3D

@export_group("Input Actions")
@export var move_left_action: StringName = &"move_left"
@export var move_right_action: StringName = &"move_right"
@export var move_forward_action: StringName = &"move_forward"
@export var move_back_action: StringName = &"move_back"
@export var run_action: StringName = &"run"
@export var jump_action: StringName = &"jump"
@export var dash_action: StringName = &"dash"

var _warned_about_missing_camera: bool = false


func get_move_vector_world() -> Vector3:
    var screen_input: Vector2 = Input.get_vector(
        move_left_action,
        move_right_action,
        move_forward_action,
        move_back_action
    )

    if screen_input.is_zero_approx():
        return Vector3.ZERO

    var camera: Camera3D = _get_camera()
    if camera == null:
        if not _warned_about_missing_camera:
            push_warning(
                "PlayerMovementIntentがCamera3Dを取得できません"
            )
            _warned_about_missing_camera = true

        return Vector3(
            screen_input.x,
            0.0,
            screen_input.y
        )

    _warned_about_missing_camera = false

    var camera_basis: Basis = (
        camera.get_camera_transform().basis
    )

    var screen_right_world: Vector3 = camera_basis.x
    screen_right_world.y = 0.0

    var screen_up_world: Vector3 = camera_basis.y
    screen_up_world.y = 0.0

    if screen_right_world.is_zero_approx():
        screen_right_world = Vector3.RIGHT
    else:
        screen_right_world = screen_right_world.normalized()

    # 水平に近いCameraではbasis.yのXZ成分が小さくなるため、
    # Cameraの前方で補う。
    if screen_up_world.is_zero_approx():
        screen_up_world = -camera_basis.z
        screen_up_world.y = 0.0

    if screen_up_world.is_zero_approx():
        screen_up_world = Vector3.FORWARD
    else:
        screen_up_world = screen_up_world.normalized()

    # Input.get_vectorではforward入力が負のYになる。
    var world_input: Vector3 = (
        screen_right_world * screen_input.x
        + screen_up_world * -screen_input.y
    )

    if world_input.length_squared() > 1.0:
        world_input = world_input.normalized()

    return world_input


func wants_run() -> bool:
    return Input.is_action_pressed(run_action)


func wants_jump() -> bool:
    return (
        InputMap.has_action(jump_action)
        and Input.is_action_just_pressed(jump_action)
    )


func wants_dash() -> bool:
    return (
        InputMap.has_action(dash_action)
        and Input.is_action_just_pressed(dash_action)
    )


func _get_camera() -> Camera3D:
    if is_instance_valid(camera_override):
        return camera_override

    return get_viewport().get_camera_3d()
~~~

### なぜCameraの`basis.y`を使うのか

画面上方向に対応するワールド方向は、Cameraのローカル上方向をXZ平面へ投影したものです。斜め上から見下ろすカメラなら、W入力が画面奥側のワールド方向になります。

Cameraがほぼ水平で`basis.y`のXZ成分を利用できない場合は、Cameraの前方である`-basis.z`を使います。`get_camera_transform()`はCamera3Dのoffsetなども含む表示用Transformを返します。

### カメラを回した場合

同じW入力でも、毎回現在のCamera3Dから方向を計算し直します。そのためカメラを90度回せば、Playerが進むワールド方向も90度変わります。

Motor側でカメラ相対変換をもう一度行ってはいけません。Intentが返したVector3はすでにワールド方向です。

## 3. CharacterMotor3Dを作る

作成先:

~~~text
res://entities/character/shared/movement/character_motor_3d.gd
~~~

~~~gdscript
class_name CharacterMotor3D
extends Node

@export var body: CharacterBody3D

@export_group("Horizontal Movement")
@export var acceleration: float = 20.0
@export var deceleration: float = 30.0

@export_group("Vertical Movement")
@export var use_gravity: bool = true
@export var gravity_scale: float = 1.0

var _desired_horizontal_velocity: Vector3 = Vector3.ZERO
var _gravity: float = float(
    ProjectSettings.get_setting(
        "physics/3d/default_gravity",
        9.8
    )
)


func command_move(
    move_vector_world: Vector3,
    max_speed: float
) -> void:
    var horizontal_input: Vector3 = move_vector_world
    horizontal_input.y = 0.0

    var input_strength: float = minf(
        horizontal_input.length(),
        1.0
    )

    if input_strength <= 0.0:
        _desired_horizontal_velocity = Vector3.ZERO
        return

    _desired_horizontal_velocity = (
        horizontal_input.normalized()
        * max_speed
        * input_strength
    )


func command_stop() -> void:
    _desired_horizontal_velocity = Vector3.ZERO


func physics_step(delta: float) -> void:
    assert(body != null, "CharacterMotor3D.bodyが未設定です")

    var rate: float = acceleration
    if _desired_horizontal_velocity.is_zero_approx():
        rate = deceleration

    body.velocity.x = move_toward(
        body.velocity.x,
        _desired_horizontal_velocity.x,
        rate * delta
    )
    body.velocity.z = move_toward(
        body.velocity.z,
        _desired_horizontal_velocity.z,
        rate * delta
    )

    if use_gravity:
        if body.is_on_floor() and body.velocity.y < 0.0:
            body.velocity.y = 0.0
        elif not body.is_on_floor():
            body.velocity.y -= _gravity * gravity_scale * delta

    body.move_and_slide()


func set_vertical_velocity(value: float) -> void:
    assert(body != null, "CharacterMotor3D.bodyが未設定です")
    body.velocity.y = value
~~~

## 4. Motorの呼び出し規則

Stateは`command_move()`または`command_stop()`でそのフレームの方針を設定します。PlayerはStateMachine更新後に`physics_step()`を1回だけ呼びます。

~~~gdscript
func _physics_process(delta: float) -> void:
    locomotion.physics_update(delta)
    motor.physics_step(delta)
~~~

次は禁止します。

- Stateから`body.move_and_slide()`を直接呼ぶ。
- PlayerとMotorの両方から`move_and_slide()`を呼ぶ。
- VisualやFacingからvelocityを書き換える。
- 目標速度へdeltaを掛ける。

`max_speed`は毎秒の速度です。deltaを掛けるのは、加速度、減速度、重力を積算するときです。

## 5. Inspector設定

Playerシーンで次を設定します。

| Node | プロパティ | 初期値例 |
|---|---|---:|
| PlayerMovementIntent | camera_override | 未設定 |
| PlayerMovementIntent | 各Action | Input Mapと同名 |
| CharacterMotor3D | body | Player |
| CharacterMotor3D | acceleration | 20.0 |
| CharacterMotor3D | deceleration | 30.0 |
| CharacterMotor3D | use_gravity | true |

速度2.5や5.0はMotorの固定値ではなく、WalkStateとRunStateが選びます。Motorは「どのStateか」を知りません。

## 6. 単体確認

Stateを作る前でも、デバッガや一時的な観測コードで次を確認できます。

| 条件 | 期待値 |
|---|---|
| Cameraが初期角度、W入力 | 画面上方向に対応するワールドVector3 |
| Cameraを90度回転、W入力 | 前の結果からXZ方向が90度変化 |
| W+D入力 | 長さが1以下の斜め方向 |
| スティック半倒し | 長さが1未満で強さを保持 |
| current cameraなし | 警告は1回だけ、ワールド軸でフォールバック |
| command_stop | 水平速度がdecelerationで0へ近づく |
| 1物理フレーム | `move_and_slide()`は1回 |

---

[前章：プロジェクトとPlayerシーンの準備](02_project_and_scene_setup.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [04. カメラ基準のFacing](04_camera_relative_facing.md)
