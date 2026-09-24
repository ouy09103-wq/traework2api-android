# UI 全面重做 —— 任务书（给 UI AI）

> 本文件是**需求 + 提示词**。第 1 部分是给用户看的说明，第 2 部分起的「---」之间内容
> **整段复制给 UI AI** 即可。
>
> 上一版提示词（`docs\UI重构提示词.md`）对应的是**已完成**的那轮视觉重构（§5.13），
> 保留作历史。本轮是**全面重做**：不再模仿 WorkBuddy，重新设计一套视觉语言。

---

## 1. 给用户的说明（不要复制给 AI）

**这次和上次的区别**：上次是"在 WorkBuddy 风格上修 11 条不一致"；这次是**换一套自己的设计语言**。
信息架构（5 个 Tab 的职责、二级页）**保持不变**，重做的是视觉与排版。

**为什么不用重写业务**：所有 UI 都在 `MainActivity.kt` 一个文件里，用 helper 函数拼装，
业务逻辑（网络/存储/状态）在别的文件里，UI AI 碰不到也不该碰。

**风险与边界**（都写进提示词了）：
- 项目**只有 okhttp 一个依赖**，**没有 androidx / Material / Compose** → 只能用平台原生 View。
- **不能用布局 XML**（`res/layout/` 目录不存在），必须纯 Kotlin 代码组装。
- 颜色必须走 `values` / `values-night` 资源，否则深色模式坏掉。

**验收方式**：UI AI 必须自己跑 `assembleDebug --offline` + 装模拟器逐页截图（浅色/深色），
并如实报告没验证到的部分。

---

# 任务：全面重做这个 Android App 的 UI（重新设计视觉语言）

## 一、项目背景

- 路径：`D:\myApp\traework2api-Android`
- 这是一个 **Trae 本地 OpenAI 兼容 API 网关**的安卓 App：在手机本地端口（8788）
  开一个 HTTP 服务，把 OpenAI 格式请求转发到 Trae，供 Chatbox / Cursor / Claude Code 等客户端使用。
- 包名 `com.example.traework2api`，Kotlin，`minSdk 24`，`targetSdk 36`，`jvmTarget 17`。
- 用户是**纯小白、中文界面**。界面的首要任务是**让他看懂「服务开没开、账号通不通、下一步点哪」**。

## 二、硬约束（违反即不可用，务必遵守）

1. **绝不能引入任何新依赖**。`app/build.gradle.kts` 里目前**只有**
   `com.squareup.okhttp3:okhttp:4.12.0`。**没有 androidx、没有 Material Components、没有 Compose。**
   所以只能用平台 View：`LinearLayout` / `FrameLayout` / `ScrollView` / `TextView` / `Button` /
   `EditText` / `AlertDialog` / `Toast` / `GradientDrawable`。
   `Notification` 也只能用 `android.app.Notification.Builder`（**不用** `NotificationCompat`）。
   ⚠️ `Button` 是**平台 Button**（不是 MaterialButton），样式能力有限；`AlertDialog` 也是平台的。
2. **绝不能创建布局 XML**。`res/layout/` 目录**当前不存在**，请保持不存在。
   全部 UI 必须用 Kotlin 代码组装。
3. **颜色必须走资源**：`res/values/colors.xml` + `res/values-night/colors.xml` 两套。
   **不要在 Kotlin 里写字面量色值**（`Color.parseColor("#...")` / `0xFF...` 都不行），
   否则深色模式必坏。尺寸/字号走 `MainActivity.kt` 的 `companion object` design token。
4. **不要改业务逻辑**。数据怎么来、怎么存、网络怎么发、状态怎么判断 —— **一行都别动**。
   你只改「长什么样」和「怎么排版」。
5. **不要删功能、不要删页面**。5 个 Tab 的职责与二级页结构见第四节，**保持不变**。
6. 中文界面。

## 三、当前代码结构（先读这些，别通读全工程）

