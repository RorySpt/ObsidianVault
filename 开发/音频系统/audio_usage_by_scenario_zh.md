# 音频系统按场景使用指南

本文按实际用途说明音频系统应该怎么用。原则上：

- 只播放一次、无需后续控制的声音，优先使用 `play_one_shot()`
- 需要暂停、停止、淡出、循环、跟随实体的声音，使用 `AudioComponent`
- 长音频使用 `stream=true`
- 短音效使用 `stream=false`
- 需要空间定位的声音使用 `is_3d=true` 和 `spatialized=true`

## 支持格式

当前支持：

| 格式 | 扩展名 | 推荐用途 |
|------|--------|----------|
| WAV | `.wav` | 低延迟短音效、UI 音效、需要精确循环的资源 |
| Ogg Vorbis | `.ogg` | BGM、环境声、发布资源 |
| MP3 | `.mp3` | 长音频、临时资源、体积敏感资源 |
| FLAC | `.flac` | 无损音频、高质量源文件 |

当前不支持 `.aac`、`.m4a`、`.opus`、`.wma`、`.mid` 和 tracker/module 类格式。仓库里虽然有 FFmpeg，但音频系统现在不走 FFmpeg 解码。

## 背景音乐

适合关卡音乐、菜单音乐、环境长循环音乐。

```cpp
SoundAsset* music = new_object<SoundAsset>();
music->set_source_path("data/audio/music/level_theme.wav");
music->set_stream(true);
music->set_looping(true);
music->set_3d(false);

AudioComponent* audio = new_object<AudioComponent>();
audio->set_sound(music);
audio->set_bus_type(EAudioBusType::Music);
audio->set_spatialized(false);
audio->set_auto_play(true);
audio->set_play_on_begin(false);
entity->add_component(audio);
```

切换音乐时建议使用淡出淡入：

```cpp
g_engine->audio_system()->stop_component_with_fade(old_music_component, 1.0f);
new_music_component->play();
g_engine->audio_system()->fade_component_volume(new_music_component, 1.0f, 1.0f);
```

推荐配置：

| 配置 | 推荐值 |
|------|--------|
| Bus | `Music` |
| 格式 | `.ogg` / `.mp3` / `.flac` |
| `stream` | `true` |
| `looping` | `true` |
| `is_3d` | `false` |
| `spatialized` | `false` |

## UI 音效

适合按钮点击、菜单切换、弹窗、确认、取消、错误提示。

```cpp
AudioPlayDesc desc;
desc.bus_type = EAudioBusType::Ui;
desc.spatialized = false;
desc.volume = 1.0f;

g_engine->audio_system()->play_one_shot(click_sound, desc);
```

UI 音效通常不需要挂 `AudioComponent`，也不需要 3D。

推荐配置：

| 配置 | 推荐值 |
|------|--------|
| Bus | `Ui` |
| 格式 | `.wav` / `.ogg` |
| `stream` | `false` |
| `looping` | `false` |
| `is_3d` | `false` |
| 播放方式 | `play_one_shot()` |

## 普通 2D 音效

适合获得物品、任务完成、系统反馈、不需要空间定位的游戏音效。

```cpp
AudioPlayDesc desc;
desc.bus_type = EAudioBusType::Sfx;
desc.spatialized = false;
desc.volume = 1.0f;

g_engine->audio_system()->play_one_shot(reward_sound, desc);
```

如果声音需要后续淡出或停止，可以改用 `AudioComponent`。

## 瞬时 3D 音效

适合枪声、爆炸、脚步、撞击、开门、落地、命中特效。

```cpp
AudioPlayDesc desc;
desc.bus_type = EAudioBusType::Sfx;
desc.spatialized = true;
desc.position = hit_position;
desc.min_distance = 200.0f;
desc.max_distance = 4000.0f;
desc.volume = 1.0f;
desc.attenuation_model = EAudioAttenuationModel::InverseDistance;

g_engine->audio_system()->play_one_shot(impact_sound, desc);
```

这类声音不建议挂组件，除非它需要跟随实体或后续控制。

推荐配置：

| 配置 | 推荐值 |
|------|--------|
| Bus | `Sfx` |
| 格式 | `.wav` / `.ogg` |
| `stream` | `false` |
| `looping` | `false` |
| `is_3d` | `true` |
| 播放方式 | `play_one_shot()` |

## 固定 3D 环境音源

适合火焰、瀑布、发电机、机器运转、传送门、区域环境声。

```cpp
SoundAsset* ambient = new_object<SoundAsset>();
ambient->set_source_path("data/audio/ambient/waterfall.wav");
ambient->set_stream(true);
ambient->set_looping(true);
ambient->set_3d(true);
ambient->set_min_distance(300.0f);
ambient->set_max_distance(5000.0f);

AudioComponent* audio = new_object<AudioComponent>();
audio->set_sound(ambient);
audio->set_bus_type(EAudioBusType::Sfx);
audio->set_spatialized(true);
audio->set_auto_play(true);
entity->add_component(audio);
```

