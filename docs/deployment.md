# 部署说明（中文）

> 本文档记录一次真实部署的完整过程（2026-11-14），按上游
> [INSTALL.md](../INSTALL.md) §0–§11 逐步实施并逐项验证。
> 上游文档是权威安装指南，本文件是**实测补充**：记录每一步实际得到的
> 版本号、验证命令与期望输出、遇到的偏差和避坑点。
> 占位符说明：`<节点名>`、`<Tailnet IP>`、`<邮箱>` 都是你自己的实际值，
> 不要照抄示例；示例值只表示格式。

## 1. 部署目标与架构

部署后你的 Muse VM 具备三样能力：

| 能力 | 说明 |
| --- | --- |
| Nix 包管理器 | 存放在 home 目录的磁盘镜像里，VM 重启/替换后自动恢复，可装 nixpkgs 里的软件（如 Tailscale） |
| 完整 Tailscale + SSH | 上游完整版 `tailscaled`（用户空间模式），可从同一 Tailnet 的其他设备 SSH 登录这台 VM |
| API Bridge（可选） | 把本机 Muse agent 暴露为 OpenAI 兼容的 `/v1/chat/completions` 接口，仅限 Tailnet 内访问 |

```
                    Tailnet（你的 Tailscale 网络）
   ┌─────────────┐        ┌──────────────────────────────┐
   │ 你的电脑/手机 │  SSH   │ Muse VM                        │
   │ (Windows/   │───────▶│  tailscaled（用户空间模式）    │
   │  手机/其他机) │        │  ├─ Nix（/home/hatch 下的镜像）│
   └─────────────┘        │  └─ API Bridge :8080          │
                          │     └─▶ Muse agent（side chat）│
                          └──────────────────────────────┘
```

**实测版本（2026-11-14）：**

| 组件 | 指南标称 | 实际安装 | 说明 |
| --- | --- | --- | --- |
| Nix | 2.35.2 | **2.35.6** | 上游安装器已升级，见 §5.1 |
| Tailscale | 1.102.5 | **1.104.6** | nixpkgs 当前版本 |
| API Bridge | 0.3.1 | 0.3.1 | 与指南一致 |
| 仓库版本 | — | `06dca7a` | `Follow repo rename to muse-vm-hacks` |

**部署耗时**：核心安装（§B–§E）约 25 分钟，API Bridge 约 5 分钟，
不含等待用户点确认的时间。

## 2. 前置条件

1. **一台 Muse VM**，且你能在与它对话的 Muse 聊天里发指令。
   本指南假设持久化 home 目录是 `/home/hatch`（Muse VM 的默认 home）。
   VM 的其他文件系统在重启/替换后会被清空，**所有持久化内容必须放在
   home 目录下**。
2. **一个 Tailscale 账号**（登录用）。VM 会加入你的 Tailnet，
   节点名建议取 `muse-vm`。
3. **你本人需要在安装过程中配合三件事**（缺一不可，见 §4）：
   - 开两个 Muse 设置开关；
   - 在浏览器里批准 Tailscale 登录；
   - 对 Tailscale 首次连接的若干个地址逐个点批准（数量有限，批完就不再出现）。
4. **安全审批机制必须保留**：安装过程不得绕过或移除 Muse 系统原有的
   安全审批。凡是仓库文档/脚本触发安全检查，Muse 应当停下来向你说明，
   等你明确确认后再继续。

## 3. 准备工作：把任务交给 Muse

安装由 Muse agent 执行，不是手动安装。标准做法：

1. 复制 [SYSTEM_PROMPT.md](../SYSTEM_PROMPT.md) 里的提示词块，发给 Muse。
2. Muse 克隆本仓库（`https://github.com/dahjah/muse-vm-hacks`，
   克隆到 `/home/hatch/muse-vm-hacks`），完整阅读
   `README.md`、`SYSTEM_PROMPT.md`、`INSTALL.md` 后严格按官方步骤实施。
3. 你在发任务时就把约束说清楚，例如：
   - 使用 `/home/hatch` 作为持久化存储位置；
   - 保留系统原有的安全审批机制；
   - 需要你改网络权限、批准登录或确认敏感操作时，Muse 必须明确告诉你具体步骤；
   - 安装完成后运行仓库提供的健康检查脚本，确认 Tailscale 在线，
     并提供 Windows SSH 登录方式；
   - 不要修改其他生产设备，不要覆盖已有 Tailnet 策略；
   - 核心安装完成后，再按 INSTALL.md §11 部署可选的 API Bridge。