- `app/src/main/java/com/example/traework2api/MainActivity.kt`（约 1030 行，**你要改的就是它**）
  - `companion object`（第 40 行起）= **design token**：`RADIUS_*` / `GAP` / `PAGE_PAD_*` /
    `CARD_PAD` / `CONTENT_MAX_DP` / `FS_*`（字号）/ `MASKED_KEY`
  - `Page` enum（第 79 行）= 5 个 Tab（标题/副标题/图标资源）
  - `onCreate`（第 112 行）= 手搭「顶部栏 + 内容区 + 底部导航」；**开头有 7 个启动自检**
    （第 116–134 行，`LocalFakeServer` / `TraeRegion` / `TraeAuthStore` / `TraeAuthApi` /
    `TraeIntlUpstream` / `Gateway` / `KeepAlive`），失败会直接崩溃（**这是预期行为，别删别改**）
  - `render(page)`（第 208 行）= 切 Tab
  - 五个页面函数：`overviewPage()` / `accountPage()` / `modelsPage()` / `recordsPage()` / `morePage()`
  - `keepAlivePage()`（第 564 行）= **二级页**（保活设置），从「更多」进入
  - 文件末尾（第 889 行起）= **helper 组件库**（`scroll` / `card` / `row` / `text` / `mono` /
    `caption` / `section` / `divider` / `chip`(+4 语义封装) / `labelValue` / `metric` /
    `searchInput` / `linkAction` / `action` / `shape` / `color` / `dp` / `lp` / `top` / `weighted`）
- **别的 Kotlin 文件一律不要改逻辑**：`Gateway.kt` / `GatewayService.kt` / `KeepAlive.kt` /
  `LocalFakeServer.kt` / `TraeUpstream.kt` / `TraeIntlUpstream.kt` / `TraeAuthApi.kt` /
  `TraeAuthStore.kt` / `TraeRegion.kt` / `ModelsRepo.kt` / `UsageStore.kt`。
  （`KeepAlive.kt` 里的 `check()` / `rom()` / `romGuide()` 是**给保活页供数据**的，
   你**可以调它们的返回值排版**，但不要改它们的实现。）
- 图标：`res/drawable/ic_*.xml`（6 个矢量图：`ic_account` / `ic_dashboard` / `ic_history` /
  `ic_launcher` / `ic_models` / `ic_more`）。**你可以重画这些矢量图**（换线条/形状），
  也可以**新增**图标文件，但**不要引入图片位图**（保持矢量）。
- 配色：`res/values/colors.xml` + `res/values-night/colors.xml`，语义化命名 `wb_*`。
- `docs\UI现状清单.md` = **当前的设计规范**（token 表、helper 说明、页面结构、踩坑）。
  **动手前先读它**，它是现状的准确描述。

## 四、信息架构（**保持不变** —— 你会改视觉，但别删页面、别改职责）

底部导航 5 个 Tab：

| Tab | 副标题 | 必须保留的内容（自上而下） |
|---|---|---|
| **概览** | 本地服务运行状态 | 服务状态（运行中 / 已停止）→ 未运行时「服务未启动」说明 / 运行时「后台常驻中」+ 保活体检提示 + 保活入口 → **服务地址**（可复制）→ 启动/停止按钮 → 当前 Trae 账号卡 → **本地 API Key 卡（默认打码，可显示/隐藏/复制）** → 使用引导（三个端点）→ 登录/切换账号按钮 |
| **账号** | 当前 Trae 登录状态 | 当前账号详情（昵称/UID/区域/Token 状态/到期时间）→ 其它账号列表（**切换** / **删除**）→ 登录卡（复制国服登录链接 / 复制国际服登录链接〔标注"未开发"〕/ 粘贴回调完成登录）→ 积分与用量（当前可用积分 / 权益额度上限 / 今日签到 + 刷新按钮） |
| **模型** | 本地可用的模型表 | 搜索框 → 模型列表（名称 + 等宽 id + 复制按钮） |
| **记录** | 调用用量与记录 | 按模型的用量汇总 → 调用记录列表（OK/ERR 状态 + 模型 + 时间 + 用量明细） |
| **更多** | 说明与致谢 | 关于（含版本号）/ 致谢 / 版权声明 / 账号区域说明（国际服标注「未开发」）/ **保活入口（进二级页）** |

**二级页「保活设置」**（从「更多」进入，顶栏标题变「保活设置」，返回可回「更多」）：
总状态（`还有 N 项待开启` / `已就绪`）→ 一键配置入口 → 逐项清单（✓/⚠ + 说明 +
可一键开启的给「去开启」按钮）→ 本机 ROM 引导（含「复制这些路径」）。

