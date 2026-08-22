# 04. カメラ基準のFacing

[前章：入力IntentとMotor](03_movement_intent_and_motor.md) ｜ [目次へ](../README.md) ｜ [次章：衣服とVisual](05_outfit_and_visual.md)

## 解決する問題

Playerがワールドの+Z方向を向いているという情報だけでは、`idle_up`と`idle_down`のどちらを表示すべきか決まりません。現在のCamera3DがPlayerをどちら側から見ているかで、画面上の見え方が変わるためです。

たとえばCamera3Dが+Z側からPlayerを見ているとき、Playerが+Z方向を向いていればPlayerはカメラ側を向いています。この場合は後頭部を表す`idle_up`ではなく、正面を表す`idle_down`を表示します。

この資料では、次の2種類の向きを区別します。

| 値 | 意味 | カメラ回転で変化するか |
|---|---|---|
| world_facing | キャラクターがワールドXZ平面のどちらを向いているか | 変化しない |
| screen_facing | 現在のCamera3Dから見たDown、Left、Right、Up | 変化する |

## 判定の考え方

Camera3Dの表示用Basisから、画面右と画面上に対応するワールドXZ方向を作ります。

~~~text
horizontal = world_facing・screen_right_world
vertical   = world_facing・screen_up_world

abs(horizontal) > abs(vertical)
    horizontal > 0 なら RIGHT
    horizontal < 0 なら LEFT

それ以外
    vertical > 0 なら UP
    vertical < 0 なら DOWN
~~~

`vertical < 0`がDOWNである点が重要です。world_facingが画面下側、つまりカメラ側を向くと、画面上方向との内積が負になるためです。

## 1. CameraRelativeFacing3Dを作る

作成先:

~~~text
res://entities/character/shared/visual/camera_relative_facing_3d.gd
~~~

~~~gdscript
class_name CameraRelativeFacing3D
extends Node

enum ScreenFacing {
    DOWN,
    LEFT,
    RIGHT,
    UP,
}

signal screen_facing_changed(
    previous_facing: int,
    current_facing: int
)

@export var camera_override: Camera3D
@export var default_world_facing: Vector3 = Vector3.BACK

var _world_facing: Vector3 = Vector3.BACK
var _screen_facing: int = ScreenFacing.DOWN


func initialize() -> void:
    set_world_facing(default_world_facing)
    refresh()


func set_world_facing(direction: Vector3) -> void:
    var horizontal_direction: Vector3 = direction
    horizontal_direction.y = 0.0

    if horizontal_direction.is_zero_approx():
        return

    _world_facing = horizontal_direction.normalized()


func get_world_facing() -> Vector3:
    return _world_facing


func get_screen_facing() -> int:
    return _screen_facing


## Camera回転だけでも結果が変わり得るため、Visual更新時に呼ぶ。
func refresh() -> bool:
    var next_facing: int = _calculate_screen_facing()
    if next_facing == _screen_facing:
        return false

    var previous_facing: int = _screen_facing
    _screen_facing = next_facing
    screen_facing_changed.emit(previous_facing, _screen_facing)
    return true


func get_animation_suffix() -> StringName:
    match _screen_facing:
        ScreenFacing.DOWN:
            return &"down"
        ScreenFacing.LEFT:
            return &"left"
        ScreenFacing.RIGHT:
            return &"right"
        ScreenFacing.UP:
            return &"up"

    return &"down"


func _calculate_screen_facing() -> int:
    var camera: Camera3D = _get_camera()
    if camera == null:
        return _screen_facing

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

    if screen_up_world.is_zero_approx():
        screen_up_world = -camera_basis.z
        screen_up_world.y = 0.0

    if screen_up_world.is_zero_approx():
        screen_up_world = Vector3.FORWARD
    else:
        screen_up_world = screen_up_world.normalized()

    var horizontal: float = _world_facing.dot(
        screen_right_world
    )
    var vertical: float = _world_facing.dot(
        screen_up_world
    )

    if absf(horizontal) > absf(vertical):
        if horizontal >= 0.0:
            return ScreenFacing.RIGHT
        return ScreenFacing.LEFT

    if vertical >= 0.0:
        return ScreenFacing.UP
    return ScreenFacing.DOWN


