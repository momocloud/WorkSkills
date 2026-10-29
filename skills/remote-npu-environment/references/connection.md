# 连接与排查

### 正常连接

本地 Bash 示例；先替换占位符。使用 SSH 别名时保留其所需的客户端配置：

```bash
REMOTE_HOST='<IP、域名或 SSH 别名>'
REMOTE_USER='<普通用户名>'
ssh -o BatchMode=yes -o ConnectTimeout=10 \
  -o StrictHostKeyChecking=yes -o UpdateHostKeys=no \
  "${REMOTE_USER}@${REMOTE_HOST}"
```

按需增加 `-p <端口>`、`-i <本地私钥路径>`、`-o IdentitiesOnly=yes` 或 `-J <跳板机>`。日常连接可加 `-o LogLevel=QUIET`；诊断失败时去掉静默选项，必要时使用 `-v`。参数见 [OpenSSH 文档](https://man.openbsd.org/ssh)。

只有确实遇到本地 SSH 配置权限或配置冲突时，才尝试 `-F /dev/null`；它会跳过配置，使用前须显式补齐真实主机、端口、密钥和跳板机等信息，不能无条件套用。

### 按错误分类处理

| 现象 | 检查与下一步 |
| --- | --- |
| 无法解析、网络不可达、超时 | 核对地址、端口、VPN、路由及跳板机；根据证据修正后再试，仍不可达则说明所需网络条件 |
| Connection refused | 核对端口与主机状态；请用户或管理员检查 SSH 服务，不据此断定账号不存在 |
| 首次连接，无可信主机密钥 | 从管理员或可信控制台获取指纹，核对后写入对应的 known_hosts；`ssh-keyscan` 只能采集密钥，不能证明其可信 |
| 主机密钥变化 | 可能重装或目标变化；先确认机器身份和新指纹，再备份并更新该主机及端口对应的记录，不删除整个 known_hosts 或关闭校验 |
| Permission denied (publickey) | 核对用户名、密钥及 SSH agent；必要时显式指定密钥。请管理员检查账号、公钥授权和目录权限，不默认改用 root |
| 登录后身份、Home 或权限不符 | 停止写入，核对账号和目录；区分文件不存在与权限不足，不用 sudo 掩盖问题 |
| 登录成功，但环境或命令缺失 | 先检查路径、挂载及非交互 Shell 配置是否加载，再按[环境检查与复现](environment.md)判断是否重建 |

每次重试应有明确依据，不无限重复相同命令、不猜测其他账号或扫描无关机器。无法解决时报告已尝试的方式、错误类型和需要补充的信息；诊断日志不得包含凭据。
