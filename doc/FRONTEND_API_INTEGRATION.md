# 智能金融理财平台 - 前端对接文档

> 版本: 1.18.0 | 更新日期: 2025-12-19  
> 基于 Spring Cloud 微服务架构，统一通过 API Gateway 访问
> 
> **v1.18.0 更新**：新增图形验证码、短信验证码登录、账号自助解冻接口；新增顾问 KPI 看板接口  
> **v1.17.0 更新**：AI 对话默认流式输出 (SSE)，`/ai/chat/ask` 改为 GET 方法并返回 SSE 流，实现打字机效果  
> **v1.16.0 更新**：邮箱/手机验证机制优化 - 注册时不强制验证，登录和用户信息接口新增 `emailVerified`、`phoneVerified` 字段  
> **v1.15.0 更新**：Python数据服务集成 BaoStock + Tushare + AkShare 三数据源，移除所有模拟数据，新增港股/ETF/期货/板块等接口  
> **v1.14.0 更新**：员工编号支持系统自动生成（ADV+6位随机字符）；响应加密功能已可启用；清理5张冗余数据库表  
> **v1.13.0 更新**：重构顾问体系为 B2B 准入机制，顾问为独立专业身份；新增管理员创建/管理顾问账号接口  
> **v1.12.0 更新**：新增顾问选择与更换模块（浏览顾问、申请顾问、管理审批）  
> **v1.11.0 更新**：新增用户-顾问通信模块（我的顾问、消息、预约咨询）  
> **v1.10.0 更新**：新增 GET /user/info 接口，登录响应扩展 email/phone/avatar，风险评估支持前端传参，新增响应加密功能  
> **v1.9.0 更新**：集成 AkShare 金融数据，替换 iTick API，支持 A股指数/A股股票/美股/加密货币/外汇实时行情，数据自动同步至 MySQL  
> **v1.8.2 更新**：集成 iTick 金融数据 API，新增股票/加密货币/外汇实时行情、K线、盘口深度接口  
> **v1.8.1 更新**：新增行情概览、自选管理、智能提醒、顾问预警、产品对比、客户标签接口  
> **v1.7.0 更新**：重构 AI 对话模块，统一存储与接口，新增会话管理功能  
> **v1.6.4 更新**：完善企业级告警策略配置说明（冷却、聚合、静默、分级）  
> **v1.6.3 更新**：添加通知服务真实化说明（短信/邮件验证码发送）  
> **v1.6.2 更新**：修正 AI 推荐、理财方案接口响应字段名  
> **v1.6.1 更新**：修正 AI 问答接口响应格式（role/content 字段）  
> **v1.6.0 更新**：添加风险事件管理、冻结用户、最近活动记录接口  
> **v1.5.2 更新**：风控服务接口改为通过网关代理访问，产品内容查询接口添加风险等级筛选  
> **v1.5.1 更新**：添加典型调用流程图，完善各模块枚举定义说明  
> **v1.5.0 更新**：修正网关路由配置，移除未通过网关暴露的接口，明确权限说明

---

## 目录

0. [典型调用流程](#0-典型调用流程)
1. [接口规范](#1-接口规范)
2. [认证与授权](#2-认证与授权)
3. [用户服务接口](#3-用户服务接口)
4. [资产服务接口](#4-资产服务接口)
5. [交易服务接口](#5-交易服务接口)
6. [AI理财服务接口](#6-ai理财服务接口)
7. [内容服务接口](#7-内容服务接口)
8. [通知服务接口](#8-通知服务接口)
9. [管理后台接口](#9-管理后台接口)
10. [统计分析接口](#10-统计分析接口)
11. [风控服务接口](#11-风控服务接口-管理端)
12. [错误码说明](#12-错误码说明)
13. [附录一：枚举定义](#附录一枚举定义)
14. [附录二：通知服务真实化配置](#附录二通知服务真实化配置)
15. [附录三：金融行情数据接口](#附录三金融行情数据接口)
16. [附录四：响应数据加密](#附录四响应数据加密)
17. [附录五：数据库冗余表清理](#附录五数据库冗余表清理v1140)

---

## 0. 典型调用流程

### 0.1 用户认证流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   前端应用   │     │   API网关    │     │  用户服务    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │  1. POST /auth/register               │
       │──────────────────>│──────────────────>│
       │                   │   返回 userId     │
       │<──────────────────│<──────────────────│
       │                   │                   │
       │  2. POST /auth/login                  │
       │──────────────────>│──────────────────>│
       │                   │  返回 Token       │
       │<──────────────────│<──────────────────│
       │                   │                   │
       │  3. 携带 Token 访问业务接口            │
       │  Authorization: Bearer {accessToken}  │
       │──────────────────>│  验证Token后转发   │
       │                   │──────────────────>│
       │                   │                   │
```

**关键步骤：**

1. **注册** → `POST /auth/register` → 获取 `userId`
2. **登录** → `POST /auth/login` → 获取 `accessToken` 和 `refreshToken`
3. **业务请求** → 请求头携带 `Authorization: Bearer {accessToken}`
4. **Token过期** → 使用 `refreshToken` 刷新，或重新登录

---

### 0.2 理财产品交易流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   前端应用   │     │   交易服务   │     │   风控服务   │     │   资产服务   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ 1. GET /asset/overview (查看资产)      │                   │
       │───────────────────────────────────────────────────────────>│
       │<───────────────────────────────────────────────────────────│
       │                   │                   │                   │
       │ 2. GET /trade/products (查看产品)      │                   │
       │──────────────────>│                   │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
       │ 3. POST /trade/orders (创建订单)       │                   │
       │──────────────────>│                   │                   │
       │                   │ 4. 风控校验 ────────>│                   │
       │                   │<────────────────────│                   │
       │                   │                   │                   │
       │                   │ 5. 更新资产 ──────────────────────────>│
       │<──────────────────│                   │<──────────────────│
       │   返回订单结果     │                   │                   │
```

**关键步骤：**

1. **查看资产** → `GET /asset/overview`
2. **浏览产品** → `GET /trade/products`
3. **下单买入** → `POST /trade/orders` (需要交易密码)
4. **查看订单** → `GET /trade/orders`
5. **查看持仓** → `GET /asset/details`

---

### 0.3 顾问服务流程（ADVISOR）

```
1. 获取客户列表      → GET  /user/advisor/clients
2. 查看客户详情      → GET  /user/advisor/clients/{userId}
3. 查看客户资产      → GET  /user/advisor/clients/{userId}/assets
4. AI生成理财方案    → POST /user/advisor/plans/generate
5. 保存并发送方案    → POST /user/advisor/plans/{planId}/send
6. 记录服务记录      → POST /user/advisor/clients/{userId}/service-records
7. 查看风险预警      → GET  /user/advisor/alerts
8. 处理预警          → POST /user/advisor/alerts/{alertId}/process
9. 管理客户标签      → GET/POST/DELETE /user/advisor/clients/{userId}/tags
```

---

### 0.4 管理员操作流程（ADMIN）

```
1. 用户管理
   ├── 查询用户列表  → GET  /admin/users
   ├── 禁用用户      → PUT  /admin/users/{id}/status?status=0
   └── 重置密码      → POST /admin/users/{id}/reset-password

2. 角色权限管理
   ├── 角色列表      → GET  /admin/roles
   ├── 分配权限      → POST /admin/roles/{roleId}/permissions
   └── 用户授权      → POST /admin/users/{userId}/roles

3. 系统监控
   ├── 系统状态      → GET  /admin/system/status
   ├── CPU趋势       → GET  /admin/system/cpu-trend
   └── 服务状态      → GET  /admin/system/services
```

---

## 1. 接口规范

### 1.1 服务端口配置

| 服务名称                 | 直连端口 | 网关路由前缀                              | 备注                 |
| -------------------- | ---- | ----------------------------------- | ------------------ |
| **gateway-service**  | 8001 | -                                   | API 网关入口           |
| user-service         | 8101 | `/auth/**`, `/user/**`, `/admin/**` |                    |
| asset-service        | 8201 | `/asset/**`                         |                    |
| ai-finance-service   | 8301 | `/ai/**`                            |                    |
| trade-service        | 8401 | `/trade/**`                         |                    |
| content-service      | 8501 | `/content/**`                       |                    |
| notification-service | 8601 | `/notification/**`                  | StripPrefix=1      |
| risk-control-service | 8701 | `/admin/risk/**` (via user-service) | 通过 user-service 代理 |
| analysis-service     | 8801 | `/analysis/**`                      | 需 ADMIN 权限         |

### 1.2 基础URL

```
# 网关访问（推荐）
http://localhost:8001

# 服务直连（开发调试）
http://localhost:{服务端口}
```

### 1.3 统一响应格式

```json
{
  "code": 200,
  "msg": "success",
  "data": { }
}
```

| 字段   | 类型     | 说明            |
| ---- | ------ | ------------- |
| code | int    | 200=成功，500=失败 |
| msg  | string | 响应消息（可为 null） |
| data | any    | 响应数据（可为 null） |

### 1.4 请求头规范

| Header            | 必填  | 说明                         |
| ----------------- | --- | -------------------------- |
| `Authorization`   | 是*  | Bearer {accessToken}，登录后携带 |
| `Content-Type`    | 是   | application/json           |
| `X-Auth-UserId`   | 否   | 网关自动注入，服务直连时手动传            |
| `X-Auth-Username` | 否   | 网关自动注入                     |

---

## 2. 认证与授权

### 2.1 用户注册

**POST** `/auth/register`

**请求体：**

```json
{
  "username": "zhangsan",
  "email": "zhangsan@example.com",
  "phone": "13800138000",
  "realName": "张三",
  "password": "SecurePass123!",
  "gender": 1,
  "birthDate": "1990-01-01"
}
```

| 字段        | 类型     | 必填  | 说明                 |
| --------- | ------ | --- | ------------------ |
| username  | string | 是   | 用户名，最长50字符         |
| email     | string | 是   | 邮箱                 |
| phone     | string | 是   | 手机号，11位            |
| realName  | string | 否   | 真实姓名               |
| password  | string | 是   | 密码，6-64位           |
| gender    | int    | 否   | 性别：1=男，2=女         |
| birthDate | string | 否   | 出生日期，格式：yyyy-MM-dd |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": 1001
}
```

> data 为新注册用户的 userId

> **v1.16.0 验证机制变更**：
> - 注册时填写的邮箱和手机号**不会立即验证**，状态默认为"未验证"
> - 只有已被其他用户**验证绑定**的邮箱/手机才会阻止注册（未验证的允许重复填写）
> - 用户注册成功后可在个人中心发起验证绑定

---

### 2.2 用户登录

**POST** `/auth/login`

**请求体：**

```json
{
  "username": "zhangsan",
  "password": "SecurePass123!",
  "captcha": "1234",
  "deviceType": 1,
  "deviceId": "web-browser-001"
}
```

| 字段         | 类型     | 必填  | 说明                                    |
| ---------- | ------ | --- | ------------------------------------- |
| username   | string | 是   | 用户名/邮箱/手机号（别名：identifier、email、phone） |
| password   | string | 是   | 密码                                    |
| captcha    | string | 否   | 图形验证码                                 |
| deviceType | int    | 否   | 设备类型：1=Web，2=iOS，3=Android            |
| deviceId   | string | 否   | 设备唯一标识                                |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "userId": 1001,
    "username": "zhangsan",
    "realName": "张三",
    "email": "zhangsan@example.com",
    "emailVerified": false,
    "phone": "13800138000",
    "phoneVerified": true,
    "avatar": "https://cdn.example.com/avatars/1001.jpg",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "roles": ["USER"],
    "permissions": ["user:read", "trade:create"]
  }
}
```

| 字段            | 类型      | 说明                       |
| ------------- | ------- | ------------------------ |
| userId        | long    | 用户ID                     |
| username      | string  | 用户名                      |
| realName      | string  | 真实姓名                     |
| email         | string  | 邮箱                         |
| emailVerified | boolean | 邮箱是否已验证                 |
| phone         | string  | 手机号                        |
| phoneVerified | boolean | 手机是否已验证                 |
| avatar        | string  | 头像URL                       |
| accessToken   | string  | 访问令牌                     |
| refreshToken  | string  | 刷新令牌                     |
| expiresIn     | long    | 过期时间（秒）                  |
| roles         | array   | 角色列表                     |

> **v1.16.0 验证状态说明**：
> - 注册时填写的邮箱/手机**不会立即验证**，`emailVerified`/`phoneVerified` 默认为 `false`
> - 用户可在个人中心发起验证绑定，验证成功后状态变为 `true`
> - 前端可根据验证状态在界面上显示"未验证"标签并引导用户完成验证

---

### 2.2.1 获取图形验证码

**GET** `/auth/captcha`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "captchaId": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
    "captchaImage": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
    "expireSeconds": 300
  }
}
```

| 字段           | 类型   | 说明                           |
| -------------- | ------ | ------------------------------ |
| captchaId      | string | 验证码唯一标识，后续验证时需携带 |
| captchaImage   | string | Base64编码的验证码图片         |
| expireSeconds  | int    | 验证码有效期（秒），默认300秒   |

> **使用说明**：
> - 图形验证码用于登录、发送短信验证码等场景的防刷验证
> - 前端将 `captchaImage` 直接作为 `<img>` 标签的 `src` 属性展示
> - 验证码为一次性使用，验证成功后自动失效

---

### 2.2.2 发送登录验证码

**POST** `/auth/login-sms/code`

**请求参数：**

| 参数  | 类型   | 必填 | 说明              |
| ----- | ------ | ---- | ----------------- |
| phone | string | 是   | 手机号，11位数字   |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> **注意事项**：
> - 该手机号必须已注册，否则返回错误
> - 发送频率限制：60秒内只能发送一次
> - 验证码有效期：10分钟

---

### 2.2.3 短信验证码登录

**POST** `/auth/login-sms`

**请求体：**

```json
{
  "phone": "13800138000",
  "code": "123456",
  "captchaId": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "captcha": "AB12"
}
```

| 字段      | 类型   | 必填 | 说明                         |
| --------- | ------ | ---- | ---------------------------- |
| phone     | string | 是   | 手机号，11位数字              |
| code      | string | 是   | 短信验证码，6位数字           |
| captchaId | string | 否   | 图形验证码ID（用于防刷）      |
| captcha   | string | 否   | 图形验证码（用于防刷）        |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "userId": 1001,
    "username": "zhangsan",
    "realName": "张三",
    "email": "zhangsan@example.com",
    "emailVerified": false,
    "phone": "13800138000",
    "phoneVerified": true,
    "avatar": "https://cdn.example.com/avatars/1001.jpg",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "roles": ["USER"]
  }
}
```

> 响应字段与密码登录接口 (2.2) 一致

---

### 2.3 用户登出

**POST** `/auth/logout`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 用户ID |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 2.4 修改密码

**POST** `/auth/change-password`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 用户ID（Query参数） |

**请求体：**

```json
{
  "oldPassword": "OldPass123!",
  "newPassword": "NewPass456!"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 2.5 忘记密码 - 申请重置

**POST** `/auth/forgot-password/request`

**请求体：**

```json
{
  "identifier": "zhangsan@example.com"
}
```

> identifier 可以是用户名、邮箱或手机号

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 2.6 忘记密码 - 确认重置

**POST** `/auth/forgot-password/reset`

**请求体：**

```json
{
  "identifier": "zhangsan@example.com",
  "code": "123456",
  "newPassword": "NewPass789!"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

## 3. 用户服务接口

### 3.1 获取用户基本信息

**GET** `/user/info`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1001,
    "username": "zhangsan",
    "realName": "张三",
    "email": "zhangsan@example.com",
    "emailVerified": false,
    "phone": "13800138000",
    "phoneVerified": true,
    "avatar": "https://cdn.example.com/avatars/1001.jpg",
    "gender": 1,
    "birthDate": "1990-01-01",
    "createTime": "2024-11-01T10:00:00",
    "lastLoginTime": "2024-12-11T14:00:00"
  }
}
```

| 字段            | 类型       | 说明                 |
| ------------- | -------- | ------------------ |
| id            | long     | 用户ID               |
| username      | string   | 用户名                |
| realName      | string   | 真实姓名               |
| email         | string   | 邮箱                 |
| emailVerified | boolean  | 邮箱是否已验证               |
| phone         | string   | 手机号                |
| phoneVerified | boolean  | 手机是否已验证               |
| avatar        | string   | 头像URL              |
| gender        | int      | 性别：1=男，2=女         |
| birthDate     | string   | 出生日期               |
| createTime    | datetime | 注册时间               |
| lastLoginTime | datetime | 最后登录时间             |

> **v1.16.0 验证状态说明**：
> - `emailVerified: false` 表示邮箱未验证，前端可显示"未验证"标签
> - `phoneVerified: false` 表示手机未验证，前端可显示"未验证"标签
> - 用户可通过 `/user/security/bind-phone` 或 `/user/security/bind-email` 接口发起验证绑定

---

### 3.2 获取用户画像

**GET** `/user/profile`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "userId": 1001,
    "riskPreference": 2,
    "financialGoal": "子女教育",
    "annualIncome": 300000.00,
    "occupation": "IT工程师",
    "educationLevel": "本科",
    "investmentExperience": 3,
    "familyStatus": 1
  }
}
```

---

### 3.3 保存/更新用户画像

**POST** `/user/profile`

**请求体：**

```json
{
  "riskPreference": 2,
  "financialGoal": "子女教育",
  "annualIncome": 300000.00,
  "occupation": "IT工程师",
  "educationLevel": "本科",
  "investmentExperience": 3,
  "familyStatus": 1
}
```

| 字段                   | 类型      | 说明                            |
| -------------------- | ------- | ----------------------------- |
| riskPreference       | int     | 风险偏好：1=保守，2=稳健，3=平衡，4=积极，5=激进 |
| financialGoal        | string  | 理财目标                          |
| annualIncome         | decimal | 年收入                           |
| occupation           | string  | 职业                            |
| educationLevel       | string  | 学历                            |
| investmentExperience | int     | 投资经验（年）                       |
| familyStatus         | int     | 家庭状况：1=单身，2=已婚无孩，3=已婚有孩       |

---

### 3.4 更新用户基本信息

**PUT** `/user/info`

> 更新当前登录用户的真实姓名、性别、生日、头像等信息

**请求体：**

```json
{
  "realName": "张三",
  "gender": 1,
  "birthDate": "1990-01-15",
  "avatarUrl": "https://oss.example.com/images/avatar.jpg"
}
```

| 字段        | 类型     | 必填  | 说明               |
| --------- | ------ | --- | ---------------- |
| realName  | string | 否   | 真实姓名             |
| gender    | int    | 否   | 性别：1=男，2=女       |
| birthDate | string | 否   | 出生日期（yyyy-MM-dd） |
| avatarUrl | string | 否   | 头像URL（OSS完整地址）   |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1001,
    "username": "user001",
    "realName": "张三",
    "email": "user@example.com",
    "phone": "13800138000",
    "avatar": "https://oss.example.com/images/avatar.jpg",
    "gender": 1,
    "birthDate": "1990-01-15",
    "createTime": "2024-11-01T10:00:00",
    "lastLoginTime": "2024-12-16T14:00:00"
  }
}
```

---

### 3.5 更新用户头像

**PUT** `/user/avatar`

> 单独更新当前登录用户的头像URL

**请求体：**

```json
{
  "avatarUrl": "https://oss.example.com/images/avatar.jpg"
}
```

| 字段        | 类型     | 必填  | 说明             |
| --------- | ------ | --- | -------------- |
| avatarUrl | string | 是   | 头像URL（OSS完整地址） |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": "https://oss.example.com/images/avatar.jpg"
}
```

> **头像上传完整流程：**
> 
> 1. 调用 `POST /content/media/upload-token`，bizType=3（头像）
> 2. 前端使用 PUT 方法上传文件到返回的 `uploadUrl`
> 3. 上传成功后调用 `POST /content/media/confirm` 确认
> 4. 调用 `PUT /user/avatar` 更新用户头像URL
> 
> **产品图片上传流程：****
> 
> 1. POST /content/media/upload-token (bizType=1) → 获取上传凭证
> 2. PUT uploadUrl → 上传文件到OSS
> 3. POST /content/media/confirm (bindContentType=product, bindContentId=xxx, asCover=true) → 确认并绑定产品

---

### 3.6 提交风险评估

**POST** `/user/risk-assessment`

**请求体：**

```json
{
  "score": 35,
  "riskLevel": 3
}
```

| 字段        | 类型  | 必填  | 说明                                 |
| --------- | --- | --- | ---------------------------------- |
| score     | int | 是   | 评估得分（10-50）                        |
| riskLevel | int | 是   | 风险等级（1-5）：1=保守，2=稳健，3=平衡，4=积极，5=激进 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "userId": 1001,
    "riskLevel": 3,
    "score": 35,
    "assessmentDetails": "您的风险评估得分为35分，风险等级为平衡型。",
    "assessmentTime": "2024-12-11T14:30:00"
  }
}
```

> **变更说明**：v1.10.0 版本该接口由后端自动评估改为前端提交评估结果，便于前端实现评估问卷交互。

---

### 3.7 自动风险评估（基于画像）

**POST** `/user/risk-assessment/auto`

> 基于用户画像自动计算风险评估，无需请求体

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "userId": 1001,
    "riskLevel": 3,
    "score": 65,
    "assessmentDetails": "根据您的画像信息...",
    "assessmentTime": "2024-12-11T14:30:00"
  }
}
```

---

### 3.8 获取最新风险评估

**GET** `/user/risk-assessment/latest`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "userId": 1001,
    "riskLevel": 3,
    "score": 65,
    "assessmentDate": "2024-11-28T10:30:00",
    "validUntil": "2025-11-28T10:30:00"
  }
}
```

---

### 3.9 查询登录日志

**GET** `/user/security/login-logs?days=30`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| days | int | 否 | 30 | 查询天数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "logId": 1,
      "userId": 1001,
      "ipAddress": "192.168.1.100",
      "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
      "deviceInfo": "Chrome 119, Windows 10",
      "loginTime": "2024-11-28T10:30:00",
      "logoutTime": "2024-11-28T18:00:00",
      "status": 1
    }
  ]
}
```

> status: 0=登录失败，1=登录成功

---

### 3.10 设置交易密码

**POST** `/user/security/trade-password/set`

**请求体：**

```json
{
  "tradePassword": "123456",
  "confirmPassword": "123456"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.11 修改交易密码

**POST** `/user/security/trade-password/change`

**请求体：**

```json
{
  "oldPassword": "123456",
  "newPassword": "654321",
  "confirmPassword": "654321"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.12 发送大额交易验证码

**POST** `/user/security/large-trade/code`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 验证码将发送到用户绑定的手机或邮箱

---

### 3.13 绑定手机号

**POST** `/user/security/bind-phone/code?phone=13900139000`

> 发送绑定手机验证码

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

**POST** `/user/security/bind-phone`

**请求体：**

```json
{
  "phone": "13900139000",
  "code": "123456"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.14 绑定邮箱

**POST** `/user/security/bind-email/code?email=new@example.com`

> 发送绑定邮箱验证码

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

**POST** `/user/security/bind-email`

**请求体：**

```json
{
  "email": "new@example.com",
  "code": "123456"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.15 解绑手机号

**POST** `/user/security/unbind-phone`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.16 解绑邮箱

**POST** `/user/security/unbind-email`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.17 冻结账号

**POST** `/user/security/freeze`

> 将当前登录账号状态置为禁用，并强制下线所有登录会话

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 3.17.1 发送解冻验证码

**POST** `/user/security/unfreeze/code`

**请求参数：**

| 参数       | 类型   | 必填 | 说明                           |
| ---------- | ------ | ---- | ------------------------------ |
| verifyType | string | 是   | 验证方式：`phone`-手机，`email`-邮箱 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> **注意事项**：
> - 需要用户已绑定对应的手机号或邮箱
> - 验证码有效期：10分钟
> - 发送频率限制：60秒内只能发送一次

---

### 3.17.2 自助解冻账号

**POST** `/user/security/unfreeze`

**请求体：**

```json
{
  "code": "123456",
  "verifyType": "phone"
}
```

| 字段       | 类型   | 必填 | 说明                           |
| ---------- | ------ | ---- | ------------------------------ |
| code       | string | 是   | 验证码，6位数字                 |
| verifyType | string | 是   | 验证方式：`phone`-手机，`email`-邮箱 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> **功能说明**：
> - 用户可通过验证绑定的手机或邮箱自助解冻被冻结的账号
> - 解冻成功后，账号状态恢复为正常，可重新登录
> - 验证码错误次数超过3次后验证码失效，需重新获取

---

### 3.18 顾问客户管理

> ⚠️ **权限要求**：需要 `ADVISOR` 权限，无权限访问返回 403 Forbidden

#### 3.18.1 获取客户列表

**GET** `/user/advisor/clients?page=1&size=20&status=active&keyword=张三`

| 参数      | 类型     | 必填  | 默认值 | 说明    |
| ------- | ------ | --- | --- | ----- |
| page    | int    | 否   | 1   | 页码    |
| size    | int    | 否   | 20  | 每页条数  |
| status  | string | 否   | -   | 客户状态  |
| keyword | string | 否   | -   | 搜索关键词 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "relationship_id": 1,
        "user_id": 1001,
        "relationship_status": 1,
        "assigned_at": "2024-01-15T10:00:00.000+08:00",
        "username": "zhangsan",
        "real_name": "张三",
        "phone": "13800138001",
        "email": "zhangsan@example.com",
        "avatar_url": "https://example.com/avatar.jpg",
        "risk_preference": 3,
        "total_assets": 500000.00,
        "risk_level": "MODERATE",
        "risk_score": 65
      }
    ],
    "total": 1,
    "size": 20,
    "current": 1,
    "pages": 1
  }
}
```

> **注意**：`status` 参数默认为 `1`（活跃），只返回活跃状态的客户。如需查看所有状态客户，请显式传递 `status=` 空值。

| 字段 | 类型 | 说明 |
|------|------|------|
| relationship_id | long | 关系ID |
| user_id | long | 客户用户ID |
| relationship_status | int | 关系状态：1=活跃，2=已结束 |
| assigned_at | datetime | 分配时间 |
| username | string | 用户名 |
| real_name | string | 真实姓名 |
| phone | string | 手机号 |
| email | string | 邮箱 |
| avatar_url | string | 头像URL |
| risk_preference | int | 风险偏好等级 |
| total_assets | decimal | 总资产 |
| risk_level | string | 风险等级（来自最新风险评估）|
| risk_score | int | 风险评分（来自最新风险评估）|

---

#### 3.18.2 获取客户详情

**GET** `/user/advisor/clients/{userId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "user_id": 1001,
    "username": "zhangsan",
    "real_name": "张三",
    "phone": "13800138001",
    "email": "zhangsan@example.com",
    "avatar_url": "https://example.com/avatar.jpg",
    "gender": 1,
    "birth_date": "1990-01-15",
    "risk_preference": 3,
    "financial_goal": "子女教育",
    "annual_income": 200000,
    "occupation": "工程师",
    "investment_experience": 3,
    "assigned_at": "2024-01-15T10:00:00",
    "relationship_status": 1
  }
}
```

---

#### 3.18.3 获取客户资产概览

**GET** `/user/advisor/clients/{userId}/assets`

---

#### 3.18.4 获取客户交易记录

**GET** `/user/advisor/clients/{userId}/trades?page=1&size=20`

---

#### 3.18.5 获取客户风险评估

**GET** `/user/advisor/clients/{userId}/risk`

---

#### 3.18.6 添加服务记录

**POST** `/user/advisor/clients/{userId}/service-records`

**请求体：**

```json
{
  "serviceType": "咨询",
  "serviceContent": "为客户提供基金配置建议",
  "serviceTime": "2024-11-28T14:00:00"
}
```

---

#### 3.18.7 获取客户分类统计

**GET** `/user/advisor/clients/statistics`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "totalClients": 50,
    "activeClients": 45,
    "newClientsThisMonth": 5,
    "byRiskLevel": {
      "conservative": 10,
      "moderate": 25,
      "aggressive": 15
    }
  }
}
```

---

### 3.19 顾问理财方案管理

> ⚠️ **权限要求**：需要 `ADVISOR` 权限，无权限访问返回 403 Forbidden

#### 3.19.1 AI生成理财方案

**POST** `/user/advisor/plans/generate`

**请求体：**

```json
{
  "clientId": 1001,
  "title": "养老规划方案",
  "goal": "60岁前积累200万养老金",
  "riskLevel": 2,
  "initialCapital": 100000.00,
  "monthlyInvestment": 5000.00,
  "years": 20
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "planId": 1,
    "title": "养老规划方案",
    "clientId": 1001,
    "goal": "60岁前积累200万养老金",
    "planContent": "## 一、目标分析\n...\n## 二、配置建议\n...",
    "portfolios": [
      {"name": "债券型基金", "percentage": 50},
      {"name": "股票型基金", "percentage": 30},
      {"name": "货币基金", "percentage": 20}
    ],
    "expectedReturn": 0.065,
    "status": "DRAFT"
  }
}
```

---

#### 3.19.2 获取方案列表

**GET** `/user/advisor/plans?clientId=1001&page=1&size=20`

---

#### 3.19.3 发送方案给客户

**POST** `/user/advisor/plans/{planId}/send`

---

#### 3.19.4 导出PDF

**GET** `/user/advisor/plans/{planId}/export`

> 返回 PDF 文件下载 URL

---

### 3.20 顾问预警管理

> ⚠️ **权限要求**：需要 `ADVISOR` 权限

> **枚举定义：**
> 
> | 枚举                | 值               | 说明    |
> | ----------------- | --------------- | ----- |
> | **alertType**     | POSITION_RISK   | 仓位风险  |
> |                   | LOSS_ALERT      | 亏损预警  |
> |                   | CONCENTRATION   | 集中度过高 |
> |                   | OVERDUE         | 产品到期  |
> | **status**        | PENDING         | 待处理   |
> |                   | PROCESSING      | 处理中   |
> |                   | PROCESSED       | 已处理   |
> |                   | IGNORED         | 已忽略   |
> | **processAction** | CONTACT         | 联系客户  |
> |                   | IGNORE          | 忽略    |
> |                   | REDUCE_POSITION | 建议减仓  |

#### 3.20.1 获取客户风险预警列表

**GET** `/user/advisor/alerts?status=PENDING&page=1&size=20`

| 参数     | 类型     | 必填  | 默认值 | 说明     |
| ------ | ------ | --- | --- | ------ |
| status | string | 否   | -   | 预警状态筛选 |
| page   | int    | 否   | 1   | 页码     |
| size   | int    | 否   | 20  | 每页条数   |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "id": 1,
        "advisor_id": 100,
        "client_id": 1001,
        "client_name": "zhangsan",
        "client_real_name": "张三",
        "alert_type": "CONCENTRATION",
        "risk_level": 3,
        "message": "客户持仓集中度过高，单一产品占比超过60%",
        "status": "PENDING",
        "triggered_at": "2025-12-10T10:00:00"
      }
    ],
    "total": 5,
    "size": 20,
    "current": 1,
    "pages": 1
  }
}
```

#### 3.20.2 获取预警详情

**GET** `/user/advisor/alerts/{alertId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "advisor_id": 100,
    "client_id": 1001,
    "client_name": "zhangsan",
    "client_real_name": "张三",
    "alert_type": "CONCENTRATION",
    "risk_level": 3,
    "message": "客户持仓集中度过高，单一产品占比超过60%",
    "detail_json": "{\"productId\":2001,\"percentage\":65.5}",
    "status": "PENDING",
    "triggered_at": "2025-12-10T10:00:00",
    "risk_preference": 2,
    "financial_goal": "养老规划"
  }
}
```

#### 3.20.3 处理预警

**POST** `/user/advisor/alerts/{alertId}/process`

**请求体：**

```json
{
  "action": "CONTACT",
  "note": "已电话联系客户，建议调整持仓结构"
}
```

| 字段     | 类型     | 必填  | 说明                                  |
| ------ | ------ | --- | ----------------------------------- |
| action | string | 是   | 处理动作：CONTACT/IGNORE/REDUCE_POSITION |
| note   | string | 否   | 处理备注                                |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 3.21 客户标签管理

> ⚠️ **权限要求**：需要 `ADVISOR` 权限

> **枚举定义：**
> 
> | 枚举          | 值        | 说明   |
> | ----------- | -------- | ---- |
> | **tagType** | NEED     | 需求标签 |
> |             | INTEREST | 兴趣标签 |
> |             | BEHAVIOR | 行为标签 |

#### 3.21.1 获取客户标签

**GET** `/user/advisor/clients/{userId}/tags`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "advisor_id": 100,
      "client_id": 1001,
      "tag_name": "养老规划",
      "tag_type": "NEED",
      "create_time": "2025-12-10T10:00:00"
    },
    {
      "id": 2,
      "advisor_id": 100,
      "client_id": 1001,
      "tag_name": "稳健偏好",
      "tag_type": "INTEREST",
      "create_time": "2025-12-10T10:05:00"
    }
  ]
}
```

