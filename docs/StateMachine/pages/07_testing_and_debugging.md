# 07. テスト・デバッグ・失敗例

[前章：拡張パターン](06_extensions_and_composition.md) ｜ [目次へ](../README.md)

**関連する参考動画:** [StarterStateMachine.mp4](../StarterStateMachine.mp4)、[HowToMakeAStateMachine.mp4](../HowToMakeAStateMachine.mp4)、[AdvancedStateMachine.mp4](../AdvancedStateMachine.mp4)

## 実装前チェックリスト

- [ ] Stateの一覧と遷移図を書いた。
- [ ] 各Stateのenter、継続処理、exitを表にした。
- [ ] Enum方式では不足する理由がある。
- [ ] Stateに置かない永続データを決めた。
- [ ] Input、Motor、Visualの責務を決めた。
- [ ] move_and_slideを呼ぶ場所が1つに決まっている。
- [ ] 通常更新と物理更新のどちらで各遷移を判定するか決めた。
- [ ] 外部割り込みの入口を決めた。
- [ ] 複数FSMを使う場合、各FSMが所有するプロパティを決めた。

## 実行時に守る不変条件

StateMachineが正しく動いているかは、次の不変条件で確認できます。

1. 初期化後はactive_stateが必ず1つある。
2. active_stateはStateMachineの子である。
3. 非アクティブStateのupdateとphysics_updateは呼ばれない。
4. 遷移ごとに旧State.exitが1回、新State.enterが1回呼ばれる。
5. 同一Stateへの要求ではenterとexitを繰り返さない。
6. 非アクティブStateからの遷移要求は無視される。
7. 1物理フレームでmove_and_slideは最大1回だけ呼ばれる。
8. 遷移要求後に旧Stateの副作用が続かない。

## 遷移ログ

StateMachineのstate_changedを使うと、個々のStateへprintを散らさずに遷移を観測できます。

~~~gdscript
func _ready() -> void:
    locomotion.state_changed.connect(_on_state_changed)


func _on_state_changed(
    previous_state: State,
    current_state: State
) -> void:
    var previous_name: StringName = &"<none>"
    if previous_state != null:
        previous_name = previous_state.name

    print(
        "[Locomotion] %s -> %s"
        % [previous_name, current_state.name]
    )
~~~

ログにはMachine名、遷移元、遷移先、物理フレーム番号、理由を含めると追跡しやすくなります。

~~~text
[physics:1842][Locomotion] Walk -> Run reason=run_pressed
~~~

本番ビルドで不要なら、debugフラグまたはOS.is_debug_buildで出力を制御します。

## 遷移理由を残す

遷移dataへreasonを入れると、同じ遷移でも原因を区別できます。

~~~gdscript
request_transition(
    idle_state,
    {&"reason": &"movement_released"}
)
return
~~~

StateMachineがstate_changedとは別に詳細シグナルを持つ設計も可能です。

~~~gdscript
signal transition_completed(
    previous_state: State,
    current_state: State,
    data: Dictionary
)
~~~

## 最小テスト表

Idle、Walk、Runだけでも、次を手動または自動で確認します。

| 初期State | 操作・条件 | 期待State | 追加確認 |
|---|---|---|---|
| Idle | 移動入力あり | Walk | walk enterが1回 |
| Walk | 入力なし | Idle | 速度が減速する |
| Walk | runを押す | Run | Walk処理がemit後に続かない |
| Run | runを離す | Walk | runエフェクトがexitで止まる |
| Run | 入力なし | Idle | Walkを経由しない設計どおりか |
| 任意 | 同じStateを要求 | 変化なし | enter/exitが増えない |
| Walk | 非アクティブIdleが遷移要求 | Walk維持 | 要求が無視される |
| 任意 | 遷移先がnull | エラー検出 | 黙って停止しない |

## フレーム順を確認するテスト

Stateごとに一時的なイベント記録を入れ、遷移順序を検証できます。

~~~text
Walk.physics_update begin
Walk request Run
Walk.exit
Run.enter
StateMachine state_changed
Walk.physics_update return
~~~

Run.enterより後にWalkのvelocity設定やアニメーション再生が記録された場合、遷移要求後のreturnが抜けています。

## よくある不具合

### initial_stateが設定されていない

症状:

- 起動直後にnull参照エラーになる。
- updateが何も実行しない。

対策:

- Editorでinitial_stateを必須設定する。
- initialize時にnullをpush_errorまたはassertする。
- 暗黙に最初の子を選ぶ場合は、その規則をドキュメント化する。

### Stateではない子Nodeが混ざっている

症状:

- setupやシグナル接続時に存在しないメソッドを呼ぶ。

対策:

- child is Stateを確認する。
- StateMachineの直下はStateだけにする。
- 補助NodeはStateの子またはActor側へ置く。

### シグナルを接続していない

症状:

- Stateがrequest_transitionを呼んでも何も起こらない。

対策:

