# 06. 拡張パターン

[前章：Idle・Walk・Runの実装例](05_locomotion_example.md) ｜ [目次へ](../README.md) ｜ [次章：テスト・デバッグ・失敗例](07_testing_and_debugging.md)

**主な参考動画:** [AdvancedStateMachine.mp4](../AdvancedStateMachine.mp4)

## 拡張前に確認すること

StateMachineが複雑になったとき、Stateを追加することだけが解決策ではありません。

まず複雑さの原因を分類します。

| 原因 | 主な解決策 |
|---|---|
| 同じ処理の重複 | 共通基底State、ヘルパー、コンポーネント |
| PlayerとAIで入力だけ違う | MovementIntentを交換 |
| 移動しながら攻撃したい | 並行StateMachine |
| 複数Stateで能力値を使う | Actorまたは型付きデータコンポーネント |
| 外部イベントが強制遷移する | StateMachineの公開遷移API、割り込み規則 |
| State組み合わせが爆発する | 状態軸を分割 |
| enter中に別遷移が必要 | 遷移キューまたは遅延要求 |

## 依存性注入

Stateが次のように固定パスでActorを探すと、Scene Tree構造へ強く依存します。

~~~gdscript
var body: CharacterBody3D = get_node("../../Player")
~~~

代わりに、ActorがStateMachineを初期化するときに依存を渡します。

~~~gdscript
state_machine.initialize(
    self,
    {
        &"movement_intent": movement_intent,
        &"motor": motor,
        &"visual": visual,
        &"stats": stats,
    }
)
~~~

この方法なら、Stateは特定のPlayerインスタンスや固定パスではなく、必要な型との契約へ依存します。

依存が増えすぎた場合はDictionaryを拡張し続けず、PlayerStateContextなどの型付きContextオブジェクトを作ると、必須項目と型を明確にできます。

## 継承による共通State

複数Stateで共通する処理を上位Stateへ置けます。

~~~text
State
└── PlayerState
    └── GroundedState
        ├── IdleState
        ├── WalkState
        └── RunState
~~~

GroundedStateには、接地中に共通する重力、落下検出、地面スナップ、共通アニメーション処理などを置けます。

~~~gdscript
class_name GroundedState
extends PlayerState

@export var fall_state: State


func check_fall_transition() -> bool:
    if body.is_on_floor():
        return false

    request_transition(fall_state)
    return true
~~~

具体Stateでは次のように利用します。

~~~gdscript
func physics_update(delta: float) -> void:
    if check_fall_transition():
        return

    # このState固有の処理
~~~

継承は「派生Stateが基底Stateとして扱っても契約を壊さない」範囲に留めます。継承階層が深くなり、どのsuperが何をするか追えなくなったら、共通処理をMotor、Visual、Abilityなどのコンポーネントへ移します。

## Dashへの拡張

DashはWalkと移動処理を共有しつつ、一定時間方向を固定し、ジャンプや通常入力を無視する例です。

~~~gdscript
class_name PlayerDashState
extends PlayerState

@export var idle_state: State
@export var walk_state: State
@export var dash_speed: float = 9.0
@export var dash_duration: float = 0.2

var _remaining_time: float
var _dash_direction: Vector2


func enter(
    previous_state: State,
    data: Dictionary = {}
) -> void:
    super(previous_state, data)

    _remaining_time = dash_duration
    _dash_direction = data.get(
        &"move_vector",
        movement_intent.get_move_vector()
    )

    if _dash_direction.is_zero_approx():
        _dash_direction = Vector2.DOWN

    _dash_direction = _dash_direction.normalized()


func physics_update(delta: float) -> void:
    _remaining_time -= delta
    motor.move(_dash_direction, dash_speed, delta)

    if _remaining_time > 0.0:
        return

    if movement_intent.get_move_vector().is_zero_approx():
        request_transition(idle_state)
    else:
        request_transition(walk_state)
~~~

Dash方向を遷移データで渡すことで、遷移前に決定した方向をState間で共有できます。

## 並行StateMachine

1つのFSMは排他的です。WalkとAttackを同時に有効にしたいとき、WalkAttack、RunAttack、JumpAttackという全組み合わせを作るとState数が爆発します。

独立性の高い状態軸は別のFSMにします。

~~~text
Player
├── LocomotionStateMachine
│   ├── Idle
│   ├── Walk
│   ├── Run
│   └── Dash
└── ActionStateMachine
    ├── Free
    ├── Attack
    ├── Interact
    └── Cast
~~~

Actorは両方へ処理を委譲します。

~~~gdscript
func _unhandled_input(event: InputEvent) -> void:
    action_state_machine.handle_input(event)
    locomotion_state_machine.handle_input(event)


func _process(delta: float) -> void:
    locomotion_state_machine.update(delta)
    action_state_machine.update(delta)


