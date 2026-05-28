# 音频系统 API 参考

## AudioSystem

获取入口：

```cpp
AudioSystem* audio_system = g_engine->audio_system();
```

### 组件播放

```cpp
void play_component(AudioComponent* component, bool restart = false);
void pause_component(AudioComponent* component, bool pause);
void stop_component(AudioComponent* component);
void replay_component(AudioComponent* component);
```

`play_component()` 是异步入队接口，不返回本次播放的新 handle。需要直接获得 handle 的短音效使用 `play_one_shot()`。

### 一次性播放

```cpp
AudioVoiceHandle play_one_shot(SoundAsset* sound, const AudioPlayDesc& desc = {});
bool preload_sound(SoundAsset* sound);
```

`play_one_shot()` 适合 UI、脚步、枪声、爆炸、短语音等不需要持续控制的声音。

### 淡入淡出

```cpp
void fade_voice_volume(AudioVoiceHandle handle, float target_volume, float duration);
void stop_voice_with_fade(AudioVoiceHandle handle, float duration);
void fade_component_volume(AudioComponent* component, float target_volume, float duration);
void stop_component_with_fade(AudioComponent* component, float duration);
```

组件进入 `FadingOut` 状态后，系统不会继续用组件音量覆盖淡出曲线。

### Bus 控制

```cpp
void set_master_volume(float volume);
float get_master_volume() const;

void set_bus_volume(EAudioBusType bus_type, float volume);
float get_bus_volume(EAudioBusType bus_type) const;
void fade_bus_volume(EAudioBusType bus_type, float target_volume, float duration);
void set_bus_muted(EAudioBusType bus_type, bool muted);
bool is_bus_muted(EAudioBusType bus_type) const;
```

音量小于 0 时会被截断到 0。

### Mixer Snapshot

```cpp
AudioMixerSnapshot capture_mixer_snapshot() const;
void apply_mixer_snapshot(const AudioMixerSnapshot& snapshot, float fade_duration = 0.0f);
```

用于临时改变多个 Bus 后恢复，例如暂停菜单和过场动画。

### 其他接口

```cpp
void stop_all();
void dump_stats();
bool is_audio_enabled() const;
bool should_allow_playback() const;

Delegate<AudioVoiceHandle> on_voice_finished;
```

编辑器中只有引擎处于 playing 状态时允许实际播放。

## AudioComponent

`AudioComponent` 挂在 Entity 上，用于持续控制一个音源。

```cpp
AudioComponent* audio = new_object<AudioComponent>();
audio->set_sound(sound);
audio->set_bus_type(EAudioBusType::Sfx);
audio->set_spatialized(true);
entity->add_component(audio);
```

### 播放控制

```cpp
audio->play();
audio->pause();
audio->resume();
audio->stop();
audio->replay();
```

这些接口最终进入 `AudioSystem` 命令队列。

### 属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `sound_` | `SoundAsset*` | `nullptr` | 音频资产 |
| `auto_play_` | `bool` | `true` | begin 时播放，activate 时也会播放 |
| `play_on_begin_` | `bool` | `false` | 仅 begin 时播放一次 |
| `override_looping_` | `bool` | `false` | 是否覆盖资产循环设置 |
| `loop_override_` | `bool` | `false` | 覆盖后的循环值 |
| `spatialized_` | `bool` | `true` | 是否作为 3D 音源播放 |
| `volume_` | `float` | `1.0` | 组件音量，会与资产音量相乘 |
| `pitch_` | `float` | `1.0` | 相对播放速度 |
| `pan_` | `float` | `0.0` | 2D 声像，-1 左，0 中，1 右 |
| `priority_` | `int32_t` | `0` | 大于 0 时保护 voice，降低被抢占概率 |
| `bus_type_` | `EAudioBusType` | `Sfx` | 输出到哪个 Bus |
| `min_distance_` | `float` | `1.0` | 3D 衰减最小距离 |
| `max_distance_` | `float` | `100.0` | 3D 衰减最大距离 |

### 状态查询

```cpp
EAudioPlaybackStatus get_playback_status() const;
bool is_playing() const;
bool is_paused() const;
```

状态：

| 状态 | 说明 |
|------|------|
| `Stopped` | 未播放 |
| `PlayPending` | 已入队等待播放 |
| `Playing` | 正在播放 |
| `Paused` | 已暂停 |
| `Stopping` | 已请求停止 |
| `FadingOut` | 正在淡出停止 |

## SoundAsset

`SoundAsset` 保存音频文件路径和播放元数据。

```cpp
SoundAsset* sound = new_object<SoundAsset>();
sound->set_source_path("data/audio/sfx/click.wav");
sound->set_stream(false);
sound->set_looping(false);
sound->set_3d(false);
sound->set_volume(1.0f);
```

属性：

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `source_path_` | `std::string` | `engine/data/audio/default.wav` | 音频文件路径 |
| `stream_` | `bool` | `false` | 是否流式播放 |
| `looping_` | `bool` | `false` | 是否循环 |
| `volume_` | `float` | `1.0` | 资产基础音量 |
| `is_3d_` | `bool` | `true` | 是否允许 3D 播放 |
| `min_distance_` | `float` | `1.0` | 3D 衰减最小距离 |
| `max_distance_` | `float` | `100.0` | 3D 衰减最大距离 |
| `attenuation_model_` | `EAudioAttenuationModel` | `InverseDistance` | 3D 衰减模型 |

修改会影响运行时版本。下次播放或预加载时，`AudioDevice` 会按版本重建运行时音频对象。

## AudioPlayDesc

`AudioPlayDesc` 用于 `play_one_shot()`。

```cpp
struct AudioPlayDesc
{
    EAudioBusType bus_type = EAudioBusType::Sfx;
    float volume = 1.0f;
    float pitch = 1.0f;
    float pan = 0.0f;
    int32_t priority = 0;
    bool spatialized = false;
    Vec3 position = k_zero_vector;
    float min_distance = 1.0f;
    float max_distance = 100.0f;
    EAudioAttenuationModel attenuation_model = EAudioAttenuationModel::InverseDistance;
};
```

## 枚举

### EAudioBusType

```cpp
Master,
Music,
Sfx,
Ui,
Voice
```

### EAudioAttenuationModel

```cpp
None,
InverseDistance,
LinearDistance,
ExponentialDistance
```

### EAudioBackendType

```cpp
Miniaudio,
Auto,
Null
```

## 支持格式

当前通过 SoLoud `Wav` / `WavStream` 加载文件。

| 格式 | 扩展名 | 建议用途 |
|------|--------|----------|
| WAV | `.wav` | 低延迟短音效、UI、需要精确循环的资源 |
| Ogg Vorbis | `.ogg` | BGM、环境声、发布资源 |
| MP3 | `.mp3` | 长音频、临时资源、体积敏感资源 |
| FLAC | `.flac` | 无损音频、高质量源文件 |

未接入：`.aac`、`.m4a`、`.opus`、`.wma`、`.mid`、tracker/module 类格式。

## 配置

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

## 控制台命令

```text
audio.stop_all
audio.dump_stats
audio.master_volume
audio.master_volume 0.8
```