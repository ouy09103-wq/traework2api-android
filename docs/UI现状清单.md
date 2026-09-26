# UI 设计规范与现状（交接材料）

> 目的：让 UI AI / 下一个开发者**不需要读整个 `MainActivity.kt`** 就能上手改视觉。
> 结论先行：**样式极好改** —— 全部 UI 集中在**一个文件**、用**一组 helper 函数**拼装，
> 改样式基本只改 helper 那一段 + `companion` 里的 design token，**不用碰任何业务逻辑**。
>
> 本清单描述**改造后的现状**（2026-09-24 完成「多风格 + 独立亮暗」改造并实测）。
>
> ⚠️ **本轮最大变化**：颜色**不再从 `R.color.wb_*` 取**，而是「语义名 → 运行期查表」
> （见 §2.5 与 §2.6）。`res/values/colors.xml` / `values-night/colors.xml` 仍保留，
> 但现在**只代表默认风格「靛蓝」**，供启动主题与 6 个矢量图标使用。

---

## 1. 一句话架构

**纯代码组装，零布局 XML。** 没有任何 `activity_main.xml` / `fragment_*.xml`。

```
MainActivity.kt
├── companion object      → 只留字号 token（FS_*）与打码常量；圆角/间距**不在**这里了
├── attachBaseContext     → 先加载外观偏好 + 按亮暗覆写 uiMode（比 onCreate 还早）
├── onCreate              → 外观自检 → 手搭 顶部栏 / 内容区 / 底部导航 → 恢复上次所在页
├── render(page)          → 切 Tab：换标题 + 重建内容区 + 刷导航选中态
├── redraw()              → 按“现在在哪”重绘（Tab 或二级页都还原）
├── 五个页面函数           → overviewPage / accountPage / modelsPage / recordsPage / morePage
├── 两个二级页             → keepAlivePage（保活设置）/ stylePage（外观风格）
└── 一组 helper            → scroll / card / row / text / mono / caption / section / divider
                            chip(+4 个语义封装) / labelValue / metric / searchInput
                            linkAction / action / shape / color / dp / lp / top / weighted

AppStyle.kt（新）
├── Style                 → 一套风格：中文名 + 圆角/间距形态 + 亮暗两份色表（19 个语义键）
├── STYLES                → 靛蓝 / 石墨 / 苔绿 / 落日
├── color(name)           → 按“当前风格 + 当前亮暗”运行期取色（缺键回退 on_surface）
├── Mode                  → 跟随系统 / 浅色 / 深色
├── applyNightOverride()  → 用 createConfigurationContext 把亮暗落到 uiMode
└── selfCheck()           → 色表齐全 / 色值不透明 / 对比度 ≥ 4.5:1
```

**关键点**：所有页面都用 `card { ... }` / `row { ... }` 这类 Kotlin DSL 拼装，
`card {}` 的 lambda 里 `this` 就是 `LinearLayout`，`addView(...)` 直接写。
→ **改 `card()` 一个函数，全 App 每张卡片同时变样。**

---

## 2. Design token（改样式先看这里）

**字号**在 `MainActivity.kt` 的 `companion object`（`FS_*`，不随风格变）；
**圆角 / 间距 / 内容宽度**在 `AppStyle.Style` 里（**每套风格一套值**），
`MainActivity` 只暴露同名只读属性（`RADIUS_CARD` / `GAP` / `PAGE_PAD_*` / `CARD_PAD` /
`BAR_PAD_*` / `CONTENT_MAX_DP`），页面里**不许再写字面量**。

### 2.1 圆角（原来 12/16/20/24 四套 → 收成三档语义，且**每套风格各一组**）

| token | 靛蓝 | 石墨 | 苔绿 | 落日 | 用在哪 |
|---|---|---|---|---|---|
| `RADIUS_CARD` | 16 | 6 | 24 | 20 | 卡片、搜索框（容器类） |
| `RADIUS_CONTROL` | 12 | 6 | 18 | 14 | 按钮（实心/描边） |
| `RADIUS_PILL` | 100 | 8 | 100 | 100 | chip、导航选中胶囊（给大值即全圆角；石墨刻意给 8 做"方正"） |