#### 3.21.2 添加客户标签

**POST** `/user/advisor/clients/{userId}/tags`

**请求体：**

```json
{
  "tagName": "子女教育",
  "tagType": "NEED"
}
```

| 字段      | 类型     | 必填  | 说明           |
| ------- | ------ | --- | ------------ |
| tagName | string | 是   | 标签名称         |
| tagType | string | 否   | 标签类型，默认 NEED |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 3,
    "advisor_id": 100,
    "client_id": 1001,
    "tag_name": "子女教育",
    "tag_type": "NEED",
    "create_time": "2025-12-10T15:00:00"
  }
}
```

#### 3.21.3 删除客户标签

**DELETE** `/user/advisor/clients/{userId}/tags/{tagId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 3.22 我的顾问

> 用户端接口，用于查看顾问信息、发送消息、预约咨询

#### 3.22.1 获取我的顾问信息

**GET** `/user/my-advisor`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 1,
    "advisorName": "advisor001",
    "realName": "李明",
    "avatar": "https://example.com/avatar.jpg",
    "phone": "138****8888",
    "email": "liming@example.com",
    "title": "高级理财顾问",
    "introduction": "拥有8年金融从业经验...",
    "specialties": ["资产配置", "基金投资", "养老规划"],
    "serviceYears": 8,
    "clientCount": 156,
    "rating": 4.8,
    "assignedAt": "2024-01-15T10:00:00"
  }
}
```

> 如果未分配顾问，`data` 返回 `null`

---

#### 3.22.2 获取与顾问的消息列表

**GET** `/user/my-advisor/messages?page=1&size=50`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| page | int | 否 | 1 | 页码 |
| size | int | 否 | 50 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "messageId": 1,
        "senderId": 1,
        "senderType": "ADVISOR",
        "senderName": "李明",
        "receiverId": 100,
        "content": "您好，请问有什么可以帮您的？",
        "type": "TEXT",
        "isRead": true,
        "createdAt": "2024-12-10T10:00:00"
      }
    ],
    "total": 20,
    "current": 1,
    "size": 50
  }
}
```

---

#### 3.22.3 给顾问发送消息

**POST** `/user/my-advisor/messages`

**请求体：**

```json
{
  "content": "我想咨询一下基金定投的问题",
  "type": "TEXT"
}
```

| 字段      | 类型     | 必填  | 说明                          |
| ------- | ------ | --- | --------------------------- |
| content | string | 是   | 消息内容                        |
| type    | string | 否   | 消息类型：TEXT/IMAGE/FILE，默认TEXT |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "messageId": 3,
    "senderId": 100,
    "senderType": "USER",
    "senderName": "张三",
    "receiverId": 1,
    "content": "我想咨询一下基金定投的问题",
    "type": "TEXT",
    "isRead": false,
    "createdAt": "2024-12-10T10:10:00"
  }
}
```

---

#### 3.22.4 标记消息已读

**POST** `/user/my-advisor/messages/{messageId}/read`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.22.5 获取我的预约列表

**GET** `/user/my-advisor/appointments?status=PENDING&page=1&size=20`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | string | 否 | 状态筛选：PENDING/CONFIRMED/COMPLETED/CANCELLED |
| page | int | 否 | 页码 |
| size | int | 否 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "appointmentId": 1,
        "userId": 100,
        "advisorId": 1,
        "advisorName": "李明",
        "scheduledTime": "2024-12-15T14:00:00",
        "duration": 30,
        "topic": "资产配置咨询",
        "description": "希望了解如何优化当前的资产配置",
        "status": "CONFIRMED",
        "createdAt": "2024-12-10T10:00:00"
      }
    ],
    "total": 5,
    "current": 1,
    "size": 20
  }
}
```

