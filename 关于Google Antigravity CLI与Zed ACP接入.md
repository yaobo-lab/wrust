# Google Antigravity CLI 安装与 Zed ACP 接入

本文整理 Windows 下 Google Antigravity CLI（`agy`）的安装、登录、更新，以及通过官方 ACP Registry 接入 Zed 的方法。同时记录当前开发环境的实际安装状态和故障排查方式。

> 本文只使用 Google LLC 发布到 ACP Registry 的官方 Antigravity ACP Server，不使用第三方 OAuth 或 ACP 包装器。

## 1. 当前环境和验证状态

当前开发环境：

| 项目 | 当前状态 |
| --- | --- |
| 操作系统 | Windows 10 |
| 项目目录 | Zed 通过 WSL 打开 `/data/code/source` |
| Windows 用户 | `C:\Users\admin` |
| Antigravity CLI | 已安装 |
| CLI 版本 | `1.2.11` |
| CLI 路径 | `C:\Users\admin\AppData\Local\agy\bin\agy.exe` |
| Zed ACP 配置 | 已配置为 Registry Agent |
| 官方 ACP Server | 已下载 |
| ACP Server 版本 | `1.4.5` |
| ACP 登录 | 尚未完成 |

当前 Zed 配置文件位于：

```text
C:\Users\admin\AppData\Roaming\Zed\settings.json
```

其中已经包含：

```json
"agent_servers": {
    "antigravity-acp": {
        "type": "registry"
    }
}
```

因此当前机器不需要再次修改 `settings.json`，也不要把它改成第三方 Custom Agent。

## 2. 安装 Antigravity CLI

### 2.1 使用官方 PowerShell 安装脚本

打开 Windows PowerShell，执行：

```powershell
irm https://antigravity.google/cli/install.ps1 | iex
```

默认安装目录：

```text
C:\Users\<用户名>\AppData\Local\agy\bin
```

官方安装脚本会完成以下操作：

- 从 Google 服务下载 `agy.exe`；
- 对下载内容执行完整性校验；
- 执行 `agy install` 配置命令和环境变量；
- 配置 CLI 自动更新能力。

### 2.2 验证安装

重新打开 PowerShell，然后执行：

```powershell
agy --version
```

如果当前终端尚未刷新 PATH，可以使用完整路径：

```powershell
& "$env:LOCALAPPDATA\agy\bin\agy.exe" --version
```

当前机器的实际输出为：

```text
1.2.11
```

### 2.3 修复 PATH

如果已经安装，但 PowerShell 提示找不到 `agy`，执行：

```powershell
& "$env:LOCALAPPDATA\agy\bin\agy.exe" install
```

执行后关闭并重新打开终端。

### 2.4 更新 CLI

```powershell
agy update
```

## 3. 首次启动和 Google 登录

建议先在 Windows PowerShell 中完成 CLI 初始化和账号登录，不要从 WSL 终端首次启动 Windows 版 `agy.exe`。

进入一个可信代码目录，然后启动：

```powershell
cd C:\path\to\your\project
agy
```

首次运行可能要求配置：

1. 终端颜色主题；
2. Alt-Screen 或 Inline 显示模式；
3. 是否信任当前工作区；
4. Google 账号认证。

登录时会打开默认浏览器。根据实际账号类型选择个人 Google 账号或 Gemini Enterprise，并在浏览器中完成授权。

浏览器 OAuth 必须由用户本人确认，不能由自动化工具代替完成。

登录后，可以输入一个只读测试：

```text
List the files in the current workspace without modifying anything.
```

退出 CLI：

```text
/exit
```

也可以按 `Ctrl+D`。

如当前版本支持退出登录，可在交互界面执行：

```text
/logout
```

## 4. 通过 ACP Registry 接入 Zed

Antigravity CLI 和 Zed ACP Server 是两个独立的程序包。安装 `agy` 可以用于终端操作和完成统一账号认证，但不会自动把 ACP Agent 写入 Zed 配置。