### 2.2 间距（原来是调用处手传 8/10/12 混用，现在**跟着风格走**）

| token | 靛蓝 | 石墨 | 苔绿 | 落日 | 用在哪 |
|---|---|---|---|---|---|
| `GAP` | 12 | 10 | 16 | 12 | `card()` 默认上边距；页面里所有卡片间距统一 |
| `PAGE_PAD_H` | 20 | 16 | 24 | 20 | 页面左右内边距 |
| `PAGE_PAD_TOP` / `BOTTOM` | 16 / 24 | 14 / 18 | 20 / 28 | 18 / 24 | 页面上下内边距 |
| `CARD_PAD` | 16 | 14 | 20 | 18 | 卡片四边内边距 |
| `BAR_PAD_TOP` / `BOTTOM` | 14 / 12 | 12 / 10 | 16 / 14 | 14 / 12 | 顶栏上下内边距 |

> 这就是"切风格连**形态**一起变"的实现方式：只改颜色的话，四套风格看起来会是同一个东西。

### 2.3 字号（原来一半在代码 int、一半在 XML 样式 → 全部收进代码；**不随风格变**）

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

| token | 靛蓝 | 石墨 | 苔绿 | 落日 | 作用 |
|---|---|---|---|---|---|
| `CONTENT_MAX_DP` | 640 | 680 | 600 | 640 | 内容最大宽度。手机竖屏 = 屏宽（不受影响）；平板/横屏 = 该值并居中，不再让一行文字横跨整个 3440px 屏幕 |
| `elevation` | 1 | 0 | 2 | 1 | 卡片阴影高度（石墨刻意压平，配合小圆角更"硬"） |

### 2.5 颜色 —— **按语义名运行期取色**（本轮核心，别再改回资源 id）

- 色值**只在 `AppStyle.kt` 的 4 套风格 × 亮/暗 8 份色表**里；界面与 helper 只写语义名
  （`color("primary")` / `text(..., "on_surface_variant")`），**不写 `R.color.*`，不写字面量**。
- 19 个语义键（沿用 Material 3 命名）：`primary` / `primary_container` / `on_primary` /
  `on_primary_container` / `secondary` / `background` / `surface` / `surface_container` /
  `surface_variant` / `on_surface` / `on_surface_variant` / `outline` / `outline_variant` /
  `success(+_container)` / `warning(+_container)` / `error(+_container)`。
- ⚠️ **不能用 `getColor(R.color.wb_*)` 做"按风格取色"**：资源 id 是编译期常量，
  只有 `values/` 与 `values-night/` 两份，**换不了风格**。
- `res/values/colors.xml` / `values-night/colors.xml` 保留，但只代表**默认风格「靛蓝」**，
  供启动主题 `Theme.TraeWork2API` 与 6 个矢量图标使用。
- 取不到键时**回退 `on_surface`**（绝不返回 0 —— 那是全透明，文字会直接看不见）。

下面这张表只是**默认风格「靛蓝」**的取值（= `values/colors.xml` + `values-night/colors.xml`）；
另外 3 套风格的色值在 `AppStyle.kt` 里，每套亮/暗各一份。