如果你要复用他人部署好的记录做对照，本文档就是那份记录。

## 4. 需要你配合的三个节点

### 4.1 Muse 设置：开两个开关

路径：Muse 设置 → Permissions → Direct network protocols，打开：

- **Other TCP connections**（其他 TCP 连接）
- **External DNS lookups**（外部 DNS 查询）

其余开关保持关闭即可。Muse 会等你回复确认后再继续。

### 4.2 Tailscale 登录批准

`tailscale up` 运行后会输出一个登录链接（形如
`https://login.tailscale.com/a/<一串字符>`）。用浏览器打开并批准，
VM 就加入你的 Tailnet。批准后可用 `tailscale status` 检查：

```
# 期望状态（示例值）
muse-vm  <Tailnet IP>  <邮箱>  linux  -
BackendState: Running，Online: true
```

> 注意：初次运行时，到 `controlplane.tailscale.com` 的请求可能先超时几次
> （批准提示尚未落定），随后成功并输出登录链接，这是正常的，等它即可，
> 不要重复执行 `tailscale up`。

### 4.3 Tailscale 首次连接的审批提示

首次启动时，Tailscale 会联系全球各地的中继服务器选最优线路，
每个新地址都会弹一次审批。数量有限，逐个批准，批完就不再出现。

### 4.4 （可选）ACL 策略：默认不要动

仓库提供一份可选的只收（receive-only）ACL 策略
（`files/tailscale/policy.jsonc`，tag:muse）。**除非你明确要求，
否则不要应用它，不要覆盖已有 Tailnet 策略。** 默认做法是让 VM
作为普通 Tailnet 成员加入，不加 `--advertise-tags`。

## 5. 分步实施（对应 INSTALL.md §0–§11）

### 5.0 §0：先读文档再动手

Muse 在运行任何命令前，先完整读完 `README.md`、`SYSTEM_PROMPT.md`、
`INSTALL.md`。如果读文档或脚本的结果触发了安全检查，Muse 应当：

1. 停下，不运行任何东西；
2. 用 plain words 说明是哪一步没通过检查；
3. 给你两个 plain-text 选项：`1 继续` / `2 先不装`；
4. 只有收到你的 `1` 才会继续。

本次实测中这个检查共触发三次（文档、安装脚本、Bridge hook 提示词），
三次都是用户确认 `1` 后才继续。

### 5.1 Phase B：安装文件

```sh
cd /home/hatch/muse-vm-hacks
sh files/scripts/install-files.sh   # 应返回 exit 0
```

安装结果（都在 `/home/hatch` 下，持久化）：

- `/home/hatch/bin/nixwrap` 及 symlink farm（`nix`、`nix-env`、
  `nix-shell`、`tailscale`、`tailscaled` 等都指向 `nixwrap`）
- `/home/hatch/scripts/{bootstrap-tailscale.sh,install-files.sh,doctor.sh}`
- `/home/hatch/hooks/scripts/nix-boot-trigger.sh`
- `/home/hatch/workspace/nix/{setup-nix.sh,ignored-acls.txt,xattr-retry.c}`
- 标志文件 `/home/hatch/.config/vm-tailscale/autostart`
- 清单 `/home/hatch/.config/vm-setup/manifest`

**先审计再运行**：这些脚本做的事情应与 INSTALL.md 一致——把文件拷到
home 下、从 nixos.org 取 Nix 安装器、按 sha256 pin 从本仓库的 GitHub
release 取 `xattr-retry.so`、挂载 nix 磁盘镜像、启动 tailscaled。
若发现不一致，先停下问用户。

### 5.2 Phase C：装 Nix（注意版本漂移）

```sh
/home/hatch/workspace/nix/setup-nix.sh
```

**版本漂移（本次实测命中）：**

- 上游 nixos.org 的安装器现在提供 **Nix 2.35.6**，不是指南写的 2.35.2。
  `setup-nix.sh` 里写死的 `*-nix-2.35.2` 路径匹配会失败（exit 1），
  但 Nix 本身其实已装好，`nix --version` 可用。
