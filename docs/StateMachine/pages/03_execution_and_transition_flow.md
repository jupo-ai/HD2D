# 03. 処理命令と状態遷移の流れ

[前章：Node型StateMachineの作り方](02_building_node_fsm.md) ｜ [目次へ](../README.md) ｜ [次章：移動コンポーネントの設計](04_movement_components.md)

**主な参考動画:** [HowToMakeAStateMachine.mp4](../HowToMakeAStateMachine.mp4)、[StarterStateMachine.mp4](../StarterStateMachine.mp4)

## 最も重要な原則

Stateは「遷移したい」と要求し、StateMachineが遷移を実行します。

具体Stateがactive_stateを直接書き換えたり、遷移先Stateのenterを直接呼んだりしてはいけません。exit、active_state更新、enterの順序をStateMachineへ集中させることで、どの遷移も同じライフサイクルを通ります。

## 起動時の処理順

~~~mermaid
sequenceDiagram
    participant Godot
    participant ActorNode as Actor
    participant Machine as StateMachine
    participant Idle

    Godot->>ActorNode: _ready()
    ActorNode->>Machine: initialize(actor, dependencies)
    Machine->>Idle: setup(actor, dependencies)
    Note over Machine: 全Stateをsetupし<br/>遷移シグナルを接続
    Machine->>Machine: _change_state(initial_state)
    Machine->>Machine: active_state = Idle
    Machine->>Idle: enter(null, {})
    Machine-->>ActorNode: 初期化完了
~~~

初期Stateにも通常の遷移関数を使用します。これにより、ゲーム開始時だけenterが呼ばれない、ログに初期Stateが残らない、といった特別扱いを避けられます。

## 物理フレームの処理順

~~~mermaid
flowchart TD
    A[GodotがActor._physics_processを呼ぶ] --> B[StateMachine.physics_update]
    B --> C[active_state.physics_update]
    C --> D[MovementIntentから意図を読む]
    D --> E[Stateが移動方針を決める]
    E --> F[Motorがvelocityを更新]
    F --> G[move_and_slide]
    G --> H{遷移条件を満たすか}
    H -- いいえ --> I[物理フレーム終了]
    H -- はい --> J[request_transition]
    J --> K[StateMachineが要求を検証]
    K --> L[旧State.exit]
    L --> M[active_stateを更新]
    M --> N[新State.enter]
    N --> I
~~~

CharacterBody3Dのmove_and_slideは、_physics_processまたはそこから呼ばれた処理で実行します。velocityは速度なので、目標速度そのものへdeltaを掛けません。加速量や重力を積算するときにはdeltaを使います。

また、is_on_floor、is_on_wall、is_on_ceilingは直前のmove_and_slideによる衝突結果を表します。判定とmove_and_slideの順序を決め、プロジェクト内で統一します。

## シグナル遷移のシーケンス

~~~mermaid
sequenceDiagram
    participant Walk
    participant Machine as StateMachine
    participant Run
    participant Observer as Debug / Visual

    Walk->>Walk: run入力を検出
    Walk->>Machine: transition_requested(Walk, Run, data)
    Machine->>Machine: Walkがactiveか検証
    Machine->>Walk: exit(Run)
    Machine->>Machine: active_state = Run
    Machine->>Run: enter(Walk, data)
    Machine->>Observer: state_changed(Walk, Run)
    Machine-->>Walk: emit呼び出しへ戻る
~~~

Godotの通常のシグナル接続では、emitすると接続先Callableが呼び出されます。そのため、emitが戻った時点ですでにactive_stateが別Stateへ変わっていることがあります。

具体Stateでは、遷移要求後に必ずreturnする習慣を付けます。

~~~gdscript
func physics_update(delta: float) -> void:
    if movement_intent.wants_run():
        request_transition(run_state)
        return

    # 遷移後に旧Stateの処理を続けない
    motor.move(input_vector, walk_speed, delta)
~~~

returnしないと、Runへenterした直後にWalk側のアニメーションやvelocityが上書きされることがあります。

## 1フレームの中で誰が何回呼ばれるか

基準構成では次のようになります。

| コールバック | 呼び出し元 | 回数 | 用途 |
|---|---|---:|---|
| handle_input | Actor._unhandled_input | 入力イベントごと | 押下・解放イベント |
| update | Actor._process | 描画フレームごとに1回 | 非物理処理 |
| physics_update | Actor._physics_process | 固定物理フレームごとに1回 | 移動・衝突 |
| enter | StateMachine | Stateへ入るときに1回 | 初期化 |
| exit | StateMachine | Stateを出るときに1回 | 後始末 |

通常更新と物理更新の両方で同時に遷移条件を判定すると、同じ描画フレーム中に2回遷移する可能性があります。物理状態に関する遷移はphysics_update、UIや非物理タイマーに関する遷移はupdateというように、判定場所を決めます。

