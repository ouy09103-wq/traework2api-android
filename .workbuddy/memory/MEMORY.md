# 项目长期记忆（traework2api-Android）

## 项目性质

Trae 本地 OpenAI 兼容 API 网关的安卓 App（本地 8788 端口）。Kotlin、minSdk 24、
**依赖只有 okhttp 4.12.0**（无 androidx / Material / Compose）。用户是纯小白、中文界面。

## 硬约束（改任何 UI 前先看这里）

1. **零新依赖、零布局 XML**：`res/layout/` 不存在且必须保持不存在，
   全部 UI 用 Kotlin 在 `MainActivity.kt` 里组装（helper + `card {}` / `row {}` 这类 DSL）。
2. **颜色只写语义名，色值只在 `AppStyle.kt`**：界面写 `color("primary")`。
   ⚠️ **绝不可以用 `getColor(R.color.wb_*)` 取色** —— 资源 id 是编译期常量，
   只能区分 values / values-night，**换不了风格**（`res/values/colors.xml` 现在只代表
   默认风格「靛蓝」，供启动主题与 6 个矢量图标准）。
   取不到色时回退 `on_surface`（返回 0 = 全透明 = 文字看不见）。
3. **圆角/间距只用 token**（`RADIUS_*` / `GAP` / `PAGE_PAD_*` / `CARD_PAD` / `CONTENT_MAX_DP`），
   它们随风格变，页面里不要手写数字；字号用 `FS_*`。
4. **不动业务逻辑**：`Gateway` / `GatewayService` / `KeepAlive` / `LocalFakeServer` /
   `TraeUpstream` / `TraeIntlUpstream` / `TraeAuthApi` / `TraeAuthStore` / `TraeRegion` /
   `ModelsRepo` / `UsageStore` 的实现一律不改（`KeepAlive.check/rom/romGuide` 只读其返回值排版）。
5. **不要删功能/页面**：5 Tab + 2 个二级页（保活设置 / 外观风格）。
   这几个细节极易被"重做"时误删：API Key 默认打码（`MASKED_KEY`）、服务默认关闭的空态说明、
   国际服「未开发」标注 + 两个确认弹窗、`chip()` 里的 `lp(-2,-2)`（修 chip 被拉成横幅的 bug）、
   底栏点击监听挂**外层**容器（不是内层 TextView）。

## 多风格架构（2026-09-24 建立）

- `AppStyle.kt`：`STYLES`（靛蓝/石墨/苔绿/落日）× `Mode`（跟随系统/浅色/深色）。
  每套风格含圆角/间距形态 + 亮暗两份 19 键色表。
- 取色 `AppStyle.color(name)`；亮暗通过 `attachBaseContext` + `createConfigurationContext`
  覆写 uiMode 落地（所以平台对话框、光标也跟着变）。
- **切风格/亮暗一律 `recreate()`**（颜色/圆角/间距散在几十个控件上，逐个回刷会漏），
  `onSaveInstanceState` 保存当前 Tab 与二级页，重建后能回原处。
- **"刷新界面"要用 `redraw()`，不要用 `render(currentPage)`** —— 后者会把停在二级页的用户
  踢回 Tab 页（实测过）。
- 新增/换风格必须过 `AppStyle.selfCheck()`（色表齐全 + 不透明 + 对比度 ≥4.5:1），
  它在 `MainActivity.onCreate` 开头调用，不达标直接崩是**预期行为**。
- 风格自检共 8 个 `selfCheck`（原 7 个 + AppStyle）。

## 视觉语言（2026-09-24 第二轮深化）

- **卡片三层次**：`card(tone = CardTone.NORMAL / SOFT / EMPHASIS)`。
  `SOFT`(surface_container) 用于说明与空态；`EMPHASIS`(primary_container) 用于本页最该看的那块
  （概览的服务地址卡）。**有底色的面板不加阴影**（elevation=0），靠颜色分层。
- **状态一律「圆点 + 文字」**：`statusChip(value, tone, on)`，实心=成立、空心=未成立。
  不要只靠颜色表达状态（色盲 + 任务书 §5.2 要求）。
- **顶栏 / 底栏底色都是 `background`**（与页面同色），各自保留 1dp 分隔线；
  "凸起"的只有卡片。改成 `surface` 会让整屏一样平。
