---
name: rancher-cluster-inspection
description: This skill should be used when the user asks to "inspection", "inspect cluster", "health check", "patrol", "cluster review", "巡检", "集群巡检", "健康检查", "集群体检", "日常巡检", "安全巡检", "变更前检查", "变更后检查", "定期检查", or discusses systematic cluster health inspection and patrol (系统化集群巡检和健康检查).
version: 1.0.0
---

# Rancher 集群巡检

本技能提供专业的 Kubernetes 集群巡检能力，覆盖节点健康、资源容量、工作负载状态、异常事件和系统组件等维度，生成结构化巡检报告。

## 主要 Sub-Agent

### `rancher-cluster-inspector`
**用于**: 所有巡检任务

**能力**:
- 完整巡检：覆盖 6 大维度（集群信息、节点健康、资源容量、工作负载、异常事件、系统组件）
- 快速巡检：集群信息 + 节点健康 + 异常事件
- 专项巡检：节点专项、工作负载专项、事件专项
- 多集群巡检：并行巡检多个集群，生成总览
- 评分体系：每个维度独立评分（A/B/C/D），整体评分

**传递参数**:
```json
{
  "cluster": "c-abc123",
  "scope": "full" | "quick" | "nodes" | "workloads" | "events",
  "namespaces": [],
  "include_system": true,
  "include_recommendations": true
}
```

## 决策树

```
用户请求：
├─ "集群巡检" / "cluster inspection" / "健康检查" / "集群体检"
│  └─ 委托给 rancher-cluster-inspector（scope: "full"）
│
├─ "快速检查" / "quick check" / "简单看看集群状态"
│  └─ 委托给 rancher-cluster-inspector（scope: "quick"）
│
├─ "节点巡检" / "检查所有节点" / "node inspection"
│  └─ 委托给 rancher-cluster-inspector（scope: "nodes"）
│
├─ "工作负载巡检" / "应用健康检查" / "workload inspection"
│  └─ 委托给 rancher-cluster-inspector（scope: "workloads"）
│
├─ "事件巡检" / "检查异常事件" / "event inspection"
│  └─ 委托给 rancher-cluster-inspector（scope: "events"）
│
├─ "巡检所有集群" / "全部集群体检" / "inspect all clusters"
│  └─ 获取集群列表 → 并行启动多个 rancher-cluster-inspector
│
├─ "变更前检查" / "pre-change check"
│  └─ 委托给 rancher-cluster-inspector（scope: "full"，记录基线）
│
└─ "变更后检查" / "post-change check"
   └─ 委托给 rancher-cluster-inspector（scope: "full"，与基线对比）
```

## 并行执行模式

### 模式 1: 单集群完整巡检
```
用户: "对 production 集群做一次完整巡检"

→ 启动 rancher-cluster-inspector
  参数: { cluster: "c-abc123", scope: "full" }
→ Agent 内部并行采集所有维度数据
→ 生成结构化巡检报告（含评分和建议）
```

### 模式 2: 多集群并行巡检
```
用户: "巡检所有集群"

→ 步骤 1: 直接调用 cluster_list 获取所有集群
→ 步骤 2: 为每个集群并行启动 rancher-cluster-inspector
  Agent 1: rancher-cluster-inspector（production）
  Agent 2: rancher-cluster-inspector（staging）
  Agent 3: rancher-cluster-inspector（dev）
→ 步骤 3: 汇总所有集群报告，生成多集群巡检总览
```

### 模式 3: 指定命名空间巡检
```
用户: "巡检 production 集群的 app 和 monitoring 命名空间"

→ 启动 rancher-cluster-inspector
  参数: { cluster: "c-abc123", scope: "workloads", namespaces: ["app", "monitoring"] }
→ 聚焦指定命名空间的工作负载和事件
```

### 模式 4: 变更前后对比巡检
```
用户: "做一次变更前巡检"

→ 启动 rancher-cluster-inspector（scope: "full"）
→ 保存报告作为基线

用户（变更后）: "做变更后检查"

→ 启动 rancher-cluster-inspector（scope: "full"）
→ 与之前的基线对比，高亮变化项
```

