# 远端用户与权限配置（root 终端命令参考）

以下内容供生成「远端 `root` 终端执行命令」时选用。**只输出用户实际需要的部分**，并把用户已确认的参数直接填进变量定义；不要直接替用户执行。

## 1. 前置只读探查命令

当用户尚未确认磁盘空间、发行版、现有用户/组或 SSH 策略时，先提供以下只读检查命令：

```bash
# 查看各挂载点可用空间（用于决定非 /home 的家目录落点）
df -hT

# 查看系统发行版
cat /etc/os-release

# 检查目标用户名与候选目录是否已存在（按需替换）
getent passwd "<USERNAME>" || echo "用户不存在"
ls -ld "<USER_HOME>" 2>/dev/null || echo "目标家目录尚不存在"

# 检查 docker 组、docker socket 与可选的 NPU 设备属组
getent group docker || echo "docker 组不存在"
ls -l /var/run/docker.sock 2>/dev/null || true
ls -l /dev/davinci* /dev/davinci_manager /dev/devmm_svm /dev/hisi_hdc 2>/dev/null || true

# 检查 sshd 关键限制项与 SELinux 状态
sshd -T 2>/dev/null | grep -E '^(pubkeyauthentication|authorizedkeysfile|allowusers|allowgroups|denyusers|denygroups|strictmodes|usepam) ' || \
  grep -Ein '^(PubkeyAuthentication|AuthorizedKeysFile|AllowUsers|AllowGroups|DenyUsers|DenyGroups|StrictModes)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null || true
command -v getenforce >/dev/null && getenforce || echo "SELinux 未启用"
```

## 2. 创建普通用户与自定义家目录

### 关键注意点

- 统一优先使用 `useradd`，避免 `adduser` 在 Debian/Ubuntu 与 RHEL/openEuler 之间参数不兼容。
- 当家目录不在 `/home` 下（例如 `/data/users/alice`）：
  1. 必须先 `mkdir -p "$(dirname "$USER_HOME")"`，否则 `useradd -m -d` 会因父目录不存在而失败。
  2. **OpenSSH `StrictModes` 权限陷阱**：`sshd` 要求从 `/` 到 `$USER_HOME` 的各级祖先目录对目标用户可穿越（有 `x` 权限），且**不能是 group/other 可写（例如 `777`）**，否则即使 `~/.ssh` 为 `700`、公钥文件为 `600`，免密登录仍会被静默拒绝。
  3. **账号已存在或目录已存在时**：不盲目重复创建，也不对未知已有目录执行 `chown -R`（避免破坏他人数据或共享目录归属）。
- **无密码账号与 PAM 锁定态**：部分发行版（如 RHEL/openEuler 或特定 PAM 配置）下，`useradd` 新建账号在 `/etc/shadow` 中密码字段为 `!`（锁定态），可能导致 SSH 公钥登录被拒。
  - 若用户需要密码（如执行 `sudo`）：交互执行 `passwd "$USERNAME"`。
  - 若用户明确只要免密登录、不设密码：用 `usermod -p '*' "$USERNAME"` 将密码字段设为 `*`（禁用口令登录但不锁定账号），比保持 `!` 更稳妥。

### 命令模板

```bash
USERNAME='<确认的用户名>'
USER_HOME='<确认的绝对家目录路径，如 /data/users/alice>'
USER_SHELL='<确认的Shell，如 /bin/bash>'

# 1) 检查账号是否已存在；已存在则停止，避免误覆盖
if id "$USERNAME" >/dev/null 2>&1; then
  echo "账号 $USERNAME 已存在，请先核对："
  getent passwd "$USERNAME"
else
  # 2) 确保家目录的父目录存在（适用于非 /home 路径）
  mkdir -p "$(dirname "$USER_HOME")"

  # 3) 创建用户并初始化自定义家目录
  useradd -m -d "$USER_HOME" -s "$USER_SHELL" "$USERNAME"

  # 若目标目录事先已为空目录存在，确保属主与基础权限正确
  USER_GROUP="$(id -gn "$USERNAME")"
  chown "$USERNAME:$USER_GROUP" "$USER_HOME"
  chmod 750 "$USER_HOME"
fi

# 4) 核对账号信息及从根目录到家目录的各级权限（确认无 777/他人可写目录）
getent passwd "$USERNAME"
namei -l "$USER_HOME"
```

密码策略二选一（按用户回答只保留对应一项）：

```bash
# 选项 A：为用户设置登录密码（需交互输入两次，用于 sudo 鉴权或控制台登录）
passwd "$USERNAME"

# 选项 B：仅允许 SSH 公钥登录，不设登录密码（将 shadow 设为 '*' 避免部分系统因锁定态拒绝公钥）
usermod -p '*' "$USERNAME"
```

## 3. 附加组与权限配置

### 加入 `docker` 组（按需）

```bash
if getent group docker >/dev/null 2>&1; then
  usermod -aG docker "$USERNAME"
  id "$USERNAME"
else
  echo "当前系统不存在 docker 组，请先确认 Docker 是否已安装及运行方式。"
fi
```

> 提醒用户：`docker` 组具备接近 `root` 的主机控制能力；新组权限需在新登录会话中生效。

