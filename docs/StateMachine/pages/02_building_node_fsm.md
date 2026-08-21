# 02. Node型StateMachineの作り方

[前章：FSMの基礎と方式選択](01_foundations_and_design_choices.md) ｜ [目次へ](../README.md) ｜ [次章：処理命令と状態遷移の流れ](03_execution_and_transition_flow.md)

**主な参考動画:** [HowToMakeAStateMachine.mp4](../HowToMakeAStateMachine.mp4)、[StarterStateMachine.mp4](../StarterStateMachine.mp4)

## 完成時の構成

最小構成は、State基底クラス、StateMachine、具体State、Actorの4層です。

~~~text
Player（CharacterBody3D）
├── Visual
├── MovementIntent
├── CharacterMotor3D
└── LocomotionStateMachine
    ├── Idle
    ├── Walk
    └── Run
~~~

StateMachineの子NodeだけをStateとして扱います。別用途のNodeを混在させる場合は、初期化時に明示的に除外するか、構成エラーとして警告します。

## 1. State基底クラス

State基底クラスは、すべての具体Stateが同じ方法で呼び出せることを保証するインターフェースです。

~~~gdscript
class_name State
extends Node

signal transition_requested(
    from_state: State,
    to_state: State,
    data: Dictionary
)

var actor: Node
var dependencies: Dictionary


func setup(target_actor: Node, injected_dependencies: Dictionary) -> void:
    actor = target_actor
    dependencies = injected_dependencies


func enter(_previous_state: State, _data: Dictionary = {}) -> void:
    pass


func exit(_next_state: State) -> void:
    pass


func handle_input(_event: InputEvent) -> void:
    pass


func update(_delta: float) -> void:
    pass


func physics_update(_delta: float) -> void:
    pass


func request_transition(
    next_state: State,
    data: Dictionary = {}
) -> void:
    transition_requested.emit(self, next_state, data)
~~~

### 各メソッドの役割

| メソッド | 呼ばれるタイミング | 置く処理 |
|---|---|---|
| setup | StateMachine初期化時に1回 | Actorやコンポーネント参照の受け取り |
| enter | Stateが有効になった直後 | アニメーション開始、タイマー初期化 |
| exit | 別Stateへ移る直前 | 一時エフェクト停止、接続解除、後始末 |
| handle_input | 入力イベント受信時 | 押した瞬間だけ必要な操作 |
| update | 描画フレームごと | UI、非物理タイマー、表示処理 |
| physics_update | 固定物理フレームごと | velocity、衝突、移動判断 |
| request_transition | 遷移したいとき | StateMachineへ要求を通知 |

request_transitionを基底クラスに置くことで、基底シグナルは明示的に使用されます。また、具体Stateがシグナル名へ直接依存せず、後からログや遷移データ処理を追加できます。

Stateのスクリプトでは、組み込みの_processや_physics_processを定義しません。StateMachineが現在Stateだけを明示的に呼ぶため、非アクティブStateが裏で動くことを防げます。

## 2. StateMachine

StateMachineは遷移の唯一の実行者です。

~~~gdscript
class_name StateMachine
extends Node

signal state_changed(previous_state: State, current_state: State)

@export var initial_state: State

var active_state: State
var _initialized: bool = false


func initialize(
    actor: Node,
    dependencies: Dictionary = {}
) -> void:
    if _initialized:
        return

    for child: Node in get_children():
        if child is State:
            var state: State = child as State
            state.setup(actor, dependencies)

            if not state.transition_requested.is_connected(
                _on_transition_requested
            ):
                state.transition_requested.connect(
                    _on_transition_requested
                )
        else:
            push_warning(
                "%s はStateではないため初期化されません" % child.name
            )

    _initialized = true
    _change_state(initial_state, {})


func handle_input(event: InputEvent) -> void:
    if active_state != null:
        active_state.handle_input(event)


func update(delta: float) -> void:
    if active_state != null:
        active_state.update(delta)


func physics_update(delta: float) -> void:
    if active_state != null:
        active_state.physics_update(delta)


func request_transition(
    next_state: State,
    data: Dictionary = {}
) -> void:
    _change_state(next_state, data)


func _on_transition_requested(
    from_state: State,
    to_state: State,
    data: Dictionary
) -> void:
    if from_state != active_state:
        return

    _change_state(to_state, data)


