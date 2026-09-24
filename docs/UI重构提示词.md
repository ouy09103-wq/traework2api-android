# 给 UI AI 的提示词（直接复制下面「---」之间的内容）

> 本文件是给用户复制用的。真正要交给 UI AI 的是下面那段 markdown。

---

# 任务：重写一个 Android App 的全部 UI（纯代码组装，无布局 XML）

## 项目
- 路径：`D:\myApp\traework2api-Android`
- 这是一个把 Trae（AI 编程助手）的账号额度，变成手机本地一个
  **OpenAI 兼容 API 网关**的安卓 App（本地端口 8788）。
- 包名 `com.example.traework2api`，Kotlin，`jvmTarget 17`，`minSdk 24`，`targetSdk 36`。

## 硬约束（违反则不可用，务必遵守）
1. **绝不能引入任何新依赖**。项目目前**只有 okhttp 4.12.0** 一个依赖。
   **没有 androidx、没有 Material Components、没有 Compose。**
   所以只能用平台 View（`LinearLayout`/`FrameLayout`/`TextView`/`Button`/`AlertDialog` 等），
   `Notification` 也只能用 `android.app.Notification.Builder`（不用 `NotificationCompat`）。
2. **绝不能使用布局 XML**。全部 UI 必须用 Kotlin 代码组装。
   `res/layout/` 目录现在是空的，请保持为空。
3. **不要改任何业务逻辑**。数据怎么来、怎么存、网络怎么发、状态怎么判断 —— 一行都别动。
   你只改「长什么样」和「怎么排版」。
4. 颜色、尺寸**必须走资源**（`res/values/colors.xml` + `res/values-night/colors.xml` 两套），
   **不要在代码里写字面量色值**，否则深色模式会坏。
5. 中文界面。

## 当前代码结构
- `app/src/main/java/com/example/traework2api/MainActivity.kt`（约 1000 行）
  - 一个 Activity，**底部导航 5 个 Tab**：概览 / 账号 / 模型 / 记录 / 更多
  - 每页一个函数：`overviewPage()` / `accountPage()` / `modelsPage()` /
    `recordsPage()` / `morePage()`
  - 另有一个**二级页** `keepAlivePage()`（保活设置），从「更多」页进入，
    顶栏显示「保活设置」，带返回
  - `companion object` 里是 design token（圆角/间距/字号常量）
  - 文件末尾是 **helper 组件库**（`card()` / `text()` / `caption()` / `chip()` /
    `row()` / `action()` / `scroll()` / `divider()` / `metric()` / `linkAction()` /
    `mono()` / `section()` / `labelValue()` 等）
  - `Page` enum 定义 5 个 Tab（标题/副标题/图标）
- `KeepAlive.kt`：保活体检数据与跳转（**不是 UI，别动它的逻辑**）
- 图标：`res/drawable/ic_*.xml`（6 个矢量图：account/dashboard/history/launcher/models/more）
- 配色：`res/values/colors.xml` + `values-night/colors.xml`，语义化命名
  （`wb_primary` / `wb_surface` / `wb_on_surface_variant` / `wb_error` / `wb_success` /
  `wb_warning` / `wb_outline_variant` 等）

## 设计要求
- **不要模仿 WorkBuddy / 任何现有 App 的视觉风格**。请重新设计一套。
- 风格自定（可以现代、克制、有品位），但必须：
  - 支持**浅色 + 深色**模式
  - 支持**手机竖屏 + 平板横屏**（内容要有最大宽度限制，别让文字横跨整个屏幕）
  - 触控目标不小于 **48dp**（无障碍）
  - 字号层级清晰（标题 / 卡片标题 / 正文 / 说明 / 数值 至少 5 档明确区分）
  - 状态要能一眼看出（服务运行中 / 已停止、各项保活是否就绪）

## 信息架构（保持不变）
5 个 Tab 的职责**不能改**，你可以改视觉、排版、配色、图标，但**别删页面、别改职责**：

| Tab | 内容 |
|---|---|
| **概览** | 服务状态 chip +（未运行时）「服务未启动」卡 /（运行时）「后台常驻中」卡 → 服务地址卡（可复制）→ 启停按钮 → 当前 Trae 账号卡 → 本地 API Key 卡（**默认打码**，可显示/复制）→ 使用引导 |
| **账号** | 当前账号详情 + 其它账号列表（切换/删除）+ 登录卡（**复制登录链接 / 粘贴回调完成登录**）+ 积分与用量（可用积分/额度上限/今日签到 + 刷新） |
| **模型** | 搜索框 + 模型列表（名称 + id + 复制） |
| **记录** | 按模型的用量汇总 + 调用记录列表（状态/模型/时间/用量明细） |
| **更多** | 关于（含版本号）/ 致谢 / 版权声明 / 账号区域说明 / **保活入口（进二级页）** |

**二级页「保活设置」**（从「更多」进入）：
总状态卡（`还有 N 项待开启` / `已就绪`）→ 一键配置卡 → 逐项清单（✓/⚠ + 说明 +
可一键的给「去开启」按钮）→ 本机 ROM 引导卡（含「复制这些路径」）。

## 交付要求
- 改完必须能通过：`.\gradlew.bat assembleDebug --offline`
- 在模拟器上**逐页截图验证**（浅色 + 深色各一轮），确认没有崩、没有错位
- 汇报：改了哪些文件、设计思路、以及**任何你没能验证的地方**（如实说，别写成"已验证"）
- 别碰 `KeepAlive.kt` / `Gateway.kt` / `GatewayService.kt` / `LocalFakeServer.kt` /
  `Trae*.kt` 的**逻辑**（样式相关的调用可以调整）

## 不要做的事
- 不要引入 androidx / Compose / 任何依赖
- 不要创建布局 XML
- 不要改业务逻辑（网络/存储/状态判断）
- 不要删除任何现有功能
- 不要写死颜色值
- **不要动 `app/src/main/java/.../MainActivity.kt` 之外的 Kotlin 文件的逻辑**
