---
title: Pro-code 高代码开发模式详细设计
date: 2026-02-27
tags:
  - 详细设计
  - pro-code
  - AI Agent平台
status: draft
---

# Pro-code 高代码开发模式详细设计

## 1. 模块设计

### 1.1 模块划分

```
com.example.platform
├── app                          # 应用管理模块
│   ├── controller
│   │   └── AppController.java
│   ├── service
│   │   ├── AppService.java
│   │   └── ProCodeConfigService.java
│   ├── entity
│   │   ├── AgentApp.java
│   │   └── ProCodeConfig.java
│   └── repository
│       └── AppRepository.java
│
├── dev                          # 开发管理模块
│   ├── controller
│   │   ├── RepoController.java
│   │   ├── IterationController.java
│   │   └── PipelineController.java
│   ├── service
│   │   ├── GitService.java
│   │   ├── IterationService.java
│   │   └── PipelineService.java
│   ├── entity
│   │   ├── GitRepo.java
│   │   ├── Iteration.java
│   │   ├── IterationPhase.java
│   │   └── Pipeline.java
│   ├── statemachine
│   │   ├── PipelineStateMachine.java
│   │   ├── PipelineState.java
│   │   ├── PipelineEvent.java
│   │   └── PipelineAction.java
│   └── integration
│       ├── GitLabClient.java
│       └── WebIdeClient.java
│
└── ops                          # 运维管理模块（接口定义）
    ├── controller
    │   └── DeployController.java
    ├── dto
    │   ├── DeployRequest.java
    │   ├── DeployResponse.java
    │   └── DeployStatusResponse.java
    └── service
        └── DeployService.java   # 空实现，二期补充
```

### 1.2 核心类设计

**应用管理：**

```java
// AgentApp.java
@Entity
public class AgentApp {
    private Long id;
    private String name;
    private DevMode devMode;        // LOW_CODE, PRO_CODE
    private String techStack;       // python, java, nodejs
}

// ProCodeConfig.java
@Entity
public class ProCodeConfig {
    private Long appId;
    private String gitRepoUrl;
    private String scaffoldTemplate;
    private String webhookSecret;
}
```

**开发管理：**

```java
// GitRepo.java
@Entity
public class GitRepo {
    private Long id;
    private Long appId;
    private String externalGitUrl;
    private GitType gitType;        // GITLAB, GITHUB, GITEE
    private String credentialId;
    private String webhookUrl;
}

// Iteration.java
@Entity
public class Iteration {
    private Long id;
    private Long appId;
    private String name;
    private IterationStatus status;
    private String sourceBranch;
    private String currentPhase;
}

// Pipeline.java（支持两种类型）
@Entity
public class Pipeline {
    private Long id;
    private PipelineType type;          // ITERATION / PHASE
    private Long parentId;              // 阶段流水线关联迭代流水线ID
    private Long refId;                 // 迭代流水线关联迭代ID
    private PipelineStatus status;
    private String currentStage;        // 迭代流水线：TESTING/STAGING/RELEASE
                                        // 阶段流水线：CODE_CHECK/BUILD/PACKAGE/DEPLOY
    private DeployStatus deployStatus;  // 仅阶段流水线使用
    private String deployId;            // 仅阶段流水线使用
    private String checkpoints;         // JSON
}
```

---

## 2. 流程设计

### 2.1 应用创建时序图

```mermaid
sequenceDiagram
    participant U as 用户
    participant AC as AppController
    participant AS as AppService
    participant GS as GitService
    participant DB as 数据库
    participant GL as GitLab

    U->>AC: POST /apps
    AC->>AS: createApp(request)
    AS->>DB: 保存 AgentApp
    alt pro-code模式
        AS->>GS: initGitRepo(appId, gitUrl)
        GS->>GL: 创建 webhook
        GL-->>GS: webhook_url
        GS->>DB: 保存 GitRepo
    end
    AS-->>AC: AppResponse
    AC-->>U: 应用创建成功
```

### 2.2 迭代发布时序图

