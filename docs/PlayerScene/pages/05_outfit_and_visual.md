# 05. 衣服とVisual

[前章：カメラ基準のFacing](04_camera_relative_facing.md) ｜ [目次へ](../README.md) ｜ [次章：Idle・Walk・RunとPlayer本体](06_locomotion_and_player.md)

## Visualが解決する組み合わせ

LocomotionStateは`idle`、`walk`、`run`という論理アニメーションだけを要求します。CharacterVisual3Dが次の3要素を組み合わせ、AnimatedSprite3Dへ渡す実アニメーションを決めます。

~~~text
論理動作 walk
+ 画面上の向き left
+ 衣服 indoor
= indoor用SpriteFramesの walk_left
~~~

Stateへ`walk_outdoor_left`のような名前を持たせません。衣服はSpriteFrames Resourceそのものを切り替え、全衣服で共通のアニメーション命名規則を使います。

## 1. アニメーション名を統一する

Outdoor、Indoor、Nakedの各SpriteFramesへ、最初は次の12アニメーションを作ります。

| 論理動作 | 必須アニメーション名 |
|---|---|
| Idle | `idle_down`、`idle_left`、`idle_right`、`idle_up` |
| Walk | `walk_down`、`walk_left`、`walk_right`、`walk_up` |
| Run | `run_down`、`run_left`、`run_right`、`run_up` |

命名規則を衣服ごとに変えないでください。Visualが衣服名をアニメーション名へ含める必要がなくなり、Swimsuitも同じ12名を用意するだけで追加できます。

## 2. CharacterOutfit Resourceを作る

作成先:

~~~text
res://entities/character/shared/visual/character_outfit.gd
~~~

~~~gdscript
class_name CharacterOutfit
extends Resource

@export var id: StringName
@export var sprite_frames: SpriteFrames
@export var fallback_animation: StringName = &"idle_down"
~~~

このResourceは衣服の定義であり、現在の衣服状態ではありません。現在どれを着ているかはCharacterVisual3Dが保持します。

### 衣服Resourceを3つ作る

1. FileSystemで`res://entities/character/player/outfits/`を開く。
2. 右クリックして`New > Resource`を選ぶ。
3. `CharacterOutfit`を選ぶ。
4. 次の表どおり保存して設定する。

| ファイル | id | sprite_frames | fallback_animation |
|---|---|---|---|
| `outdoor.tres` | `outdoor` | Outdoor用SpriteFrames | `idle_down` |
| `indoor.tres` | `indoor` | Indoor用SpriteFrames | `idle_down` |
| `naked.tres` | `naked` | Naked用SpriteFrames | `idle_down` |

## 3. CharacterVisual3Dを作る

作成先:

~~~text
res://entities/character/shared/visual/character_visual_3d.gd
~~~

~~~gdscript
class_name CharacterVisual3D
extends Node

signal outfit_changed(
    previous_outfit: StringName,
    current_outfit: StringName
)

@export var sprite: AnimatedSprite3D
@export var facing: CameraRelativeFacing3D
@export var outfits: Array[CharacterOutfit] = []
@export var initial_outfit: StringName = &"outdoor"

var _logical_animation: StringName = &"idle"
var _current_outfit: CharacterOutfit
var _current_outfit_id: StringName = &""


func initialize() -> void:
    assert(sprite != null, "CharacterVisual3D.spriteが未設定です")
    assert(facing != null, "CharacterVisual3D.facingが未設定です")

    facing.initialize()

    if set_outfit(initial_outfit):
        return

    if outfits.is_empty():
        push_error("CharacterVisual3D.outfitsが空です")
        return

    set_outfit(outfits[0].id)


func set_locomotion(logical_animation: StringName) -> void:
    if logical_animation == _logical_animation:
        return

    _logical_animation = logical_animation
    _refresh_animation(false, true)


## Cameraが回転しただけでも方向別アニメーションを更新する。
func refresh() -> void:
    if facing.refresh():
        _refresh_animation(true, false)


