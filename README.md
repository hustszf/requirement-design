# CCOC 自动化运营包配置系统 — 需求设计

## 一、项目背景

CCOC 用户运营平台已具备三类核心数据：
- **用户数据**：用户基本信息、行为数据、标签体系
- **设备数据**：终端型号、网络状态、在线状态
- **权益数据**：权益发放记录、使用状态、有效期

现需新增**自动化运营包配置**能力，支持运营人员通过可视化配置，自动筛选目标用户、生成运营包并触达用户，减少人工操作，提升运营效率。

## 二、核心需求梳理

### 2.1 配置基础信息

| 字段 | 说明 | 示例 |
|------|------|------|
| 配置名称 | 唯一标识该运营配置 | "权益未使用提醒-外呼" |
| 配置描述 | 简要说明配置目的 | "针对权益发放超3天未使用的用户进行外呼提醒" |
| 配置状态 | 启用 / 停用 | 启用 |
| 生效时间 | 配置生效的时间范围 | 2026-10-23 ~ 2026-12-31 |

### 2.2 数据筛选规则（运营包数据来源）

支持从三类数据源中，按条件筛选目标用户：

**数据源选择：**
- 用户数据（用户属性、行为特征）
- 设备数据（终端信息、网络状态）
- 权益数据（权益类型、发放时间、使用状态）

**筛选条件配置：**
- 支持多条件组合（AND / OR）
- 每个条件：字段 + 运算符 + 值
- 运算符：等于、不等于、大于、小于、包含、不包含、在范围内、为空、不为空

**示例规则：**
```
权益数据.发放时间 < 当前时间 - 3天
AND 权益数据.使用状态 = 未使用
AND 用户数据.用户状态 = 正常
```

### 2.3 运营包生成设置

| 字段 | 说明 | 示例 |
|------|------|------|
| 生成数量上限 | 单次运营包最大记录数 | 1000 |
| 执行频率 | 一次性 / 每天 / 每周 / 自定义Cron | 每天 |
| 执行时间 | 具体执行时间点 | 09:00 |
| 去重策略 | 同一用户重复命中的处理 | 7天内不重复触达 |
| 优先级 | 多个配置命中同一用户时的优先级 | 高 |

### 2.4 触达方式配置

**支持的触达渠道：**

| 渠道 | 说明 |
|------|------|
| 智能外呼 | AI 自动外呼，支持话术模板 |
| 短信 | 短信推送，需选择短信模板 |
| APP 推送 | 应用内消息推送 |
| 微信消息 | 公众号 / 服务号消息推送 |
| 人工坐席 | 分配至人工坐席跟进 |

**触达配置项：**
- 触达模板选择（从模板库中选取）
- 触达时段限制（如 09:00-21:00）
- 触达频次控制（如单用户每天最多1次）
- 失败重试策略（重试次数、间隔）

### 2.5 工单配置

| 字段 | 说明 | 示例 |
|------|------|------|
| 是否创建工单 | 开关控制 | 是 |
| 工单场景 | 预设的工单场景类型 | 权益激活提醒 / 套餐续约 / 故障回访 |
| 工单优先级 | 紧急 / 高 / 中 / 低 | 中 |
| 工单分配规则 | 自动分配 / 指定团队 / 指定人员 | 自动分配 |
| 工单关联 | 关联触达记录 | 是 |
| 工单超时设置 | 工单处理超时时间 | 24小时 |

### 2.6 执行与监控

- 运营包生成后支持人工审核再执行
- 实时查看运营包执行进度
- 触达成功率、工单处理率等数据看板
- 异常告警（生成失败、触达异常）

## 三、配置流程

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  基础信息    │ -> │  筛选规则    │ -> │  生成设置    │ -> │  触达配置    │ -> │  工单配置    │
│  配置        │    │  配置        │    │  配置        │    │  配置        │    │  (可选)      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                                                    │
                                                                                    v
                                                                            ┌─────────────┐
                                                                            │  预览 & 保存  │
                                                                            └─────────────┘
```

## 四、数据模型

### 运营包配置表 (auto_operation_config)

```sql
CREATE TABLE auto_operation_config (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    config_name     VARCHAR(100) NOT NULL COMMENT '配置名称',
    config_desc     VARCHAR(500) COMMENT '配置描述',
    status          TINYINT DEFAULT 1 COMMENT '状态: 0-停用 1-启用',
    start_time      DATETIME COMMENT '生效开始时间',
    end_time        DATETIME COMMENT '生效结束时间',
    filter_rules    JSON NOT NULL COMMENT '筛选规则JSON',
    max_count       INT DEFAULT 1000 COMMENT '生成数量上限',
    exec_frequency  VARCHAR(50) COMMENT '执行频率: once/daily/weekly/cron',
    exec_time       VARCHAR(20) COMMENT '执行时间',
    exec_cron       VARCHAR(100) COMMENT '自定义Cron表达式',
    dedup_days      INT DEFAULT 7 COMMENT '去重天数',
    priority        TINYINT DEFAULT 2 COMMENT '优先级: 1-低 2-中 3-高 4-紧急',
    reach_channel   VARCHAR(50) NOT NULL COMMENT '触达渠道',
    reach_template  VARCHAR(100) COMMENT '触达模板ID',
    reach_time_start VARCHAR(10) COMMENT '触达开始时段',
    reach_time_end  VARCHAR(10) COMMENT '触达结束时段',
    reach_max_daily INT DEFAULT 1 COMMENT '单用户日最大触达次数',
    retry_count     INT DEFAULT 0 COMMENT '失败重试次数',
    retry_interval  INT DEFAULT 60 COMMENT '重试间隔(分钟)',
    create_ticket   TINYINT DEFAULT 0 COMMENT '是否创建工单: 0-否 1-是',
    ticket_scene    VARCHAR(100) COMMENT '工单场景',
    ticket_priority TINYINT DEFAULT 2 COMMENT '工单优先级',
    ticket_assign   VARCHAR(50) COMMENT '工单分配规则',
    ticket_timeout  INT DEFAULT 24 COMMENT '工单超时(小时)',
    need_review     TINYINT DEFAULT 1 COMMENT '是否需要审核: 0-否 1-是',
    created_by      VARCHAR(50) COMMENT '创建人',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 筛选规则 JSON 结构示例

```json
{
  "logic": "AND",
  "conditions": [
    {
      "dataSource": "entitlement",
      "field": "issue_date",
      "operator": "before_days",
      "value": 3
    },
    {
      "dataSource": "entitlement",
      "field": "usage_status",
      "operator": "equals",
      "value": "unused"
    },
    {
      "dataSource": "user",
      "field": "status",
      "operator": "equals",
      "value": "active"
    }
  ]
}
```

## 五、原型说明

交互原型文件：`prototype/index.html`

直接在浏览器中打开即可体验完整的配置流程，包含：
- 步骤式配置向导（5步）
- 动态筛选条件增删
- 触达方式与模板选择
- 工单配置开关与详情
- 配置预览与 JSON 导出