func _physics_process(delta: float) -> void:
    locomotion_state_machine.physics_update(delta)
    action_state_machine.physics_update(delta)
~~~

### 並行FSMの所有権

両方のFSMが同じプロパティを書き換えると、呼び出し順で結果が変わります。各FSMが所有するものを先に決めます。

| StateMachine | 所有するもの | 直接書き換えないもの |
|---|---|---|
| Locomotion | 通常移動意図、移動速度、Facing | 攻撃HitBox、攻撃コンボ |
| Action | 攻撃段階、HitBox、行動ロック | move_and_slide、通常速度 |
| Status | Stun、Dead、無敵 | 通常入力の細部 |

攻撃中に移動速度を落とす場合、ActionStateがvelocityを直接上書きするのではなく、MovementModifierへ倍率を登録し、LocomotionまたはMotorが最終速度を計算します。

## FSM間の協調

移動中だけ攻撃できる、重攻撃中はDashできない、といった規則には複数の方法があります。

単純な場合:

~~~gdscript
if (
    action_state_machine.active_state == free_action_state
    and movement_intent.wants_run()
):
    request_transition(run_state)
~~~

規則が増えた場合は、CanAct、MovementLock、AbilityGateなどの専用コンポーネントへ許可判断を集めます。

~~~gdscript
if ability_gate.can_start(&"dash"):
    request_transition(dash_state)
~~~

FSM同士が互いの具体Stateを大量に参照し始めたら、分割したはずの状態軸が再び密結合しています。共通の能力・ロック表現へ引き上げます。

## 共有データの置き場所

データの寿命で置き場所を決めます。

| データ | 寿命 | 推奨場所 |
|---|---|---|
| HP、最大HP | Actor生存中 | Healthコンポーネント |
| Stamina | Actor生存中 | Stats / Staminaコンポーネント |
| 攻撃クールダウン | Stateをまたぐ | AbilityCooldowns |
| Dash残り時間 | Dash中だけ | DashStateのフィールド |
| Knockback方向 | 遷移時だけ | 遷移data |
| 現在State | Machine生存中 | StateMachine |
| 衣服状態 | Actor生存中 | Visual |

StateMachineに万能Dictionaryを置く方法は試作では便利ですが、キー名のタイプミスや型の不一致が実行時まで分かりません。永続データは可能な限り型付きコンポーネントにします。

## 外部割り込みと優先度

Damage、Stun、Dead、Cutsceneなどは、通常遷移より優先されることがあります。

最初は次のような優先規則を文章で決めます。

~~~text
Dead > Cutscene > Stun > Knockback > Attack > Dash > Run > Walk > Idle
~~~

規則が少ない間はHealthやActorからStateMachine.request_transitionを呼ぶだけで十分です。割り込みが競合するようになったら、遷移要求へpriorityを付け、物理フレーム末に最も高い要求を1つだけ適用する方式を検討します。

## enter・exit中の遷移と遷移キュー

シグナルは接続方法によっては現在の呼び出し中に遷移処理へ入ります。enter内でさらにrequest_transitionすると、遷移が入れ子になります。

最初の実装では次の規則を推奨します。

- enterとexitから遷移を要求しない。
- TimerやAnimation完了からの要求は、送信元がactive_stateか検証する。
- 1回のupdateまたはphysics_updateで要求は1つにする。
- 要求後はreturnする。

本当に必要なら、StateMachineにpending_transitionを1件だけ保存し、現在の遷移完了後または次の物理フレーム冒頭で適用します。無制限に連鎖適用しないよう、1フレームの遷移回数へ上限を設けます。

## ゲームプレイFSMとVisualの分離

ゲームプレイStateと表示Stateは一対一とは限りません。

例:

- Run中でも上半身はAttackを再生する。
- Walk中に衣服だけがOUTDOORからINDOORへ変わる。
- Stun中でもFacingは最後の方向を維持する。
- Deadではループしない死亡アニメーションへ固定する。

LocomotionStateがAnimatedSprite3DのSpriteFramesを直接交換するのではなく、Visualへ論理アニメーションを要求し、Visualが衣服・向き・装備から表示を解決すると、FSMの遷移グラフが見た目の差分で増えません。

## 階層StateMachineと継承は区別する

「階層StateMachine」という言葉は、次の2つの意味で使われることがあります。

1. Stateクラスの継承階層で処理を再利用する。
2. Groundedの内部にIdle / Walk / Runという子FSMを持つ、真の入れ子FSM。

動画のDash例は主に1の継承による再利用です。2の入れ子FSMは、上位Stateの共通enter/exit、イベント伝播、履歴復帰など追加規則が必要です。

まずは継承または並行FSMで十分か確認し、入れ子FSMは状態数と共通遷移が本当に多い場合に導入します。

---

[前章：Idle・Walk・Runの実装例](05_locomotion_example.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [07. テスト・デバッグ・失敗例](07_testing_and_debugging.md)