func set_outfit(outfit_id: StringName) -> bool:
    if outfit_id == _current_outfit_id:
        return true

    var next_outfit: CharacterOutfit = _find_outfit(outfit_id)
    if next_outfit == null:
        push_error("未登録の衣服IDです: %s" % outfit_id)
        return false

    if next_outfit.sprite_frames == null:
        push_error(
            "%sのSpriteFramesが未設定です" % outfit_id
        )
        return false

    var had_previous_outfit: bool = _current_outfit != null
    var previous_outfit_id: StringName = _current_outfit_id
    var previous_frames: SpriteFrames = sprite.sprite_frames
    var previous_animation: StringName = sprite.animation
    var previous_frame: int = sprite.frame
    var previous_frame_progress: float = sprite.frame_progress
    var was_playing: bool = sprite.is_playing()

    var previous_phase: float = _calculate_normalized_phase(
        previous_frames,
        previous_animation,
        previous_frame,
        previous_frame_progress
    )

    _current_outfit = next_outfit
    _current_outfit_id = next_outfit.id
    sprite.sprite_frames = next_outfit.sprite_frames
    facing.refresh()

    var desired_animation: StringName = _build_animation_name()
    var target_animation: StringName = _resolve_animation(
        desired_animation
    )

    # 同じ実アニメーションが旧・新SpriteFramesの両方にある場合だけ、
    # 再生位置を継承する。別のフォールバックへ移る場合は先頭から。
    var preserve_phase: bool = (
        had_previous_outfit
        and previous_frames != null
        and previous_frames.has_animation(desired_animation)
        and previous_animation == desired_animation
        and target_animation == desired_animation
    )

    var should_play: bool = true
    if had_previous_outfit:
        should_play = was_playing

    _play_at_phase(
        target_animation,
        previous_phase if preserve_phase else 0.0,
        should_play
    )

    outfit_changed.emit(
        previous_outfit_id,
        _current_outfit_id
    )
    return true


func get_outfit_id() -> StringName:
    return _current_outfit_id


func _refresh_animation(
    preserve_phase: bool,
    force_play: bool
) -> void:
    if sprite.sprite_frames == null:
        return

    var target_animation: StringName = _resolve_animation(
        _build_animation_name()
    )

    if target_animation == &"":
        sprite.stop()
        return

    if target_animation == sprite.animation:
        if force_play and not sprite.is_playing():
            sprite.play(target_animation)
        return

    var phase: float = 0.0
    if preserve_phase:
        phase = _calculate_normalized_phase(
            sprite.sprite_frames,
            sprite.animation,
            sprite.frame,
            sprite.frame_progress
        )

    _play_at_phase(
        target_animation,
        phase,
        sprite.is_playing() or force_play
    )


func _build_animation_name() -> StringName:
    return StringName(
        "%s_%s"
        % [_logical_animation, facing.get_animation_suffix()]
    )


func _resolve_animation(
    desired_animation: StringName
) -> StringName:
    var frames: SpriteFrames = sprite.sprite_frames
    if frames == null:
        return &""

    if frames.has_animation(desired_animation):
        return desired_animation

    var idle_same_direction: StringName = StringName(
        "idle_%s" % facing.get_animation_suffix()
    )
    if frames.has_animation(idle_same_direction):
        return idle_same_direction

    if (
        _current_outfit != null
        and frames.has_animation(
            _current_outfit.fallback_animation
        )
    ):
        return _current_outfit.fallback_animation

    if frames.has_animation(&"default"):
        return &"default"

    var names: PackedStringArray = frames.get_animation_names()
    if not names.is_empty():
        return StringName(names[0])

    push_error(
        "%sには再生可能なアニメーションがありません"
        % _current_outfit_id
    )
    return &""


func _play_at_phase(
    animation_name: StringName,
    normalized_phase: float,
    should_play: bool
) -> void:
    if animation_name == &"":
        sprite.stop()
        return

    sprite.play(animation_name)
    _apply_normalized_phase(
        sprite.sprite_frames,
        animation_name,
        normalized_phase
    )

    if not should_play:
        sprite.pause()


func _calculate_normalized_phase(
    frames: SpriteFrames,
    animation_name: StringName,
    frame_index: int,
    frame_progress: float
) -> float:
    if (
        frames == null
        or not frames.has_animation(animation_name)
    ):
        return 0.0

    var frame_count: int = frames.get_frame_count(
        animation_name
    )
    if frame_count <= 0:
        return 0.0

    var safe_frame: int = clampi(
        frame_index,
        0,
        frame_count - 1
    )
    var total_duration: float = 0.0
    var elapsed_duration: float = 0.0

    for index: int in range(frame_count):
        var duration: float = frames.get_frame_duration(
            animation_name,
            index
        )
        total_duration += duration

        if index < safe_frame:
            elapsed_duration += duration
        elif index == safe_frame:
            elapsed_duration += duration * clampf(
                frame_progress,
                0.0,
                1.0
            )

    if total_duration <= 0.0:
        return 0.0

    return clampf(
        elapsed_duration / total_duration,
        0.0,
        0.999999
    )


