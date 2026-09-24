# TraeWork2API-Android 完整需求与实施方案

> 文档用途：作为后续 AI Agent、开发者和测试人员的统一需求说明。
>
> 当前目标：独立开发 `traework2api-android`，将 TraeWork SOLO CN 的对话能力包装成运行在 Android 手机上的本地 OpenAI 兼容 API 服务。
>
> 文档状态：需求与实施方案已确认，尚未开始修改源代码。

---

## 1. 项目背景

### 1.1 最初设想

最初考虑将：

- `jilin0105/WorkBuddy2API-Android`
- 二次改造成可以反向代理多个平台的 Android 本地 API 网关
- 目标平台包括 Trae、ZCode 等

经过分析后，确认原 WorkBuddy Android 项目本质上是：

```text
OpenAI / Anthropic / Responses 客户端
        ↓
Android 本地 HTTP API 网关
        ↓
WorkBuddy / CodeBuddy 专用上游适配
        ↓
WorkBuddy 上游服务
```

原项目并不是简单的透明 HTTP 代理，核心业务大量绑定了 WorkBuddy 的：

- 后端地址
- 国内版/国际版区域
- 登录和 OAuth
- Access Token / Refresh Token
- 模型接口
- 额度查询
- 自动签到
- 多账号轮换
- WorkBuddy 特有请求头
- WorkBuddy 特有响应格式

因此，不适合直接把 Trae 逻辑堆进原项目。

### 1.2 最终确定的方向

最终决定不在原 `WorkBuddy2API-Android` 项目中添加 Trae 分支，而是单独创建一个新项目：

```text
traework2api-android
```

两个项目彼此独立：

```text
WorkBuddy2API-Android
    原项目，保持不变

traework2api-android
    新项目，TraeWork 专用
```

两个 APK 应满足：

- 可以同时安装
- 使用不同的 Application ID
- 使用不同的包名
- 使用不同的数据库
- 使用不同的 SharedPreferences
- 不互相覆盖配置
- 不互相迁移账号
- 不修改原 WorkBuddy 项目

---

## 2. 参考项目

### 2.1 Android 基础参考项目

```text
https://github.com/jilin0105/WorkBuddy2API-Android
```

它提供可以复用的 Android 通用能力：

- Kotlin Android 工程
- 原生 View UI
- Android 前台服务
- `ServerSocket` 本地 HTTP 服务
- OpenAI Chat 兼容接口
- Anthropic Messages 兼容接口
- OpenAI Responses 兼容接口
- API Key 管理
- SQLite 存储
- AndroidKeyStore / AES-GCM 加密
- 用量记录
- WakeLock
- 开机启动
- Android 定时任务
- 服务状态显示
- 悬浮球保活

### 2.2 TraeWork 协议参考项目

```text
https://github.com/Sliverkiss/traework2api
```

该项目目前实现了：

```text
TraeWork SOLO CN
        ↓
TraeWork 私有上游接口
        ↓
OpenAI /v1/chat/completions
```

已确认其功能包括：

- OpenAI 兼容 API
- `POST /v1/chat/completions`
- `GET /v1/models`
- 流式响应
- 非流式响应
- 多账号轮转
- 账号冷却
- 自动签到
- 积分查询
- Token 自动刷新
- 登录闭环
- 工具调用转换
- TraeWork SSE 转 OpenAI SSE

### 2.3 代码复用授权

已联系 `traework2api` 作者，作者已经允许进行二次修改和创作。

---

## 3. 最终产品目标

### 3.1 目标数据流

```text
Claude Code / Cherry Studio / OpenAI SDK / 其他客户端
        ↓
http://手机IP:端口/v1/chat/completions
        ↓
TraeWork2API-Android
        ↓
TraeWork SOLO CN 上游服务
```

### 3.2 第一版核心目标

第一版必须优先完成：

1. 独立 Android 工程
2. 独立 APK
3. TraeWork 账号登录或凭据导入
4. TraeWork Token 安全保存
5. `/healthz`
6. `/v1/models`
7. `/v1/chat/completions` 非流式
8. `/v1/chat/completions` 流式
9. OpenAI 请求转换为 TraeWork 请求
10. TraeWork SSE 转换为 OpenAI SSE
11. OpenAI 工具调用兼容
12. Token 过期刷新
13. 本地 API Key 鉴权
14. 基础日志脱敏
15. 真机可运行

### 3.3 第二阶段目标

第一版稳定后再实现：

- 多账号轮换
- 按积分或优先级选择账号
- 401/429/1005/5xx 冷却
- 连续失败禁用
- 自动签到
- 积分查询
- 用量记录
- 模型缓存
- 服务保活增强
- 更完整的 Android UI

### 3.4 暂不作为第一版目标的功能

暂时不要一开始加入：

- Anthropic `/v1/messages`
- OpenAI `/v1/responses`
- 多平台供应商抽象
- Z.ai Provider
- WorkBuddy Provider
- TraeWork 之外的平台
- 账号批量注册
- 代理转售
- 公网开放
- 绕过平台登录限制
- 绕过证书校验
- 绕过限流或套餐限制
- 复杂自动化登录
- 复杂悬浮窗功能

原因是第一版最重要的是先验证 TraeWork 单账号、普通聊天、流式响应和工具调用链路。

---

## 4. 非目标与边界

### 4.1 不修改原项目

绝对要求：