系统会按实体 `TransformComponent` 每帧同步声源位置。

推荐配置：

| 配置 | 推荐值 |
|------|--------|
| Bus | `Sfx` |
| 格式 | `.ogg` / `.wav` |
| `stream` | `true` |
| `looping` | `true` |
| `is_3d` | `true` |
| 播放方式 | `AudioComponent` |

## 移动物体持续音

适合车辆、飞行器、移动机械、角色身上的循环音。

```cpp
SoundAsset* engine_loop = new_object<SoundAsset>();
engine_loop->set_source_path("data/audio/sfx/engine_loop.wav");
engine_loop->set_stream(true);
engine_loop->set_looping(true);
engine_loop->set_3d(true);

AudioComponent* audio = new_object<AudioComponent>();
audio->set_sound(engine_loop);
audio->set_bus_type(EAudioBusType::Sfx);
audio->set_spatialized(true);
audio->set_auto_play(false);
entity->add_component(audio);

audio->play();
```

音源会跟随实体移动，并参与 3D 衰减和多普勒计算。

## 角色语音

适合 NPC 台词、角色喊话、世界中的语音。

```cpp
AudioPlayDesc desc;
desc.bus_type = EAudioBusType::Voice;
desc.spatialized = true;
desc.position = npc_position;
desc.priority = 1;

g_engine->audio_system()->play_one_shot(voice_line, desc);
```

重要语音可以设置 `priority > 0`，降低被 voice limit 抢占的概率。

## 全局语音和旁白

适合教程旁白、剧情解说、广播、系统播报。

```cpp
AudioPlayDesc desc;
desc.bus_type = EAudioBusType::Voice;
desc.spatialized = false;
desc.priority = 1;

g_engine->audio_system()->play_one_shot(narration, desc);
```

这类声音通常不需要空间化。

## 暂停和过场混音

适合暂停菜单压低背景音乐、过场动画降低音效、打开 UI 时降低环境声。

```cpp
AudioMixerSnapshot normal = g_engine->audio_system()->capture_mixer_snapshot();

g_engine->audio_system()->fade_bus_volume(EAudioBusType::Music, 0.25f, 0.3f);
g_engine->audio_system()->fade_bus_volume(EAudioBusType::Sfx, 0.5f, 0.3f);

g_engine->audio_system()->apply_mixer_snapshot(normal, 0.3f);
```

需要临时改变多个 Bus 时，优先使用 `AudioMixerSnapshot` 保存和恢复状态。

## Listener

通常把 `AudioListenerComponent` 挂到主摄像机实体上。

```cpp
AudioListenerComponent* listener = new_object<AudioListenerComponent>();
listener->set_priority(10);
camera_entity->add_component(listener);
```

选择规则：

- 优先使用主摄像机实体上的 listener
- 如果有多个 active listener，按 `priority` 选择
- 如果没有 listener，则 fallback 到主摄像机

## 预加载

需要避免首次播放卡顿时，可以提前加载音频运行时对象。

```cpp
g_engine->audio_system()->preload_sound(sound);
```

常见用法是在关卡初始化、UI 打开前、角色语音包加载后预加载。

## 选择 AudioComponent 还是 play_one_shot

| 场景 | 推荐方式 |
|------|----------|
| UI 点击 | `play_one_shot()` |
| 脚步、枪声、爆炸 | `play_one_shot()` |
| BGM | `AudioComponent` |
| 环境循环声 | `AudioComponent` |
| 跟随实体的循环音 | `AudioComponent` |
| 需要暂停、恢复、停止 | `AudioComponent` |
| 需要淡出停止 | `AudioComponent` |
| 不需要后续控制的短音效 | `play_one_shot()` |

## Bus 使用建议

| Bus | 用途 |
|-----|------|
| `Master` | 全局音量 |
| `Music` | 背景音乐 |
| `Sfx` | 游戏音效和环境声 |
| `Ui` | 界面音效 |
| `Voice` | 角色语音、旁白、播报 |

## 配置建议

`config/engine.yaml`：

```yaml
audio:
  enabled: true
  backend: "Miniaudio"
  sample_rate: 48000
  buffer_size: 2048
  max_active_voices: 64
  global_volume: 1.0
  music_volume: 1.0
  sfx_volume: 1.0
  ui_volume: 1.0
  voice_volume: 1.0
  doppler_factor: 1.0
  speed_of_sound: 343.3
```

## 常用调试命令

```text
audio.stop_all
audio.dump_stats
audio.master_volume
audio.master_volume 0.8
```