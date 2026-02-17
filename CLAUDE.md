# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 提供处理此代码库的指导。

## 项目概述

这是一个 Claude Code 插件，提供 Rancher 多集群 Kubernetes 管理技能，包括集群管理、资源排查、容量分析、部署管理、资源发现和集群巡检。插件采用 **Sub-Agent + Skill** 架构，Skill 负责触发，Agent 负责干活。

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
│   ├── cluster-management/SKILL.md      # 集群/项目管理
│   ├── resource-troubleshooting/SKILL.md # 资源排查
│   ├── capacity-analysis/SKILL.md       # 容量分析
│   ├── deployment-management/SKILL.md   # 部署管理
│   ├── resource-discovery/SKILL.md      # 资源发现
│   └── cluster-inspection/SKILL.md      # 集群巡检
├── .gitignore
├── CLAUDE.md                            # 本文件
├── LICENSE                              # MIT 许可证
└── README.md
```

## 架构：Sub-Agent + Skill

### Skill 层
- **目的**：意图识别和触发逻辑
- **职责**：决定调用哪个 Sub-Agent
- **特点**：轻量、快速、最小逻辑

### Sub-Agent 层
- **目的**：在隔离上下文中执行复杂操作
- **职责**：获取数据、执行分析、返回结构化结果
- **特点**：独立上下文、可并行、范围聚焦

### 为什么这样设计？

每个 Sub-Agent 有独立上下文，不会互相干扰；多个 Agent 可以并行跑，比如同时分析多个集群或多个节点。Skill 只管「什么时候行动」，Agent 只管「怎么干」，职责清晰。

## Agent 定义 (AGENT.md)

每个 Agent 定义包括：

```yaml
---
name: rancher-agent-name
description: 何时使用此 Agent
tools: ["mcp__rancher__xxx"]  # 可用 MCP 工具
parallel: true                # 可与其他 Agent 并行运行
---

# Agent 职责
# 输入/输出格式
# 执行策略
```

## Skill 定义 (SKILL.md)

Skill 委托给 Agent：

```yaml
---
name: skill-name
description: 触发条件...
version: 1.0.0
---

## 可用 Sub-Agent
- `agent-name` - 何时使用

## 并行模式
并行 Agent 执行示例