- 不修改 `WorkBuddy2API-Android`
- 不在原项目中增加 `TraeProvider`
- 不添加 `if (provider == "trae")` 一类分支
- 不删除原项目文件
- 不修改原项目包名
- 不复用原项目运行时数据库

可以复制原项目作为新工程起点，但复制后必须将两个工程彻底分离。

### 4.2 不将 TraeWork 当成稳定公共 API

TraeWork 当前适配使用的部分接口和请求头属于版本敏感的客户端协议，可能发生：

- 域名变化
- 路径变化
- 请求字段变化
- SSE 事件变化
- Token 结构变化
- 客户端版本变化
- 设备字段要求变化
- 模型配置变化
- 工具调用字段变化

项目文案应如实说明：

```text
本项目是针对 TraeWork 当前协议的个人学习和自用适配，
不保证长期兼容，也不代表 TraeWork 官方支持。
```

### 4.3 账号使用边界

只允许使用：

- 本人账号
- 已获得授权的账号
- 符合 TraeWork 服务条款的使用方式

不实现：

- 批量注册账号
- 盗用账号
- 分享他人 Token
- 规避账号限制
- 规避套餐限制
- 面向他人收费代理
- 大规模转售额度
- 绕过安全机制

---

## 5. 现有项目分析

### 5.1 WorkBuddy Android 项目中的通用能力

可以参考或移植：

- `ApiHostService.kt`
  - ServerSocket 监听
  - HTTP 请求读取
  - API Key 认证
  - HTTP 响应写回
  - SSE 输出
- `ProtocolAdapters.kt`
  - OpenAI 与 Anthropic 的部分转换逻辑
  - 工具调用转换思路
  - 流式响应转换思路
- `NativeStore.kt`
  - SQLiteOpenHelper
  - 数据库事务
  - 用量记录
  - AndroidKeyStore 加密
  - AES-GCM
- `BootReceiver.kt`
  - 开机启动
- `FloatingWindowKeeper.kt`
  - 可选的悬浮球保活
- `ApiHostService.kt` 的前台服务逻辑
  - Notification
  - WakeLock
  - 服务维护线程
  - AlarmManager 看门狗
- `MainActivity.kt`
  - 原生 View UI 组织方式
  - 服务状态显示
  - 设置界面

### 5.2 不能直接保留的 WorkBuddy 业务

新项目中应删除或重写：

- WorkBuddy/CodeBuddy 后端地址
- WorkBuddy 国内版/国际版区域判断
- WorkBuddy 账号 JSON 结构
- WorkBuddy OAuth 逻辑
- WorkBuddy Refresh Token 接口
- WorkBuddy Access Token 请求头
- WorkBuddy 模型请求格式
- WorkBuddy 额度查询
- WorkBuddy 自动签到
- WorkBuddy 账号优先级策略
- WorkBuddy `RegionConfig.kt`
- WorkBuddy `AccountSync.kt`
- WorkBuddy 专用 `NativeCore.kt`
- WorkBuddy 专用资产和文字
- WorkBuddy 专用模型列表

新工程要以 TraeWork 为中心重新组织命名和职责。

---

## 6. TraeWork 参考项目分析

### 6.1 目录职责对应关系

参考项目的结构：

```text
cmd/server/              HTTP 服务入口
cmd/signin/              批量签到工具
cmd/credit/              积分查询工具
internal/auth/           账号凭据解析和保存
internal/pool/           多账号池和冷却
internal/scheduler/      定时签到和 Token 预刷新
internal/server/         OpenAI API 路由
internal/upstream/       TraeWork 上游客户端和协议转换
```

Android 对应规划：

| Go 参考项目 | Android 新项目 |
|---|---|
| `internal/upstream/client.go` | `TraeClient.kt` |
| `internal/upstream/constants.go` | `TraeEndpoints.kt` |
| `internal/upstream/headers.go` | `TraeHeaders.kt` |
| `internal/upstream/payload.go` | `TraePayloadAdapter.kt` |
| `internal/upstream/solosse.go` | `TraeSseConverter.kt` |
| `internal/auth/auth.go` | `TraeAuth.kt` |
| `internal/pool/pool.go` | `TraeAccountPool.kt` |
| `internal/scheduler/scheduler.go` | `TraeScheduler.kt` |
| `internal/server/handler.go` | `TraeApiHostService.kt` |
| `cmd/server/config.go` | Android 设置和 `TraeStore.kt` |
| `state.json` | SQLite 账号状态字段 |
| `auths/trae-*.json` | 加密 SQLite 账号记录 |

### 6.2 TraeWork 上游接口

参考项目当前使用的接口包括：

```text
/api/agent/v3/llm_utils_chat
/api/ide/v1/get_detail_param
/cloudide/api/v3/trae/oauth/ExchangeToken
/cloudide/api/v3/trae/GetUserInfo
/trae/api/v2/ug/checkin_credits/status
/trae/api/v2/ug/checkin_credits/claim
/trae/api/v2/pay/ide_user_ent_usage
```

第一版重点关注：

```text
模型列表接口
聊天接口
ExchangeToken
GetUserInfo
Refresh Token
```

积分和签到接口放到后续阶段。

### 6.3 请求头

参考项目分别为 SOLO 请求、用户网关请求和 OAuth 请求构造请求头。

需要重点抽象：

