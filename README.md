# ssh-gateway-for-l

通过公网服务器的指定端口直接访问跳板机后面的内网服务器，跳板机私钥只保存在公网服务器上，不分发给用户。

```text
用户 ──ssh -p 10022──▶ 公网服务器 ──SSH 隧道(跳板机私钥)──▶ 跳板机 ──TCP──▶ 目标服务器:22
```

用户执行 `ssh -p 10022 用户名@公网服务器IP` 即直接登录目标服务器，认证由目标服务器完成（用户使用自己在目标服务器上的账号/密钥）。跳板机与目标服务器无需任何改动。

实现方式：一个 Bash 脚本 `ssh-gateway`，用 systemd 模板单元为每条隧道运行一个 `ssh -L` 进程，异常退出后自动重连。

## 目录结构

所有内容都在安装目录内，解压即用：

```text
ssh-gateway/
├── ssh-gateway              # 管理脚本（唯一入口）
├── ssh-gateway@.service     # systemd 单元，由 install 生成
├── tunnels/<name>.conf      # 每条隧道一个配置文件
├── keys/                    # 跳板机私钥（目录权限 700）
└── known_hosts              # 跳板机 host key，首次连接时自动记录
```

目录之外只会产生 `/etc/systemd/system/` 下的几个软链接，由 `install` / `start` 创建，由 `stop` / `remove` / `uninstall` 删除。

## 环境要求

- Linux + systemd
- bash、openssh-client（`ssh`）、iproute2（`ss`）
- root 权限（管理 systemd 服务）

## 部署

以下命令均在公网服务器上执行。

### 1. 放置程序

建议放在 root 所有的目录下。服务以 root 运行，目录不能让普通用户可写：

```bash
sudo mkdir -p /opt/ssh-gateway
sudo cp ssh-gateway /opt/ssh-gateway/
sudo chmod 755 /opt/ssh-gateway/ssh-gateway
cd /opt/ssh-gateway
```

### 2. 注册服务

```bash
sudo ./ssh-gateway install
```

该命令会创建 `tunnels/`、`keys/`，生成单元文件，并通过 `systemctl link` 注册到 systemd。以后如果移动了安装目录，需要在新位置重新执行一次 `install`。

### 3. 添加跳板机密钥

跳板机必须能用这把密钥登录，即对应公钥已在跳板机用户的 `~/.ssh/authorized_keys` 中。

**方式 A：使用已有的私钥**（跳板机已接受它）

```bash
# 从本机上传到公网服务器后，放入 keys/ 并收紧权限
sudo install -m 600 -o root -g root jump.key /opt/ssh-gateway/keys/jump.key
rm jump.key   # 删除上传用的临时副本
```

**方式 B：在公网服务器上新生成一对密钥**（私钥从不离开公网服务器）

```bash
sudo ssh-keygen -t ed25519 -N '' -C ssh-gateway -f /opt/ssh-gateway/keys/jump.key
sudo cat /opt/ssh-gateway/keys/jump.key.pub
```

然后请跳板机管理员把输出的公钥追加到跳板机用户的 `~/.ssh/authorized_keys`。

注意：

- **私钥不能有密码**，服务在后台运行，无法交互输入。已有私钥带密码的，可以先在副本上去除：`ssh-keygen -p -N '' -f jump.key`。
- 私钥权限必须是 `600`，否则 ssh 会拒绝使用（`check` 会检查）。
- 多个跳板机可以放多把密钥，如 `keys/jumpA.key`、`keys/jumpB.key`，在各自的隧道配置里分别引用。

### 4. 添加隧道配置

```bash
sudo ./ssh-gateway add target01
sudo vi tunnels/target01.conf
```

配置示例：

```ini
# 公网服务器上的监听地址和端口
LISTEN_ADDR=0.0.0.0
LISTEN_PORT=10022

# 跳板机
JUMP_HOST=1.2.3.4
JUMP_PORT=22
JUMP_USER=jumpuser
JUMP_KEY=keys/jump.key

# 目标服务器（从跳板机访问的地址）
TARGET_HOST=192.168.1.20
TARGET_PORT=22
```