func _get_camera() -> Camera3D:
    if is_instance_valid(camera_override):
        return camera_override

    return get_viewport().get_camera_3d()
~~~

## 2. 移動時にworld_facingを更新する

LocomotionStateは、0ではない移動意図をMotorへ渡すとき、同じワールド方向をFacingへ渡します。

~~~gdscript
func command_motion(
    move_vector_world: Vector3,
    max_speed: float
) -> void:
    if not move_vector_world.is_zero_approx():
        facing.set_world_facing(move_vector_world)

    motor.command_move(move_vector_world, max_speed)
~~~

停止時にはworld_facingをVector3.ZEROへ戻しません。Idleは最後に向いていたワールド方向を維持します。

## 3. Idle中もCamera回転へ追従する

world_facingが変わらなくても、Camera3Dが回ればscreen_facingは変わります。そのため、移動したときだけ`refresh()`を呼ぶ設計では不足します。

Playerの`_process()`からCharacterVisual3Dの`refresh()`を呼び、その中でFacingも更新します。

~~~gdscript
func _process(delta: float) -> void:
    locomotion.update(delta)
    visual.refresh()
~~~

これにより、Playerが+Zを向いたままIdle中でも、カメラがPlayerの背面側から正面側へ回り込めば、`idle_up`から`idle_down`へ切り替わります。

## 4. 具体例

Cameraがワールド+Z側からPlayerを見ており、画面上方向がワールド-Zへ対応しているとします。

| world_facing | Cameraからの見え方 | screen_facing | Idleアニメーション |
|---|---|---|---|
| +Z | カメラ側を向く | DOWN | `idle_down` |
| -Z | カメラへ背を向ける | UP | `idle_up` |
| Cameraの画面右方向 | 右向き | RIGHT | `idle_right` |
| Cameraの画面左方向 | 左向き | LEFT | `idle_left` |

カメラを180度回転すると、world_facingを変更していなくてもUPとDOWNが反転します。

## 5. 斜め方向の扱い

初期実装は、内積の絶対値が大きい軸を選ぶ4方向判定です。45度付近では小さな入力変化により上下と左右が頻繁に切り替わる可能性があります。

実際にちらつきが発生した場合は、次の順で対処します。

1. 前回のscreen_facingを保持する。
2. 新しい軸が前回の軸を一定値以上上回った場合だけ変更する。
3. 8方向素材が用意できた場合はScreenFacingを8方向へ拡張する。

アニメーション素材が4方向しかない段階で、角度を細かく保持しても表示上の方向は増えません。最初は単純な支配軸判定で十分です。

## 6. PlayerとNPCで共有できる理由

PlayerとNPCでworld_facingの入力元は異なります。

- Playerはカメラ相対入力から作ったワールド移動方向を使う。
- NPCはNavigationAgent3Dの次の経路点へ向かうワールド方向を使う。

しかし、どちらもFacingへ渡す時点ではワールドVector3です。CameraRelativeFacing3Dは入力元を知らず、現在のCamera3Dから見た表示方向だけを算出します。そのためNPCの方向別Spriteにも同じコンポーネントを使えます。

## 7. 確認項目

- [ ] Vector3.ZEROを渡しても最後のworld_facingが失われない。
- [ ] Camera正面側へ向くとDOWNになる。
- [ ] Cameraへ背を向けるとUPになる。
- [ ] Cameraを90度回すとLEFTまたはRIGHTへ変化する。
- [ ] Idle中のCamera回転だけでscreen_facingが変わる。
- [ ] PlayerとNPCの両方がワールド方向を同じAPIへ渡せる。

---

[前章：入力IntentとMotor](03_movement_intent_and_motor.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [05. 衣服とVisual](05_outfit_and_visual.md)