Zed 应通过 ACP Registry 安装官方 Agent。

### 4.1 从 Agent Settings 安装

在 Zed 中：

1. 按 `Ctrl+Shift+P` 打开命令面板；
2. 执行 `agent: open settings`；
3. 进入 `External Agents`；
4. 点击 `Add Agent`；
5. 选择 `Install from Registry`；
6. 搜索并安装 `Google Antigravity`。

也可以在命令面板直接执行：

```text
zed: acp registry
```

### 4.2 当前机器的官方 ACP 包

当前 Zed 已经下载 Windows x64 官方 ACP Server：

```text
C:\Users\admin\AppData\Local\Zed\external_agents\registry\antigravity-acp\
v_1.4.5_6c2906ba8ee98ee2_9db1e24e09a5ce41\agy_acp_server.exe
```

当前 ACP Registry 清单信息：

| 属性 | 值 |
| --- | --- |
| Agent ID | `antigravity-acp` |
| 名称 | `Google Antigravity` |
| 作者 | `Google LLC` |
| 版本 | `1.4.5` |
| 启动文件 | `agy_acp_server.exe` |
| 下载域名 | `dl.google.com` |

Windows x64 官方压缩包：

```text
https://dl.google.com/agy-extensions/releases/windows/
agy-acp-server-1.4.5-windows-x86_64.zip
```

Registry 中登记的压缩包 SHA-256：

```text
d24f4c73c9f96d4f44384e281c855a56c72d8eaa9c5b0fad7ccb1635c6b63f50
```

该 SHA-256 对应下载的 ZIP 压缩包，不能直接与解压后的 `agy_acp_server.exe` 哈希比较。

## 5. 在 Zed 中完成登录

完成 Registry 安装后：

1. 打开 Zed Agent 面板；
2. 新建一个 External Agent Thread；
3. 在 Agent 选择器中选择 `Google Antigravity`；
4. 点击界面中的 `Sign In`、`Log in` 或相同含义的按钮；
5. 在默认浏览器中完成 Google 登录；
6. 返回 Zed，重新创建或继续 Antigravity Thread。

Google 文档说明 Antigravity CLI、App 和 IDE 集成采用统一认证体系。如果 Zed 首次登录界面启动失败，建议先在 Windows PowerShell 中执行：

```powershell
agy
```

完成 CLI 登录后，重启 Zed，再次打开 Google Antigravity Thread。

## 6. 验证 ACP 可用性

在 Google Antigravity Thread 中发送以下只读任务：

```text
只列出当前工作区根目录中的文件，不要创建、修改或删除任何文件。
```

成功标准：

- 不再出现 `Server exited with status exit code: 1`；
- Agent 能返回当前工作区信息；
- Zed 中能看到 Antigravity 提供的会话界面；
- 只读请求没有产生文件修改。

然后可以执行更完整的只读验证：

```text
分析当前项目使用的编程语言、构建系统和主要目录，不要修改文件。
```

## 7. 当前日志结论

当前 Zed 日志确认官方 Server 曾经成功启动：

```text
Starting AGY ACP Server...
```

随后出现：

```text
settings: path=C:\Users\admin\.gemini\antigravity-acp\settings.json status=missing
```

以及：

```text
Server exited with status exit code: 1
```

这表明：

1. Zed 的 ACP Registry 配置已经生效；
2. 官方 `agy_acp_server.exe` 可以被 Zed 启动；
3. `C:\Users\admin\.gemini\antigravity-acp\settings.json` 尚未生成；
4. 当前尚未完成首次认证，或者首次认证流程提前退出。

其中 `status=missing` 指的是 Antigravity 自己的认证配置，不是 Zed 的 `settings.json` 丢失。

## 8. 故障排查

### 8.1 Server 启动后立即退出

按照以下顺序处理：

1. 关闭当前失败的 Antigravity Thread；
2. 在 Windows PowerShell 中运行 `agy`；
3. 完成 Google 登录和首次初始化；
4. 完全退出并重新打开 Zed；
5. 新建 Google Antigravity External Agent Thread；
6. 再次点击登录或发送只读测试消息。