- 处理：不要改脚本硬凑版本号。依次执行：

```sh
/home/hatch/bin/nix-channel --update        # 应 exit 0
/home/hatch/bin/nix-env -iA nixpkgs.tailscale  # 约 40 秒，装 Tailscale 1.104.6
```

- 验证：`/home/hatch/bin/nix --version` 应输出 `nix (Nix) 2.35.6`；
  `/home/hatch/bin/nix-channel --list` 应有 `nixpkgs-unstable`；
  `/home/hatch/bin/xattr-retry.so` 存在，
  sha256 应为 `19038c96d55f08eda8388b41a68be4c9bcdc3554fa7fc6e1fe5985c5189852e2`
  （与仓库 pin 一致）。
- **关键约束**：`/nix` 不会在普通 shell 会话里自动挂载，挂载是按会话的，
  由 `nixwrap` 负责。**永远通过 `/home/hatch/bin/nix*` 调用 Nix 工具**。

### 5.3 Phase D：开机恢复 hook

按 INSTALL.md §6 精确注册 hook `nix-boot-trigger`：

- 脚本：`~/hooks/scripts/nix-boot-trigger.sh`
- 提示词：`None.`
- 轮询：5 秒；超时：240 秒；投递：main chat

验证：

1. 干跑（dry-run）两个分支都要静默通过：
   - marker 存在 → 静默（实测约 29ms）；
   - marker 不存在 → 静默“would bootstrap”（实测约 39ms）。
2. 启用后，首次 live 轮询应完成一次 bootstrap（实测 104ms，
   输出 Nix 2.35.6），随后的轮询静默（约 25ms），
   marker `/tmp/.nix-bootstrapped` 存在。

### 5.4 Phase E：启动 Tailscale

daemons 通常已被 `nixwrap` 的 bootstrap 拉起（若手动再启动会报
`address already in use`，无害，说明它已在运行）。

```sh
# 以分离方式运行，日志记到 ~/.local/state/tailscale/up.log
tailscale up --ssh --hostname=muse-vm
```

- 输出登录链接后，按 §4.2 在浏览器里批准。
- 验证：`up.log` 末尾应为 `Success.`，AuthURL 为空；`tailscale status`
  应显示节点 `muse-vm`、Online、BackendState Running、Health 为空。
- 状态持久化在 `/home/hatch/.local/state/tailscale/`，
  autostart + boot hook 保证重启后自动重连，无需重新登录。

### 5.5 SSH 验证

从同一 Tailnet 的其他机器验证（不需要密码/密钥，Tailscale SSH
按 Tailnet 身份认证）：

```sh
ssh root@muse-vm                 # 或 ssh root@<Tailnet IP>
```

实测：从 VPS（同一 Tailnet）和另一台 Tailnet 机器 SSH 登录均成功，
身份均为 Tailnet 账号邮箱，目标用户为 `root`。

Windows 用法：在已加入同一 Tailnet 的 Windows 上，
确认 Tailscale 已连接并显示 `muse-vm` 在线，然后在终端（PowerShell
或命令提示符）执行：

```powershell
ssh root@muse-vm
ssh root@<Tailnet IP>   # 备用：节点名解析不到时用 IP
```

若 Windows 没有直连路径：先 SSH 到同一 Tailnet 的中转机，
再从中转机执行 `ssh root@muse-vm`。

> 注意：内置的平台版 Tailscale（`/opt/hatch/bin/tailscale`）是最小客户端，
> 不能接受入站连接，保持 “Not connected” 不用管；以
> `/home/hatch/bin/tailscale` 为准。

### 5.6 健康检查 doctor.sh

```sh
sh /home/hatch/scripts/doctor.sh
```

实测结果（核心安装后）：**4/5 PASS**；部署 Bridge 后：**6/7 PASS**。

| 检查项 | 结果 | 说明 |
| --- | --- | --- |
| layout | PASS | |
| shim | PASS | |
| profile | PASS | |
| tailscale | PASS | |
| profile api-bridge | PASS | 部署 Bridge 后新增 |
| bridge | PASS | 部署 Bridge 后新增 |
| **nix** | **FAIL** | **假失败，见下** |

