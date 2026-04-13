# 共享相册：设计评审（Plan Design Review）

**需求来源**：`docs_共享相册App需求文档.md` **第 4 章**（含 §4.1 全局 tokens、§4.3 交互反馈）  
**评审类型**：plan-design-review（信息架构、关键状态、M3 映射、「AI 味」规避）  
**文档状态**：草案  
**日期**：2026-04-13

---

## 维度速评（0–10，并说明到 10 还差什么）

| 维度 | 分 | 到 10 需要补什么 |
|------|----|------------------|
| 信息架构 | 7 | 各屏入口/返回栈、未连接 vs 已连接的首页差异、设备页与「创建/加入」关系写死成一张 IA 图 |
| 关键状态 | 6 | §4.3 只举了相册空、泛化「网络错误」；需按屏列出加载/空/错/权限拒绝 |
| 视觉与 M3 对齐 | 8 | §4.1 已锚定 M3；需把「绿/红/黄状态」映射到 **M3 语义色 roles**，避免硬编码 RGB |
| 组件可实施性 | 7 | 步骤指示器、三格信号、沉浸式详情工具栏在 Compose 里要指定 **具体 M3 组件组合** |

---

## 1. 各屏信息架构（对照 §4.2）

### 1.1 首页 `GalleryScreen`（§4.2.1）

- **一级信息**  
  - 当前上下文：**相册名称**（已连接）或 **应用名**（未连接）  
  - **连接状态**（与 §4.1 语义色一致）  
  - **照片网格**（主内容）  
- **二级信息**  
  - 缩略图上的 **标记数量/星级**（叠加层）  
- **全局动作**  
  - **FAB**：已连接 → 进入系统选图；未连接 → **「创建」**（文档写弹出创建对话框，与 §4.2.3 流程需统一：对话框 vs 独立 Screen）  
  - **头像** → 设置（§4.2.1）  
- **导航隐含**  
  - 点击 cell → **详情**  
  - （文档未写但工程必需）**加入相册 / 扫描设备** 的入口：建议放在 TopAppBar 菜单或次级 CTA，避免只靠 FAB「创建」

### 1.2 照片详情 `DetailScreen`（§4.2.2）

- **一级**：全屏照片（可隐藏 chrome）  
- **二级**：**Surface Variant 信息带**（上传者、时间、尺寸、大小）  
- **三级**：横向 **操作栏**（下载、收藏、标记、删除）  
- **可展开**：标记汇总、评论列表（若功能开启）  
- **导航**：返回、左右滑切图（与 §2.1.3 交叉：切图属详情行为，IA 上属同级「堆栈内横向」）

### 1.3 创建相册引导 `CreateScreen`（§4.2.3）

- **模式**：简单（仅相册名）/ 高级（昵称、WiFi Direct / 热点）  
- **结构**：**步骤指示器** → **表单区（OutlinedTextField）** → **主按钮「创建相册」**  
- **状态**：创建中按钮 **loading + 禁用**  

### 1.4 设备列表 `DeviceListScreen`（§4.2.4）

- **页眉**：**在线人数**（如「3 人在线」）  
- **主体**：设备 **卡片列表**（头像、昵称、连接时长、上传数、最近活动）  
- **卡片右侧**：**三格信号**  
- **页脚/次级**：「邀请更多人」→ 二维码（文档可选）  
- **卡片内动作**：详情 / 移除连接 / 按设备筛图（文档写「点击卡片」承载，需二级页面或 Bottom sheet 避免一屏塞满）

---

## 2. 关键交互状态：空 / 错 / 加载（按屏补齐 §4.3）

