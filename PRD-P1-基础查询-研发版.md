# Dino English 运营平台（CRM）· 一期「基础查询」研发版 PRD

> 面向研发的需求规格说明书（含数据模型、接口契约、状态机、校验规则、错误码、验收标准）。
> 上游产品 PRD（含交互原型）：<https://jiangshuang12345-tech.github.io/CRM-PRD/#mapping>
> 原型仓库：<https://github.com/jiangshuang12345-tech/CRM>　|　在线原型：<https://jiangshuang12345-tech.github.io/CRM/#/channels>

---

## 0. 文档说明

| 项 | 内容 |
| --- | --- |
| 版本 | v1.0（一期 · 基础查询） |
| 范围 | 渠道管理、用户基本信息查询、订单信息查询、登录鉴权 |
| 不在本期 | 标签 / 黑名单 / 角色权限 / 邀约提报 / 第三方整合（Breeze、Apespire）/ 写类订单操作（退款等）|
| 目标读者 | 后端、前端、测试 |
| 排期建议 | 3～4 周 |

本期定位「查得到、看得清、能归因」：渠道结构标准化并可生码归因；客服/运营可自助查询用户基本信息与订单信息，替代线下散表。

### 0.1 核心阻塞（开发前必须拉齐）

1. **渠道 code 口径**：与投放侧统一「生码规则 + 归因回传字段」，否则用户注册渠道归因无法闭环。
2. **数据来源**：用户、订单数据是新建库还是对接现有业务库（只读视图 / 同步 / 直连）需在 M1 前确认。
3. **用户状态判定**：体验/付费/流失依赖「商品包有效期 + 续订状态」，需确认这些字段由订单/履约侧提供。

---

## 1. 全局约定（Global Conventions）

### 1.1 技术栈与部署

- 前端：Vite + React + TypeScript + Ant Design 5。
- 当前原型为纯前端 Mock（localStorage）；一期需替换为真实后端接口。
- 接口风格：RESTful + JSON；时间统一返回 ISO8601（带时区）或时间戳，前端按北京时间（UTC+8）展示。
- 字符集：UTF-8；金额相关字段见 §1.6。

### 1.2 鉴权（登录）

- 登录方式：仅允许 `@dinoai.ai` 邮箱 + 邮箱验证码（OTP）。
- 流程：
  1. `POST /auth/otp` 入参 `{ email }`，服务端校验邮箱后缀为 `@dinoai.ai`，下发验证码。
  2. `POST /auth/login` 入参 `{ email, code }`，校验通过返回 `accessToken`（建议 JWT）。
- 校验规则：
  - 邮箱后缀非 `@dinoai.ai` → 拒绝（`ERR_EMAIL_DOMAIN`）。
  - 验证码 6 位数字，有效期建议 5 分钟，最多尝试 5 次。
- 所有业务接口需在 Header 携带 `Authorization: Bearer <token>`；未登录/过期返回 `401`。

### 1.3 统一响应结构

```json
{
  "code": 0,
  "message": "ok",
  "data": { }
}
```

- `code = 0` 表示成功；非 0 为业务错误码（见 §1.7）。
- HTTP 状态码：`200` 业务正常，`400` 参数错误，`401` 未鉴权，`403` 无权限，`404` 资源不存在，`409` 冲突，`500` 服务异常。

### 1.4 分页约定

- 列表接口统一分页入参：`page`（从 1 开始，默认 1）、`pageSize`（默认 20，最大 100）。
- 列表返回结构：

```json
{
  "list": [],
  "total": 0,
  "page": 1,
  "pageSize": 20
}
```

### 1.5 业务线 · 币种 · 国家码映射（枚举）

| 业务线 code | 业务线 | 本地币种 | 国家码 | 可选币种 |
| --- | --- | --- | --- | --- |
| `KR` | 韩国 | KRW | +82 | KRW / USD |
| `SA` | 沙特 | SAR | +966 | SAR / USD |
| `MY` | 马来 | MYR | +60 | MYR / USD |
| `VN` | 越南 | VND | +84 | VND / USD |
| `OTHER` | 其他 | USD | +1 | USD |