```mermaid
sequenceDiagram
    participant U as 用户
    participant IC as IterationController
    participant IS as IterationService
    participant PS as PipelineService
    participant GL as GitLab
    participant DB as 数据库

    U->>IC: POST /iterations/{id}/release
    IC->>IS: release(iterationId)
    IS->>DB: 查询迭代
    IS->>GL: 创建 tag
    GL-->>IS: tag创建成功
    IS->>DB: 更新迭代状态
    IS->>PS: createPipeline(iterationId, tag)
    PS->>DB: 创建流水线
    PS->>PS: 启动状态机
    PS-->>IS: pipelineId
    IS-->>IC: ReleaseResponse
    IC-->>U: 发布流程启动
```

### 2.3 流水线状态机

**两种流水线类型复用同一套状态机能力：**

#### 迭代流水线状态机

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> DEVELOPING: START
    DEVELOPING --> TESTING: ENTER_TESTING
    TESTING --> STAGING: TESTING_SUCCESS
    STAGING --> RELEASE: STAGING_SUCCESS
    RELEASE --> RELEASED: RELEASE_SUCCESS
    RELEASED --> [*]
    
    note right of RELEASE
        进入发布阶段时
        创建阶段流水线
    end note
```

#### 阶段流水线状态机

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> RUNNING: START
    RUNNING --> CODE_CHECK: ENTER_CODE_CHECK
    CODE_CHECK --> BUILD: CODE_CHECK_SUCCESS
    CODE_CHECK --> FAILED: CODE_CHECK_FAILED
    BUILD --> PACKAGE: BUILD_SUCCESS
    BUILD --> FAILED: BUILD_FAILED
    PACKAGE --> DEPLOY: PACKAGE_SUCCESS
    PACKAGE --> FAILED: PACKAGE_FAILED
    DEPLOY --> DEPLOYING: DEPLOY_START
    DEPLOYING --> SUCCESS: DEPLOY_SUCCESS
    DEPLOYING --> FAILED: DEPLOY_FAILED
    FAILED --> [*]
    SUCCESS --> [*]
    
    note right of DEPLOYING
        长期异步阶段
        支持轮询/回调
    end note
```

#### 流水线类型对比

| 属性 | 迭代流水线 | 阶段流水线 |
|------|-----------|-----------|
| type | ITERATION | PHASE |
| 阶段 | DEVELOPING/TESTING/STAGING/RELEASE | CODE_CHECK/BUILD/PACKAGE/DEPLOY |
| parentId | null | 迭代流水线ID |
| refId | 迭代ID | 阶段ID |
| 部署相关字段 | 不使用 | 使用 |

#### 一期简化

迭代流水线只有 RELEASE 一个阶段，进入时自动创建阶段流水线。

### 2.4 部署异步交互时序图

```mermaid
sequenceDiagram
    participant PS as PipelineService
    participant DS as DeployService
    participant OPS as 运维模块
    participant DB as 数据库

    PS->>DS: deploy(image, config)
    DS->>OPS: POST /deploy/create
    OPS-->>DS: deploy_id
    DS->>DB: 更新 deploy_status=DEPLOYING
    
    loop 轮询/等待回调
        DS->>OPS: GET /deploy/{id}/status
        OPS-->>DS: status
        alt 部署中
            DS->>DB: 状态保持 DEPLOYING
        else 部署成功
            DS->>DB: 更新 deploy_status=DEPLOY_SUCCESS
            DS-->>PS: 触发状态机事件
        else 部署失败
            DS->>DB: 更新 deploy_status=DEPLOY_FAILED
            DS-->>PS: 触发状态机事件
        end
    end
```

### 2.5 流水线推进驱动机制

**统一的阶段配置模型：**

所有流水线阶段使用统一的配置模型，通过`driveMode`区分三种驱动方式。

#### 阶段配置数据结构

```java
// StageConfig.java
public class StageConfig {
    private String stage;              // 阶段名称
    private DriveMode driveMode;       // MANUAL / AUTO / ASYNC
    private RetryMode retryMode;       // AUTO / MANUAL
    private int maxRetries;            // 自动重试最大次数
    private int currentRetries;        // 当前已重试次数
    
    // 仅 ASYNC 模式使用
    private int pollInterval;          // 轮询间隔（秒）
    private int pollTimeout;           // 超时时间（秒）
}

// DriveMode.java
public enum DriveMode {
    MANUAL,    // 手动推进
    AUTO,      // 自动推进
    ASYNC      // 异步驱动（轮询+回调）
}
```

