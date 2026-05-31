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

| # | 条件 | 检查命令 | 必须 |
|---|------|----------|------|
| 1 | Bun >= 1.0 | `bun --version` | 是 |
| 2 | OpenCode 可用 | `which opencode` 或在 PATH 中 | 是 |
| 3 | 至少一个模型 API Key | 检查环境变量或配置 | 是 |
| 4 | Node.js >= 18（Bun 一般自带） | `node --version` | 否 |
| 5 | 磁盘空间 > 500MB | `df -h` | 否 |

### 2.1 已知信息
- 平台：macOS（darwin）
- 已有 GitHub token 配置完成（xymu2026-boop）
- 当前工作目录：`/Users/limuxy`

---

## 三、安装步骤

### Step 1: 安装 oh-my-openagent 插件

```bash
# 方式一：通过 OpenCode 插件机制安装
opencode plugin install oh-my-openagent

# 方式二：如果上条不可用，用 bunx 初始化
bunx oh-my-openagent@latest init
```

**预期结果**：在当前目录或 OpenCode 配置目录下生成 `oh-my-openagent.json` 配置文件。

**失败处理**：若安装失败，检查 bun 版本和网络连接；记录错误日志。

---

### Step 2: 生成基础配置文件

确认配置文件位置（按优先级）：
1. 项目目录下的 `oh-my-openagent.json`
2. OpenCode 全局配置目录下的 `oh-my-openagent.json`

如未自动生成，手动创建最小配置：

```json
{
  "background_task": {
    "defaultConcurrency": 3,
    "providerConcurrency": {
      "openai": 2,
      "google": 3,
      "anthropic": 2
    }
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

**配置原则**（来自官方文档）：
- 先用默认配置跑通，不做过度调优
- team_mode 默认关闭
- planner_enabled 开启（保证 @plan 可用）
- 只启用最基础的内置 skills

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
| bun 未安装 | 低 | 阻塞 | 先检查，必要时 `curl -fsSL https://bun.sh/install | bash` |
| 模型 API Key 未配置 | 中 | 阻塞 | 先检查环境变量；提示用户补充 |
| 安装过程修改全局配置 | 低 | 中 | 先备份已有配置 |
| 代理名冲突 | 低 | 低 | 检查 opensource agent 列表 |
| doctor 报网络错误 | 低 | 中 | 检查代理设置、重试 |

---

## 六、回滚方案

若安装后出现问题且无法快速修复：

```bash
# 移除插件
opencode plugin remove oh-my-openagent

# 或删除配置文件
rm oh-my-openagent.json

# 恢复备份
cp oh-my-openagent.json.bak oh-my-openagent.json
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

*方案版本：v1.0 | 创建时间：2026-05-31 | 状态：待审批*