**nix FAIL 是过期检查，不是故障**：`doctor.sh` 把版本号写死成
2.35.2，而实际装的是 2.35.6。只比对版本字符串，不影响功能。
**不要改 doctor.sh 去强行通过**，那会让它与 manifest 脱节。
判断标准是功能：`nix --version` 可用、Tailscale 在线、Bridge 健康。

### 5.7 §11：部署 API Bridge（可选模块）

在核心安装完成、用户明确确认后部署（本次是第三次安全确认后才动手）。

**步骤：**

1. 安装包（路径按你的仓库检出位置调整）：

```sh
/home/hatch/bin/nix-env -f files/api-bridge/api-bridge.nix -i
# 应得到 muse-api-bridge 0.3.1
```

2. 安装 Bridge 文件：

```sh
sh files/scripts/install-files.sh --with-bridge
```

这会装好 `/home/hatch/scripts/bootstrap-api-bridge.sh` 和
`/home/hatch/hooks/scripts/api-bridge.sh`，并创建标志
`/home/hatch/.config/vm-api-bridge/autostart`，更新 manifest。

3. **修复一个上游缺口（本次实测命中）**：
   `bootstrap-api-bridge.sh` 会把 `~/bridge/bin/*` 链接到
   `~/bridge/src/` 下的工具副本，但仓库里没有任何脚本会创建
   `~/bridge/src/`。处理：手动把 `bridge-respond`、`bridge-key`、
   `bridge-session` 从 `files/api-bridge/` 拷到
   `/home/hatch/bridge/src/`（`chmod +x`），再跑 bootstrap，
   `~/bridge/bin/` 下的符号链接（bridge-respond、bridge-key、
   bridge-reply、bridge-next、bridge-close）就正常了。

4. 建一个专用的 side chat（例如命名为 “API Bridge”），记下它的
   chat_id——Bridge 每次请求都会在这个对话里跑一个 agent 回合。
5. 注册 hook `api-bridge`：脚本 `~/hooks/scripts/api-bridge.sh`，
   轮询 5 秒，超时 30 秒，投递到上一步的 side chat，
   提示词用 INSTALL.md §11 的 worker 提示词原文。
   干跑应静默，显示 “no pending bridge jobs”（空队列是正常的）。
   然后启用 hook。
6. 启动服务器（一次 nix 命令，走 nixwrap bootstrap），
   日志在 `~/.local/state/api-bridge/{server.log,server.pid,bootstrap.log}`。
   默认监听 `127.0.0.1:8080`。

**验证（必须逐项通过）：**

```sh
curl http://127.0.0.1:8080/health
# {"status": "ok"}

# 带初始密钥请求一次聊天（密钥见 §6.1，模型固定为 muse）
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Authorization: Bearer <初始密钥>" \
  -H "Content-Type: application/json" \
  -d '{"model":"muse","messages":[{"role":"user","content":"Reply with exactly the word BridgeOK"}]}'
# 期望 choices[0].message.content 为 BridgeOK（实测约 7.7 秒）
```

**API 说明：**

| 端点 | 说明 |
| --- | --- |
| `GET /health` | 无需认证，返回 `{"status": "ok"}` |
| `GET /v1/models` | Bearer 认证，返回唯一模型 `muse` |
| `POST /v1/chat/completions` | Bearer 认证，OpenAI 兼容的聊天接口 |
| `/session/*` | 工具会话内部路由（客户端按需使用） |

- **没有图片、视频、文件下载端点**：Bridge 只回文字。让 agent
  生成的图片/视频会落在 VM 磁盘上，须另行取回（例如
  `scp root@muse-vm:<路径> .`）。
- 每次请求都是 API Bridge side chat 里的一次正常 agent 回合，
  花正常 Muse 用量；请求一次只处理一个；服务器最长等 600 秒，
  超时返回 504；响应里的 usage 字段恒为 0。
- `stream: true` 时以单个 chunk 返回完整回复再加 `[DONE]`。
- 带 tools 的请求可能变成 live session，工具在**客户端**那侧执行，
  绝不在本 VM 上执行；同时最多一个 live session。

**密钥管理：**

```sh
bridge-key add <名字>     # 打印一次，原始值不存任何地方
bridge-key list
bridge-key revoke <名字>  # 修改后服务端无需重启即可生效
```

- 初始密钥（名字 `initial`）只存在 `/home/hatch/bridge/token`
  （权限 600），哈希形态的 keyring 在
  `/home/hatch/bridge/keys.json`。