## 遷移の正しい順序

このガイドでは次の順序を採用します。

1. 遷移先が有効か検証する。
2. 同じStateへの遷移なら終了する。
3. previous_stateを保存する。
4. previous_state.exit(next_state)を呼ぶ。
5. active_stateをnext_stateへ更新する。
6. next_state.enter(previous_state, data)を呼ぶ。
7. state_changedを通知する。

active_stateを更新してからenterを呼ぶため、enter内でStateMachineの現在Stateを確認すると自分自身になります。

exit内で遷移を要求したり、enter内で即座に別Stateへ遷移したりすると、遷移が入れ子になります。最初の実装では禁止し、必要になったら第6章の「割り込みと遷移キュー」を検討します。

## 遷移方式の比較

動画では主に2種類の方式が示されています。

### Stateが次Stateを返す方式

~~~gdscript
func physics_update(delta: float) -> State:
    if should_run:
        return run_state

    motor.move(input_vector, walk_speed, delta)
    return null
~~~

StateMachineは戻り値を受け取り、nullでなければ遷移します。

利点:

- 制御フローが戻り値として見える。
- 1回の更新につき遷移要求を1つに制限しやすい。
- 同期処理が単純でテストしやすい。

欠点:

- Timer、Animation、外部ダメージなど、更新関数外からの遷移に別経路が必要になる。
- 遷移先に加えてデータを渡すと戻り値用オブジェクトが必要になる。

### シグナルで遷移を要求する方式

~~~gdscript
func physics_update(_delta: float) -> void:
    if should_run:
        request_transition(run_state)
        return
~~~

利点:

- 入力、Timer、アニメーション完了、検出シグナルから同じ経路で要求できる。
- StateとStateMachineの結合を弱めやすい。
- 遷移元、遷移先、付加データをまとめて通知できる。

欠点:

- emit後も旧Stateの処理が続くため、returnが必要になる。
- 遷移要求が同一フレームに複数届く場合の規則が必要になる。
- 接続忘れや、非アクティブStateからの遅延要求を検証する必要がある。

どちらを選んでも、exit → active更新 → enterの実行責任はStateMachineへ集中させます。

## 遷移データ

Stateのフィールドへ一時値を書いてから遷移するより、遷移要求にデータを添える方が、値の寿命と受け渡し元が明確になります。

~~~gdscript
request_transition(
    knockback_state,
    {
        &"direction": hit_direction,
        &"strength": 6.0,
        &"source": attacker,
    }
)
return
~~~

遷移先ではenterで受け取ります。

~~~gdscript
func enter(
    _previous_state: State,
    data: Dictionary = {}
) -> void:
    var direction: Vector3 = data.get(
        &"direction",
        Vector3.ZERO
    )
    var strength: float = data.get(&"strength", 0.0)
    body.velocity = direction * strength
~~~

Dictionaryは小さな受け渡しには便利ですが、恒久的なキャラクターデータには使いません。HPやスタミナは型付きコンポーネントへ置きます。

## 外部イベントからの遷移

死亡、カットシーン開始、強制ワープなどは、現在State自身が検出しない場合があります。その場合はActorやHealthコンポーネントからStateMachineのrequest_transitionを呼びます。

~~~gdscript
func _on_health_depleted() -> void:
    locomotion_state_machine.request_transition(dead_state)
~~~

外部遷移も直接active_stateを書き換えず、同じ遷移関数を通します。

頻繁な割り込み条件をすべてのStateへ重複記述する必要が出たら、StateMachine側のガード、能力コンポーネント、上位StateMachineのいずれかへ引き上げる合図です。

## 更新直後に新Stateも動かすか

Walkへの遷移をIdle.physics_update中に要求した場合、そのフレームではWalk.enterまで実行されますが、Walk.physics_updateは次の物理フレームからです。

これは制御を単純に保つための推奨動作です。新Stateも同じフレームにphysics_updateすると、遷移が連鎖して無限ループになる危険があります。

入力反応の1物理フレーム遅延が問題になる場合は、次のいずれかを選びます。

- State外のMotorを毎フレーム必ず実行し、Stateは目標速度だけを設定する。
- 遷移前のStateでもそのフレームの移動を適用する。
- enterで最初の速度やアニメーションを設定する。

新Stateのphysics_updateを即座に再帰呼び出しする方法は、明確な回数上限がない限り避けます。

## 公式仕様で確認しておく点

- [Signal](https://docs.godotengine.org/en/stable/classes/class_signal.html) — connectとemit
- [Idle and Physics Processing](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html) — 通常更新と固定物理更新
- [CharacterBody3D](https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html) — move_and_slideと接地判定

---

[前章：Node型StateMachineの作り方](02_building_node_fsm.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [04. 移動コンポーネントの設計](04_movement_components.md)