func _change_state(
    next_state: State,
    data: Dictionary
) -> void:
    if not _initialized:
        push_error("StateMachineが初期化されていません")
        return

    if next_state == null:
        push_error("遷移先Stateがnullです")
        return

    if next_state.get_parent() != self:
        push_error(
            "%s はこのStateMachineの子ではありません"
            % next_state.name
        )
        return

    if next_state == active_state:
        return

    var previous_state: State = active_state

    if previous_state != null:
        previous_state.exit(next_state)

    active_state = next_state
    active_state.enter(previous_state, data)
    state_changed.emit(previous_state, active_state)
~~~

### なぜ遷移元を検証するのか

非アクティブStateから遅れてシグナルが届く可能性があります。たとえばState内でTimerやアニメーション完了シグナルを使っていると、exit後にコールバックが実行されることがあります。

from_state != active_stateを無視することで、以前のStateが現在の状態を上書きする事故を防ぎます。

### なぜState参照を使うのか

文字列やNode名で遷移先を探す方式もありますが、次の違いがあります。

| 指定方法 | 利点 | 欠点 |
|---|---|---|
| State参照 | 型検査、Inspectorで選択、改名に比較的強い | State同士の参照配線が増える |
| StringName | 疎結合に見える、辞書化しやすい | タイプミスが実行時まで分からない |
| NodePath | 階層を表現できる | Scene Tree変更の影響を受ける |
| 遷移ID Enum | グラフを集中管理しやすい | StateとIDの対応表が必要 |

このガイドでは、同一StateMachine内の小さなグラフにはState参照を使います。大規模なデータ駆動FSMが必要になった段階で、遷移表やResource化を検討します。

## 3. Actorから初期化して処理を委譲する

StateMachineがPlayerをget_parentで推測するのではなく、Actorから明示的に依存を渡します。

~~~gdscript
extends CharacterBody3D

@onready var state_machine: StateMachine = (
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
    state_machine.initialize(
        self,
        {
            &"movement_intent": movement_intent,
            &"motor": motor,
            &"visual": visual,
        }
    )


func _unhandled_input(event: InputEvent) -> void:
    state_machine.handle_input(event)


func _process(delta: float) -> void:
    state_machine.update(delta)


func _physics_process(delta: float) -> void:
    state_machine.physics_update(delta)
~~~

この形には次の利点があります。

- StateMachineがScene Treeの相対位置を仮定しない。
- Player用入力をAI用入力へ交換できる。
- 複数StateMachineを使う場合に、Actor側で呼び出し順を決められる。
- StateMachine自身の_processとActorからの手動呼び出しが重複しない。

StateMachine自身に_processと_physics_processを実装する方式も成立します。ただし、Actorからも呼ぶと1フレームに2回更新されるため、どちらか一方へ統一します。

## 4. Godotエディタでの組み立て手順

1. Actorの子にNodeを追加し、LocomotionStateMachineと命名する。
2. StateMachineスクリプトを割り当てる。
3. その子にIdle、Walk、RunというNodeを追加する。
4. 各NodeへStateを継承した具体スクリプトを割り当てる。
5. StateMachineのinitial_stateへIdleを設定する。
6. 各具体Stateのexportされた遷移先へ、Idle、Walk、Runを割り当てる。
7. Actorの_readyでStateMachineをinitializeする。
8. Actorの入力、通常更新、物理更新をStateMachineへ委譲する。

~~~text
LocomotionStateMachine
├── Idle
│   └── walk_state = ../Walk
├── Walk
│   ├── idle_state = ../Idle
│   └── run_state  = ../Run
└── Run
    ├── idle_state = ../Idle
    └── walk_state = ../Walk
~~~

## 初期化時の不変条件

実行前に、最低限次を満たしている必要があります。

- initial_stateがnullではない。
- initial_stateがStateMachineの子である。
- すべての具体StateがStateを継承している。
- 必須依存コンポーネントがinitializeで渡されている。
- StateMachineの更新方法が「Actorから委譲」か「自分で処理」のどちらか一方である。

開発中は、黙ってreturnするよりassertやpush_errorで構成ミスを早く見つける方が安全です。

---

[前章：FSMの基礎と方式選択](01_foundations_and_design_choices.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [03. 処理命令と状態遷移の流れ](03_execution_and_transition_flow.md)