- initializeで全Stateの接続を行う。
- Signal.is_connectedで二重接続を防ぐ。
- 初期化完了フラグと構成ログを出す。

### 非アクティブStateから遷移する

症状:

- Timer終了後に突然以前のStateへ戻る。
- アニメーション完了が別State中に発火する。

対策:

- 遷移シグナルへfrom_stateを含める。
- from_state == active_stateを検証する。
- exitでTimer停止やシグナル解除を行う。

### emit後に処理が続く

症状:

- 新Stateのenter後に旧アニメーションへ戻る。
- Runへ入ったのにWalk速度で上書きされる。

対策:

~~~gdscript
request_transition(run_state)
return
~~~

### move_and_slideを複数箇所で呼ぶ

症状:

- 接地判定が不安定になる。
- ノックバックが消える。
- 速度が呼び出し順で変わる。

対策:

- CharacterMotorを物理移動の唯一の実行者にする。
- 並行FSMはMotorへ直接命令せず、ModifierやIntentを渡す。

### velocityへdeltaを掛ける

誤り:

~~~gdscript
body.velocity = direction * speed * delta
body.move_and_slide()
~~~

修正:

~~~gdscript
body.velocity = direction * speed
body.move_and_slide()
~~~

velocityは毎秒の速度であり、move_and_slideが物理ステップを扱います。

### is_on_floorを古い結果で判断する

is_on_floorは直前のmove_and_slideの衝突結果です。現在フレームで移動する前に参照する場合、それは前回物理フレームの接地状態です。

前回結果で事前判断するのか、move_and_slide後の結果で遷移するのかを意図的に選び、State間で統一します。

### StringNameのタイプミス

症状:

- 辞書から遷移先が取得できない。
- State名変更後だけ遷移しない。

対策:

- 小規模FSMでは型付きState参照を使う。
- 文字列方式では起動時に全遷移先を検証する。
- 警告だけで黙ってreturnせず、遷移元と要求名をログへ出す。

### ActorとStateMachineの両方が更新する

症状:

- タイマーが2倍で進む。
- 1回の入力で2回遷移する。
- move_and_slideが2回呼ばれる。

対策:

- Actorから委譲する方式か、StateMachine自身が_processを持つ方式か、どちらか一方にする。

## デバッグ表示

開発中はactive_state.nameを画面へ表示すると、入力と状態の不一致をすぐ発見できます。

表示候補:

- Locomotion active_state
- Action active_state
- move_vector
- velocity
- is_on_floor / is_on_wall
- wants_run / wants_jump
- 最後の遷移理由
- 1物理フレームのmove_and_slide呼び出し回数

デバッグUIはStateから直接更新せず、StateMachineやActorの観測値を読むだけにします。

## 拡張時の回帰テスト

### Dashを追加したとき

- Dash中に通常入力で方向が変わらない。
- Dash終了後に入力ありならWalk、なしならIdleへ戻る。
- Dash中の外部Stunが設計した優先度で割り込む。
- DashのTimerがexit後に遷移を要求しない。

### Attack FSMを追加したとき

- WalkとAttackが同時に有効になれる。
- 両FSMからmove_and_slideが重複して呼ばれず、Motorの1か所だけが実行する。
- 攻撃中移動倍率が呼び出し順に依存しない。
- Deadへ入ると両FSMの通常入力が停止する。

### AI入力へ交換したとき

- Stateコードを変更せずに移動する。
- targetがnullなら安全に停止する。
- AIのrun判断とPlayerのrun入力が同じState遷移を通る。

## 推奨する実装順

1. Idleだけで初期化、enter、updateを確認する。
2. IdleとWalkの双方向遷移を追加する。
3. Motorを追加し、move_and_slideを1物理フレーム最大1回にする。
4. MovementIntentをPlayer入力で実装する。
5. Runを追加する。
6. 遷移ログと不変条件の検証を追加する。
7. AI用MovementIntentへ交換して再利用性を確認する。
8. 必要になってからDash、Attack FSM、共有データを追加する。

一度にすべての拡張を入れず、各段階で処理順と所有権を確認します。

## 最終レビュー

実装が完成したら、次の問いに答えられる状態を目指します。

- 今のactive_stateはどれか。
- そのStateを誰がいつ更新するか。
- どの条件でどこへ遷移するか。
- exitとenterはどの順で呼ばれるか。
- 入力、物理移動、表示、能力値を誰が所有するか。
- 外部割り込みが通常遷移と競合したとき、どちらが勝つか。
- 非アクティブStateが遷移を要求したらどうなるか。
- 1フレームにmove_and_slideが何回呼ばれるか。

これらをコードを実行せずにScene Tree、遷移図、責務表から説明できれば、StateMachine周辺の命令の流れは十分に整理されています。

## 公式資料

- [Idle and Physics Processing](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html)
- [Signal](https://docs.godotengine.org/en/stable/classes/class_signal.html)
- [CharacterBody3D](https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html)

---

[前章：拡張パターン](06_extensions_and_composition.md) ｜ [目次へ](../README.md)