---

#### 3.22.6 创建预约

**POST** `/user/my-advisor/appointments`

**请求体：**

```json
{
  "scheduledTime": "2024-12-15T14:00:00",
  "duration": 30,
  "topic": "资产配置咨询",
  "description": "希望了解如何优化当前的资产配置"
}
```

| 字段            | 类型       | 必填  | 说明            |
| ------------- | -------- | --- | ------------- |
| scheduledTime | datetime | 是   | 预约时间（必须是未来时间） |
| duration      | int      | 否   | 时长（分钟），默认30   |
| topic         | string   | 是   | 咨询主题          |
| description   | string   | 否   | 详细说明          |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "appointmentId": 2,
    "userId": 100,
    "advisorId": 1,
    "advisorName": "李明",
    "scheduledTime": "2024-12-15T14:00:00",
    "duration": 30,
    "topic": "资产配置咨询",
    "status": "PENDING",
    "createdAt": "2024-12-11T09:00:00"
  }
}
```

---

#### 3.22.7 取消预约

**POST** `/user/my-advisor/appointments/{appointmentId}/cancel`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 只能取消状态为 PENDING 或 CONFIRMED 的预约

---

### 3.23 顾问工作台

> 顾问端接口，用于查看客户消息、管理预约（需 ADVISOR 角色）

#### 3.23.1 获取与客户的消息列表

**GET** `/user/advisor/clients/{userId}/messages?page=1&size=50`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 客户ID（路径参数） |
| page | int | 否 | 页码 |
| size | int | 否 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "messageId": 1,
        "senderId": 100,
        "senderType": "USER",
        "senderName": "张三",
        "receiverId": 1,
        "content": "我想了解基金定投",
        "type": "TEXT",
        "isRead": true,
        "createdAt": "2024-12-10T10:05:00"
      }
    ],
    "total": 10,
    "current": 1,
    "size": 50
  }
}
```

---

#### 3.23.2 给客户发送消息

**POST** `/user/advisor/clients/{userId}/messages`

**请求体：**

```json
{
  "content": "您好，关于您咨询的基金定投问题...",
  "type": "TEXT"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "messageId": 4,
    "senderId": 1,
    "senderType": "ADVISOR",
    "senderName": "李明",
    "receiverId": 100,
    "content": "您好，关于您咨询的基金定投问题...",
    "type": "TEXT",
    "isRead": false,
    "createdAt": "2024-12-10T10:15:00"
  }
}
```

---

#### 3.23.3 获取顾问的预约列表

**GET** `/user/advisor/appointments?clientId=100&status=PENDING&page=1&size=20`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| clientId | long | 否 | 客户ID筛选 |
| status | string | 否 | 状态筛选 |
| page | int | 否 | 页码 |
| size | int | 否 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "appointmentId": 1,
        "userId": 100,
        "userName": "张三",
        "userPhone": "138****0001",
        "advisorId": 1,
        "scheduledTime": "2024-12-15T14:00:00",
        "duration": 30,
        "topic": "资产配置咨询",
        "description": "希望了解如何优化当前的资产配置",
        "status": "PENDING",
        "createdAt": "2024-12-10T10:00:00"
      }
    ],
    "total": 8,
    "current": 1,
    "size": 20
  }
}
```

---

#### 3.23.4 确认预约

**POST** `/user/advisor/appointments/{appointmentId}/confirm`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 将预约状态从 PENDING 更新为 CONFIRMED

---

#### 3.23.5 完成预约

**POST** `/user/advisor/appointments/{appointmentId}/complete`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 将预约状态从 CONFIRMED 更新为 COMPLETED

---

### 3.24 顾问通信模块枚举定义

| 枚举                    | 值         | 说明   |
| --------------------- | --------- | ---- |
| **senderType**        | USER      | 普通用户 |
|                       | ADVISOR   | 理财顾问 |
| **messageType**       | TEXT      | 文本消息 |
|                       | IMAGE     | 图片消息 |
|                       | FILE      | 文件消息 |
| **appointmentStatus** | PENDING   | 待确认  |
|                       | CONFIRMED | 已确认  |
|                       | COMPLETED | 已完成  |
|                       | CANCELLED | 已取消  |

---

### 3.25 顾问选择

> 用户端接口，支持**智能推荐 + 最终确认**模式
> 
> **推荐策略**：
> 
> 1. **硬性过滤**：排除评分<4.0、不接受新客户、已满员的顾问
> 2. **算法打分**：基于用户KYC（风险偏好、资金量、理财目标）匹配顾问擅长领域
> 3. **展示Top 3**：返回匹配度最高的3位顾问，每位附带匹配理由
> 4. **用户选择**：用户拥有最终"联系"的权利

#### 3.25.1 智能推荐顾问

**GET** `/user/advisors/recommend`

> 基于用户KYC问卷智能匹配顾问，返回匹配度最高的3位顾问及匹配理由

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "advisorId": 1,
      "name": "李明",
      "avatarUrl": "https://example.com/avatar.jpg",
      "title": "资深理财顾问",
      "introduction": "拥有10年金融从业经验...",
      "specialization": ["养老规划", "资产配置", "基金投资"],
      "qualification": ["AFP", "CFP"],
      "experienceYears": 10,
      "rating": 4.9,
      "clientCount": 156,
      "matchScore": 92,
      "matchReasons": [
        "擅长养老规划，与您的理财目标高度契合",
        "服务风格与您的稳健型风险偏好相匹配",
        "从业10年，经验丰富，适合高净值客户服务",
        "服务评分4.9，深受客户好评"
      ],
      "primaryReason": "擅长养老规划，与您的理财目标高度契合",
      "matchDetail": {
        "specializationScore": 38,
        "riskMatchScore": 28,
        "experienceScore": 18,
        "ratingScore": 8
      }
    },
    {
      "advisorId": 2,
      "name": "王强",
      "matchScore": 85,
      "primaryReason": "专业背景与您的资产增值目标相关",
      "..."
    },
    {
      "advisorId": 3,
      "name": "张华",
      "matchScore": 78,
      "primaryReason": "从业8年，专业可靠",
      "..."
    }
  ]
}
```

| 字段            | 类型     | 说明                                         |
| ------------- | ------ | ------------------------------------------ |
| matchScore    | int    | 匹配分数（0-100），满分构成：专业领域40+风险偏好30+经验资质20+评价10 |
| matchReasons  | array  | 匹配理由列表                                     |
| primaryReason | string | 主要匹配理由（用于卡片展示）                             |
| matchDetail   | object | 各维度详细分数                                    |

**匹配算法说明：**

- **专业领域匹配（40分）**：用户财务目标关键词 vs 顾问擅长领域
- **风险偏好匹配（30分）**：用户风险等级 vs 顾问服务风格
- **经验资质匹配（20分）**：用户资金量/经验 vs 顾问从业年限/资质认证
- **服务评价匹配（10分）**：顾问评分加权

> 如用户尚未完成KYC问卷，使用默认画像（平衡型、资产增值目标）

---

#### 3.25.2 计算与指定顾问的匹配度

**GET** `/user/advisors/{advisorId}/match`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 5,
    "name": "赵敏",
    "matchScore": 72,
    "primaryReason": "擅长基金投资，与您的需求相关",
    "matchReasons": ["擅长基金投资，与您的需求相关", "服务评分4.6，口碑良好"],
    "matchDetail": {
      "specializationScore": 25,
      "riskMatchScore": 22,
      "experienceScore": 15,
      "ratingScore": 10
    }
  }
}
```

---

#### 3.25.3 获取可选顾问列表

**GET** `/user/advisors?page=1&size=10&specialty=资产配置&keyword=理财`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| size | int | 否 | 每页条数，默认10 |
| specialty | string | 否 | 擅长领域筛选 |
| keyword | string | 否 | 关键词搜索（姓名/简介） |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "advisorId": 1,
        "realName": "李明",
        "avatar": "https://example.com/avatar.jpg",
        "title": "高级理财顾问",
        "introduction": "拥有8年金融从业经验...",
        "specialties": ["资产配置", "基金投资", "养老规划"],
        "serviceYears": 8,
        "clientCount": 156,
        "rating": 4.8,
        "acceptingNew": true
      }
    ],
    "total": 15,
    "current": 1,
    "size": 10
  }
}
```

> 只返回接受新客户且未满员的顾问，按评分降序排列

---

#### 3.25.4 获取顾问详情

**GET** `/user/advisors/{advisorId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 1,
    "realName": "李明",
    "avatar": "https://example.com/avatar.jpg",
    "title": "高级理财顾问",
    "introduction": "拥有8年金融从业经验，擅长为客户提供全方位的资产配置建议和投资组合管理服务。",
    "specialties": ["资产配置", "基金投资", "养老规划", "税务筹划"],
    "serviceYears": 8,
    "clientCount": 156,
    "rating": 4.8,
    "acceptingNew": true,
    "certifications": ["AFP", "CFP"],
    "education": "北京大学金融学硕士"
  }
}
```

---

#### 3.25.5 申请选择/更换顾问

**POST** `/user/advisor/apply`

**请求体：**

```json
{
  "advisorId": 2,
  "reason": "希望获得基金投资方面的专业指导"
}
```

| 字段        | 类型     | 必填  | 说明     |
| --------- | ------ | --- | ------ |
| advisorId | long   | 是   | 目标顾问ID |
| reason    | string | 否   | 申请原因   |

**响应示例（需审批模式）：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "requestId": 1,
    "status": "PENDING",
    "message": "申请已提交，请等待审核"
  }
}
```

**响应示例（自动分配模式）：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "requestId": 1,
    "status": "APPROVED",
    "message": "顾问分配成功"
  }
}
```

**错误响应：**

- `您有进行中的申请，请勿重复提交`
- `该顾问暂不接受新客户`
- `该顾问客户已满`

---

#### 3.25.6 获取申请记录

**GET** `/user/advisor/apply/history`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "requestId": 1,
      "targetAdvisorId": 2,
      "targetAdvisorName": "王强",
      "targetAdvisorAvatar": "https://example.com/avatar2.jpg",
      "reason": "希望获得基金投资方面的专业指导",
      "status": "PENDING",
      "adminNote": null,
      "createdAt": "2024-12-11T10:00:00",
      "processedAt": null
    }
  ]
}
```

---

#### 3.25.7 取消申请

**POST** `/user/advisor/apply/{requestId}/cancel`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 只能取消状态为 PENDING 的申请

---

### 3.26 顾问管理（管理端）

> 管理员审批顾问申请、直接分配顾问（需 ADMIN 角色）

#### 3.26.1 获取顾问申请列表

**GET** `/admin/advisor/requests?status=PENDING&page=1&size=20`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | string | 否 | 状态筛选：PENDING/APPROVED/REJECTED/CANCELLED |
| page | int | 否 | 页码 |
| size | int | 否 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "requestId": 1,
        "userId": 100,
        "userName": "张三",
        "userPhone": "138****0001",
        "currentAdvisorId": 1,
        "currentAdvisorName": "李明",
        "targetAdvisorId": 2,
        "targetAdvisorName": "王强",
        "reason": "希望获得基金投资方面的专业指导",
        "status": "PENDING",
        "createdAt": "2024-12-11T10:00:00"
      }
    ],
    "total": 5,
    "current": 1,
    "size": 20
  }
}
```

---

#### 3.26.2 批准申请

**POST** `/admin/advisor/requests/{requestId}/approve`

**请求体（可选）：**

```json
{
  "note": "审批通过"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.26.3 拒绝申请

**POST** `/admin/advisor/requests/{requestId}/reject`

**请求体：**

```json
{
  "note": "该顾问本月客户配额已满"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.26.4 直接分配顾问

**POST** `/admin/advisor/assign`

**请求体：**

```json
{
  "userId": 100,
  "advisorId": 2,
  "note": "根据用户需求手动分配"
}
```

| 字段        | 类型     | 必填  | 说明   |
| --------- | ------ | --- | ---- |
| userId    | long   | 是   | 用户ID |
| advisorId | long   | 是   | 顾问ID |
| note      | string | 否   | 备注   |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

> 会自动解除用户原有顾问关系，并关闭该用户所有待处理申请

---

### 3.23 顾问选择模块枚举定义

| 枚举                | 值         | 说明  |
| ----------------- | --------- | --- |
| **requestStatus** | PENDING   | 待审批 |
|                   | APPROVED  | 已批准 |
|                   | REJECTED  | 已拒绝 |
|                   | CANCELLED | 已取消 |

---

### 3.24 顾问账号管理（B2B准入）

> ⚠️ **权限要求**：需要 `ADMIN` 角色  
> **设计理念**：顾问是平台的独立专业身份（B2B准入机制），不是用户角色的一种。管理员通过后台创建顾问账号，系统同时生成顾问专业信息和登录账号。

#### 3.24.1 创建顾问账号

**POST** `/admin/advisors`

> 同时创建顾问专业信息（`advisor_db.advisors`）和登录账号（`user_db.users`），事务保证原子性

**请求体：**

```json
{
  "employeeId": "ADV3K8M2P",
  "name": "李明",
  "email": "liming@company.com",
  "phone": "13800138000",
  "specialization": ["资产配置", "基金投资", "养老规划"],
  "experienceYears": 8,
  "qualification": ["AFP", "CFP"],
  "introduction": "拥有8年金融从业经验，擅长为客户提供全方位的资产配置建议",
  "maxClients": 200,
  "username": "advisor_liming",
  "password": "InitialPass123!"
}
```

| 字段              | 类型     | 必填  | 说明                                                  |
| --------------- | ------ | --- | --------------------------------------------------- |
| employeeId      | string | 否   | 员工编号（唯一），不传则系统自动生成（格式：ADV+6位随机大写字母数字，如 `ADV3K8M2P`） |
| name            | string | 是   | 顾问姓名                                                |
| email           | string | 否   | 邮箱                                                  |
| phone           | string | 否   | 手机号                                                 |
| specialization  | array  | 否   | 专业领域（JSON数组）                                        |
| experienceYears | int    | 否   | 从业年限                                                |
| qualification   | array  | 否   | 资质认证（JSON数组）                                        |
| introduction    | string | 否   | 个人简介                                                |
| maxClients      | int    | 否   | 最大客户数，默认200                                         |
| username        | string | 是   | 登录用户名                                               |
| password        | string | 是   | 初始密码                                                |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 1,
    "employeeId": "ADV3K8M2P",
    "name": "李明",
    "email": "liming@company.com",
    "phone": "13800138000",
    "specialization": ["资产配置", "基金投资", "养老规划"],
    "experienceYears": 8,
    "qualification": ["AFP", "CFP"],
    "introduction": "拥有8年金融从业经验...",
    "maxClients": 200,
    "acceptingNew": true,
    "rating": 5.0,
    "status": 1,
    "userId": 1001,
    "clientCount": 0,
    "createdAt": "2025-12-11T20:00:00"
  }
}
```

**错误响应：**

- `员工编号已存在`（仅当手动指定编号时）
- `用户名、邮箱或手机号已被使用`

---

#### 3.24.2 更新顾问信息

**PUT** `/admin/advisors/{advisorId}`

**请求体：**

```json
{
  "name": "李明",
  "email": "liming_new@company.com",
  "phone": "13900139000",
  "specialization": ["资产配置", "基金投资", "养老规划", "税务筹划"],
  "experienceYears": 9,
  "qualification": ["AFP", "CFP", "CFA"],
  "introduction": "更新后的个人简介...",
  "maxClients": 250
}
```

> 仅更新顾问专业信息，不影响登录账号

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.24.3 获取顾问详情

**GET** `/admin/advisors/{advisorId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 1,
    "employeeId": "ADV20241211001",
    "name": "李明",
    "email": "liming@company.com",
    "phone": "13800138000",
    "avatarUrl": "https://example.com/avatar.jpg",
    "specialization": ["资产配置", "基金投资", "养老规划"],
    "experienceYears": 8,
    "qualification": ["AFP", "CFP"],
    "introduction": "拥有8年金融从业经验...",
    "maxClients": 200,
    "acceptingNew": true,
    "rating": 4.8,
    "status": 1,
    "userId": 1001,
    "clientCount": 156,
    "createdAt": "2025-12-11T20:00:00",
    "updatedAt": "2025-12-11T20:30:00"
  }
}
```

| 字段              | 类型      | 说明           |
| --------------- | ------- | ------------ |
| advisorId       | long    | 顾问ID         |
| employeeId      | string  | 员工编号         |
| name            | string  | 顾问姓名         |
| specialization  | array   | 专业领域         |
| experienceYears | int     | 从业年限         |
| qualification   | array   | 资质认证         |
| introduction    | string  | 个人简介         |
| maxClients      | int     | 最大客户数        |
| acceptingNew    | boolean | 是否接受新客户      |
| rating          | decimal | 评分（1.0-5.0）  |
| status          | int     | 状态：0=禁用，1=启用 |
| userId          | long    | 关联的登录账号ID    |
| clientCount     | int     | 当前客户数        |

---

#### 3.24.4 分页查询顾问列表

**GET** `/admin/advisors?status=1&keyword=李明&page=1&size=20`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | int | 否 | 状态筛选：0=禁用，1=启用 |
| keyword | string | 否 | 关键词搜索（姓名/员工编号） |
| page | int | 否 | 页码，默认1 |
| size | int | 否 | 每页条数，默认20 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "advisorId": 1,
        "employeeId": "ADV20241211001",
        "name": "李明",
        "email": "liming@company.com",
        "phone": "13800138000",
        "specialization": ["资产配置", "基金投资"],
        "experienceYears": 8,
        "rating": 4.8,
        "status": 1,
        "userId": 1001,
        "clientCount": 156,
        "createdAt": "2025-12-11T20:00:00"
      }
    ],
    "total": 25,
    "current": 1,
    "size": 20,
    "pages": 2
  }
}
```

