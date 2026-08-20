# HD-2D技術解説 — 『OCTOPATH TRAVELER』開発講演からGodot 4へ

![HD-2Dの定義](screenshot/001_hd2d_definition.jpg)

本書は、Unreal Fest East 2018の講演映像「Nintendo Switch『OCTOPATH TRAVELER』はこうして作られた」を、HD-2Dゲームを実際に設計・実装するための技術資料として再構成したものです。逐語的な書き起こしではなく、映像で示された判断と改善過程を抽出し、このプロジェクトの **Godot 4.7 / Forward+** 環境へ読み替えています。

> [!IMPORTANT]
> 講演で使われたUE4の機能名とGodotの機能名は一対一ではありません。本書では、見た目や設計上の目的が同じになる代替手段を示します。数値は完成形のプリセットではなく、対象シーンと実機で測定するための出発点です。

## HD-2Dとは何か

講演ではHD-2Dを、ドット絵の進化系として、昔ながらのドット絵に3DCGの画面効果を加えた幻想的な世界と説明しています。重要なのは、ローポリゴンの3D空間にピクセルキャラクターを置くだけでは完成しない点です。

HD-2Dらしさは、次の要素をひとつの画面設計へ統合した結果として現れます。

- 輪郭と動きが明快なピクセルキャラクター
- 奥行き、遮蔽、パースを持つ3D舞台
- 素材感を補うライティングと局所的な発光
- 被写界深度、グロー、カラー調整などの撮影的な演出
- キャラクターと背景のスケール、密度、情報量の意図的な調停

したがって本書では、HD-2Dを特定のシェーダーではなく、**アセット、カメラ、光、ポストプロセス、データ管理を横断するアートディレクション方式**として扱います。

## 目次

| 章 | 動画のおおよその範囲 | 主題 | Godotでの焦点 |
|---|---:|---|---|
| [01. HD-2Dの画面設計](pages/01_hd2d_design.md) | 03:00–12:20 | スケールの違和感、ドット感、試作から製品版 | Camera3D、Sprite3D、テクスチャ取込、画面評価 |
| [02. キャラクターとマテリアル](pages/02_character_and_material.md) | 12:20–15:20 | モーション量産、共通マテリアル | AnimatedSprite3D、SpriteFrames、ShaderMaterial |
| [03. マテリアル／シェーダー表現](pages/03_shader_techniques.md) | 15:20–21:30 | World UV、アルファ、法線、揺れ、グラデーション、発光 | spatial shader、頂点カラー、instance uniform |
| [04. ポストプロセス](pages/04_post_processing.md) | 21:30–25:10 | Bloom、DOF、レンズ表現、環境光、ビネット | WorldEnvironment、CameraAttributes、Compositor |
| [05. 地形・町・室内](pages/05_world_building.md) | 25:10–31:25 | グレーモデル、モジュール、地形、川、室内切替 | GridMap、PackedScene、Path3D、MultiMesh |
| [06. バトル演出](pages/06_battle_direction.md) | 31:25–35:10 | カメラ、専用ライト、動的ポストプロセス | AnimationPlayer、Tween、専用Environment |
| [07. 起動時間と参照設計](pages/07_loading_and_references.md) | 36:10–45:30 | ハード参照、ソフト参照、常駐範囲 | ResourceLoader、非同期ロード、Autoload最小化 |
| [08. スプライトアニメーションのメモリ](pages/08_sprite_memory.md) | 45:30–52:00 | Flipbookの保持、テーブル化、断片化 | アトラス、SpriteFrames、Resourceの寿命 |
| [09. オブジェクト寿命と性能計測](pages/09_lifecycle_and_profiling.md) | 52:00–63:10 | GCヒッチ、検索・削除コスト、オブジェクト数 | queue_free、RefCounted、ObjectDB Profiler、MultiMesh |

## UE4からGodot 4への対応表

| 講演中のUE4概念 | Godot 4での主な対応 | 読み替え時の注意 |
|---|---|---|
| PaperFlipbook / PaperSprite | `AnimatedSprite3D` / `SpriteFrames` | 1フレーム1画像ではなく、アトラス共有を基本にする |
| Material / Material Instance | `ShaderMaterial` / `instance uniform` | 共有Materialを複製しすぎず、個体差はインスタンス値へ寄せる |
| World Position Offset | spatial shaderの`vertex()`で`VERTEX`を変形 | ローカル座標かワールド座標かを明示する |
| PostProcessVolume | `WorldEnvironment`、`Camera3D.environment`、`Compositor` | Godotの環境は優先順位方式。空間ブレンドは自前制御が必要 |
| Landscape | インポートMesh、`GridMap`、手続きMesh、必要に応じた地形アドオン | 標準にUE Landscape相当の統合地形編集機能はない |
| Spline river | `Path3D` + カーブ追従Mesh生成 | Path3D単体は描画しない。断面からMeshを生成する |
| Hard / Soft Reference | `preload`・直接Resource参照 / パス・UID + `ResourceLoader` | 参照グラフとResourceキャッシュを意識する |
| Level | `PackedScene`とシーンツリーの枝 | 常駐シーンと遷移シーンを分離する |
| UObject GC | `Node.queue_free()`、`RefCounted`、C# GC | GDScriptではUE型の一括GCを前提にしない |
| Obj List / GCログ | Debugger、Monitors、Video RAM、ObjectDB Profiler | スナップショット差分と再現可能な操作列で調べる |

## このプロジェクトでの推奨レイヤー

```text
Main（常駐を最小化）
├── Services           入力、保存、音声など。コンテンツを直接参照しない
├── TransitionLayer    フェードとロード進捗
└── SceneSlot
    ├── FieldScene     町・フィールド・ダンジョンのいずれか
    └── BattleScene    必要な期間だけインスタンス化
```

アート側はさらに、`WorldEnvironment`、`CameraRig`、`LightingRig`、`WorldGeometry`、`Characters`、`Effects`を分けます。これにより、画づくりの調整対象とロード境界が一致しやすくなります。

## 制作時の基本原則

1. **基準ショットを先に固定する。** 解像度、カメラ、キャラクターの画面上の高さ、基準となる光を決めてから量産します。
2. **ピクセル素材と3D背景を別々に完成させない。** 同じカメラで並べ、密度とスケールを毎回確認します。
3. **ポストプロセスは素材を救う工程にしない。** Base Colorとライティングで読める画面を作り、最後に演出意図を加えます。
4. **参照と常駐範囲をアセット制作と同時に設計する。** 見た目が同じでも、読み込み単位でファイルを分けます。
5. **最適化は計測値と再現手順を残す。** 起動、フィールド、戦闘、メニューを同じ操作列で比較します。

## 資料の扱い

スクリーンショットは同梱動画から技術解説用に抽出したものです。画像内の著作物および商標の権利は各権利者に帰属します。配布・公開時は、元動画の利用条件とプロジェクトの公開範囲を別途確認してください。

### 参照動画

- [参考_OctopathTraveler_HD2D.mp4](参考_OctopathTraveler_HD2D.mp4)

### Godot公式資料

- [Environment and post-processing](https://docs.godotengine.org/en/latest/tutorials/3d/environment_and_post_processing.html)
- [Spatial shaders](https://docs.godotengine.org/en/latest/tutorials/shaders/shader_reference/spatial_shader.html)
- [ResourceLoader](https://docs.godotengine.org/en/latest/classes/class_resourceloader.html)
- [Debugger panel](https://docs.godotengine.org/en/latest/tutorials/scripting/debug/debugger_panel.html)

---

**次のページ:** [01. HD-2Dの画面設計](pages/01_hd2d_design.md)