- **图标语言**：5 个导航图标 = 24dp 视口 + 1.7dp 描边 + 圆头圆角，色值是占位、靠 `setTint` 刷。
  新增图标请沿用这套参数。
- **桌面图标 = 自适应图标**（用户 2026-09-24 拍板）：`mipmap-anydpi-v26/ic_launcher.xml`
  （background + foreground + monochrome）+ 背景 `drawable/ic_launcher_background.xml`（满幅靛蓝）
  + 前景 `drawable/ic_launcher_foreground.xml`（白色 ⇄）+ API 24–25 回落 `mipmap-anydpi/ic_launcher.xml`。
  ⚠️ 前景图形要按**中间 72×72 安全区**定尺寸（现约 54/108），别照抄满幅方形的比例。
  ⚠️ 桌面图标**不跟 App 内风格切**（要 activity-alias 黑魔法），固定靛蓝当招牌色。
- 卡底本身就是 `primary_container` 时，里面的 `linkAction` 要传 `overContainer = true`
  （底改 surface），否则和卡同色看不见。
- 新增/修改底色的卡，要同步往 `AppStyle.CONTRAST_PAIRS` 里加"文字 vs 该底色"的组合。
- **弹窗一律用 `confirmDialog()`**，不要用 `AlertDialog.Builder(this).setTitle(...)`：
  平台弹窗的配色来自 `colorAccent`（编译期资源），**多风格下必然脱节**。
  `confirmDialog` 自绘标题/正文/按钮 + 把 window 背景换成当前风格的 `surface` + 圆角。
- **二级页顶栏必须有「← 返回」**（`backAction`，放标题上方）：不能只靠系统返回键。
- **破坏性操作**（删除）用 `action(..., danger = true)`（error 色描边），不要用普通实心主色键。
- **空态要补**：搜索/过滤类页面在"无结果"时必须给一张 `SOFT` 卡说明，
  否则用户看到的是整页空白，会以为坏了。
- **refresh 用"原地改控件"，不要重建页面**（`redraw()`/`render()` 都会 `removeAllViews()`
  → **滚动位置弹回页首**，用户看不到自己刚点的那张卡）。做法：构建时把"怎么刷数据"
  登记成句柄（如 `quotaApply`），刷新前后各调一次。
  ⚠️ 试过"记住 scrollY 再滚回去"，**不可靠**（新 ScrollView 未量完高度，`scrollTo` 会被夹回 0）。
- 需要"原地更新"的值控件，用 `metricOf(label, valueView)`（值控件由调用方持有）。

## 验证方法（模拟器）

设备：MuMu、`adb -s 127.0.0.1:5555`（同机还有 emulator-5554，不指定会报多设备）。
构建：`./gradlew.bat assembleDebug --offline`。

- 崩溃：`adb logcat -b crash -d` 应为空。
- 结构：`uiautomator dump` + 读文案。
- 截图：`screencap -p` + `pull`（**不要** `exec-out >`）。
- ⚠️ **判断颜色别靠肉眼**：缩略图分不出 `#F7F7FC` 与 `#FFFFFF`，我因此误判"改动没生效"。
  写个纯 Python PNG 取样器（zlib 解 IDAT + 反过滤）直接读像素，一次确认到位。
- ⚠️ 模拟器锁死横屏：`wm size` / `user_rotation` 都转不了真竖屏；
  用 `wm size 2340x1080` + `wm density 420` 得到 411dp 宽的 app 窗口代替，测完 reset。
- ⚠️ 点控件别用固定坐标（坐标随风格变，差上百 px）；用 dump 出来的 bounds 动态定位。
- ⚠️ `recreate()` 后 ≥3s 再点，否则输入会被丢。
- ⚠️ Git Bash 下 adb 加 `MSYS_NO_PATHCONV=1`，但 `install`/`pull` 的文件路径要写 `D:/...`
  （写 `/d/...` 会静默失败，白跑一轮）。
- ⚠️ 点弹窗按钮要 **x、y 都从 bounds 算**：两键等宽并排、中间隔 8dp、弹窗居中，
  固定 x 会点在间隙上（表现是"点了没反应"）。
- ⚠️ 二级页 dump 的第一个 `text` 是「← 返回」，不是页标题；文案断言用部分匹配。
