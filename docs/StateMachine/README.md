# Godot 4 StateMachine 設計・実装ガイド

このドキュメントは、同梱された3本の参考動画を起点に、Godot 4.7で再利用可能なStateMachineを設計・実装するための技術資料として再構成したものです。

動画内のコードを順番に和訳したものではありません。動画で扱われているEnum方式、Node方式、状態遷移、依存性注入、移動入力のコンポーネント化、階層化、並行StateMachine、共有データという論点を整理し、3Dキャラクターにも適用できる一貫した設計へまとめています。

> [!IMPORTANT]
> この資料は、現在プロジェクト内にある作成途中のStateMachineを完成形とは見なしていません。これから実装方針を選択するための独立した設計資料です。

## 先に結論

このガイドで推奨する基本構成は、次のとおりです。

- 1つのStateMachineにつき、同時に有効なStateは1つだけにする。
- Stateは状態固有の判断を担当し、StateMachineは遷移の実行順序を管理する。
- Stateは遷移先へ直接切り替えず、遷移を要求する。
- キャラクター本体は、入力・フレーム更新・物理更新をStateMachineへ委譲する。
- 入力を直接読む処理と、CharacterBody3Dを実際に動かす処理を分ける。
- 体力、スタミナ、クールダウンなどの永続データをStateに所有させない。
- 移動と攻撃のように同時実行したい状態軸は、必要に応じて別々のStateMachineへ分ける。

~~~mermaid
flowchart LR
    Input[プレイヤー入力 / AI判断] --> Intent[MovementIntent]
    Intent --> State[現在のState]
    State -->|移動方針| Motor[CharacterMotor3D]
    Motor --> Body[CharacterBody3D]
    State -->|遷移要求| Machine[StateMachine]
    Machine -->|exit → 切替 → enter| State
    State --> Visual[Visual / Animation]
~~~

StateMachineはキャラクターの全機能を収容する箱ではありません。状態によって変わる判断だけを分離し、入力、物理移動、表示、能力値などは交換可能な依存コンポーネントとして扱うと、State数が増えても責務が崩れにくくなります。

## 目次

| 章 | 内容 | 読み終えたときに決められること |
|---|---|---|
| [01. FSMの基礎と方式選択](pages/01_foundations_and_design_choices.md) | FSMの成立条件、Enum方式とNode方式 | その対象にStateMachineが必要か、どの方式を使うか |
| [02. Node型StateMachineの作り方](pages/02_building_node_fsm.md) | State、StateMachine、所有者の完全な骨格 | 最小構成をゼロから実装する方法 |
| [03. 処理命令と状態遷移の流れ](pages/03_execution_and_transition_flow.md) | 初期化、入力、物理更新、シグナル、enter/exit | 1フレーム内で何がどの順に呼ばれるか |
| [04. 移動コンポーネントの設計](pages/04_movement_components.md) | 入力意図、移動方針、物理実行の分離 | PlayerとAIで同じStateを再利用する方法 |
| [05. Idle・Walk・Runの実装例](pages/05_locomotion_example.md) | 3Dキャラクター向けの具体例 | Inspectorの接続を含む実装手順 |
| [06. 拡張パターン](pages/06_extensions_and_composition.md) | 継承、並行FSM、共有データ、割り込み | Dash、Attack、Damage、AIへ拡張する方法 |
| [07. テスト・デバッグ・失敗例](pages/07_testing_and_debugging.md) | 不変条件、ログ、よくある不具合 | 実装を安全に検証する方法 |

## 実装方式の早見表

| 状況 | 推奨方式 |
|---|---|
| On / Off、3状態程度の砲台、短いUI状態 | Enum + match |
| Idle / Walk / Run / Jump / Fallなど、状態ごとに処理が増える | Node型StateMachine |
| 移動しながら攻撃する | 移動FSMと行動FSMを分ける |
| PlayerとAIで同じ移動Stateを使う | MovementIntentを交換する |
| DashがWalkとほぼ同じ処理を持つ | 共通基底Stateまたは継承を検討する |
| 体力、スタミナ、クールダウンを複数Stateで使う | Actorまたは専用コンポーネントに置く |
| 見た目のアニメーションだけを遷移させたい | AnimationTreeのStateMachineも検討する |

## 参考動画と、この資料で抽出した主題

時間は内容を探すためのおおよその目安です。

| 動画 | おおよその範囲 | この資料で扱う主題 |
|---|---:|---|
| [StarterStateMachine.mp4](StarterStateMachine.mp4) | 全編 | Enum方式、Node方式、Stateが次Stateを返す構成、Playerからの処理委譲 |
| [HowToMakeAStateMachine.mp4](HowToMakeAStateMachine.mp4) | 全編 | State共通インターフェース、Stateを子Nodeにする構成、シグナルによる遷移 |
| [AdvancedStateMachine.mp4](AdvancedStateMachine.mp4) | 前半 | 依存性注入、PlayerとAIで交換できる移動コンポーネント |
| [AdvancedStateMachine.mp4](AdvancedStateMachine.mp4) | 中盤 | 継承によるState再利用、Dashへの拡張 |
| [AdvancedStateMachine.mp4](AdvancedStateMachine.mp4) | 後半 | 並行StateMachine、共有データ、複数の状態軸 |

## この資料の基準環境

- Godot 4.7
- GDScript
- CharacterBody3D
- AnimatedSprite3Dなどの表示コンポーネント
- NodeをStateとして配置するState Pattern

2Dでも、Vector3をVector2へ、CharacterBody3DをCharacterBody2Dへ読み替えれば、設計上の流れは同じです。

## 公式資料

- [Idle and Physics Processing](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html)
- [Signal](https://docs.godotengine.org/en/stable/classes/class_signal.html)
- [CharacterBody3D](https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html)
- [Input](https://docs.godotengine.org/en/stable/classes/class_input.html)

---

**次のページ:** [01. FSMの基礎と方式選択](pages/01_foundations_and_design_choices.md)
