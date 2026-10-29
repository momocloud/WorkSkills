# WorkSkills

可通过 `npx skills add` 安装的工作技能集合。

| Skill | 用途 |
| --- | --- |
| [remote-npu-environment](skills/remote-npu-environment/SKILL.md) | 多机器 SSH 排查、普通用户初始化、PyTorch/torch-npu/CANN 版本选择、环境安装及重装恢复 |

## 从本地安装

在希望使用 skill 的项目目录运行，将源路径替换为本机 `WorkSkills` 的实际路径：

```bash
npx skills add /path/to/WorkSkills --skill remote-npu-environment
```

命令会让你选择目标 agent。指定 Codex 并直接完成项目级安装：

```bash
npx skills add /path/to/WorkSkills --skill remote-npu-environment --agent codex -y
```

需要跨项目使用时增加 `--global`。只查看可发现的技能：

```bash
npx skills add /path/to/WorkSkills --list
```

本地路径、Git 仓库来源及选项见 [skills CLI 官方说明](https://github.com/vercel-labs/skills/blob/main/docs/commands.md)。本项目遵循 [标准技能目录结构](https://github.com/vercel-labs/skills/blob/main/docs/creating-skills.md)，无需发布 npm 包。

## 从 Git 仓库安装

将本目录作为仓库内容推送到 Git 服务后，可将本地路径替换为实际 Git URL：

```bash
npx skills add <实际仓库URL> --skill remote-npu-environment
```

这里仅提供可安装的目录；尚未创建或发布远端仓库。上述占位符不能直接运行。

## 使用

例如：

> 使用 remote-npu-environment，检查这台机器的 SSH 登录，并恢复我指定锁文件中的环境。

> 使用 remote-npu-environment，为这台新机器准备 NPU 开发环境；先和我确认 PyTorch、torch-npu 版本。

技能及参考文件全部包含在 `skills/remote-npu-environment/` 内，安装后无需原工作区的 `REMOTE_ENVIRONMENT.md`。维护时以技能目录内的文件为安装内容，原约定文档不会被自动同步。
