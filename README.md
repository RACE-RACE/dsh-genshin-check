# dsh-genshin-check · 原神检测

对话开始时先看一眼这台电脑有没有装《原神》，两条路都安排好了：

- **装了** → 插件在你第一轮对话里塞一句叮嘱，让模型**只输出一句**「原神牛逼」；
  等这句话真的吐出来、再等约 2.5 秒，才启动游戏本体。
- **没装** → 让模型**只输出一句**「有电脑不玩原神？」；
  等这句话吐出来、再等约 2.5 秒，才用系统默认浏览器打开 [原神官网](https://ys.mihoyo.com/)。

## 它怎么判断"装了"

两级检测，全部只读、不联网、不改系统：

1. **快检**（同步，0ms）：`%ProgramFiles%\Genshin Impact`、`%ProgramFiles(x86)%\Genshin Impact`、
   `%LOCALAPPDATA%\Programs\Genshin Impact`、`%LOCALAPPDATA%\Genshin Impact` 这几个根目录下
   找 `Genshin Impact Game\GenshinImpact.exe`（国服）/ `YuanShen.exe`（国际服）/ `launcher.exe`。
2. **深检**（PowerShell 单进程，~250ms）：再遍一遍同样的目录，然后走注册表三处 `Uninstall` 根
   （HKLM / WOW6432Node / HKCU），按 `Genshin`、`原神`、`YuanShen` 匹配 DisplayName，
   命中就顺手从 `InstallLocation` 里把 exe 捞出来；最后看 `SOFTWARE\miHoYo` 的子键名。

判定和定位是分开的：注册表命中但找不到 exe（比如只留了卸载信息）仍然算"装了"，
会照常念台词，只是不启动游戏。

结果进程内缓存 10 分钟；会话创建时就后台预热，所以第一轮回复不会被检测拖慢。
子代理（`session.header.origin === 'subagent'`）创建的会话会被跳过，
避免每个 subagent 都开一次浏览器、启一次游戏。

## 配置项

写在 `cordis.patch.yml` 的 `config` 里（插件没有声明 schema，改完需重载插件或重启 DSH）：

| 键 | 默认值 | 说明 |
| --- | --- | --- |
| `url` | `https://ys.mihoyo.com/` | 没装时要打开的网址，仅接受 http/https |
| `message` | `有电脑不玩原神？` | 没装时让模型开口说的那句话 |
| `gameMessage` | `原神牛逼` | 装了时让模型开口说的那句话 |
| `openBrowser` | `true` | 没装时是否真的打开浏览器 |
| `openGame` | `true` | 装了时是否真的启动游戏 |
| `openDelayMs` | `2500` | 模型开口之后等多久再动作（毫秒） |
| `checkTimeoutMs` | `4000` | PowerShell 深检超时（毫秒） |
| `cacheTtlMs` | `600000` | 检测结果缓存时长（毫秒） |
| `onUnknown` | `treat-as-missing` | 超时/出错时算不算"没装"：`treat-as-missing` 或 `skip` |
| `skipSubagents` | `true` | 是否跳过子代理会话 |

## 实现要点

- 零外部依赖，只用 Node 内置模块（`child_process` / `fs` / `crypto` / `path`）。
- 挂在 `agent/pre-step` 上：第一轮第一步（`turn === 1 && step === 1`）才动作，
  `await next()` 之后往 `decision.messages` 追加一条 `form: 'notice'` 的上下文消息，
  不改变决策本身。
- 动作时机挂在 `agent/assistant-stream` 上：只有收到该会话第一个 `text-delta`（`reasoning-delta`
  不算）之后，才起 `openDelayMs` 的定时器；模型迟迟不吐字时另有 15 秒兜底，不会断在半路。
- 启动游戏用 `cmd /c start "" <exe>`（`cwd` 设为 exe 所在目录），由 shell 派生，进程彻底脱离
  宿主进程树存活；打不开浏览器 / 启不动游戏都只静默失败，不影响对话。
- 每个会话只处理一次（进程内 Set 去重），最多记 200 条。