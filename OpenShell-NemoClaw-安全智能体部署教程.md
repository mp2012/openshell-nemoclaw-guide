# 用 OpenShell + NemoClaw 部署安全智能体 · 官方教程

> 本文基于 NVIDIA 官方文档整理，所有安装/策略命令均来自官方原文（逐字摘录），并如实标注来源与少数未能逐字抓取之处。
> 整理日期：2026-10-29

## 文档来源

- OpenShell 安装：`docs.nvidia.com/openshell/latest/about/installation`（v0.1.6）
- OpenShell 上手（Run Your First Agent）：`docs.nvidia.com/openshell/dev/get-started/quickstart`
- NemoClaw 上手（OpenClaw）：`docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/get-started/quickstart`
- 示例 provider 档案：`github.com/NVIDIA/OpenShell/main/providers/openrouter.yaml`
- 策略 Schema：`github.com/NVIDIA/OpenShell/main/docs/how-it-works/policies/schema.mdx`
- 网络规则：`github.com/NVIDIA/OpenShell/main/docs/how-it-works/policies/network-rules.mdx`
- 策略管理：`github.com/NVIDIA/OpenShell/main/docs/how-it-works/policies/manage-policies.mdx`

---

## 0. 架构与两条路径

- **OpenShell** = 底层安全运行时：Gateway（控制面，Linux 监听 `127.0.0.1:17670`）+ Supervisor + Sandbox + Policy Prover。内核三件套 **Landlock（文件）+ seccomp（syscall）+ 网络命名空间**；凭据以占位符注入，只对授权端点换入真值。
- **NemoClaw** = 在 OpenShell 沙盒内安全运行 agent 的官方蓝图，自带 OpenShell，支持 **OpenClaw（默认）/ Hermes / LangChain Deep Agents**。

**路径 A（OpenShell 原生）**：自选镜像+策略，跑任意 agent（如 OpenCode），更灵活。**仅适用于"单独管理 OpenShell"的场景。**
**路径 B（NemoClaw）**：官方蓝图一键部署 OpenClaw 等，更简单。**本项目（OpenShell + NemoClaw 并用）以路径 B 为主路径。**

> ⚠️ **官方约束（NemoClaw 托管环境）**：NemoClaw README 明确规定，在 NemoClaw 托管的环境下，**禁止直接运行** `openshell self-update`、`npm update -g openshell`、`openshell gateway start --recreate` 或 `openshell sandbox create`。需要创建/重建 OpenShell 网关或沙箱时，一律用 `nemoclaw onboard`；除非你打算单独管理 OpenShell，并随后重跑 `nemoclaw onboard`。因此：本项目的沙箱创建、网关管理请走 `nemoclaw` CLI，不要直接敲 `openshell sandbox create`。

---

## 路径 A：OpenShell 原生部署（逐字命令）

> 注：以下为 OpenShell 原生用法。若你已处于 NemoClaw 托管环境，请勿直接执行其中的 `openshell sandbox create` 等命令（见上方官方约束），应改走 `nemoclaw` CLI。

### A1 安装 OpenShell

官方**不通过 pip**。任选其一：

```bash
# 方法1：官方一键脚本（Linux/macOS）
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh

# 方法2：Snap（注意 Docker 不能用 snap 版）
sudo snap install openshell

# 方法3：Kubernetes（Helm）
helm upgrade --install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version 0.1.0-pre.3
```

### A2 验证网关

```bash
openshell status
systemctl --user status openshell-gateway
# 登出后保持常驻（可选）：sudo loginctl enable-linger $USER
```

### A3 配置 provider（凭证 + 端点策略）

provider 档案定义"哪个程序、发往哪个端点、用哪个凭据"，是最小权限控制：

```bash
openshell profile import \
  --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml

export OPENROUTER_API_KEY=<你的key>     # 真凭据只存凭据库，仅发往 openrouter.ai
openshell provider create \
  --name openrouter \
  --type openrouter \
  --from-existing
```

> 官方 `openrouter.yaml` 注释说明：`binaries` 字段决定哪些进程（如 `/usr/local/bin/opencode`）可触达端点；`OPENROUTER_API_KEY` 作为 bearer token **只发往 openrouter.ai，绝不到别处**；端点访问由 L7 策略强制。校验：`openshell provider profile lint -f providers/openrouter.yaml`。

### A4 创建沙箱并跑 agent

```bash
openshell sandbox create \
  --name my-agent \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free
```

### A5 运行时按需授权