| 语义 | 靛蓝 亮色 | 靛蓝 暗色 |
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
| `contentWidth()` | → Int | `min(屏宽, CONTENT_MAX_DP)`（**值随风格变**），供 `scroll` 与顶栏内层共用 |
| `card(topMargin=GAP, tone=NORMAL) { }` | → `LinearLayout` | 三种层次见下表；**改这里 = 改所有卡片** |
| `row(top=0) { }` | 横向 LinearLayout | `gravity=CENTER_VERTICAL` |
| `text(v, size=15, bold=false, colorName)` | → `TextView` | 基元，`setLineSpacing(0, 1.12f)`；**颜色传语义名**（如 `"on_surface"`） |
| `mono(v, size=14, ...)` | → `TextView` | **等宽字体**：服务地址、模型 id、记录时间戳用它，数字/标识符能对齐 |
| `caption(v)` | = `text(v, 13, false, "on_surface_variant")` | 说明文字 |
| `section(v)` | 14 bold 灰字 + padding(18/8) | 区块标题，与卡片标题拉开层级 |
| `divider()` | → `View` | 1dp 分隔线（顶栏、底栏与内容区分层） |
| `chip(v, backgroundName, textName, dotTone?, dotOn)` | 底层**横向**容器 | 圆角 `RADIUS_PILL`；传 `dotTone` 会加状态点；**显式 `lp(-2,-2)`**，见下 |
| `statusChip(v, tone, on)` | 带状态点的胶囊 | **状态一眼可见**：点 + 字（`on` 决定实心/空心） |
| `dot(tone, on, size)` | → `View` | 状态圆点：实心 = 成立，空心（描边）= 未成立，色盲也能分辨 |
| `chipOk/chipWarn/chipInfo` | 语义封装 | 直接用它们，别在调用处手配颜色对 |
| `labelValue(label, value)` | → `View` | 键值行；标签列 **`minWidth=96dp`**（不再是死宽，长标签能撑开），值可长按复制 |
| `metricOf(label, valueView)` | → `LinearLayout` | 指标块：小标签在上、数值在下。**值控件由调用方给** —— 这样刷新数据可以原地改文字（见下） |
| `shareBar(value, max)` | → `View` | 占比条（`surface_variant` 轨道 + `primary` 填充，用 weight 分两段）。记录页的模型消耗汇总用它 |
| `quotaValue(v)` | → `String` | 积分数值位：刷新中「…」、没拿到「—」 |
| `searchInput(hint)` | → `EditText` | 与卡片同一套 `surface` 底 + 描边 + `RADIUS_CARD` 圆角 |
| `linkAction(label, overContainer=false, click)` | → `TextView` | 小胶囊按钮，`primary_container` 底，高 **48dp**；卡底本身是 `primary_container` 时传 `overContainer=true`（改 `surface` 底，否则看不见） |
| `action(label, outlined, full, danger, click)` | → `Button` | 主按钮，`RADIUS_CONTROL` 圆角；`full=true` 撑满宽、高 48dp；**`danger=true`**（删除这类破坏性操作）文字与描边走 `error` 色 |
| `confirmDialog(title, message, okText, cancelText?, danger?, content?, onOk)` | → `AlertDialog` | **统一风格的确认弹窗**：标题/正文/按钮全部自己画，窗口底换成当前风格的 `surface` + 圆角。⚠️ 别用 `AlertDialog.Builder(this).setTitle(...)` —— 那个走平台主题，按钮高亮写死靛蓝、底是系统灰，**换风格就会脱节** |
| `shape(fillName, radiusDp, strokeName?, strokeDp=1)` | `GradientDrawable` | **所有背景的唯一来源**；`strokeDp` 供选中态加粗描边 |
| `color/dp/lp/top/weighted` | 工具 | `color(name)` 走 `AppStyle.color`；`top(v)` = 上边距快捷写法 |

### 3.1 chip 的两个坑（都踩过）

1. **必须显式 `layoutParams = lp(-2, -2)`**。竖直容器里 `addView(chip(...))` 会被拉成**整条横幅**
   （实测 `bounds=[43,246][3397,314]`，横跨整屏）。`chip()` 现在返回**横向 LinearLayout**
   （要放状态点），这行 wrap 钉在容器上，别删。
2. **颜色必须走语义封装**。原先调用处手传颜色对，容易配错（success/warning/primary/error 四种组合散落各处）。

### 3.2 底部导航选中态（别改回"整块背景"）