## 工作流

### 步骤 1: 识别巡检类型
- 完整巡检 vs 快速巡检 vs 专项巡检？
- 单集群 vs 多集群？
- 是否指定命名空间？

### 步骤 2: 获取集群信息
如果用户提供集群名称而非 ID：
```
→ 使用 cluster_list（name: "关键词"）搜索
→ 获取匹配的集群 ID
```

如果用户要求巡检"所有集群"：
```
→ 使用 cluster_list 获取完整列表
```

### 步骤 3: 启动巡检 Agent
```
Task({
  subagent_type: "general-purpose",
  description: "巡检集群 " + cluster_name,
  prompt: `你是 rancher-cluster-inspector。对集群 ${cluster}（${cluster_name}）执行${scope}巡检。检查节点健康、资源容量、工作负载状态、异常事件和系统组件，生成结构化巡检报告，包含评分和改进建议。`
})
```

### 步骤 4: 展示巡检报告
- 评分概览表格
- 各维度详细检查结果
- 问题清单（按严重程度排序）
- 改进建议（按优先级排序）

## 响应格式

### 单集群巡检报告
```
## 集群巡检报告: production (c-abc123)

### 巡检概览
- 巡检时间: 2025-01-15 10:30
- 巡检范围: 完整巡检
- **整体评分: B（良好）**

### 评分概览
| 维度 | 评分 | 状态 |
|------|------|------|
| 集群基础信息 | A | ✅ 正常 |
| 节点健康 | B | ⚠️ 注意 |
| 资源容量 | A | ✅ 正常 |
| 工作负载健康 | B | ⚠️ 注意 |
| 异常事件 | A | ✅ 正常 |
| 系统组件 | A | ✅ 正常 |

### 问题清单
| 严重程度 | 维度 | 问题 | 建议 |
|----------|------|------|------|
| ⚠️ | 节点 | node-5 NotReady | 检查 kubelet |
| ⚠️ | 工作负载 | 2 个 Pod CrashLoopBackOff | 查看日志 |

### 改进建议
1. **[紧急]** 修复 node-5
2. **[建议]** 排查崩溃 Pod
```

### 多集群巡检总览
```
## 多集群巡检总览

| 集群 | 评分 | 节点 | 容量 | 工作负载 | 事件 | 关键问题 |
|------|------|------|------|----------|------|----------|
| production | B | ⚠️ | ✅ | ⚠️ | ✅ | 1 节点 NotReady |
| staging | A | ✅ | ✅ | ✅ | ✅ | 无 |
| dev | C | ⚠️ | ⚠️ | ⚠️ | ⚠️ | 容量不足 |

### 集群详情
[各集群独立巡检报告...]
```

## 巡检最佳实践

1. **日常巡检**：每天执行一次快速巡检（quick），关注节点和事件
2. **周巡检**：每周执行一次完整巡检（full），覆盖所有维度
3. **变更巡检**：重大变更前后各做一次完整巡检，对比差异
4. **事件驱动**：收到告警后执行对应专项巡检
5. **多集群**：定期对所有集群做完整巡检，生成健康趋势

## 错误处理

- **metrics-server 未安装**: 跳过实际使用率检查，仅展示请求/限制数据，在报告中注明
- **集群不可达**: 标记为巡检失败，报告集群连接问题
- **权限不足**: 尽可能巡检可访问的资源，在报告中注明权限限制
- **数据不完整**: 基于可用数据生成报告，标注缺失项

## 与其他技能的关系

| 巡检发现问题 | 后续行动 | 使用技能 |
|-------------|----------|----------|
| 节点 NotReady | 深入分析节点 | capacity-analysis |
| Pod CrashLoopBackOff | 诊断 Pod | resource-troubleshooting |
| Deployment 不可用 | 查看部署变更 | deployment-management |
| 资源不足 | 容量规划 | capacity-analysis |
| 可疑事件 | 追溯资源变更 | resource-discovery |