- `Authorization`
- `Cloud-IDE-JWT`
- `X-Cloudide-Token`
- `X-Ide-Token`
- `X-Uid`
- `X-App-Id`
- `X-App-Version`
- `X-Ide-Version`
- `X-Ide-Version-Code`
- `X-Ide-Version-Type`
- `X-Device-Type`
- `X-OS-Version`
- `X-Device-Brand`
- `X-Machine-Id`
- `X-Device-Id`
- `Request-Traffic-Type`
- `User-Agent`

实现时要求：

- 将版本字段集中定义
- 不在多个类中散落硬编码
- 不把真实 Token 写入日志
- 不把用户 Cookie 当成固定常量
- 发生版本变化时可以集中更新

### 6.4 请求体转换

标准 OpenAI 请求：

```json
{
  "model": "glm-5.2",
  "messages": [
    {
      "role": "user",
      "content": "你好"
    }
  ],
  "stream": true,
  "tools": [],
  "tool_choice": "auto"
}
```

TraeWork 请求需要按照参考项目的行为转换，核心字段包括：

```json
{
  "messages": [],
  "function": "solo_work_lite",
  "stream": true,
  "config_name": "实际模型配置",
  "model": "实际模型名"
}
```

必须验证并实现：

- system 消息
- user 消息
- assistant 消息
- tool 消息
- 空 content
- 文本 content
- 多段 content
- `assistant.tool_calls`
- `tools`
- `tool_choice`
- 工具参数对象转字符串
- 模型别名转换
- 默认模型
- stream 强制或保持策略

参考项目的提交记录表明，TraeWork 的工具参数格式与 OpenAI 不完全一致，必须使用固定样例测试。

### 6.5 SSE 事件

参考项目处理的 TraeWork SSE 事件包括：

```text
metadata
timing_cost
output
extra_info
token_usage
done
error
```

转换目标是 OpenAI Chat Completions SSE：

```text
data: {"id":"...","object":"chat.completion.chunk","choices":[...]}

data: [DONE]
```

必须处理：

- 文本增量
- reasoning 内容
- 工具调用增量
- 多个工具调用
- 工具参数增量
- finish reason
- token usage
- 上游错误
- 空事件
- 非法事件
- SSE 不完整
- 上游提前断开
- 客户端提前断开

---

## 7. 推荐的新项目结构

建议结构如下：

```text
traework2api-android/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── README.md
├── LICENSE
├── NOTICE.md
├── app/
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   └── src/
│       ├── test/
│       │   └── java/com/example/traework2api/
│       │       ├── TraePayloadAdapterTest.kt
│       │       ├── TraeSseConverterTest.kt
│       │       ├── TraeAuthTest.kt
│       │       └── TraeAccountPoolTest.kt
│       └── main/
│           ├── AndroidManifest.xml
│           ├── assets/
│           │   └── trae-models.json
│           ├── java/com/example/traework2api/
│           │   ├── MainActivity.kt
│           │   ├── TraeApiHostService.kt
│           │   ├── TraeClient.kt
│           │   ├── TraeAuth.kt
│           │   ├── TraeHeaders.kt
│           │   ├── TraeEndpoints.kt
│           │   ├── TraePayloadAdapter.kt
│           │   ├── TraeSseConverter.kt
│           │   ├── TraeModels.kt
│           │   ├── TraeStore.kt
│           │   ├── TraeAccountPool.kt
│           │   ├── TraeScheduler.kt
│           │   ├── BootReceiver.kt
│           │   └── FloatingWindowKeeper.kt
│           └── res/
│               ├── layout/
│               ├── values/
│               ├── values-night/
│               ├── drawable/
│               └── xml/
```

文件名可以根据最终工程风格调整，但职责必须保持清晰。

---

## 8. 工程配置要求

### 8.1 独立身份

建议：

```text
项目名：traework2api-android
应用名：TraeWork2API Android
包名：使用自己的唯一包名
Application ID：使用自己的唯一 applicationId
```

例如：

```text
com.example.traework2api
```

正式发布前替换成自己的域名反写包名。

### 8.2 构建环境

沿用参考 Android 项目的工程要求：

- JDK 17
- Android SDK
- Kotlin
- Android Gradle Plugin
- minSdk 沿用基础项目
- OkHttp 可以沿用基础项目已经使用的版本

不因为 TraeWork 项目是 Go 就把 Go 运行时打包进 Android。

### 8.3 依赖原则

优先使用基础项目已有依赖：

- Kotlin 标准库
- Android SDK
- OkHttp
- 原有 JSON 库
- SQLite

不要因为 TraeWork 参考项目使用 Go，就将 Go 编译器、Go runtime 或 Docker 带进 APK。

---

## 9. 文件级需求

### 9.1 `settings.gradle.kts`

修改为独立项目名称：

```text
traework2api-android
```

### 9.2 `app/build.gradle.kts`

修改：

- namespace
- applicationId
- 应用版本
- 依赖
- 测试配置

确保与原 WorkBuddy APK 的 applicationId 不同。

### 9.3 `AndroidManifest.xml`

配置：

- 独立 Activity
- 独立前台 Service
- 独立 BootReceiver
- Internet 权限
- 前台服务权限
- 通知权限适配
- 必要的悬浮窗权限
- Android 版本兼容配置

不要引用原项目的包名组件。

### 9.4 `TraeApiHostService.kt`

负责：

- 启动和停止本地 HTTP 服务
- 监听端口
- 请求行解析
- Header 解析
- Body 读取
- 本地 API Key 鉴权
- 路由分发
- `/healthz`
- `/v1/models`
- `/v1/chat/completions`
- `/status`
- 非流式响应
- SSE 流式响应
- 客户端断开处理
- 上游连接释放
- 错误响应

