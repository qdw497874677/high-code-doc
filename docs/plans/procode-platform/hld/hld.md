---
title: Pro-code 高代码开发模式概要设计
date: 2026-02-27
tags:
  - 概要设计
  - pro-code
  - AI Agent平台
status: draft
---

# Pro-code 高代码开发模式概要设计

## 1. 背景 & 目标

### 业务背景

现有 AI Agent 低代码开发平台需要升级，支持高代码（pro-code）开发模式。与低代码通过可视化配置不同，高代码通过 SDK 进行开发，面向外部开发者提供更灵活的定制能力。

### 要解决的问题

- 支持外部开发者通过 SDK 开发 AI Agent
- 提供完整的开发管理能力（Git、迭代、流水线）
- 与现有低代码模式共存，共享平台基础设施

### 成功指标

- 外部开发者可通过脚手架快速初始化项目
- 支持完整的迭代发布流程
- 流水线支持断点续跑，保证可靠性

---

## 2. 架构设计

### 系统架构图

```mermaid
flowchart TB
    subgraph Platform["AI Agent 平台"]
        subgraph AppMgr["应用管理 (升级)"]
            LC[低代码配置]
            PC[Pro-code配置]
        end
        
        subgraph DevMgr["开发管理 (新增)"]
            CRM[代码仓库管理]
            IM[迭代/版本管理]
            PM[流水线管理]
            WI[WebIDE-后续]
        end
        
        subgraph OpsMgr["运维管理 (接口定义)"]
            DC[deploy/create]
            DS[deploy/status]
            CB[deploy/callback]
        end
    end
    
    subgraph External["外部依赖"]
        GitLab[外部GitLab]
        K8s[K8s Operator]
    end
    
    subgraph Storage["存储层"]
        MySQL[(元数据存储)]
    end
    
    AppMgr --> MySQL
    DevMgr --> MySQL
    DevMgr --> GitLab
    OpsMgr --> K8s
    OpsMgr --> MySQL
```

### 技术选型

| 组件 | 技术 | 选型理由 |
|------|------|----------|
| 后端框架 | Java + Spring Boot | 复用现有技术栈，降低学习成本 |
| Git 操作 | JGit / GitLab4J API | 支持多种 Git 托管平台 |
| 流水线 | Spring State Machine | 持久化 + 断点续跑，与 Spring 生态集成 |
| 存储 | MySQL | 复用现有数据库，运维成本低 |

### 关键设计决策

1. **单体扩展**：在现有服务中增加高代码模块，共用数据库和基础能力
2. **高/低代码共存不可切换**：应用创建时确定模式，后续不可更改
3. **运维接口先行**：一期定义接口规范，实现留白
4. **高/低代码共享运维接口**：部署、监控等运维能力共享
5. **流水线配置系统固定**：阶段配置不可修改，使用默认值

---

## 3. 核心流程

### 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| **迭代** | Iteration | 一次完整的开发周期，从功能开发到发布上线 |
| **迭代阶段** | Iteration Phase | 迭代的生命周期阶段：DEVELOPING（开发中）→ TESTING（测试）→ STAGING（预发）→ RELEASE（发布） |
| **流水线** | Pipeline | 定义一系列有序执行步骤的状态机，支持持久化和断点续跑 |
| **迭代流水线** | Iteration Pipeline | 管理迭代阶段流转的流水线，控制迭代从开发到发布的整体流程 |
| **阶段流水线** | Phase Pipeline | 管理单个迭代阶段内执行步骤的流水线，隶属于迭代流水线 |
| **执行步骤** | Execution Step | 阶段流水线内的原子操作：CODE_CHECK（代码检查）→ BUILD_IMAGE（构建镜像）→ DEPLOY（部署） |
| **代码版本** | Code Version | 面向业务的抽象概念，表示项目代码的一个快照。具体实现使用 Git Tag |
| **Pipeline Hook** | Pipeline Hook | 流水线执行步骤完成后的回调机制，允许平台内部模块在特定事件发生时被通知 |

### 3.1 应用创建流程