---

#### 3.24.5 禁用顾问

**POST** `/admin/advisors/{advisorId}/disable`

> 禁用顾问后，顾问无法登录，也不会出现在用户可选顾问列表中

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.24.6 启用顾问

**POST** `/admin/advisors/{advisorId}/enable`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 3.24.7 删除顾问

**DELETE** `/admin/advisors/{advisorId}`

> 同时删除顾问专业信息和登录账号。**注意：需确保顾问无客户后才能删除**

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

**错误响应：**

- `顾问仍有关联客户，无法删除`

---

### 3.25 顾问账号管理枚举定义

| 枚举                | 值   | 说明     |
| ----------------- | --- | ------ |
| **advisorStatus** | 0   | 禁用     |
|                   | 1   | 启用     |
| **acceptingNew**  | 0   | 不接受新客户 |
|                   | 1   | 接受新客户  |

---

### 3.27 顾问体系架构说明

```
┌─────────────────────────────────────────────────────────────────┐
│                        顾问体系架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    B2B准入     ┌──────────────────┐          │
│  │   管理员      │ ───────────→  │ advisor_db.advisors │          │
│  │ (ROLE_ADMIN) │    创建账号    │   (顾问专业信息)    │          │
│  └──────────────┘               └────────┬─────────┘          │
│                                          │                     │
│                                   1:1 关联│advisor_id          │
│                                          ↓                     │
│                                 ┌──────────────────┐          │
│                                 │  user_db.users   │          │
│                                 │   (登录账号)      │          │
│                                 └────────┬─────────┘          │
│                                          │                     │
│                                   登录时 │ 检查 advisor_id     │
│                                          ↓                     │
│                                 ┌──────────────────┐          │
│                                 │ 返回 ADVISOR 角色 │          │
│                                 └──────────────────┘          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  关键表：                                                        │
│  • advisor_db.advisors - 顾问专业信息（员工编号、资质、评分等）      │
│  • advisor_db.client_relationships - 顾问-客户关系               │
│  • user_db.users.advisor_id - 关联顾问身份                       │
│  • user_db.advisor_change_request - 用户申请更换顾问              │
└─────────────────────────────────────────────────────────────────┘
```

**核心流程：**

1. **管理员创建顾问** → `POST /admin/advisors` → 同时创建 `advisors` + `users` 记录
2. **顾问登录** → `POST /auth/login` → 系统检测 `user.advisor_id` 不为空 → 返回 `ADVISOR` 角色
3. **用户申请顾问** → `POST /user/advisor/apply` → 创建申请 → 管理员审批 → 建立关系
4. **顾问服务客户** → 使用 `ADVISOR` 角色权限的接口（`/user/advisor/**`）

---

## 4. 资产服务接口

> **枚举定义：**
> 
> | 枚举                  | 值   | 说明   |
> | ------------------- | --- | ---- |
> | **assetType**       | 1   | 现金   |
> |                     | 2   | 基金   |
> |                     | 3   | 股票   |
> |                     | 4   | 债券   |
> |                     | 5   | 理财产品 |
> | **transactionType** | 1   | 买入   |
> |                     | 2   | 卖出   |
> |                     | 3   | 分红   |
> |                     | 4   | 转入   |
> |                     | 5   | 转出   |

### 4.1 获取资产总览

**GET** `/asset/overview`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 1,
    "userId": 1001,
    "totalAssets": 500000.00,
    "totalLiabilities": 50000.00,
    "netAssets": 450000.00,
    "totalEarnings": 25000.00,
    "dailyEarnings": 150.00,
    "lastUpdateTime": "2024-11-28T10:00:00"
  }
}
```

---

### 4.2 获取资产类型占比（饼图数据）

**GET** `/asset/overview/type-share`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {"assetType": 1, "typeName": "现金", "amount": 100000.00, "percentage": 20.0},
    {"assetType": 2, "typeName": "基金", "amount": 200000.00, "percentage": 40.0},
    {"assetType": 3, "typeName": "股票", "amount": 150000.00, "percentage": 30.0},
    {"assetType": 4, "typeName": "债券", "amount": 50000.00, "percentage": 10.0}
  ]
}
```

---

### 4.3 获取资产趋势（折线图数据）

**GET** `/asset/overview/trend?days=30`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| days | int | 否 | 30 | 天数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {"snapshotDate": "2024-11-01", "totalAssets": 480000.00},
    {"snapshotDate": "2024-11-02", "totalAssets": 485000.00},
    {"snapshotDate": "2024-11-28", "totalAssets": 500000.00}
  ]
}
```

---

### 4.4 查询资产明细列表

**GET** `/asset/details?assetType=2`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| assetType | int | 否 | 资产类型：1=现金，2=基金，3=股票，4=债券 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "userId": 1001,
      "assetType": 2,
      "assetName": "招商中证白酒指数",
      "productCode": "161725",
      "holdingAmount": 10000.00,
      "holdingShares": 500.00,
      "costPrice": 20.00,
      "currentPrice": 22.50,
      "marketValue": 11250.00,
      "earnings": 1250.00,
      "earningsRate": 12.5
    }
  ]
}
```

---

### 4.5 新增资产明细

**POST** `/asset/details`

**请求体：**

```json
{
  "assetType": 1,
  "assetName": "活期存款",
  "assetCode": "CASH001",
  "quantity": 50000.00,
  "costPrice": 1.00,
  "marketPrice": 1.00,
  "totalValue": 50000.00,
  "currency": "CNY"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "detailId": 1,
    "userId": 1001,
    "assetType": 1,
    "assetName": "活期存款",
    "assetCode": "CASH001",
    "quantity": 50000.00,
    "costPrice": 1.00,
    "marketPrice": 1.00,
    "totalValue": 50000.00,
    "profitLoss": 0.00,
    "currency": "CNY",
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T10:30:00"
  }
}
```

---

### 4.6 更新资产明细

**PUT** `/asset/details/{detailId}`

**请求体：**

```json
{
  "assetName": "活期存款-更新",
  "totalValue": 60000.00
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "detailId": 1,
    "userId": 1001,
    "assetType": 1,
    "assetName": "活期存款-更新",
    "assetCode": "CASH001",
    "quantity": 60000.00,
    "costPrice": 1.00,
    "marketPrice": 1.00,
    "totalValue": 60000.00,
    "profitLoss": 0.00,
    "currency": "CNY",
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T11:00:00"
  }
}
```

---

### 4.7 删除资产明细

**DELETE** `/asset/details/{detailId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 4.8 资产归集（合并）

**POST** `/asset/details/merge?detailIds=1,2,3&newName=合并资产`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| detailIds | List<Long> | 是 | 要合并的资产ID列表，至少2个 |
| newName | string | 否 | 合并后的名称 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "detailId": 10,
    "userId": 1001,
    "assetType": 2,
    "assetName": "合并资产",
    "assetCode": null,
    "quantity": 1500.00,
    "costPrice": 10.00,
    "marketPrice": 12.00,
    "totalValue": 18000.00,
    "profitLoss": 3000.00,
    "currency": "CNY",
    "createdAt": "2024-11-28T11:00:00",
    "updatedAt": "2024-11-28T11:00:00"
  }
}
```

---

### 4.9 资产拆分

**POST** `/asset/details/{detailId}/split?amounts=10000,20000,30000`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| amounts | List<Double> | 是 | 拆分金额列表 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "detailId": 11,
      "userId": 1001,
      "assetType": 2,
      "assetName": "资产拆分-1",
      "totalValue": 10000.00,
      "createdAt": "2024-11-28T11:00:00"
    },
    {
      "detailId": 12,
      "userId": 1001,
      "assetType": 2,
      "assetName": "资产拆分-2",
      "totalValue": 20000.00,
      "createdAt": "2024-11-28T11:00:00"
    },
    {
      "detailId": 13,
      "userId": 1001,
      "assetType": 2,
      "assetName": "资产拆分-3",
      "totalValue": 30000.00,
      "createdAt": "2024-11-28T11:00:00"
    }
  ]
}
```

---

### 4.10 记录资产快照

**POST** `/asset/overview/snapshot`

> 手动触发记录当日资产快照

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "snapshotId": 30,
    "userId": 1001,
    "snapshotDate": "2024-11-28",
    "totalAssets": 500000.00,
    "cashAssets": 100000.00,
    "investmentAssets": 350000.00,
    "otherAssets": 50000.00,
    "totalLiabilities": 50000.00,
    "netAssets": 450000.00,
    "createdAt": "2024-11-28T10:30:00"
  }
}
```

---

### 4.11 导出资产明细

**GET** `/asset/details/export?assetType=2`

> 返回 Excel 文件下载

---

### 4.12 分页查询资产交易流水

**GET** `/asset/transactions?assetType=2&startDate=2024-11-01&endDate=2024-11-30&page=1&size=10`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| assetType | int | 否 | - | 资产类型 |
| startDate | date | 否 | - | 开始日期 (yyyy-MM-dd) |
| endDate | date | 否 | - | 结束日期 (yyyy-MM-dd) |
| page | int | 否 | 1 | 页码 |
| size | int | 否 | 10 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "id": 1,
        "userId": 1001,
        "assetType": 2,
        "transactionType": 1,
        "amount": 10000.00,
        "transactionTime": "2024-11-28T10:30:00",
        "description": "买入招商中证白酒指数基金"
      }
    ],
    "total": 1,
    "size": 10,
    "current": 1,
    "pages": 1
  }
}
```

> transactionType: 1=买入，2=卖出，3=分红，4=转入，5=转出

---

### 4.13 查询资产收益趋势

**GET** `/asset/earnings/trend?assetType=2&days=30`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| assetType | int | 否 | - | 资产类型 |
| days | int | 否 | 30 | 天数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "userId": 1001,
      "assetType": 2,
      "earningDate": "2024-11-01",
      "earningAmount": 150.00,
      "earningRate": 0.0150
    },
    {
      "id": 2,
      "userId": 1001,
      "assetType": 2,
      "earningDate": "2024-11-02",
      "earningAmount": 180.00,
      "earningRate": 0.0175
    }
  ]
}
```

---

## 5. 交易服务接口

> **枚举定义：**
> 
> | 枚举              | 值   | 说明  |
> | --------------- | --- | --- |
> | **orderType**   | 1   | 买入  |
> |                 | 2   | 卖出  |
> | **orderStatus** | 1   | 待处理 |
> |                 | 2   | 处理中 |
> |                 | 3   | 已成交 |
> |                 | 4   | 已取消 |
> |                 | 5   | 失败  |

### 5.1 创建订单（买入/卖出）

**POST** `/trade/orders`

**请求体：**

```json
{
  "productId": 2001,
  "orderType": 1,
  "quantity": 100.00,
  "price": 10.50,
  "amount": 1050.00,
  "tradePassword": "123456",
  "largeTradeCode": "654321"
}
```

| 字段             | 类型      | 必填  | 说明                 |
| -------------- | ------- | --- | ------------------ |
| productId      | long    | 是   | 产品ID               |
| orderType      | int     | 是   | 订单类型：1=买入，2=卖出     |
| quantity       | decimal | 是   | 数量/份额              |
| price          | decimal | 否   | 单价                 |
| amount         | decimal | 是   | 交易金额               |
| tradePassword  | string  | 是   | 交易密码               |
| largeTradeCode | string  | 否   | 大额交易验证码（金额≥10万时必填） |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "orderId": 10001,
    "orderNo": "O202411281030001001",
    "userId": 1001,
    "productId": 2001,
    "orderType": 1,
    "orderStatus": 1,
    "quantity": 100.00,
    "price": 10.50,
    "amount": 1050.00,
    "fee": 0.00,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T10:30:00"
  }
}
```

---

### 5.2 撤销订单

**POST** `/trade/orders/cancel`

**请求体：**

```json
{
  "orderId": 10001,
  "tradePassword": "123456"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 5.3 查询订单详情

**GET** `/trade/orders/{orderId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "orderId": 10001,
    "orderNo": "O202411281030001001",
    "userId": 1001,
    "productId": 2001,
    "orderType": 1,
    "orderStatus": 3,
    "quantity": 100.00,
    "price": 10.50,
    "amount": 1050.00,
    "fee": 5.25,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T10:35:00"
  }
}
```

---

### 5.4 查询订单列表

**GET** `/trade/orders`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "orderId": 10001,
      "orderNo": "O202411281030001001",
      "userId": 1001,
      "productId": 2001,
      "orderType": 1,
      "orderStatus": 3,
      "quantity": 100.00,
      "price": 10.50,
      "amount": 1050.00,
      "fee": 5.25,
      "createdAt": "2024-11-28T10:30:00",
      "updatedAt": "2024-11-28T10:35:00"
    }
  ]
}
```

---

### 5.5 分页查询订单（带过滤）

**GET** `/trade/orders/filter?status=1&type=1&page=1&size=10`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| status | int | 否 | - | 订单状态：1=待支付，2=已支付，3=已完成，4=已取消，5=失败 |
| type | int | 否 | - | 订单类型：1=买入，2=卖出 |
| page | int | 否 | 1 | 页码 |
| size | int | 否 | 10 | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "orderId": 10001,
      "orderNo": "O202411281030001001",
      "userId": 1001,
      "productId": 2001,
      "orderType": 1,
      "orderStatus": 1,
      "quantity": 100.00,
      "price": 10.50,
      "amount": 1050.00,
      "fee": 0.00,
      "createdAt": "2024-11-28T10:30:00",
      "updatedAt": "2024-11-28T10:30:00"
    }
  ]
}
```

---

### 5.6 赎回申请

**POST** `/trade/orders/{orderId}/redeem?amount=5000&tradePassword=123456`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| amount | decimal | 否 | 赎回金额，不填则全额赎回 |
| tradePassword | string | 是 | 交易密码 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "orderId": 10002,
    "orderNo": "O202411281100001001",
    "userId": 1001,
    "productId": 2001,
    "orderType": 2,
    "orderStatus": 1,
    "quantity": 50.00,
    "price": 10.50,
    "amount": 5000.00,
    "fee": 0.00,
    "createdAt": "2024-11-28T11:00:00",
    "updatedAt": "2024-11-28T11:00:00"
  }
}
```

---

### 5.7 失败订单重试

**POST** `/trade/orders/{orderId}/retry?tradePassword=123456`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "orderId": 10001,
    "orderNo": "O202411281030001001",
    "userId": 1001,
    "productId": 2001,
    "orderType": 1,
    "orderStatus": 2,
    "quantity": 100.00,
    "price": 10.50,
    "amount": 1050.00,
    "fee": 5.25,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T11:30:00"
  }
}
```

---

### 5.8 获取赎回提醒列表

**GET** `/trade/orders/reminders`

> 返回即将到期或建议赎回的订单列表

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "orderId": 10003,
      "orderNo": "O202410281030001001",
      "userId": 1001,
      "productId": 2003,
      "orderType": 1,
      "orderStatus": 3,
      "quantity": 500.00,
      "price": 10.00,
      "amount": 5000.00,
      "fee": 25.00,
      "createdAt": "2024-10-28T10:30:00",
      "updatedAt": "2024-10-28T10:35:00"
    }
  ]
}
```

---

### 5.9 导出订单列表

**GET** `/trade/orders/export?status=3&type=1`

> 返回 Excel 文件下载

---

### 5.10 获取行情概览

**GET** `/trade/market/overview`

> 使用 AkShare + Python 数据服务获取实时行情，数据每 5 分钟同步至 MySQL

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "indexList": [
      {
        "code": "000001",
        "name": "上证指数",
        "value": 3150.25,
        "change": 0.85,
        "changePercent": 0.85,
        "high": 3165.00,
        "low": 3140.00,
        "volume": 325000000,
        "turnover": 420000000000
      },
      { "code": "399001", "name": "深证成指", "value": 10250.50, "change": -0.35 },
      { "code": "399006", "name": "创业板指", "value": 2050.80, "change": 1.25 }
    ],
    "hotStocks": [
      {
        "code": "600519",
        "name": "贵州茅台",
        "price": 1850.00,
        "changePercent": 1.25,
        "volume": 1500000,
        "turnover": 2800000000
      },
      { "code": "000858", "name": "五粮液", "price": 165.50, "changePercent": -0.85 }
    ],
    "updateTime": "2025-12-11 00:15:00"
  }
}
```

---

### 5.11 自选产品管理

#### 5.11.1 获取自选列表

**GET** `/trade/market/watchlist`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "productId": 2001,
      "productName": "招商中证白酒指数",
      "productCode": "161725",
      "productType": 2,
      "riskLevel": 3,
      "currentPrice": 1.2500,
      "changeRate": 2.35,
      "lastUpdateTime": "2025-12-10T15:00:00",
      "sortOrder": 0
    }
  ]
}
```

#### 5.11.2 添加自选

**POST** `/trade/market/watchlist`

支持通过 `productId` 或 `code` 添加自选，**至少提供一个参数**。

| 参数        | 类型     | 必填  | 说明                          |
| --------- | ------ | --- | --------------------------- |
| productId | long   | 否   | 产品ID（与 code 二选一）            |
| code      | string | 否   | 产品代码，如 AAPL、000001.SZ（与 productId 二选一） |

**请求示例：**

```
# 通过产品ID添加
POST /trade/market/watchlist?productId=2001

# 通过产品代码添加（适用于热门股票）
POST /trade/market/watchlist?code=AAPL
POST /trade/market/watchlist?code=000001.SZ
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

> **注意**：通过 `code` 添加自选时，如果产品不存在于数据库，系统会**自动创建**产品记录。
> 
> 自动识别规则：
> - `*.SZ` / `*.SH` → A股（CNY）
> - 纯大写字母（如 `AAPL`）→ 美股（USD）
> - `*.HK` → 港股（HKD）
> - 其他 → 其他类型（USD）

#### 5.11.3 移除自选（通过产品ID）

