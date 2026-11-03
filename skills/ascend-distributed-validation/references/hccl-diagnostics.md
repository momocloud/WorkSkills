# HCCL 状态与日志定位

## 选择本次诊断配置

先确定本次失败入口、环境、原始命令和协作方的定位要求。诊断时沿用具体用例的启动方式；采集 HCCL 日志并不意味着应该使用某个固定 launcher、单机或固定卡数。

以下是一组曾由 HCCL 定位人员提供的配置，可在当前定位请求要求该方案时使用；它不是所有版本/芯片的统一默认设置，也不保证每次都会产生同样格式的文件。核对当前版本说明、实现或实际日志，保留用户指定的值及其来源，不因旧实验结果擅自替换配置。

| 变量 | 采集或分析时的注意点 |
| --- | --- |
| `TORCH_HCCL_STATUS_SAVE_ENABLE=1` | 请求状态保存；运行后核实是否实际生成文件 |
| `TORCH_HCCL_STATUS_SAVE_PATH` | 指向本次独立状态目录，核对文件与 rank/PID 的对应关系 |
| `TORCH_HCCL_ASYNC_ERROR_HANDLING=1` | 涉及异步错误处理行为，不应当作纯日志开关 |
| `ASCEND_PROCESS_LOG_PATH` | 指向本次独立日志目录，收集实际生成的主机及设备日志 |
| `ASCEND_GLOBAL_LOG_LEVEL=1` | 本方案的日志级别取值；根据当前版本/请求使用，不默认长期保持 |
| `HCCL_ENTRY_LOG_ENABLE=1` | 本方案的入口日志设置；检查实际输出而非只检查 shell 环境 |
| `HCCL_OP_EXPANSION_MODE=AICPU_CacheDisable` | 执行模式实验选项；按定位要求启用，不默认对所有测试设置 |

在环境激活完成后设置本次变量，避免激活脚本覆盖日志路径。优先限于本次命令及其子进程，或独立诊断 shell；不要写进全局启动文件，也不要让下一轮普通性能测试意外继承这些设置。

下面只创建新目录和设置变量，**没有规定或执行测试命令**。`TASK_WORK_DIR` 必须先设为本次已核实的任务工作根目录；不是从旧项目复制的机器路径。

```bash
: "${TASK_WORK_DIR:?请先设置当前任务工作根目录}"
mkdir -p "$TASK_WORK_DIR/logs"
DIAG_DIR="$(mktemp -d "$TASK_WORK_DIR/logs/hccl-diag-XXXXXXXX")"
mkdir -p "$DIAG_DIR/hccl" "$DIAG_DIR/plog"

export TORCH_HCCL_STATUS_SAVE_ENABLE=1
export TORCH_HCCL_STATUS_SAVE_PATH="$DIAG_DIR/hccl"
export TORCH_HCCL_ASYNC_ERROR_HANDLING=1
export ASCEND_PROCESS_LOG_PATH="$DIAG_DIR/plog"
export ASCEND_GLOBAL_LOG_LEVEL=1
export HCCL_ENTRY_LOG_ENABLE=1
export HCCL_OP_EXPANSION_MODE=AICPU_CacheDisable
```

接下来执行已经从当前用例确定的命令。不要为使模板“完整”自行补入某个脚本名或 launcher。新目录无需 `rm -rf`；用户要求清空旧路径时，先确认路径和归属，保存仍有价值的证据。

## 收集与阅读

保存实际命令、环境覆盖、源码身份、运行日志、状态文件、主机/设备日志及清理记录。多 rank、多节点时标明 node、rank、PID、通信域与文件路径；不能默认某一个 rank 的日志代表所有 rank。日志滚动、缺文件或无法访问时，写明采集缺口。

大日志先清点文件和体积，再按异常时间窗口、PID/rank 和通信域缩小范围；保留关键行号及上下文，而非只摘最终退出错误。打包原始日志并校验传输完整性，摘要可独立提供。用户仅要求分析时使用已有日志，不因缺一份文件自动复跑。

读取状态文件时结合实际格式检查最后通信操作、状态、通信域 ID、序号、异常和时间信息。只有与当前源码的建域顺序及运行证据对应后，才将域解释为 native/symm 等角色。PyTorch 的通信序号与 HCCL `opIndex` 可能属于不同编号体系，不能直接等同。

从主机和设备日志核对：

- 变量是否被读取、解析为何值；只记录 export 不能证明后端采用了所期望的路径。
- 首个通信/内核异常及后续连带错误，和运行器的超时/清理行为分别是什么。
- 失败操作、dtype、count、输入/输出字节数和域/算法信息；与具体用例的尺寸语义对齐后再定位失败档位。
- 状态快照是否采于同一时刻，是否存在旧快照、日志交错或滚动。序号不同不直接证明某个 rank 最先出错。

## 解释限制与交接

例如，主机日志可能显示 `aicpuCacheEnable[0]`，设备日志同时出现 `AICPU_CacheDisable is not supported`。两处都应保留并请实现方解释；不能仅凭后一条认定变量未读取、所有版本都不支持，或实际缓存一定开启/关闭。

`507018` 等错误码用于归类线索，不能单独划定 torch-npu/HCCL/CANN 的责任。一次同时改变多个变量的诊断运行，也不能当作仅修改某一个开关的因果实验。

交接材料应包含复现入口和实际参数、环境/源码、原始错误、变量解析证据、关键日志行号、各 rank 状态及未解决的问题。指出诊断流程结束与测试是否通过的区别。材料准备好不等于获准向协作方发送，按用户的发送要求处理。
