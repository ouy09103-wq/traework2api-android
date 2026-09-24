# UI 设计规范与现状（交接材料）

> 目的：让 UI AI / 下一个开发者**不需要读整个 `MainActivity.kt`** 就能上手改视觉。
> 结论先行：**样式极好改** —— 全部 UI 集中在**一个文件**、用**一组 helper 函数**拼装，
> 改样式基本只改 helper 那一段 + `companion` 里的 design token，**不用碰任何业务逻辑**。
>
> 本清单描述**改造后的现状**（2026-09-23 完成一轮视觉重构并逐页实测）。

---

## 1. 一句话架构

**纯代码组装，零布局 XML。** 没有任何 `activity_main.xml` / `fragment_*.xml`。

```
MainActivity.kt
├── companion object      → design token（圆角 / 间距 / 字号 / 内容最大宽度 / 色值走资源）
├── onCreate              → 手搭 顶部栏 / 内容区 / 底部导航
├── render(page)          → 切 Tab：换标题 + 重建内容区 + 刷导航选中态
├── 五个页面函数           → overviewPage / accountPage / modelsPage / recordsPage / morePage
└── 一组 helper            → scroll / card / row / text / mono / caption / section / divider
                            chip(+4 个语义封装) / labelValue / metric / searchInput
                            linkAction / action / shape / color / dp / lp / top / weighted
```

**关键点**：所有页面都用 `card { ... }` / `row { ... }` 这类 Kotlin DSL 拼装，
`card {}` 的 lambda 里 `this` 就是 `LinearLayout`，`addView(...)` 直接写。
→ **改 `card()` 一个函数，全 App 每张卡片同时变样。**

---

## 2. Design token（改样式先看这里）

全部在 `MainActivity.kt` 的 `companion object`，页面里**不许再写字面量**。

### 2.1 圆角（原来 12/16/20/24 四套 → 收成三档语义）

| token | 值 | 用在哪 |
|---|---|---|
| `RADIUS_CARD` | 16f | 卡片、搜索框（容器类） |
| `RADIUS_CONTROL` | 12f | 按钮（实心/描边） |
| `RADIUS_PILL` | 100f | chip、导航选中胶囊、小标签（给大值即全圆角） |

### 2.2 间距（原来是调用处手传 8/10/12 混用）

| token | 值 | 用在哪 |
|---|---|---|
| `GAP` | 12 | `card()` 默认上边距；页面里所有卡片间距统一 |
| `PAGE_PAD_H` | 20 | 页面左右内边距 |
| `PAGE_PAD_TOP` / `PAGE_PAD_BOTTOM` | 16 / 24 | 页面上下内边距 |
| `CARD_PAD` | 16 | 卡片四边内边距 |

### 2.3 字号（原来一半在代码 int、一半在 XML 样式 → 全部收进代码）

| token | 值 | 用在哪 |
|---|---|---|
| `FS_TITLE` | 22f | 顶部栏标题（`sans-serif-medium`） |
| `FS_SECTION` | 14 | 区块标题（小号灰字，层级低于卡片标题） |
| `FS_CARD_TITLE` | 17 | 卡片标题 |
| `FS_METRIC` | 24 | 主指标数值（账号页「当前可用积分」） |
| `FS_BODY` | 15 | 正文（`text()` 默认值） |
| `FS_VALUE` | 14 | 键值对的值 |
| `FS_CAPTION` | 13 | 说明文字 |
| `FS_CHIP` | 12 | chip、底部导航文字 |

### 2.4 自适应

| token | 值 | 作用 |
|---|---|---|
| `CONTENT_MAX_DP` | 640 | 内容最大宽度。手机竖屏 = 屏宽（不受影响）；平板/横屏 = 640dp 并居中，不再让一行文字横跨整个 3440px 屏幕 |

### 2.5 颜色 —— 单一色板，**没有硬编码色值**