**DELETE** `/trade/market/watchlist/{productId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

#### 5.11.4 移除自选（通过产品代码）

**DELETE** `/trade/market/watchlist/code/{code}`

| 参数   | 类型     | 必填  | 说明                   |
| ---- | ------ | --- | -------------------- |
| code | string | 是   | 产品代码，如 AAPL、000001.SZ |

**请求示例：**

```
DELETE /trade/market/watchlist/code/AAPL
DELETE /trade/market/watchlist/code/000001.SZ
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 5.12 多产品对比

**POST** `/trade/products/compare`

> 最多支持 5 个产品同时对比

**请求体：**

```json
{
  "productIds": [1, 2, 3, 4, 5]
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "products": [
      {
        "productId": 1,
        "productName": "招商中证白酒指数",
        "productCode": "161725",
        "riskLevel": 3,
        "returnRate1Y": 15.5,
        "returnRate3Y": 45.2,
        "volatility": 18.3,
        "fees": {
          "purchase": 0.15,
          "redeem": 0.5,
          "management": 1.0
        }
      }
    ]
  }
}
```

---

### 5.13 获取最大可投金额

**GET** `/trade/orders/max-amount?productId=2001`

| 参数        | 类型   | 必填  | 说明   |
| --------- | ---- | --- | ---- |
| productId | long | 是   | 产品ID |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "maxAmount": 50000.00,
    "availableBalance": 100000.00,
    "productLimit": 50000.00,
    "riskLimitRemaining": 80000.00
  }
}
```

| 字段                 | 说明             |
| ------------------ | -------------- |
| maxAmount          | 最大可投金额（取三者最小值） |
| availableBalance   | 账户可用余额         |
| productLimit       | 产品投资限额         |
| riskLimitRemaining | 风控剩余额度         |

---

### 5.14 股票K线数据

**GET** `/trade/market/stock/kline?region=us&code=AAPL&kType=8&limit=100`

> 基于 iTick API 获取真实股票 K 线数据

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| region | string | 否 | us | 地区代码：`us`=美股，`hk`=港股，`cn`=A股 |
| code | string | 是 | - | 股票代码，如 `AAPL`、`TSLA` |
| kType | int | 否 | 8 | K线周期：1=1分钟，5=5分钟，8=日线，9=周线，10=月线 |
| limit | int | 否 | 100 | 返回数据条数，最大500 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "time": "2025-12-10T00:00:00",
      "timestamp": 1733788800000,
      "open": 248.50,
      "high": 252.30,
      "low": 247.80,
      "close": 251.25,
      "volume": 45000000,
      "turnover": 11250000000
    }
  ]
}
```

---

### 5.15 股票实时报价

**GET** `/trade/market/stock/quote?region=us&code=AAPL`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| region | string | 否 | us | 地区代码 |
| code | string | 是 | - | 股票代码 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "code": "AAPL",
    "price": 251.25,
    "preClose": 248.50,
    "open": 248.80,
    "high": 252.30,
    "low": 247.80,
    "bidPrice": 251.20,
    "askPrice": 251.30,
    "bidVolume": 500,
    "askVolume": 800,
    "volume": 45000000,
    "turnover": 11250000000,
    "change": 1.11,
    "updateTime": "2025-12-10T16:00:00"
  }
}
```

---

### 5.16 股票盘口深度

**GET** `/trade/market/stock/depth?region=us&code=AAPL`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "asks": [
      [251.30, 800],
      [251.35, 1200],
      [251.40, 500]
    ],
    "bids": [
      [251.20, 500],
      [251.15, 1000],
      [251.10, 750]
    ]
  }
}
```

> `asks`=卖盘，`bids`=买盘，每项格式为 `[价格, 数量]`

---

### 5.17 加密货币K线数据

**GET** `/trade/market/crypto/kline?code=BTCUSDT&kType=8&limit=100`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| code | string | 是 | - | 交易对，如 `BTCUSDT`、`ETHUSDT` |
| kType | int | 否 | 8 | K线周期 |
| limit | int | 否 | 100 | 返回数据条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "time": "2025-12-10T00:00:00",
      "timestamp": 1733788800000,
      "open": 97500.00,
      "high": 99200.00,
      "low": 96800.00,
      "close": 98500.00,
      "volume": 125000.00,
      "turnover": 12312500000
    }
  ]
}
```

---

### 5.18 加密货币实时报价

**GET** `/trade/market/crypto/quote?code=BTCUSDT`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "code": "BTCUSDT",
    "price": 98500.00,
    "preClose": 97500.00,
    "open": 97600.00,
    "high": 99200.00,
    "low": 96800.00,
    "bidPrice": 98495.00,
    "askPrice": 98505.00,
    "volume": 125000.00,
    "change": 1.03,
    "updateTime": "2025-12-10T21:30:00"
  }
}
```

---

### 5.19 外汇实时报价