### 配置 `sudo` 权限（按需）

推荐使用 `/etc/sudoers.d/<USERNAME>` 独立配置文件 + `visudo -cf` 语法校验：跨 Ubuntu/Debian/openEuler/CentOS 通用、不破坏主 `/etc/sudoers`、且可随时安全撤销。

```bash
SUDO_FILE="/etc/sudoers.d/${USERNAME}"

# 方式 A：常规 sudo（执行 sudo 时需要输入该用户密码；需已为该用户设置密码）
cat <<EOF > "$SUDO_FILE"
${USERNAME} ALL=(ALL:ALL) ALL
EOF

# 方式 B：免密 sudo（仅在用户明确要求免密 sudo 时使用，替代方式 A）
# cat <<EOF > "$SUDO_FILE"
# ${USERNAME} ALL=(ALL:ALL) NOPASSWD: ALL
# EOF

chmod 440 "$SUDO_FILE"
visudo -cf "$SUDO_FILE"
```

若用户仅需要特定命令的 `sudo` 权限，将上面规则中的 `ALL` 替换为经核实的命令绝对路径清单，并在 `visudo -cf` 通过后生效。若校验失败，立即删除该文件（`rm -f "$SUDO_FILE"`）排查，避免破坏 `sudo`。

### 加入硬件设备组（如 NPU / GPU，按需）

```bash
# 先按前面 ls -l /dev/davinci* 等查到的实际属组替换 <DEVICE_GROUP>
# 不要假定所有机器都使用 HwHiAiUser，也不要把设备节点改成 chmod 777
DEVICE_GROUP='<查到的实际设备属组>'
getent group "$DEVICE_GROUP" && usermod -aG "$DEVICE_GROUP" "$USERNAME"
```

### 额外数据 / 工作目录授权（按需）

```bash
WORK_DIR='<确认的额外工作目录绝对路径>'
USER_GROUP="$(id -gn "$USERNAME")"
mkdir -p "$WORK_DIR"
chown "$USERNAME:$USER_GROUP" "$WORK_DIR"
ls -ld "$WORK_DIR"
```

## 4. 配置 SSH 公钥（`authorized_keys`）与登录限制核查

### 写入公钥（幂等、防重复、剔除 Windows `\r`）

```bash
USER_GROUP="$(id -gn "$USERNAME")"
PUB_KEY='<粘贴本机公钥整行内容，例如 ssh-ed25519 AAAA... comment>'

install -d -m 700 -o "$USERNAME" -g "$USER_GROUP" "$USER_HOME/.ssh"
AUTH_KEYS="$USER_HOME/.ssh/authorized_keys"

# 若已有 authorized_keys，先做时间戳备份
[ -f "$AUTH_KEYS" ] && cp -a "$AUTH_KEYS" "${AUTH_KEYS}.bak.$(date +%Y%m%d%H%M%S)"
touch "$AUTH_KEYS"

# 剔除可能混入的 Windows 回车符，避免重复追加
CLEAN_KEY="$(printf '%s' "$PUB_KEY" | tr -d '\r')"
grep -qxF "$CLEAN_KEY" "$AUTH_KEYS" || printf '%s\n' "$CLEAN_KEY" >> "$AUTH_KEYS"

chown "$USERNAME:$USER_GROUP" "$AUTH_KEYS"
chmod 600 "$AUTH_KEYS"
ls -ld "$USER_HOME" "$USER_HOME/.ssh" "$AUTH_KEYS"
```

> 若用户有多端（例如 Windows 和 WSL 使用不同密钥），重复上述 `PUB_KEY` 追加片段即可把多把公钥都写入同一个 `authorized_keys`。

### 非 `/home` 家目录的 SELinux 上下文处理（RHEL / CentOS / openEuler）

若前面检查发现 `getenforce` 返回 `Enforcing`，且 `$USER_HOME` 不在 `/home` 下，OpenSSH 常因缺少 `ssh_home_t` 标签而拒绝公钥。给出如下条件修复命令：

```bash
if command -v getenforce >/dev/null 2>&1 && [ "$(getenforce)" = "Enforcing" ]; then
  if command -v semanage >/dev/null 2>&1; then
    semanage fcontext -a -e /home "$(dirname "$USER_HOME")" 2>/dev/null || true
    restorecon -Rv "$USER_HOME"
  else
    # 缺少 semanage 时至少先为 .ssh 补齐常见上下文（提示可安装 policycoreutils-python-utils）
    chcon -R -t ssh_home_t "$USER_HOME/.ssh"
  fi
fi
```

### 核查 `sshd_config` 白名单（如有 `AllowUsers` / `AllowGroups`）

仅当前置探查显示存在 `AllowUsers`、`AllowGroups` 或 `PubkeyAuthentication no` 时提醒用户按实际配置放行，并遵守两条安全底线：

1. 修改前备份配置，修改后**必须先执行 `sshd -t` 校验无误**再重载（`systemctl reload sshd || systemctl reload ssh`）。
2. **保留当前 `root` 终端会话不要关闭**，等本机用新建普通用户验证登录成功后再退出 `root`。
