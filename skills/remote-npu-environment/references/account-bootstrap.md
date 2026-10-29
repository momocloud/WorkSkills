# 普通用户的管理员初始化

日常任务必须直接以已确认的普通用户登录，不自动回退到 root，也不通过 root 登录后切换用户来执行任务。

**创建账号和配置设备访问属于独立的管理员初始化步骤。** 普通用户登录失败不等于账号不存在，应先由管理员核实。用户或管理员可按以下步骤操作；agent 只有在用户已明确授权该机器的初始化范围、且提供可用管理员入口时才执行。已有该项授权时不重复索取确认。

以下示例在目标机器的管理员终端执行，适用于提供 `useradd` 的 Linux 系统；先替换占位符，不与普通用户任务脚本混用：

```bash
account_name='<确认的普通用户名>'
account_home='<确认的绝对 Home 路径>'
getent passwd "$account_name"
# 仅在确认账号不存在、目标目录没有归属冲突后执行下一条。
useradd --create-home --home-dir "$account_home" --shell /bin/bash "$account_name"
```

- 账号已存在时核对 UID/GID、Home、Shell 和目录归属，不重复创建或递归更改已有目录所有者。不要固定复用旧机器的 UID/GID；若保留磁盘或共享存储，先核对原文件属主，避免 UID 冲突。
- 使用客户端提供的公钥，不传输私钥。核实实际 SSH 授权文件路径；默认使用 `~/.ssh/authorized_keys` 时，按以下步骤配置：

  ```bash
  # 继续在管理员终端执行；账号与 Home 已核实。
  account_group=$(id -gn "$account_name")
  install -d -m 700 -o "$account_name" -g "$account_group" "$account_home/.ssh"
  touch "$account_home/.ssh/authorized_keys"
  # 将已核实的客户端公钥作为完整一行追加到 authorized_keys，保留已有内容并避免重复。
  chown "$account_name:$account_group" "$account_home/.ssh/authorized_keys"
  chmod 600 "$account_home/.ssh/authorized_keys"
  ```

  修改前备份已有授权文件，并确认路径未通过符号链接指向其他账号。集中认证、自定义 AuthorizedKeysFile 或 SELinux 由管理员按实际配置处理，不为登录方便关闭安全机制。
- 核对 NPU 设备节点实际属组和驱动要求。需要附加组时使用 `usermod -aG <确认的设备访问组> <用户名>`；不得假定所有机器都使用 `HwHiAiUser`，也不得把设备节点改成对所有人可写。
- 只有确实需要 Docker 且用户已授权时才加入 `docker` 组，不将其作为 Python/NPU 环境的默认前提。Docker 访问具有接近 root 的主机控制能力。
- 初始化结束后退出管理员会话，从客户端重新以普通用户登录，检查 `id`、Home 可写性和设备访问；新增组权限须通过新会话验证。后续安装和测试均以普通用户执行。
- 没有管理员入口时，向用户提供缺失项和操作说明，不绕过权限。驱动、系统包或 SSH 服务缺失也交由管理员处理，不扩大账号初始化的授权范围。
