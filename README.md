# 智能金融理财平台 - 移动端

> 基于 uni-app x 的跨平台智能金融理财移动应用

## 技术栈

- **框架**: uni-app x (Vue 3)
- **语言**: UTS (UniApp TypeScript)
- **样式**: SCSS
- **UI框架**: 自研组件库 (sg-*)
- **目标平台**: 微信小程序、支付宝小程序、Android、iOS、鸿蒙

## 项目结构

```
SteadyGameFront-Mobility/
├── api/                    # API接口层
│   ├── auth.uts           # 认证相关接口
│   ├── user.uts           # 用户相关接口
│   ├── asset.uts          # 资产相关接口
│   ├── trade.uts          # 交易相关接口
│   ├── ai.uts             # AI服务接口
│   └── index.uts          # 统一导出
├── components/             # 通用组件库
│   ├── sg-button/         # 按钮组件
│   ├── sg-navbar/         # 导航栏组件
│   ├── sg-tabbar/         # 底部导航组件
│   ├── sg-input/          # 输入框组件
│   ├── sg-card/           # 卡片组件
│   ├── sg-modal/          # 模态框组件
│   ├── sg-loading/        # 加载组件
│   ├── sg-empty/          # 空状态组件
│   ├── sg-list-item/      # 列表项组件
│   └── sg-asset-card/     # 资产卡片组件
├── config/                 # 配置文件
│   └── index.uts          # 全局配置
├── pages/                  # 页面目录
│   ├── home/              # 首页
│   ├── asset/             # 资产页
│   ├── ai/                # AI助手页
│   ├── mine/              # 我的页
│   ├── login/             # 登录页
│   ├── register/          # 注册页
│   ├── risk-assess/       # 风险评估页
│   ├── ai-chat/           # AI对话页
│   ├── orders/            # 订单列表页
│   └── products/          # 产品列表页
├── static/                 # 静态资源
│   └── tabbar/            # TabBar图标
├── styles/                 # 全局样式
│   ├── variables.scss     # 样式变量
│   └── common.scss        # 通用样式类
├── utils/                  # 工具函数
│   ├── storage.uts        # 本地存储
│   ├── date.uts           # 日期处理
│   ├── format.uts         # 格式化
│   ├── validate.uts       # 表单验证
│   ├── request.uts        # HTTP请求
│   ├── platform.uts       # 平台兼容
│   └── index.uts          # 统一导出
├── doc/                    # 文档目录
│   ├── MobilityFunctionDemand.md    # 功能需求文档
│   └── FRONTEND_API_INTEGRATION.md  # API对接文档
├── App.uvue               # 应用入口
├── main.uts               # 主入口文件
├── pages.json             # 页面配置
├── manifest.json          # 应用配置
└── uni.scss               # uni-app样式变量
```

## 功能模块

### 普通用户端

1. **账号与安全**
   - 手机号注册
   - 密码登录/短信验证码登录
   - 生物识别登录（指纹/面容）
   - 交易密码管理
   - 账号冻结/解冻

2. **用户画像与风险评估**
   - 分步式风险测评问卷
   - 风险等级展示
   - 一键重测

3. **资产管理**
   - 资产总览（环形图展示）
   - 资产明细查看
   - 收益统计

4. **智能理财服务**
   - AI产品推荐
   - 智能理财规划
   - AI问答（支持语音）

5. **交易执行**
   - 产品申购/赎回
   - 订单管理
   - 行情查看

6. **信息与服务**
   - 理财知识
   - 客服服务

### 理财顾问端

1. 客户管理
2. AI辅助服务
3. 业绩管理

## 开发规范

1. **代码分离**: HTML/JS/CSS 分离成独立文件
2. **组件化开发**: 可复用功能抽取为组件
3. **工具化**: 封装通用工具函数
4. **目录隔离**: 合理组织文件结构
5. **注释规范**: 企业级标准注释

## 快速开始

```bash
# 安装HBuilderX
# 导入项目
# 运行到模拟器或真机
```

## API配置

修改 `config/index.uts` 中的 `BASE_URL` 为后端网关地址：

```typescript
export const BASE_URL = 'http://localhost:8001'
```

## 版本历史

- v1.0.0 - 初始版本，包含基础功能模块