| 屏幕 | 加载 | 空 | 错误 / 异常 | 备注（对齐文档） |
|------|------|-----|-------------|------------------|
| 首页网格 | 占位灰底 + 可选 **CircularProgressIndicator**（首屏）；分页底部 **LinearProgress**（§4.3） | 「还没有照片」+ 引导上传（§4.3）；未连接时应用名 + 明确 **下一步**（创建/加入） | Snackbar；连接失败、列表拉取失败 **重试**（§4.3） | 快速滚动暂停加载由 Glide（§4.2.1），UI 层勿再叠满屏 spinner |
| 详情 | 原图渐进（§5.1）；可用 **线性不确定进度** 或模糊占位 | 极少出现；若资源 404 → **错误文案 + 返回** | 下载失败 Snackbar；删除用 **AlertDialog**（§4.3） | 危险操作用 **红色 FilledButton**（§4.3） |
| 创建 | 按钮上 **CircularProgress** + 禁用（§4.2.3） | 不适用 | 创建失败：Snackbar 或 Dialog；**权限/热点失败** 单独文案 | 步骤间校验错误用 **OutlinedTextField error** |
| 设备列表 | 首次进入 **列表骨架或 progress** | 「暂无其他设备」+ 说明「等待加入」或「去扫描」 | 断开连接、权限导致发现失败 | 与 §2.2.1 连接状态图标（灰/黄/绿/红）联动文案 |

**连接状态与 §4.1 色彩**：未连接灰、连接中黄闪、已连接绿、异常红（§2.2.1 与 §4.1 一致）；实现时用 **M3 color roles**（见第 3 节表），不要四段纯色硬编码。

---

## 3. 与 Material 3（Compose Material3）组件映射表

| 文档元素（§4） | 建议 M3 组件 / API | 说明 |
|----------------|-------------------|------|
| TopAppBar + 标题 + 右侧图标 | `TopAppBar` / `CenterAlignedTopAppBar` + `IconButton` | 连接状态可用 **单一 IconButton** 或 `BadgedBox`（若有事件数） |
| 照片网格 3 列、4dp 间距、16dp 边距 | `LazyVerticalGrid` + `GridCells.Fixed(3)`，`horizontalArrangement`/`verticalArrangement` **4dp**，`contentPadding` **16dp** | 与 §4.1 8dp 网格兼容（4dp 为文档明确例外） |
| 缩略图占位 + 淡入 | Glide + `Crossfade`；占位用 `Surface` **tonal** | 避免纯灰块无层次 |
| FAB / 扩展「创建」 | `FloatingActionButton` / `ExtendedFloatingActionButton` | 未连接用 Extended 符合 §4.2.1 |
| 详情顶栏半透明 | `TopAppBar` + `containerColor` **scrim**（如 `MaterialTheme.colorScheme.surface.copy(alpha=0.85f)`） | 或用 M3 `LargeFlexibleTopAppBar` 视滚动行为 |
| 信息面板 Surface Variant | `Surface(color = colorScheme.surfaceVariant)` + `Column` | §4.2.2 |
| 底部操作栏 | `Row` + `IconButton` / `FilledTonalIconButton`；删除用 **error** 色 `IconButton` | 与 §4.3 危险操作红色按钮一致 |
| 标记/评论展开 | `ModalBottomSheet` 或 **可展开区域** + `HorizontalDivider` | M3 无专用「标记抽屉」，Bottom sheet 更贴手 |
| 步骤指示器 | `LinearProgressIndicator(progress)` 分段 或 **Step 行**：`AssistChip` / 小圆点 + `Text` | M3 无 Wizard；避免用 `NavigationBar` 冒充步骤 |
| OutlinedTextField | `OutlinedTextField`（`label`/`supportingText`） | §4.2.3 |
| 创建 CTA | `Button`（`Filled`） | §4.2.3 |
| 设备卡片 | `ElevatedCard` 或 `Card`（filled tonal 更贴 M3） | 圆角 **12dp**（§4.1） |
| 三格信号 | 自定义 `Row` + 3×`Box` 或 **图标** `SignalCellularAlt` 分档 tint | 与 `theme.colorScheme` 绑定 |
| Ripple | `Modifier.clickable`（默认 ripple）或 ripple 主题扩展 | §4.3 |
| 删除确认 | `AlertDialog` + `TextButton` + `Button`（error） | §4.3 |
| Snackbar | `Scaffold` + `SnackbarHost` + `SnackbarHostState` | §4.3 |
| 空状态插图 | **简单矢量 / 品牌色几何图形**（Compose `Canvas` 或轻量 SVG） | §4.3 要求品牌色；忌复杂 3D 插画 |