agent 运行中申请新访问时，**主机侧 Gateway 审批**（不自动放行）：

```bash
openshell rule get          # 查看待审请求
openshell rule approve <id> # 批准
openshell rule reject  <id> # 拒绝
openshell rule approve-all  # 批量批准
```

---

## 路径 B：NemoClaw 部署 OpenClaw（逐字命令）

### B1 前置

- Docker Engine 就绪（`docker info`）。**Node.js 22.16+**（可视化凭据表单需 **22.19+**）；本机当前未安装，安装器可自动补齐，或用非交互式环境变量传 key（如 `NVIDIA_INFERENCE_API_KEY`）绕过凭据表单。

### B2 安装（非交互式，适合无 TTY 环境）

```bash
export NEMOCLAW_DISABLE_TELEMETRY=1
export NEMOCLAW_NON_INTERACTIVE=1
export NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1
export NEMOCLAW_AGENT=openclaw
export NEMOCLAW_SANDBOX_NAME=my-assistant

# 推理提供商二选一（官方支持列表）：
export NEMOCLAW_PROVIDER=build            # NVIDIA Endpoints，需下方 key
export NVIDIA_INFERENCE_API_KEY=<你的key>
# export NEMOCLAW_PROVIDER=vllm           # 已有本地 vLLM 时（数据不出机，推荐用于安全场景）

curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

官方支持的 `NEMOCLAW_PROVIDER`：`build`(NVIDIA Endpoints) / `openrouter` / `openai` / `anthropic` / `gemini` / `ollama`(本地) / `vllm`(已有 vLLM)。

### B3 启动与验证

```bash
nemoclaw my-assistant status            # 等到打印 "NemoClaw is ready"
nemoclaw my-assistant dashboard-url --quiet
nemoclaw launch my-assistant            # 启动 TUI
```

Dashboard：http://127.0.0.1:18789/（URL 含网关令牌）。

---

## 安全策略（Policy-as-Code）官方机制

- **Provider 档案**：凭据只注入授权端点，`binaries` 限定可触达的进程（最小权限）。
- **运行时审批**：agent 提议新访问，主机用 `openshell rule approve/reject` 裁决。
- **Policy Prover**：对策略变更做形式化校验，扩大风险访问需人工复核。
- 完整策略字段见下方「策略 YAML 字段完全版」。
- **NemoClaw 默认策略**：`nemoclaw-blueprint/policies/openclaw-sandbox.yaml`，**默认拒绝（deny-by-default）**——除显式列出的端点外一切出站都被拦，未列出端点会在 OpenShell TUI 中实时请求你批准（`openshell term`），一次性批准仅对当前会话有效。仓库提供若干预设策略（presets），但具体预设名称（如 PyPI/Docker Hub/Slack/Jira）**以仓库 `policies/` 目录或官方 Network Policies 页为准**，勿凭记忆套用。

## 遥测与隐私

```bash
export OPENSHELL_TELEMETRY_ENABLED=false   # OpenShell 默认发送匿名运营计数
export NEMOCLAW_DISABLE_TELEMETRY=1        # NemoClaw 同上
```

## 卸载（官方给出）

```bash
# Debian/Ubuntu
systemctl --user disable --now openshell-gateway
sudo apt remove openshell
rm -rf "${XDG_STATE_HOME:-$HOME/.local/state}/openshell"
# Snap
systemctl --user disable --now snap.openshell.user-gateway.service
sudo snap remove --purge openshell
```

## 执行声明

以上 `curl | bash/sh` 均为**官方安装方式**，会执行远程脚本并起后台服务/容器。落地执行前应 review 脚本内容，并确认环境已隔离/可信。

---

# 策略 YAML 字段完全版（OpenShell Policy Schema）

## 1. 顶层字段

策略文件为**最大 4 MiB** 的 YAML，未知字段或重复键会被**拒绝**。

| 字段 | 类型 | 必填 | 生效时机 | 说明 |
|---|---|---|---|---|
| `version` | integer | **是** | — | 架构版本，必须为 `1` |
| `filesystem_policy` | object | 否 | Startup | 工作负载可读写路径 |
| `landlock` | object | 否 | Startup | Landlock 执行行为 |
| `process` | object | 否 | Startup | 运行用户/组 |
| `network_policies` | map | 否 | **Live**（热加载） | 哪些二进制可访问哪些端点 |
| `network_middlewares` | map | 否 | Live | 对允许流量应用的中间件 |

> `filesystem_policy`/`landlock`/`process` **仅在沙箱启动时生效**，改了需重建沙箱；`network_policies` 可运行时热更新。

## 2. `filesystem_policy`

| 字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `include_workdir` | bool | 省略本块时 `true`；显式写出时 `false` | 是否包含工作目录 |
| `read_only` | list[str] | `[]` | 允许**读取**的绝对路径 |
| `read_write` | list[str] | `[]` | 允许**读写**的绝对路径 |

约束：路径必须绝对、不含 `..`、≤4096 字节；`read_write` 不能包含 `/`；最多 256 条。

```yaml
filesystem_policy:
  include_workdir: true
  read_only: [/usr, /lib, /etc]
  read_write: [/tmp]
