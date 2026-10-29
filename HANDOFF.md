# Handoff 文档 · OpenShell + NemoClaw 安全智能体部署

> 生成日期：2026-10-29
> 目的：交接本项目已完成的调研、结论与产出物，便于后续接手者直接推进落地或继续补全文档。

---

## 1. 项目背景与目标

- **目标**：使用 NVIDIA **OpenShell**（智能体安全运行时）与 **NemoClaw**（在 OpenShell 沙盒内运行 agent 的官方蓝图）构建/部署"安全智能体"。
- **范围**：已完成**可行性分析 + 官方文档整理 + 本机实际部署**（NemoClaw + OpenShell 沙箱 + 本地 Ollama 推理已跑通并验证）。
- **澄清**：该栈的本意是"**安全地运行** agent"（沙盒+策略治理），而非"agent 本身做安全防御"。若要做安全运维类 agent，应在该栈之上开发/接入。

---

## 2. 已完成工作（清单）

1. **本机可行性分析**：采集 OS/CPU/内存/磁盘/GPU/容器/运行时/网络，判定算力与 OS 完全满足。
2. **网络约束排查**：发现 `github.com` 直连超时是 **IPv6 探测卡死**所致；强制 IPv4 / SSH 均可通；NGC(`nvcr.io`)、PyPI、raw.githubusercontent 均可达；本机还运行 squid 代理（非必需）。
3. **确认仓库**：`github.com/NVIDIA/OpenShell`（主仓库）与 `github.com/NVIDIA/NemoClaw` 均存在且可列远程；本机已配 GitHub SSH key（用户 `mp2012`）。
4. **官方文档调研**：读取两仓库 README、OpenShell 安装页与 Quickstart、NemoClaw OpenClaw Quickstart，并通过 GitHub API 定位并抓取策略文档（`schema.mdx` / `network-rules.mdx` / `manage-policies.mdx`）。
5. **产出教程文档**：整理出含逐字官方命令的安装部署教程 + 策略 YAML 字段完全版。

---

## 3. 关键结论（本机可行性）

| 维度 | 结论 |
|---|---|
| OS | Ubuntu 24.04.5 LTS x86_64 ✅（官方支持） |
| CPU/内存/磁盘 | AMD EPYC 7V13 ×24 核 / 216GB / 641GB 可用 ✅ |
| GPU | NVIDIA A100 80GB，Driver 595.91，CUDA 13.2 ✅（可做本地 vLLM 推理） |
| 容器 | Docker 29.1.3 + nvidia-container-runtime（GPU 透传就绪）✅ |
| 权限 | `azureuser` 在 `sudo` + `docker` 组 ✅ |
| pip | 未安装；apt 可装 `python3-pip`，但 **OpenShell 运行时不走 pip**（见坑#2） |
| GitHub | 直连超时（IPv6 问题），**SSH 已认证可用**，raw/ghcr/nvcr/PyPI 可达 ✅ |

**结论：完全可行**。唯一短板（GitHub）已通过 IPv4/SSH 解决，无需科学上网。

---

## 4. 踩坑与重要更正（务必阅）

- **坑#1 — GitHub 超时是 IPv6 导致**：`curl https://github.com` 默认先试 IPv6 卡死；用 `curl -4` 或 `git@github.com:...`（SSH）即可。
- **更正#1 — OpenShell 不走 pip**：早期误说 `pip install openshell`；官方安装页明确未采用 pip（PyPI 上的 `openshell` 非官方运行时）。正确通道：`install.sh` / Snap / `.deb` / Helm。
- **坑#2 — 策略文档路径**：`docs.nvidia.com/openshell/latest/policies` 与 `/dev/policies` 均 **404**；真实路径在仓库 `docs/how-it-works/policies/`（通过 GitHub API 列出 `docs/` 子目录找到）。
- **坑#3 — NemoClaw Express 仅在 DGX/WSL 触发**：本机为通用 Azure Ubuntu VM，不会弹 Express 提示，需用**非交互式环境变量**安装（已给出完整 env 清单）。
- **坑#4 — Node 版本**：NemoClaw 依赖 **Node.js 22.16+**（凭据表单需 22.19+）、npm 10+；本机当前未安装，安装器可自动补齐，或用环境变量传 key（如 `NVIDIA_INFERENCE_API_KEY`）绕过凭据表单。（注：其他资料曾写"Node 20+"，以官方 README 的 22.16+ 为准。）
- **坑#5 — NemoClaw 托管环境禁止直接动 OpenShell（官方原文）**：NemoClaw README 规定，在 NemoClaw 托管环境下，需创建/重建 OpenShell 网关或沙箱时一律用 `nemoclaw onboard`；**除非打算单独管理 OpenShell 并随后重跑 `nemoclaw onboard`，否则避免直接执行** `openshell self-update`、`npm update -g openshell`、`openshell gateway start --recreate`、`openshell sandbox create`。本项目（OpenShell + NemoClaw 并用）应以 `nemoclaw` CLI 为主入口。
- **坑#6 — Ollama 须绑回环，且会切断经 `host.docker.internal` 访问 Ollama 的其它容器**：NemoClaw onboard 的安全检查要求 Ollama 仅监听 `127.0.0.1`（否则其 protected loopback route 的令牌校验会被绕过）。本机原有 `ollama` 容器发布在 `0.0.0.0:11434`，且被 `aicomic-middleware-api-1` / `aicomic-middleware-worker-1` 经 `OLLAMA_URL=http://host.docker.internal:11434` 依赖。最终按用户决定**重建 `ollama` 容器为仅绑 `127.0.0.1:11434`**（middleware 失去 Ollama 访问，由用户确认可接受）。重建命令见第 6 节。

