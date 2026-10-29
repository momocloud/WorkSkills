# 环境检查与复现

在 SKILL.md 要求的目标版本已明确后使用本流程。

## 登录后的检查与恢复决策

核对普通用户身份及实际 Home，再切换到工作根目录。检查系统与架构、Python、uv、任务目录、启动脚本、虚拟环境、CANN 实际入口及驱动状态。可使用 `id`、`getent passwd`、`uname -m`、`cat /etc/os-release`、`command -v uv`、`python3 --version`、`npu-smi info` 等只读检查。

| 检查结果 | 处理方式 |
| --- | --- |
| 配置未加载 | 文件存在但命令不可用时，检查 PATH 并显式加载用户配置；非交互 SSH 不保证执行 `.bashrc`，不能直接认定环境丢失 |
| 环境存在且匹配选择 | 加载任务启动脚本，核对解释器和实际包版本，完成必要验证后复用，不重复安装 |
| 环境存在但版本不同 | 告知实际版本，按已选目标创建独立环境，不覆盖原环境或修改其他任务共用的默认配置 |
| 文件缺失或虚拟环境失效 | 排查挂载、Home 变化及重装导致的解释器链接失效；从清单重建新环境，不直接复制旧 `.venv` 跨系统恢复 |
| CANN 缺失或不完整 | 区分不存在与不可读，检查入口和运行库；可在用户目录安装配套版本，驱动问题交由管理员处理 |
| 无恢复记录 | 先询问目标版本，收集当前硬件与系统约束再制定安装步骤，不能猜测丢失环境的版本 |

同一 IP 上的机器可能已经重装，账号、公钥、目录、驱动和设备状态均须重新核实。旧记录中的“安装成功”“测试通过”或“系统 CANN 不完整”只描述当时的机器。

## 按已确认版本新建或重建环境

以下步骤以普通用户执行，每步成功后再继续。所有路径以已核实的工作根目录为准，不重定义 Shell 的 `HOME`。

### 准备目录与用户配置

在工作根目录内准备 `.local/bin`、`.config`、`.cache`、`.local/share`、`.local/state`、`tmp`，在任务目录内准备 `requirements/`、`packages/`、`logs/`、`runtime/`。核对真实路径和归属，避免经符号链接写出边界。

在用户环境配置脚本中设置以下变量，再启动安装器和 Python 工具。`WORK_ROOT` 表示已核实的绝对工作根目录：

| 用途 | 路径或值 |
| --- | --- |
| `TMPDIR`、`TMP`、`TEMP` | `$WORK_ROOT/tmp` |
| `XDG_CACHE_HOME`、`XDG_CONFIG_HOME` | `$WORK_ROOT/.cache`、`$WORK_ROOT/.config` |
| `XDG_DATA_HOME`、`XDG_STATE_HOME` | `$WORK_ROOT/.local/share`、`$WORK_ROOT/.local/state` |
| `UV_CACHE_DIR`、`PIP_CACHE_DIR` | `$WORK_ROOT/.cache/uv`、`$WORK_ROOT/.cache/pip` |
| `UV_PYTHON_INSTALL_DIR`、`UV_TOOL_DIR` | `$WORK_ROOT/.local/share/uv/python`、`$WORK_ROOT/.local/share/uv/tools` |
| `UV_INSTALL_DIR`、`UV_TOOL_BIN_DIR` | `$WORK_ROOT/.local/bin` |
| `PYTHONUSERBASE` | `$WORK_ROOT/.local` |
| `PIP_REQUIRE_VIRTUALENV` | `true` |