**GET** `/trade/market/forex/quote?code=EURUSD`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| code | string | 是 | 货币对，如 `EURUSD`、`GBPUSD`、`USDJPY`、`USDCNH` |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "code": "EURUSD",
    "price": 1.0525,
    "preClose": 1.0540,
    "open": 1.0538,
    "high": 1.0560,
    "low": 1.0510,
    "bidPrice": 1.0524,
    "askPrice": 1.0526,
    "change": -0.14,
    "updateTime": "2025-12-10T21:30:00"
  }
}
```

---

### 5.20 K线周期类型说明

| kType | 说明   |
| ----- | ---- |
| 1     | 1分钟  |
| 2     | 3分钟  |
| 3     | 5分钟  |
| 4     | 15分钟 |
| 5     | 30分钟 |
| 6     | 1小时  |
| 7     | 4小时  |
| 8     | 日线   |
| 9     | 周线   |
| 10    | 月线   |

---

## 6. AI理财服务接口

> **v1.17.0 更新**：AI 对话默认使用 SSE 流式输出，实现打字机效果  
> **v1.7.0 重构说明**：AI 对话模块已统一重构，支持多轮对话和会话管理。旧接口 `/ai/qa/*` 已废弃。

### 6.1 发送消息（流式 SSE）

**GET** `/ai/chat/ask?message=如何配置基金组合&sessionId=abc123&sessionType=general`

> ⚠️ **v1.17.0 变更**：接口方法从 POST 改为 GET，响应格式从 JSON 改为 SSE 流式输出。

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| message | string | 是 | 用户消息 |
| sessionId | string | 否 | 会话ID，不填则自动创建新会话（UUID格式） |
| sessionType | string | 否 | 会话类型：`general`=通用问答（默认），`plan`=规划相关，`risk`=风险相关 |

**响应格式：** `text/event-stream` (SSE)

**SSE 数据流示例：**

```
data: 根据您的

data: 风险评估结果，

data: 建议您采用以下

data: 配置方案：\n1. 股票型基金

data: 40%\n2. 债券型

data: 基金 40%\n3.

data: 货币基金 20%

data: [DONE]
```

**特殊标记：**
| 标记 | 说明 |
|------|------|
| `data: [DONE]` | 流式输出完成 |
| `data: [ERROR] xxx` | 发生错误，后跟错误信息 |

**前端调用示例（JavaScript fetch）：**

```javascript
async function sendMessage(message, sessionId) {
  const params = new URLSearchParams({ message });
  if (sessionId) params.append('sessionId', sessionId);

  const response = await fetch(`/ai/chat/ask?${params}`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Accept': 'text/event-stream'
    }
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let fullContent = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value);
    for (const line of text.split('\n')) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') {
          console.log('对话完成');
          return fullContent;
        } else if (data.startsWith('[ERROR]')) {
          throw new Error(data);
        } else {
          // 解码换行符
          const chunk = data.replace(/\\n/g, '\n');
          fullContent += chunk;
          updateChatUI(fullContent);  // 实时更新 UI
        }
      }
    }
  }
  return fullContent;
}
```

**Vue 3 + TypeScript 示例：**

```typescript
import { ref } from 'vue';

const chatContent = ref('');
const isStreaming = ref(false);

async function chat(message: string, sessionId?: string) {
  isStreaming.value = true;
  chatContent.value = '';

  const params = new URLSearchParams({ message });
  if (sessionId) params.append('sessionId', sessionId);

  try {
    const response = await fetch(`/ai/chat/ask?${params}`, {
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`,
        'Accept': 'text/event-stream'
      }
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      for (const line of decoder.decode(value).split('\n')) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          if (data === '[DONE]') break;
          if (!data.startsWith('[ERROR]')) {
            chatContent.value += data.replace(/\\n/g, '\n');
          }
        }
      }
    }
  } finally {
    isStreaming.value = false;
  }
}
```

> **注意事项**：
> 1. 接口使用 **GET** 方法
> 2. 需要在请求头中携带 `Authorization: Bearer {token}`
> 3. 换行符在传输时会被转义为 `\n`，前端需要解码还原
> 4. 会话消息会在流式输出完成后**自动保存到数据库**
> 5. sessionId 首次可不传，会自动生成并在后续请求中携带以保持上下文

---

### 6.2 获取会话列表

**GET** `/ai/chat/sessions?limit=20`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| limit | int | 否 | 20 | 返回条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "sessionId": "abc123def456",
      "sessionType": "general",
      "title": "如何配置基金组合",
      "messageCount": 10,
      "lastMessageTime": "2024-12-03T15:30:00",
      "createdAt": "2024-12-03T10:30:00"
    },
    {
      "sessionId": "xyz789ghi012",
      "sessionType": "plan",
      "title": "养老规划咨询",
      "messageCount": 6,
      "lastMessageTime": "2024-12-02T18:00:00",
      "createdAt": "2024-12-02T14:00:00"
    }
  ]
}
```

**响应字段说明：**
| 字段 | 类型 | 说明 |
|------|------|------|
| sessionId | string | 会话ID |
| sessionType | string | 会话类型 |
| title | string | 会话标题（第一条用户消息的前50字符） |
| messageCount | int | 消息数量 |
| lastMessageTime | datetime | 最后消息时间 |
| createdAt | datetime | 会话创建时间 |

---

### 6.3 获取会话详情（消息列表）

**GET** `/ai/chat/sessions/{sessionId}?limit=100`

**路径参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| sessionId | string | 是 | 会话ID |

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| limit | int | 否 | 100 | 返回条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "userId": 1001,
      "sessionId": "abc123def456",
      "sessionType": "general",
      "role": "user",
      "content": "如何配置基金组合",
      "createdAt": "2024-12-03T10:30:00",
      "isDeleted": false
    },
    {
      "id": 2,
      "userId": 1001,
      "sessionId": "abc123def456",
      "sessionType": "general",
      "role": "assistant",
      "content": "根据您的风险评估结果，建议您采用以下配置方案...",
      "createdAt": "2024-12-03T10:30:05",
      "isDeleted": false
    }
  ]
}
```

---

### 6.4 删除会话

**DELETE** `/ai/chat/sessions/{sessionId}`

**路径参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| sessionId | string | 是 | 会话ID |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 6.5 分页查询推荐记录

**GET** `/ai/recommendations?type=1&page=1&size=10`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | int | 否 | 推荐类型：1=产品推荐，2=资产配置，3=风险提示 |
| page | int | 否 | 页码，默认1 |
| size | int | 否 | 每页条数，默认10 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "recommendationId": 1,
        "userId": 1001,
        "productId": 2001,
        "recommendationType": 1,
        "recommendationScore": 0.8500,
        "recommendationReason": "该基金近3年收益率达到45%，波动率较低...",
        "createdAt": "2024-11-28T10:30:00"
      }
    ],
    "total": 1,
    "size": 10,
    "current": 1,
    "pages": 1
  }
}
```

---

### 6.6 生成新的智能推荐

**POST** `/ai/recommendations/generate?productId=2001&type=1&riskLevel=3&goal=养老`

| 参数        | 类型     | 必填  | 说明       |
| --------- | ------ | --- | -------- |
| productId | long   | 是   | 产品ID     |
| type      | int    | 否   | 推荐类型，默认1 |
| riskLevel | int    | 否   | 风险等级     |
| goal      | string | 否   | 理财目标     |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "recommendationId": 10,
    "userId": 1001,
    "productId": 2001,
    "recommendationType": 1,
    "recommendationScore": 0.8200,
    "recommendationReason": "基于您的养老规划目标和平衡型风险偏好，该产品历史年化收益稳定在8%左右...",
    "createdAt": "2024-11-28T11:00:00"
  }
}
```

---

### 6.7 智能产品推荐

**POST** `/ai/recommendations/smart?riskLevel=3&goal=子女教育`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| riskLevel | int | 否 | 风险等级，不填则自动获取用户风险评估结果 |
| goal | string | 否 | 理财目标 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "recommendationId": 1,
      "userId": 1001,
      "productId": 2001,
      "recommendationType": 1,
      "recommendationScore": 0.8500,
      "recommendationReason": "该基金近3年收益率达到45%，波动率较低，适合您的稳健型风险偏好...",
      "createdAt": "2024-11-28T10:30:00"
    }
  ]
}
```

---

### 6.8 智能产品推荐（带缓存）

**POST** `/ai/recommendations/smart-cached?riskLevel=3&goal=子女教育`

> 优先从缓存获取推荐结果，缓存失效时重新生成

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "recommendationId": 1,
      "userId": 1001,
      "productId": 2001,
      "recommendationType": 1,
      "recommendationScore": 0.8500,
      "recommendationReason": "该基金近3年收益率达到45%，波动率较低...",
      "createdAt": "2024-11-28T10:30:00"
    }
  ]
}
```

---

### 6.9 清除推荐缓存

**DELETE** `/ai/recommendations/cache`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 6.10 生成理财方案

**POST** `/ai/plan/generate?title=子女教育规划&goal=5年内攒够50万教育金&riskLevel=2`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| title | string | 是 | 方案标题 |
| goal | string | 是 | 理财目标描述 |
| riskLevel | int | 否 | 风险等级 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "planId": 1,
    "userId": 1001,
    "planTitle": "子女教育规划",
    "planContent": "## 一、目标分析\n您的目标是5年内积累50万...\n\n## 二、资产配置建议\n...",
    "riskLevel": 2,
    "expectedReturn": 0.0600,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T10:30:00"
  }
}
```

**响应字段说明：**
| 字段 | 类型 | 说明 |
|------|------|------|
| planId | long | 方案ID |
| planTitle | string | 方案标题 |
| planContent | string | 方案内容（Markdown格式） |
| riskLevel | int | 风险等级 |
| expectedReturn | decimal | 预期年化收益率 |
| createdAt | datetime | 创建时间 |
| updatedAt | datetime | 更新时间 |

---

### 6.11 生成高级理财方案（含收益测算）

**POST** `/ai/plan/advanced-generate`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| title | string | 是 | 方案标题 |
| goal | string | 是 | 理财目标 |
| riskLevel | int | 否 | 风险等级 |
| initialCapital | decimal | 否 | 初始资金 |
| monthlyInvestment | decimal | 否 | 每月定投金额 |
| years | int | 否 | 投资年限 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "plans": [
      {
        "planId": "conservative",
        "riskLevel": 2,
        "title": "保守型配置",
        "goal": "60岁前积累200万养老金",
        "expectedReturnRate": 0.05,
        "expectedFinalAmount": 1850000.00,
        "downsideAmount": 1650000.00,
        "assetAllocation": [
          {"assetType": "债券基金", "percentage": 60},
          {"assetType": "货币基金", "percentage": 30},
          {"assetType": "股票基金", "percentage": 10}
        ],
        "description": "该方案风险较低，适合保守型投资者..."
      },
      {
        "planId": "balanced",
        "riskLevel": 3,
        "title": "平衡型配置",
        "goal": "60岁前积累200万养老金",
        "expectedReturnRate": 0.08,
        "expectedFinalAmount": 2650000.00,
        "downsideAmount": 2100000.00,
        "assetAllocation": [
          {"assetType": "债券基金", "percentage": 40},
          {"assetType": "货币基金", "percentage": 20},
          {"assetType": "股票基金", "percentage": 40}
        ],
        "description": "该方案风险收益平衡，适合稳健型投资者..."
      }
    ]
  }
}
```

**响应字段说明：**
| 字段 | 类型 | 说明 |
|------|------|------|
| plans | array | 多套方案列表 |
| planId | string | 方案标识 |
| expectedReturnRate | decimal | 预期年化收益率 |
| expectedFinalAmount | decimal | 预期最终金额 |
| downsideAmount | decimal | 悲观情况下金额 |
| assetAllocation | array | 资产配置比例 |
| description | string | 方案说明 |

---

### 6.12 查询理财方案列表

**GET** `/ai/plan/list`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "planId": 1,
      "userId": 1001,
      "planTitle": "子女教育规划",
      "planContent": "## 一、目标分析...",
      "riskLevel": 2,
      "expectedReturn": 0.0600,
      "createdAt": "2024-11-28T10:30:00",
      "updatedAt": "2024-11-28T10:30:00"
    }
  ]
}
```

---

### 6.13 查询理财方案详情

**GET** `/ai/plan/{planId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "planId": 1,
    "userId": 1001,
    "planTitle": "子女教育规划",
    "planContent": "## 一、目标分析\n您的目标是5年内积累50万教育金...\n\n## 二、资产配置建议\n1. 债券型基金 40%\n2. 股票型基金 30%\n3. 货币基金 30%\n\n## 三、执行建议\n...",
    "riskLevel": 2,
    "expectedReturn": 0.0600,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T10:30:00"
  }
}
```

---

### 6.14 生成方案语音讲解

**GET** `/ai/plan/{planId}/narration`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": "您好，这是为您定制的子女教育规划方案。根据您5年内积累50万教育金的目标，结合您的稳健型风险偏好，我们建议您采用以下配置方案..."
}
```

---

### 6.15 调整理财方案

**POST** `/ai/plan/{planId}/adjust?request=增加股票配置比例`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "planId": 1,
    "userId": 1001,
    "planTitle": "子女教育规划（已调整）",
    "planContent": "## 一、目标分析\n...\n\n## 二、调整后资产配置建议\n1. 股票型基金 50%\n2. 债券型基金 30%\n3. 货币基金 20%\n\n## 三、调整说明\n根据您的要求，已增加股票配置比例...",
    "riskLevel": 3,
    "expectedReturn": 0.0800,
    "createdAt": "2024-11-28T10:30:00",
    "updatedAt": "2024-11-28T11:30:00"
  }
}
```

---

### 6.16 获取可用AI模型列表

**GET** `/ai/model/list`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "modelId": "qwen-max",
      "modelName": "通义千问-Max",
      "description": "阿里云通义千问大模型",
      "enabled": true
    },
    {
      "modelId": "qwen-plus",
      "modelName": "通义千问-Plus",
      "description": "轻量级通义千问",
      "enabled": true
    }
  ]
}
```

---

### 6.17 获取当前使用的模型

**GET** `/ai/model/current`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "modelId": "qwen-max",
    "modelName": "通义千问-Max",
    "description": "阿里云通义千问大模型",
    "enabled": true
  }
}
```

---

### 6.18 切换AI模型

**POST** `/ai/model/switch?modelId=qwen-plus`

| 参数      | 类型     | 必填  | 说明   |
| ------- | ------ | --- | ---- |
| modelId | string | 是   | 模型ID |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

### 6.19 检查模型健康状态

**GET** `/ai/model/health/{modelId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": true
}
```

---

## 7. 内容服务接口

> **枚举定义：**
> 
> | 枚举                   | 值   | 说明      |
> | -------------------- | --- | ------- |
> | **status** (内容状态)    | 0   | 草稿      |
> |                      | 1   | 待审核     |
> |                      | 2   | 已发布     |
> |                      | 3   | 已下架     |
> |                      | 4   | 审核拒绝    |
> | **riskLevel** (风险等级) | 1   | R1-低风险  |
> |                      | 2   | R2-中低风险 |
> |                      | 3   | R3-中风险  |
> |                      | 4   | R4-中高风险 |
> |                      | 5   | R5-高风险  |

### 7.1 后台查询产品列表

**GET** `/content/products/admin?page=1&size=10&status=1&categoryId=1&riskLevel=3&keyword=基金`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| page | int | 否 | 1 | 页码 |
| size | int | 否 | 10 | 每页条数 |
| status | int | 否 | - | 状态：0=草稿，1=待审核，2=已发布，3=已下架，4=审核拒绝 |
| categoryId | int | 否 | - | 分类ID |
| riskLevel | int | 否 | - | 风险等级：1=R1低风险，2=R2中低，3=R3中，4=R4中高，5=R5高 |
| keyword | string | 否 | - | 搜索关键词 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "productId": 1,
        "productCode": "FUND001",
        "title": "招商中证白酒指数基金",
        "subtitle": "白酒行业龙头ETF",
        "summary": "跟踪中证白酒指数，投资白酒龙头企业...",
        "categoryId": 1,
        "tags": "基金,白酒,ETF",
        "riskLevelCode": "R3",
        "riskLevel": 3,
        "coverImageUrl": "https://oss.example.com/products/fund001.jpg",
        "status": 2,
        "createdAt": "2024-11-01T10:00:00",
        "publishedAt": "2024-11-05T14:00:00"
      }
    ],
    "total": 1,
    "size": 10,
    "current": 1,
    "pages": 1
  }
}
```

---

### 7.2 前台查询已上线产品列表

**GET** `/content/products/online?page=1&size=10&categoryId=1&keyword=基金`

**请求参数：**
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| page | int | 否 | 1 | 页码 |
| size | int | 否 | 10 | 每页条数 |
| categoryId | int | 否 | - | 分类ID |
| keyword | string | 否 | - | 搜索关键词 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "productId": 1,
        "productCode": "FUND001",
        "title": "招商中证白酒指数基金",
        "subtitle": "白酒行业龙头ETF",
        "summary": "跟踪中证白酒指数，投资白酒龙头企业...",
        "categoryId": 1,
        "riskLevelCode": "R3",
        "riskLevel": 3,
        "coverImageUrl": "https://oss.example.com/products/fund001.jpg",
        "publishedAt": "2024-11-05T14:00:00"
      }
    ],
    "total": 1,
    "size": 10,
    "current": 1,
    "pages": 1
  }
}
```

---

### 7.3 获取产品内容详情

**GET** `/content/products/{id}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "productId": 1,
    "productCode": "FUND001",
    "title": "招商中证白酒指数基金",
    "subtitle": "白酒行业龙头ETF",
    "summary": "跟踪中证白酒指数，投资白酒龙头企业...",
    "detailRichText": "<h2>产品介绍</h2><p>本基金跟踪中证白酒指数...</p>",
    "categoryId": 1,
    "tags": "基金,白酒,ETF",
    "riskLevelCode": "R3",
    "riskLevel": 3,
    "coverImageUrl": "https://oss.example.com/products/fund001.jpg",
    "docUrl": "https://oss.example.com/docs/fund001.pdf",
    "status": 2,
    "auditComment": "审核通过",
    "aiReviewResult": "风险等级合规，内容无违规",
    "aiRiskLevel": 3,
    "createdBy": "admin",
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-05T14:00:00",
    "publishedAt": "2024-11-05T14:00:00"
  }
}
```

---

### 7.4 保存或更新产品内容

**POST** `/content/products`

**请求体：**

```json
{
  "productCode": "FUND002",
  "title": "易方达消费行业基金",
  "subtitle": "消费行业精选",
  "summary": "投资于消费行业优质企业...",
  "detailRichText": "<h2>产品介绍</h2>...",
  "categoryId": 1,
  "tags": "基金,消费,行业",
  "riskLevelCode": "R3",
  "riskLevel": 3,
  "coverImageUrl": "https://oss.example.com/products/fund002.jpg"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.5 提交产品内容审核

**POST** `/content/products/{id}/submit`

> 将草稿或审核拒绝的产品提交审核，并触发AI预审

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.6 审核通过产品内容

**POST** `/content/products/{id}/approve`

**请求体：**

```json
{
  "auditor": "admin",
  "comment": "审核通过"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.7 审核拒绝产品内容

**POST** `/content/products/{id}/reject`

**请求体：**

```json
{
  "auditor": "admin",
  "comment": "内容不合规"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.8 下架产品内容

**POST** `/content/products/{id}/offline`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.9 查询内容分类列表

**GET** `/content/categories`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "categoryId": 1,
      "categoryName": "基金",
      "parentCategoryId": null,
      "contentType": 1,
      "sortOrder": 1,
      "status": 1,
      "createdAt": "2024-11-01T10:00:00"
    }
  ]
}
```

---

### 7.10 按类型查询分类

**GET** `/content/categories/type/{type}`

| 参数   | 类型  | 必填  | 说明                    |
| ---- | --- | --- | --------------------- |
| type | int | 是   | 内容类型：1=理财知识，2=资讯，3=视频 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "categoryId": 1,
      "categoryName": "基金入门",
      "parentCategoryId": null,
      "contentType": 1,
      "sortOrder": 1,
      "status": 1
    }
  ]
}
```

---

### 7.11 查询分类树

**GET** `/content/categories/tree`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "categoryId": 1,
      "categoryName": "投资理财",
      "parentCategoryId": null,
      "contentType": 1,
      "children": [
        {
          "categoryId": 2,
          "categoryName": "基金入门",
          "parentCategoryId": 1,
          "contentType": 1,
          "children": []
        }
      ]
    }
  ]
}
```

---

### 7.12 查询分类详情

**GET** `/content/categories/{id}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "categoryId": 1,
    "categoryName": "基金入门",
    "parentCategoryId": null,
    "contentType": 1,
    "sortOrder": 1,
    "status": 1,
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-01T10:00:00"
  }
}
```

---

### 7.13 新增分类

**POST** `/content/categories`

**请求体：**

```json
{
  "categoryName": "债券投资",
  "parentCategoryId": 1,
  "contentType": 1,
  "sortOrder": 2
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.14 更新分类

**PUT** `/content/categories/{id}`

**请求体：**

```json
{
  "categoryName": "债券投资入门",
  "sortOrder": 3
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.15 删除分类

**DELETE** `/content/categories/{id}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.16 切换分类状态

**POST** `/content/categories/{id}/toggle?status=1`

| 参数     | 类型  | 必填  | 说明           |
| ------ | --- | --- | ------------ |
| status | int | 是   | 状态：0=禁用，1=启用 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.17 AI内容合规检查

**POST** `/content/ai/check`

> 调用大模型对理财内容进行合规审查

**请求体：**

```json
{
  "title": "高收益理财产品推荐",
  "content": "本产品年化收益高达20%，保本保息，零风险...",
  "contentType": 1
}
```

| 字段          | 类型     | 必填  | 说明                    |
| ----------- | ------ | --- | --------------------- |
| title       | string | 是   | 内容标题                  |
| content     | string | 是   | 内容正文                  |
| contentType | int    | 是   | 内容类型：1=理财知识，2=资讯，3=视频 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "result": "该内容存在以下合规问题：\n1. '保本保息'承诺违反资管新规\n2. '零风险'表述不符合监管要求\n3. 年化收益20%缺乏风险提示\n\n建议修改：...",
    "riskLevel": 3
  }
}
```

> riskLevel: 1=低风险，2=中风险，3=高风险

---

### 7.18 获取媒体上传凭证

**POST** `/content/media/upload-token`

> 生成 OSS 预签名 URL，前端可直接上传媒体文件到阿里云 OSS

**请求体：**

```json
{
  "fileName": "product-cover.jpg",
  "bizType": 1,
  "mediaType": "image/jpeg",
  "sizeBytes": 1024000,
  "expireSeconds": 600
}
```

| 字段            | 类型     | 必填  | 默认值 | 说明                                |
| ------------- | ------ | --- | --- | --------------------------------- |
| fileName      | string | 是   | -   | 文件名                               |
| bizType       | int    | 是   | -   | 业务类型：1=产品封面，2=资讯图片，3=头像，4=视频，5=文档 |
| mediaType     | string | 否   | -   | 媒体MIME类型                          |
| sizeBytes     | long   | 否   | -   | 文件大小（字节）                          |
| expireSeconds | long   | 否   | 600 | 上传链接有效期（秒）                        |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "uploadUrl": "https://bucket.oss-cn-hangzhou.aliyuncs.com/images/1702544000000_product-cover.jpg?OSSAccessKeyId=xxx&Expires=xxx&Signature=xxx",
    "objectKey": "images/1702544000000_product-cover.jpg",
    "expireSeconds": 600,
    "bucket": "smartfinancial-media"
  }
}
```

> **上传流程（前端直传OSS）：**
> 
> 1. 前端调用此接口获取 `uploadUrl`
> 2. 前端使用 PUT 方法直接上传文件到 `uploadUrl`
> 3. 上传成功后调用 `/content/media/confirm` 确认
>
> ⚠️ 如遇到 CORS 403 错误，请使用下方的服务端代理上传接口。

---

### 7.19 服务端代理上传

**POST** `/content/media/upload`

> 前端上传文件到后端，后端再上传到OSS，避免跨域问题。一步完成上传和记录。

**请求格式：** `multipart/form-data`

| 字段     | 类型          | 必填 | 说明                                       |
| -------- | ------------- | ---- | ------------------------------------------ |
| file     | File          | 是   | 要上传的文件                               |
| bizType  | int           | 是   | 业务类型：1=产品封面，2=资讯图片，3=头像，4=视频，5=文档 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 123,
    "url": "https://steady-game.oss-cn-guangzhou.aliyuncs.com/content/images/1702544000000_avatar.jpg",
    "objectKey": "content/images/1702544000000_avatar.jpg"
  }
}
```

> **推荐使用此接口**，无需处理 CORS 和预签名 URL。

---

### 7.20 上传完成确认

**POST** `/content/media/confirm`

> 前端上传成功后回调，记录媒体资源并可选绑定到业务内容

**请求体：**

```json
{
  "bizType": 1,
  "objectKey": "images/1702544000000_product-cover.jpg",
  "contentType": "image/jpeg",
  "sizeBytes": 1024000,
  "duration": null,
  "operator": "admin",
  "bindContentType": "product",
  "bindContentId": 123,
  "asCover": true
}
```

| 字段              | 类型      | 必填  | 说明               |
| --------------- | ------- | --- | ---------------- |
| bizType         | int     | 是   | 业务类型（同上传凭证）      |
| objectKey       | string  | 是   | OSS 对象键（从上传凭证获取） |
| contentType     | string  | 否   | 媒体MIME类型         |
| sizeBytes       | long    | 否   | 文件大小             |
| duration        | int     | 否   | 视频时长（秒），仅视频类型需要  |
| operator        | string  | 否   | 操作人              |
| bindContentType | string  | 否   | 绑定内容类型：product   |
| bindContentId   | long    | 否   | 绑定内容ID           |
| asCover         | boolean | 否   | 是否作为封面（绑定产品时生效）  |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "assetId": 1001,
    "bizType": 1,
    "bucket": "smartfinancial-media",
    "objectKey": "images/1702544000000_product-cover.jpg",
    "url": "https://cdn.example.com/images/1702544000000_product-cover.jpg",
    "contentType": "image/jpeg",
    "sizeBytes": 1024000,
    "createdAt": "2024-12-14T15:00:00"
  }
}
```

---

### 7.20 分页查询媒体资源

**GET** `/content/media/page?page=1&size=10&bizType=1&keyword=产品`

| 参数      | 类型     | 必填  | 默认值 | 说明     |
| ------- | ------ | --- | --- | ------ |
| page    | int    | 否   | 1   | 页码     |
| size    | int    | 否   | 10  | 每页条数   |
| bizType | int    | 否   | -   | 业务类型筛选 |
| keyword | string | 否   | -   | 关键词搜索  |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "assetId": 1001,
        "bizType": 1,
        "url": "https://cdn.example.com/images/xxx.jpg",
        "contentType": "image/jpeg",
        "sizeBytes": 1024000,
        "createdAt": "2024-12-14T15:00:00"
      }
    ],
    "total": 100,
    "size": 10,
    "current": 1,
    "pages": 10
  }
}
```

---

### 7.21 按类型查询媒体

**GET** `/content/media/type/{bizType}`

| 参数      | 类型  | 必填  | 说明   |
| ------- | --- | --- | ---- |
| bizType | int | 是   | 业务类型 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "assetId": 1001,
      "bizType": 1,
      "url": "https://cdn.example.com/images/xxx.jpg",
      "contentType": "image/jpeg",
      "sizeBytes": 1024000
    }
  ]
}
```

---

### 7.22 搜索媒体资源

**GET** `/content/media/search?keyword=产品&bizType=1&limit=20`

| 参数      | 类型     | 必填  | 默认值 | 说明     |
| ------- | ------ | --- | --- | ------ |
| keyword | string | 否   | -   | 关键词    |
| bizType | int    | 否   | -   | 业务类型   |
| limit   | int    | 否   | 20  | 返回数量上限 |

---

### 7.23 删除媒体资源

**DELETE** `/content/media/{id}`

| 参数  | 类型   | 必填  | 说明     |
| --- | ---- | --- | ------ |
| id  | long | 是   | 媒体资源ID |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.24 批量删除媒体资源

**DELETE** `/content/media/batch`

**请求体：**

```json
[1001, 1002, 1003]
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 7.25 媒体类型统计

**GET** `/content/media/stats/type`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    { "bizType": 1, "typeName": "产品封面", "count": 120 },
    { "bizType": 2, "typeName": "资讯图片", "count": 85 },
    { "bizType": 3, "typeName": "头像", "count": 200 },
    { "bizType": 4, "typeName": "视频", "count": 15 },
    { "bizType": 5, "typeName": "文档", "count": 30 }
  ]
}
```

---

## 8. 通知服务接口

> 网关路由：`/notification/**` → notification-service（StripPrefix=1，去掉 `/notification` 前缀）
> 
> ⚠️ **注意**：需要在 Nacos 中更新网关配置后生效

> **枚举定义：**
> 
> | 枚举                   | 值         | 说明   |
> | -------------------- | --------- | ---- |
> | **notificationType** | TRADE     | 交易通知 |
> |                      | RISK      | 风险提醒 |
> |                      | SYSTEM    | 系统通知 |
> |                      | PROMOTION | 活动推广 |
> | **channel**          | IN_APP    | 站内消息 |
> |                      | SMS       | 短信   |
> |                      | EMAIL     | 邮件   |
> |                      | PUSH      | 推送   |
> | **status**           | PENDING   | 待发送  |
> |                      | SENT      | 已发送  |
> |                      | FAILED    | 发送失败 |

### 8.1 查询通知列表

**GET** `/notification/notifications?userId=1001&page=1&size=20`

| 参数     | 类型   | 必填  | 默认值 | 说明   |
| ------ | ---- | --- | --- | ---- |
| userId | long | 是   | -   | 用户ID |
| page   | int  | 否   | 1   | 页码   |
| size   | int  | 否   | 20  | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "userId": 1001,
      "notificationType": "TRADE",
      "channel": "IN_APP",
      "title": "订单成交通知",
      "content": "您的买入订单已成交，成交金额1050.00元",
      "status": "SENT",
      "bizId": "10001",
      "bizType": "ORDER",
      "isRead": false,
      "createTime": "2024-11-28T10:35:00"
    }
  ]
}
```

---

### 8.2 查询未读数量

**GET** `/notification/notifications/unread-count?userId=1001`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": 5
}
```

---

### 8.3 标记已读

**POST** `/notification/notifications/{id}/read`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 8.4 全部已读

**POST** `/notification/notifications/read-all?userId=1001`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 8.5 获取通知摘要

**GET** `/notification/notifications/summary?userId=1001`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "unreadCount": 5,
    "recentNotifications": [
      {
        "id": 1,
        "userId": 1001,
        "notificationType": "TRADE",
        "title": "订单成交通知",
        "content": "您的买入订单已成交",
        "isRead": false,
        "createTime": "2024-11-28T10:35:00"
      }
    ]
  }
}
```

---

### 8.6 智能提醒规则管理

> **枚举定义：**
> 
> | 枚举            | 值              | 说明     |
> | ------------- | -------------- | ------ |
> | **ruleType**  | PRICE_CHANGE   | 价格变动提醒 |
> |               | RETURN_RATE    | 收益率提醒  |
> |               | POSITION_ALERT | 仓位提醒   |
> | **direction** | UP             | 上涨触发   |
> |               | DOWN           | 下跌触发   |
> |               | BOTH           | 双向触发   |

#### 8.6.1 获取提醒规则列表

**GET** `/notification/alerts/rules`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "userId": 1001,
      "ruleType": "PRICE_CHANGE",
      "productId": 2001,
      "threshold": 5.0,
      "direction": "BOTH",
      "enabled": true,
      "lastTriggeredAt": null,
      "createTime": "2025-12-10T10:00:00"
    }
  ]
}
```

#### 8.6.2 创建提醒规则

**POST** `/notification/alerts/rules`

**请求体：**

```json
{
  "ruleType": "PRICE_CHANGE",
  "productId": 2001,
  "threshold": 5.0,
  "direction": "UP"
}
```

| 字段        | 类型      | 必填  | 说明              |
| --------- | ------- | --- | --------------- |
| ruleType  | string  | 是   | 规则类型            |
| productId | long    | 否   | 关联产品ID（空表示全局规则） |
| threshold | decimal | 是   | 阈值（百分比）         |
| direction | string  | 否   | 触发方向，默认 BOTH    |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "id": 2,
    "userId": 1001,
    "ruleType": "PRICE_CHANGE",
    "productId": 2001,
    "threshold": 5.0,
    "direction": "UP",
    "enabled": true,
    "createTime": "2025-12-10T15:00:00"
  }
}
```

#### 8.6.3 更新提醒规则

**PUT** `/notification/alerts/rules/{ruleId}`

**请求体：**

```json
{
  "threshold": 10.0,
  "enabled": false
}
```

#### 8.6.4 删除提醒规则

**DELETE** `/notification/alerts/rules/{ruleId}`

---

### 8.7 提醒历史记录

#### 8.7.1 获取提醒历史

**GET** `/notification/alerts/history?page=1&size=20`

| 参数   | 类型  | 必填  | 默认值 | 说明   |
| ---- | --- | --- | --- | ---- |
| page | int | 否   | 1   | 页码   |
| size | int | 否   | 20  | 每页条数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "id": 1,
        "userId": 1001,
        "ruleId": 1,
        "ruleType": "PRICE_CHANGE",
        "productId": 2001,
        "productName": "招商中证白酒指数",
        "message": "招商中证白酒指数 涨幅达到 5.2%，触发您设置的 5% 提醒",
        "triggeredAt": "2025-12-10T14:30:00",
        "isRead": false
      }
    ],
    "total": 10
  }
}
```

#### 8.7.2 标记提醒已读

**POST** `/notification/alerts/history/{alertId}/read`

---

## 9. 管理后台接口

> ⚠️ **权限要求**：以下接口需要管理员权限（`ADMIN`），无权限访问返回 403 Forbidden
> 
> 通过网关 `/admin/**` 路由访问

### 9.1 用户管理

#### 9.1.1 分页查询用户列表

**GET** `/admin/users?page=1&size=20&username=张三&status=1`

| 参数       | 类型     | 必填  | 说明             |
| -------- | ------ | --- | -------------- |
| page     | int    | 否   | 页码，默认1         |
| size     | int    | 否   | 每页条数，默认20      |
| username | string | 否   | 用户名（模糊匹配）      |
| phone    | string | 否   | 手机号（模糊匹配）      |
| status   | int    | 否   | 用户状态：1=启用，0=禁用 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "userId": 1001,
        "username": "zhangsan",
        "email": "zhangsan@example.com",
        "phone": "13800138000",
        "realName": "张三",
        "gender": 1,
        "avatarUrl": "https://oss.example.com/avatars/1001.jpg",
        "status": 1,
        "createdAt": "2024-11-01T10:00:00",
        "lastLoginTime": "2024-11-30T03:45:00",
        "roles": ["USER"]
      }
    ],
    "total": 100,
    "size": 20,
    "current": 1,
    "pages": 5
  }
}
```

---

#### 9.1.2 获取用户详情

**GET** `/admin/users/{userId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "userId": 1001,
    "username": "zhangsan",
    "email": "zhangsan@example.com",
    "emailVerified": false,
    "phone": "13800138000",
    "phoneVerified": true,
    "realName": "张三",
    "gender": 1,
    "birthDate": "1990-01-01",
    "avatarUrl": "https://oss.example.com/avatars/1001.jpg",
    "status": 1,
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-28T10:00:00",
    "roles": ["USER"],
    "riskLevel": 3
  }
}
```

> **v1.16.0 新增字段**：`emailVerified`（邮箱验证状态）、`phoneVerified`（手机验证状态）

---

#### 9.1.3 设置用户状态

**POST** `/admin/users/{userId}/status?status=1`

| 参数     | 类型  | 必填  | 说明        |
| ------ | --- | --- | --------- |
| status | int | 是   | 1=启用，0=禁用 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.1.4 分配角色给用户

**POST** `/admin/users/assign-roles`

**请求体：**

```json
{
  "userId": 1001,
  "roleIds": [1, 2, 3]
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.1.5 重置用户密码

**POST** `/admin/users/{userId}/reset-password?newPassword=NewPass123!`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 9.2 角色管理

#### 9.2.1 分页查询角色

**GET** `/admin/roles?page=1&size=20&keyword=管理`

| 参数      | 类型     | 必填  | 说明        |
| ------- | ------ | --- | --------- |
| page    | int    | 否   | 页码，默认1    |
| size    | int    | 否   | 每页条数，默认20 |
| keyword | string | 否   | 搜索关键词     |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "roleId": 1,
        "roleCode": "ADMIN",
        "roleName": "系统管理员",
        "description": "拥有系统所有权限",
        "status": 1,
        "createdAt": "2024-11-01T10:00:00"
      }
    ],
    "total": 5,
    "size": 20,
    "current": 1,
    "pages": 1
  }
}
```

---

#### 9.2.2 获取所有启用的角色

**GET** `/admin/roles/enabled`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "roleId": 1,
      "roleCode": "ADMIN",
      "roleName": "系统管理员",
      "description": "拥有系统所有权限",
      "status": 1
    },
    {
      "roleId": 5,
      "roleCode": "USER",
      "roleName": "普通用户",
      "description": "普通注册用户",
      "status": 1
    }
  ]
}
```

---

#### 9.2.3 获取角色详情

**GET** `/admin/roles/{roleId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "roleId": 1,
    "roleCode": "ADMIN",
    "roleName": "系统管理员",
    "description": "拥有系统所有权限",
    "status": 1,
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-01T10:00:00"
  }
}
```

---

#### 9.2.4 创建角色

**POST** `/admin/roles`

**请求体：**

```json
{
  "roleName": "运营管理员",
  "roleCode": "OPERATION",
  "description": "负责日常运营管理",
  "status": 1
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.2.5 更新角色

**PUT** `/admin/roles/{roleId}`

**请求体：**

```json
{
  "roleName": "运营管理员（更新）",
  "description": "负责日常运营管理和数据分析"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.2.6 删除角色

**DELETE** `/admin/roles/{roleId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.2.7 启用/禁用角色

**POST** `/admin/roles/{roleId}/status?status=1`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.2.8 获取角色的权限列表

**GET** `/admin/roles/{roleId}/permissions`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "permissionId": 1,
      "permissionCode": "user:read",
      "permissionName": "用户查看"
    },
    {
      "permissionId": 2,
      "permissionCode": "user:write",
      "permissionName": "用户编辑"
    }
  ]
}
```

---

#### 9.2.9 分配权限给角色

**POST** `/admin/roles/{roleId}/permissions`

**请求体：**

```json
[1, 2, 3, 4, 5]
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 9.3 权限管理

#### 9.3.1 获取所有权限

**GET** `/admin/permissions`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "permissionId": 1,
      "permissionCode": "user:read",
      "permissionName": "用户查看",
      "description": "查看用户列表和详情的权限"
    },
    {
      "permissionId": 2,
      "permissionCode": "user:write",
      "permissionName": "用户编辑",
      "description": "编辑用户信息的权限"
    }
  ]
}
```

---

#### 9.3.2 获取权限详情

**GET** `/admin/permissions/{permissionId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "permissionId": 1,
    "permissionCode": "user:read",
    "permissionName": "用户查看",
    "description": "查看用户列表和详情的权限",
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-01T10:00:00"
  }
}
```

---

#### 9.3.3 创建权限

**POST** `/admin/permissions`

**请求体：**

```json
{
  "permissionName": "用户查看",
  "permissionCode": "user:read",
  "description": "查看用户列表和详情的权限"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.3.4 更新权限

**PUT** `/admin/permissions/{permissionId}`

**请求体：**

```json
{
  "permissionName": "用户查看（更新）",
  "description": "查看用户列表、详情和日志的权限"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

#### 9.3.5 删除权限

**DELETE** `/admin/permissions/{permissionId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 9.4 系统监控

> ⚠️ **权限要求**：需要 `ADMIN` 权限，否则返回 403 Forbidden

#### 9.4.1 获取系统状态

**GET** `/admin/system/status`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "cpu": 35.5,
    "memory": 68.2,
    "disk": 55.0,
    "network": 45
  }
}
```

---

#### 9.4.2 获取CPU趋势

**GET** `/admin/system/cpu-trend?range=1h`

| 参数    | 类型     | 必填  | 默认值 | 说明             |
| ----- | ------ | --- | --- | -------------- |
| range | string | 否   | 1h  | 时间范围：1h/6h/24h |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {"time": "10:00", "value": 35.2},
    {"time": "10:05", "value": 38.5},
    {"time": "10:10", "value": 32.1}
  ]
}
```

---

#### 9.4.3 获取服务状态

**GET** `/admin/system/services`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "name": "user-service",
      "status": "running",
      "uptime": "5d 12h 30m",
      "memory": "512MB",
      "cpu": "2.5%"
    },
    {
      "name": "trade-service",
      "status": "running",
      "uptime": "5d 12h 30m",
      "memory": "768MB",
      "cpu": "3.2%"
    }
  ]
}
```

---

#### 9.4.4 获取请求统计

**GET** `/admin/system/request-stats?range=24h`

| 参数    | 类型     | 必填  | 默认值 | 说明             |
| ----- | ------ | --- | --- | -------------- |
| range | string | 否   | 24h | 时间范围：1h/6h/24h |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "labels": ["00:00", "01:00", "02:00"],
    "values": [1500, 1200, 800],
    "totalRequests": 50000,
    "avgResponseTime": 85.5
  }
}
```

---

## 10. 统计分析接口

> ⚠️ **权限要求**：需要 `ADMIN` 或 `RISK_MANAGER` 权限，无权限访问返回 403 Forbidden
> 
> 通过网关 `/analysis/**` 路由访问

### 10.1 平台总览

#### 10.1.1 获取平台总览统计

**GET** `/analysis/admin/overview`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "totalUsers": 10000,
    "activeUsers": 3500,
    "totalAssets": 50000000.00,
    "todayTrades": 1200,
    "todayTradeAmount": 5000000.00
  }
}
```

---

#### 10.1.2 获取每日活跃用户统计

**GET** `/analysis/admin/user-active?days=7`

| 参数   | 类型  | 必填  | 默认值 | 说明   |
| ---- | --- | --- | --- | ---- |
| days | int | 否   | 7   | 统计天数 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {"date": "2024-11-22", "activeUsers": 3200},
    {"date": "2024-11-23", "activeUsers": 3400},
    {"date": "2024-11-24", "activeUsers": 3100},
    {"date": "2024-11-25", "activeUsers": 3500}
  ]
}
```

---

#### 10.1.3 获取顾问业绩统计

**GET** `/analysis/admin/advisor-performance?statMonth=2024-11-01`

| 参数        | 类型   | 必填  | 说明                 |
| --------- | ---- | --- | ------------------ |
| statMonth | date | 否   | 统计月份，格式 yyyy-MM-dd |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "advisorId": 1,
      "advisorName": "李顾问",
      "clientCount": 150,
      "totalAum": 15000000.00,
      "tradeCount": 320,
      "commission": 45000.00
    }
  ]
}
```

---

#### 10.1.4 最近活动记录

**GET** `/analysis/activities/recent?limit=10`

| 参数    | 类型  | 必填  | 默认值 | 说明        |
| ----- | --- | --- | --- | --------- |
| limit | int | 否   | 10  | 返回条数，最大20 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "id": 1,
      "title": "新用户注册",
      "description": "活动详情 1",
      "time": "2024-11-30 10:30:00",
      "type": "success",
      "user": "系统"
    },
    {
      "id": 2,
      "title": "订单创建",
      "description": "活动详情 2",
      "time": "2024-11-30 10:20:00",
      "type": "info",
      "user": "系统"
    }
  ]
}
```

---

#### 10.1.5 顾问 KPI 看板

> ⚠️ **权限要求**：需要 `ADMIN` 或 `ADVISOR` 权限

**GET** `/analysis/advisor/dashboard`

> 获取当前登录顾问的 KPI 看板数据，聚合业绩、客户、佣金、服务质量、排名等信息

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "advisorId": 1001,
    "advisorName": "李顾问",
    "statDate": "2024-12-19",
    "totalClientCount": 150,
    "newClientCount": 12,
    "totalAssetsManaged": 15000000.00,
    "monthlyNewInvestment": 2500000.00,
    "monthlyDealAmount": 3200000.00,
    "monthlyCommission": 48000.00,
    "monthlyServiceCount": 45,
    "averageRating": 4.8,
    "satisfactionRate": 96.0,
    "performanceRank": 3,
    "totalAdvisorCount": 25,
    "performanceTrends": [
      { "month": "2024-07", "dealAmount": 2800000.00, "newClientCount": 8, "commission": 42000.00 },
      { "month": "2024-08", "dealAmount": 2900000.00, "newClientCount": 10, "commission": 43500.00 },
      { "month": "2024-09", "dealAmount": 3000000.00, "newClientCount": 9, "commission": 45000.00 },
      { "month": "2024-10", "dealAmount": 3100000.00, "newClientCount": 11, "commission": 46500.00 },
      { "month": "2024-11", "dealAmount": 3050000.00, "newClientCount": 10, "commission": 45750.00 },
      { "month": "2024-12", "dealAmount": 3200000.00, "newClientCount": 12, "commission": 48000.00 }
    ]
  }
}
```

| 字段                   | 类型    | 说明                       |
| ---------------------- | ------- | -------------------------- |
| advisorId              | long    | 顾问ID                     |
| advisorName            | string  | 顾问姓名                   |
| statDate               | date    | 统计日期                   |
| totalClientCount       | int     | 管理客户总数               |
| newClientCount         | int     | 本月新增客户数             |
| totalAssetsManaged     | decimal | 管理总资产（元）           |
| monthlyNewInvestment   | decimal | 本月新增投资额（元）       |
| monthlyDealAmount      | decimal | 本月成交金额（元）         |
| monthlyCommission      | decimal | 本月佣金收入（元）         |
| monthlyServiceCount    | int     | 本月服务次数               |
| averageRating          | decimal | 平均客户评分（5分制）      |
| satisfactionRate       | decimal | 客户满意度（百分比）       |
| performanceRank        | int     | 业绩排名                   |
| totalAdvisorCount      | int     | 顾问总数                   |
| performanceTrends      | array   | 最近6个月业绩趋势          |
| performanceTrends.month | string  | 月份                       |
| performanceTrends.dealAmount | decimal | 成交金额               |
| performanceTrends.newClientCount | int | 新增客户数              |
| performanceTrends.commission | decimal | 佣金收入                |

---

**GET** `/analysis/advisor/dashboard/{advisorId}`

> 管理员查询指定顾问的 KPI 看板数据

> ⚠️ **权限要求**：仅 `ADMIN` 权限

| 参数      | 类型 | 必填 | 说明    |
| --------- | ---- | ---- | ------- |
| advisorId | long | 是   | 顾问ID  |

**响应格式**：同上

---

### 10.2 运营报表

#### 10.2.1 用户活跃度统计

**GET** `/analysis/operation/user-activity?startDate=2024-11-01&endDate=2024-11-30`

| 参数        | 类型   | 必填  | 说明   |
| --------- | ---- | --- | ---- |
| startDate | date | 是   | 开始日期 |
| endDate   | date | 是   | 结束日期 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "dau": [{"date": "2024-11-01", "count": 3200}],
    "wau": 8500,
    "mau": 12000,
    "avgSessionDuration": 15.5
  }
}
```

---

#### 10.2.2 用户留存率

**GET** `/analysis/operation/user-retention?days=7`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "day1Retention": 0.65,
    "day3Retention": 0.45,
    "day7Retention": 0.32
  }
}
```

---

#### 10.2.3 交易转化率

**GET** `/analysis/operation/trade-conversion?startDate=2024-11-01&endDate=2024-11-30`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "viewToCartRate": 0.25,
    "cartToOrderRate": 0.60,
    "orderToPayRate": 0.85,
    "overallConversionRate": 0.13
  }
}
```

---

#### 10.2.4 产品热度排行

**GET** `/analysis/operation/product-hot?topN=10`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "productId": 2001,
      "productName": "招商中证白酒指数基金",
      "viewCount": 15000,
      "tradeCount": 1200,
      "tradeAmount": 5000000.00,
      "hotScore": 95.5
    }
  ]
}
```

---

#### 10.2.5 产品热度详情

**GET** `/analysis/operation/product-hot/{productId}`

---

#### 10.2.6 按类型查询产品热度

**GET** `/analysis/operation/product-hot/type/{productType}?topN=10`

---

### 10.3 合规报表

#### 10.3.1 审计日志统计

**GET** `/analysis/compliance/audit-log?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.3.2 按操作类型统计审计日志

**GET** `/analysis/compliance/audit-log/type/{operationType}?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.3.3 获取合规报表

**GET** `/analysis/compliance/report?reportDate=2024-11-30`

---

#### 10.3.4 合规报表列表

**GET** `/analysis/compliance/reports?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.3.5 大额交易报告

**GET** `/analysis/compliance/large-trade?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.3.6 可疑交易报告

**GET** `/analysis/compliance/suspicious-trade?startDate=2024-11-01&endDate=2024-11-30`

---

### 10.4 报表导出

#### 10.4.1 导出顾问业绩报表（Excel）

**GET** `/analysis/export/advisor-performance?statMonth=2024-11-01`

> 返回 Excel 文件下载

---

#### 10.4.2 导出佣金统计报表（Excel）

**GET** `/analysis/export/commission?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.4.3 导出运营报表（Excel）

**GET** `/analysis/export/operation?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.4.4 导出合规报表（Excel）

**GET** `/analysis/export/compliance?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.4.5 导出审计日志报表（Excel）

**GET** `/analysis/export/audit-log?startDate=2024-11-01&endDate=2024-11-30`

---

#### 10.4.6 导出合规报表（PDF）

**GET** `/analysis/export/compliance/pdf?reportDate=2024-11-30`

---

## 11. 风控服务接口（管理端）

> ⚠️ **权限要求**：需要 `ADMIN` 权限
> 
> 风控服务接口通过 user-service 代理访问，统一走网关 `/admin/risk/**` 路由

### 11.1 分页查询风控规则

**GET** `/admin/risk/rules?page=1&size=20&ruleType=LARGE_AMOUNT&enabled=1`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "configId": 1,
        "ruleCode": "LARGE_AMOUNT_001",
        "ruleName": "大额交易限制",
        "ruleType": "LARGE_AMOUNT",
        "description": "单笔交易金额超过50万需要额外验证",
        "enabled": 1,
        "priority": 100,
        "params": {"threshold": 500000},
        "createdAt": "2024-11-01T10:00:00"
      }
    ],
    "total": 10,
    "size": 20,
    "current": 1,
    "pages": 1
  }
}
```

---

### 11.2 获取启用的规则列表

**GET** `/admin/risk/rules/enabled`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": [
    {
      "configId": 1,
      "ruleCode": "LARGE_AMOUNT_001",
      "ruleName": "大额交易限制",
      "ruleType": "LARGE_AMOUNT",
      "enabled": 1,
      "priority": 100
    }
  ]
}
```

---

### 11.3 获取规则详情

**GET** `/admin/risk/rules/{configId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "configId": 1,
    "ruleCode": "LARGE_AMOUNT_001",
    "ruleName": "大额交易限制",
    "ruleType": "LARGE_AMOUNT",
    "description": "单笔交易金额超过50万需要额外验证",
    "enabled": 1,
    "priority": 100,
    "params": {"threshold": 500000},
    "createdAt": "2024-11-01T10:00:00",
    "updatedAt": "2024-11-01T10:00:00"
  }
}
```

---

### 11.4 创建风控规则

**POST** `/admin/risk/rules`

**请求体：**

```json
{
  "ruleCode": "CUSTOM_RULE_001",
  "ruleName": "自定义规则",
  "ruleType": "CUSTOM",
  "description": "自定义风控规则描述",
  "enabled": 1,
  "priority": 100,
  "params": {
    "threshold": 50000
  }
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.5 更新风控规则

**PUT** `/admin/risk/rules/{configId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.6 删除风控规则

**DELETE** `/admin/risk/rules/{configId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.7 启用/禁用规则

**POST** `http://localhost:8701/admin/risk/rules/{configId}/enabled?enabled=1`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.8 获取全局风控配置

**GET** `http://localhost:8701/admin/risk/rules/global-config`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "globalEnabled": true,
    "singleTradeLimit": 500000,
    "dailyTradeLimit": 2000000,
    "largeAmountThreshold": 50000,
    "freqMaxRequests": 10,
    "freqWindowSeconds": 300
  }
}
```

---

### 11.9 批量更新阈值配置

**POST** `http://localhost:8701/admin/risk/rules/batch-threshold`

**请求体：**

```json
{
  "singleTradeLimit": 500000,
  "dailyTradeLimit": 1000000,
  "largeAmountThreshold": 100000,
  "freqMaxRequests": 10
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.10 获取规则引擎运行状态

**GET** `http://localhost:8701/admin/risk/rules/runtime-status`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "engineStatus": "RUNNING",
    "loadedRulesCount": 15,
    "lastRefreshTime": "2024-11-28T10:30:00",
    "ruleExecutionStats": {
      "totalExecutions": 10000,
      "blockedCount": 150,
      "passedCount": 9850
    }
  }
}
```

---

### 11.11 更新全局风控开关

**POST** `http://localhost:8701/admin/risk/rules/global-enabled?enabled=true`

| 参数      | 类型      | 必填  | 说明               |
| ------- | ------- | --- | ---------------- |
| enabled | boolean | 是   | true=启用，false=禁用 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.12 刷新规则引擎

**POST** `http://localhost:8701/admin/risk/rules/refresh`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.13 分页查询风险事件

**GET** `/admin/risk/events?page=1&size=20&level=3&status=1`

**请求参数：**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| size | int | 否 | 每页条数，默认20 |
| level | int | 否 | 严重等级：1=低，2=中，3=高 |
| status | int | 否 | 状态：1=待处理，2=处理中，3=已处理 |

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "records": [
      {
        "eventId": 1,
        "userId": 1001,
        "ruleId": 5,
        "eventType": "大额交易",
        "eventDetail": "单笔交易金额超过50万",
        "severityLevel": 3,
        "status": 1,
        "triggeredAt": "2024-11-30T10:30:00",
        "handleResult": null,
        "handledAt": null
      }
    ],
    "total": 10,
    "size": 20,
    "current": 1,
    "pages": 1
  }
}
```

---

### 11.14 获取风险事件详情

**GET** `/admin/risk/events/{eventId}`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": {
    "eventId": 1,
    "userId": 1001,
    "ruleId": 5,
    "eventType": "大额交易",
    "eventDetail": "单笔交易金额超过50万",
    "severityLevel": 3,
    "status": 1,
    "triggeredAt": "2024-11-30T10:30:00",
    "handleResult": null,
    "handledAt": null
  }
}
```

---

### 11.15 处理风险事件

**POST** `/admin/risk/events/{eventId}/process`

**请求体：**

```json
{
  "action": "warning",
  "note": "已发送风险提醒"
}
```

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.16 冻结用户账号

**POST** `/admin/risk/users/{userId}/freeze`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

### 11.17 解冻用户账号

**POST** `/admin/risk/users/{userId}/unfreeze`

**响应示例：**

```json
{
  "code": 200,
  "msg": null,
  "data": null
}
```

---

## 12. 错误码说明

### 12.1 HTTP 状态码

| 错误码 | 说明          |
| --- | ----------- |
| 200 | 成功          |
| 400 | 参数错误        |
| 401 | 未登录/Token过期 |
| 403 | 无权限         |
| 404 | 资源不存在       |
| 429 | 请求过于频繁      |
| 500 | 服务器内部错误     |

### 12.2 业务错误码

| 错误码  | 说明             |
| ---- | -------------- |
| 1001 | 用户不存在          |
| 1002 | 密码错误           |
| 1003 | 用户已禁用          |
| 1004 | 验证码错误          |
| 2001 | 订单不存在          |
| 2002 | 订单状态不允许操作      |
| 2003 | 交易密码错误         |
| 2004 | 大额交易验证码错误      |
| 3001 | 风控拦截           |
| 3002 | 产品风险等级超出用户承受能力 |
| 4001 | AI服务暂时不可用      |

### 12.3 统一错误消息 (ErrorMessageEnum)

> v1.4.0 新增：所有控制器错误消息已统一使用枚举

| 枚举常量                        | 错误消息                    |
| --------------------------- | ----------------------- |
| USER_ID_REQUIRED            | userId 不能为空             |
| USER_ID_NOT_PROVIDED        | 用户未登录或网关未传递用户ID         |
| USER_NOT_LOGGED_IN          | 用户未登录                   |
| USER_NOT_FOUND              | 用户不存在                   |
| ROLE_NOT_FOUND              | 角色不存在                   |
| PERMISSION_NOT_FOUND        | 权限不存在                   |
| ASSET_NOT_FOUND             | 资产不存在                   |
| ASSET_MERGE_MIN             | 至少选择两个资产进行合并            |
| ASSET_TYPE_MISMATCH         | 只能合并同类型资产               |
| RISK_EVENT_NOT_FOUND        | 事件不存在                   |
| RISK_RULE_NOT_FOUND         | 规则配置不存在                 |
| RISK_ASSESSMENT_NOT_FOUND   | 尚未完成风险评估                |
| ORDER_CANCEL_FAILED         | 撤销失败                    |
| ORDER_REDEEM_FAILED         | 赎回失败：订单不存在或状态不允许赎回      |
| ORDER_RETRY_FAILED          | 重试失败：订单不存在或状态不允许重试      |
| TRADE_VERIFY_FAILED         | 下单失败：交易密码或大额交易验证码错误或已失效 |
| TRADE_PASSWORD_MISMATCH     | 两次输入的交易密码不一致            |
| NEW_TRADE_PASSWORD_MISMATCH | 两次输入的新交易密码不一致           |
| QUESTION_REQUIRED           | 问题内容不能为空                |
| MODEL_NOT_FOUND             | 未找到当前模型                 |
| PLAN_GENERATE_FAILED        | 方案不存在或生成失败              |
| PLAN_ADJUST_FAILED          | 方案不存在或调整失败              |
| AI_SERVICE_UNAVAILABLE      | AI 服务未配置或未启用            |
| AI_SERVICE_BUSY             | AI 服务暂时不可用，请稍后重试        |
| SERVICE_UNAVAILABLE         | 服务不可用                   |

---

## 附录

### A. 订单状态枚举

| 值   | 说明  |
| --- | --- |
| 1   | 待支付 |
| 2   | 已支付 |
| 3   | 已完成 |
| 4   | 已取消 |
| 5   | 失败  |

### B. 资产类型枚举

| 值   | 说明  |
| --- | --- |
| 1   | 现金  |
| 2   | 基金  |
| 3   | 股票  |
| 4   | 债券  |
| 5   | 保险  |
| 6   | 其他  |

### C. 风险等级枚举

| 值   | 说明  |
| --- | --- |
| 1   | 保守型 |
| 2   | 稳健型 |
| 3   | 平衡型 |
| 4   | 积极型 |
| 5   | 激进型 |

### D. 通知类型枚举 (NotificationTypeEnum)

| 值            | 说明   |
| ------------ | ---- |
| SYSTEM       | 系统通知 |
| TRADE        | 交易通知 |
| RISK_ALERT   | 风险警报 |
| RISK_REMIND  | 风险提醒 |
| PROMOTION    | 营销活动 |
| ANNOUNCEMENT | 平台公告 |

### E. 通知渠道枚举 (NotificationChannelEnum)

| 值        | 说明    |
| -------- | ----- |
| SMS      | 短信    |
| EMAIL    | 邮件    |
| APP_PUSH | APP推送 |
| IN_APP   | 站内信   |
| WECHAT   | 微信    |

### F. 告警等级枚举 (AlertLevelEnum)

| 值        | 说明  |
| -------- | --- |
| LOW      | 低级  |
| MEDIUM   | 中级  |
| HIGH     | 高级  |
| CRITICAL | 严重  |

### G. 启用状态枚举 (EnableStatusEnum)

| 值   | 说明  |
| --- | --- |
| 0   | 禁用  |
| 1   | 启用  |

### H. 内容状态枚举 (ContentStatusEnum)

| 值   | 说明  |
| --- | --- |
| 0   | 草稿  |
| 1   | 已发布 |
| 2   | 已下架 |

---

## 附录二：通知服务真实化配置

### 1. 短信验证码

**触发场景**：
| 接口 | 场景 |
|------|------|
| POST `/user/security/bind-phone/code?phone=xxx` | 绑定手机发送验证码 |
| POST `/user/security/large-trade/code` | 大额交易发送验证码 |

### 2. 邮件验证码

**触发场景**：
| 接口 | 场景 |
|------|------|
| POST `/user/security/bind-email/code?email=xxx` | 绑定邮箱发送验证码 |

### 3. 说明

- 开发/测试环境下验证码不会真实发送，可在服务日志中查看
- 生产环境会真实发送短信和邮件

---

## 附录三：金融行情数据接口

> 行情数据通过 trade-service 提供，后台定时任务每 5 分钟从 Python 数据服务同步至数据库

### 1. 市场概览

**GET** `/trade/market/overview`

> 返回主要指数 + 热门股票综合数据（从数据库读取已同步的数据）

**响应示例**：

```json
{
  "code": 200,
  "data": {
    "indices": [...],
    "hotStocks": [...],
    "crypto": [...],
    "forex": [...]
  }
}
```

### 2. 股票K线数据

**GET** `/trade/market/stock/kline`

| 参数     | 类型     | 必填  | 默认值 | 说明               |
| ------ | ------ | --- | --- | ---------------- |
| region | string | 否   | cn  | 地区：cn/us/hk      |
| code   | string | 是   | -   | 股票代码             |
| kType  | int    | 否   | 8   | K线类型：1=1分钟, 8=日线 |
| limit  | int    | 否   | 100 | 返回数量             |

**示例**：`/trade/market/stock/kline?region=cn&code=600519&kType=8&limit=100`

### 3. 股票实时报价

**GET** `/trade/market/stock/quote`

| 参数     | 类型     | 必填  | 默认值 | 说明          |
| ------ | ------ | --- | --- | ----------- |
| region | string | 否   | cn  | 地区：cn/us/hk |
| code   | string | 是   | -   | 股票代码        |

**示例**：

- A股：`/trade/market/stock/quote?region=cn&code=600519`
- 美股：`/trade/market/stock/quote?region=us&code=AAPL`
- 港股：`/trade/market/stock/quote?region=hk&code=00700`

### 4. 股票盘口深度

**GET** `/trade/market/stock/depth`

| 参数     | 类型     | 必填  | 默认值 | 说明          |
| ------ | ------ | --- | --- | ----------- |
| region | string | 否   | cn  | 地区：cn/us/hk |
| code   | string | 是   | -   | 股票代码        |

### 5. 加密货币接口

| 接口                                                          | 说明     |
| ----------------------------------------------------------- | ------ |
| GET `/trade/market/crypto/kline?code=BTC&kType=8&limit=100` | 加密货币K线 |
| GET `/trade/market/crypto/quote?code=BTC`                   | 加密货币报价 |

### 6. 外汇接口

**GET** `/trade/market/forex/quote?code=USDCNY`

### 7. 自选股管理

| 接口                                      | 方法     | 说明              |
| --------------------------------------- | ------ | --------------- |
| `/trade/market/watchlist`               | GET    | 获取自选列表          |
| `/trade/market/watchlist?productId=xxx` | POST   | 添加自选（通过产品ID）    |
| `/trade/market/watchlist?code=xxx`      | POST   | 添加自选（通过产品代码）    |
| `/trade/market/watchlist/{productId}`   | DELETE | 移除自选（通过产品ID）    |
| `/trade/market/watchlist/code/{code}`   | DELETE | 移除自选（通过产品代码）    |

### 8. 产品对比

**POST** `/trade/products/compare`

**请求体**：

```json
{
  "productIds": [1, 2, 3]
}
```

> 最多支持 5 个产品对比

---

### 附：Python 数据服务接口（内部使用）

> 以下接口运行在 `localhost:8888`，仅供后端定时任务同步使用，前端不直接调用

| 接口                     | 说明      |
| ---------------------- | ------- |
| `/api/index/cn`        | A股指数    |
| `/api/stock/cn/list`   | A股股票列表  |
| `/api/stock/cn/quote`  | A股单只报价  |
| `/api/stock/cn/kline`  | A股K线    |
| `/api/stock/hk/list`   | 港股列表    |
| `/api/stock/hk/quote`  | 港股报价    |
| `/api/stock/us/list`   | 美股列表    |
| `/api/stock/us/quote`  | 美股报价    |
| `/api/crypto/list`     | 加密货币列表  |
| `/api/crypto/quote`    | 加密货币报价  |
| `/api/forex/list`      | 外汇列表    |
| `/api/forex/quote`     | 外汇报价    |
| `/api/fund/etf/list`   | ETF基金列表 |
| `/api/fund/open/list`  | 开放式基金列表 |
| `/api/futures/list`    | 期货列表    |
| `/api/sector/list`     | 板块列表    |
| `/api/sector/stocks`   | 板块成分股   |
| `/api/market/overview` | 市场概览    |

---

## 附录四：响应数据加密

> v1.10.0 新增：防止用户通过浏览器 Network 直接查看敏感数据

### 1. 加密机制

- **算法**：AES/CBC/PKCS5Padding
- **密钥**：16 字节对称密钥
- **启用方式**：网关配置 `gateway.custom.encrypt.enabled: true`

### 2. 加密响应格式

启用加密后，响应体格式变为：

```json
{
  "encrypted": true,
  "data": "Base64编码的加密数据..."
}
```

### 3. 默认加密路径

- `/user/info`
- `/user/profile`
- `/user/risk-assessment/**`
- `/asset/**`
- `/trade/**`

### 4. 前端解密

```javascript
import CryptoJS from 'crypto-js';

const AES_KEY = 'SmartFin2025Key!';
const AES_IV = 'SmartFinance2025';

export function decryptResponse(encryptedData) {
  const key = CryptoJS.enc.Utf8.parse(AES_KEY);
  const iv = CryptoJS.enc.Utf8.parse(AES_IV);
  const decrypted = CryptoJS.AES.decrypt(encryptedData, key, {
    iv: iv,
    mode: CryptoJS.mode.CBC,
    padding: CryptoJS.pad.Pkcs7
  });
  return JSON.parse(decrypted.toString(CryptoJS.enc.Utf8));
}
```

> **注意**：开发环境建议关闭加密（`enabled: false`），便于调试。

---

## 附录五：数据库冗余表清理（v1.14.0）

> 本次清理移除了 5 张功能重复的冗余表

### 已删除的冗余表

| 删除的表                                | 替代表                                      | 说明                              |
| ----------------------------------- | ---------------------------------------- | ------------------------------- |
| `user_db.advisor_profile`           | `advisor_db.advisors`                    | 顾问信息统一到 advisors 表              |
| `user_db.user_advisor_relation`     | `advisor_db.client_relationships`        | 用户-顾问关系统一到 client_relationships |
| `admin_db.audit_logs`               | `risk_db.audit_logs`                     | 审计日志统一到 risk_db                 |
| `ai_db.ai_chat_history`             | `ai_db.ai_conversations`                 | AI 对话统一到 conversations 表        |
| `notification_db.message_templates` | `notification_db.notification_templates` | 通知模板统一到 notification_templates  |

> **注意**：如需回滚，可使用 `db/SteadyGame-DB-Backup.sql` 恢复