#### 迭代流水线默认配置

迭代流水线所有阶段默认配置为 **MANUAL**（手动推进）。

```java
// 迭代流水线默认配置
Map<String, StageConfig> iterationDefaults = Map.of(
    "DEVELOPING", new StageConfig("DEVELOPING", MANUAL, MANUAL, 0),
    "TESTING", new StageConfig("TESTING", MANUAL, MANUAL, 0),
    "STAGING", new StageConfig("STAGING", MANUAL, MANUAL, 0),
    "RELEASE", new StageConfig("RELEASE", MANUAL, MANUAL, 0)
);
```

#### 阶段流水线默认配置

```java
// 阶段流水线默认配置
Map<String, StageConfig> phaseDefaults = Map.of(
    "CODE_CHECK", new StageConfig("CODE_CHECK", AUTO, AUTO, 3),
    "BUILD", new StageConfig("BUILD", AUTO, AUTO, 2),
    "PACKAGE", new StageConfig("PACKAGE", AUTO, MANUAL, 0),
    "DEPLOY", new StageConfig("DEPLOY", ASYNC, MANUAL, 0, 30, 1800)
);
```

#### 手动推进时序图（MANUAL）

适用于迭代流水线所有阶段，以及阶段流水线中配置为MANUAL的阶段。

```mermaid
sequenceDiagram
    participant U as 用户
    participant PS as PipelineService
    participant SM as StateMachine
    participant DB as 数据库

    U->>PS: POST /pipelines/{id}/proceed
    PS->>DB: 查询流水线状态
    PS->>PS: 校验是否可推进
    alt 可推进
        PS->>SM: 触发下一阶段事件
        SM->>DB: 更新流水线状态
        PS-->>U: 推进成功
    else 不可推进
        PS-->>U: 推进失败
    end
```

#### 自动推进时序图（AUTO）

适用于阶段流水线的 CODE_CHECK、BUILD、PACKAGE 阶段。

```mermaid
sequenceDiagram
    participant SM as StateMachine
    participant PS as PipelineService
    participant SC as StageConfig
    participant DB as 数据库

    SM->>PS: 阶段执行完成(status)
    PS->>SC: 获取阶段配置
    
    alt 阶段成功 && driveMode=AUTO
        PS->>SM: 触发下一阶段事件
        SM->>DB: 更新流水线状态
    else 阶段成功 && driveMode=MANUAL
        PS->>DB: 更新状态，等待手动触发
    else 阶段失败
        alt retryMode=AUTO && currentRetries < maxRetries
            PS->>SC: currentRetries++
            PS->>DB: 更新重试次数
            PS->>SM: 触发当前阶段重试
        else retryMode=MANUAL
            PS->>DB: 更新状态，等待手动重试
        end
    end
```

#### 异步驱动时序图（ASYNC）

适用于阶段流水线的 DEPLOY 阶段，采用轮询+回调双保险机制。

**轮询任务时序图：**

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant PS as PipelineService
    participant DS as DeployService
    participant OPS as 运维模块
    participant DB as 数据库

    loop 每隔pollInterval
        S->>PS: 检查DEPLOYING状态的流水线
        PS->>DS: 查询部署状态
        DS->>OPS: GET /deploy/{id}/status
        OPS-->>DS: status
        
        alt 部署成功
            DS->>DB: 更新deploy_status=DEPLOY_SUCCESS
            DS->>PS: 触发状态机事件(SUCCESS)
        else 部署失败
            DS->>DB: 更新deploy_status=DEPLOY_FAILED
            DS->>PS: 触发状态机事件(FAILED)
        else 超时
            DS->>DB: 更新deploy_status=DEPLOY_TIMEOUT
            DS->>PS: 触发状态机事件(FAILED)
        else 部署中
            Note over DS: 继续等待
        end
    end
