# WorkSkills

可通过 `npx skills add` 安装的工作技能集合。

| Skill | 用途 |
| --- | --- |
| [ascend-distributed-context](skills/ascend-distributed-context/SKILL.md) | 新工作目录导入昇腾计算框架组分布式背景、torch-npu 仓库语境、RFC/PR 模板、worktree 分支规范及通用工作习惯 |
| [remote-user-bootstrap](skills/remote-user-bootstrap/SKILL.md) | 通过问答生成在已有 root 的服务器上创建普通用户（支持自定义家目录）、配置 docker/sudo/设备权限及 Windows/WSL 免密登录的命令 |
| [remote-npu-environment](skills/remote-npu-environment/SKILL.md) | 多机器 SSH 排查、普通用户初始化、PyTorch/torch-npu/CANN 版本选择、环境安装及重装恢复 |
| [ascend-distributed-validation](skills/ascend-distributed-validation/SKILL.md) | 依具体用例执行分布式验证、整理可复现报告，以及采集分析 HCCL 状态与日志 |

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

> 使用 remote-user-bootstrap，先问我用户名、家目录位置、本机系统和权限需求，然后给我一整套在 root 服务器建普通用户并配好 Windows/WSL 免密登录的命令（不要直接操作）。

> 使用 remote-npu-environment，检查这台机器的 SSH 登录，并恢复我指定锁文件中的环境。

> 使用 remote-npu-environment，为这台新机器准备 NPU 开发环境；先和我确认 PyTorch、torch-npu 版本。

> 使用 ascend-distributed-validation，先根据这些用例的说明和实现确定各自的执行方式，再按指定范围验证并生成报告。

> 使用 ascend-distributed-validation，分析现有的 HCCL 日志并整理定位材料，不重跑用例。

各技能按当前任务独立选用；`remote-user-bootstrap` 只通过问答产出可复制执行的初始化命令，不直接操作本机或远端机器；远程环境准备不默认触发测试，验证技能不预设启动器、卡数或固定测试矩阵。

安装其他 skill 时，将上面安装命令的 `--skill remote-npu-environment` 替换为 `--skill remote-user-bootstrap`、`--skill ascend-distributed-context` 或 `--skill ascend-distributed-validation`。

每个技能的内容都包含在对应的 `skills/<skill-name>/` 内，安装后无需原任务目录或原工作区的 `REMOTE_ENVIRONMENT.md`。维护时以技能目录内的文件为安装内容，原约定文档不会被自动同步。