| 字段 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `LISTEN_ADDR` | 否 | `0.0.0.0` | 监听地址。`0.0.0.0` 对外开放；`127.0.0.1` 仅限本机 |
| `LISTEN_PORT` | 是 | | 用户连接的端口，各隧道之间不能重复 |
| `JUMP_HOST` | 是 | | 跳板机 IP 或域名 |
| `JUMP_PORT` | 否 | `22` | 跳板机 SSH 端口 |
| `JUMP_USER` | 是 | | 跳板机登录用户 |
| `JUMP_KEY` | 是 | | 私钥路径，相对路径以安装目录为基准，也可写绝对路径 |
| `TARGET_HOST` | 是 | | 目标服务器地址（跳板机视角下的内网 IP 或主机名） |
| `TARGET_PORT` | 否 | `22` | 目标服务器 SSH 端口 |

格式规则：每行一个 `KEY=VALUE`，值不能包含空格；`#` 之后为注释。不支持 IPv6 地址。出现未知字段或非法值时，`start` / `check` 会报出具体行号。

每个目标服务器对应一个配置文件，文件名即隧道名（字母、数字、`_`、`-`）。

### 5. 启动并检查

```bash
sudo ./ssh-gateway start target01    # 启动并设置开机自启
sudo ./ssh-gateway check target01    # 检查配置、服务、端口，并验证能否连通目标服务器
```

`check` 全部通过时，最后一行输出类似 `target reachable: SSH-2.0-OpenSSH_...`，表示整条链路已打通。

首次连接时，会自动记录跳板机的 host key 到 `known_hosts`；此后 host key 一旦变化就拒绝连接。

### 6. 开放端口

如果公网服务器启用了防火墙或云安全组，需要放行 `LISTEN_PORT`（建议限制来源 IP），例如：

```bash
sudo ufw allow 10022/tcp
```

### 7. 用户连接

```bash
ssh -p 10022 用户名@公网服务器IP
```

也可以写入用户本机的 `~/.ssh/config`：

```text
Host target01
    HostName 公网服务器IP
    Port 10022
    User 用户名
```

## 管理命令

| 命令 | 说明 |
|---|---|
| `install` | 注册 systemd 单元 |
| `uninstall` | 停止所有隧道，删除全部 systemd 痕迹 |
| `add <name>` | 从模板创建 `tunnels/<name>.conf` |
| `remove <name>` | 停止隧道并删除其配置 |
| `start <name>` | 启动隧道并设置开机自启 |
| `stop <name>` | 停止隧道并取消开机自启 |
| `reload [name]` | 修改配置后重启隧道使其生效，省略 name 则处理全部已启用的隧道 |
| `status` | 列出所有隧道的状态和路由 |
| `check [name]` | 校验配置并测试到目标服务器的完整链路 |
| `run <name>` | 前台运行隧道（供 systemd 调用，也可用于调试） |

除 `add`、`status`、`check`、`run` 外，其余命令都需要 root 权限。在 root 所有的安装目录下，建议所有命令都用 `sudo` 执行。

## 自动重连

- ssh 进程退出（网络中断、跳板机重启等）后，systemd 每 5 秒重试一次，没有次数上限。
- 连接无响应时，约 45 秒（`ServerAliveInterval=15` × 3）判定断开，然后重建隧道。
- 监听端口转发失败（例如端口被占用）时，ssh 直接退出并重试，不会停留在"假活"状态。

## 故障排查

```bash
sudo ./ssh-gateway check target01          # 定位是哪一环出问题
journalctl -u ssh-gateway@target01 -f      # 查看 ssh 日志
sudo ./ssh-gateway run target01            # 前台运行，直接看到 ssh 输出（需先 stop）
```

常见问题：

- `Permission denied (publickey)`（日志中）：跳板机不接受该私钥，检查公钥是否已加入跳板机，以及 `JUMP_USER` 是否正确。
- `REMOTE HOST IDENTIFICATION HAS CHANGED`：跳板机 host key 变了。确认变更合法后，删除旧记录：`sudo ssh-keygen -R '1.2.3.4' -f known_hosts`（端口不是 22 时写成 `'[1.2.3.4]:2222'`），再执行 `reload`。
- `check` 报 `no SSH banner from target`：隧道已建立，但跳板机连不上目标服务器，检查 `TARGET_HOST` / `TARGET_PORT`。

## 卸载

```bash
sudo /opt/ssh-gateway/ssh-gateway uninstall
sudo rm -rf /opt/ssh-gateway
```

执行后系统中不再有任何残留（journald 中的历史日志除外）。
