# oh-my-openagent 安装执行方案

> 执行者：OpenCode Agent（执行角色）
> 审批者：Hermes（Plan Gate / 审查角色）
> 协作方式：GitHub PR Review
> 目标环境：macOS（darwin），当前用户 home 目录

---

## 一、目标与范围

### 1.1 目标
在当前 macOS 环境中安装并配置 oh-my-openagent，使其作为 OpenCode 的多代理编排插件正常工作。

### 1.2 范围（MUST DO）
- 安装 oh-my-openagent 插件包
- 生成并配置 `oh-my-openagent.json`
- 配置模型 provider（至少一个可用）
- 运行 `doctor` 诊断确认无错误
- 验证核心代理（Sisyphus、Prometheus、Atlas、Oracle、Explore）可见可调用

### 1.3 不在此次范围（MUST NOT DO）
- 不创建自定义 skill（使用默认内置 skill 起步）
- 不开启 team_mode（默认关闭）
- 不修改任何项目业务代码
- 不创建/修改项目 AGENTS.md（后续再考虑）
- 不配置 /ulw-loop 或 /ralph-loop 自动执行

---

## 二、前置条件检查

| # | 条件 | 检查命令 | 必须 | 失败处理 |
|---|------|----------|------|----------|
| 1 | OpenCode 已安装 | `which opencode` 或在 PATH 中 | **是** | **阻塞** — 停止安装，提示用户先安装 OpenCode（https://opencode.ai/） |
| 2 | Bun >= 1.0 | `bun --version` | **是** | **阻塞** — `curl -fsSL https://bun.sh/install \| bash` 后重试 |
| 3 | 代理连通性 | `curl -x http://127.0.0.1:7897 -sI https://registry.npmjs.org/ \| head -1` | **是** | **阻塞** — 本机外网必须走代理，否则 bun/npm install 超时 |
| 4 | 至少一个模型 API Key / 聚合器 endpoint | 检查环境变量或 OpenCode 配置 | **是** | **阻塞** — 提示用户配置 API key 或聚合器地址 |
| 5 | Node.js >= 18（Bun 一般自带） | `node --version` | 否 | OK，Bun 自带 runtime |
| 6 | 磁盘空间 > 500MB | `df -h` | 否 | 提醒清理 |

### 2.1 已知环境信息
- 平台：macOS（darwin）
- 外网代理：`127.0.0.1:7897`（**所有包管理器必须走此代理**）
- 模型 provider：通过聚合器（yunwu.ai、4sapi.com）接入，不直连大厂 API
- 已有 GitHub token 配置完成（xymu2026-boop）
- 当前工作目录：`/Users/limuxy`
- OpenCode 配置目录：待探测（通常 `~/.config/opencode/` 或 `~/.opencode/`）

---

## 三、安装步骤

### Step 0: 预检（Pre-flight Check）

执行前必须确认以下 3 项全部通过，否则停止流程：

```bash
# 0.1 检查 OpenCode 是否安装
which opencode || echo "BLOCKED: OpenCode not found"
# 若 BLOCKED → 停止，提示先安装 OpenCode

# 0.2 设置代理并验证连通性
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export ALL_PROXY=http://127.0.0.1:7897
curl -x http://127.0.0.1:7897 -sI https://registry.npmjs.org/ | head -1
# 预期: HTTP/2 200
# 若失败 → 停止，检查代理是否正常运行

# 0.3 探测 OpenCode 配置目录
OPENCODE_DIR=$(opencode config path 2>/dev/null || echo "$HOME/.config/opencode")
echo "OpenCode config dir: $OPENCODE_DIR"
```

**门控规则**：0.1 和 0.2 任一失败 → **立即停止，不继续**。

---

### Step 1: 安装 oh-my-openagent 插件

> 以下命令均需在代理环境中执行

```bash
# 确保代理已设置（从 Step 0 延续）
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897

# 方式一：通过 OpenCode 插件机制安装
opencode plugin install oh-my-openagent

# 方式二：如果上条不可用，用 bunx 初始化
bunx oh-my-openagent@latest init --directory "$OPENCODE_DIR"
```

**预期结果**：在 **OpenCode 配置目录**（`~/.config/opencode/` 或探测到的路径）下生成 `oh-my-openagent.json` 配置文件。**不在当前 repo 目录生成**。

**失败处理**：若安装超时或网络错误，确认代理环境变量已设置后重试；记录错误日志。

---

### Step 2: 生成基础配置文件

配置文件路径（**必须在 OpenCode 配置目录**）：
```
$OPENCODE_DIR/oh-my-openagent.json
```

如未自动生成，手动创建最小配置（**注意：provider 使用通用占位，由 doctor 或用户自行补充聚合器 endpoint**）：

```json
{
  "background_task": {
    "defaultConcurrency": 3,
    "providerConcurrency": {}
  },
  "sisyphus_agent": {
    "disabled": false,
    "planner_enabled": true,
    "replace_plan": true
  },
  "skills": {
    "enable": ["git-master", "review-work"],
    "disable": []
  }
}
```

> **说明**：该环境通过聚合器（yunwu.ai、4sapi.com）使用模型，不直连大厂 API。`providerConcurrency` 留空让 doctor 自动检测可用 provider，避免写死 OpenAIs/Google/Anthropic。

**配置原则**（来自官方文档）：
- 先用默认配置跑通，不做过度调优
- team_mode 默认关闭
- planner_enabled 开启（保证 @plan 可用）
- 只启用最基础的内置 skills
- provider 让系统自动检测，不预先写死

