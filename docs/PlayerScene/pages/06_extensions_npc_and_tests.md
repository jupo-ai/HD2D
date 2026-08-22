# 06. 拡張・NPC化・テスト

[前章：Idle・Walk・RunとPlayer本体](05_locomotion_and_player.md) ｜ [目次へ](../README.md)

## 拡張先を決める基準

新しい要素を追加するときは、名前ではなく振る舞いの軸から置き場所を選びます。

| 追加要素 | 置き場所 | 理由 |
|---|---|---|
| Dash | LocomotionStateMachine | 通常移動と排他的で、時間・固定方向・enterがある |
| Jump、Fall | LocomotionStateMachine | 接地移動と排他的で、Y速度と着地遷移がある |
| Attack | 別のActionStateMachine | WalkやRunと同時実行できる可能性がある |
| Stun、Dead | Status FSMまたは優先割り込み | 通常入力より高い優先度で複数軸を止める |
| NPC操作 | NavigationMovementIntent | StateとMotorはPlayerと同じ契約を使える |

## Dashを追加する

### 1. Input Map

`dash` Actionを追加し、キーまたはゲームパッドボタンを割り当てます。

### 2. DashState

作成先:

~~~text
res://entities/character/shared/locomotion/character_dash_state.gd
~~~

~~~gdscript
class_name CharacterDashState
extends LocomotionState

@export var idle_state: State
@export var walk_state: State
@export var run_state: State
@export var dash_speed: float = 9.0
@export var dash_duration: float = 0.2

var _remaining_time: float = 0.0
var _dash_direction: Vector3 = Vector3.ZERO


func enter(
    previous_state: State,
    data: Dictionary = {}
) -> void:
    super(previous_state, data)

    _remaining_time = dash_duration
    _dash_direction = data.get(
        &"move_vector",
        movement_intent.get_move_vector_world()
    )
    _dash_direction.y = 0.0

    if _dash_direction.is_zero_approx():
        _dash_direction = facing.get_world_facing()

    _dash_direction = _dash_direction.normalized()


func physics_update(delta: float) -> void:
    _remaining_time -= delta
    command_motion(_dash_direction, dash_speed)

    if _remaining_time > 0.0:
        return

    var move_vector: Vector3 = get_move_vector_world()
    if move_vector.is_zero_approx():
        request_transition(idle_state)
        return

    if movement_intent.wants_run():
        request_transition(run_state)
        return

    request_transition(walk_state)
~~~

### 3. 遷移元へ追加する

Idle、Walk、Runへ`dash_state`参照を追加し、通常遷移より前に次を判定します。

~~~gdscript
if movement_intent.wants_dash():
    request_transition(
        dash_state,
        {&"move_vector": move_vector}
    )
    return
~~~

Dash中は通常入力で方向を変えず、enter時に確定したワールド方向を使います。クールダウンやスタミナを導入したら、`wants_dash()`とは別にAbilityコンポーネントで実行可否を確認します。

### 4. 方向別アニメーション

AnimatedSprite3Dへ設定した1つのSpriteFramesへ、`dash_down`、`dash_left`、`dash_right`、`dash_up`を追加し、DashStateの`logical_animation`を`dash`にします。素材がまだない場合はDirectionalSpriteAnimator3Dのフォールバック規則により同方向Idleが表示されますが、開発中に警告を追加して不足を把握できるようにします。

## JumpとFallを追加する

Jumpはボタン名ではなく空中ロコモーションです。GroundedのIdle・Walk・Runと排他的なので、同じLocomotionStateMachineへJumpとFallを追加します。

### 1. Input Map

`jump` Actionを追加します。

### 2. Grounded StateからJumpを要求する

Idle、Walk、Runで、接地中かつ`wants_jump()`がtrueならJumpを要求します。

~~~gdscript
if body.is_on_floor() and movement_intent.wants_jump():
    request_transition(
        jump_state,
        {&"move_vector": move_vector}
    )
    return
~~~

### 3. JumpのenterでY速度を設定する

~~~gdscript
class_name CharacterJumpState
extends LocomotionState

@export var fall_state: State
@export var jump_speed: float = 6.0
@export var air_move_speed: float = 2.5


func enter(
    previous_state: State,
    data: Dictionary = {}
) -> void:
    super(previous_state, data)
    motor.set_vertical_velocity(jump_speed)


func physics_update(_delta: float) -> void:
    command_motion(
        get_move_vector_world(),
        air_move_speed
    )

    if body.velocity.y <= 0.0:
        request_transition(fall_state)
        return
~~~

FallStateは空中移動を続け、`body.is_on_floor()`がtrueになったら入力に応じてIdle、Walk、Runへ戻します。CharacterMotor3Dは既存どおり重力と`move_and_slide()`を1回だけ担当します。

`is_on_floor()`は直前の`move_and_slide()`結果です。この構成では着地したMotor実行の次の物理フレームにFallからGrounded Stateへ遷移します。その1フレーム差が問題になる場合だけ、Motorの移動後イベントを設計します。

## NPCへ再利用する

NPCではPlayerMovementIntentをNavigationMovementIntentへ交換します。

作成先:

~~~text
res://entities/character/npc/navigation_movement_intent.gd
~~~

~~~gdscript
class_name NavigationMovementIntent
extends MovementIntentSource

@export var actor: Node3D
@export var navigation_agent: NavigationAgent3D
@export var run_distance: float = 6.0


func get_move_vector_world() -> Vector3:
    if actor == null or navigation_agent == null:
        return Vector3.ZERO

    if navigation_agent.is_navigation_finished():
        return Vector3.ZERO

    var next_position: Vector3 = (
        navigation_agent.get_next_path_position()
    )
    var direction: Vector3 = (
        next_position - actor.global_position
    )
    direction.y = 0.0

    if direction.is_zero_approx():
        return Vector3.ZERO

    return direction.normalized()


