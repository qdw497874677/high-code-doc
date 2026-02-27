---
title: Pro-code 高代码开发模式技术概要设计
date: 2026-02-27
tags:
  - 概要设计
  - pro-code
  - AI Agent平台
status: draft
---

# Pro-code 高代码开发模式技术概要设计

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

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Agent 平台                             │
├─────────────┬─────────────────────┬─────────────────────────┤
│  应用管理    │     开发管理         │       运维管理          │
│  (升级)     │     (新增)          │       (接口定义)        │
├─────────────┼─────────────────────┼─────────────────────────┤
│ 低代码配置   │ Git仓库对接          │                         │
│ Pro-code配置│ 迭代/版本管理        │    deploy/create        │
│             │ 流水线管理           │    deploy/status        │
│             │                      │    deploy/callback      │
└─────────────┴─────────────────────┴─────────────────────────┘
         │              │                      │
         ▼              ▼                      ▼
    [元数据存储]   [外部GitLab]           [K8s基础设施]
                  [WebIDE服务]
```

### 技术选型

| 组件 | 技术 | 说明 |
|------|------|------|
| 后端框架 | Java + Spring Boot | 复用现有技术栈 |
| Git 操作 | JGit / GitLab4J API | 对接外部 GitLab |
| 流水线 | Spring State Machine | 持久化 + 断点续跑 |
| 存储 | MySQL | 复用现有数据库 |

### 关键设计决策

1. **单体扩展**：在现有服务中增加高代码模块，共用数据库和基础能力
2. **高/低代码共存不可切换**：应用创建时确定模式，后续不可更改
3. **运维接口先行**：一期定义接口规范，实现留白

---

## 3. 模块设计

### 3.1 应用管理模块（升级）

在现有低代码应用管理基础上，增加 pro-code 配置能力。

```
┌─────────────────────────────────────────────────────────────┐
│                        应用管理模块                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Agent应用    │  │ 低代码配置   │  │ Pro-code配置        │  │
│  │ - id        │  │ - 流程配置   │  │ - git_repo_url      │  │
│  │ - name      │  │ - 节点配置   │  │ - tech_stack        │  │
│  │ - dev_mode  │  │ - 知识库     │  │ - scaffold_template │  │
│  │             │  │             │  │ - webhook_secret    │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**dev_mode 取值**：
- `low-code`：低代码模式
- `pro-code`：高代码模式

### 3.2 开发管理模块（新增）

```
┌─────────────────────────────────────────────────────────────┐
│                        开发管理模块                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Git仓库     │  │ 迭代管理    │  │ 流水线              │  │
│  │ - repo_id   │  │ - iter_id   │  │ - pipeline_id       │  │
│  │ - app_id    │  │ - app_id    │  │ - iter_id           │  │
│  │ - external_ │  │ - name      │  │ - stage             │  │
│  │   git_info  │  │ - source_   │  │ - status            │  │
│  │ - webhook   │  │   branch    │  │ - state_machine_id  │  │
│  └─────────────┘  │ - phases[]  │  │ - checkpoints       │  │
│                   └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 运维管理模块（接口定义）

一期只定义接口规范，实现留白。

---

## 4. 迭代与阶段模型

### 迭代生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                     迭代生命周期                             │
│                                                             │
│   创建迭代 ──► 开发中 ──► 测试阶段 ──► 预发阶段 ──► 发布阶段  │
│      │                      │          │           │       │
│      ▼                      ▼          ▼           ▼       │
│  选择来源分支           拉测试分支   关联来源分支  打tag+发布  │
│                                                             │
│   一期简化：创建迭代 ──► 发布阶段（直接打tag）                 │
└─────────────────────────────────────────────────────────────┘
```

### 迭代数据模型

```yaml
Iteration:
  id: string
  app_id: string
  name: string
  status: DRAFT | DEVELOPING | TESTING | STAGING | RELEASING | RELEASED
  source_branch: string        # 来源分支
  current_phase: string        # 当前阶段
  phases:                      # 阶段列表
    - type: RELEASE            # 一期只有 RELEASE
      branch: "main"           # 关联分支
      tag: "v1.0.0"            # 阶段产出的tag
      status: PENDING | RUNNING | SUCCESS | FAILED
  created_at: timestamp
  updated_at: timestamp
```

### 一期简化

- 只支持 `RELEASE` 阶段
- 进入发布阶段时：直接对来源分支打 tag
- tag 格式：`v{major}.{minor}.{patch}` 或用户自定义

---

## 5. 流水线设计

### 流水线状态机