### 必须保留的**功能性细节**（很容易被"重做"时误删，特别注意）

1. **本地 API Key 默认打码**（`MASKED_KEY`），点「显示」才展开 —— 防截图泄露。别改成默认真实值。
2. **服务默认关闭**，概览页必须有明确的「未启动」说明（否则用户看到地址却连不上，以为坏了）。
3. **国际服（SG）标注「未开发」**：`TraeRegion.isSupported` 为 false 的区域，
   chip 文案是「国际服 · 未开发」（琥珀/警示色），并附 `unsupportedNote` 说明；
   点「登录国际服」和「切换到国际服账号」各有一个**确认弹窗**。**这些弹窗别删。**
4. **保活入口 + 待开启项计数**（`还有 N 项…`）在概览页与更多页都要能进。
5. **chip 必须 wrap_content**：`chip()` 里那行 `layoutParams = lp(-2, -2)` **是修 bug 的，别删** ——
   竖直 `LinearLayout` 里不加它，chip 会被拉成**整条横幅**（实测横跨全屏）。
6. **底部导航点击区**：外层容器撑满 1/5 宽并挂 `setOnClickListener`；内层 `TextView` 只做视觉。
   **别把点击监听只挂到内层**（会让点击区变小）。

## 五、设计要求（这次要**重新设计**，不是微调）

**不要模仿 WorkBuddy，也不要把上面那套 `wb_*` 配色当作必须保留的**。
你可以重新定义整套色板与视觉语言 —— 但必须满足下列要求：

### 5.1 基本要求
- **浅色 + 深色两套都要成立**（`values` / `values-night` 双份资源，逐个页面都别漏）。
- **手机竖屏 + 平板横屏都要好看**：内容要有**最大宽度限制**并居中（现在 `CONTENT_MAX_DP = 640`，
  可以调整数值），别让文字横跨 3440px 宽的屏幕。
- **触控目标 ≥ 48dp**（无障碍最低要求）。
- **字号层级至少 5 档清晰区分**：页面标题 / 卡片标题 / 正文 / 说明 / 数值。
- **状态一眼可见**：服务运行中 vs 已停止、各项保活是否就绪、Token 是否有效、账号属于哪个区域。
- **文字对比度达标**：正文与背景对比度 ≥ 4.5:1（WCAG AA），说明文字也别太低。

### 5.2 视觉方向（自定，但要"现代、克制、有品位"）
你可以自由发挥，比如：更讲究的留白节奏、更清晰的信息分组、更克制的用色（大面积中性色 +
少量强调色）、状态用**色彩 + 图标/形状**双重编码（不只靠颜色，色盲也能分辨）。
建议做但由你决定：
- 统一的「设计语言」应该能从**一处**改全站（现在的 helper + token 结构就是为此设计的，**保持这个性质**）。
- 卡片/分隔/圆角的处理要有**一致性**（别同一个 App 里出现 4 套圆角）。

### 5.3 明确**不要**做的事
- 不要引入 androidx / Compose / Material / 任何依赖。
- 不要创建布局 XML。
- 不要改业务逻辑（网络 / 存储 / 状态判断）。
- 不要删除任何现有功能或页面。
- 不要写死颜色值（必须走资源）。
- 不要用位图图片资源（保持矢量图标）。
- 不要改 `MainActivity.kt` 之外的 Kotlin 文件的**逻辑**。

## 六、建议的动手顺序

1. 读 `docs\UI现状清单.md`（现状规范）+ `MainActivity.kt` 的 `companion`（token）
   与**文件末尾的 helper 库**（第 889 行起）。
2. 先定**设计语言**：色板（亮/暗）、圆角档位、间距节奏、字号层级 —— 全部落成 token。
3. 改 **helper 库**（`card()` / `text()` / `chip()` / `action()` / `divider()` …）——
   这一步会让**全站**同时变样。
4. 逐页调整**信息分组与排版**（不改职责）。
5. 顶栏 / 底部导航 / 二级页顶栏。
6. 构建 → 装机 → **逐页截图**（浅色 + 深色）→ 修错位。

## 七、怎么验证（现成的路，照抄即可）