- 结构是 **外层 `LinearLayout` 撑满 1/5 宽（保证点击区）+ 内层 `TextView` 包住图标与文字**。
- 选中背景贴在内层 `TextView` 上（pill 圆角）→ **胶囊宽度由内容决定**，
  手机竖屏和 3440px 平板横屏都好看。
- 早先用 `InsetDrawable` 固定内缩，在手机窄屏下会把指示器挤成一条，已弃用。
- 点击监听挂在外层容器上，别只挂 `TextView`。
- 选中胶囊的**形态跟着风格走**（`RADIUS_PILL`）：靛蓝/苔绿/落日是胶囊，石墨（8dp）是方正小块。

### 3.3 卡片三层次 + 页面框架（本轮建立）

**卡片不是都长一样**，用 `CardTone` 拉开优先级：

| tone | 底色 | 描边 / 阴影 | 用在哪 |
|---|---|---|---|
| `NORMAL`（默认） | `surface` | 1dp `outline_variant` + 风格 elevation | 常规内容卡（账号、模型、记录、保活…） |
| `SOFT` | `surface_container` | 无 | **说明 / 空态**（服务未启动·后台常驻中、使用引导、未登录、暂无记录） |
| `EMPHASIS` | `primary_container` | 无 | 本页最该看的那块（概览的**服务地址卡**） |

> 有底色的面板**不加阴影** —— 靠颜色分层，再叠阴影会显脏。

**页面框架**：顶栏与底栏的底色都是 **`background`**（和页面同色），各自保留 1dp 分隔线。
这样"凸起"的只有卡片，层次由卡片承载；旧版顶栏/底栏用 `surface`，和卡片同色，整屏看着一样平。
⚠️ 分隔线要留着：顶栏与内容同色后，卡片滑到边界处没有线会很突兀。

**状态双编码**：状态类信息一律「**圆点 + 文字**」（`statusChip`），圆点实心 = 成立、
空心 = 未成立 —— 不只靠颜色，色盲也能分辨（任务书 §5.2 要求）。

### 3.4 图标（全矢量，两组）

**底部导航 5 个**：`res/drawable/ic_dashboard|ic_account|ic_models|ic_history|ic_more.xml`
—— 统一 **24dp 视口 + 1.7dp 描边 + 圆头圆角**（`ic_more` 是三个实心点，视觉重量相当）。
色值只是占位，实际颜色由代码 `mutate().setTint()` 按当前风格刷（**换风格必须重新 tint**，
否则图标不跟着变色）。

**桌面图标**（`android:icon="@mipmap/ic_launcher"`）：

| 文件 | 作用 |
|---|---|
| `mipmap-anydpi-v26/ic_launcher.xml` | **自适应图标**（API 26+）：background + foreground + monochrome |
| `drawable/ic_launcher_background.xml` | 背景层：满幅纯色 `#4F46E5` |
| `drawable/ic_launcher_foreground.xml` | 前景层：白色「双向箭头」＝请求转发出去/响应回来 |
| `mipmap-anydpi/ic_launcher.xml` | API 24–25 回落：满幅方形旧形态（图形更大，因为整块方形都可见） |

- ⚠️ **前景图形必须落在画布中间 72×72 的可见区（圆形遮罩时约 66）内**，
  现在图形宽约 54（= 画布的 50%），任何遮罩都不会被切。
- ⚠️ **桌面图标不能跟着 App 内风格切换**（那要挂多个 `activity-alias` 再动态启用，属黑魔法），
  所以固定一个「招牌色」= 默认风格的靛蓝。要换只改 `ic_launcher_background.xml` 一行。
- `monochrome` 复用前景层（系统只取它的 alpha 上色），支持 Android 13+ 的「主题图标」。
- 校验是否真的用上了自适应图标：`aapt2 dump badging app-debug.apk | grep icon`
  应指向 `res/mipmap-anydpi-v26/ic_launcher.xml`。

### 3.5 刷新数据：原地改控件，**不要整体重建**

