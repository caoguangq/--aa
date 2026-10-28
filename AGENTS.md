# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## 项目概述

苍穹外卖 — 一个完整的外卖点餐平台，包含三个子项目：

| 子项目 | 目录 | 说明 |
|--------|------|------|
| 后端服务 | `sky-take-out/` | Java Spring Boot 多模块 Maven 项目 |
| 管理后台 | `project-sky-admin-vue-ts/` | Vue 2 + TypeScript 管理端 SPA |
| 用户小程序 | `mp-weixin/` | UniApp 编译后的微信小程序 |

## 后端 (`sky-take-out/`)

### 构建与运行

```bash
# 编译
cd sky-take-out
mvn clean compile

# 运行（主类 com.sky.SkyApplication，端口 8080）
mvn spring-boot:run

# 打包
mvn clean package -DskipTests
```

配置文件在 `sky-server/src/main/resources/`：
- `application.yml` — 主配置，激活 `dev` profile
- `application-dev.yml` — 开发环境具体值（数据库、Redis、阿里云OSS、微信支付、AI/vLLM、百度地图）

⚠️ `application-dev.yml` 包含真实密钥（数据库密码、微信支付密钥、阿里云OSS密钥、百度地图AK），不要提交到公开仓库。

### 模块结构

```
sky-take-out/
├── pom.xml              # 父 POM，Spring Boot 2.7.3，统一依赖版本管理
├── sky-common/          # 共享常量、异常、工具类、统一返回结果
├── sky-pojo/            # DTO / Entity / VO（纯数据对象，无业务逻辑）
└── sky-server/          # 主应用：Controller → Service → Mapper
```

**模块依赖关系**：`sky-server` → `sky-common` + `sky-pojo`

### 后端分层架构

```
controller/  (admin/  + user/)
    ↓
service/  (接口)  →  service/impl/  (实现)
    ↓
mapper/  (MyBatis 接口，XML 在 resources/mapper/)
```

### 双通道 API 设计

URL 路径前缀区分两个端，由 `WebMvcConfiguration` 注册不同的 JWT 拦截器：

| 路径前缀 | 拦截器 | 说明 |
|----------|--------|------|
| `/admin/**` | `JwtTokenAdminInterceptor` | 管理端（员工登录 `/admin/employee/login` 除外） |
| `/user/**` | `JwtTokenUserInterceptor` | 用户端（登录 `/user/user/login`、店铺状态 `/user/shop/status` 和领券中心 `/user/coupon/center` 除外） |

- Admin JWT：请求头 `token`，secret 为 `itcast`，claim 中存 `empId`
- User JWT：请求头 `authentication`，secret 为 `itheima`，claim 中存 `userId`
- 用户身份存入 `BaseContext.threadLocal`（ThreadLocal），通过 `BaseContext.getCurrentId()` 获取

### 关键设计模式

- **AOP 自动填充**：`@AutoFill` 注解 + `AutoFillAspect` 切面，在 Mapper 方法执行前自动注入 `create_time`/`create_user`/`update_time`/`update_user`。`OperationType` 枚举区分 INSERT（填全部四个字段）和 UPDATE（只填 update 字段）。通过反射调用 setter，要求实体类必须有对应的 setter 方法。
- **统一响应**：`Result<T>` 类封装所有 API 返回，`GlobalExceptionHandler` 全局捕获 `BaseException`（业务异常，返回中文错误消息）和 `SQLIntegrityConstraintViolationException`（唯一键冲突，提取重复字段名返回"xxx已存在"）。没有兜底的 `Exception` 处理器，未处理的异常类型会走 Spring Boot 默认 500。
- **分页**：使用 PageHelper，返回 `PageResult`
- **API 文档**：Knife4j（Swagger 增强），启动后访问 `/doc.html`
- **WebSocket**：两个独立的 WebSocket 端点（见下方 WebSocket 章节）
- **Jackson 配置**：`JacksonObjectMapper` 在消息转换器链最前面，日期格式为 `yyyy-MM-dd HH:mm`（不带秒），注册了 `LocalDateTime`/`LocalDate`/`LocalTime` 自定义序列化器

### 定时任务

`SkyApplication` 上标注 `@EnableScheduling`，三个定时任务：