```

## 3. `process`

| 字段 | 类型 | 说明 |
|---|---|---|
| `run_as_user` | string | `sandbox` 或数字 UID（1–4294967294，**拒绝 root**） |
| `run_as_group` | string | 同上，对应 GID |

```yaml
process:
  run_as_user: "1500"
  run_as_group: "1500"
```

## 4. `network_policies`（重点）

结构：命名规则的 map，**键不能以 `_provider_` 开头**。每条规则含：

- `name`（可选，日志显示名，默认用键名）
- `endpoints`（list，可选）
- `binaries`（list，可选；**空列表不匹配任何二进制**）

### 4.1 Endpoint 对象字段（verbatim）

**目标（Destination）**

- `host`（string，条件必填）主机名/IP/通配符，如 `*.internal.example`
- `port` / `ports`（int / list[int]，二选一）
- `path`（string，可选）同主机同端口下选路 glob
- `allowed_ips`（list[str]，可选）IP/CIDR 限制

**检查（Inspection）**

- `protocol`：`rest` / `websocket` / `graphql` / `mcp` / `json-rpc` / `tcp`（省略=仅连接级）
- `tls`：`skip`（不终止 TLS）/ `Automatic`（默认）
- `enforcement`：`enforce`（阻断）/ `audit`（**默认，仅记录**）
- `access`：`read-only` / `read-write` / `full`
- `rules` / `deny_rules`（list）：允许/拒绝规则，**拒绝优先于允许**
- `allow_encoded_slash`（bool，默认 `false`）

**凭据（Credential）**

- `credential_binding.provider`（string）
- `request_body_credential_rewrite` / `websocket_credential_rewrite`（bool）
- `allow_uninspected_credentials`（bool）
- `credential_signing`：`sigv4` / `sigv4:body` / `sigv4:no_body`
- `signing_service` / `signing_region`

**协议选项**：`graphql_*`、`mcp.versions`(默认 `["2025-11-25"]`)、`mcp.max_body_bytes`(65536)、`mcp.strict_tool_names`(true)、`json_rpc.max_body_bytes`(65536) 等。

### 4.2 Binary 对象

- `path`（**必填**）可执行文件真实路径或 glob，如 `/usr/bin/curl`

### 4.3 `rules` / `deny_rules` 内字段

- `method`：`GET`/`POST`/`*` 或协议特定（`WEBSOCKET_TEXT`、`initialize`、`tools/call` 等）
- `path`：URI glob
- `operation_type`（GraphQL）：`query` / `mutation` + `fields`
- `tool`（MCP）：工具名

## 5. 官方完整示例（schema.mdx 原文）

```yaml
version: 1
filesystem_policy:
  include_workdir: true
  read_only: [/usr, /lib, /etc]
  read_write: [/tmp]
network_policies:
  github_rest_api:
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        enforcement: enforce
        access: read-only
    binaries:
      - path: /usr/bin/gh
  npm_registry:
    endpoints:
      - host: registry.npmjs.org
        port: 443
        protocol: rest
        enforcement: enforce
        access: read-only
        allow_encoded_slash: true
    binaries:
      - path: /usr/bin/node
```

## 6. 网络规则常用写法（network-rules.mdx 原文片段）

```yaml
# 基础（仅 host/port，无请求检查）
network_policies:
  example_api:
    endpoints:
      - host: api.example.com
        port: 443
      - host: uploads.example.com
        port: 443
    binaries:
      - path: /usr/bin/curl
      - path: /usr/bin/wget

# 允许/拒绝方法（deny 优先）
github_repository_api:
  endpoints:
    - host: api.github.com
      port: 443
      protocol: rest
      enforcement: enforce
      rules:
        - allow:
            method: "*"
            path: "/repos/<owner>/<repo>/**"
      deny_rules:
        - method: "*"
          path: "/repos/<owner>/<repo>/hooks/**"
  binaries:
    - path: /usr/bin/gh