### 8.2 查看 ACP 通信日志

在 Zed 命令面板执行：

```text
dev: open acp logs
```

重点搜索以下内容：

```text
antigravity
agy_acp
authentication
settings
exited
```

Windows 上的 Zed 主日志通常位于：

```text
C:\Users\admin\AppData\Local\Zed\logs\Zed.log
```

### 8.3 重新安装官方 ACP Server

如果仍然启动失败：

1. 执行 `agent: open settings`；
2. 进入 `External Agents`；
3. 卸载 `Google Antigravity`；
4. 重新通过 `Install from Registry` 安装；
5. 不要手动指定 `agy_acp_server.exe` 路径。

### 8.4 Windows 和 WSL 混合环境

当前 Zed 是 Windows 进程，但项目位于 WSL：

```text
/data/code/source
```

ACP Server 实际运行于 Windows：

```text
C:\Users\admin\AppData\Local\Zed\external_agents\...
```

登录成功后，如果 Agent 可以连接但无法读取项目文件，可能是 Windows ACP 进程无法正确处理 WSL 工作区路径。

可以临时打开一个 Windows 本地目录进行对比测试，例如：

```text
C:\code\test-project
```

如果 Windows 本地项目正常而 WSL 项目失败，可以确认问题位于跨环境路径兼容，而不是账号或 ACP 配置。

Google 当前入门文档仍将 Zed 支持环境标注为 macOS/Linux，但 ACP Registry 已经提供 Google 官方 Windows x64 和 ARM64 包。当前机器也确实能够启动 Windows ACP Server，不过 Windows 与 WSL 的组合仍可能存在边缘兼容问题。

## 9. 安全和第三方包装器

不要使用来源不明的第三方 Antigravity OAuth 或 ACP 包装器，例如：

```text
antigravity-acp
refined-antigravity-acp
```

当前 ACP Registry 已提供作者为 `Google LLC`、下载自 `dl.google.com` 的官方 ACP Server。使用官方 Registry 版本可以减少以下风险：

- 第三方读取或转发 OAuth 凭据；
- 账号使用方式违反 Google 服务条款；
- 第三方实现与官方 ACP 协议版本不兼容；
- 更新来源和二进制完整性无法确认。

外部 Agent 的账号、计费、数据保留和隐私条款由对应服务提供方负责，不由 Zed 管理。

## 10. 重装和卸载 CLI

当前 `agy --help` 没有独立的 `uninstall` 子命令。

重新安装可以再次执行官方脚本：

```powershell
irm https://antigravity.google/cli/install.ps1 | iex
```

完全卸载时：

1. 退出所有 `agy` 进程；
2. 删除 `C:\Users\admin\AppData\Local\agy`；
3. 如果用户 PATH 中仍有 `C:\Users\admin\AppData\Local\agy\bin`，再手动删除该 PATH 条目。

删除 `C:\Users\admin\.gemini` 可能清除 Antigravity 或其他 Gemini 工具的认证与配置。普通重装不应删除该目录。

## 11. 官方资料

- Google Antigravity 入门：<https://antigravity.google/docs/getting-started?tab=cli>
- Google Antigravity IDE Extensions：<https://antigravity.google/docs/ide/extensions>
- Zed External Agents：<https://zed.dev/docs/ai/external-agents>
- ACP Registry：<https://github.com/agentclientprotocol/registry>
- ACP Registry 索引：<https://cdn.agentclientprotocol.com/registry/v1/latest/registry.json>

## 12. 当前结论

当前机器已经完成：

- Google Antigravity CLI 安装；
- CLI `1.2.11` 版本验证；
- Zed Registry Agent 配置；
- Google 官方 ACP Server `1.4.5` 下载；
- ACP Server 启动路径验证。

尚需用户本人完成浏览器 OAuth。登录后，在 Zed 中发送一条只读任务，即可完成最终可用性验证。
