# Rancher Assistant for Claude Code

Claude Code 插件，用于 Rancher 多集群 Kubernetes 管理。采用 **Sub-Agent + Skill** 架构，Skill 负责意图识别，Agent 负责干活，各自独立上下文、支持并行。

## 架构特点

```
┌─────────────────────────────────────────────────────────────┐
│  Skill Layer (触发器)                                         │
│  - 识别用户意图                                               │
│  - 决定调用哪个 Sub-Agent                                    │
│  - 汇总结果                                                   │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Sub-Agent 1 │   │  Sub-Agent 2 │   │  Sub-Agent N │
│  (独立上下文) │   │  (独立上下文) │   │  (独立上下文) │
└──────────────┘   └──────────────┘   └──────────────┘
```

### 为什么这样设计？

- 每个 Sub-Agent 有独立上下文，不会互相干扰
- 多个 Agent 可以并行跑，比如同时分析多个集群或多个节点
- Skill 只管「什么时候触发」，Agent 只管「怎么干活」，各司其职

## 安装

### 1. 添加插件市场

```bash
/plugin marketplace add futuretea/rancher-assistant
```

### 2. 安装插件

```bash
/plugin install rancher-assistant@rancher-assistant
```

### 3. 验证安装

```bash
/plugin list
```

安装成功后，你应该能看到 `rancher-assistant` 在已安装插件列表中。

## 包含的技能

### 1. Cluster Management（集群管理）

**触发词**: "list clusters", "cluster overview", "list projects", "compare clusters", "集群列表", "集群概览"

**执行方式**:
- 简单操作（列出集群/项目）直接执行
- 集群概览委托给 `rancher-cluster-explorer`
- 多集群对比并行处理

### 2. Resource Troubleshooting（资源排查）

**触发词**: "diagnose pod", "pod logs", "why is pod failing", "check events", "troubleshoot", "诊断 Pod", "排查问题", "查看日志"

**执行方式**:
- 简单日志/事件查询直接执行
- Pod 诊断委托给 `rancher-pod-diagnostician`
- 部署排查委托给 `rancher-deployment-tracker`
- 多 Pod 诊断可并行启动多个诊断 Agent

### 3. Capacity Analysis（容量分析）

**触发词**: "node capacity", "cluster capacity", "resource usage", "node health", "容量", "节点分析", "资源使用"

**执行方式**:
- 委托给 `rancher-node-analyzer`
- 多集群容量对比并行处理
- 全面审查并行启动 cluster-explorer + node-analyzer

### 4. Deployment Management（部署管理）

**触发词**: "rollout history", "deployment changes", "watch deployment", "diff", "what changed", "部署历史", "变更", "跨集群对比"

**执行方式**:
- 委托给 `rancher-deployment-tracker`
- 支持跨集群资源 diff 对比
- 支持实时监控资源变更

### 5. Resource Discovery（资源发现）

**触发词**: "list resources", "get all", "dependency tree", "what depends on", "资源列表", "依赖关系", "资源清单"

**执行方式**:
- 委托给 `rancher-resource-scout`
- 支持命名空间资源清查
- 支持资源依赖关系树分析
- 多命名空间/多集群并行搜索

### 6. Cluster Inspection（集群巡检）

**触发词**: "inspection", "inspect cluster", "health check", "patrol", "巡检", "集群巡检", "健康检查", "集群体检", "日常巡检"

**执行方式**:
- 委托给 `rancher-cluster-inspector`
- 支持完整巡检（full）、快速巡检（quick）和专项巡检（nodes/workloads/events）
- 多集群并行巡检，生成多集群总览
- 覆盖 6 大维度：集群信息、节点健康、资源容量、工作负载、异常事件、系统组件
- 各维度独立评分（A/B/C/D），生成结构化报告
- 支持变更前后对比巡检

## 项目结构

