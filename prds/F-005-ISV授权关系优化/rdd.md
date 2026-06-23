---
status: rdd-complete
phase: 2
context-loaded:
  - platform-support
  - user-persona
  - product-background
  - business-glossary
story-version: 1
created: 2026-06-07
---

# RDD：ISV授权关系优化

## 用户故事（已确认）

As an ISV 开发者（将 eSignGlobal 签署能力集成进自有系统的技术负责人），
I want to 建立不因固定时效失效的授权关系，其 refreshToken 有效性与被授权用户的订单周期保持一致，
So that 集成系统不会因授权到期被迫中断，同时平台的计费权益边界也能得到合理保护。

---

## 节 1：需求摘要

| 维度 | 内容 |
|------|------|
| 业务对象 | ISV 授权关系、OAuth Token（accessToken / refreshToken） |
| 用户角色 | 核心：ISV 开发者（受益于授权永久有效） |
| 使用场景 | ISV 系统集成长期运行，不因授权 365 天到期被强制中断 |
| 核心动作 | 系统：去除授权关系 365 天固定时效；refreshToken 有效期与被授权用户订单到期时间对齐（token expiry = 订单 end date） |
| 预期价值 | 企业集成不再因固定时效被强制中断；token 生命周期与客户付费周期自然对齐 |
| 成功指标 | ISV 因授权到期导致的集成中断投诉清零 |
| 需求优先级 | P1 |

---

## 节 2：初步方案

### 方案边界

- **本版做**：
  - 授权关系去除 365 天固定时效
  - refreshToken 有效期与被授权用户的订单到期时间绑定（token expiry = 订单 end date），订单有效则 token 有效

- **本版不做**：
  - 用户侧"已授权应用"管理页面
  - 撤销功能（用户主动撤销 ISV 授权）
  - Webhook 推送 `authorization.revoked` 事件
  - ISV 侧主动撤销接口
  - 平台管理员后台强制撤销
  - Scope 级细粒度撤销

- **后续规划**：
  - 用户侧已授权应用管理页面 + 撤销能力
  - ISV 侧撤销接口
  - 撤销触发 Webhook 通知

### 功能模块

| 模块名 | 类型 | 说明 |
|--------|------|------|
| 授权关系存储 | 调整 | 去除 expiry 字段或将 expiry 改为 null，授权关系不再设定时失效 |
| refreshToken 存储 | 调整 | 发放 refreshToken 时，expiry 取被授权用户当前订单的 end date（而非固定时长） |

### 核心流程

- **正常使用流**：
  授权建立 → 发放 refreshToken（expiry = 被授权用户订单 end date）→ accessToken 24h 到期 → 用 refreshToken 换新 accessToken（在订单有效期内无限循环）→ 订单到期后 refreshToken 自然过期，ISV 需引导用户重新授权


### 风险提示

- P1：授权关系长期有效后，token 泄露无自助修复手段（撤销能力后续版本才做）——当前阶段需在 ISV 开发文档中明确说明如需紧急解除授权须联系平台客服处理（合规类）
- P2：ISV 系统用户注销账号时，授权关系无法自动清理，形成"孤儿授权"——撤销接口与已授权应用管理页后续版本一起跟进（依赖类）
- P3：订单续费后 refreshToken 不会自动延期，需用户重新授权才能获得新的 refreshToken——这是预期行为，但需在 ISV 开发文档中说明清楚（体验类）
