# ZetaEngine 配置系统设计目标

## 1. 多源配置合并

配置文件按 **source_id + 路径 + 优先级(priority)** 注册到 `ConfigSourceStore`。运行时树(`runtime_tree_`)按优先级从低到高合并所有 source 的 YAML 文档，高优先级覆盖低优先级。schema 默认值作为最底层兜底。

内置 source：
- `engine.default` (priority=10)：引擎默认配置 `config/engine.yaml`
- `editor.default` (priority=10)：编辑器默认配置 `config/editor.yaml`
- `project.application` (priority=20)：项目级配置 `project/config/application.yaml`
- `user.settings` (priority=30)：用户设置 `project/saved/config/user_settings.yaml`

## 2. Schema 驱动的注册制

所有可写的配置项必须预先注册 schema（路径、默认值、校验器、生效策略、写入目标 source）。未注册的路径 **拒绝写入**，防止随意扩散配置项。通过 `cfg<T>()` + `register_config()` 以声明式 DSL 注册：

```cpp
register_config("project.application", "audio"_pp, {
    cfg<float>("global_volume"_pp, 1.0f, "user.settings")
        .validator(ConfigValidators::range(0.0f, 1.0f))
        .apply_policy(EConfigApplyPolicy::Immediate),
});
```

## 3. 类型安全的强类型访问

通过 `ConfigTraits<T>` 特化为每个配置 struct 定义 path、default、encode/decode、schemas。上层使用简洁的泛型 API：

- `get_config<AudioConfig>()` / `set_config<AudioConfig>(val)`
- `update_config<AudioConfig>([](AudioConfig& c){ ... })`
- `subscribe_config<AudioConfig>(callback)`
- `explain_config<AudioConfig>()`

同时也支持弱类型的 `get<T>(path, fallback)` / `set(path, value)` 通用路径访问。

## 4. 变更订阅与通知

`on_config_changed` 委托支持两种粒度的订阅：
- **精确路径订阅** — 只监听某个具体配置项
- **前缀订阅** — 监听某个子树下的所有变更（如监听整个 `audio` section）

`ConfigChangeEvent` 携带完整上下文：变更路径、source、新旧值、变更来源、生效策略。

## 5. 可组合的校验框架

`ConfigValidator` 封装校验函数，支持 `&&` 和 `||` 运算符组合多个校验规则。内置 `ConfigValidators::range(min, max)` 等常用校验器。校验在写入前执行，失败则拒绝写入并记录警告。

## 6. 即时生效 vs 重启生效

`EConfigApplyPolicy` 区分两种生效策略：
- **Immediate** — 变更即时生效（如音量调节）
- **RestartRequired** — 需要重启引擎（大多数配置）

重启需求的路径记录在 `pending_restart_paths_` 中，外部可通过 `has_pending_restart()` / `pending_restart_paths()` 查询。

## 7. 运行时覆盖（不持久化）

`set_runtime_override()` 提供最高优先级的临时覆盖，不写入任何文件。适合编辑器的实时预览、命令行参数覆盖等场景。`clear_runtime_override()` 可清除恢复。

## 8. 配置可解释性

`explain(path)` 返回 `ConfigExplainResult`，逐项展示：
- 该配置项的 **schema 默认值**
- **文件来源值**（来自哪个 source）
- **最终有效值**（合并 runtime override 之后）

方便调试"这个值到底从哪来的"问题。

## 9. 变更来源追踪

`EConfigChangeSource` 枚举追踪每次变更的发起方：`System`、`ConfigFile`、`Editor`、`Console`、`CVar`、`RuntimeCode`。便于日志审计和 UI 展示。

## 10. 零拷贝属性路径

`PropertyPathView` 是 constexpr 的路径视图，不持有字符串内存，用位压缩存储最多 16 段（22 位 offset + 10 位 size），支持 `"users[0].name"` 语法。`_pp` 字面量提供编译期解析和校验。`PropertyPath` 是 owning 版本，适合事件/缓存场景。

## 11. PropertyTree 统一数据模型

轻量封装 `YAML::Node`，共享句柄语义（拷贝不深拷贝），提供安全的 `try_get<T>` / `get(path, fallback)` 不抛异常，`set` 自动创建中间节点。`ObservablePropertyTree` 在此基础上增加写入变更通知。

## 12. 脏标记与持久化

每个 source 跟踪 dirty 状态。`save(source_id)` 和 `save_dirty_sources()` 将内存中的变更写回 YAML 文件。配置修改 → 标记脏 → 显式保存的流程保证数据安全。

---

**一句话总结**：这是一个以 schema 注册制为核心、多源优先级合并为机制、强类型访问为上层接口、支持校验/订阅/覆盖/解释的引擎级配置系统。