- 全站只有一个前缀 `wb_*`（Material 3 语义命名），亮色 `values/colors.xml`、暗色 `values-night/colors.xml`。
- **`tw_*` 那套重复色已删除**（原先 7 个色与 `wb_*` 完全相同，改色要改两处）。
- 主题 `Theme.TraeWork2API` 也改读 `wb_*`，主题与界面不会再不同步。
- 改色只改这两个文件，代码一行都不用动。

| 语义 | `wb_` 亮色 | 暗色 |
|---|---|---|
| primary | `#4F46E5` | `#BEC2FF` |
| primary_container | `#E6E7FF` | `#34358B` |
| on_primary | `#FFFFFF` | `#20205F` |
| background | `#F7F7FC` | `#101016` |
| surface | `#FFFFFF` | `#19191F` |
| surface_container | `#F0F0F8` | `#23232B` |
| surface_variant | `#E7E7F0` | `#2E2E38` |
| on_surface | `#1B1B22` | `#E5E1EA` |
| on_surface_variant | `#5F5F6B` | `#B5B3C0` |
| outline / outline_variant | `#777783` / `#C7C7D1` | `#8B8B97` / `#3A3A44` |
| success / container | `#16794B` / `#D7F6E5` | `#66E3A0` / `#1E4A34` |
| warning / container | `#8A5100` / `#FFE9C2` | `#FFD08A` / `#5A3D17` |
| error / container | `#BA1A1A` / `#FFDAD6` | `#FFB4A8` / `#5A2B2B` |

---

## 3. Helper 逐个说明（这就是"组件库"）

| Helper | 签名要点 | 说明 |
|---|---|---|
| `scroll { }` | `LinearLayout.() -> Unit` | 页面容器。**已内建内容限宽居中**，五个页面全部走它（`modelsPage` 也收口了） |
| `contentWidth()` | → Int | `min(屏宽, 640dp)`，供 `scroll` 与顶栏内层共用 |
| `card(topMargin=GAP) { }` | → `LinearLayout` | 白底 + 16dp 圆角 + 1dp 描边 + 1dp elevation。**改这里 = 改所有卡片** |
| `row(top=0) { }` | 横向 LinearLayout | `gravity=CENTER_VERTICAL` |
| `text(v, size=15, bold=false, colorRes)` | → `TextView` | 基元，`setLineSpacing(0, 1.12f)` |
| `mono(v, size=14, ...)` | → `TextView` | **等宽字体**：服务地址、模型 id、记录时间戳用它，数字/标识符能对齐 |
| `caption(v)` | = `text(v, 13, false, on_surface_variant)` | 说明文字 |
| `section(v)` | 14 bold 灰字 + padding(18/8) | 区块标题，与卡片标题拉开层级 |
| `divider()` | → `View` | 1dp 分隔线（顶栏、底栏与内容区分层） |
| `chip(v, bgRes, textRes)` | 底层胶囊 | 圆角 pill；**显式 `lp(-2,-2)`**，见下 |
| `chipOk/chipWarn/chipDanger/chipInfo` | 语义封装 | 直接用它们，别在调用处手配颜色对 |
| `labelValue(label, value)` | → `View` | 键值行；标签列 **`minWidth=96dp`**（不再是死宽，长标签能撑开），值可长按复制 |
| `metric(label, value)` | → `LinearLayout` | 指标块：小标签在上、数值在下，并排展示两个次要指标 |
| `searchInput(hint)` | → `EditText` | 与卡片同一套白底 + 描边 + 16dp 圆角 |
| `linkAction(label, click)` | → `TextView` | 小胶囊按钮，`primary_container` 底 |
| `action(label, outlined, full, click)` | → `Button` | 主按钮，12dp 圆角；`full=true` 撑满宽、高 48dp |
| `shape(fillRes, radiusDp, strokeRes)` | `GradientDrawable` | **所有背景的唯一来源** |
| `color/dp/lp/top/weighted` | 工具 | `top(v)` = 上边距快捷写法 |