# MCP 工具级管控
mcp_server:
  endpoints:
    - host: mcp.example.com
      port: 443
      path: /mcp
      protocol: mcp
      enforcement: enforce
      rules:
        - allow: { method: tools/call, tool: read_status }
      deny_rules:
        - method: tools/call
          tool: delete_resource
  binaries:
    - path: /usr/bin/python3.12

# 原生 TCP（无请求检查）
postgres:
  endpoints:
    - host: db.internal.example
      port: 5432
      protocol: tcp
  binaries:
    - path: /usr/lib/postgresql/*/bin/psql
```

## 7. 策略管理命令（manage-policies.mdx 原文）

```bash
# 建沙箱时套用策略（fs/landlock/process 仅启动时生效）
openshell sandbox create --name my-sandbox --policy ./policy.yaml
export OPENSHELL_SANDBOX_POLICY="$PWD/policy.yaml"   # 之后创建免显式传参

# 查看
openshell policy get my-sandbox --base        # 用户设定策略
openshell policy get my-sandbox --full        # 含 provider 附加规则
openshell sandbox get my-sandbox --policy-only > effective-policy.yaml
openshell policy list my-sandbox              # 各次修订及加载状态

# 运行时热更新网络规则（仅 network_policies）
openshell policy update my-sandbox --rule-name github_readonly \
  --binary /usr/bin/curl \
  --add-endpoint api.github.com:443:read-only:rest:enforce --wait
openshell policy update my-sandbox --rule-name github_readonly \
  --binary /usr/bin/curl \
  --add-allow 'api.github.com:443:POST:/repos/*/*/issues' --wait
openshell policy update my-sandbox --remove-rule github_readonly --wait
# 预览不应用：把 --wait 换成 --dry-run

# 整体替换（中间件/GraphQL/凭据绑定等 policy update 覆盖不到的变更）
openshell policy get my-sandbox --base        # 保存为 policy.yaml 并编辑
openshell policy set my-sandbox --policy policy.yaml --wait

# 回滚
openshell policy list my-sandbox
openshell policy get my-sandbox --rev 2 --base # 取早期修订
openshell policy set my-sandbox --policy previous-base.yaml --wait

# 全局策略（需管理员角色）
openshell policy get --global --full
openshell policy set --global --policy ./global-policy.yaml
openshell policy list --global
openshell policy delete --global
```

## 8. 关键行为

- **匹配是非有序防火墙**：多条规则可同时命中，**拒绝优先于允许**（与文件顺序无关）。
- **二进制须写真实路径**（kernel 报告的路径）；脚本类（pip/npm）要列解释器（如 `/usr/bin/python3.12`）。
- **`enforcement: enforce` 才阻断**；默认 `audit` 只记录不阻断——生产收紧时务必显式设 `enforce`。
- **`filesystem_policy`/`process` 改了要重建沙箱**：`openshell sandbox delete` 后 `create --policy`。

> 说明：`landlock` 与 `network_middlewares` 的嵌套字段本文未逐字展开（schema 文档中它们为顶层字段，详细嵌套在同一 `schema.mdx` 内）。

---

# 部署后使用实录（本机实测，含截图）

> 以下为本机（A100-Ubuntu）2026-10-29 实际部署后的使用流程与排障记录。截图存于同目录 `screenshots/`。

## 1. 访问 Dashboard（SSH 隧道）

Dashboard 只监听宿主机回环 `127.0.0.1:18789`，远程访问需在**本地笔记本**做端口转发：

```bash
ssh -L 18789:127.0.0.1:18789 azureuser@<host>
```

然后本地浏览器打开 `http://localhost:18789/`。

## 2. 获取 Gateway 令牌（必须交互终端）

OpenClaw 网关令牌**只在交互 TTY 中输出**：`nemoclaw my-assistant dashboard-url --quiet` 内部为非交互调用会失败；在沙箱内非交互执行 `openclaw gateway auth-token --show` 也会被拒（"Refusing to print the Gateway token outside an interactive terminal"）。正确取法：

```bash
nemoclaw my-assistant connect            # 进入沙箱交互 shell
openclaw gateway auth-token --show       # 打印令牌（交互终端才允许）
exit                                      # 退回宿主机
```

![进入沙箱并用 openclaw gateway auth-token --show 取令牌（令牌已打码）](screenshots/03-connect-and-auth-token-redacted.png)