---

### Step 3: 运行健康检查

```bash
bunx oh-my-openagent doctor
```

**检查项**：
- [ ] 系统环境兼容性
- [ ] 配置文件语法正确
- [ ] 工具链可用
- [ ] 模型 provider 连接正常
- [ ] 无权限/路径错误

**若 doctor 报错**：逐项排查，修到 0 error 为止。

---

### Step 4: 验证核心代理可用性

在 OpenCode 中依次发起以下调用确认代理存在：

| 代理 | 验证命令 | 预期 |
|------|----------|------|
| Sisyphus | `@Sisyphus hello` | 有响应 |
| Prometheus | `@plan "check system status"` | 生成计划（哪怕简单） |
| Atlas | 完成 plan gate 后 `/start-work` | 开始调度 |
| Explore | 询问"搜索当前目录文件结构" | 返回文件列表 |
| Oracle | 询问"评估此项目架构风险" | 给出只读分析 |

---

## 四、验证清单（8 项 Gate）

| # | 检查项 | 通过标准 | 权重 |
|---|--------|----------|------|
| V1 | 安装成功 | `bunx oh-my-openagent doctor` 无 error | P0 |
| V2 | 配置生成 | `oh-my-openagent.json` 存在且语法有效 | P0 |
| V3 | Sisyphus 可用 | 能接受并响应任务 | P0 |
| V4 | Prometheus 可用 | `@plan` 能生成计划 | P1 |
| V5 | Atlas 可用 | `/start-work` 能调度执行 | P1 |
| V6 | Explore 可用 | 能搜索代码库 | P1 |
| V7 | Oracle 可用 | 能给出只读分析 | P2 |
| V8 | 不影响现有环境 | 原有 OpenCode 行为不变 | P0 |

**通过条件**：所有 P0 项必须通过，P1 至少通过 2 项。

---

## 五、风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| OpenCode 未安装 | 中 | 阻塞 | Step 0 检查，失败则停止并提示安装 |
| 代理未连通 | 高 | 阻塞 | Step 0 验证代理，所有命令前置 `export HTTP_PROXY/HTTPS_PROXY` |
| bun 未安装 | 低 | 阻塞 | 先检查，必要时 `curl -fsSL https://bun.sh/install \| bash` |
| 模型 API Key / 聚合器未配置 | 中 | 阻塞 | 先检查环境变量；提示用户补充 |
| 安装过程修改全局配置 | 低 | 中 | 先备份已有配置 |
| 代理名冲突 | 低 | 低 | 检查 opensource agent 列表 |
| doctor 报网络错误 | 低 | 中 | 确认代理已设置、重试 |

---

## 六、回滚方案

若安装后出现问题且无法快速修复：

```bash
# 移除插件
opencode plugin remove oh-my-openagent

# 或删除配置文件（注意路径在 OpenCode 配置目录）
rm "$OPENCODE_DIR/oh-my-openagent.json"

# 恢复备份
cp "$OPENCODE_DIR/oh-my-openagent.json.bak" "$OPENCODE_DIR/oh-my-openagent.json"
```

回滚后确认：`opencode` 恢复正常使用，无残留错误。

---

## 七、审批决策

> **审查者（Hermes）请在 PR Review 中给出以下判断之一：**

- **READY_TO_EXECUTE**：方案完整、边界清晰、可执行
- **NEEDS_REVISION**：存在模糊点、遗漏前提、验收不明确 —— 请标记具体问题

### 审查聚焦点
1. 前置条件是否遗漏？（特别是 macOS 特殊项）
2. 执行步骤是否有模糊/跳过的地方？
3. 验证标准是否可量化、可判断？
4. 哪些地方可能导致 Agent 自作主张？
5. 回滚方案是否充分？

---

## 八、执行后归档

安装执行完成后：
1. 在 `oh-my-openagent.json` 同目录生成 `INSTALL_RESULT.md` 记录结果
2. 记录 doctor 完整输出
3. 记录每个验证项的通过/失败状态
4. 如遇问题，记录归因

---

*方案版本：v1.1 | 创建时间：2026-05-31 | 状态：待复审 | 修订：响应 Hermes Plan Gate 审查*

---

## 附录：修订记录（v1.0 → v1.1）

| # | Hermes 意见 | 严重度 | 修复内容 | 位置 |
|---|------------|--------|----------|------|
| 1 | 未检查 OpenCode 是否存在，失败无处理 | 🔴 阻塞 | 新增 **Step 0 预检**：检查 OpenCode、代理、配置目录；任一步失败→停止。前置条件表增加"失败处理"列 | §2, §3 Step 0 |
| 2 | 忽略代理环境 `127.0.0.1:7897` | 🔴 阻塞 | 前置条件增加代理连通性验证；Step 0 设置 `HTTP_PROXY/HTTPS_PROXY`；Step 1 所有命令前置代理环境变量；风险评估增加"代理未连通" | §2, §3 Step0/1, §5 |
| 3 | Provider 写死 OpenAI/Google/Anthropic | 🟡 重要 | `providerConcurrency` 改为空对象，让 doctor 自动检测；JSON 配置加注释说明聚合器环境 | §3 Step 2 |
| 4 | 配置文件路径模糊 | 🟡 重要 | 明确路径为 `$OPENCODE_DIR/oh-my-openagent.json`；Step 0 探测 OpenCode 配置目录；回滚方案同步修正 | §3 Step 0/2, §6 |