> 业务线 code 由研发与产品最终敲定，本表给出建议值；所有模块共用此枚举。

### 1.6 金额规则

- 金额字段建议以「最小货币单位整数」或 `decimal(18,2)` 存储，避免浮点误差。
- 列表展示需带币种（如 `₩` / `﷼` / `RM` / `₫` / `$` 或 ISO 4217 代码）。
- KRW、VND 等无小数币种展示时不带小数位（前端按币种格式化）。

### 1.7 错误码（建议，可按团队规范调整）

| code | 含义 | 触发场景 |
| --- | --- | --- |
| `0` | 成功 | — |
| `ERR_PARAM` | 参数校验失败 | 必填缺失 / 格式错误 |
| `ERR_EMAIL_DOMAIN` | 邮箱域名不允许 | 非 `@dinoai.ai` |
| `ERR_OTP_INVALID` | 验证码错误/过期 | 登录 |
| `ERR_UNAUTHORIZED` | 未登录/Token 失效 | 缺/过期 token |
| `ERR_NOT_FOUND` | 资源不存在 | 查询不存在 ID |
| `ERR_CHANNEL_CODE_EXISTS` | 渠道 code 冲突 | 生码重复 |
| `ERR_CHANNEL_HAS_CHILDREN` | 渠道含下级 | 删除校验 |
| `ERR_INTERNAL` | 服务异常 | 兜底 |

---

## 2. 模块 1.1 渠道管理（P0）

维护标准化渠道分级结构，并在任意层级生成实际投放使用的渠道 code，作为用户注册归因的唯一口径。

### 2.1 功能清单

| 功能点 | 说明 | 优先级 |
| --- | --- | --- |
| 新增渠道类型 | 如 自然流量 / landingpage / KOL，作为树的根节点 | P0 |
| 分级渠道维护 | 在类型下逐级新增 一/二/三级渠道（如 国家 → 媒体 → 达人）| P0 |
| 生成 / 查看渠道 code | 任意层级生成唯一 code，可一键复制；已生成则展示并支持复制 | P0 |
| 重命名 / 删除 | 删除级联删除下级渠道与 code（二次确认）| P1 |

### 2.2 数据模型

`channel_node`（渠道树节点，类型节点也存于此表，用 `level=0` 表示根/类型）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | bigint / uuid | 主键 |
| `parent_id` | bigint / null | 父节点；类型节点为 null |
| `name` | varchar(64) | 节点名称（同父下不可重名）|
| `level` | tinyint | 0=类型，1=一级，2=二级，3=三级（一期最大 3）|
| `channel_type` | varchar(32) | 冗余存所属类型 code（用于 code 前缀）|
| `code` | varchar(64) / null | 渠道 code，可空（未生码时）|
| `path` | varchar | 物化路径（如 `1/4/9`），便于级联与展示 |
| `created_by` | varchar | 创建人 |
| `created_at` / `updated_at` | datetime | 时间 |

约束：
- `(parent_id, name)` 唯一。
- `code` 全局唯一。
- `level <= 3`。

### 2.3 渠道 code 生成规则

- 格式：`{类型前缀}_{子串}_{随机串}`，示例：`natural_kr_aso_4f9k2a`。
  - 类型前缀来自渠道类型（如 自然流量 → `natural`，KOL → `kol`，landingpage → `lp`）。
  - 随机串：建议 6 位 `[a-z0-9]`，保证全局唯一（生成后查重，冲突重试）。
- 任意层级均可生码；同一节点已生成则返回已存在 code，不重复生成。
- **生码规则需与投放侧确认**：前缀映射表、随机串长度、是否需要校验位/可读性。

### 2.4 接口设计