| 任务 | 频率 | 行为 |
|------|------|------|
| `OrderTask.processTimeoutOrder()` | 每分钟 | 取消超过15分钟未支付的待付款订单 |
| `OrderTask.processDeliveryOrder()` | 每天凌晨1点 | 将派送中超过60分钟的订单标记为已完成 |
| `CouponTask.expireCoupons()` | 每天凌晨1点 | 批量标记过期优惠券 |

### WebSocket

**1. 通知推送** (`/ws/{sid}`，`WebSocketServer.java`)
- JSR 356 `@ServerEndpoint`，维护 `Map<String, Session>` 管理连接
- `WebSocketConfiguration` 暴露 `ServerEndpointExporter`
- 用于管理端实时订单提醒（厨房大屏），支持单播和广播

**2. AI 对话** (`/ws/ai/{userId}`，`AiChatEndpoint.java`)
- 每个消息在独立 Thread 中调用 `AiChatService.chat()`，支持流式响应

### AI / LLM 子系统

完整的 AI 对话功能，通过 WebSocket 提供流式聊天：

| 类 | 职责 |
|----|------|
| `AiChatService` | 核心编排：构建中餐馆助手系统提示词，维护对话状态，工具调用循环（最多5轮），流式 vLLM 响应 |
| `VllmClient` | HTTP 客户端，发送 streaming chat completions 请求到 vLLM 端点（阿里云 DashScope `qwen3.7-plus`） |
| `AiTool` | 工具接口，6 个实现：`SearchDishTool`、`GetCartTool`、`GetOrderHistoryTool`、`GetCouponsTool`、`CalcDistanceTool`、`GetWeatherTool` |
| `AiToolRegistry` | 收集所有 `AiTool` Bean，构建 OpenAI 兼容的工具定义 |
| `AiToolExecutor` | 按名称分发工具调用 |
| `BaiduMapService` | 百度地图 API 集成（地理编码、距离计算） |
| `AiGraphNode` / `AiGraphRouter` | 图路由接口（`nodes/` 目录为空，尚未实现） |

配置：`application-dev.yml` 中 `sky.ai` 段（模型名、max-tokens=2048、temperature=0.7、timeout=60s、api-key 有默认空值）。

⚠️ Spring 允许循环引用（`allow-circular-references: true`），Spring bean 图中存在循环依赖。

### 异常体系

所有异常在 `sky-common` 中定义，均继承 `BaseException extends RuntimeException`：
- 认证类：`AccountLockedException`、`AccountNotFoundException`、`PasswordErrorException`、`LoginFailedException`
- 业务约束类：`DeletionNotAllowedException`、`SetmealEnableFailedException`
- 领域类：`AddressBookBusinessException`、`ShoppingCartBusinessException`、`OrderBusinessException`、`CouponBusinessException`
- 会话类：`UserNotLoginException`、`PasswordEditFailedException`

错误消息中文常量集中在 `MessageConstant` 中。

### 常量与枚举

- `AutoFillConstant`：AOP 反射用 setter 方法名
- `JwtClaimsConstant`：JWT claim key 常量
- `MessageConstant`：30+ 中文错误消息
- `PasswordConstant`：`DEFAULT_PASSWORD = "123456"`
- `StatusConstant`：`ENABLE = 1`，`DISABLE = 0`
- 枚举仅 `OperationType`（INSERT / UPDATE）

### 数据库

无 Flyway/Liquibase。基础表结构在仓库外创建。唯一的 SQL 脚本是 `sql/coupon.sql`（手动迁移，创建 `coupon`、`user_coupon` 表，给 `orders` 加 `coupon_id`/`discount_amount` 列）。

### 日志

SLF4J + Logback（Spring Boot 默认），所有类用 `@Slf4j`。YAML 中配置日志级别：`com.sky.mapper: debug`，`com.sky.service: info`，`com.sky.controller: info`。无自定义 `logback-spring.xml`。

### 测试

`spring-boot-starter-test` 已声明但项目中无 JUnit 测试类。`sky-server/src/test/` 下只有一个 `POItest.java`（手动 main 方法测试 POI Excel 读写），包名 `com.jxaiqtc` 而非 `com.sky`，属于 spike 代码。

### 技术栈要点