```mermaid
flowchart TD
    A[创建应用] --> B{选择开发模式}
    B -->|low-code| C[配置低代码参数]
    B -->|pro-code| D[填写Git仓库地址]
    D --> E[从列表选择技术栈]
    E --> F[自动关联脚手架模板]
    C --> G[完成创建]
    F --> G
```

**说明**：
- **技术栈**：用户从预定义列表选择（一期仅支持 `python`，后续扩展 `java`、`nodejs`）
- **脚手架模板**：根据选择的技术栈自动关联对应模板，无需用户手动配置

### 3.2 迭代发布流程

#### 完整流程

迭代从创建到发布经历多个阶段，每个阶段先创建流水线（含初始化），再执行流水线。

```mermaid
flowchart LR
    subgraph 创建迭代
        A1[创建迭代] --> A2[选择来源分支]
    end

    subgraph DEVELOPING
        A2 --> B1[功能开发]
    end

    B1 --> C0[创建阶段流水线]
    C0 --> C1

    subgraph TESTING阶段流水线
        C1[CODE_CHECK] --> C2[BUILD]
        C2 --> C3[PACKAGE]
        C3 --> C4[DEPLOY]
    end

    C4 --> D0[创建阶段流水线]
    D0 --> D1

    subgraph STAGING阶段流水线
        D1[CODE_CHECK] --> D2[BUILD]
        D2 --> D3[PACKAGE]
        D3 --> D4[DEPLOY]
    end

    D4 --> E0[创建代码版本]
    E0 --> E1

    subgraph RELEASE阶段流水线
        E1[CODE_CHECK] --> E2[BUILD]
        E2 --> E3[PACKAGE]
        E3 --> E4[DEPLOY]
        E4 --> E5[RELEASED]
    end
```

#### 简化流程（一期实现）

一期简化为单阶段发布流程，跳过测试和预发阶段。

```mermaid
flowchart LR
    subgraph 创建代码版本
        subgraph 创建迭代
            A1[创建迭代] --> A2[选择来源分支]
        end
        A2 --> B0[创建RELEASE迭代流水线]
    end

    B0 --> C1

    subgraph RELEASE阶段流水线
        C1[CODE_CHECK] --> C2[BUILD_IMAGE]
        C2 --> C3[DEPLOY]
        C3 --> C4[RELEASED]
    end
```

**说明**：
- 创建代码版本是一个业务动作，包含创建迭代（选择来源分支）和创建 RELEASE 迭代流水线
- 一期阶段流水线简化为三步：CODE_CHECK（代码检查）→ BUILD_IMAGE（构建镜像）→ DEPLOY（发布部署）

#### 创建代码版本

进入 RELEASE 阶段时，先创建代码版本（含打 Tag），再创建阶段流水线。

```mermaid
flowchart TD
    A[进入RELEASE阶段] --> B[用户填写版本信息]
    B --> C{version_name为空}
    C -->|是| D[生成默认值v时间戳]
    C -->|否| E[使用用户输入]
    D --> F[生成完整Tag]
    E --> F
    F --> G[在Git仓库打Tag]
    G --> H[创建CodeVersion记录]
    H --> I[版本信息写入Pipeline.context]
    I --> J[创建阶段流水线]
    J --> K[启动流水线执行]
```

**代码版本数据结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `iteration_id` | bigint | 关联迭代（一对一） |
| `version_name` | varchar(64) | 用户指定的版本标识 |
| `description` | text | 版本描述 |
| `tag` | varchar(128) | Git 仓库中的实际 Tag |
| `commit_hash` | varchar(40) | Tag 指向的 commit |

**用户交互：**

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `version_name` | 否 | `v{timestamp}` | 用户可自定义版本标识 |
| `description` | 否 | 空 | 版本描述 |

**元数据存储：**

```json
{
  "pipelineId": "xxx",
  "stage": "RELEASE",
  "context": {
    "codeVersionId": 123,
    "versionName": "v1.0.0",
    "description": "首次发布",
    "tag": "v1.0.0",
    "branch": "main",
    "commitHash": "abc123"
  }
}
```

#### 完整 vs 简化 对比