`render()` / `redraw()` 都会 `removeAllViews()` 重建内容区 —— **滚动位置会被重置到页首**。
对"点一下刷新"这类操作，后果是用户看不到自己刚点的那张卡在变
（实测：给积分刷新加了「…」加载态，却因为重建，加载态一次都没被看见）。

所以"改了数据再把界面刷一下"的地方要**原地更新**：
`accountPage` 构建时把"怎么把最新数据刷进这几个控件"登记到 `quotaApply`，
刷新前/后各调一次它，页面不重建 → 滚动位置不丢、也不闪。
（值控件因此必须由调用方持有 —— 这就是 `metricOf(label, valueView)` 存在的原因。）

> ⚠️ 试过给 `redraw()` 加"记住滚动位置、画完再滚回去"，**不可靠**：新 ScrollView 的内容
> 还没量完高度，`scrollTo` 会被夹回 0；`post` 里注册 `OnGlobalLayoutListener` 也稳定不了。
> 结论：**要保住滚动位置，就别重建页面。**

> 例外：启停服务**必须**重建（「服务未启动」卡与「后台常驻中」卡是两张不同的卡）。
> 那个按钮在页面上部、用户本来就停在顶部，跳一下可以接受。

---

## 4. 五个页面结构

底部导航 5 项，`Page` enum 驱动（标题+图标都从 enum 取）：

| Tab | 标题 / 副标题 | 内容（自上而下） |
|---|---|---|
| **概览** | 本地服务运行状态 | 状态 chip（启停中显示「启动中…/停止中…」）→（**未运行时**）「服务未启动」空态卡 /（**运行时**）「后台常驻中」卡（含保活体检提示 + 「检查保活设置」入口）→ 服务地址强调卡（等宽主色地址 + 复制）→ 启停按钮（进行中禁用 + 显示进展）→ Trae 账号卡 → 本地 API Key 卡（**默认打码**，显示/隐藏 + 复制）→ 使用引导卡（3 个端点各带「复制」）→ 登录按钮 |
| **账号** | 当前 Trae 登录状态 | 当前账号卡 → 其它账号列表（切换/删除）→ 登录卡（国服/国际服链接 + 粘贴回调）→ 积分与用量卡（**24sp 主数值** + 权益额度/今日签到指标块 + 刷新；刷新中数值位显示「…」、按钮变「刷新中…」） |
| **模型** | 本地可用的模型表 | **未加载时空态**：只有一张说明卡（「模型目录尚未加载」+ 说明 + 「刷新模型目录」大按钮），搜索框与列表**隐藏** → **已加载**：搜索框 → 工具条（「刷新模型目录」按钮 + `已加载 · N 个模型`；刷新中按钮置灰变「刷新中…」）→ **搜不到时的空态卡**（`SOFT`，见 §3.3）→ 模型卡列表（名称 + 等宽 id + 复制）。⚠️ 无内置兜底清单，重启回空状态（0.4.3） |
| **记录** | 调用用量与记录 | 模型消耗汇总卡（每行数字下带**占比条**）→ 调用记录卡（OK/ERR chip + 模型 + **右对齐等宽时间戳** + 用量明细） |
| **更多** | 说明与致谢 | **外观（入口卡，显示当前风格 + 亮暗）→ 可进二级页** / 关于（含**版本号**）/ 致谢 / 版权声明 / 保活（入口卡 + `N 项待开启`，可进二级页） |

**二级页**（从「更多」进入；顶栏换标题，系统返回键回「更多」，底部 Tab 高亮保持在「更多」）：