func wants_run() -> bool:
    if actor == null or navigation_agent == null:
        return false

    return actor.global_position.distance_to(
        navigation_agent.target_position
    ) >= run_distance
~~~

### NPCシーンの構成

~~~text
NPC (CharacterBody3D)
├── CollisionShape3D
├── AnimatedSprite3D
├── NavigationAgent3D
├── NavigationMovementIntent
├── CharacterMotor3D
├── CameraRelativeFacing3D
├── DirectionalSpriteAnimator3D
└── LocomotionStateMachine
    ├── Idle
    ├── Walk
    └── Run
~~~

NPCでもCameraRelativeFacing3Dを使います。NPCの移動判断はカメラ非依存ですが、方向別Spriteの見え方はPlayerと同じ現在Camera3Dを基準にする必要があるためです。

再利用されるもの:

- StateとStateMachine
- LocomotionState、Idle、Walk、Run
- CharacterMotor3D
- CameraRelativeFacing3D
- DirectionalSpriteAnimator3D

交換するもの:

- PlayerMovementIntentからNavigationMovementIntent
- Player本体の入力・所有コードからNPCのAI・Target設定コード

PlayerとNPCのライフサイクル委譲コードまで同じになった段階で、`CharacterActor3D`基底クラスまたは共通シーンを抽出します。NPCを作る前から深い継承階層を用意する必要はありません。

## カメラ基準Facingのテスト表

world_facingを固定し、Camera3Dだけを回して確認します。

| 操作 | 期待結果 |
|---|---|
| PlayerがCamera側を向く | `*_down` |
| PlayerがCameraへ背を向ける | `*_up` |
| Playerが画面右を向く | `*_right` |
| Playerが画面左を向く | `*_left` |
| Idle中にCameraを90度回す | world_facingは不変、screen_facingだけ変化 |
| Cameraを切り替える | 新しいcurrent camera基準へ更新 |

少なくともCameraの方位0度、90度、180度、270度で、Idle、Walk、Runを確認します。

## 方向別アニメーションのテスト表

| 状態 | 操作 | 期待結果 |
|---|---|---|
| Idle | Camera側を向く | `idle_down`を再生 |
| Walk | 画面左へ向く | `walk_left`を再生 |
| Run | Cameraだけを90度回す | 対応する`run_*`へ切り替え、ロコモーションStateはRunのまま |
| Walk | 方向だけを変更 | 新しい`walk_*`へおおよその再生位置を継承 |
| Dash | `dash_*`が未作成 | 同方向の`idle_*`へフォールバック |
| 任意 | SpriteFramesが空 | エラーを出し、再生を停止 |

すべての操作で、AnimatedSprite3Dの`sprite_frames`参照が同じままであることも確認します。

## StateMachineのテスト表

| 初期State | 条件 | 期待State | 追加確認 |
|---|---|---|---|
| Idle | 移動入力あり | Walk | Walk.enterが1回 |
| Walk | 入力なし | Idle | Motorへcommand_stop |
| Walk | runあり | Run | request後にWalk処理を継続しない |
| Run | runなし | Walk | 次の物理フレームからwalk_speed |
| Run | 入力なし | Idle | Walkを経由しない |
| 任意 | 同じStateを要求 | 変化なし | enter・exitを繰り返さない |
| 任意 | Camera回転 | Stateは変わらない | 再生する方向別アニメーションだけ変わる |

## 物理処理の不変条件

- [ ] 1物理フレームでactiveなLocomotionStateは1つだけである。
- [ ] active Stateは毎物理フレーム`command_move()`か`command_stop()`を呼ぶ。
- [ ] `move_and_slide()`はCharacterMotor3Dだけが呼ぶ。
- [ ] velocityの水平成分はMotorだけが変更する。
- [ ] Y速度を変更する公開経路はMotorへ集約する。
- [ ] 遷移要求後は旧Stateが直ちにreturnする。
- [ ] Player入力とNPC判断は同じワールドVector3契約を返す。

## 旧Playerから切り替える最終手順

1. 新Playerを専用テストMapで検証する。
2. 入力、衝突、カメラ回転、Idle・Walk・Runの4方向表示を確認する。
3. Main、Map、SpawnerのPackedScene参照を新Playerへ変更する。
4. 旧PlayerのNodePath、script preload、型名、シグナル接続をプロジェクト全体で検索する。
5. セーブデータがScene Pathを保持している場合は移行処理を用意する。
6. 旧Playerを削除する。
7. Godotを再読込し、Parse Error、missing dependency、orphaned connectionを確認する。
8. 新規ゲームと既存セーブの両方で起動確認する。

## 完成条件

次の質問へ、Scene Treeとコードから即答できれば初期Playerは完成です。

- Player入力をワールド方向へ変換するのはどこか。
- Playerの最後のワールド向きを保持するのはどこか。
- Cameraから見た上下左右を決めるのはどこか。
- `move_and_slide()`を呼ぶのはどこか。
- Idle、Walk、Runを切り替えるのはどこか。
- Player用SpriteFramesを設定する場所はどこか。
- 論理アニメーション名とCamera基準の向きから、実アニメーション名を決めるのはどこか。
- Dash用アニメーションが未作成の場合、何へフォールバックするか。
- NPC化するとき、どのコンポーネントだけを交換するか。

---

[前章：Idle・Walk・RunとPlayer本体](05_locomotion_and_player.md) ｜ [目次へ](../README.md)