第一版只要求 OpenAI Chat 路径完整可用。

### 9.5 `TraeClient.kt`

负责：

- 建立 TraeWork HTTP 请求
- 连接超时
- 读取超时
- 写入超时
- 获取模型
- 发起聊天
- Refresh Token
- 获取用户信息
- 解析 HTTP 错误
- 支持取消

不要在该类中混入 UI 逻辑。

### 9.6 `TraeAuth.kt`

负责：

- 生成或展示登录链接
- 接收回调 URL
- 解析回调参数
- ExchangeToken
- GetUserInfo
- Refresh Token
- 过期时间计算
- Token 刷新锁
- 账号删除

第一版推荐手动粘贴回调 URL，避免复杂的自动回调机制。

### 9.7 `TraeHeaders.kt`

负责：

- TraeWork 请求头构造
- OAuth 请求头构造
- 用户信息请求头构造
- 版本信息集中管理
- 设备信息集中管理

必须避免日志输出敏感 Header。

### 9.8 `TraeEndpoints.kt`

负责：

- Agent Host
- User Gateway Host
- OAuth Host
- Console Host
- 所有 API 路径
- 版本相关默认值

不要把 URL 分散在多个文件。

### 9.9 `TraePayloadAdapter.kt`

负责：

- OpenAI → TraeWork 请求转换
- Message 转换
- Tools 转换
- Tool Choice 转换
- Function Call 转换
- 参数序列化
- 模型名转换

必须可以脱离网络单元测试。

### 9.10 `TraeSseConverter.kt`

负责：

- TraeWork SSE 解析
- 事件聚合
- OpenAI SSE 输出
- 文本增量
- reasoning 增量
- 工具调用增量
- token usage
- finish reason
- `[DONE]`
- 错误事件

必须可以使用固定 SSE 样例单元测试。

### 9.11 `TraeModels.kt`

定义：

- 账号模型
- 模型模型
- API 请求模型
- SSE 事件模型
- 错误模型
- 工具调用状态模型
- Token 状态模型

避免所有逻辑都使用匿名 JSON Map。

### 9.12 `TraeStore.kt`

负责：

- SQLite 初始化
- 数据库版本
- Trae 账号
- 加密 Token
- 模型缓存
- 应用 API Key
- 用量日志
- 设置
- 冷却状态
- 数据清理

建议表：

```text
trae_accounts
trae_models
apps
usage_logs
settings
```

### 9.13 `TraeAccountPool.kt`

第一版可以只有单账号接口，但结构应允许后续加入：

- 多账号
- 优先级
- 积分排序
- 冷却
- 禁用
- 恢复
- 请求重试

### 9.14 `TraeScheduler.kt`

第一版只实现必要的 Token 预刷新。

后续再加入：

- 自动签到
- 积分刷新
- 定时模型刷新
- 账号健康检查

### 9.15 `MainActivity.kt`

UI 内容：

```text
首页
账号
模型
用量
设置
```

不要出现 WorkBuddy、CodeBuddy 或 Region 文字。

---

## 10. 数据库设计

### 10.1 `trae_accounts`

建议字段：

```text
id                 INTEGER PRIMARY KEY
uid                TEXT NOT NULL UNIQUE
nickname           TEXT NOT NULL DEFAULT ''
access_token_enc   TEXT NOT NULL
refresh_token_enc  TEXT NOT NULL
expires_at         INTEGER NOT NULL DEFAULT 0
domain             TEXT NOT NULL DEFAULT ''
api_host           TEXT NOT NULL DEFAULT ''
machine_id         TEXT NOT NULL DEFAULT ''
device_id          TEXT NOT NULL DEFAULT ''
enterprise_id      TEXT NOT NULL DEFAULT ''
enabled            INTEGER NOT NULL DEFAULT 1
priority           INTEGER NOT NULL DEFAULT 0
credits            REAL
failure_count      INTEGER NOT NULL DEFAULT 0
cooldown_until     INTEGER NOT NULL DEFAULT 0
created_at         INTEGER NOT NULL
updated_at         INTEGER NOT NULL
```

### 10.2 `trae_models`

建议字段：

```text
id                 INTEGER PRIMARY KEY
model_id           TEXT NOT NULL UNIQUE
display_name       TEXT NOT NULL DEFAULT ''
config_name        TEXT NOT NULL DEFAULT ''
context_window     INTEGER NOT NULL DEFAULT 0
max_tokens         INTEGER NOT NULL DEFAULT 0
enabled            INTEGER NOT NULL DEFAULT 1
fetched_at         INTEGER NOT NULL
```

### 10.3 `apps`

用于本地客户端 API Key：

```text
id                 INTEGER PRIMARY KEY
name               TEXT NOT NULL
key_hash           TEXT NOT NULL UNIQUE
key_prefix         TEXT NOT NULL
key_enc            TEXT NOT NULL
enabled            INTEGER NOT NULL DEFAULT 1
created_at         INTEGER NOT NULL
updated_at         INTEGER NOT NULL
```

客户端 API Key 与 TraeWork Token 必须完全分离。

### 10.4 `usage_logs`

建议记录：

```text
ts
model
protocol
account_uid
input_tokens
output_tokens
reasoning_tokens
total_tokens
latency_ms
status
error
input_content_truncated
output_content_truncated
```

不要记录：