| 二级页 | 顶栏标题 / 副标题 | 内容 |
|---|---|---|
| **保活设置** | 保活设置 / 让服务在后台长期活着 | 总状态（`还有 N 项待开启` / `已就绪`）→ 一键配置 → 逐项清单（✓/⚠ + 说明 + 可一键的给「去开启」）→ 本机 ROM 引导（含「复制这些路径」） |
| **外观风格**（本轮新增） | 外观风格 / 配色与形态，随你挑 | 4 张风格卡（左侧色样预览 + 中文名 + 形态说明；选中项 = 主色 2dp 描边 + 「✓ 使用中」）→ 亮暗三选一（跟随系统 / 浅色 / 深色，圆形单选标记）→ 一句"切换后立即生效并记住"的说明 |

> 二级页状态用 `showKeepAlive` / `showStyle` 两个标记 + `onSaveInstanceState` 保存，
> 因此 `recreate()`（切风格）与系统夜间模式切换都能**回到原处**，不会被弹回 Tab 页。
> 二级页顶栏**一定有「← 返回」**（`backAction`，标题上方）—— 不能只靠系统返回键，
> 纯小白不一定知道那个键。返回键放在标题**上方**而不是左侧，是为了保住"标题左边缘与内容区对齐"。

**顶栏**：22sp 标题 + 13sp 副标题，`surface` 底，**内层限宽居中对齐内容区**，底部 1dp 分隔线。
**底栏**：5 项等宽、高 64dp，顶部 1dp 分隔线，选中态见 §3.2。

---

## 5. 原「已知不一致」11 条的处置

| # | 原问题 | 现状 |
|---|---|---|
| 1 | 圆角三套体系（12/16/20/24） | ✅ 收成三档语义 token（容器 / 控件 / 胶囊）——本轮起**每套风格各一组**值 |
| 2 | 卡片间距手传 8/10/12 | ✅ `card()` 默认 `GAP`，页面不再传间距（`GAP` 也随风格变） |
| 3 | `modelsPage` 没用 `scroll()`，抄了一遍内边距 | ✅ 收口到 `scroll {}` |
| 4 | 字号 int ≠ sp、代码与 XML 两处 | ✅ 全部收进 `FS_*`，XML 里的文本样式已删 |
| 5 | `styles.xml` 三条死样式零引用 | ✅ `WbText.Caption` / `WbBottomNavItem` / `WbStatusChip` 已删；`WbText.Title` 也搬进代码，XML 只剩主题 |
| 6 | 底栏注释写「4 Tab」实际 5 个 | ✅ 注释已改，且改为由 `Page.entries` 驱动说明 |
| 7 | `tw_` 与 `wb_` 两套重复色 | ✅ 删掉 `tw_*`，主题也改读 `wb_*`，改色只需改一处 |
| 8 | `chip` 无语义、颜色手传 | ✅ 加 `chipOk/chipWarn/chipDanger/chipInfo`；并修掉竖直容器里被拉满的 bug |
| 9 | `labelValue` 左列固定 96dp 长标签会挤 | ✅ 改 `minWidth=96dp`，长标签自动撑开 |
| 10 | 深色模式未逐页实测 | ✅ 已实测概览页 / 账号页（卡片、chip、胶囊、主色文字均正常） |
| 11 | 无横竖屏专门处理 | ✅ 内容限宽居中 + 顶栏内层限宽 + 底栏内容自适应胶囊；平板横屏（3440×1440）与手机尺寸窗口（411dp 宽）**都已实测** |

### 本轮（2026-09-24 多风格）额外新增
- **`AppStyle.kt`：4 套风格（靛蓝 / 石墨 / 苔绿 / 落日）× 亮暗，运行期可切**（见 §2.5、§4）。
- **「外观风格」二级页** + 「更多」页的外观入口卡（显示当前风格与亮暗）。
- 取色从 `R.color.wb_*`（编译期常量）改为**语义名运行期查表**，全站 33 处。
- 圆角/间距 token 改为**读当前风格**，切风格时形态一起变。
- `redraw()`：修掉"停在二级页时被后台回调踢回 Tab 页"的 bug。
- `linkAction` 触控高度 40dp → 48dp（无障碍下限）。
- **卡片三层次 `CardTone`**（见 §3.3）：服务地址升级为强调卡（底色面板），说明/空态改柔和卡。
- **状态双编码**：状态点 + 文字（`statusChip`），按在服务状态、转发状态、调用 OK/ERR 上。
- **顶栏/底栏底色改为 `background`**，让"凸起"的只有卡片（层次由卡片承载）。
- **6 个矢量图标重画**（`ic_dashboard` / `ic_account` / `ic_models` / `ic_history` / `ic_more`）：
  统一 24dp 视口 + 1.7dp 描边 + 圆头圆角，与色板同一套线条语言（`ic_launcher` 未动）。
