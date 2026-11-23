# dsh-hot-reload

DeepSeek Harness（DSH）插件热重载技能：不重启 `dsh-web` 让运行中的宿主用上插件新代码，并判断“这次能不能热应用、还是必须重启”。

- 技能正文：[skills/dsh-hot-reload/SKILL.md](skills/dsh-hot-reload/SKILL.md)
- 适用：DSH Web profile（已实测 0.2.2-rc.3 / 0.2.2-rc.5）；Plugin Manager 快速路径、外部换包时的 profile 重算触发、探针 + journal 验证、会话 revision 语义、失败与回滚边界。

## 安装

技能由 DSH 的 skills 目录发现。全局安装用符号链接而不是复制，本仓库是唯一来源：

```sh
ln -sfn ~/Git/github/dsh-hot-reload/skills/dsh-hot-reload ~/.dsh/skills/dsh-hot-reload
```

安装后新开会话即可在技能目录里看到 `dsh-hot-reload`。

## 维护

- 只改本仓库的 `skills/dsh-hot-reload/SKILL.md`；全局路径是指向它的符号链接，不需要同步。
- 更新后提交并推送。正在运行的会话是否需要重开，取决于它的技能目录何时刷新。

## 许可

未附带开源许可，作者保留所有权利。