| Method | Path | 说明 |
| --- | --- | --- |
| GET | `/channels/tree` | 返回整棵渠道树（含 code）|
| POST | `/channels/types` | 新增渠道类型，入参 `{ name }` |
| POST | `/channels` | 新增渠道节点，入参 `{ parentId, name }` |
| POST | `/channels/{id}/code` | 为节点生成 code（已存在则返回原 code）|
| PUT | `/channels/{id}` | 重命名，入参 `{ name }` |
| DELETE | `/channels/{id}` | 删除（级联，二次确认在前端）|

`GET /channels/tree` 返回示例：

```json
{
  "code": 0,
  "data": [
    {
      "id": 1, "name": "自然流量", "level": 0, "code": null,
      "children": [
        { "id": 4, "name": "韩国", "level": 1, "code": "natural_kr_4f9k2a", "children": [] }
      ]
    }
  ]
}
```

### 2.5 业务规则与校验

- 新增节点的 `level = 父节点 level + 1`，超过 3 拒绝（`ERR_PARAM`）。
- 同一父节点下 `name` 不可重复。
- 删除节点：若存在下级，前端二次确认后级联删除其所有子孙节点与对应 code；后端按 `path` 前缀批量删除。
- code 一经生成不建议修改（影响已投放归因）；如需「重新生码」需产品确认是否保留历史。

### 2.6 关键流程

1. 新增渠道类型 → 输入名称（如 KOL）。
2. 在类型/渠道节点逐级新增下级渠道。
3. 在目标层级「生成 code」→ 弹窗展示 → 复制。
4. 该 code 用于投放链接；用户由此注册时写入 `channel_code`，在用户中心可见 → 形成归因闭环。

---

## 3. 模块 1.2 用户基本信息查询（P0）

客服/运营按多条件检索用户，查看基本信息、注册方式与登录账号、注册归因与状态，并可维护可编辑的基础资料。

### 3.1 本期更新点（务必实现）

- 支持 **5 种注册方式**：谷歌邮箱 / Facebook / kakao / 手机号 / AppID。
- 列表新增「注册方式」「登录账号」两列。
- 非手机号方式（谷歌邮箱 / Facebook / AppID）手机号列显示 `—`。
- 学生姓名列**仅展示本地名**（已去掉英文名；无本地名时回退展示名）。
- 筛选项新增「注册方式（登录账号类型）」维度，可单独筛选。

### 3.2 功能清单

| 功能点 | 说明 | 优先级 |
| --- | --- | --- |
| 关键字搜索 | 学生ID / 姓名 / 登录账号 / 手机号 模糊搜索 | P0 |
| 多注册方式展示 | 5 种方式 + 登录账号；支持非手机号注册 | P0 |
| 条件筛选 | 业务线、用户状态、注册方式（登录账号类型）| P0 |
| 列表 + 分页 | 展示基本字段，支持分页与每页条数 | P0 |
| 修改基本信息 | 本地名 / 性别 / 出生日期 / 业务线 | P1 |

### 3.3 数据模型

`user`（学生/用户）

| 字段 | 类型 | 可编辑 | 说明 |
| --- | --- | --- | --- |
| `student_id` | varchar | 否 | 用户唯一标识 |
| `local_name` | varchar | 是 | 本地名（列表展示）|
| `display_name` | varchar | 否 | 展示名（本地名为空时回退）|
| `register_type` | enum | 否 | 注册方式，见 §3.5 |
| `login_account` | varchar | 否 | 登录账号（邮箱 / FB / kakaoID / AppID / 手机号）|
| `phone` | varchar / null | 否 | 仅 kakao / 手机号 方式有，其余为 null（展示 `—`）|
| `gender` | enum | 是 | 性别（男/女/未知）|
| `birth_date` | date / null | 是 | 出生日期 |
| `business_line` | enum | 是 | 业务线（见 §1.5）|
| `channel_code` | varchar / null | 否 | 注册渠道 code，关联渠道管理 |
| `country_code` | varchar | 否 | 注册地国家码（如 +82）|
| `register_time` | datetime | 否 | 注册时间（北京时间展示）|
| `user_status` | enum | 否 | 注册/体验/付费/流失（系统流转）|