- **桌面图标改成自适应图标**（靛蓝底 + 白色双向箭头），见 §3.4。
- **收尾四件**（第二轮）：
  1. 二级页顶栏加**「← 返回」**（原来只能按系统返回键，小白会卡住）；
  2. 模型页**搜索无结果时的空态卡**（原来整页空白，像坏了）；
  3. 4 个确认弹窗改用自绘的 `confirmDialog()`，**配色跟着当前风格**（原来走平台主题、写死靛蓝）；
  4. 「更多」页的说明类卡（应用说明 / 致谢 / 版权与免责声明）改用 `SOFT`，
     与全站"说明 = 柔和层"的判断保持一致。
- 上一轮那批：顶栏 / 底栏 1dp 分隔线、服务地址强调卡、API Key 默认打码、
  记录页等宽时间戳、更多页版本号、`copyToClipboard()` 收口。
- **再加三件**（第三轮）：
  5. 记录页「模型消耗汇总」每行加**占比条**（按相对最大那组，不是占总量 —— 模型多了每条都只剩一小截）；
  6. 概览「使用引导」的 3 个端点各配「复制」（复制的是**完整 URL**，提示里把 URL 念出来）；
  7. 两处反馈空档：「刷新积分」时数值位显示「…」+ 按钮变「刷新中…」；
     启停服务时 chip 与按钮都变「启动中…/停止中…」且按钮禁用。
     ⚠️ 刷新积分走**原地更新**（见 §3.5），不要用重建。

---

## 6. 四条硬约束（**别违反**）

1. **不要引入布局 XML / 不要引入任何依赖**（项目至今**仅 okhttp 4.12.0**，见 `app/build.gradle.kts`；
   交接文档 §1 明确"不要加依赖"）。
2. **不要动业务逻辑**：`render()` 的 Tab 路由、各页的数据读取（`TraeAuthStore`/`TraeUpstream`/
   `UsageStore`/`ModelsRepo`）保持原样。**只改 helper 与各页的视觉结构**。
3. **颜色一律写语义名，色值只放 `AppStyle.kt`**：不要写字面量色值，
   **也不要用 `R.color.wb_*` 取色**（资源 id 是编译期常量，换不了风格）。
4. **圆角/间距只用 token**（`RADIUS_*` / `GAP` / `PAGE_PAD_*` / `CARD_PAD` / `CONTENT_MAX_DP`），
   这些值随风格变，**不要在新页面里手写数字**。字号用 `FS_*`。

---

## 7. 改动后怎么验证（现成的路）

```powershell
# 构建
.\gradlew.bat assembleDebug --offline

# 装 + 重启（八个启动自检失败会直接崩，属预期；本轮新增 AppStyle.selfCheck）
$adb='D:\Android\sdk\platform-tools\adb.exe'; $d='127.0.0.1:5555'
& $adb -s $d install -r app\build\outputs\apk\debug\app-debug.apk
& $adb -s $d shell am force-stop com.example.traework2api
& $adb -s $d shell am start -n com.example.traework2api/.MainActivity
& $adb -s $d logcat -b crash -d -t 20          # 应为空

# 看界面结构（不需要眼睛，能读到所有文案）
& $adb -s $d shell uiautomator dump /sdcard/u.xml
& $adb -s $d shell "cat /sdcard/u.xml"
```