---

## 5. 已产出物

目录：`/home/azureuser/openshell-nemoclaw-guide/`

- `OpenShell-NemoClaw-安全智能体部署教程.md`（约 19KB，488 行）—— 安装部署教程 + 策略字段完全版 + **部署后使用实录（含 4 张截图：登录页 / 取令牌 / 连接重置排障 / 首条超时）**，官方命令部分逐字来自官方文档。
- `screenshots/`——部署使用过程截图（共 4 张；其中 `03-connect-and-auth-token.png` 因含明文 Gateway 令牌**未纳入公开仓库**，已在教程中脱敏省略）。
- `HANDOFF.md`（本文件）。

---

## 6. 实际部署记录（2026-10-29 已完成）

**结果：部署成功且已验证。** 沙箱 `my-assistant` Phase=Ready，OpenClaw v2026.9.2，模型 `qwen3.8:27b`（本地 Ollama），GPU(CUDA) 在沙箱内已验证，OpenShell 0.0.116，Dashboard `127.0.0.1:18789` 监听，网关 `8080` 监听。

### 6.1 关键步骤（已执行）
1. **脚本审查（只读）**：clone NVIDIA/NemoClaw 的 `lkg` ref 到 `/tmp/nemoclaw-lkg`，审查 `scripts/install.sh`（8000 行，Apache-2.0）。结论：安装 Node(经 nvm, 带 SHA 校验)、委派 `install-openshell.sh`、走 `nemoclaw onboard`（不直接 `openshell sandbox create`）、Ollama 经 `:11435` 代理；无越权操作。
2. **GitHub HTTPS 修复**：bootstrap 用 HTTPS `git clone github.com` 触发 `GnuTLS recv error -110`（IPv6/TLS）。用 `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0='url.git@github.com:.insteadOf' GIT_CONFIG_VALUE_0='https://github.com/'` 把 GitHub 重写为 SSH（**不改全局 git 配置**），安装成功。
3. **Ollama 回环化（坑#6）**：现有 `ollama` 容器发布 `0.0.0.0:11434` 被 NemoClaw 安全检查拒绝。按用户决定**重建为仅绑 `127.0.0.1:11434`**（middleware 失去 Ollama 由用户承担）。命令：
   ```bash
   docker stop ollama && docker rm ollama
   docker run -d --name ollama --restart unless-stopped --network nocobase-ai_default \
     -p 127.0.0.1:11434:11434 \
     -e OLLAMA_HOST=0.0.0.0:11434 -e OLLAMA_MAX_LOADED_MODELS=1 \
     -e OLLAMA_CONTEXT_LENGTH=32768 -e OLLAMA_KEEP_ALIVE=1h \
     -e NVIDIA_DRIVER_CAPABILITIES=compute,utility -e NVIDIA_VISIBLE_DEVICES=all \
     -e LD_LIBRARY_PATH=/usr/local/nvidia/lib:/usr/local/nvidia/lib64 \
     -v /home/azureuser/nocobase-ai/ollama:/root/.ollama \
     ollama/ollama:latest serve
   ```
4. **执行安装与续跑**：
   ```bash
   export OPENSHELL_TELEMETRY_ENABLED=false NEMOCLAW_DISABLE_TELEMETRY=1 NEMOCLAW_NON_INTERACTIVE=1 \
     NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1 NEMOCLAW_AGENT=openclaw NEMOCLAW_SANDBOX_NAME=my-assistant \
     NEMOCLAW_PROVIDER=ollama NEMOCLAW_OLLAMA_PORT=11434 NEMOCLAW_OLLAMA_PROXY_PORT=11435
   bash /tmp/nemoclaw.sh            # 首次：跑到 onboard 第3步因 Ollama 非回环失败
   nemoclaw onboard --resume       # Ollama 改回环后续跑，约 12–13 分钟（沙箱构建）
   ```
   > 注：`nemoclaw` CLI 由安装器装入 nvm(node 22)；**新 shell 需先** `export NVM_DIR="$HOME/.nvm"; . "$NVM_DIR/nvm.sh"; nvm use 22` 才能调用 `nemoclaw`。