```

**回调接口时序图：**

```mermaid
sequenceDiagram
    participant OPS as 运维模块
    participant DS as DeployService
    participant PS as PipelineService
    participant DB as 数据库

    OPS->>DS: POST /deploy/callback
    DS->>DB: 查询deploy_id
    alt 存在
        DS->>DB: 更新deploy_status
        DS->>PS: 触发状态机事件
        DS-->>OPS: 200 OK
    else 不存在
        DS-->>OPS: 404 Not Found
    end
```

---

## 3. 接口定义

### 3.1 创建应用

- **路径**：`POST /apps`
- **描述**：创建应用（低代码或高代码）

**请求参数**：

| 参数 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| name | String | 是 | 1-100字符 | 应用名称 |
| devMode | String | 是 | 枚举: low-code, pro-code | 开发模式 |
| techStack | String | pro-code必填 | 枚举: python | 技术栈 |
| gitRepoUrl | String | pro-code必填 | URL格式 | Git仓库地址 |
| scaffoldTemplate | String | 否 | - | 脚手架模板 |

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 123,
    "name": "my-agent",
    "devMode": "pro-code",
    "techStack": "python"
  }
}
```

**错误码**：

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 1001 | 参数错误 |
| 1002 | 应用名称已存在 |
| 1003 | Git仓库不可访问 |

### 3.2 配置 Pro-code

- **路径**：`PUT /apps/{id}/procode-config`
- **描述**：配置 pro-code 参数

**请求参数**：

| 参数 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| gitRepoUrl | String | 是 | URL格式 | Git仓库地址 |
| scaffoldTemplate | String | 否 | - | 脚手架模板 |
| credentialId | String | 是 | - | Git凭证ID |

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "appId": 123,
    "gitRepoUrl": "https://gitlab.example.com/my-agent.git",
    "webhookUrl": "https://platform.example.com/webhook/xxx"
  }
}
```

### 3.3 创建迭代

- **路径**：`POST /iterations`
- **描述**：创建迭代

**请求参数**：

| 参数 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| appId | Long | 是 | - | 应用ID |
| name | String | 是 | 1-100字符 | 迭代名称 |
| sourceBranch | String | 是 | - | 来源分支 |

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 456,
    "appId": 123,
    "name": "v1.0.0",
    "status": "DRAFT",
    "sourceBranch": "main"
  }
}
```

### 3.4 进入发布阶段

- **路径**：`POST /iterations/{id}/release`
- **描述**：进入发布阶段，打tag并触发流水线

**请求参数**：

| 参数 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| tag | String | 是 | v开头的版本号 | 发布版本tag |

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "iterationId": 456,
    "tag": "v1.0.0",
    "pipelineId": 789,
    "status": "RUNNING"
  }
}
```

### 3.5 获取流水线状态

- **路径**：`GET /pipelines/{id}`
- **描述**：获取流水线执行状态

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 789,
    "type": "PHASE",
    "parentId": 888,
    "refId": 456,
    "status": "RUNNING",
    "currentStage": "DEPLOY",
    "deployStatus": "DEPLOYING",
    "checkpoints": [
      {
        "stage": "CODE_CHECK",
        "status": "SUCCESS",
        "output": {"commitId": "abc123"}
      },
      {
        "stage": "BUILD",
        "status": "SUCCESS",
        "output": {"dockerfile": "..."}
      },
      {
        "stage": "PACKAGE",
        "status": "SUCCESS",
        "output": {"image": "registry/xxx:v1.0.0"}
      }
    ],
    "createdAt": "2026-02-27T10:00:00Z",
    "updatedAt": "2026-02-27T10:05:00Z"
  }
}
```

### 3.6 断点续跑

- **路径**：`POST /pipelines/{id}/resume`
- **描述**：从中断点恢复流水线执行

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "pipelineId": 789,
    "status": "RUNNING",
    "resumeFromStage": "DEPLOY"
  }
}
```

### 3.7 迭代流水线推进

- **路径**：`POST /iterations/{id}/proceed`
- **描述**：手动推进迭代流水线到下一阶段

**请求参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| targetPhase | String | 是 | 目标阶段: TESTING/STAGING/RELEASE |

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "iterationId": 456,
    "currentPhase": "RELEASE",
    "phasePipelineId": 789
  }
}
```