### 3.1 chip 的两个坑（都踩过）

1. **必须显式 `layoutParams = lp(-2, -2)`**。竖直容器里 `addView(chip(...))` 会被拉成**整条横幅**
   （实测 `bounds=[43,246][3397,314]`，横跨整屏）。已在 `chip()` 内部钉死为 wrap。
2. **颜色必须走语义封装**。原先调用处手传颜色对，容易配错（success/warning/primary/error 四种组合散落各处）。

### 3.2 底部导航选中态（别改回"整块背景"）

- 结构是 **外层 `LinearLayout` 撑满 1/5 宽（保证点击区）+ 内层 `TextView` 包住图标与文字**。
- 选中背景贴在内层 `TextView` 上（pill 圆角）→ **胶囊宽度由内容决定**，
  手机竖屏和 3440px 平板横屏都好看。
- 早先用 `InsetDrawable` 固定内缩，在手机窄屏下会把指示器挤成一条，已弃用。
- 点击监听挂在外层容器上，别只挂 `TextView`。

---

## 4. 五个页面结构

底部导航 5 项，`Page` enum 驱动（标题+图标都从 enum 取）：

| Tab | 标题 / 副标题 | 内容（自上而下） |
|---|---|---|
| **概览** | 本地服务运行状态 | 状态 chip →（**未运行时**）「服务未启动」空态卡 /（**运行时**）「后台常驻中」卡（含保活体检提示 + 「检查保活设置」入口）→ 服务地址强调卡（等宽主色地址 + 复制）→ 启停按钮 → Trae 账号卡 → 本地 API Key 卡（**默认打码**，显示/隐藏 + 复制）→ 使用引导卡 → 登录按钮 |
| **账号** | 当前 Trae 登录状态 | 当前账号卡 → 其它账号列表（切换/删除）→ 登录卡（国服/国际服链接 + 粘贴回调）→ 积分与用量卡（**24sp 主数值** + 权益额度/今日签到指标块 + 刷新） |
| **模型** | 本地可用的模型表 | 搜索框 → 模型卡列表（名称 + 等宽 id + 复制） |
| **记录** | 调用用量与记录 | 模型消耗汇总卡 → 调用记录卡（OK/ERR chip + 模型 + **右对齐等宽时间戳** + 用量明细） |
| **更多** | 说明与致谢 | 关于（含**版本号**）/ 致谢 / 版权声明 / 账号区域（国际服标注「未开发」） |

**顶栏**：22sp 标题 + 13sp 副标题，`wb_surface` 底，**内层限宽居中对齐内容区**，底部 1dp 分隔线。
**底栏**：5 项等宽、高 64dp，顶部 1dp 分隔线，选中态见 §3.2。

---

## 5. 原「已知不一致」11 条的处置

| # | 原问题 | 现状 |
|---|---|---|
| 1 | 圆角三套体系（12/16/20/24） | ✅ 收成三档语义 token（容器 16 / 控件 12 / 胶囊 pill） |
| 2 | 卡片间距手传 8/10/12 | ✅ `card()` 默认 `GAP=12`，页面不再传间距 |
| 3 | `modelsPage` 没用 `scroll()`，抄了一遍内边距 | ✅ 收口到 `scroll {}` |
| 4 | 字号 int ≠ sp、代码与 XML 两处 | ✅ 全部收进 `FS_*`，XML 里的文本样式已删 |
| 5 | `styles.xml` 三条死样式零引用 | ✅ `WbText.Caption` / `WbBottomNavItem` / `WbStatusChip` 已删；`WbText.Title` 也搬进代码，XML 只剩主题 |
| 6 | 底栏注释写「4 Tab」实际 5 个 | ✅ 注释已改，且改为由 `Page.entries` 驱动说明 |
| 7 | `tw_` 与 `wb_` 两套重复色 | ✅ 删掉 `tw_*`，主题也改读 `wb_*`，改色只需改一处 |
| 8 | `chip` 无语义、颜色手传 | ✅ 加 `chipOk/chipWarn/chipDanger/chipInfo`；并修掉竖直容器里被拉满的 bug |
| 9 | `labelValue` 左列固定 96dp 长标签会挤 | ✅ 改 `minWidth=96dp`，长标签自动撑开 |
| 10 | 深色模式未逐页实测 | ✅ 已实测概览页 / 账号页（卡片、chip、胶囊、主色文字均正常） |
| 11 | 无横竖屏专门处理 | ✅ 内容限宽 640dp 居中 + 顶栏内层限宽 + 底栏内容自适应胶囊；手机竖屏（1080×2340@420）与平板横屏（3440×1312）**都已实测** |

