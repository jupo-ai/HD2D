# 02. プロジェクトとPlayerシーンの準備

[前章：全体設計と責務](01_architecture_and_responsibilities.md) ｜ [目次へ](../README.md) ｜ [次章：入力IntentとMotor](03_movement_intent_and_motor.md)

## この章の完了状態

この章ではスクリプトの中身をまだ実装しません。次を先に準備します。

- Input Mapへ初期操作を登録する。
- 共有コードとPlayer固有コードの保存先を作る。
- Playerシーンへ必要なNodeを順番に追加する。
- AnimatedSprite3DとCollisionShape3Dの基礎設定を行う。
- 後の章で設定するInspector参照を一覧化する。

## 0. 既存Playerを置き換える前の確認

既存Playerシーンを削除して作り直す場合でも、削除は参照先の切替後に行う方が安全です。

1. Gitで現在の変更を確認し、必要な作業を保存する。
2. 新しいPlayerを既存Playerとは別パスへ作る。
3. テスト用シーンで新Player単体を動かす。
4. MainやMapが参照するPackedSceneを新Playerへ変更する。
5. 外部NodePath、シグナル接続、Autoload参照を検索する。
6. 参照がなくなったことを確認してから旧Playerを削除する。

最初から旧Playerを削除する場合は、少なくともGit履歴から復元できる状態を確認してください。旧実装の構造を新設計へコピーする必要はありませんが、衝突寸法やSpriteのpixel_sizeなど、ゲーム固有の調整値を比較する資料にはできます。

## 1. Input Mapを登録する

このドキュメント作成時点では、プロジェクト固有のInput Actionは未登録です。Godotエディタで`Project > Project Settings > Input Map`を開き、次を追加します。

| Action | 初期割当例 | 用途 |
|---|---|---|
| `move_left` | A、左スティック左 | 画面左へ移動する意図 |
| `move_right` | D、左スティック右 | 画面右へ移動する意図 |
| `move_forward` | W、左スティック上 | 画面上へ移動する意図 |
| `move_back` | S、左スティック下 | 画面下へ移動する意図 |
| `run` | Shift、ゲームパッドボタン | 走りたい意図 |

将来用の`jump`や`dash`は、Stateを追加する章で登録します。未実装のActionを大量に先行登録する必要はありません。

### 入力名の意味

`move_forward`はワールドの-Z方向を意味しません。「現在のカメラで見た画面上方向」という操作意図です。PlayerMovementIntentがCamera3Dを使い、毎フレーム適切なワールドXZ方向へ変換します。

## 2. フォルダを作る

FileSystemドックまたはOSのファイル操作で、次の順にフォルダを用意します。

~~~text
res://entities/character/shared/state_machine/
res://entities/character/shared/movement/
res://entities/character/shared/locomotion/
res://entities/character/shared/visual/
res://entities/character/npc/
~~~

ファイルは後の章で次の順に作成します。

1. `state.gd`と`state_machine.gd`
2. `movement_intent_source.gd`
3. `player_movement_intent.gd`
4. `character_motor_3d.gd`
5. `camera_relative_facing_3d.gd`
6. `directional_sprite_animator_3d.gd`
7. `locomotion_state.gd`
8. `character_idle_state.gd`
9. `character_walk_state.gd`
10. `character_run_state.gd`
11. `player.gd`

基底クラスから具体クラスの順に作ると、`class_name`が未認識のために発生する一時的なParse Errorを減らせます。

## 3. StateMachine基盤を用意する

次の既存資料にあるコードを、共有フォルダへ実装します。

