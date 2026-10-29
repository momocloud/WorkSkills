# WorkSkills

可通过 `npx skills add` 安装的工作技能集合。

| Skill | 用途 |
| --- | --- |
| [ascend-distributed-context](skills/ascend-distributed-context/SKILL.md) | 新工作目录导入昇腾计算框架组分布式背景、torch-npu 仓库语境及通用工作习惯 |
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

## 从 GitCode 安装

本目录内容推送到 [momocloud/WorkSkills](https://gitcode.com/momocloud/WorkSkills) 后，可使用完整 Git URL 安装：

```bash
npx skills add https://gitcode.com/momocloud/WorkSkills.git --skill remote-npu-environment
```

全局安装到 Codex：

```bash
npx skills add https://gitcode.com/momocloud/WorkSkills.git --skill remote-npu-environment --agent codex --global -y
```

私有仓库需要本机 Git 已具备访问权限；不要把令牌写进安装命令或仓库文件。

## 使用

例如：

> 使用 ascend-distributed-context 初始化这个新工作目录，保存通用工作背景。

> 使用 remote-npu-environment，检查这台机器的 SSH 登录，并恢复我指定锁文件中的环境。

> 使用 remote-npu-environment，为这台新机器准备 NPU 开发环境；先和我确认 PyTorch、torch-npu 版本。

安装背景 skill 时，将上面安装命令的 `--skill remote-npu-environment` 替换为 `--skill ascend-distributed-context` 即可。

每个技能的内容都包含在对应的 `skills/<skill-name>/` 内，安装后无需原任务目录或原工作区的 `REMOTE_ENVIRONMENT.md`。维护时以技能目录内的文件为安装内容，原约定文档不会被自动同步。
