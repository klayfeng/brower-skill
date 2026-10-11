# browser-skill

通过 `bsk` CLI 驱动用户**已登录的 Chromium 浏览器**的 Agent Skill：访问/读取页面、填表单、抓数据、点击流程、UI 回归测试等。核心文件是 `SKILL.md`。

## 在新电脑上安装

把本仓库直接克隆为 skill 目录（skill 约定放在 `~/.agents/skills/` 下）：

```bash
git clone https://github.com/klayfeng/brower-skill.git ~/.agents/skills/browser-skill
```

Windows PowerShell：

```powershell
git clone https://github.com/klayfeng/brower-skill.git "$env:USERPROFILE\.agents\skills\browser-skill"
```

## 前置依赖

`SKILL.md` 只是"说明书"，实际运行还需要装好：

1. **bsk CLI**（BrowserSkill 命令行 + daemon）
2. **Chrome 扩展**（与 daemon 配对连接）

远程连接 / 配对的官方指南见 SKILL.md 中引用的文档：
<https://github.com/fish0710/BrowserSkill-internal/blob/fork-release/docs/remote-extension-connection.md>

## 文件说明

| 文件 | 用途 |
| --- | --- |
| `SKILL.md` | Skill 本体（给 Agent 看的操作指南） |
| `.bsk-source` | bsk 的来源标记（bundled + sha256），用于更新校验 |
| `.bsk.lock` | bsk 锁文件（空） |