- MyBatis + Druid 连接池，驼峰命名自动映射
- Redis（Spring Data Redis + Spring Cache 注解 `@EnableCaching`）
- 阿里云 OSS 文件上传、微信支付 v3、Apache POI Excel 导出
- JWT 库 `jjwt:0.9.1`
- 定时任务（`@EnableScheduling`）

## 管理后台 (`project-sky-admin-vue-ts/`)

### 构建与运行

```bash
cd project-sky-admin-vue-ts
yarn install       # 安装依赖
yarn serve         # 开发运行（端口 8888，host 0.0.0.0）
yarn build         # 生产构建
yarn build:uat     # UAT 构建（使用 .env.production.uat）
yarn lint          # ESLint 检查
yarn svg           # 从 src/icons/svg 生成 Vue SVG 图标组件
yarn test:unit     # Jest 单元测试
yarn test:e2e      # Cypress E2E 测试
```

### 架构

- **框架**：Vue 2.6 + TypeScript 3.6（vue-class-component + vue-property-decorator）
- **UI 库**：Element UI
- **状态管理**：Vuex 3（vuex-module-decorators），token 通过 js-cookie 持久化
- **路由**：Vue Router，Layout 为根组件，子页面懒加载（webpack chunk name 注释）
- **图表**：ECharts 5（注入为 `Vue.prototype.$echarts`）
- **HTTP**：Axios，baseURL 为 `/api`，超时 10 分钟
- **代理**：开发模式下 `/api` 代理到后端 `VUE_APP_URL`（`pathRewrite` 去除 `/api` 前缀）
- **CSS**：SCSS，通过 style-resources-loader 自动注入 `_variables.scss` 和 `_mixins.scss`
- **构建**：Vue CLI 3 + webpack 4，非 Vite

### 路由结构

单一 `src/router.ts`，一个 Layout 父路由包含所有子页面：

| 路由 | 组件 | 说明 |
|------|------|------|
| `/login` | `views/login/` | 无需认证，隐藏 |
| `/404` | `views/404.vue` | 无需认证，隐藏 |
| `/` → Layout | `layout/` | 主布局（侧边栏 + 导航栏 + AppMain） |
| `/dashboard` | `views/dashboard/` | 工作台，`affix: true` 固定标签页 |
| `/statistics` | `views/statistics/` | 数据统计 |
| `/order` | `views/orderDetails/` | 订单管理 |
| `/setmeal` | `views/setmeal/` | 套餐管理 |
| `/setmeal/add` | `views/setmeal/addSetmeal.vue` | 添加套餐（隐藏） |
| `/dish` | `views/dish/` | 菜品管理 |
| `/dish/add` | `views/dish/addDishtype.vue` | 添加菜品（隐藏） |
| `/category` | `views/category/` | 分类管理 |
| `/coupon` | `views/coupon/` | 优惠券管理 |
| `/employee` | `views/employee/` | 员工管理 |
| `/employee/add` | `views/employee/addEmployee.vue` | 添加员工（隐藏） |

`meta.hidden: true` 的路由不出现在侧边栏。`meta.notNeedAuth: true` 的路由跳过认证守卫。`*` 重定向到 `/404`。

### 权限 / 认证模型

**认证守卫** (`src/permission.ts`，`beforeEach`)：
1. 检查 cookie 中是否存在 `token`
2. 有 token → 放行；没有且 `notNeedAuth` 不为 true → 跳转 `/login`
3. `afterEach` 设置 `document.title`

**Token 流程**：
- 登录 → `POST /employee/login` → token 同时存入 Vuex state 和 js-cookie
- 每次 HTTP 请求 → Axios 拦截器自动附加请求头 `token`
- 用户信息缓存在 cookie 的 JSON 字符串中（`GetUserInfo()` 从 cookie 读取，不从 API 获取）
- 登出 → `POST /employee/logout` → 清除 Vuex 和所有 cookie

**HTTP 请求层** (`src/utils/request.ts`)：
- 请求拦截器：除 `/login` 外，无 token 时重定向到 `/login`；GET 请求手动将嵌套对象序列化为 bracket notation query string
- 重复请求合并：用 md5 对 `url + method + data` 做哈希，通过 `axios.CancelToken` 取消重复的 inflight 请求
- 响应拦截器：HTTP 401 → 跳转 `/login`

### 环境变量

