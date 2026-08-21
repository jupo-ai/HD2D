# 01. FSMの基礎と方式選択

[目次へ](../README.md) ｜ [次章：Node型StateMachineの作り方](02_building_node_fsm.md)

**主な参考動画:** [StarterStateMachine.mp4](../StarterStateMachine.mp4)

## StateMachineが解決する問題

Finite State Machine（FSM）は、対象が取り得る状態を有限個に分け、現在有効な状態に対応する処理だけを実行する仕組みです。

たとえばキャラクターがIdle、Walk、Runのどれか1つだけであるなら、各フレームで大量のフラグを組み合わせて動作を推測する必要はありません。現在のStateを1つ保持し、そのStateへ更新を委譲できます。

FSMとして最低限必要なのは次の4要素です。

1. 有限個のStateが定義されている。
2. 1つのStateMachine内では、現在のStateが1つに決まる。
3. Stateごとの実行処理がある。
4. どの条件でどのStateへ移るかという遷移規則がある。

~~~mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Walk: move_vector != 0
    Walk --> Idle: move_vector == 0
    Walk --> Run: runを押している
    Run --> Walk: runを離した
    Run --> Idle: move_vector == 0
~~~

## Stateとフラグの違い

次のような値が増え続けたら、FSMを検討する合図です。

~~~gdscript
var is_idle: bool
var is_walking: bool
var is_running: bool
var is_attacking: bool
var is_stunned: bool
~~~

この構成では、is_idleとis_runningが同時にtrueになるなど、無効な組み合わせを防ぐ責任がコード全体へ分散します。

ただし、すべてのboolをStateへ変える必要はありません。is_on_floor、has_weapon、can_interactなどは、現在のモードではなく世界や能力の事実です。Stateとデータを区別します。

| 種類 | 例 | 主な置き場所 |
|---|---|---|
| 排他的な振る舞い | Idle、Walk、Run、Dash | StateMachine |
| 世界の事実 | is_on_floor、target_is_visible | Actorまたは検出コンポーネント |
| 能力値 | HP、stamina、move_speed | Statsなどのコンポーネント |
| 短い入力意図 | move_vector、wants_run | Input / Intentコンポーネント |
| 表示状態 | facing、clothing、current_animation | Visualコンポーネント |

## Enum方式とNode方式

### Enum + match

小規模な対象には、Enum方式が最小で分かりやすい選択です。

~~~gdscript
enum TurretState {
    SCANNING,
    FIRING,
    RECHARGING,
}

var state: TurretState = TurretState.SCANNING


func _physics_process(delta: float) -> void:
    match state:
        TurretState.SCANNING:
            _scan(delta)
        TurretState.FIRING:
            _fire(delta)
        TurretState.RECHARGING:
            _recharge(delta)
~~~

利点は、ファイル数と配線が少なく、処理を一か所で追えることです。欠点は、状態数やライフサイクル処理が増えると1つのスクリプトが肥大化することです。

### Node型StateMachine

各StateをNodeとして配置し、共通のState基底クラスを継承させます。

~~~text
Player
└── LocomotionStateMachine
    ├── Idle
    ├── Walk
    └── Run
~~~

利点は次のとおりです。

- 状態ごとにコードを分けられる。
- Inspectorから遷移先や調整値を設定できる。
- Scene Treeで利用可能なStateを確認できる。
- Stateの継承やコンポーネント注入により再利用しやすい。

一方で、単純な対象へ使うとファイルと参照が増え、処理を追いにくくなります。Node方式は常に上位互換ではありません。

## 方式を選ぶ判断基準

次の質問で2つ以上に該当するなら、Node方式を検討する価値があります。

- Stateごとにenterとexitの後始末が必要か。
- 状態固有の物理処理が数行を超えるか。
- 状態ごとに固有のexport値があるか。
- 同じStateを別キャラクターへ再利用したいか。
- Dash、Knockback、Stunなどが今後増えるか。
- 状態遷移をScene TreeやInspectorから確認したいか。

反対に、状態が2～3個で、各処理が短く、再利用もしない場合はEnum方式の方が保守しやすいことがあります。

## AnimationTreeのStateMachineとは別物

GodotのAnimationTreeにもStateMachineがありますが、主目的はアニメーションの遷移です。ゲームプレイ上のStateMachineとは責務が異なります。

たとえば、ゲームプレイStateがRunへ入ったとき、Visualコンポーネントを通じてAnimationTreeのrunへ遷移させる構成は有効です。しかし、AnimationTreeだけに「スタミナが切れたらRunからWalkへ戻る」といったゲームルールまで持たせると、ゲームプレイロジックと表示ロジックが混ざります。

## Stateを設計する前に表を書く

コードを書く前に、次の表を作ると遷移漏れを見つけやすくなります。

| State | enter | 継続処理 | exit | 遷移条件 |
|---|---|---|---|---|
| Idle | idle表示、速度を減速 | 入力を監視 | 特になし | 入力あり → Walk |
| Walk | walk表示 | 歩行速度で移動 | 特になし | 入力なし → Idle、runあり → Run |
| Run | run表示 | 走行速度で移動 | 走行エフェクト停止 | 入力なし → Idle、runなし → Walk |

この表の「継続処理」がほぼ同じStateは、データだけ変えればよい可能性があります。逆に、遷移条件やライフサイクルが大きく異なるなら、独立Stateにする意味があります。

## 責務の境界

推奨する責務分担は次のとおりです。

| 要素 | 担当 |
|---|---|
| StateMachine | 現在Stateの保持、更新の委譲、遷移順序、遷移検証 |
| State | 状態固有の判断、StateMachineへの遷移要求 |
| Actor | Nodeの所有、依存コンポーネントの初期化、更新順の決定 |
| MovementIntent | Player入力やAI判断を移動意図へ変換 |
| CharacterMotor | velocity更新とmove_and_slide |
| Visual | アニメーション名、向き、服装差分の解決 |
| Stats / Ability | HP、スタミナ、クールダウン、利用可否 |

StateがInput、CharacterBody、AnimatedSprite、HP、サウンドをすべて直接操作し始めると、再利用性が急速に下がります。最初から完全に抽象化する必要はありませんが、「状態固有の方針」と「どの状態でも共通する実行処理」は分けて考えます。

## 避けたい設計

- 現在Stateを複数のboolから毎フレーム推測する。
- State自身がactive_stateを書き換える。
- exitを呼ばずに別Stateのenterだけを呼ぶ。
- すべてのStateが個別にInputを読み、move_and_slideを呼ぶ。
- StateごとにHPやスタミナのコピーを持つ。
- 文字列だけで遷移先を指定し、タイプミスを実行時まで検出できない。
- Idle、Walk、Run、Attackの全組み合わせを1つのFSMへ作る。

## このガイドで採用する方式

以降は、次の特性を持つNode型FSMを基準にします。

- Stateは子Nodeとして配置する。
- Stateは共通インターフェースを継承する。
- StateからStateMachineへはシグナルで遷移を要求する。
- 遷移先は可能な限りState参照で指定する。
- StateMachineだけがexit、active_state更新、enterを実行する。
- Actorが入力・通常更新・物理更新をStateMachineへ委譲する。

動画では「Stateが次Stateを返す方式」と「シグナルで要求する方式」の両方が扱われています。どちらも成立しますが、この資料では外部イベントや並行StateMachineへ拡張しやすいシグナル方式を中心にし、方式比較は第3章で扱います。

---

[目次へ](../README.md) ｜ **次のページ:** [02. Node型StateMachineの作り方](02_building_node_fsm.md)