func _apply_normalized_phase(
    frames: SpriteFrames,
    animation_name: StringName,
    normalized_phase: float
) -> void:
    var frame_count: int = frames.get_frame_count(
        animation_name
    )
    if frame_count <= 0:
        return

    var total_duration: float = 0.0
    for index: int in range(frame_count):
        total_duration += frames.get_frame_duration(
            animation_name,
            index
        )

    if total_duration <= 0.0:
        sprite.set_frame_and_progress(0, 0.0)
        return

    var target_duration: float = clampf(
        normalized_phase,
        0.0,
        0.999999
    ) * total_duration
    var accumulated_duration: float = 0.0

    for index: int in range(frame_count):
        var duration: float = frames.get_frame_duration(
            animation_name,
            index
        )
        var is_last_frame: bool = index == frame_count - 1

        if (
            target_duration <= accumulated_duration + duration
            or is_last_frame
        ):
            var progress: float = 0.0
            if duration > 0.0:
                progress = (
                    target_duration - accumulated_duration
                ) / duration

            sprite.set_frame_and_progress(
                index,
                clampf(progress, 0.0, 1.0)
            )
            return

        accumulated_duration += duration


func _find_outfit(outfit_id: StringName) -> CharacterOutfit:
    for outfit: CharacterOutfit in outfits:
        if outfit != null and outfit.id == outfit_id:
            return outfit

    return null
~~~

## 4. 衣服変更時の再生位置

単純に`frame`だけをコピーすると、衣服ごとにフレーム数が違う場合に位置がずれたり、範囲外になったりします。この実装では、各フレームの相対durationを合計し、アニメーション全体の0～1の位置へ変換してから新しいSpriteFramesへ適用します。

| 条件 | 動作 |
|---|---|
| 新しいSpriteFramesにも同じ実アニメーションがある | 相対的な再生位置、再生中・停止中の状態を継承する |
| 同じ実アニメーションがない | 同方向のIdleからフォールバックを探し、先頭から再生する |
| 同方向Idleもない | Outfitのfallback_animationを使う |
| fallback_animationもない | `default`、それもなければ最初のアニメーションを使う |
| アニメーションが1つもない | エラーを出して停止する |

異なる意味のアニメーションへ移るときに再生位置を継承しないことが重要です。たとえば`run_left`がないため`idle_left`へフォールバックする場合、Runの75%地点をIdleへコピーせず、Idleの先頭から再生します。

## 5. Camera回転時の再生位置

カメラ回転で`walk_down`から`walk_left`へ変わる場合、`refresh()`は同じSpriteFrames内で相対位置を継承します。足運びの周期を保ったまま方向だけを切り替えられます。

一方、IdleからWalkのように論理動作が変わるとき、`set_locomotion()`は先頭から新アニメーションを再生します。

## 6. Inspector設定

CharacterVisual3Dで次を設定します。

| プロパティ | 値 |
|---|---|
| sprite | Player/AnimatedSprite3D |
| facing | Player/CameraRelativeFacing3D |
| outfits[0] | outdoor.tres |
| outfits[1] | indoor.tres |
| outfits[2] | naked.tres |
| initial_outfit | `outdoor` |

OutfitのIDは重複させません。起動時検証を追加する場合は、全Resourceのnull、空ID、重複ID、SpriteFrames未設定、必須12アニメーションを確認します。

## 7. 衣服を変更する呼び出し

屋外・屋内のAreaやメニューから、Visualの公開APIだけを呼びます。

~~~gdscript
visual.set_outfit(&"indoor")
visual.set_outfit(&"outdoor")
visual.set_outfit(&"naked")
~~~

LocomotionStateMachineのactive_stateは変更しません。Run中に衣服を変えてもRunのままで、表示だけが同じ`run_*`へ切り替わります。

## 8. 公式APIとの対応

- AnimatedSprite3Dの`frame_progress`は現在フレーム内の0～1の進捗です。
- `set_frame_and_progress()`は、frame設定時にprogressが0へ戻ることを避けて両方を設定できます。
- SpriteFramesの`get_frame_duration()`は各フレームの相対durationを返します。
- SpriteFramesの`get_frame_count()`と`has_animation()`で切替前に安全性を検証します。

詳細は[AnimatedSprite3D](https://docs.godotengine.org/en/stable/classes/class_animatedsprite3d.html)と[SpriteFrames](https://docs.godotengine.org/en/stable/classes/class_spriteframes.html)の公式リファレンスを参照してください。

## 9. 確認項目

- [ ] 3衣服で共通のアニメーション名を使っている。
- [ ] Walk途中の衣服変更で歩行周期が継続する。
- [ ] フレーム数が異なる衣服間でも範囲外にならない。
- [ ] 新しいSpriteFramesに現在アニメーションがなくてもエラー停止しない。
- [ ] フォールバック時は異なる動作の途中位置をコピーしない。
- [ ] カメラ回転による方向変更で再生周期が継続する。
- [ ] 衣服変更でLocomotionのactive_stateが変わらない。

---

[前章：カメラ基準のFacing](04_camera_relative_facing.md) ｜ [目次へ](../README.md) ｜ **次のページ:** [06. Idle・Walk・RunとPlayer本体](06_locomotion_and_player.md)