| 变量 | development | production | 用途 |
|------|-------------|------------|------|
| `VUE_APP_URL` | `http://localhost:8080/admin` | `''` | devServer 代理目标 |
| `VUE_APP_BASE_API` | `/api` | `/api` | Axios baseURL |
| `VUE_APP_DELETE_PERMISSIONS` | `true` | `false` | 是否启用删除功能 |
| `VUE_APP_SOCKET_URL` | `ws://localhost:8080/ws/` | `''` | WebSocket 地址 |

### Vuex Store

使用 `vuex-module-decorators` 的动态模块：

- **`AppModule`**：`sidebar`（开合状态/动画）、`device`（Desktop/Mobile）、`statusNumber`
- **`UserModule`**：`token`、`name`、`avatar`、`storeId`、`roles`、`username`、`userInfo`

组件通过导出的模块实例直接访问（`UserModule.username`、`UserModule.Login(...)`），不使用 `$store` 或 `mapState`。

### 组件编码模式

所有页面使用 class-style 组件 + 装饰器：
```typescript
@Component({ name: 'PageName', components: { ... } })
export default class extends Vue {
  private data = ''
  get computedValue() { return ... }           // computed = getter
  created() { this.init() }                    // 每个页面统一在 created 中调 init()
  private async init() { ... }
}
```

- API 响应判定：`String(res.data.code) === '1'` 为成功（不是 `=== 200`）
- 错误提示：`this.$message.error('请求出错了：' + err.message)`
- 销毁操作：`this.$confirm()` 确认弹窗
- 分页：`page`/`pageSize`/`counts` + `<el-pagination>`
- 全局注入：`Vue.prototype.moment`、`Vue.prototype.$echarts`、`Vue.prototype.$checkProcessEnv`

### TypeScript 配置

- `strict: false`（严格模式关闭）
- `experimentalDecorators: true`（装饰器语法必需）
- 路径别名 `@/*` → `src/*`
- `.vue` 文件排除在 ts 编译之外（由 vue-loader 处理），通过 `src/shims-vue.d.ts` 声明模块类型

### 测试

**单元测试（Jest）**：
- 配置文件 `jest.config.js`，匹配 `tests/unit/**/*.spec.(js|jsx|ts|tsx)`
- `ts-jest` 转换 `.ts`，`vue-jest` 转换 `.vue`
- 开启覆盖率收集（`src/utils/**` 和 `src/components/**`）
- 实际只有 2 个测试文件：`Breadcrumb.spec.ts`、`validate.spec.ts`
- 测试模式：`createLocalVue()` + `mount()` from `@vue/test-utils`

**E2E 测试（Cypress）**：
- `cypress.json` 存在，`tests/e2e/plugins/index.js` 存在
- 没有实际的测试 spec 文件

### 图标系统

两种图标方案并存：
1. **iconfont CSS**：侧边栏菜单使用 CSS class（icon-dashboard、icon-order 等）
2. **SVG 图标**：`src/icons/svg/` 存放源 SVG，`yarn svg` 生成 `.ts` 组件到 `src/icons/components/`，通过 `vue-svgicon` 使用

## 微信小程序 (`mp-weixin/`)

⚠️ **这是编译产物，不是源代码。** 原始 UniApp 源项目在 `D:/project/project-rjwm-weixin-uniapp/`，通过 HBuilderX 编译到 `mp-weixin/`。所有逻辑打包在 `common/runtime.js`、`common/vendor.js`、`common/main.js` 四个 webpack bundle 中，不能直接编辑 `.vue` 文件。

### 项目结构

```
mp-weixin/
├── app.js                # 入口：require 三个 bundle
├── app.json              # 页面注册 + 窗口配置（无 tabBar）
├── app.wxss              # 导入 common/main.wxss
├── project.config.json   # 微信开发者工具配置（appid: wxe5180c45ab1b3f09）
├── common/               # 编译产物（runtime.js、vendor.js、main.js、main.wxss）
├── pages/                # 12 个页面（每个目录 .js/.wxml/.wxss/.json）
├── components/           # 8 个全局组件
├── node-modules/         # ⚠️ 手动维护（注意是连字符），仅 @dcloudio/uni-ui 下 2 个组件
├── uni_modules/          # 5 个 uni-ui 模块
└── static/               # 38 个图片资源
```

### 页面列表（12个，无 tabBar，完全自定义导航）