```
rancher-assistant/
├── .claude/
│   └── settings.local.json              # 工具权限配置
├── .claude-plugin/
│   ├── marketplace.json                 # 插件市场元数据
│   └── plugin.json                      # 插件元数据
├── agents/                              # Sub-Agent 定义
│   ├── cluster-explorer/AGENT.md        # 多集群导航 Agent
│   ├── pod-diagnostician/AGENT.md       # Pod 诊断 Agent
│   ├── node-analyzer/AGENT.md           # 节点分析 Agent
│   ├── deployment-tracker/AGENT.md      # 部署追踪 Agent
│   ├── resource-scout/AGENT.md          # 资源发现 Agent
│   └── cluster-inspector/AGENT.md       # 集群巡检 Agent
├── skills/                              # Skill 触发器
│   ├── cluster-management/SKILL.md
│   ├── resource-troubleshooting/SKILL.md
│   ├── capacity-analysis/SKILL.md
│   ├── deployment-management/SKILL.md
│   ├── resource-discovery/SKILL.md
│   └── cluster-inspection/SKILL.md
├── .gitignore
├── CLAUDE.md
├── LICENSE
└── README.md
```

## 使用示例

### 集群管理

```
# 列出所有集群
"列出所有 Rancher 集群"

# 集群概览
"production 集群的整体状况"
→ 启动 cluster-explorer
→ 并行获取容量、节点、项目信息
→ 生成概览报告

# 多集群对比
"对比 production 和 staging 集群"
→ 并行启动 2 个 cluster-explorer
→ 对比容量和健康状况
```

### 资源排查

```
# 查看日志
"查看 Pod api-server-abc123 的日志"
→ 直接调用 kubernetes_logs

# Pod 诊断
"诊断 Pod api-server-abc123"
→ 启动 pod-diagnostician
→ 并行获取 Pod 详情 + 日志 + 事件
→ 生成诊断报告

# 多 Pod 诊断
"诊断命名空间 production 中所有失败的 Pod"
→ 先列出异常 Pod
→ 并行启动多个 pod-diagnostician
→ 汇总所有诊断结果
```

### 容量分析

```
# 节点健康检查
"所有节点的健康状况"
→ 启动 node-analyzer
→ 分析所有节点状态

# 集群容量
"集群还有多少可用容量？"
→ 启动 node-analyzer
→ 获取 CPU/内存/Pod 使用率
→ 给出容量规划建议

# 多集群容量对比
"哪个集群还有空间部署新应用？"
→ 并行启动多个 node-analyzer
→ 对比各集群可用容量
```

### 部署管理

```
# 发布历史
"api-server 的发布历史"
→ 启动 deployment-tracker
→ 展示修订版本列表

# 跨集群对比
"对比 staging 和 production 的 api-server"
→ 启动 deployment-tracker
→ 使用 kubernetes_diff 跨集群对比
→ 展示镜像版本、副本数等差异

# 监控滚动更新
"监控 api-server 的更新过程"
→ 启动 deployment-tracker
→ 使用 kubernetes_watch 持续监控
→ 报告变更过程
```

### 资源发现

```
# 命名空间清查
"production 命名空间里有什么资源？"
→ 启动 resource-scout
→ 使用 kubernetes_get_all 获取所有资源
→ 按类型汇总

# 依赖关系
"Deployment api-server 依赖什么？"
→ 启动 resource-scout
→ 使用 kubernetes_dep 获取依赖树
→ 展示 ConfigMap、Secret、PVC 等依赖

# 最近变更
"最近 1 小时创建了哪些资源？"
→ 启动 resource-scout
→ 使用 kubernetes_get_all（since: "1h"）
→ 展示最近创建的资源
```

### 集群巡检

```
# 完整巡检
"对 production 集群做一次完整巡检"
→ 启动 cluster-inspector
→ 并行采集节点、容量、工作负载、事件、系统组件数据
→ 生成巡检报告（含评分和建议）

# 快速巡检
"快速检查一下集群状态"
→ 启动 cluster-inspector（scope: quick）
→ 检查节点健康 + 异常事件
→ 生成快速报告

# 多集群巡检
"巡检所有集群"
→ 获取集群列表
→ 并行启动多个 cluster-inspector
→ 生成多集群巡检总览

# 变更前检查
"做一次变更前巡检"
→ 完整巡检，保存为基线

# 专项巡检
"检查所有节点健康状况" / "工作负载巡检" / "检查异常事件"
→ 启动 cluster-inspector（scope: nodes/workloads/events）
```