```powershell
# 1) 构建（必须离线可过）
.\gradlew.bat assembleDebug --offline

# 2) 装机 + 重启（7 个启动自检失败会直接崩，属预期）
$adb='D:\Android\sdk\platform-tools\adb.exe'; $d='127.0.0.1:5555'
& $adb -s $d install -r app\build\outputs\apk\debug\app-debug.apk
& $adb -s $d shell am force-stop com.example.traework2api
& $adb -s $d shell am start -n com.example.traework2api/.MainActivity
& $adb -s $d logcat -b crash -d -t 20          # 应为空

# 3) 看界面结构（不需要眼睛就能读到全部文案与控件）
& $adb -s $d shell uiautomator dump /sdcard/u.xml
& $adb -s $d shell "cat /sdcard/u.xml"

# 4) 截图（注意：不要用 exec-out >，会损坏 PNG）
& $adb -s $d shell screencap -p /sdcard/x.png
& $adb -s $d pull /sdcard/x.png .

# 5) 深色模式
& $adb -s $d shell cmd uimode night yes     # 重启 App 后截图；测完 no 恢复

# 6) 手机竖屏（测完必须 reset，否则模拟器一直是手机分辨率）
& $adb -s $d shell wm size 1080x2340
& $adb -s $d shell wm density 420
#   ...测完...
& $adb -s $d shell wm size reset
& $adb -s $d shell wm density reset
```

**环境事实**：
- 设备是 **MuMu 模拟器**，同时出现 `127.0.0.1:5555` 与 `emulator-5554` 两个条目（同一台），
  **必须 `-s 127.0.0.1:5555`**，否则报 `more than one device/emulator`。
- 模拟器当前是**平板横屏**（面板 1440×3440，`cur=3440x1440`，应用区 3440×1312）。
  **底栏 5 项实测 y≈1243，中心 x≈344 / 1032 / 1720 / 2408 / 3096**
  （依次为 概览 / 账号 / 模型 / 记录 / 更多）。改了底栏布局这些坐标就变，仅作参考。
  注意 `uiautomator dump` 里同一个文案可能在别处也出现（例如顶部栏标题），
  认底栏那一行要用 **y≈1185–1302** 这个范围去筛。
- Git Bash 下 adb 要加 `MSYS_NO_PATHCONV=1` 前缀，否则 `/sdcard/x.png` 会被转成 Windows 路径。
- 五个 Tab 的切换：可以用 `adb shell input tap <x> <y>` 点底栏坐标，
  或直接用 `uiautomator dump` 确认渲染结果。

## 八、交付要求

1. 改完必须通过：`.\gradlew.bat assembleDebug --offline`
2. 装机后 `logcat -b crash -d` 为空（启动自检全过）。
3. **逐页截图验证**：浅色 5 页 + 二级页；深色至少概览/账号页（能全测更好）。
4. 手机竖屏与平板横屏**各测一轮**，测完记得 `reset`。
5. 汇报时必须说清：
   - 改了哪些文件 / 哪些函数；
   - **设计思路**（色板怎么定的、层级怎么分的）；
   - **任何你没能验证的地方**（如实说，别写成"已验证"）。

## 九、参考：上一轮踩过的坑（**别再踩**）

1. **chip 在竖直 `LinearLayout` 里会被拉成整条横幅**（实测 `bounds=[43,246][3397,314]`）。
   根因未完全定位，`layoutParams = lp(-2, -2)` 可解 —— 已在 `chip()` 内钉死，别删。
2. **`Button` 是平台 Button**：`setBackgroundColor` 会丢掉圆角，必须用
   `background = shape(fillRes, radius)`（`GradientDrawable`）才能保住圆角与描边。
3. **深色模式下图标不会自动变色**：矢量图必须用 `setTint()` 跟着主题色走
   （现在底栏选中态就是这么处理的，`mutate().setTint(...)`）。
4. **别用 `InsetDrawable` 做导航指示器**：窄屏下会被挤成一条，已弃用。
5. **adb 拉截图不要用 `exec-out >`**（PNG 会坏），用 `screencap` + `pull`。
6. **改 `wm size` 后忘记 `reset`** 会让模拟器一直是手机分辨率，后面所有截图都错。