| 对比项 | 完整流程 | 简化流程（一期） |
|--------|----------|------------------|
| 迭代阶段 | DEVELOPING → TESTING → STAGING → RELEASE | 直接 RELEASE |
| 环境数量 | 3个（测试、预发、生产） | 1个（生产） |
| 阶段流水线次数 | 3次（测试、预发、发布各一次） | 1次 |
| 执行步骤 | CODE_CHECK → BUILD → PACKAGE → DEPLOY | CODE_CHECK → BUILD_IMAGE → DEPLOY |
| 适用场景 | 正式生产环境 | 快速迭代 / 内部测试 |

### 3.3 流水线架构

**两种流水线类型（复用流水线能力）：**

| 类型 | 说明 | 状态流转 |
|------|------|----------|
| **迭代流水线** | 管理迭代阶段流转 | DEVELOPING → TESTING → STAGING → RELEASE → RELEASED |
| **阶段流水线** | 管理执行步骤流转 | CODE_CHECK → BUILD_IMAGE → DEPLOY（一期）|

**关系：**
```
迭代流水线 (ITERATION)
  │
  ├── 阶段流水线 (PHASE, stage=TESTING)
  ├── 阶段流水线 (PHASE, stage=STAGING)
  └── 阶段流水线 (PHASE, stage=RELEASE)
```

**并发控制：**
- 同一迭代同时只能有一个流水线在执行
- 新流水线创建时检查是否有运行中的流水线

**一期简化：**
- 迭代流水线：只有 RELEASE 一个阶段
- 进入发布阶段时，自动创建阶段流水线

### 3.4 迭代流水线执行流程

```mermaid
flowchart LR
    A[DEVELOPING] --> B[TESTING]
    B --> C[STAGING]
    C --> D[RELEASE]
    D --> E[RELEASED]
```

### 3.5 阶段流水线执行流程（一期）

```mermaid
flowchart LR
    A[CODE_CHECK] --> B[BUILD_IMAGE]
    B --> C[DEPLOY]
    C --> D{部署状态}
    D -->|成功| E[SUCCESS]
    D -->|失败| F[FAILED]
    D -->|超时| F
```

**执行步骤说明：**

| 步骤 | 说明 |
|------|------|
| CODE_CHECK | 代码检查（语法、依赖、安全扫描） |
| BUILD_IMAGE | 构建镜像：基础镜像 + 启动脚本 + tag信息整合存储 |
| DEPLOY | 发布部署：将构建产物信息传递给运维模块触发部署 |

### 3.6 流水线推进驱动机制

**统一的阶段配置模型：**

所有流水线阶段使用统一的配置模型，通过配置区分不同驱动方式。

| 配置项 | 说明 | 可选值 |
|-------|------|--------|
| `driveMode` | 驱动方式 | MANUAL（手动推进）/ AUTO（自动推进）/ ASYNC（异步驱动） |
| `retryMode` | 失败后的重试方式 | AUTO / MANUAL |
| `maxRetries` | 自动重试最大次数 | 0-5 |
| `pollInterval` | 轮询间隔（仅ASYNC） | 秒 |
| `pollTimeout` | 超时时间（仅ASYNC） | 秒 |

**三种驱动方式说明：**

| 驱动方式 | 说明 | 适用场景 |
|---------|------|----------|
| MANUAL | 阶段完成后需手动点击进入下一阶段 | 需人工确认的关键阶段 |
| AUTO | 阶段成功后自动进入下一阶段 | 可自动流转的阶段 |
| ASYNC | 阶段启动后异步等待结果，轮询+回调双保险 | 长时间异步操作（如部署） |

#### 迭代流水线默认配置

迭代流水线的所有阶段默认配置为**手动推进**，确保人对发布流程有完全控制。

| 阶段 | driveMode | retryMode | maxRetries |
|------|-----------|-----------|------------|
| DEVELOPING | MANUAL | MANUAL | 0 |
| TESTING | MANUAL | MANUAL | 0 |
| STAGING | MANUAL | MANUAL | 0 |
| RELEASE | MANUAL | MANUAL | 0 |