### 6.2 验证（实测）
- `nemoclaw my-assistant status`：Phase Ready；Inference(ollama backend) healthy `http://127.0.0.1:11434/api/tags`；auth proxy healthy `:11435`；**Policies: none**（尚未附加，见第 7 节）。
- `ss -ltnp`：Dashboard `127.0.0.1:18789`、OpenShell 网关 `127.0.0.1:8080` + `172.24.0.1:8080` 均在监听（注：文档所述网关端口 `17670` 为旧版/其它构建；**以本机 `ss -ltnp` 实测的 `8080` 为准**）。
- 推理路由：日志 "Native local inference responds inside the sandbox" ✅。

### 6.3 访问方式
- 本机浏览器：`http://127.0.0.1:18789/`
- 终端交互：`nemoclaw launch my-assistant`
- 远程（SSH 会话）：`ssh -L 18789:127.0.0.1:18789 azureuser@<host>` 后开上方 URL
- Dashboard 令牌 URL：`nemoclaw my-assistant dashboard-url --quiet`（含网关令牌，勿外泄）

---

## 7. 待办 / 下一步（剩余项）

按优先级：

1. **策略预设/实战（当前最紧迫）**：`nemoclaw my-assistant status` 显示 `Policies: none`。建议按教程接入 NemoClaw 默认拒绝策略（`nemoclaw-blueprint/policies/openclaw-sandbox.yaml`）或仓库 presets，并**验证"访问未授权域名被拦截"**（用 `openshell term` 实时批准/拒绝；一次性批准仅对当前会话有效）。
2. **（可选文档补全）** 把 `landlock` 与 `network_middlewares` 的嵌套字段补进教程（schema.mdx 中已有，本轮未逐字展开）。
3. **本地推理备选**：若后续想用 `vllm` 而非 Ollama，需在 A100 起 vLLM 服务并托管模型，再 `nemoclaw inference set --provider vllm --sandbox my-assistant`。
4. **middleware 影响跟进**：`aicomic-middleware-api/worker-1` 因 Ollama 改回环已失去 Ollama 访问；若需恢复，可单独给 middleware 起一个仍发布在 `0.0.0.0` 的 Ollama，或将其 `OLLAMA_URL` 改指向其它可用 Ollama 实例。

---

## 8. 风险与注意事项

- **`curl | bash/sh` 安装**：执行远程脚本并起后台服务/容器。落地前必须 review 脚本，确认环境隔离/可信。
- **遥测默认开启**：OpenShell / NemoClaw 默认发送匿名运营计数，关闭见 `OPENSHELL_TELEMETRY_ENABLED=false` 与 `NEMOCLAW_DISABLE_TELEMETRY=1`。
- **OpenShell 网关**：本机实测监听 `127.0.0.1:8080`（文档所述 `17670` 为旧版/其它构建，以 `ss -ltnp` 为准），仅本机；NemoClaw Dashboard `127.0.0.1:18789`（URL 含网关令牌，勿外泄）。
- **凭据安全**：provider 档案中密钥只注入授权端点；切勿把真实 key 写进聊天/文档，用环境变量传递。
- **`filesystem_policy`/`process` 改了要重建沙箱**；`network_policies` 可热更新。
- **NemoClaw 托管环境勿直接动 OpenShell**：避免 `openshell self-update` / `npm update -g openshell` / `openshell gateway start --recreate` / `openshell sandbox create`，一律用 `nemoclaw onboard`（见坑#5）。

---

## 9. 命令速查

```bash
# OpenShell 安装
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
# 验证网关
openshell status
# 跑 agent（OpenCode 示例）
openshell profile import --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml
export OPENROUTER_API_KEY=<key>
openshell provider create --name openrouter --type openrouter --from-existing
openshell sandbox create --name my-agent --from ghcr.io/anomalyco/opencode:latest --provider openrouter -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free

# NemoClaw 安装（非交互式）
export NEMOCLAW_NON_INTERACTIVE=1 NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1 NEMOCLAW_AGENT=openclaw NEMOCLAW_SANDBOX_NAME=my-assistant
export NEMOCLAW_PROVIDER=build NVIDIA_INFERENCE_API_KEY=<key>   # 或 vllm
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
nemoclaw my-assistant status && nemoclaw launch my-assistant
```

> 注：上方 OpenShell 原生命令仅适用于"单独管理 OpenShell"。本项目若用 NemoClaw 托管，沙箱/网关操作请改用 `nemoclaw` CLI（见坑#5），不要直接执行 `openshell sandbox create`。详细字段与示例见同目录教程文档。
