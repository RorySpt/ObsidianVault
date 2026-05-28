# 音频系统设计说明

音频系统把 SoLoud 封装成引擎子系统，对外暴露引擎语义，不让 gameplay、组件和工具代码直接依赖后端 API。

## 分层

```text
AudioComponent / AudioListenerComponent
    ↓
AudioSystem
    ↓
AudioScene
    ↓
AudioDevice
    ↓
AudioSoundRuntime / SoundAsset
    ↓
SoLoud
```

各层职责：

| 层 | 职责 |
|----|------|
| `AudioComponent` | 实体上的持续音源，表达播放意图和音源参数 |
| `AudioListenerComponent` | 实体上的监听者，通常挂在主摄像机上 |
| `AudioSystem` | 系统入口，处理命令队列、World 切换、Listener 同步、组件同步和 Bus 控制 |
| `AudioScene` | 保存单个 World 的音频组件、监听者、组件 voice handle 和运行时位置状态 |
| `AudioDevice` | 封装 SoLoud，负责设备初始化、Bus、voice handle 映射、3D 参数和后端调用 |
| `SoundAsset` | 音频资产配置，包括文件路径、流式加载、循环、3D 衰减参数 |
| `AudioSoundRuntime` | 将 `SoundAsset` 加载成 SoLoud `Wav` 或 `WavStream` |

## 播放模型

组件播放是异步入队：

```cpp
audio_component->play();
```

`play()` 会调用 `AudioSystem::play_component()`，只把命令写入队列。真正播放发生在 `AudioSystem::tick()` 中。这样可以避免组件生命周期回调中直接操作音频后端，也便于合并同一组件在一帧内的重复命令。

一次性音效使用同步接口：

```cpp
AudioVoiceHandle handle = g_engine->audio_system()->play_one_shot(sound, desc);
```

`play_one_shot()` 适合短音效、UI 音效、脚步、枪声、爆炸等不需要持续控制的声音。

## Voice Handle

`AudioVoiceHandle` 由 `id + generation` 组成：

```cpp
struct AudioVoiceHandle
{
    uint32_t id = 0;
    uint32_t generation = 0;
};
```

`AudioDevice` 内部维护引擎 handle 到 SoLoud handle 的映射。voice 播放结束后，`cleanup_runtime_state()` 会回收 id，并在复用时递增 generation，避免旧 handle 误命中新 voice。

## 组件状态

`AudioComponent` 维护播放状态：

```cpp
enum class EAudioPlaybackStatus : uint8_t
{
    Stopped,
    PlayPending,
    Playing,
    Paused,
    Stopping,
    FadingOut,
};
```

常用查询：

```cpp
audio->get_playback_status();
audio->is_playing();
audio->is_paused();
```

状态由 `AudioSystem` 根据命令处理、后端 voice 有效性和淡出流程维护。

## 3D 音频

3D 音源需要同时满足：

- `SoundAsset::is_3d() == true`
- `AudioComponent::spatialized_ == true`，或 `AudioPlayDesc::spatialized == true`

系统每帧同步：

- Listener 位置、朝向、速度
- 3D source 位置、速度
- min/max distance
- attenuation model

世界坐标会在 `AudioSystem` 中转换为音频坐标，并按 `k_unit_scale` 换算距离。

## Listener 选择

监听者选择规则：

1. 优先选择主摄像机实体上的 active `AudioListenerComponent`
2. 多个 listener 同时存在时，优先级高的胜出
3. 没有 listener 时 fallback 到主摄像机

通常把 listener 挂到主摄像机实体：

```cpp
AudioListenerComponent* listener = new_object<AudioListenerComponent>();
listener->set_priority(10);
camera_entity->add_component(listener);
```

## Bus

当前 Bus：

| Bus | 用途 |
|-----|------|
| `Master` | 全局音量 |
| `Music` | 背景音乐 |
| `Sfx` | 游戏音效和环境声 |
| `Ui` | 界面音效 |
| `Voice` | 语音、旁白、播报 |

每个 Bus 支持音量、静音和淡入淡出。`AudioMixerSnapshot` 可以保存和恢复所有 Bus 状态，适合暂停菜单、过场动画和 UI ducking。

## 格式支持

当前引擎通过 SoLoud `Wav` / `WavStream` 加载音频文件，实际支持：

| 格式 | 扩展名 | 说明 |
|------|--------|------|
| WAV | `.wav` | 低延迟，适合短音效和需要快速响应的声音 |
| Ogg Vorbis | `.ogg` | 体积小，适合 BGM、环境声和多数发布资源 |
| MP3 | `.mp3` | 体积小，适合长音频；循环点精度不如 wav/ogg 可控 |
| FLAC | `.flac` | 无损，适合高质量音频，但体积和解码成本更高 |

当前没有接入 FFmpeg 解码，也没有接入 `.aac`、`.m4a`、`.opus`、`.wma`、`.mid` 或 tracker/module 类格式。

## 当前边界

已经支持：

- 2D / 3D 播放
- 组件音源
- 一次性音效
- Bus 音量和静音
- Bus 淡入淡出
- Mixer Snapshot
- 组件淡出停止
- 预加载
- World 切换清理
- voice handle generation
- max active voices
- priority voice 保护

尚未完整支持：

- importer、转码和打包后的音频资源管线
- 引擎侧完整 voice budget 和优先级抢占策略
- 同类音效限频
- 音频事件系统
- 混响区、遮挡、传播
- 音频编辑器预览、波形、时长显示