#### 阶段流水线默认配置（一期）

| 阶段 | driveMode | retryMode | maxRetries | 说明 |
|------|-----------|-----------|------------|------|
| CODE_CHECK | AUTO | AUTO | 3 | 自动检查，失败自动重试 |
| BUILD_IMAGE | AUTO | AUTO | 2 | 自动构建镜像，失败自动重试 |
| DEPLOY | ASYNC | MANUAL | 0 | 异步部署，轮询+回调 |
| | | | | pollInterval=30s, pollTimeout=30min |

#### DEPLOY 阶段异步驱动

采用**轮询 + 回调双保险**机制：

- **轮询**：流水线定时查询运维模块部署状态
- **回调**：运维模块主动推送部署结果
- 任一方式收到成功/失败，即可推进状态机

```mermaid
flowchart TD
    A[进入DEPLOY阶段] --> B[调用deploy/create]
    B --> C[等待异步结果]
    C --> D{状态获取方式}
    D -->|轮询| E[定时查询deploy/status]
    D -->|回调| F[接收deploy/callback]
    E --> G{部署状态}
    F --> G
    G -->|进行中| C
    G -->|成功| H[DEPLOY SUCCESS]
    G -->|失败| I[DEPLOY FAILED]
    G -->|超时| I
```

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `pollInterval` | 30s | 轮询间隔 |
| `pollTimeout` | 30min | 超时时间 |

### 3.7 Pipeline Hook 机制

流水线提供 Hook 机制，允许平台内部模块在执行步骤完成后被通知。

#### 触发事件

| 事件 | 触发时机 |
|------|----------|
| `CODE_CHECK_COMPLETED` | 代码检查完成 |
| `BUILD_IMAGE_COMPLETED` | 构建镜像完成 |
| `DEPLOY_COMPLETED` | 发布部署完成 |
| `PIPELINE_SUCCEEDED` | 流水线整体成功 |
| `PIPELINE_FAILED` | 流水线整体失败 |

#### 执行特性

- **异步执行**：流水线不等待回调完成
- **自动重试**：回调失败后自动重试（最多3次）
- **失败处理**：记录日志，不影响流水线状态
- **一期范围**：仅支持平台内部模块通过 Spring Event 注册回调

#### Hook 配置存储

Hook 配置保存在 Pipeline 的 context 中：

```json
{
  "pipelineId": "xxx",
  "stage": "RELEASE",
  "context": {
    "codeVersionId": 123,
    "versionName": "v1.0.0",
    "hooks": [
      {
        "eventType": "DEPLOY_COMPLETED",
        "listenerBean": "appStatusUpdateListener",
        "enabled": true
      }
    ]
  }
}
```

#### 事件数据结构

```java
public class PipelineEvent {
    private Long pipelineId;
    private String eventType;
    private String stage;
    private Boolean success;
    private Map<String, Object> context;
    private LocalDateTime occurredAt;
}
```

#### 触发流程

```mermaid
flowchart TD
    A[执行步骤完成] --> B[发布Spring Event]
    B --> C[异步执行Listener]
    C --> D{执行结果}
    D -->|成功| E[记录日志]
    D -->|失败| F[重试]
    F --> G{重试次数}
    G -->|小于3次| C
    G -->|大于等于3次| H[记录失败日志]
```

#### 典型用例

应用管理模块注册 `DEPLOY_COMPLETED` 事件，部署完成后更新应用状态为"已发布"。

---

## 4. 模块划分

### 4.1 应用管理模块（升级）

| 功能 | 说明 |
|------|------|
| Agent 应用管理 | 增加dev_mode字段区分高/低代码 |
| Pro-code 配置 | 存储 Git 仓库、技术栈、脚手架等配置 |

### 4.2 开发管理模块（新增）

| 功能 | 说明 |
|------|------|
| Git 仓库对接 | 对接外部 GitLab，管理 webhook（一期保留设计，不开放配置） |
| 迭代管理 | 管理迭代、阶段、版本 |
| 流水线管理 | 流水线创建、执行、断点续跑 |
| 脚手架模板 | 一期内置 python 模板，后续开放自定义 |