> 注：上图令牌已打码。明文令牌仅在交互终端打印，切勿将带令牌的截图外发/提交；一旦泄露立即轮换：沙箱内 `openclaw doctor --generate-gateway-token`。

> ⚠️ 上图含明文令牌：令牌等于 agent 的完全访问凭证，文档/截图对外分享前务必打码；已泄露时在沙箱内执行 `openclaw doctor --generate-gateway-token` 轮换。

## 3. 登录页填令牌

登录页 **WebSocket URL** 保持 `ws://localhost:18789`，令牌粘进 `Gateway 令牌` 框（密码框留空），点「连接」：

![OpenClaw 登录页（需 Gateway 令牌）](screenshots/02-dashboard-login.png)

## 4. 聊天验证与"首条消息超时"排障

登录后发消息验证 `qwen3.8:27b`。**第一条消息可能超时**——Ollama 的 17GB 模型在 keep-alive 到期后会卸载，下一条消息触发冷加载（约 30-60 秒），可能拖过请求窗口：

![首条消息超时（模型冷加载导致）](screenshots/05-chat-timeout-cold-load.png)

预热方法（宿主机执行，`keep_alive` 顺带续期 1 小时）：

```bash
curl -sS -m 600 http://127.0.0.1:11434/api/generate \
  -d '{"model":"qwen3.8:27b","prompt":"hi","stream":false,"keep_alive":"1h"}'
```

超时配置无需修改：沙箱内 `/sandbox/.openclaw/openclaw.json` 中 `agents.defaults.timeoutSeconds` 与 `agents.defaults.inference.timeoutSeconds` 均已是 600。

## 5. "连接被重置"排障（网关 ForwardTcp 资源耗尽）

症状：浏览器访问 `localhost:18789` 报"连接被重置"：

![浏览器连接被重置](screenshots/04-conn-reset-forward-down.png)

转发链路：`浏览器 → 127.0.0.1:18789（openshell forward 进程）→ 网关 8080（gRPC）→ 沙箱内 openclaw-gateway`。本次根因有两层：

1. 网关日志出现 `ForwardTcp ... rpc.grpc.status_code=8（RESOURCE_EXHAUSTED）`，转发被拒（可疑诱因：沙箱内 openclaw 进程异常空转占满连接）；
2. `systemctl --user restart nemoclaw-openshell-gateway` 重启网关后，onboard 时拉起的 18789 转发进程**不会自动恢复**，需手动重建。

修复命令（宿主机执行）：

```bash
# 1) 重启网关（用户级 systemd 单元；注意单元名不是 openshell-gateway.service）
systemctl --user restart nemoclaw-openshell-gateway

# 2) 重建 18789 转发（沿用 onboard 原参数）
nohup ~/.local/bin/openshell --gateway nemoclaw \
  --gateway-endpoint https://127.0.0.1:8080 --workspace default \
  forward service my-assistant --target-port 18789 \
  --target-host 127.0.0.1 --local 127.0.0.1:18789 \
  >/tmp/os-forward.log 2>&1 &

# 3) 验证（返回 200 即恢复）
curl -sS -m 12 -o /dev/null -w "%{http_code}\n" http://127.0.0.1:18789/
```

> 说明：`openshell forward` 仅是端口转发，不属于官方禁止的 `openshell sandbox create` / `gateway start --recreate` / `self-update` 禁区。但该 nohup 进程**主机重启后不会自启**，届时重跑第 2 步即可。另注意 8080 是 gRPC/mTLS 端点，`curl https://127.0.0.1:8080` 返回 000 属正常现象，不代表网关故障——判断网关是否健康要看 `journalctl --user -u nemoclaw-openshell-gateway` 与 18789 的 curl 结果。

## 6. 日常速查

| 目的 | 命令 |
|---|---|
| 进沙箱交互 shell | `nemoclaw my-assistant connect` |
| 取 Gateway 令牌 | 沙箱内 `openclaw gateway auth-token --show` |
| 状态 / 日志 | `nemoclaw my-assistant status` / `logs --follow` |
| 出站被拦（`CONNECT tunnel failed, response 403`） | `nemoclaw my-assistant logs --tail 50` 看是哪条规则；`openshell term` 实时批准 |
| 预热模型（防首条超时） | 上文带 `keep_alive` 的 curl |
| 重启网关 | `systemctl --user restart nemoclaw-openshell-gateway` |
| 重建 18789 转发 | 上文 nohup 命令（主机重启后需重跑） |
