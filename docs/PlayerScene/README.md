# Playerシーン再構築ガイド

この資料は、既存のPlayerシーンを完成形として流用せず、Godot 4.7でPlayerをゼロから組み立て直すための設計・実装手順です。

[StateMachine設計・実装ガイド](../StateMachine/README.md)のNode型FSM、依存性注入、Intent・State・Motorの分離を前提にしながら、次の要件を具体的なシーン構成とGDScriptへ落とし込みます。

- ロコモーションはIdle、Walk、Runから始める。
- 衣服はOutdoor、Indoor、Nakedから始める。
- Playerのワールド上の向きを、現在のCamera3Dから見た上下左右へ変換して表示する。
- カメラが回転しただけでも、`idle_up`、`idle_down`などを正しく選び直す。
- Dash、Jump、Swimsuitを既存コードの分岐追加ではなく、StateやResourceの追加で拡張できるようにする。
- Player固有の入力だけを交換し、Motor、Facing、Visual、ロコモーションStateをNPCでも再利用できるようにする。

> [!IMPORTANT]
> このフォルダにあるのは実装仕様書です。この作業ではPlayerシーンやGDScript本体は作成しません。実装時は各章の順番でファイルとNodeを作成してください。

## 採用する設計

Playerの「移動状態」と「衣服状態」は、同じStateMachineへ入れません。Walk中にもOutdoorでいられるように、両者は独立した状態軸だからです。

| 状態軸 | 実装 | 初期要素 | 追加方法 |
|---|---|---|---|
| ロコモーション | Node型StateMachine | Idle、Walk、Run | DashStateやJumpStateを追加する |
| 衣服 | CharacterVisual3Dが保持するOutfit Resource | Outdoor、Indoor、Naked | Swimsuit用Resourceを登録する |
| ワールド上の向き | CameraRelativeFacing3D | 最後に移動した水平ベクトル | PlayerとNPCの移動意図から更新する |
| 画面上の向き | CameraRelativeFacing3DがCamera3D基準で算出 | Down、Left、Right、Up | 8方向化する場合だけ判定を拡張する |

~~~mermaid
flowchart LR
    PlayerInput[Player入力] --> PlayerIntent[PlayerMovementIntent]
    AiDecision[AI判断] --> AiIntent[NavigationMovementIntent]
    PlayerIntent --> Locomotion[Locomotion StateMachine]
    AiIntent --> Locomotion
    Locomotion --> Motor[CharacterMotor3D]
    Locomotion --> Facing[CameraRelativeFacing3D]
    Motor --> Body[CharacterBody3D]
    Facing --> Visual[CharacterVisual3D]
    Outfit[CharacterOutfit Resource] --> Visual
    Visual --> Sprite[AnimatedSprite3D]
~~~

## 実装する順番

順番には意味があります。下位の契約を先に作り、上位のStateやPlayerがそれを利用する形にします。

| 順番 | ページ | 完了条件 |
|---:|---|---|
| 1 | [01. 全体設計と責務](pages/01_architecture_and_responsibilities.md) | 状態軸、所有権、Scene Treeを説明できる |
| 2 | [02. プロジェクトとPlayerシーンの準備](pages/02_project_and_scene_setup.md) | Input Map、フォルダ、空のNode構成が決まる |
| 3 | [03. 入力IntentとMotor](pages/03_movement_intent_and_motor.md) | カメラ相対入力をワールド方向へ変換し、Motorを1回だけ実行できる |
| 4 | [04. カメラ基準のFacing](pages/04_camera_relative_facing.md) | カメラ回転を含めて上下左右を判定できる |
| 5 | [05. 衣服とVisual](pages/05_outfit_and_visual.md) | SpriteFrames切替、再生位置継承、フォールバックが動く |
| 6 | [06. Idle・Walk・RunとPlayer本体](pages/06_locomotion_and_player.md) | FSM、全コンポーネント、Playerの更新順がつながる |
| 7 | [07. 拡張・NPC化・テスト](pages/07_extensions_npc_and_tests.md) | Dash、Jump、Swimsuit、NPCへ安全に拡張できる |

## 完成時のScene Tree

~~~text
Player (CharacterBody3D) [player.gd]
├── CollisionShape3D
├── AnimatedSprite3D
├── PlayerMovementIntent (Node)
├── CharacterMotor3D (Node)
├── CameraRelativeFacing3D (Node)
├── CharacterVisual3D (Node)
└── LocomotionStateMachine (Node)
    ├── Idle (Node)
    ├── Walk (Node)
    └── Run (Node)
~~~

AnimatedSprite3Dは表示だけを担当します。衣服の選択、論理アニメーション名、画面上の向き、実際に再生するアニメーション名の解決はCharacterVisual3Dへ置きます。

## 基準環境

- Godot 4.7.2
- GDScript
- CharacterBody3D
- AnimatedSprite3DとSpriteFrames
- 4方向アニメーション
- XZ平面を水平移動、Y軸を上下方向として扱う3Dシーン

## 関連資料

- [StateMachine：Node型StateMachineの作り方](../StateMachine/pages/02_building_node_fsm.md)
- [StateMachine：処理命令と状態遷移の流れ](../StateMachine/pages/03_execution_and_transition_flow.md)
- [StateMachine：移動コンポーネントの設計](../StateMachine/pages/04_movement_components.md)
- [AnimatedSprite3D公式リファレンス](https://docs.godotengine.org/en/stable/classes/class_animatedsprite3d.html)
- [SpriteFrames公式リファレンス](https://docs.godotengine.org/en/stable/classes/class_spriteframes.html)
- [Camera3D公式リファレンス](https://docs.godotengine.org/en/stable/classes/class_camera3d.html)

---

**次のページ:** [01. 全体設計と責務](pages/01_architecture_and_responsibilities.md)