## 工作流
1. 解析用户请求
2. 确定 Agent 策略（单个/并行）
3. 启动 Sub-Agent
4. 汇总结果
```

## 并行执行模式

### 多集群对比
```javascript
// 为每个集群并行启动 Agent
const clusters = ["c-abc123", "c-def456", "c-ghi789"];
const tasks = clusters.map(c => Task({
  subagent_type: "general-purpose",
  description: `分析集群 ${c}`,
  prompt: `你是 rancher-cluster-explorer。分析集群 ${c} 的整体状况。`
}));
const results = await Promise.all(tasks);
```

### 多节点分析
```javascript
// 并行分析多个节点
const nodes = ["node-1", "node-2", "node-3"];
const tasks = nodes.map(n => Task({
  subagent_type: "general-purpose",
  description: `分析节点 ${n}`,
  prompt: `你是 rancher-node-analyzer。分析集群 c-abc123 中节点 ${n} 的健康状况和资源使用情况。`
}));
```

### 多 Pod 诊断
```javascript
// 并行诊断多个 Pod
const pods = ["pod-a", "pod-b", "pod-c"];
const tasks = pods.map(p => Task({
  subagent_type: "general-purpose",
  description: `诊断 Pod ${p}`,
  prompt: `你是 rancher-pod-diagnostician。诊断集群 c-abc123 命名空间 default 中 Pod ${p} 的问题。`
}));
```

## 可用 MCP 工具

**Kubernetes 资源操作（读取）：**
- `mcp__rancher__kubernetes_get` -- 获取单个 Kubernetes 资源（Pod、Deployment、Service、Secret 等）
- `mcp__rancher__kubernetes_list` -- 列出 Kubernetes 资源，支持 label selector 过滤
- `mcp__rancher__kubernetes_get_all` -- 获取集群中所有资源（类似 ketall），包括通常隐藏的 ConfigMap、Secret、RBAC、CRD
- `mcp__rancher__kubernetes_describe` -- 描述资源及其关联事件（类似 kubectl describe）
- `mcp__rancher__kubernetes_events` -- 列出 Kubernetes 事件，支持按命名空间、对象名称和类型过滤

**Pod 诊断：**
- `mcp__rancher__kubernetes_inspect_pod` -- 全面 Pod 诊断：详情、父工作负载、指标、日志
- `mcp__rancher__kubernetes_logs` -- 获取 Pod 日志，支持关键词过滤、时间范围、多 Pod 聚合

**节点与容量：**
- `mcp__rancher__kubernetes_node_analysis` -- 分析节点健康状况和资源使用
- `mcp__rancher__kubernetes_capacity` -- 集群资源容量概览（类似 kube-capacity）

**部署与变更：**
- `mcp__rancher__kubernetes_rollout_history` -- 查看 Deployment 发布历史
- `mcp__rancher__kubernetes_diff` -- 比较两个资源版本的差异（git-style diff）
- `mcp__rancher__kubernetes_watch` -- 监控资源变更并返回 diff

**资源关系：**
- `mcp__rancher__kubernetes_dep` -- 显示资源的依赖/被依赖树（类似 kube-lineage）

**Kubernetes 资源操作（写入）：**
- `mcp__rancher__kubernetes_create` -- 创建 Kubernetes 资源（需要 read_only=false）
- `mcp__rancher__kubernetes_patch` -- 使用 JSON Patch 修补资源（需要 read_only=false）
- `mcp__rancher__kubernetes_delete` -- 删除 Kubernetes 资源（需要 read_only=false 且 disable_destructive=false）

**Rancher 管理：**
- `mcp__rancher__cluster_list` -- 列出所有可用 Rancher 集群
- `mcp__rancher__project_list` -- 列出 Rancher 项目

### 输出格式

大多数工具支持 `format` 参数：
- `json`：JSON 格式输出（默认）
- `table`：表格格式输出（人类可读）
- `yaml`：YAML 格式输出

### 敏感数据控制

涉及 Secret 资源的工具支持 `showSensitiveData` 参数：
- 默认 `false`：Secret 数据用 `***` 遮蔽
- 设为 `true`：显示实际值（需要全局 `--show-sensitive-data` 启用）

## 何时使用 Sub-Agent

### 始终使用 Sub-Agent 的场景
- 需要多步分析或多源数据获取（Pod 详情 + 日志 + 事件）
- 可并行化的调用（多集群、多节点、多 Pod）
- 复杂的容量分析或趋势对比
- 集群巡检（系统化多维度健康检查）

### 直接调用 MCP 工具的场景
- 已知具体参数的简单查询（获取单个资源、查看日志）
- 参数明确的单工具调用（列出集群、列出项目）
- 简单的 CRUD 操作（创建、修补、删除资源）

## 集群巡检

巡检是对集群的系统化健康检查，覆盖 6 大维度：

1. **集群基础信息**：状态、版本、项目
2. **节点健康**：Ready 状态、Conditions、Taints、版本一致性
3. **资源容量**：CPU/内存请求/限制/使用率、Pod 数量、过度分配
4. **工作负载健康**：Deployment/StatefulSet/DaemonSet 可用性、异常 Pod
5. **异常事件**：Warning 事件、高频重复事件、关键事件类型
6. **系统组件**：kube-system、cattle-system 核心组件状态

巡检范围：
- **full**：完整巡检（所有维度）
- **quick**：快速巡检（节点 + 事件）
- **nodes/workloads/events**：专项巡检

评分体系：A（优秀）→ B（良好）→ C（一般）→ D（较差）

## 添加新组件

### 添加 Sub-Agent

1. 创建 `agents/<agent-name>/AGENT.md`
2. 定义 Agent 名称、描述、工具、并行能力
3. 记录输入/输出格式
4. 从相关 Skill 引用

### 添加 Skill

1. 创建 `skills/<skill-name>/SKILL.md`
2. 在前言中定义触发条件
3. 记录调用哪个 Sub-Agent
4. 提供并行执行模式

## 测试

1. 安装插件：`/plugin install rancher-assistant@rancher-assistant`
2. 使用用户查询测试 Skill 触发
3. 验证 Sub-Agent 接收正确参数
4. 检查并行执行是否按预期工作

## 重要说明

- Sub-Agent 使用 `subagent_type: "general-purpose"` 运行
- 长时间运行的并行任务使用 `run_in_background: true`
- 向 Sub-Agent 提供清晰的结构化提示，在 Skill 层处理 Agent 失败
- Agent 名称统一使用 `rancher-` 前缀
- 所有 Kubernetes 工具都需要 `cluster` 参数（集群 ID），使用 `cluster_list` 获取
- 默认 `read_only=true`，写操作需要配置 rancher-mcp-server 开启
- 输出格式支持 json、table、yaml，按需选择