| 页面 | 路径 | 说明 |
|------|------|------|
| 首页 | `pages/index/index` | 主点餐页（含 AI 对话框） |
| 订单/购物车 | `pages/order/index` | 购物车下单 |
| 菜品详情 | `pages/details/index` | 菜品/套餐详情 |
| 支付 | `pages/pay/index` | 支付页 |
| 下单成功 | `pages/success/index` | 成功页 |
| 网络异常 | `pages/nonet/index` | 无网络页 |
| 地址簿 | `pages/address/address` | 地址列表 |
| 备注 | `pages/remark/index` | 订单备注 |
| 个人中心 | `pages/my/my` | 用户信息 |
| 添加/编辑地址 | `pages/addOrEditAddress/...` | 地址表单 |
| 历史订单 | `pages/historyOrder/...` | 订单列表 |
| 订单详情 | `pages/orderDetail/...` | 单个订单详情 |

导航栏配置：标题"苍穹外卖"，橙色背景 `#E95F3C`，白色文字。

### 状态管理（Vuex 3.6.2）

Store 编译在 `vendor.js` 中，通过 `Vue.prototype.$store` 全局注入。主要 state：

| 字段 | 用途 |
|------|------|
| `token` | 认证 token |
| `sessionId` | 会话 ID |
| `baseUserInfo` | 微信用户信息（name、avatar） |
| `storeInfo` / `shopInfo` | 店铺信息 |
| `shopStatus` | 店铺营业状态 |
| `orderListData` | 购物车列表 |
| `orderData` | 当前订单数据 |
| `addressData` | 选中地址 |
| `dishTypeIndex` | 当前分类索引 |
| `remarkData` / `arrivals` | 备注和到店时间 |

全部使用同步 mutation（无 async action），页面通过 `mapState` / `mapMutations` 访问。

### API 层

- **Base URL**：`http://localhost:8080`（编译在 `utils/env.js` 中）
- **请求封装**：`utils/request.js`，对 `uni.request()` 做 Promise 包装
- **请求头**：`authentication: <token>`（从 Vuex store 读取）
- **成功判定**：`data.code == 200 || data.code === 1`
- **43 个 API 函数** 在 `pages/api/api.js`：下单、支付、地址、购物车、菜品、套餐、订单、用户登录、店铺状态等
- **图片下载**：`${baseUrl}/common/download?name=${image}`

### 登录流程

1. 检查 Vuex 中 token 是否为空
2. 弹窗提示"授权微信登录后才能点餐！"
3. `uni.login({ provider: 'weixin' })` 获取临时 code
4. `uni.getUserProfile({ desc: '登录' })` 获取微信用户信息
5. `POST /user/user/login` 发送 `{ code }`，后端返回 token
6. Token 存入 Vuex + `wx.setStorageSync('uni_id_token', token)`

### 组件

- **全局组件**（`components/`）：`uni-nav-bar`、`uni-icons`、`uni-status-bar`、`uni-popup`、`uni-phone`、`uni-piker`、`empty`、`reach-bottom`
- **页面级组件**（`pages/common/`）：`Navbar/navbar`（自定义导航栏）、`simple-address/simple-address`（地址展示）
- **uni_modules**（5 个包）：`uni-badge`、`uni-easyinput`、`uni-list`、`uni-popup`、`uni-transition`
- `node-modules/`（注意是连字符不是下划线）手动维护，只含 `@dcloudio/uni-ui/lib/` 下 `uni-easyinput` 和 `uni-list-item`

### 特有功能

- **AI 聊天**：首页有 AI 助手对话框，调用 `wx.request` 的 `enableChunked: true` + `onChunkReceived` 实现 SSE 流式接收
- **WebSocket**：连接 `wss://socket-canzg.itheima.net/ws`，使用 STOMP 协议做实时更新
- **支付**：配置了 `oauth: ['weixin']` 和 `payment: ['wxpay']`
- **微信基础库**：`libVersion: "3.16.1"`，ES6 转译关闭（bundle 已预编译）

### 开发注意事项

- **没有 `package.json`**，不能 `npm install`
- 修改逻辑需要去原始 UniApp 源项目，用 HBuilderX 重新编译
- `node-modules/`（连字符）已提交 Git，不要删除
- `static/` 中有 "ruiji" 品牌相关图片（`logo_ruiji.png` 等）