## Sub-Agent 并行执行模式

并行的思路很简单：把可以同时跑的任务拆给不同 Agent，最后汇总。下面是几种典型场景：

### 多集群并行分析

```javascript
User: "分析所有集群的状况"
→ Parallel:
   Task({ agent: "cluster-explorer", cluster: "c-abc123" })
   Task({ agent: "cluster-explorer", cluster: "c-def456" })
   Task({ agent: "cluster-explorer", cluster: "c-ghi789" })
→ 汇总对比
```

### 多节点并行诊断

```javascript
User: "分析 node-1、node-2、node-3 的负载"
→ Parallel:
   Task({ agent: "node-analyzer", node: "node-1" })
   Task({ agent: "node-analyzer", node: "node-2" })
   Task({ agent: "node-analyzer", node: "node-3" })
→ 对比负载数据
```

### 多 Pod 并行排查

```javascript
User: "诊断这三个 Pod"
→ Parallel:
   Task({ agent: "pod-diagnostician", pod: "pod-a" })
   Task({ agent: "pod-diagnostician", pod: "pod-b" })
   Task({ agent: "pod-diagnostician", pod: "pod-c" })
→ 汇总诊断结果
```

### 多集群并行巡检

```javascript
User: "巡检所有集群"
→ Parallel:
   Task({ agent: "cluster-inspector", cluster: "c-abc123" })
   Task({ agent: "cluster-inspector", cluster: "c-def456" })
   Task({ agent: "cluster-inspector", cluster: "c-ghi789" })
→ 生成多集群巡检总览
```

### 跨集群资源对比

```javascript
User: "对比三个环境中的 api-server"
→ Parallel:
   Task({ agent: "deployment-tracker", cluster: "dev", diff: true })
   Task({ agent: "deployment-tracker", cluster: "staging", diff: true })
   Task({ agent: "deployment-tracker", cluster: "production", diff: true })
→ 汇总差异
```

## 依赖

- Claude Code 插件系统
- Rancher MCP 服务器

> **注意**: 本插件需要搭配 [rancher-mcp-server](https://github.com/futuretea/rancher-mcp-server) 使用。请先完成 rancher-mcp-server 的安装和配置，详见其 [README](https://github.com/futuretea/rancher-mcp-server#readme)。

### 输出格式

大多数 MCP 工具支持 `format` 参数：
- `json`：JSON 格式（默认，适合程序处理）
- `table`：表格格式（人类可读）
- `yaml`：YAML 格式

### 分页参数

`kubernetes_list` 和其他列表工具支持分页：
- `limit`：每页数量（默认 100）
- `page`：页码（从 1 开始）

## 开发

### 添加 Sub-Agent

1. 创建 `agents/<agent-name>/AGENT.md`
2. 定义 agent 名称、工具集、并行能力
3. 说明输入输出格式
4. 从 skill 中引用

### 添加 Skill

1. 创建 `skills/<skill-name>/SKILL.md`
2. 定义触发条件（description）
3. 说明何时调用哪个 sub-agent
4. 提供并行执行模式示例

添加新功能时，值得想一下：能不能跟多个集群并行跑？能不能复用已有 Agent？能不能减少 API 调用次数？

### Agent 文件格式

```markdown
---
name: rancher-agent-name       # 统一 rancher- 前缀
description: When to use this agent
tools: ["mcp__rancher__xxx"]   # 需要的 MCP 工具
parallel: true                 # 是否支持并行
---

# Agent Title

## Responsibilities
What this agent does

## Input Format
Parameters it accepts

## Output Format
Structured response format

## Execution Strategy
How to parallelize operations
```

## License

MIT