- Authorization
- Cookie
- Access Token
- Refresh Token
- 完整请求头

---

## 11. 登录和 Token 流程

### 11.1 第一版手动回调流程

```text
Android 生成或展示登录 URL
        ↓
用户用浏览器打开
        ↓
用户完成 TraeWork 登录
        ↓
浏览器跳转到回调地址
        ↓
用户复制地址栏完整回调 URL
        ↓
粘贴回 Android 应用
        ↓
解析回调信息
        ↓
ExchangeToken
        ↓
GetUserInfo
        ↓
加密写入 SQLite
        ↓
显示账号状态
```

### 11.2 Token 刷新

要求：

- 请求前判断是否即将过期
- 使用互斥锁避免并发刷新
- 只允许一个刷新请求执行
- Refresh Token 轮换后立即保存
- 新 Access Token 使用 AndroidKeyStore 加密
- 刷新失败时返回清晰错误
- 401 时最多自动刷新并重试一次
- 刷新失败不能无限重试

### 11.3 凭据日志安全

日志只能显示：

```text
uid
nickname 脱敏值
Token 是否存在
Token 是否即将过期
```

禁止显示：

```text
完整 Access Token
完整 Refresh Token
Cookie
完整回调 URL
Authorization Header
```

---

## 12. 对外 API 需求

### 12.1 `GET /healthz`

示例响应：

```json
{
  "status": "ok",
  "running": true
}
```

不应泄露账号 Token。

### 12.2 `GET /status`

建议返回：

```json
{
  "running": true,
  "accounts": 1,
  "healthy_accounts": 1,
  "models": 32
}
```

只返回脱敏后的状态信息。

### 12.3 `GET /v1/models`

输出 OpenAI 兼容模型列表：

```json
{
  "object": "list",
  "data": [
    {
      "id": "glm-5.2",
      "object": "model",
      "owned_by": "trae-work"
    }
  ]
}
```

### 12.4 `POST /v1/chat/completions`

第一版必须支持：

- `model`
- `messages`
- `stream`
- `temperature`
- `max_tokens`
- `tools`
- `tool_choice`
- 合理的常用参数

不支持或无法映射的参数应：

- 明确忽略，或
- 返回标准错误

不能静默产生错误语义。

### 12.5 非流式响应

输出应兼容标准 OpenAI Chat Completions：

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 0,
    "completion_tokens": 0,
    "total_tokens": 0
  }
}
```

### 12.6 流式响应

必须输出：

```text
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

以标准 SSE 形式输出，并在结束时输出：

```text
data: [DONE]

```

---

## 13. 开发环境和软件建议

### 13.1 推荐电脑环境

推荐使用电脑完成主要开发：

- Windows、macOS 或 Linux
- Android Studio
- JDK 17
- Android SDK
- Git
- adb
- curl
- Python 3.11+ 或 Node.js 20+，用于协议原型和测试
- VS Code，可选
- Postman 或 Insomnia，可选
- Android Studio Logcat

### 13.2 手机环境

手机可以辅助：

- AiCode 阅读和修改源码
- 执行简单构建
- 查看日志
- 安装 APK
- 真机测试
- 使用 Termux 做 curl/Python 测试

手机不适合承担全部工作：

- Android Studio 不适合在手机上运行
- Gradle 编译慢
- Android SDK 占用大
- 大型 Kotlin 文件阅读和调试不方便
- 抓取桌面端请求较困难

推荐分工：

```text
电脑：协议验证、代码修改、编译、调试
手机：AI 辅助、安装 APK、真机运行测试
```

---

## 14. 推荐大模型

### 14.1 推荐顺序

1. Claude Opus 最新版本
2. GPT-5-Codex 或同级最新 Coding 模型
3. Claude Sonnet 最新版本
4. GLM-5.3 Coding 或同级 Coding 模型

### 14.2 最佳组合

```text
架构分析和协议移植：Claude Opus 最新版
多文件编码和 Gradle 修复：GPT-5-Codex 最新版
日常小修改和测试补充：Claude Sonnet 最新版
低成本简单迭代：GLM-5.3 Coding
```

### 14.3 只能选一个时

优先选择：

- 当前可用的 Claude Opus，或
- 当前可用的 GPT-5-Codex

选择标准：

- 上下文窗口大
- 支持工程工具调用
- 能读取多个文件
- 能运行构建命令
- 能根据真实编译错误修复
- 不会在未验证时声称成功

### 14.4 AI Agent 的使用原则

不要让 AI 一次性完成整个项目。应当：

1. 先只读分析
2. 再建立独立工程
3. 再移植请求体
4. 再移植 SSE
5. 再移植认证
6. 再加入数据库
7. 再加入 UI
8. 最后加入多账号和调度

每个阶段都要构建和测试。

---

## 15. AI Agent 分阶段提示词

### 15.1 阶段 0：只读分析

```text
请只读分析以下两个项目，不要修改任何文件：

1. WorkBuddy2API-Android
2. traework2api

目标是独立开发 traework2api-android，
不是在 WorkBuddy2API-Android 中增加 Trae 分支。

请输出：
1. WorkBuddy Android 中可以复用的通用 Android 能力
2. 必须删除的 WorkBuddy 专用逻辑
3. traework2api 中的 Trae 请求链路
4. Trae 登录和 Token 刷新链路
5. OpenAI 请求到 Trae 请求的字段映射
6. Trae SSE 到 OpenAI SSE 的字段映射
7. Android 新项目的文件级规划
8. 数据库设计
9. 测试计划

不要修改文件。
不要读取、输出或保存真实 Token、Cookie、API Key。
```

