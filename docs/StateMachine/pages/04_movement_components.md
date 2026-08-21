# 04. 移動コンポーネントの設計

[前章：処理命令と状態遷移の流れ](03_execution_and_transition_flow.md) ｜ [目次へ](../README.md) ｜ [次章：Idle・Walk・Runの実装例](05_locomotion_example.md)

**主な参考動画:** [AdvancedStateMachine.mp4](../AdvancedStateMachine.mp4)

## 動画のMoveComponentが分離しているもの

発展編の重要な点は、StateがInputを直接読む設計から、移動意図を返すコンポーネントを経由する設計へ変えていることです。

ここでいう移動コンポーネントは、必ずしもCharacterBodyを動かす物理実行部ではありません。動画では主に、次の問いへ答える入力インターフェースとして使われています。

- どの方向へ移動したいか。
- 今ジャンプしたいか。

「ジャンプしたい」と「ジャンプできる」は別です。入力コンポーネントは意図だけを返し、接地中か、スタミナがあるか、行動不能ではないかという許可判断はStateやAbility側が行います。

## 推奨する3層分離

3Dキャラクターでは、移動を次の3層へ分けると責務が明確になります。

~~~mermaid
flowchart LR
    PlayerInput[PlayerMovementIntent] -->|Vector2 / wants_run| State
    AIInput[AIMovementIntent] -->|同じ契約| State
    State[Idle / Walk / Run] -->|速度とルールを選択| Motor[CharacterMotor3D]
    Motor -->|velocity + move_and_slide| Body[CharacterBody3D]
    Body -->|衝突結果| State
~~~

| 層 | 責務 | 知ってよいもの |
|---|---|---|
| MovementIntent | Player入力やAI判断を共通形式へ変換 | Input、Target、NavigationAgent |
| State | 移動可否、歩行か走行か、遷移規則 | Intent、Stats、現在の物理状態 |
| CharacterMotor | 加減速、velocity、move_and_slide | CharacterBody3D、移動パラメータ |

この分離により、PlayerMovementIntentをAIMovementIntentへ交換しても、Idle、Walk、Runを再利用できます。

## MovementIntentの契約

GDScriptには一般的な言語と同じinterface宣言がないため、共通基底Nodeで契約を表します。

~~~gdscript
class_name MovementIntentSource
extends Node


## XをワールドX、YをワールドZとして扱う移動意図
func get_move_vector() -> Vector2:
    return Vector2.ZERO


## 走りたいか。実際に走れるかはStateが判断する
func wants_run() -> bool:
    return false


## ジャンプしたいか。実際に跳べるかはStateが判断する
func wants_jump() -> bool:
    return false
~~~

契約で最初に決めるべきなのは、Vector2の意味です。この例ではVector2.xをワールドX、Vector2.yをワールドZとして扱います。

カメラ相対移動を採用するなら、MovementIntent側でカメラのforward/rightへ変換してワールド平面の意図を返すか、Motor側がカメラ参照を受け取って変換するかを決めます。両方で変換してはいけません。

## Player用MovementIntent

~~~gdscript
class_name PlayerMovementIntent
extends MovementIntentSource


func get_move_vector() -> Vector2:
    return Input.get_vector(
        &"move_left",
        &"move_right",
        &"move_up",
        &"move_down"
    )


func wants_run() -> bool:
    return Input.is_action_pressed(&"run")


func wants_jump() -> bool:
    return Input.is_action_just_pressed(&"jump")
~~~

Input.get_vectorは、4方向の入力を長さ1以下のVector2へまとめ、円形デッドゾーンも適用します。キーボード、パッド、スティックの差をStateへ持ち込まずに済みます。

## AI用MovementIntent

同じ契約をAI判断で実装できます。

~~~gdscript
class_name AIMovementIntent
extends MovementIntentSource

@export var actor: Node3D
@export var target: Node3D
@export var run_distance: float = 6.0


func get_move_vector() -> Vector2:
    if actor == null or target == null:
        return Vector2.ZERO

    var offset: Vector3 = (
        target.global_position - actor.global_position
    )
    return Vector2(offset.x, offset.z).normalized()


func wants_run() -> bool:
    if actor == null or target == null:
        return false

    return actor.global_position.distance_to(
        target.global_position
    ) >= run_distance