### 色彩映射（§4.1 → M3）

- 成功/在线 → `primary` 或 `tertiary`（二选一全应用统一），或自定义 **success** 扩展色槽（M3 默认无 success，常用 tertiary 或 semantic extension）  
- 错误/删除 → `colorScheme.error` + `onError`  
- 警告/连接中 → `errorContainer`/`onErrorContainer` 偏警示，或 **warning** 扩展；避免纯黄 `#FFEB3B` 无 on 色对比  

### 字体（§4.1）

- 使用 `MaterialTheme.typography`：`titleLarge` ≈ 24sp Medium，`bodyLarge` 16sp，`bodyMedium` 14sp 次要  

---

## 4. 容易做成「AI 味界面」的点与改写建议

| 风险点 | 典型 AI 味表现 | 改写建议（仍满足 §4） |
|--------|----------------|----------------------|
| 空状态插图 | 通用「人举手机」3D、紫蓝渐变、和相册场景无关 | 用 **1～2 个几何形 + 相册/网格隐喻**（方格 + 加号），颜色只取 **primary/secondary**，不要渐变堆叠 |
| 首页网格 | 死白底 + 高阴影卡片 + 粗圆角 24dp | 文档是 **4dp 缝 + 方缩略图**；用 **flat grid**，圆角只给 **卡片/对话框**（12dp），别给每张 cell 套 Card |
| FAB | 再加一条 `BottomAppBar` 再加大号 banner | §4.2.1 仅 FAB；**不要**叠导航栏营销条；未连接才 Extended FAB |
| 详情页 | 底部巨大渐变玻璃拟态面板 | §4.2.2 已指定 **Surface Variant**；用 **实色 + 清晰分隔**，少用 blur（性能 §5.1） |
| 设备列表 | 每张卡巨大插图、随机彩色头像 | 用 **圆形 Initials**（昵称首字）+ **统一 tonal 背景**；信号格 **细线图标**，别用五彩条形 |
| 创建引导 | 五步 wizard + 冗长欢迎语 | 文档是 **简单/高级两档**；步骤指示 **最多 2～3 步视觉**，文案 **一句辅助** 即可 |
| 状态色 | 荧光绿红按钮满屏 | 用 **M3 semantic**：在线用 **小图标 + tint**，别整页绿边；删除仅 **一个** error 主按钮 |
| 文案 | 「让我们一起记录美好瞬间吧！」 | 用 **动作导向**：「上传第一张照片」「创建相册以共享」；符合 §4.3「友好」但**像工具不像 Slogan** |
| 触摸反馈 | 到处 `Card(onClick)` 大涟漪 | §4.3 要涟漪，但 **网格 cell** 用适中 ripple，**别**整个 card elevation 跳动 |
| 图标 | Filled + Outlined 混用 | §4.1 规定 **Outlined**；全应用统一，别混 `Rounded` |

---

## 5. 评审结论（执行层）

- **IA**：在实现前画清 **未连接首页**：FAB「创建」与「加入」是否并存；若文档坚持仅 FAB，则 **TopAppBar 菜单必须承担加入**。  
- **状态**：把 §2.2.1 四种连接态写进 **设计稿与字符串表**，与 §4.1 颜色 roles 一并给开发。  
- **M3**：以 **Material3 Scaffold + TopAppBar + SnackbarHost** 为壳，网格与详情为两枚最高频屏，先定 **色型与圆角** 再铺屏，可避免 AI 模板脸。

---

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-04-13 | 初稿：与对话中 plan-design-review 输出同结构落盘 |

---

*本文档不替代 `docs_共享相册App需求文档.md`；与需求冲突时以需求文档为准。*