### 3.4 注册方式与登录账号映射（枚举）

| `register_type` | 注册方式 | 登录账号示例 | 是否带手机号 |
| --- | --- | --- | --- |
| `GOOGLE` | 谷歌邮箱 | 邮箱地址 | 否（手机号列 `—`）|
| `FACEBOOK` | Facebook | 邮箱（可拿到时）| 否（`—`）|
| `APPID` | AppID | 邮箱（可拿到时）| 否（`—`）|
| `KAKAO` | kakao | 手机号 | 是 |
| `PHONE` | 手机号 | 手机号 | 是 |

### 3.5 用户状态定义与流转（系统自动，不可手工编辑）

| `user_status` | 定义 | 判定口径 |
| --- | --- | --- |
| `REGISTERED` 注册 | 已注册未付费 | 完成注册，从未付费 |
| `TRIAL` 体验 | 已付费 7 天体验包且在有效期内 | 持有体验包 且 now ≤ 体验有效期 |
| `PAID` 付费 | 付费正式包且在有效期内（含到期自动续订）| 持有正式包 且 now ≤ 有效期 |
| `CHURNED` 流失 | 付费正式包已过有效期且未续订 | 曾持有正式包 且 now > 有效期 且 未续订 |

> 状态由「商品包有效期 + 续订结果」驱动，建议由订单/履约侧提供有效期与续订状态字段，CRM 侧计算或定时刷新。需明确：是实时计算还是定时任务回写 `user_status`。

### 3.6 接口设计

| Method | Path | 说明 |
| --- | --- | --- |
| GET | `/users` | 列表查询（关键字 + 筛选 + 分页）|
| GET | `/users/{studentId}` | 用户详情 |
| PUT | `/users/{studentId}` | 修改基本信息 |

`GET /users` 查询参数：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `keyword` | string | 模糊匹配 学生ID / 姓名 / 登录账号 / 手机号 |
| `businessLine` | enum | 业务线筛选 |
| `userStatus` | enum | 用户状态筛选 |
| `registerType` | enum | 注册方式筛选（登录账号类型）|
| `page` / `pageSize` | int | 分页 |

`PUT /users/{studentId}` 可编辑字段：`localName` / `gender` / `birthDate` / `businessLine`（其余字段只读，传入忽略或报错由团队约定）。

### 3.7 业务规则与校验

- 列表「学生姓名」= `local_name`，为空回退 `display_name`。
- 手机号列：`register_type ∈ {KAKAO, PHONE}` 展示 `phone`，否则展示 `—`。
- `keyword` 模糊匹配范围：学生ID / 姓名 / 登录账号 / 手机号。
- 修改信息校验：`businessLine` 必须在枚举内；`birthDate` 不得为未来日期。
- 用户状态不可由该接口修改（只读）。

---

## 4. 模块 1.3 订单信息查询（P0）

按多条件检索订单，查看商品、归属用户、金额与支付信息，支撑客服对账与退款核查。**一期仅查询，不含创建/退款写操作。**

### 4.1 功能清单

| 功能点 | 说明 | 优先级 |
| --- | --- | --- |
| 关键字搜索 | 订单ID / 学生ID / 商品名称 模糊搜索 | P0 |
| 条件筛选 | 订单状态、支付方式 | P0 |
| 列表 + 分页 | 原价 / 实付 / 币种 / 支付时间 | P0 |

### 4.2 数据模型