### 本轮额外新增
- 顶栏 / 底栏 1dp 分隔线（原来只靠背景色差，分层不清）。
- 概览页「服务地址」升级为强调卡：等宽 + 主色 + 一键复制。
- 本地 API Key **默认打码**，点「显示/隐藏」切换（防截图泄露），可复制。
- 记录页时间戳改等宽字体并右对齐，一列扫下来是对齐的。
- 更多的「应用说明」卡显示版本号（取自 `PackageManager`，不依赖 AGP 的 BuildConfig 开关）。
- `getSystemService/setPrimaryClip` 三处重复 → 收口到 `copyToClipboard()`。

---

## 6. 三条硬约束（**别违反**）

1. **不要引入布局 XML / 不要引入任何依赖**（项目至今**仅 okhttp 4.12.0**，见 `app/build.gradle.kts`；
   交接文档 §1 明确"不要加依赖"）。
2. **不要动业务逻辑**：`render()` 的 Tab 路由、各页的数据读取（`TraeAuthStore`/`TraeUpstream`/
   `UsageStore`/`ModelsRepo`）保持原样。**只改 helper 与各页的视觉结构**。
3. **颜色/尺寸必须走资源与 token**（`wb_*` + `values-night` + companion 里的 `FS_*`/`RADIUS_*`/`GAP`），
   不要写字面量 —— 现状很干净，保持住。

---

## 7. 改动后怎么验证（现成的路）

```powershell
# 构建
.\gradlew.bat assembleDebug --offline

# 装 + 重启（五个启动自检失败会直接崩，属预期）
$adb='D:\Android\sdk\platform-tools\adb.exe'; $d='127.0.0.1:5555'
& $adb -s $d install -r app\build\outputs\apk\debug\app-debug.apk
& $adb -s $d shell am force-stop com.example.traework2api
& $adb -s $d shell am start -n com.example.traework2api/.MainActivity
& $adb -s $d logcat -b crash -d -t 20          # 应为空

# 看界面结构（不需要眼睛，能读到所有文案）
& $adb -s $d shell uiautomator dump /sdcard/u.xml
& $adb -s $d shell "cat /sdcard/u.xml"
```

- 模拟器**横屏 3440×1312**（MuMu 平板模式）。底栏 5 项坐标 ≈ x=344,1032,1720,2408,3096 / y=1244。
- 截图：`adb shell screencap -p /sdcard/x.png` + `adb pull`（**不要**用 `exec-out >`，会损坏）。
- 深色模式实测：`adb shell cmd uimode night yes` → 重启 App → 截图 → `no` 恢复。
- 手机竖屏实测：`adb shell wm size 1080x2340` + `wm density 420` → 重启 App → 截图 →
  `wm size reset` + `wm density reset` 恢复。
- ⚠️ Git Bash 下 adb 要带 `MSYS_NO_PATHCONV=1` 前缀，否则 `/sdcard/...` 会被转成 Windows 路径。
- 模拟器内 Chatbox 连 `127.0.0.1:8788` 即本机服务，**不需要 adb forward**。