### 15.2 阶段 1：建立独立工程

```text
请创建独立的 traework2api-android Android 工程。

要求：
1. 不修改 WorkBuddy2API-Android。
2. 使用独立包名和 applicationId。
3. 使用独立数据库名称。
4. 保留必要的 Android 前台服务和本地 HTTP 网关能力。
5. 删除 WorkBuddy 相关文字、区域和业务入口。
6. 暂不接入 Trae 上游。
7. 完成后运行 Gradle 构建。
8. 如果构建失败，必须报告真实错误。
```

### 15.3 阶段 2：移植基础网关

```text
请实现 TraeWork2API-Android 的基础本地 HTTP 网关。

要求：
1. 实现 /healthz。
2. 实现本地 API Key 鉴权。
3. 保留 ServerSocket 服务。
4. 保留前台服务启动和停止。
5. 支持请求体大小限制。
6. 支持客户端断开。
7. 不接入真实 Trae Token。
8. 使用 Mock 上游进行测试。
9. 不打印敏感请求头。
10. 修改后运行单元测试和 assembleDebug。
```

### 15.4 阶段 3：移植请求转换

```text
请实现 TraePayloadAdapter.kt。

要求：
1. 将 OpenAI Chat Completions 请求转换成 TraeWork SOLO 请求。
2. 支持 model、messages、stream。
3. 支持 system、user、assistant、tool 消息。
4. 支持 assistant.tool_calls。
5. 支持 tools 和 tool_choice。
6. 按 TraeWork 实际要求处理工具参数字符串化。
7. 支持模型别名。
8. 不接入真实 Token。
9. 使用脱敏固定 JSON 样例编写单元测试。
10. 修改后运行测试和构建。
```

### 15.5 阶段 4：移植 SSE 转换

```text
请实现 TraeSseConverter.kt。

输入是 TraeWork SSE，事件包括：
metadata、timing_cost、output、extra_info、token_usage、done、error。

输出是 OpenAI Chat Completions SSE。

要求：
1. 支持文本增量。
2. 支持 reasoning 内容。
3. 支持 token usage。
4. 支持 finish reason。
5. 支持工具调用增量。
6. 支持多个工具调用。
7. 支持上游错误。
8. 支持不完整 SSE。
9. 支持 [DONE]。
10. 使用本地固定 SSE 样例测试。
11. 不使用真实账号信息。
```

### 15.6 阶段 5：移植单账号认证

```text
请实现 TraeAuth.kt 和 TraeStore.kt 的单账号版本。

要求：
1. 支持登录回调 URL 手动粘贴。
2. 支持 ExchangeToken。
3. 支持 GetUserInfo。
4. 支持保存 accessToken、refreshToken 和 expiresAt。
5. 使用 AndroidKeyStore 加密 Token。
6. 数据库不保存明文 Token。
7. Token 刷新使用互斥锁。
8. 401 时最多刷新并重试一次。
9. 日志不得出现 Token、Cookie 和完整回调 URL。
10. 使用脱敏样例测试解析逻辑。
11. 运行单元测试、构建和真机测试。
```

### 15.7 阶段 6：接入真实单账号

```text
请将单账号 TraeWork Provider 接入本地 API。

只验证：
1. /healthz
2. /v1/models
3. /v1/chat/completions 非流式
4. /v1/chat/completions 流式

要求：
1. 使用我本人账号。
2. 不要求我把 Token 发给你。
3. 凭据只从本地 UI 或本地配置读取。
4. 日志全部脱敏。
5. 失败时区分认证、模型、请求体、SSE 和网络错误。
6. 不要声称未验证的功能已经完成。
```

### 15.8 阶段 7：工具调用

```text
请实现并测试 TraeWork 工具调用兼容。

覆盖：
1. OpenAI tools 转 TraeWork tools。
2. tool_choice 转换。
3. assistant.tool_calls 转换。
4. TraeWork function_call 转 OpenAI tool_calls。
5. 流式工具调用参数增量。
6. 多个并行工具调用。
7. tool 结果回传。
8. 工具调用错误。

使用 Mock SSE 和固定 JSON 测试。
真实账号只做最终手动验证。
```

### 15.9 阶段 8：多账号和调度

```text
请在单账号稳定的基础上实现多账号池。

要求：
1. 按积分或优先级选择账号。
2. 处理 401、429、1005、5xx。
3. 账号冷却。
4. 连续失败禁用。
5. 成功恢复状态。
6. 失败后避免重复选择同一账号。
7. Token 刷新和账号状态写入 SQLite。
8. 不打印凭据。
9. 添加单元测试。
10. 再考虑自动签到和积分刷新。
```

### 15.10 阶段 9：最终测试

```text
请只检查本次 TraeWork2API-Android 改动。

运行：
1. 单元测试
2. ./gradlew assembleDebug
3. 静态检查或 lint

验证：
1. APK 独立安装
2. 可以和 WorkBuddy APK 共存
3. /healthz
4. /v1/models
5. 非流式聊天
6. 流式聊天
7. 工具调用
8. Token 刷新
9. 错误 API Key
10. 模型不存在
11. 401
12. 429
13. 1005
14. 500
15. 上游超时
16. 客户端断开
17. 手机锁屏
18. Android 进程恢复
19. 日志中无敏感凭据

如有失败，请报告失败原因和关键日志，不要虚报成功。
```

