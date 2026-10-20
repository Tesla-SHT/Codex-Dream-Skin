# Codex Dream Skin — Linux 修复记录（PR #410 基础上的本地修复）

本仓库是 [Fei-Away/Codex-Dream-Skin](https://github.com/Fei-Away/Codex-Dream-Skin) PR #410 源码
（提交 `22d54c7d1f3c0214472d82caa4052036ded2b2ba`）加上一个**本地修复提交**的完整副本。

- 修复分支：`fix/linux-261013-thread-black-screen`
- 修复提交：`d1c375d` — `fix(renderer): stop hiding the conversation container and prefer visible cached workspaces`
- 已推送到 fork：<https://github.com/Tesla-SHT/Codex-Dream-Skin/tree/fix/linux-261013-thread-black-screen>

## 它解决了什么问题

在 ChatGPT 桌面版 **26.1013.40934**（Ubuntu 22.04.5 / X11）上，PR #410 原版皮肤会出现：
**首页正常，点开任意对话后主区域整块黑屏（正文、表格、输入框全部消失）。**

根因有两层：

1. **对话容器被误判为“顶部渐变”并隐藏。**
   新版把 `data-app-shell-main-content-top-fade="visible|hidden"` 放在了包含整个对话
   正文和输入框的容器上，用属性值控制原生渐变的显隐；旧皮肤把任何带这个属性的节点
   都当作装饰渐变，套了 `display: none !important`，于是整块对话一起消失。

2. **验证器读取了缓存隐藏页。**
   新版会在 DOM 里保留导航过的旧工作区（隐藏状态）。Linux 验证器和路由检测只取
   `document.querySelector` 的第一个匹配，经常拿到隐藏的首页，得出“主区域零尺寸 /
   当前在首页”的错误结论。

## 修复内容（`d1c375d`）

| 文件 | 改动 |
|---|---|
| `tools/selectors.json` | `main-content-top-fade` 属性回退分支限定为 `[data-app-shell-main-content-top-fade]:empty`；保留 `.app-shell-main-content-top-fade` 类和 `_MainContentTopFade_` 模块两个真装饰分支 |
| `runtime/renderer-inject.js` | 路由检测 `selectorHit` 排除处于 `display:none` / `contentVisibility:hidden` 祖先中的节点（注意：不能直接用节点自身 `checkVisibility`，因为皮肤本身就会隐藏 home-icon，会形成反馈循环） |
| `linux/scripts/injector.mjs` | `verifySession` 对 shell/composer/sidebar/generic 部件改为 `visibleNode()`——在全部匹配中选第一个**可见**节点，而不是文档序第一个 |
| `tools/renderer-runtime.test.mjs` | 增加 CSS 契约断言（禁止裸的 `[data-app-shell-main-content-top-fade]` 隐藏规则）+ 隐藏缓存首页必须判为 thread 的回归 |
| `linux/tests/cached-workspace.test.mjs` | 新增：可见 thread + 隐藏缓存首页 → 验证通过；全部隐藏 → 验证必须失败 |
| `linux|macos|windows/assets/*` | 由 `tools/sync-runtime-assets.mjs` 重新生成的三端同步资产 |

## 在本机复现 / 验证的方法

```bash
# 1. 跑完整 Linux 测试（需要 Node >= 22）
NODE="$(command -v node)" bash linux/tests/run-tests.sh
# 期望最后一行：linux tests passed

# 2. 部署到用户安装目录（官方 deb 的 ChatGPT，无需 sudo）
bash linux/scripts/install-dream-skin-linux.sh --no-launch

# 3. 启动并验证
dreamskin start
dreamskin status            # 普通 status 不探测 CDP，cdp=false 是正常的
bash ~/.local/share/codex-dream-skin/scripts/status-dream-skin-linux.sh --deep
bash ~/.local/share/codex-dream-skin/scripts/verify-dream-skin-linux.sh
```

验证环境：Ubuntu 22.04.5 / GNOME X11 / ChatGPT 26.1013.40934（官方 deb，来源与
`dpkg -V` 完整性校验均通过）。验证结果：`linux tests passed`；home 与 thread 路由
独立 verify 全部 `pass: true`；`--remove` 清理干净；重启后再次验证通过。

## 已知环境坑（与本仓库无关，但会干扰安装/验证）

- **安装器要求“已安装版本出现在官方 APT 元数据里”**。如果本机 apt 列表还没刷新到
  你装的这个版本，安装会以
  `The installed Codex version is not available from the official OpenAI repository.`
  退出 1。先 `sudo apt update`（或把 ChatGPT 升到源里已有的版本）即可。
- **不要用 sandbox/隔离环境跑测试或安装**：测试会因 Node 无法调用 `/bin/bash` 失败；
  安装脚本读不到 `/var/lib/dpkg` 时会误报“找不到官方应用”。
- **会话进程保活**：从一次性执行环境（CI、agent 沙盒）里启动 ChatGPT，进程会随会话
  结束被回收。用 `systemd-run --user ...` 或正常桌面启动即可。

## 上游反馈

修复已整理为可直接提交的分支。给上游提 issue/PR 时可直接引用：
`fix/linux-261013-thread-black-screen`（`d1c375d`），复现版本 26.1013.40934，
症状“home 正常、thread 黑屏”，根因与修复见上。