```
┌─────────────────────────────────────────────────────────────┐
│                   Pipeline State Machine                    │
│                                                             │
│   INIT ──► CODE_CHECK ──► BUILD ──► PACKAGE ──► DEPLOY     │
│     │          │            │          │           │       │
│     │       FAILED      FAILED     FAILED     ┌────┴────┐  │
│     │          │            │          │      │         │  │
│     └──────────┴────────────┴──────────┘   DEPLOYING  FAILED│
│                                               │    │        │
│                                            SUCCESS  │       │
│                                               │      │       │
│                                               └──────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 流水线阶段说明（一期）

| 阶段 | 说明 | 产出 |
|------|------|------|
| CODE_CHECK | 检查代码在仓库是否存在 | commit_id |
| BUILD | 获取基础镜像，拼接启动文件 | Dockerfile |
| PACKAGE | 构建镜像，生成部署产物 | 镜像地址 + 配置 |
| DEPLOY | 调用运维模块部署接口 | 部署记录 |

### DEPLOY 阶段子状态

| 子状态 | 说明 | 触发条件 |
|--------|------|----------|
| DEPLOYING | 部署中 | 调用运维模块接口成功 |
| DEPLOY_SUCCESS | 部署成功 | 运维模块回调或轮询确认 |
| DEPLOY_FAILED | 部署失败 | 运维模块回调失败或超时 |

### 部署异步交互流程

```
┌──────────┐      调用部署接口      ┌──────────┐
│ Pipeline │ ────────────────────► │ 运维模块  │
│          │                       │          │
│          │ ◄──────────────────── │          │
│          │   返回 deploy_id      │          │
│          │                       │          │
│          │      轮询/回调        │          │
│          │ ◄───────────────────► │          │
│          │   部署状态变更        │          │
└──────────┘                       └──────────┘
```

### 流水线数据模型

```yaml
Pipeline:
  id: string
  iter_id: string
  status: PENDING | RUNNING | SUCCESS | FAILED | PAUSED
  current_stage: CODE_CHECK | BUILD | PACKAGE | DEPLOY
  deploy_status: DEPLOYING | DEPLOY_SUCCESS | DEPLOY_FAILED
  deploy_id: string           # 运维模块返回的部署ID
  checkpoints:                # 状态机持久化检查点
    - stage: CODE_CHECK
      status: SUCCESS
      output: { commit_id: "abc123" }
    - stage: BUILD
      status: SUCCESS
      output: { dockerfile: "..." }
    - stage: PACKAGE
      status: SUCCESS
      output: { image: "registry/xxx:v1.0", config: {...} }
  created_at: timestamp
  updated_at: timestamp
```

### 断点续跑机制

- 每个 stage 完成后持久化 checkpoint
- 流水线中断后恢复时，从最后一个成功的 checkpoint 继续
- 使用 Spring State Machine 的 `StateMachinePersister`

### 超时机制

- 部署超时：可配置，默认 30 分钟
- 超时后自动标记 DEPLOY_FAILED

---

## 6. 接口清单

### 应用管理模块（升级）

| 接口 | 方法 | 说明 |
|------|------|------|
| /apps | POST | 创建应用（含 dev_mode） |
| /apps/{id} | GET | 获取应用详情 |
| /apps/{id}/procode-config | PUT | 配置 pro-code 参数 |

### 开发管理模块（新增）

| 接口 | 方法 | 说明 |
|------|------|------|
| /repos | POST | 创建 Git 仓库配置 |
| /repos/{id}/webhook | GET | 获取 webhook 配置 |
| /iterations | POST | 创建迭代 |
| /iterations/{id} | GET | 获取迭代详情 |
| /iterations/{id}/release | POST | 进入发布阶段（打 tag） |
| /pipelines | POST | 创建流水线 |
| /pipelines/{id} | GET | 获取流水线状态 |
| /pipelines/{id}/resume | POST | 断点续跑 |
| /pipelines/{id}/cancel | POST | 取消流水线 |

### 运维管理模块（接口定义，实现留白）

| 接口 | 方法 | 说明 |
|------|------|------|
| /deploy/create | POST | 创建部署任务 |
| /deploy/{id}/status | GET | 查询部署状态 |
| /deploy/callback | POST | 部署状态回调 |

---

## 7. 数据库表设计

### 核心表结构

```sql
-- 应用表（升级）
ALTER TABLE agent_app ADD COLUMN dev_mode VARCHAR(20) DEFAULT 'low-code';
ALTER TABLE agent_app ADD COLUMN tech_stack VARCHAR(50);

-- Pro-code 配置表
CREATE TABLE procode_config (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    app_id BIGINT NOT NULL,
    git_repo_url VARCHAR(500) NOT NULL,
    scaffold_template VARCHAR(100),
    webhook_secret VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_app_id (app_id)
);

-- Git 仓库表
CREATE TABLE git_repo (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    app_id BIGINT NOT NULL,
    external_git_url VARCHAR(500) NOT NULL,
    git_type VARCHAR(20),
    credential_id VARCHAR(100),
    webhook_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_app_id (app_id)
);

-- 迭代表
CREATE TABLE iteration (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    app_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'DRAFT',
    source_branch VARCHAR(100) NOT NULL,
    current_phase VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 迭代阶段表
CREATE TABLE iteration_phase (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    iteration_id BIGINT NOT NULL,
    phase_type VARCHAR(20) NOT NULL,
    branch VARCHAR(100),
    tag VARCHAR(100),
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 流水线表
CREATE TABLE pipeline (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    iteration_id BIGINT NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    current_stage VARCHAR(20),
    deploy_status VARCHAR(20),
    deploy_id VARCHAR(100),
    checkpoints TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## 8. 主要风险

| 风险 | 影响 | 应对 |
|------|------|------|
| 外部 GitLab 连接不稳定 | 代码拉取失败、流水线中断 | 增加重试机制、超时配置 |
| 流水线执行时间过长 | 资源占用、用户体验差 | 异步执行 + 状态通知 |
| WebIDE 对接延迟 | 开发环境不可用 | 接口超时降级、状态检查 |
| 部署异步状态丢失 | 流水线卡死 | 轮询 + 回调双保险 |

---

## 9. 后续规划

### 一期范围

- [x] 应用管理支持 pro-code 模式
- [x] Git 仓库对接（外部 GitLab）
- [x] 迭代管理（简化：只有发布阶段）
- [x] 流水线（CODE_CHECK → BUILD → PACKAGE → DEPLOY）
- [ ] 运维管理（接口定义，实现留白）

### 二期规划

- [ ] 多阶段迭代（测试、预发、发布）
- [ ] 多技术栈支持（Java、Node.js）
- [ ] 运维管理实现
- [ ] 多环境部署