- **密钥值不要记到任何文档、笔记或聊天记录里**；
  只记位置。丢了就 revoke 后重新生成。
- 客户端配置：Base URL 用 Tailnet 地址
  `http://muse-vm:8080/v1` 或 `http://<Tailnet IP>:8080/v1`，
  模型 id 为 `muse`，API Key 填初始密钥。

## 6. 验证清单（部署完成后的逐项核对）

- [ ] `nix --version` 输出 2.35.6（或更新），`nix-channel --list` 有
      nixpkgs-unstable
- [ ] `tailscale status`：节点 `muse-vm`、Online、BackendState Running
- [ ] 从另一台 Tailnet 设备 `ssh root@muse-vm` 能登录
- [ ] hook `nix-boot-trigger` 已启用，marker `/tmp/.nix-bootstrapped` 存在
- [ ] `doctor.sh`：除 nix 版本字符串外全部 PASS
- [ ] Bridge：`GET /health` 为 `{"status": "ok"}`，
      `POST /v1/chat/completions` 返回 BridgeOK
- [ ] `~/bridge/token` 权限 600，`~/bridge/keys.json` 只有哈希

## 7. 部署后使用：Postman 测试

仓库自带一个 Postman 集合：[`postman/muse-bridge.postman_collection.json`](../postman/muse-bridge.postman_collection.json)。
导入前先把集合变量改成你自己的值：

| 变量 | 填什么 |
| --- | --- |
| `baseUrl` | `http://muse-vm:8080`（节点名解析不到时改用 `http://<Tailnet IP>:8080`） |
| `bridge_token` | `~/bridge/token` 里的初始密钥（集合里故意留空，**不要**把真密钥写进文件再分享） |

集合里有 4 个请求：Health check、List models、Chat completions、
Chat completions (stream)。要点：

- Postman 必须跑在**已加入同一 Tailnet**的设备上，否则连不上。
- Chat 请求把 Postman 的请求超时设为 600000 ms（服务端最长等 600 秒；
  简单回复几秒，复杂的几分钟）。
- 应答在 `choices[0].message.content` 里；usage 恒为 0。
- 常见失败：401 = `bridge_token` 没填或填错；连不上/超时 =
  Postman 那台设备不在 Tailnet 里，或 `muse-vm` 名字解析不到（改用 IP）。

## 8. 运维

| 事项 | 做法 |
| --- | --- |
| 看 Bridge 日志 | `~/.local/state/api-bridge/server.log` |
| 看 Tailscale 日志 | `~/.local/state/tailscale/`（`tailscaled.log`、`up.log`） |
| 重启 Bridge | 停掉服务器进程后按 §5.7 第 6 步重新启动 |
| 重启 Tailscale | `tailscale up` 会复用已有登录，通常无需重新批准 |
| VM 重启/替换后 | home 目录下的 autostart 标志 + boot hook 会自动恢复 Nix 与 Tailscale，无需重新登录；随后检查 §6 清单 |
| 新增密钥 | `bridge-key add <名字>`，马上可用 |
| 吊销密钥 | `bridge-key revoke <名字>`，立即生效 |

## 9. 故障排查

| 现象 | 原因与处理 |
| --- | --- |
| `setup-nix.sh` 报 exit 1 | 见 §5.2：Nix 2.35.6 已装好，只是脚本里的 2.35.2 路径匹配失败；按 §5.2 跑 `nix-channel --update` 与 `nix-env -iA nixpkgs.tailscale` |
| `doctor.sh` 的 nix 一项 FAIL | 见 §5.6：过期版本字符串检查，功能正常；不要改脚本 |
| `tailscale up` 初期超时 | 见 §4.2：控制面请求先超时几次是正常的，等它出登录链接 |
| `tailscale up` 报 `address already in use` | daemon 已在运行，无害 |
| GUI 模式崩溃/无界面 | 本仓库没有 GUI，所有操作都在命令行完成；不要找图形界面 |
| Bridge hook 干跑不静默 | 检查队列是否为空、提示词是否为 §11 原文 |
| Bridge 请求 401 | 密钥不对或没带 Bearer 头；用 `bridge-key list` 核对 |
| Bridge 请求 504 | 单次请求超过 600 秒，属正常上限；拆小请求或接受更长等待 |
| 重复的 bot/会话 | 不要手动再起一个 Bridge 服务器或 hook；只应有一个实例在跑 |
| `pkill -f <模式>` 把自己的 SSH 会话杀掉 | 模式串会匹配到自己的命令；**永远按精确 PID kill** |