func wants_jump() -> bool:
    return false
~~~

実際のAIではNavigationAgent3Dから次の経路点を取得し、その方向を返します。Stateは入力元が人間かAIかを知る必要がありません。

## CharacterMotor3D

物理実行をStateから分離します。

~~~gdscript
class_name CharacterMotor3D
extends Node

@export var body: CharacterBody3D
@export var acceleration: float = 20.0
@export var deceleration: float = 30.0


func move(
    input_vector: Vector2,
    max_speed: float,
    delta: float
) -> void:
    if body == null:
        return

    var input: Vector2 = input_vector.limit_length(1.0)
    var desired_velocity: Vector3 = Vector3(
        input.x,
        0.0,
        input.y
    ) * max_speed

    var rate: float = acceleration
    if input.is_zero_approx():
        rate = deceleration

    body.velocity.x = move_toward(
        body.velocity.x,
        desired_velocity.x,
        rate * delta
    )
    body.velocity.z = move_toward(
        body.velocity.z,
        desired_velocity.z,
        rate * delta
    )

    body.move_and_slide()
~~~

velocityは毎秒単位の速度です。desired_velocityへdeltaを掛けません。move_and_slideは物理ステップのdeltaを内部で使用します。加速度から速度へ近づけるrateにはdeltaを掛けます。

この例はY速度を変更しないため、重力やジャンプを別処理で管理できます。トップダウン専用で上下移動を使わない場合は、CharacterBody3Dのmotion_modeも含めてプロジェクト方針を決めます。

## move_and_slideの実行者は1つにする

1回の物理フレームで複数のStateやコンポーネントがmove_and_slideを呼ぶと、次の問題が起こります。

- 衝突結果がどの移動に対するものか分からなくなる。
- is_on_floorやis_on_wallの意味が途中で変わる。
- 移動FSMと攻撃FSMがvelocityを上書きし合う。
- 加速・ノックバック・移動床の合成順が不明になる。

推奨規則は「ActorごとにMotorを1つ置き、1物理フレーム中のmove_and_slideは最大1回にする」です。単純なState駆動例では、遷移だけを行うフレームに0回となる場合があります。接触判定を毎物理フレーム更新する必要があるなら、Stateは目標速度だけを設定し、ActorまたはMotorがフレーム末に必ず1回実行する構成へ進めます。

StateがMotor.moveを呼ぶ構成では、現在Stateだけが1回呼ぶようにします。より複雑になったら、Stateはdesired_velocityだけをMotorへ設定し、Actorが最後にMotor.physics_stepを1回呼ぶ方式へ発展させます。

## StateがInputを直接読んでもよい場合

小さなPlayer専用プロトタイプで、AI再利用の予定がなければ、StateがInputを直接読んでも実用上問題はありません。

ただし、次のいずれかが必要になった時点でIntent分離の効果が大きくなります。

- 同じ移動StateをAIにも使う。
- 操作不能、混乱、オート移動などで入力元を切り替える。
- リプレイや入力記録を実装する。
- テストで任意の入力を注入する。
- キーボードとタッチ操作の差をStateから隠す。

抽象化は目的ではありません。交換したい依存だけを境界の外へ出します。

## FacingとVisualはさらに分ける

MovementIntentが返す方向からFacingを更新する処理は、MotorまたはFacingコンポーネントへ置けます。一方、どのSpriteFramesのどのアニメーション名を再生するかはVisualの責務です。

~~~text
Walk State
├── 移動速度を選ぶ
├── Motorへ移動意図を渡す
└── Visualへ論理アニメーション walk を要求

Visual
├── Facing = DOWN
├── Clothing = OUTDOOR
└── 実アニメーション名 walk_down とSpriteFramesを解決
~~~

Stateへ服装別・向き別のアニメーション名を埋め込まないことで、移動ロジックと表示差分を分離できます。

## 公式資料

- [Input.get_vector](https://docs.godotengine.org/en/stable/classes/class_input.html)
- [CharacterBody3D](https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html)

---

[前章：処理命令と状態遷移の流れ](03_execution_and_transition_flow.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [05. Idle・Walk・Runの実装例](05_locomotion_example.md)
