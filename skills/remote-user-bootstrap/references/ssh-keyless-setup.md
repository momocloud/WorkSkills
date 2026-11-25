# 本机免密登录与 SSH 配置（客户端命令参考）

以下内容供生成「本机终端执行命令」时选用。**根据用户实际操作系统只输出对应分支**，不要把 Windows、WSL、Linux、macOS 的命令混在一起堆给用户，也不要直接替用户执行。

## 1. Windows 本机（PowerShell）

### 关键注意点

- Windows 自带 OpenSSH 客户端，但**没有 `ssh-copy-id` 命令**。
- **PowerShell 5.1 编码陷阱**：不要用 `echo ... >> ~/.ssh/config` 或 `> ~/.ssh/config`，Windows PowerShell 5.1 默认写入 UTF-16LE（带 BOM），会导致 OpenSSH 解析配置失败。使用 `Add-Content -Encoding utf8` 或 `[System.IO.File]::AppendAllText`（UTF-8 无 BOM）。
- 尽量使用 `ed25519` 密钥，若目标旧系统不支持再回退 `rsa -b 4096`。

### 准备 SSH 密钥并读取公钥

```powershell
# 1) 检查 OpenSSH 客户端与现有密钥
ssh -V
Get-ChildItem "$env:USERPROFILE\.ssh" -ErrorAction SilentlyContinue

# 2) 如无可用密钥，生成新密钥（建议按需加 -f 指定独立文件名）
ssh-keygen -t ed25519 -C "<备注，如 username@windows>" -f "$env:USERPROFILE\.ssh\id_ed25519"

# 3) 打印公钥整行内容（复制后粘贴到远端 root 命令的 PUB_KEY 中）
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

（可选）若用户希望直接从 PowerShell 通过已有 `root` SSH 把公钥推到远端新用户，可提供管道命令（仅在远端已建好用户后使用，注意剔除 `\r`）：

```powershell
Get-Content -Raw "$env:USERPROFILE\.ssh\id_ed25519.pub" | ssh -p <PORT> root@<HOST> "
  set -e
  U='<USERNAME>'; H='<USER_HOME>'; G=\$(id -gn \"\$U\")
  install -d -m 700 -o \"\$U\" -g \"\$G\" \"\$H/.ssh\"
  K=\$(tr -d '\r\n')
  touch \"\$H/.ssh/authorized_keys\"
  grep -qxF \"\$K\" \"\$H/.ssh/authorized_keys\" || printf '%s\n' \"\$K\" >> \"\$H/.ssh/authorized_keys\"
  chown \"\$U:\$G\" \"\$H/.ssh/authorized_keys\"
  chmod 600 \"\$H/.ssh/authorized_keys\"
"
```

### 写入 Windows `~/.ssh/config`（UTF-8 无 BOM）

```powershell
$sshDir = Join-Path $env:USERPROFILE ".ssh"
New-Item -ItemType Directory -Force -Path $sshDir | Out-Null
$configPath = Join-Path $sshDir "config"

$entry = @"

Host <SSH_ALIAS>
    HostName <HOST>
    Port <PORT>
    User <USERNAME>
    IdentityFile ~/.ssh/<KEY_NAME>
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
"@
# 如需跳板机，在上面块内追加：    ProxyJump <BASTION_ALIAS_OR_USER_HOST>

$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::AppendAllText($configPath, $entry.Replace("`r`n", "`n"), $utf8NoBom)
```

### 私钥权限异常修复（仅在 SSH 提示 `UNPROTECTED PRIVATE KEY FILE` 时使用）

```powershell
$keyPath = "$env:USERPROFILE\.ssh\<KEY_NAME>"
icacls $keyPath /inheritance:r
icacls $keyPath /grant:r "$($env:USERNAME):(R)"
```

---

## 2. WSL（以及 Windows + WSL 双端共用）

### 关键注意点

- **绝对不要直接把 SSH 私钥指向 `/mnt/c/Users/<WinUser>/.ssh/...`**：WSL 挂载的 Windows 分区默认权限常为 `0777`，OpenSSH 会直接报错 `Permissions 0777 ... are too open` 并拒绝使用该私钥。
- 正确做法是把密钥保存在 WSL 内部的 Linux 文件系统 `~/.ssh/` 下，并设 `700` / `600` 权限。
- 如果用户 **Windows 和 WSL 两端都需要免密登录**，给出两种清晰方案（按用户偏好选一种）：
  - **方案 A（推荐，共用一套密钥对）**：在一端生成（或使用现有 Windows 密钥），复制到另一端各自的 `~/.ssh/` 下并修正权限，远端只需写入一份公钥。
  - **方案 B（两端独立密钥）**：Windows 与 WSL 各自生成一对密钥，将两份 `.pub` 公钥都追加到远端 `$USER_HOME/.ssh/authorized_keys`。

### 情况 2A：在 WSL 内新生成密钥

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -C "<备注，如 username@wsl>" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

### 情况 2B：在 WSL 内复用 Windows 已有密钥（复制并修权限）

```bash
WIN_USER='<Windows用户名>'
KEY_NAME='<密钥文件名，如 id_ed25519>'

mkdir -p ~/.ssh && chmod 700 ~/.ssh
cp "/mnt/c/Users/${WIN_USER}/.ssh/${KEY_NAME}" "$HOME/.ssh/${KEY_NAME}"
cp "/mnt/c/Users/${WIN_USER}/.ssh/${KEY_NAME}.pub" "$HOME/.ssh/${KEY_NAME}.pub"
chmod 600 "$HOME/.ssh/${KEY_NAME}"
chmod 644 "$HOME/.ssh/${KEY_NAME}.pub"
cat "$HOME/.ssh/${KEY_NAME}.pub"
```

### 写入 WSL `~/.ssh/config`

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/config && chmod 600 ~/.ssh/config

cat <<'EOF' >> ~/.ssh/config

Host <SSH_ALIAS>
    HostName <HOST>
    Port <PORT>
    User <USERNAME>
    IdentityFile ~/.ssh/<KEY_NAME>
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
# 如需跳板机，在上面块内追加：    ProxyJump <BASTION_ALIAS_OR_USER_HOST>
```

---

## 3. 原生 Linux / macOS / Windows Git Bash

```bash
# 1) 检查或生成密钥
ls -la ~/.ssh 2>/dev/null || true
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -C "<备注>" -f ~/.ssh/id_ed25519

# 2) 打印公钥整行内容（粘贴到远端 root 命令，或用下方管道推送）
cat ~/.ssh/id_ed25519.pub
```

写入 `~/.ssh/config` 与上面 WSL 段落一致（macOS 若希望把密钥口令托管到钥匙串，可按需在对应 `Host` 块中增加 `AddKeysToAgent yes` 与 `UseKeychain yes`）。

---

## 4. 验证免密登录与权限（在本机以新建普通用户执行）

所有远端和本机配置完成后，**在尚未关闭 `root` 终端的前提下**，从本机终端执行以下验证命令（如配置了别名直接用 `<SSH_ALIAS>`，否则用 `-p <PORT> -i ~/.ssh/<KEY_NAME> <USERNAME>@<HOST>`）：

```bash
# 1) 验证纯公钥免密登录（BatchMode=yes 禁止回退密码输入）、身份、实际家目录与可写性
ssh -o BatchMode=yes -o ConnectTimeout=10 <TARGET> \
  'whoami && id && pwd && ls -ld "$HOME" && touch "$HOME/.ssh_init_check" && rm -f "$HOME/.ssh_init_check" && echo "HOME_WRITABLE_OK"'
```

按用户实际启用的权限追加对应验证命令（未启用的不输出）：

```bash
# 2) 若加入了 docker 组：验证普通用户可直接访问 docker daemon
ssh -o BatchMode=yes <TARGET> 'docker ps'

# 3) 若配置了 sudo 权限：
#    - 免密 sudo 可直接非交互检查：
ssh -o BatchMode=yes <TARGET> 'sudo -n true && sudo -l'
#    - 需密码 sudo 需分配伪终端交互输入密码：
ssh -t <TARGET> 'sudo -l'

# 4) 若加入了 NPU 等设备组：做只读设备检查
ssh -o BatchMode=yes <TARGET> 'npu-smi info'

# 5) 若配置了额外工作目录：验证读写权限
ssh -o BatchMode=yes <TARGET> 'touch "<WORK_DIR>/.write_test" && rm -f "<WORK_DIR>/.write_test" && echo "WORK_DIR_OK"'
```

> 若用户同时配置了 Windows 与 WSL 双端，提醒**两端都各自执行一次免密登录验证**。

## 5. 常见失败快速排查

| 现象 | 优先检查项（给用户的诊断命令） |
| --- | --- |
| `Permission denied (publickey)` | 本机执行 `ssh -vvv <TARGET>` 看是否发送了正确公钥；远端 root 查 `journalctl -u sshd -n 50`（或 `ssh` / `/var/log/auth.log` / `/var/log/secure`） |
| 非 `/home` 家目录公钥被拒 | 远端 root 执行 `namei -l "<USER_HOME>/.ssh/authorized_keys"` 确认各级祖先目录无 `777`/组可写；再查 `getenforce` 看是否为 SELinux 上下文问题 |
| `sshd` 提示账号不可登录 | 远端 root 查 `passwd -S "<USERNAME>"`（确认非锁定态）与 `sshd -T \| grep -Ei 'allowusers\|allowgroups\|pubkey'` |
| WSL 报 `Permissions 0777 ... are too open` | 确认 `IdentityFile` 指向 WSL 内部 `~/.ssh/` 而非 `/mnt/c/...`，并执行 `chmod 600 ~/.ssh/<KEY_NAME>` |
| Windows 报 `Bad configuration option` 或乱码 | 检查 `%USERPROFILE%\.ssh\config` 是否被 PowerShell 写成 UTF-16LE；改用 UTF-8 无 BOM 重写 |
| 首次连接提示主机指纹确认 | 核对服务器真实指纹后再确认写入 `known_hosts`，不要关闭 `StrictHostKeyChecking` |
