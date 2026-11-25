# dsh-genshin-check · 原神检测

对话开始时先看一眼这台电脑有没有装《原神》：

- **装了** → 什么都不发生，模型正常输出。
- **没装** → 插件在你的第一轮对话里塞一句叮嘱，让模型**只输出一句**「有电脑不玩原神？」；
  等这句话真的吐出来、再等约 2.5 秒，才用系统默认浏览器打开 [原神官网](https://ys.mihoyo.com/)。

## 它怎么判断"装了"

两级检测，全部只读、不联网、不改系统：

1. **快检**（同步，0ms）：`%ProgramFiles%\Genshin Impact`、`%ProgramFiles(x86)%\Genshin Impact`、
   `%LOCALAPPDATA%\Programs\Genshin Impact`、`%ProgramFiles%\miHoYo Launcher` 等常见目录。
2. **深检**（PowerShell 单进程，~250ms）：遍历注册表三处 `Uninstall` 根
   （HKLM / WOW6432Node / HKCU），按 `Genshin`、`原神`、`YuanShen` 搜索 DisplayName /
   InstallLocation / UninstallString，再看 `SOFTWARE\miHoYo` 的子键名。

结果进程内缓存 10 分钟；会话创建时就后台预热，所以第一轮回复不会被检测拖慢。
子代理（`session.header.origin === 'subagent'`）创建的会话会被跳过，避免每个 subagent 都开一个浏览器标签。

## 配置项

写在 `cordis.patch.yml` 的 `config` 里（插件没有声明 schema，改完需重载插件或重启 DSH）：

| 键 | 默认值 | 说明 |
| --- | --- | --- |
| `url` | `https://ys.mihoyo.com/` | 要打开的网址，仅接受 http/https |
| `message` | `有电脑不玩原神？` | 让模型开口说的那句话 |
| `openBrowser` | `true` | 是否真的打开浏览器 |
| `openDelayMs` | `2500` | 模型开口之后等多久再跳转（毫秒） |
| `checkTimeoutMs` | `4000` | PowerShell 深检超时（毫秒） |
| `cacheTtlMs` | `600000` | 检测结果缓存时长（毫秒） |
| `onUnknown` | `treat-as-missing` | 超时/出错时算不算"没装"：`treat-as-missing` 或 `skip` |
| `skipSubagents` | `true` | 是否跳过子代理会话 |

## 实现要点

- 零外部依赖，只用 Node 内置模块（`child_process` / `fs` / `crypto` / `path`）。
- 挂在 `agent/pre-step` 上：第一轮第一步（`turn === 1 && step === 1`）才动作，
  `await next()` 之后往 `decision.messages` 追加一条 `form: 'notice'` 的上下文消息，
  不改变决策本身。
- 跳转时机挂在 `agent/assistant-stream` 上：只有收到该会话第一个 `text-delta`（`reasoning-delta`
  不算）之后，才起 `openDelayMs` 的定时器；模型迟迟不吐字时另有 15 秒兜底，不会断在半路。
- 每个会话只处理一次（进程内 Set 去重），最多记 200 条。