- 模拟器**横屏**（MuMu 平板模式，app 区实测 3440×1440，density 2.0）。底栏 5 项坐标 ≈
  x=343,1031,1719,2407,3095 / y=1243。
- 截图：`adb shell screencap -p /sdcard/x.png` + `adb pull`（**不要**用 `exec-out >`，会损坏）。
- 深色模式实测：`cmd uimode night yes` → 重启 App → 截图 → `no` 恢复。
- **多风格实测**（本轮重点）：
  - 切风格/切亮暗后**断言**「更多」页那句 `当前：<风格> · <模式>`，比截图更可靠。
  - 4 套风格 × 亮/暗 的概览页各一张；截图存在 `docs/ui-shots/`。
  - 持久化：`am force-stop` 后重开，再断言那句话。
- ⚠️ **模拟器被厂商锁死横屏**：`wm size` 与 `cmd window user-rotation lock 0` 都无法转成真竖屏。
  想测"手机竖屏形态"要用**交换后的尺寸**：`wm size 2340x1080` + `wm density 420`
  → app 实际渲染为 1080×2340px = **411dp 宽 × 867dp 高**，接近真机竖屏。
  测完 `wm size reset` + `wm density reset`。
- ⚠️ **点控件别用固定坐标**：不同风格的 `CARD_PAD`/`GAP` 不同，同一控件在不同风格下 y 会差上百 px
  （实测因此点空过两次）。用 `uiautomator dump` 找到目标文案的 `bounds` 再算中心点。
- ⚠️ **点弹窗按钮要 x、y 都算**：两个按钮等宽并排、**中间隔 8dp**，弹窗又居中，
  所以"x 固定 1720"很可能正落在间隙上 —— 表现是"点了没反应"（实测踩到，白排查一轮）。
- ⚠️ **二级页的 dump 里第一个 `text` 是「← 返回」**，不是页面标题；
  用"取第一个文案"当"当前在哪一页"的断言会误判。
- ⚠️ 文案匹配要用**部分匹配**（`grep -F 'text="登录国际服'`）：按钮文案常带后缀
  （`登录国际服（Trae SG · 未开发）`），全匹配找不到。
- ⚠️ **抓加载态要留一点延迟**：`input tap …; screencap` 会把点击**处理之前**的那一帧拍下来
  （看着像"没反应"）。加 `sleep 0.15~0.4` 再截；但网络快时窗口可能不到 0.4 秒，也可能抓不到。
- ⚠️ 页面在视口里的**卡片要滚动到位再找**：`uiautomator dump` 不含滚出视口的节点，
  直接按文案找「刷新」会"找不到"（实测踩到）。
- ⚠️ **`recreate()` 后的 2 秒内别点**：窗口重建期间输入会被丢掉（实测：切风格后立刻点亮暗，点了没反应）。
  每次点击后留 ≥3s。
- ⚠️ **判断"某块背景到底是哪个色"不要靠肉眼**：缩略图上看不出 `#F7F7FC` 与 `#FFFFFF` 的区别，
  容易误判"改动没生效"。直接取像素最靠谱：写个 20 行的纯 Python PNG 取样器
  （zlib 解 IDAT + 反过滤），或 `pip install Pillow` 后 `Image.open(p).getpixel((x,y))`。
  实测取样确认过：顶栏 = `#F7F7FC`（=background）、柔和卡 = `#F0F0F8`（=surface_container）、
  强调卡 = `#E6E7FF`（=primary_container）、状态点是"红环 + 白心"（空心已生效）。
- ⚠️ Git Bash 下 adb 要带 `MSYS_NO_PATHCONV=1` 前缀；但 `install`/`pull` 的**文件路径要写
  Windows 风格**（`D:/...`）—— `MSYS_NO_PATHCONV=1` 会让 `/d/...` 原样传给 adb.exe 而失效。
- 模拟器内 Chatbox 连 `127.0.0.1:8788` 即本机服务，**不需要 adb forward**。