- [State基底クラス](../../StateMachine/pages/02_building_node_fsm.md#1-state基底クラス)
- [StateMachine](../../StateMachine/pages/02_building_node_fsm.md#2-statemachine)

保存先は次を推奨します。

~~~text
res://entities/character/shared/state_machine/state.gd
res://entities/character/shared/state_machine/state_machine.gd
~~~

StateMachine基盤はPlayer専用にしません。NPC、敵、ギミックにも同じ`State`と`StateMachine`を利用できます。

## 4. 新しいPlayerシーンを作る

1. `Scene > New Scene`を選ぶ。
2. ルートに`CharacterBody3D`を選ぶ。
3. Node名を`Player`にする。
4. `res://entities/character/player/player.tscn`として保存する。
5. 次の表の順番で子Nodeを追加する。

| 順番 | Node型 | Node名 | 後で割り当てるクラス |
|---:|---|---|---|
| 1 | CollisionShape3D | CollisionShape3D | なし |
| 2 | AnimatedSprite3D | AnimatedSprite3D | なし |
| 3 | Node | PlayerMovementIntent | PlayerMovementIntent |
| 4 | Node | CharacterMotor3D | CharacterMotor3D |
| 5 | Node | CameraRelativeFacing3D | CameraRelativeFacing3D |
| 6 | Node | DirectionalSpriteAnimator3D | DirectionalSpriteAnimator3D |
| 7 | Node | LocomotionStateMachine | StateMachine |
| 8 | Node | Idle | CharacterIdleState |
| 9 | Node | Walk | CharacterWalkState |
| 10 | Node | Run | CharacterRunState |

Idle、Walk、RunはLocomotionStateMachineの子にします。それ以外はPlayer直下です。

## 5. CollisionShape3Dを設定する

最初はCapsuleShape3Dを推奨します。

1. CollisionShape3Dの`Shape`へ`New CapsuleShape3D`を設定する。
2. キャラクターの足元がPlayer原点付近になるよう、CollisionShape3DのY位置を調整する。
3. Spriteの見た目ではなく、通路幅と段差に合う半径を決める。
4. Playerルートの`Floor Stop On Slope`、`Floor Max Angle`、`Floor Snap Length`は、テストマップの傾斜仕様に合わせて調整する。

CharacterBody3Dの`Motion Mode`は、重力と床判定を使うなら`Grounded`のままにします。完全なトップダウンで床や重力を使わないゲームに確定した場合だけ`Floating`を検討します。

## 6. AnimatedSprite3Dを設定する

| Inspector項目 | 初期方針 |
|---|---|
| Sprite Frames | Player用のSpriteFramesを1つ設定する |
| Billboard | Enabled |
| Pixel Size | 既存アートの解像度とワールド寸法に合わせる |
| Shaded | HD-2Dのライティング方針に合わせる |
| Alpha Cut | 透過境界と影の要件に合わせる |
| Position | 足元がPlayer原点へ合うようYを調整する |

Billboardを有効にしてSprite面をカメラへ向けることと、`idle_up`や`idle_down`を選ぶことは別問題です。Billboardは板ポリゴンの向きを変えます。CameraRelativeFacing3Dは、キャラクターのワールド上の向きがカメラからどう見えるかを判断します。

設定したSpriteFramesには、最初に次の12アニメーションを用意します。

| 論理動作 | アニメーション名 |
|---|---|
| Idle | `idle_down`、`idle_left`、`idle_right`、`idle_up` |
| Walk | `walk_down`、`walk_left`、`walk_right`、`walk_up` |
| Run | `run_down`、`run_left`、`run_right`、`run_up` |

初期実装では、このSpriteFramesを実行中に差し替えず、別のSpriteFramesを管理するResourceやIDも用意しません。

## 7. Camera3Dの前提

PlayerMovementIntentとCameraRelativeFacing3Dは、原則として次で現在のCamera3Dを取得します。

~~~gdscript
get_viewport().get_camera_3d()
~~~

このため、Playerシーンの子にCamera3Dを固定配置する必要はありません。Map側のCameraRig、追従Camera、カットシーンCameraがcurrentになれば、そのCamera3Dへ自動的に追従できます。

テストや特殊カメラ用には`camera_override`をInspectorまたはコードから指定できる設計にします。overrideがnullならViewportのcurrent cameraを使います。

## 8. 後で設定するInspector参照

全スクリプトを作成した後、次の参照を設定します。

| Node | プロパティ | 割当先 |
|---|---|---|
| CharacterMotor3D | body | Player |
| CameraRelativeFacing3D | camera_override | 通常は未設定 |
| DirectionalSpriteAnimator3D | sprite | AnimatedSprite3D |
| DirectionalSpriteAnimator3D | facing | CameraRelativeFacing3D |
| DirectionalSpriteAnimator3D | fallback_animation | `idle_down` |
| LocomotionStateMachine | initial_state | Idle |
| Idle | walk_state | Walk |
| Walk | idle_state | Idle |
| Walk | run_state | Run |
| Run | idle_state | Idle |
| Run | walk_state | Walk |

## 9. この段階のチェック

- [ ] PlayerルートがCharacterBody3Dである。
- [ ] Idle、Walk、RunだけがLocomotionStateMachine直下にある。
- [ ] Motor、Facing、DirectionalSpriteAnimator、IntentはPlayer直下にある。
- [ ] AnimatedSprite3DへPlayer用SpriteFramesを1つ設定した。
- [ ] SpriteFramesにIdle、Walk、Runの4方向アニメーションがある。
- [ ] Input Action名がコードの既定値と一致している。
- [ ] currentなCamera3Dを含むテスト用Worldを用意できる。
- [ ] 旧Playerを削除する前に、参照検索と復元手段を確認した。

---

[前章：全体設計と責務](01_architecture_and_responsibilities.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [03. 入力IntentとMotor](03_movement_intent_and_motor.md)