## 10. 安全注意事项

1. **保留系统原有的安全审批机制**：安装不得绕过或移除它。
   凡遇安全检查，先停下、向用户说明、等用户确认。
2. **不要覆盖已有 Tailnet 策略**：除非用户明确要求应用仓库的
   ACL 策略，否则不应用 `files/tailscale/policy.jsonc`。
3. **不要修改其他生产设备**：只动这台 Muse VM。
4. **密钥只放服务端**：Bridge 密钥只在 `/home/hatch/bridge/token`
   （600）与 `keys.json`（哈希）里；不要写进文档、笔记、聊天记录、
   集合文件或公开仓库。
5. **不要公开真实地址与账号**：公开文档里一律用占位符
   （`<节点名>`、`<Tailnet IP>`、`<邮箱>`、`<域名>`）；涉及其他机器、
   其他服务账号、token 的真实值不要写进仓库。
6. **Bridge 只在 Tailnet 内可达**：它监听 `127.0.0.1:8080`，
   不要把它直接暴露到公网；需要域名访问时，应在受信的机器上
   另起代理/隧道并自行加访问密码。

## 11. 延伸：域名 Web 应用（可选扩展）

本仓库只负责 VM 本身的 Nix + Tailscale + API Bridge。
如果你还想像本次实测那样，用浏览器通过域名直接使用这个 agent，
可以在**另一台你自己的服务器**上搭一个很薄的 Web 应用：

```
浏览器 ──HTTPS──▶ 你的服务器（域名 + 隧道）
                      ├─ 静态聊天页面（聊天 / 生成图片 / 生成视频 / 图生视频）
                      ├─ 服务端代理：把 /api/chat 转发给 Tailnet 内的 Bridge
                      │   （Bridge 密钥只放服务端，绝不发给浏览器）
                      └─ 媒体取回：agent 在 VM 上生成文件后，
                          经 Tailscale SSH（scp）取回，再以 /media/ 提供下载
```

要点：

- 应用服务只监听本机回环地址，对外走 Cloudflare Tunnel 之类的隧道；
  隧道配置里谁读哪个文件要先核对（实测教训：`cloudflared` 的
  systemd 服务读 `/etc/cloudflared/config.yml`，不是
  `/root/.cloudflared/config.yml`，两处要保持同步）。
- **务必加访问密码**（如 HTTP Basic Auth），否则知道域名的人都能
  用你的 agent、花你的用量。
- 视频生成可能超过 100 秒，隧道的 524 超时会先断开浏览器连接：
  生成请求要做成异步任务（立即返回任务号，前端轮询取结果），
  不能让浏览器干等一个长请求。

这份扩展不在本仓库代码里，实现时按你自己的服务器环境写，
敏感值同样只放服务端。

## 附录：关键路径

| 路径 | 内容 |
| --- | --- |
| `/home/hatch/muse-vm-hacks` | 本仓库检出位置 |
| `/home/hatch/bin/nixwrap` | Nix/Tailscale 的调用入口（所有工具走它） |
| `/home/hatch/scripts/doctor.sh` | 健康检查脚本 |
| `/home/hatch/hooks/scripts/nix-boot-trigger.sh` | 开机恢复 hook 脚本 |
| `/home/hatch/hooks/scripts/api-bridge.sh` | Bridge hook 脚本 |
| `/home/hatch/.local/state/tailscale/` | Tailscale 状态与日志 |
| `/home/hatch/bridge/token` | Bridge 初始密钥（600，只有位置可记） |
| `/home/hatch/bridge/keys.json` | Bridge 密钥哈希 keyring |
| `/home/hatch/bridge/src/` | Bridge 工具副本（bootstrap 需要，见 §5.7） |
| `~/.local/state/api-bridge/` | Bridge 服务端日志 |
| `files/api-bridge/server.py` | Bridge 服务器源码（端点定义） |
| `postman/muse-bridge.postman_collection.json` | Postman 测试集合 |