**错误码**：

| 错误码 | 说明 |
|--------|------|
| 2001 | 当前阶段不可推进 |
| 2002 | 目标阶段无效 |
| 2003 | 前置条件不满足 |

### 3.8 阶段流水线重试

- **路径**：`POST /pipelines/{id}/retry`
- **描述**：手动重试当前失败阶段

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "pipelineId": 789,
    "currentStage": "BUILD",
    "retryCount": 2,
    "maxRetries": 3
  }
}
```

**错误码**：

| 错误码 | 说明 |
|--------|------|
| 3001 | 流水线状态不允许重试 |
| 3002 | 已达到最大重试次数 |

### 3.9 阶段流水线推进

- **路径**：`POST /pipelines/{id}/proceed`
- **描述**：手动推进阶段流水线到下一阶段（当autoProceed=false时使用）

**响应格式**：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "pipelineId": 789,
    "previousStage": "PACKAGE",
    "currentStage": "DEPLOY"
  }
}
```

### 3.10 运维模块接口（定义，实现留白）

**创建部署**：`POST /deploy/create`

```json
// 请求
{
  "appId": 123,
  "image": "registry/xxx:v1.0.0",
  "config": {
    "env": {"KEY": "VALUE"},
    "resources": {"cpu": "1", "memory": "2Gi"}
  }
}

// 响应
{
  "code": 0,
  "data": {
    "deployId": "deploy-xxx",
    "status": "PENDING"
  }
}
```

**查询部署状态**：`GET /deploy/{id}/status`

```json
{
  "code": 0,
  "data": {
    "deployId": "deploy-xxx",
    "status": "RUNNING",
    "message": "正在启动容器..."
  }
}
```

**部署回调**：`POST /deploy/callback`

```json
{
  "deployId": "deploy-xxx",
  "status": "SUCCESS",
  "message": "部署成功",
  "timestamp": "2026-02-27T10:10:00Z"
}
```

---

## 4. 数据设计

### 4.1 表结构

**agent_app（应用表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| name | VARCHAR(100) | NOT NULL, UNIQUE | 应用名称 |
| dev_mode | VARCHAR(20) | NOT NULL, DEFAULT 'low-code' | 开发模式 |
| tech_stack | VARCHAR(50) | | 技术栈 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

**procode_config（Pro-code配置表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| app_id | BIGINT | NOT NULL, UNIQUE | 应用ID |
| git_repo_url | VARCHAR(500) | NOT NULL | Git仓库地址 |
| scaffold_template | VARCHAR(100) | | 脚手架模板 |
| webhook_secret | VARCHAR(100) | | Webhook密钥 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

**git_repo（Git仓库表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| app_id | BIGINT | NOT NULL, UNIQUE | 应用ID |
| external_git_url | VARCHAR(500) | NOT NULL | 外部Git地址 |
| git_type | VARCHAR(20) | | Git类型(gitlab/github/gitee) |
| credential_id | VARCHAR(100) | | 凭证ID |
| webhook_url | VARCHAR(500) | | Webhook地址 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**iteration（迭代表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| app_id | BIGINT | NOT NULL | 应用ID |
| name | VARCHAR(100) | NOT NULL | 迭代名称 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'DRAFT' | 状态 |
| source_branch | VARCHAR(100) | NOT NULL | 来源分支 |
| current_phase | VARCHAR(20) | | 当前阶段 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

**iteration_phase（迭代阶段表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| iteration_id | BIGINT | NOT NULL | 迭代ID |
| phase_type | VARCHAR(20) | NOT NULL | 阶段类型(RELEASE) |
| branch | VARCHAR(100) | | 关联分支 |
| tag | VARCHAR(100) | | 版本tag |
| status | VARCHAR(20) | DEFAULT 'PENDING' | 状态 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |

**pipeline（流水线表）**

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 主键 |
| type | VARCHAR(20) | NOT NULL | 流水线类型(ITERATION/PHASE) |
| parent_id | BIGINT | | 父流水线ID（阶段流水线关联迭代流水线） |
| ref_id | BIGINT | NOT NULL | 关联ID（迭代流水线关联迭代ID，阶段流水线关联阶段ID） |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'PENDING' | 状态 |
| current_stage | VARCHAR(50) | | 当前阶段 |
| stage_configs | TEXT | | 阶段配置(JSON)，仅阶段流水线 |
| current_retries | INT | DEFAULT 0 | 当前阶段重试次数，仅阶段流水线 |
| deploy_status | VARCHAR(20) | | 部署子状态（仅阶段流水线） |
| deploy_id | VARCHAR(100) | | 部署ID（仅阶段流水线） |
| checkpoints | TEXT | | 检查点(JSON) |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | ON UPDATE CURRENT_TIMESTAMP | 更新时间 |

**stage_configs JSON 结构示例：**

**迭代流水线默认配置：**

```json
{
  "DEVELOPING": {
    "driveMode": "MANUAL",
    "retryMode": "MANUAL",
    "maxRetries": 0
  },
  "TESTING": {
    "driveMode": "MANUAL",
    "retryMode": "MANUAL",
    "maxRetries": 0
  },
  "STAGING": {
    "driveMode": "MANUAL",
    "retryMode": "MANUAL",
    "maxRetries": 0
  },
  "RELEASE": {
    "driveMode": "MANUAL",
    "retryMode": "MANUAL",
    "maxRetries": 0
  }
}
```

**阶段流水线默认配置：**

```json
{
  "CODE_CHECK": {
    "driveMode": "AUTO",
    "retryMode": "AUTO",
    "maxRetries": 3
  },
  "BUILD": {
    "driveMode": "AUTO",
    "retryMode": "AUTO",
    "maxRetries": 2
  },
  "PACKAGE": {
    "driveMode": "AUTO",
    "retryMode": "MANUAL",
    "maxRetries": 0
  },
  "DEPLOY": {
    "driveMode": "ASYNC",
    "retryMode": "MANUAL",
    "maxRetries": 0,
    "pollInterval": 30,
    "pollTimeout": 1800
  }
}
```

### 4.2 索引设计

| 表 | 索引名 | 字段 | 类型 | 说明 |
|------|--------|------|------|------|
| agent_app | uk_name | name | UNIQUE | 应用名称唯一 |
| procode_config | uk_app_id | app_id | UNIQUE | 应用ID唯一 |
| git_repo | uk_app_id | app_id | UNIQUE | 应用ID唯一 |
| iteration | idx_app_id | app_id | INDEX | 按应用查询迭代 |
| iteration_phase | idx_iteration_id | iteration_id | INDEX | 按迭代查询阶段 |
| pipeline | idx_ref_id | ref_id | INDEX | 按关联ID查询流水线 |
| pipeline | idx_parent_id | parent_id | INDEX | 查询阶段流水线 |
| pipeline | idx_status | status | INDEX | 按状态查询 |
| pipeline | idx_type | type | INDEX | 按类型查询 |

---

## 5. 非功能需求

### 5.1 性能指标

| 指标 | 目标值 |
|------|--------|
| 接口响应时间 | < 200ms (P99) |
| 流水线并发数 | >= 100 |
| 迭代查询 | < 100ms |
| Git操作超时 | 30s |

### 5.2 可靠性

- 流水线断点续跑：状态机持久化，支持从中断点恢复
- Git操作重试：失败后自动重试3次，间隔递增
- 部署超时：默认30分钟，可配置

### 5.3 安全

- Git凭证加密存储
- Webhook签名验证
- API访问鉴权

---

## 6. 降级 & 回滚

### 6.1 降级策略

| 场景 | 降级方案 |
|------|----------|
| GitLab不可用 | 返回缓存数据，提示稍后重试 |
| WebIDE不可用 | 提示用户使用本地IDE |
| 运维模块不可用 | 流水线停在DEPLOY阶段，等待恢复 |
| 流水线执行失败 | 支持断点续跑，从失败阶段继续 |

### 6.2 回滚方案

- 迭代回滚：重新发布上一个版本的tag
- 流水线回滚：取消当前流水线，重新创建
- 数据回滚：通过数据库备份恢复

---

## 变更记录

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|----------|------|
| 2026-02-27 | v1.0 | 初稿 | - |