---

## 16. 测试计划

### 16.1 单元测试

请求转换：

- 基本 user 消息
- system 消息
- assistant 消息
- tool 消息
- 空 content
- 多段 content
- 模型别名
- 默认模型
- tools
- tool_choice
- tool 参数 JSON 字符串化
- assistant tool_calls

SSE：

- metadata
- output
- reasoning
- token_usage
- done
- error
- 空 data
- 非法 JSON
- 事件跨行
- 连接提前结束
- 多个工具调用
- 工具参数增量

认证：

- 嵌套账号 JSON
- 扁平账号 JSON
- 缺少 accessToken
- 缺少 refreshToken
- 过期时间转换
- 刷新结果解析
- Token 密文存储

账号池：

- 健康账号选择
- 冷却账号跳过
- 禁用账号跳过
- 429 冷却
- 1005 冷却
- 401 刷新
- 连续失败禁用
- 成功恢复

### 16.2 Mock 上游测试

必须使用 Mock HTTP 服务覆盖：

- 正常 JSON
- 正常 SSE
- 401
- 429
- 1005
- 404
- 500
- 502
- 超时
- 空响应
- 非法 JSON
- 非法 SSE
- 连接提前断开

### 16.3 真机测试

使用本人账号手动验证：

```bash
curl http://手机IP:端口/healthz

curl http://手机IP:端口/v1/models \
  -H "Authorization: Bearer 本地APIKey"

curl http://手机IP:端口/v1/chat/completions \
  -H "Authorization: Bearer 本地APIKey" \
  -H "Content-Type: application/json" \
  -d '{"model":"glm-5.2","messages":[{"role":"user","content":"你好"}],"stream":false}'
```

还需测试：

- 流式输出
- 工具调用
- Token 过期
- Token 自动刷新
- 错误 API Key
- 错误模型
- 手机锁屏
- 切换 Wi-Fi
- 局域网其他设备访问
- 服务停止后端口释放
- 客户端主动取消
- Android 进程被回收后的恢复行为

---

## 17. 安全要求

### 17.1 不发送给 AI 的内容

绝对不要把以下内容发送给 AI Agent：

```text
Access Token
Refresh Token
Cookie
API Key
完整 Authorization Header
完整回调 URL
账号导出 JSON
包含手机号或邮箱的完整日志
```

脱敏格式：

```text
<ACCESS_TOKEN>
<REFRESH_TOKEN>
<COOKIE>
<API_KEY>
<USER_ID>
```

### 17.2 不提交到 Git 的内容

```text
真实账号文件
真实 Token
真实 Cookie
真实 API Key
数据库文件
本地配置文件
完整抓包文件
```

### 17.3 日志要求

允许记录：

- 请求开始和结束
- 模型名
- UID
- HTTP 状态码
- 延迟
- 错误类型
- 脱敏账号昵称

禁止记录：

- 完整请求 Header
- Authorization
- Cookie
- Access Token
- Refresh Token
- 完整请求体中的敏感字段

### 17.4 网络暴露

默认建议：

- 只监听本机地址，或
- 用户主动开启局域网访问

如果开启局域网访问：

- 必须启用 API Key
- 不要把端口映射到公网
- 不要直接开放手机公网端口
- 不要在日志中显示完整凭据
- 不要允许空 API Key

---

## 18. 兼容性和风险

### 18.1 TraeWork 协议风险

风险：

- 上游接口随客户端版本变化
- 请求头字段变化
- 模型配置变化
- SSE 事件变化
- 工具调用变化
- 登录流程变化
- Token 刷新规则变化

应对：

- 集中管理版本常量
- 保存脱敏协议样例
- 为请求和 SSE 编写单元测试
- 将上游适配独立成模块
- 不让上游细节污染公共 API 网关

### 18.2 Android 后台风险

风险：

- 国产 ROM 清理后台
- Android 通知权限
- 前台服务限制
- 电池优化
- 进程被回收
- 网络切换
- 手机锁屏

应对：

- 使用前台服务
- 使用通知
- 用户引导关闭电池优化
- 维护服务状态
- 必要时使用 WakeLock
- 只在验证服务本体后加入复杂保活

### 18.3 凭据存储风险

风险：

- Token 明文写数据库
- 日志泄露
- 备份泄露
- 导出文件泄露
- 截图泄露

应对：

- AndroidKeyStore
- AES-GCM
- 数据库存密文
- 不记录完整 Token
- 不将数据库加入 Git
- 提供删除账号功能

### 18.4 代码复用风险

现在已有作者授权，但仍应：

- 保留授权证据
- 保留作者署名
- 保留参考项目地址
- 记录二改范围
- 不宣称官方支持
- 不移除其他依赖的许可证

---

## 19. 实施阶段总览

### 阶段 0：需求和代码审查

目标：

- 确认独立项目方向
- 阅读两个参考项目
- 建立文件映射
- 确认作者授权
- 确定不修改 WorkBuddy 原项目

产出：

- 本需求文档
- 文件级实施计划
- 风险清单

验收：

- 需求明确
- 新项目边界明确
- 没有将两个平台混在一起

### 阶段 1：独立工程

目标：

- 创建新目录
- 修改包名
- 修改 applicationId
- 修改数据库名
- 修改应用名
- 独立编译
- 独立安装

验收：

```text
./gradlew assembleDebug
```

新 APK 可与 WorkBuddy APK 共存。