`order`（订单，本期只读）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `order_id` | varchar | 订单唯一标识，如 `DN2026061800001` |
| `product_name` | varchar | 所购商品名称 |
| `student_id` | varchar | 归属用户，关联 `user` |
| `user_status` | enum | 用户状态（展示双标签之一）|
| `order_status` | enum | 订单状态，见 §4.3 |
| `original_amount` | decimal | 原价（含币种）|
| `paid_amount` | decimal | 实际付款金额（含币种）|
| `currency` | enum | 币种（见 §1.5）|
| `pay_method` | enum | 支付方式，见 §4.4 |
| `paid_time` | datetime / null | 成功支付时间；未支付为 `—` |

### 4.3 订单状态枚举

| `order_status` | 含义 |
| --- | --- |
| `PENDING` | 待支付 |
| `PAID` | 已支付 |
| `REFUNDED` | 已退款 |
| `CANCELLED` | 已取消 |

### 4.4 支付方式枚举

| `pay_method` | 含义 |
| --- | --- |
| `APP_STORE` | App Store |
| `GOOGLE_PLAY` | Google Play |
| `STRIPE` | Stripe |
| `PAYPAL` | PayPal |

### 4.5 接口设计

| Method | Path | 说明 |
| --- | --- | --- |
| GET | `/orders` | 列表查询（关键字 + 筛选 + 分页）|
| GET | `/orders/{orderId}` | 订单详情（可选，按需）|

`GET /orders` 查询参数：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `keyword` | string | 模糊匹配 订单ID / 学生ID / 商品名称 |
| `orderStatus` | enum | 订单状态筛选 |
| `payMethod` | enum | 支付方式筛选 |
| `page` / `pageSize` | int | 分页 |

### 4.6 业务规则

- 列表展示「用户状态 + 订单状态」双标签。
- 金额按业务线本地币种或 USD 展示，需带币种；`paid_time` 为空展示 `—`。
- 本期**不开放**创建、退款、取消等写操作；退款等待权限体系（后续）就绪再开放。

---

## 5. 验收标准（一期 · M1）

- [ ] 登录鉴权：仅 `@dinoai.ai` 邮箱可获取验证码并登录；非法域名/错误验证码被拒绝。
- [ ] 渠道管理：渠道树可增删改；任意层级可生码并复制；删除级联生效（二次确认）；用户注册渠道 code 与渠道树打通。
- [ ] 用户查询：关键字（学生ID/姓名/登录账号/手机号）+ 业务线/状态/注册方式筛选 + 分页可用；5 种注册方式与登录账号正确展示；非手机号方式手机号列为 `—`；姓名仅展示本地名；可修改本地名/性别/出生日期/业务线。
- [ ] 订单查询：关键字（订单ID/学生ID/商品）+ 订单状态/支付方式筛选 + 分页可用；金额含币种、支付时间正确（未支付为 `—`）。
- [ ] 字段口径与本 PRD 数据字典一致；业务线/币种/国家码映射一致。
- [ ] 所有列表接口遵循统一分页与响应结构；错误返回统一错误码。

---

## 6. 待确认事项（Open Questions）

1. 渠道 code 生码规则（前缀映射、随机串长度、是否可重新生码）与投放侧最终口径。
2. 用户/订单数据来源：新建库 vs 对接现有业务库（只读/同步/直连）。
3. 用户状态计算方式：实时计算 vs 定时任务回写；有效期/续订字段由谁提供。
4. 业务线 code、枚举值的最终命名。
5. 用户「修改信息」是否需要操作日志/审计（为后续权限体系预留）。
6. 列表查询是否需要导出（CSV/Excel）能力（产品未明确，默认不做）。

---

## 7. 关联资源

- 整体规划 PRD（交互文档）：<https://jiangshuang12345-tech.github.io/CRM-PRD/#mapping>
- 规划仓库：<https://github.com/jiangshuang12345-tech/CRM-PRD>
- 原型仓库：<https://github.com/jiangshuang12345-tech/CRM>
- 在线原型：<https://jiangshuang12345-tech.github.io/CRM/#/channels>
- 技术栈：Vite + React + TypeScript + Ant Design 5（原型为纯前端 Mock，数据存 localStorage）
