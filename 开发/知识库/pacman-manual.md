# Pacman 使用手册

Pacman 是 Arch Linux 的包管理器，用于安装、升级、卸载和查询软件包。

## 1. 快速上手

最常用的就是这几条：

```bash
# 1) 同步数据库并升级整个系统（推荐的日常更新方式）
sudo pacman -Syu

# 2) 搜索软件包（仓库）
pacman -Ss <关键词>

# 3) 安装软件包
sudo pacman -S <包名>

# 4) 查询已安装包信息
pacman -Qi <包名>

# 5) 卸载软件包（含不再需要的依赖和配置）
sudo pacman -Rns <包名>
```

## 2. 命令格式与全局选项

```bash
pacman <operation> [options] [targets]
```

- `operation`：操作类型，例如 `-S`、`-Q`、`-R`
- `targets`：包名、文件路径、URL 或搜索关键词

常见全局选项：

- `-h, --help`：查看帮助
- `-V, --version`：查看版本
- `--noconfirm`：跳过确认（脚本中谨慎使用）
- `--config <path>`：指定配置文件
- `--root <path>` / `--sysroot <path>`：操作其他根目录
- `--debug`：输出调试信息

## 3. 各操作说明

### 3.1 同步仓库（`-S, --sync`）

用途：从远程仓库安装/升级软件包。

常用命令：

```bash
# 安装包
sudo pacman -S <包名>

# 搜索包（仓库）
pacman -Ss <关键词>

# 查看仓库包详细信息
pacman -Si <包名>

# 刷新数据库并全量升级系统
sudo pacman -Syu

# 仅下载，不安装
sudo pacman -Sw <包名>
```

高频选项：

- `-y, --refresh`：刷新数据库（`-yy` 强制刷新）
- `-u, --sysupgrade`：升级已安装包
- `-s, --search`：搜索仓库
- `-i, --info`：查看包信息
- `-w, --downloadonly`：仅下载
- `--needed`：跳过已是最新版本的包
- `--ignore <pkg>`：忽略某个包升级

注意：

- 避免只执行 `pacman -Sy` 后再单独装包，这可能导致部分升级。
- 日常建议直接使用 `sudo pacman -Syu`。

### 3.2 查询本地包（`-Q, --query`）

用途：查询已安装软件包数据库。

常用命令：

```bash
# 列出所有已安装包
pacman -Q

# 查看已安装包详细信息
pacman -Qi <包名>

# 列出包内文件
pacman -Ql <包名>

# 查询某个文件属于哪个包
pacman -Qo /path/to/file

# 列出显式安装的包
pacman -Qe

# 列出孤立依赖包
pacman -Qdt
```

高频选项：

- `-i, --info`：包信息
- `-l, --list`：包内文件列表
- `-o, --owns`：文件归属查询
- `-e, --explicit`：显式安装包
- `-d, --deps`：依赖安装包
- `-u, --upgrades`：可升级包

### 3.3 卸载软件包（`-R, --remove`）

用途：卸载本地已安装包。

常用命令：

```bash
# 仅卸载包
sudo pacman -R <包名>

# 卸载包及不再需要的依赖
sudo pacman -Rs <包名>

# 卸载包、依赖和配置文件
sudo pacman -Rns <包名>
```

高频选项：

- `-s, --recursive`：同时处理不再需要的依赖
- `-n, --nosave`：删除配置备份
- `-c, --cascade`：级联删除依赖该包的软件（高风险）
- `-d, --nodeps`：跳过依赖检查（谨慎）

### 3.4 本地文件安装（`-U, --upgrade`）

用途：从本地文件或 URL 安装包。

```bash
# 安装本地包文件
sudo pacman -U ./pkgname.pkg.tar.zst

# 从 URL 安装
sudo pacman -U https://example.com/pkgname.pkg.tar.zst
```

常用选项：

- `--asdeps`：标记为依赖安装
- `--asexplicit`：标记为显式安装
- `--needed`：已是最新则跳过

### 3.5 文件数据库查询（`-F, --files`）

用途：查“哪个仓库包提供某个文件”。

```bash
# 刷新文件数据库
sudo pacman -Fy

# 查询文件由哪个包提供
pacman -F /usr/bin/<命令名>

# 搜索文件数据库
pacman -Fs <关键词>
```

### 3.6 数据库属性操作（`-D, --database`）

用途：修改已安装包的安装原因等元信息。

```bash
# 标记为依赖安装
sudo pacman -D --asdeps <包名>

# 标记为显式安装
sudo pacman -D --asexplicit <包名>

# 检查数据库一致性
sudo pacman -D --check
```

### 3.7 依赖测试（`-T, --deptest`）

用途：检查给定依赖是否满足。

```bash
pacman -T <依赖1> <依赖2>
```

输出为空表示依赖都已满足。

## 4. 常用场景速查

```bash
# 系统更新
sudo pacman -Syu

# 查找并安装软件
pacman -Ss <关键词>
sudo pacman -S <包名>

# 查看某文件来自哪个已安装包
pacman -Qo /path/to/file

# 查看某命令由哪个仓库包提供（先刷新 -Fy）
pacman -F /usr/bin/<命令名>
```

清理相关：

```bash
# 清理未安装包缓存（保留已安装包对应缓存）
sudo pacman -Sc

# 强力清理所有缓存（下次可能需重新下载）
sudo pacman -Scc

# 删除孤立依赖（安全写法：无孤包时不执行删除）
orphans=$(pacman -Qdtq)
[ -n "$orphans" ] && sudo pacman -Rns $orphans
```

## 5. 配置与日志

- 主配置文件：`/etc/pacman.conf`
- 镜像列表：`/etc/pacman.d/mirrorlist`
- 操作日志：`/var/log/pacman.log`

常见配置项（位于 `[options]`）：

- `HoldPkg`：保护包
- `IgnorePkg` / `IgnoreGroup`：忽略升级
- `Architecture`：架构设置
- `Color` / `VerbosePkgLists` / `ParallelDownloads`：输出和下载行为

## 6. 故障排查

### 6.1 `failed to commit transaction`

可能原因：磁盘空间不足、权限问题、文件冲突。

可检查：

```bash
df -h
sudo pacman -Syu
```

### 6.2 `invalid or corrupted package`

可能是缓存损坏。

```bash
sudo pacman -Sc
sudo pacman -Syyu
```

### 6.3 签名或密钥问题

```bash
sudo pacman-key --init
sudo pacman-key --populate archlinux
sudo pacman-key --refresh-keys
sudo pacman -S archlinux-keyring
sudo pacman -Syu
```

### 6.4 包冲突（`conflicting files` / `conflicts with`）

建议先定位冲突来源，再决定卸载、替换或手动处理文件；不要直接大范围强制覆盖。

## 7. 参考文档

- Arch Wiki: https://wiki.archlinux.org/title/Pacman
- Pacman man page: https://man.archlinux.org/man/pacman.8
- Libalpm man page: https://man.archlinux.org/man/libalpm.3

---

本手册基于 Pacman 7.1.0 整理，最后更新：2026-03-17。