### 4.3 运维管理模块（接口定义）

| 接口 | 说明 |
|------|------|
| deploy/create | 创建部署任务 |
| deploy/status | 查询部署状态 |
| deploy/callback | 部署状态回调 |

**说明：**
- 调用方为 K8s Operator
- 回滚策略由运维模块实现，高代码模块不感知
- 高/低代码应用共享运维接口

---

## 5. 接口清单

### 应用管理模块

| 接口 | 方法 | 用途 |
|------|------|------|
| /apps | POST | 创建应用 |
| /apps/{id} | GET | 获取应用详情 |
| /apps/{id}/procode-config | PUT | 配置 pro-code 参数 |

### 开发管理模块

| 接口 | 方法 | 用途 |
|------|------|------|
| /repos | POST | 创建 Git 仓库配置 |
| /repos/{id}/webhook | GET | 获取 webhook 配置 |
| /iterations | POST | 创建迭代 |
| /iterations/{id} | GET | 获取迭代详情 |
| /iterations/{id}/release | POST | 进入发布阶段 |
| /iterations/{id}/proceed | POST | 推进迭代流水线 |
| /pipelines | POST | 创建流水线 |
| /pipelines/{id} | GET | 获取流水线状态 |
| /pipelines/{id}/proceed | POST | 推进阶段流水线 |
| /pipelines/{id}/retry | POST | 重试当前阶段 |
| /pipelines/{id}/resume | POST | 断点续跑 |
| /pipelines/{id}/cancel | POST | 取消流水线 |

---

## 6. 核心数据实体

```mermaid
erDiagram
    AgentApp ||--o| ProCodeConfig : "pro-code配置"
    ProCodeConfig ||--o| GitRepo : "Git仓库"
    AgentApp ||--o{ Iteration : "迭代"
    Iteration ||--o| CodeVersion : "代码版本"
    Iteration ||--o{ IterationPhase : "阶段"
    Iteration ||--o| Pipeline : "迭代流水线"
    Pipeline ||--o{ Pipeline : "阶段流水线"
```

### 关键实体字段

| 实体 | 关键字段 | 说明 |
|------|----------|------|
| AgentApp | dev_mode | 开发模式：low-code / pro-code |
| ProCodeConfig | git_repo_id, tech_stack | 关联 Git 仓库、技术栈 |
| GitRepo | url, credential_type | 仓库地址、认证方式 |
| Iteration | app_id, source_branch, status | 关联应用、来源分支、迭代状态 |
| CodeVersion | version_name, tag, commit_hash | 版本标识、Git Tag、commit |
| Pipeline | type, parent_id, stage, context | 流水线类型、父流水线、当前阶段、上下文 |

> 完整字段定义详见详细设计（LLD）。

---

## 7. 主要风险

| 风险 | 影响 | 应对 |
|------|------|------|
| 外部 GitLab 连接不稳定 | 代码拉取失败、流水线中断 | 增加重试机制、超时配置 |
| 流水线执行时间过长 | 资源占用、用户体验差 | 异步执行 + 状态通知 |
| 部署异步状态丢失 | 流水线卡死 | 轮询 + 回调双保险 |
| 打 Tag 失败 | 版本创建失败 | 事务回滚，记录错误日志 |

---

## 8. 一期范围

### 包含

- [x] 应用管理支持 pro-code 模式
- [x] Git 仓库对接（外部 GitLab）
- [x] 迭代管理（简化：只有发布阶段）
- [x] 流水线（CODE_CHECK → BUILD_IMAGE → DEPLOY）
- [x] 脚手架模板（内置 python）

### 不包含

- [ ] 运维管理实现（仅接口定义）
- [ ] Webhook 配置入口（保留设计）
- [ ] WebIDE 集成（后续迭代）
- [ ] 自定义脚手架模板
- [ ] 流水线配置自定义
- [ ] 权限控制（应用成员即可操作）
- [ ] 安全性设计（详见 LLD）
- [ ] 可观测性设计（详见 LLD）

---

> [!note] 评审记录
> - 日期：
> - 参与人：
> - 结论：