### 阶段 2：移除 WorkBuddy 业务

目标：

- 删除 WorkBuddy 后端
- 删除 WorkBuddy 区域
- 删除 WorkBuddy OAuth
- 删除 WorkBuddy 账号模型
- 保留通用 Android 网关

验收：

- 构建成功
- UI 不再出现 WorkBuddy
- 数据库是 Trae 专用结构

### 阶段 3：基础 HTTP 网关

目标：

- `/healthz`
- API Key
- `/status`
- 前台服务
- ServerSocket

验收：

- curl 成功
- 错误 API Key 被拒绝
- 服务停止后端口释放

### 阶段 4：Trae 请求转换

目标：

- OpenAI → TraeWork
- Message 转换
- Tools 转换
- Model 转换

验收：

- Mock 请求样例全部通过

### 阶段 5：Trae SSE 转换

目标：

- TraeWork SSE → OpenAI SSE
- 文本
- reasoning
- 工具调用
- usage
- 错误

验收：

- Mock SSE 全部通过

### 阶段 6：单账号认证

目标：

- 手动回调登录
- ExchangeToken
- GetUserInfo
- 加密存储
- Refresh Token

验收：

- 本人账号登录成功
- Token 不出现在日志
- 过期后可以刷新

### 阶段 7：单账号真实聊天

目标：

- `/v1/models`
- 非流式聊天
- 流式聊天

验收：

- curl 成功
- Cherry Studio 或其他 OpenAI 客户端可调用

### 阶段 8：工具调用

目标：

- function_call
- tool_calls
- 参数增量
- tool 结果回传

验收：

- 至少一个真实工具调用链路成功
- Mock 多工具调用通过

### 阶段 9：多账号和调度

目标：

- 多账号
- 积分排序
- 冷却
- 自动刷新
- 自动签到

验收：

- 一个账号失败后可以切换
- 账号状态正确保存
- 不发生无限重试

### 阶段 10：UI、保活和发布准备

目标：

- 账号页
- 模型页
- 设置页
- 用量页
- 前台服务
- 可选保活
- 许可证和致谢
- README
- APK 构建

验收：

- 真机稳定运行
- 日志脱敏
- 与原 WorkBuddy APK 共存
- 所有测试通过

---

## 20. 最终验收标准

项目完成必须满足：

1. 项目名称为独立的 `traework2api-android`。
2. 原 `WorkBuddy2API-Android` 未被修改。
3. 两个 APK 使用不同 Application ID。
4. 两个 APK 可以同时安装。
5. 两个项目使用不同数据库和配置目录。
6. 新项目不包含无用的 WorkBuddy 业务。
7. TraeWork 单账号登录可用。
8. Token 使用 AndroidKeyStore 加密保存。
9. `/healthz` 可用。
10. `/v1/models` 可用。
11. `/v1/chat/completions` 非流式可用。
12. `/v1/chat/completions` 流式可用。
13. 工具调用可以正确转换。
14. Token 过期可以刷新。
15. 401、429、1005、5xx 有明确处理。
16. 本地 API Key 与上游 Token 分离。
17. 日志、源码和 Git 中没有真实凭据。
18. Mock 单元测试通过。
19. Gradle 构建成功。
20. 真机测试通过。
21. 作者授权和参考项目得到明确致谢。
22. 项目文案不宣称 TraeWork 官方支持。

---

## 21. 推荐的最小可行版本

如果开发资源有限，第一版只做以下内容：

```text
独立 Android 工程
        ↓
单账号手动登录
        ↓
加密保存 Token
        ↓
/v1/models
        ↓
/v1/chat/completions 非流式
        ↓
/v1/chat/completions 流式
        ↓
基础工具调用
        ↓
API Key 鉴权
```

暂时不做：

```text
多账号
自动签到
积分面板
Anthropic
Responses
复杂保活
公网部署
```

这样可以最快确认核心链路可行，之后再逐步增加功能。

---

## 22. 交给 AI Agent 时的总要求

可以将以下要求放在每个任务的开头：

```text
这是一个独立的 traework2api-android 项目。

不要修改 WorkBuddy2API-Android 原项目。
不要把 WorkBuddy 和 TraeWork 逻辑混在一起。
请基于真实读取到的代码工作，不要猜测文件结构。
修改前先读取相关文件和项目规则。
只实现当前任务，不顺手重构无关代码。
不要把真实 Token、Cookie、API Key 写入代码、日志、测试或文档。
不要读取或输出我未脱敏的凭据。
所有完成结论都必须基于实际构建、测试或真机验证。
如果失败，必须如实报告失败原因。
```

---

## 23. 当前结论

最终项目方案确定为：

```text
WorkBuddy2API-Android
    保持原样，不修改

traework2api-android
    独立工程
    TraeWork 专用
    Android 本地 OpenAI API 网关
    参考并获得 traework2api 作者授权
```

推荐执行顺序：

```text
先建立独立工程
    ↓
移除 WorkBuddy 业务
    ↓
复用 Android 网关能力
    ↓
移植 Trae 请求体
    ↓
移植 Trae SSE
    ↓
实现单账号登录
    ↓
真实单账号聊天
    ↓
工具调用
    ↓
多账号和调度
    ↓
UI、保活和发布
```

核心原则：

```text
先单账号，后多账号；
先 Chat，后 Anthropic/Responses；
先 Mock 测试，后真实账号；
先协议跑通，后 UI 和保活；
先保证独立和安全，后增加功能。
```
