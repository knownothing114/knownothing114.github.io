---
description: >-
  我开发的第一个 dsh（DeepSeek Harness）插件 dsh-notify：当 dsh 需要你注意时——权限确认、提问、计划审批、回合完成、目标完成/受阻、出错、工作流结束——在桌面弹出原生通知。本文介绍它的功能、安装方式与设计理念。
categories: [开发工具, 插件]
tags: [dsh, DeepSeek, 插件, AI, 效率工具]
---

> 开源地址：<a href="https://github.com/knownothing114/dsh-notify" target="_blank">github.com/knownothing114/dsh-notify</a>（MIT 协议，欢迎 Star ⭐）
{: .prompt-tip }

## 前言

最近深度使用 <a href="https://github.com/deepseek-ai/deepseek-harness" target="_blank">dsh（DeepSeek Harness）</a> 做各种任务时，发现一个很烦人的问题：**dsh 需要你介入的时候，完全没有任何提示**。

比如：

- 权限确认请求弹起了，我却盯着别的窗口发呆；
- agent 用 `ask_user_question` 问我问题，半天没人答；
- 它提交了一份计划等我审批，我完全没看到；
- 回合跑完了、目标完成了、或者出错了，我在摸鱼没注意到；
- 后台跑着个工作流，结束了也不知道。

终端或浏览器标签页一旦切到后台，这些时刻全都容易错过。于是我就动手写了我的第一个 dsh 插件——**dsh-notify**：一个纯观察者的桌面通知提醒插件。

## 它能做什么

dsh-notify 监听 dsh 主机事件总线，在**需要你注意的每一刻**弹出一条原生桌面通知：

| 触发器 | 触发时机 | 默认 |
|---|---|---|
| `approval` | 权限确认请求（沙箱升级、需要批准的操作等） | 开 |
| `question` | `ask_user_question` 提问 / 计划待审批（`exit_plan_mode`） | 开 |
| `turnComplete` | 回合完成（agent 回复完毕，等你下一步输入） | 开 |
| `goal` | 目标完成 / 目标受阻 | 开 |
| `error` | 会话出错 | 开 |
| `workflow` | 工作流运行结束 | 开 |

通知渠道（`channel`）支持：

- `auto`（默认）：macOS → `osascript` 系统通知；Windows → PowerShell 弹窗；其他平台 → `notify-send`
- `osascript` / `notify-send` / `powershell`：强制指定渠道
- `bell`：终端响铃（`\x07`），零依赖兜底
- `custom`：执行自定义命令模板（占位符 `{title}` `{body}` `{app}`），比如接 `terminal-notifier`
- `none`：关闭

## 设计原则：纯观察者

这是我最得意的一点：**插件是纯观察者，所有监听器都是被动的**。尤其是 `approval/request` 监听器——它只负责提醒你，**永远不会替你做决定**，总是调用 `next()` 转发请求，审批链路的行为与未安装本插件时完全一致。装了这个插件，dsh 的权限安全性不会有任何改变。

## 安装

一行命令搞定（以 `web` profile 为例，其他 profile 同理）：

```sh
# 本地目录 / git URL / 未来 npm 包名均可
dsh plugin --profile web add git+https://github.com/knownothing114/dsh-notify.git
```

该命令会安装依赖**并自动注册 bundle**。如果 dsh 版本较老没有自动追加，手动在 `~/.dsh/profiles/web/package.json` 的 `dsh.profile.bundles` 里加上 `"dsh-notify"`，然后重启 dsh web 即可。

## 配置：开箱即用 + Web 设置页

内置的默认值已经足够好用，基本不用配置。如果想要微调，有两种方式：

1. **Web 界面（推荐）**：插件注册了一个专门的 **设置 → 通知** 标签页，所有选项（总开关、渠道、应用名、声音、最小间隔、仅根会话、六个触发器开关等）都可以在表单里改，改动立即生效，**无需重启**。
2. **配置文件**：在 profile 的 `cordis.patch.yml` 里按行 id `dsh-notify` 覆盖：

```yaml
- id: dsh-notify
  config:
    channel: auto        # auto | osascript | notify-send | powershell | bell | custom | none
    sound: true          # 播放提示音（macOS）
    minIntervalMs: 3000  # 两条通知之间的最小间隔（防轰炸）
    rootsOnly: true      # 只提醒根会话，忽略子代理
    customCommand: ""    # channel 为 custom 时的命令模板
```

## 一点技术细节

- 零运行时依赖（仅两个 `@deepseek-ai` 包），可在任意 profile（`web` / `headless` / 自定义）使用；
- 所有监听器注册在根上下文，按 dsh 作用域路由规则接收所有 agent / session 事件，`rootsOnly` 通过 `ctx.agents.roots()` 过滤子代理噪音；
- 通知用分离的 `spawn` 子进程派发，**绝不会阻塞 agent 循环**，失败也只记日志；
- 为了让第三方的 `notify` 设置命名空间能被 Web UI 读写，插件包了一层 apiProxy 的设置处理器——这也是我踩坑最多、收获最大的地方（写了一篇<a href="https://github.com/knownothing114/dsh-notify/blob/main/docs/third-party-settings-namespace-exposure.md" target="_blank">开发者笔记</a>记录这个机制）；
- 30 个测试用例（单元 + apply 冒烟 + 客户端表单渲染/交互 + 集成），GitHub Actions CI 全绿。

## 使用感受与推荐

装上的当天我就觉得「回不去了」：现在把 dsh 丢在后台跑长任务，该摸鱼摸鱼，该干别的干别的，**需要我的时候它自然会叫我**。尤其是权限确认和提问——以前经常是切回来看见 agent 早就卡在等我回答，现在第一时间就能处理，整个协作节奏顺畅了很多。

如果你也在用 dsh，强烈推荐装一个试试，安装只要一分钟，绝对物超所值：

- 仓库：<a href="https://github.com/knownothing114/dsh-notify" target="_blank">https://github.com/knownothing114/dsh-notify</a>
- 文档：英文 / <a href="https://github.com/knownothing114/dsh-notify/blob/main/docs/README.zh-CN.md" target="_blank">简体中文</a> / 日本語 三语齐全
- 有问题欢迎提 <a href="https://github.com/knownothing114/dsh-notify/issues" target="_blank">Issue</a>，有想法也欢迎 PR～
