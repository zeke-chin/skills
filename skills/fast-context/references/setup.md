# Fast Context 安装与鉴权

仅在安装、配置、导入或排查凭据时读取。本工具使用 Windsurf/Devin API Key；npm 登录态与搜索鉴权无关。

## 安装 CLI

需要 Node.js 22 或更高版本。全局安装：

```bash
npm install -g @zeke-chin/fast-context-cli
fast-context --version
fast-context auth --help
```

安装后提供 `fast-context` 和 `fast-context-mcp` 两个命令。该包公开可读，安装不需要 `npm login`。不指定标签时安装 `latest`；明确需要测试渠道时使用 `npm install -g @zeke-chin/fast-context-cli@next`。更新到相应渠道的新版本也使用同一条安装命令。

临时运行而不全局安装：

```bash
npm exec --package=@zeke-chin/fast-context-cli -- fast-context search "自然语言问题" --project .
```

包内有两个可执行入口，临时运行时显式指定 `--package` 和 `fast-context`。如果全局安装报 `EEXIST`，先核对冲突命令的链接目标和 `npm ls -g --depth=0`；需要替换旧包时先卸载对应旧包，再安装，避免直接用 `--force` 覆盖未知入口。

## 先确认当前来源

```bash
fast-context auth status --json
```

输出包含 `configured`、`source`、`path`、`config_path`，不包含 Key。`source` 为 `environment`、`saved` 或 `auto-discovery`。状态查询不请求远端 API；找不到环境变量和已保存的 Key 时，会尝试读取系统凭据。`configured: true` 只表示本地找到了凭据，不能证明它仍然有效。退出码 `0` 表示已配置，`1` 表示未找到或读取失败；结合输出区分原因。

搜索与状态查询按以下优先级选择凭据：

```text
WINDSURF_API_KEY → fast-context 保存的 Key → 系统 Windsurf/Devin 登录凭据
```

自动发现已能满足搜索时，无需额外导入。已保存文件损坏会报错，不会静默切换到另一个来源。

## 根据任务选择命令

| 需求 | 命令 | 行为 |
|---|---|---|
| 将本机已登录的 Windsurf/Devin 凭据保存给 fast-context | `fast-context auth import` | 直接读取系统凭据，成功后替换本工具保存的 Key |
| 手动配置用户已有的 API Key | `fast-context auth set` | 在终端交互输入并保存；当前输入会回显 |
| 从已有 Key 文件或管道配置 | `fast-context auth set --stdin < /path/to/api-key.txt` | 从标准输入读取并保存 |
| 查看实际生效来源 | `fast-context auth status` 或 `auth status --json` | 显示来源和路径，不打印 Key |
| 删除本工具保存的 Key | `fast-context auth clear` | 不清除环境变量，也不退出 Windsurf/Devin 登录 |

`auth set` 和 `auth import` 都不验证远端有效性。不要将实际 Key 放进命令参数、聊天回复或诊断日志；手动输入应在用户终端完成。

## 从系统导入

```bash
fast-context auth import
fast-context auth status --json
```

`auth import` 不走环境变量和已保存 Key 的优先级，而是直接尝试系统来源：

1. Linux/WSL 先读取 `~/.local/share/devin/credentials.toml`。
2. 再按 `Devin`、`Deviv`、`Windsurf` 的顺序尝试桌面数据库 `User/globalStorage/state.vscdb`；macOS/Windows 直接从此步骤开始。
3. 使用第一个成功提取的 Key，保存到 `auth status` 所示的 `config_path`。

桌面数据库位于 Linux 的 `${XDG_CONFIG_HOME:-~/.config}/<应用名>/`、macOS 的 `~/Library/Application Support/<应用名>/` 或 Windows 的 `%APPDATA%/<应用名>/` 下。导入只读取源文件；数据库提取的是 `ItemTable` 中 `windsurfAuthStatus` 的 JSON `apiKey` 字段。

成功时只显示来源和保存位置。发现失败或 Key 格式无效时，原有保存文件保持不变。即使导入成功，已设置的 `WINDSURF_API_KEY` 仍优先生效；用 `auth status` 确认实际来源。

如果当前安装版本没有 `auth import`，先看 `fast-context auth --help`。在包含该功能的源码仓库根目录，可用 `node src/cli.mjs auth import`；不要假定全局安装版本与当前源码一致。`auth import` 不接受 `--json`、`--stdin` 或数据库路径参数。

## 按错误处理

- **未找到系统凭据**：查看导入错误中的尝试路径，确认对应应用在同一系统用户下已登录。WSL 使用 Linux 的发现路径；需要 Devin CLI 登录时，在 WSL 内运行 `devin login` 后再导入。用户已有 Key 时也可用 `auth set`。
- **导入后仍为旧身份或仍报 401/403**：先检查 `auth status` 的 `source`。环境变量优先于导入结果；应处理当前生效来源，而不是反复导入或清空保存文件。若凭据已失效，重新登录源应用并导入，或配置新的 Key；403 也可能涉及权限，不能仅凭状态码认定 Key 失效。
- **已保存文件损坏**：根据报错路径，用 `auth set` 或 `auth import` 替换；只有需要删除保存凭据时才用 `auth clear`。
- **验证修复**：凭据来源确实变化后，重试原来的搜索；不要额外发起无关搜索。来源和登录状态没有变化时，重复同一导入通常无助于解决问题。