将用户可执行目录加入 PATH。已有配置先检查并保留；修改 Shell 启动文件前备份，避免重复追加。非交互脚本显式加载配置。uv 参数按所选版本核对，参考 [uv 环境变量文档](https://docs.astral.sh/uv/reference/environment/)。

### 准备 uv、Python 与依赖材料

1. uv 已可用时核对版本；缺失时从官方来源或可信归档安装固定版本，位置限定在工作根目录内。保存安装器或二进制来源及校验值，不直接运行未经检查的网络脚本。
2. 选择满足目标框架要求的 Python。可复用合适的系统解释器，但不向系统 Python 安装包；若不合适，使用工作根目录内的独立 Python，并记录版本及来源。
3. **恢复已有环境**：取回用户指定记录中的锁文件、安装包清单及启动脚本，核对架构、Python ABI、框架构建及系统条件；兼容时按原锁文件安装，不重新解析为更新版本。
4. **新版本或不同平台**：按用户选择编写依赖声明，固定 PyTorch 和 torch-npu 的精确版本或 wheel 地址。核对官方 wheel、构建后缀与包索引，不默认所有 NPU 组合使用同一种 CPU wheel 或同名版本号。生成带哈希的锁文件并检查解析结果。
5. 下载不可达时，可用经核实的镜像或离线安装包，保持版本和制品校验要求；不能把下载失败处理为静默升级或降级。缺少目标版本制品时说明缺失项，请用户决定。

### 创建隔离环境并安装

下面是已准备好 uv、Python 和锁文件后的 Bash 模板。四个路径变量须事先设为实际绝对路径；虚拟环境使用尚不存在的新目录，锁文件须与用户选择匹配：

```bash
(
  set -eu
  : "${TASK_DIR:?设置任务目录}" "${PYTHON_BIN:?设置 Python 解释器路径}"
  : "${VENV_DIR:?设置新虚拟环境路径}" "${LOCK_FILE:?设置锁文件路径}"
  cd "$TASK_DIR"
  test ! -e "$VENV_DIR"
  uv venv --python "$PYTHON_BIN" --no-python-downloads "$VENV_DIR"
  uv pip sync --python "$VENV_DIR/bin/python" --require-hashes "$LOCK_FILE"
  uv pip check --python "$VENV_DIR/bin/python"
)
```

新版本可放在任务的 `envs/<环境标识>/` 下；失败时保留日志，先诊断再继续，不删除已验证环境。不得使用 `uv pip install --system`。依赖锁定与同步见 [uv 锁定环境说明](https://docs.astral.sh/uv/pip/compile/)。

### 配置 CANN 和统一激活入口

- 核对所选 torch-npu 对应的官方安装与兼容说明，入口见 [Ascend PyTorch 仓库](https://github.com/Ascend/pytorch)。实际部署记录具体版本文档或 commit，不能只引用会变化的主分支。
- 已有 CANN 完整且兼容时可加载；否则在工作根目录内安装对应 Toolkit 和硬件所需算子组件。安装包名称、组件拆分、参数和顺序以该版本说明为准，不固定旧版本命令。
- 安装前检查安装器帮助、依赖、磁盘空间和写入位置，确认普通用户安装可行，避免默认写入远端 `/tmp` 或系统目录。必须修改系统或驱动时，说明具体前提，由管理员处理或在明确授权范围内执行。
- 每套环境保存独立激活脚本，加载用户配置、准确的虚拟环境和 CANN 入口，并设置 Ascend 工作目录、进程日志及 Ascend/PyTorch/Triton 缓存；避免混用不同版本运行库。
- 不通过改写所有任务共用的默认 CANN 或 Python 指向来切换版本；在任务入口选择已验证环境。

### 验证与交付

1. 新开普通用户会话，验证登录、配置加载、解释器位置、实际包版本与目标一致。
2. 检查依赖完整性及 `torch`、`torch_npu` 导入；核对实际 CANN 和驱动信息，不能只看目录名判断版本。
3. 按 [SKILL.md 的验证要求](../SKILL.md#验证与交付)检查设备后运行单卡运算冒烟；需要分布式能力时再执行有限超时的多卡 HCCL 冒烟。无空闲设备时记录“待验证”，不宣称已全部通过。
4. 保存命令、退出码和日志；区分“已安装”“导入通过”“单卡通过”“多卡通过”和任务验收结论。验证成功后更新任务所用环境入口。
