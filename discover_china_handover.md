# 《山河之旅》(Discover China) 线上版项目交接与演进技术文档

## 1. 项目背景与核心规则还原（机制调研与数学建模）

### 1.1 原型溯源与核心机制纠偏

- **实体原型**：国内桌游品牌 Yaofish（鳐鳐鱼/游戏大陆）出品的《山河之旅（My First Journey / Discover China）》。
- **随机源（RNG）数学模型**：最初根据记忆推测为“从车票牌库翻牌”，经查证官方说明书与实拍图后，确认真实机制为**每回合投掷 2 颗定制六面木骰**（5 个面对应红、黄、绿、青、紫 5 种铁路颜色，第 6 个面为 `GO!` 万能车票）。这一机制从根本上消除了图论中偏远低度数节点因牌库抽干而导致的死锁（Deadlock）问题。

### 1.2 核心图论拓扑数据（Graph Topology）

- **节点（Nodes, $V=50$）**：以**北京**为公共唯一起点（0分），其余 **49 座城市**为计分目的地（按与北京的图论跳数及节点度数反比划分为 **1～4 分**四个等级，沿海枢纽 1～2 分，西北/东北/边疆 3～4 分）。
- **边（Edges, $E=83$）**：5 种颜色的无向边（`red`, `yellow`, `green`, `teal`, `purple`），含东南沿海（上海-台北-高雄-香港-澳门-海口）的海上虚线航线。
- **特殊图论特征**：
  - **度数为 1 的叶子节点**：全图仅 `呼伦贝尔`（连哈尔滨-红）与 `佳木斯`（连哈尔滨-青）为单边端点。
  - **最高度数枢纽（度数为 5）**：`济南` 与 `重庆` 各连接 5 条不同方向的铁路。

### 1.3 胜利条件与回合状态机

- **开局初始化**：洗混 49 张非北京城市牌，每名玩家暗抽 1 张作为**“秘密城市（回家终点）”**，再翻开 3 张作为**“公共目标城市”**（由于共享同一副无重复的 49 张城市牌库，秘密城市绝不会与公共目标城市发生冲突）。
- **回合状态流转**：
  1. `ROLL_DICE`：当前玩家投掷 2 颗六面骰，获得对应颜色的车票（可无限囤积保留至后续回合）。
  2. `SELECT_WILD`：若骰子投出 `GO!`，暂停并由玩家手动指定兑换为任意 1 种颜色的车票。
  3. `MOVE_PHASE`：玩家可消耗手牌中任意数量、任意组合的车票沿铁路网移动（支持单回合连续换乘或多步移动），也可选择保留车票直接结束回合。
  4. `SETTLEMENT`：移动过程中**途经或停留**在任意“公共目标城市”上，立即获得该卡分数，并将牌库顶新卡补足 3 张。
  5. `WIN_CHECK`：当玩家累计得分 $\ge 10$ 分，且移动抵达自己的“秘密城市”时，游戏立即结束并判定该玩家获胜。

---

## 2. 架构与功能迭代历程（从 2D 原型到 WebGL 真 3D 沙盘）

项目经历了 7 个主要版本的工程演进，逐步解决了状态流转、跨平台零依赖渲染、图论寻路完备性、三维空间交互遮挡以及程序化音效与新手引导问题：

| 迭代阶段 | 暴露的核心问题 / 需求 | 技术解决方案与架构演进 |
| :--- | :--- | :--- |
| **V1: 基础逻辑与 AI 托管** | 初始版本为纯同屏热座（Hot-seat），玩家 1 结束回合后玩家 2 无反应。 | 引入**启发式图论寻路 AI（Bot）**。计算当前节点到公共目标（满10分后切换为秘密城市）的无权最短跳数（BFS Distance），支持 2～4 人随时切换“真人 / AI”以及点击自定义修改玩家昵称；当全员设为 AI 时默认暂停，提供手动“开始/暂停 AI 对局”控制阀。 |
| **V2: 伪 3D ➔ 真 3D 引擎** | CSS `rotateX` 倾斜仅是 2.5D 平面透视；且引入外部 `OrbitControls.js` CDN 在国内网络下加载失败导致全屏黑屏。 | 切换为 **Three.js (WebGL) 真三维场景**（立体实木棋盘盒、车削木偶棋子、立体车站与圆柱铁轨、高空抛投物理翻滚 3D 木骰子）。**彻底移除外部控制器插件依赖**，在代码内手写内联基于球面坐标（Spherical Coordinates）的 3D 阻尼轨道摄像机控制器，配置双 CDN 容灾，并提供 **2D 平面视图 / 3D 沙盘视图一键切换**。 |
| **V3: 逐城连跳与中途抢分** | 玩家使用多张票跨城移动时，棋子直接一条抛物线飞到终点，跳过了中间城市。 | 将移动函数重构为**异步分步微步队列（Step-by-Step Hop Queue）**。棋子按路径节点序列逐个城市起跳（每跳 440ms + 110ms 停顿），每落脚一城实时扣减对应车票并检测是否顺路踩中公共目标卡（支持途经顺路抢分）。 |
| **V4: 多路径枚举与自主选路** | 标准 BFS 只保留第一条最短路径，导致存在多条同距离路径（不同车票颜色组合）或绕路顺道拿分路径时，玩家无法自主选择。 | 将寻路算法从“单最短路”升级为**受限深度无环全路径枚举（All Simple Paths Enumeration）**，按 `(耗票数升序, 顺路得分降序)` 排序。若前往某城市存在 $\ge 2$ 条路径，点击时弹出**多路线选择器**，支持鼠标悬停方案在沙盘中实时预览对应铁轨。 |
| **V5: 短边防遮挡与铁轨拾取** | 北京—天津等密集区短线路被立体棋子遮挡，看不清底下的铁轨颜色。 | 1. 适度外扩京津、长三角、珠三角密集区节点间距，并将棋子站位改为外扩扇形环绕；<br>2. 为 83 条铁轨包裹半径 `0.95` 的隐形碰撞圆柱纳入 `Raycaster`；<br>3. **鼠标悬停铁轨时**：弹出跟随鼠标的【路线颜色名称浮窗】，铁轨保持自身原色微增约 1 磅（`scale: 1.28`，绝不泛白），且**位于该铁轨两端的棋子自动变为 28% 半透明幽灵态（X-Ray 透视）**。 |
| **V6: 地图美术与海疆规范** | 早期底图偏白过曝、缺少省界；台湾岛与海南岛曾用简单椭圆代替且过滤了低纬度导致南海诸岛缺失；右上角秘密城市紫粉配色过于艳俗。 | 1. **4096×4096 离屏 Canvas 矢量绘本渲染**：采用 `MeshLambertMaterial` 消除高光过曝，叠加阿里云 DataV 官方 `100000_full.json` 34 省真实测绘省界与分省复古重彩设色，手绘矢量布达拉宫、万里长城、天坛、熊猫、火锅及白边模切地标贴纸；<br>2. **海疆精确测绘**：精确绘制台湾本岛、澎湖、钓鱼岛、雷州半岛、海南本岛（含岛内分区界线与海峡省界）及右下角标准**《南海诸岛》十段线附图框**；<br>3. **UI 克制化**：右侧采用复古羊皮纸与深胡桃木配色，秘密城市卡槽改为克制的古典深木配哑光铜金边框，左右信息框严格等高（`58px`）对齐。 |
| **V7: 实时合成音效与新手引导** | 缺乏听觉反馈与新玩家上手指引；原生 `alert`/`prompt` 在沙箱或嵌入式容器中易被拦截。 | 1. **Web Audio API 零依赖合成器（`SoundEngine`）**：实时合成国风五声音阶背景音乐（BGM）、木骰翻滚碰撞声、`GO!` 万能票闪烁琶音、高铁双音鸣笛与气流疾驰声、棋子连跳逐级升调落子声、1～4 分目标卡递增得分声、满 10 分达标号角声、以及真人获胜/AI抢先夺冠（失败）专属结算旋律，支持顶部独立开关与内置**音效试听台**；<br>2. **游戏说明弹窗与分步高亮引导（Onboarding Tour）**：首次打开自动触发 5 步聚光灯镂空指引（写入 `localStorage` 记忆状态，并支持随时重播），内置三页签《游戏说明》弹窗，并将改名、终局胜负结算全面替换为古风页内弹窗。 |

---

## 3. 代码模块结构字典（供后续开发快速定位）

当前交付物为单文件零构建依赖的 `discover_china_3d.html`，内部核心模块划分如下：

### 3.0 实时合成音效引擎与新手引导模块（Section 0 & 0.5）

- `SoundEngine`：基于浏览器原生 `AudioContext` 的程序化音频合成器，无需加载任何外部 `.mp3/.wav` 文件：
  - `startBGM()` / `stopBGM()`：以中国传统五声音阶（宫商角徵羽 C-D-E-G-A）驱动的古筝/木琴拨弦包络与低音伴奏循环。
  - `playDiceRoll()` / `playWildGo()`：模拟双木骰多次弹跳碰撞及 `GO!` 彩虹万能票和弦。
  - `playTrainDepart()`：双振荡器差频模拟高铁风笛（Sawtooth + Lowpass）叠加带通滤波白噪声（Bandpass White Noise）模拟列车出站气流声。
  - `playPawnHop(stepIdx)`：起跳正弦滑音 + 400ms 后木质棋子落盘低频敲击声，且在多站连跳时按 `stepIdx` 阶梯式提升音高。
  - `playScoreGain(scoreVal)` / `playScoreGoalReached()`：按 1～4 分动态扩展琶音阶数；当玩家得分首次从 `<10` 跨越至 `>=10` 时触发达标铜管号角。
  - `playVictory()` / `playDefeat()`：根据获胜方为真人玩家或 AI 对手，分别演奏凯旋交响或惜败下行旋律。
- `openRulesModal(defaultTab)` / `switchRulesTab(tabName)`：管理三页签游戏说明弹窗（核心规则、操作技巧、音效试听台）。
- `startOnboardingTour(stepIdx)` / `renderOnboardingStep()`：基于 DOM `getBoundingClientRect()` 动态计算聚光灯高亮框（`#tour-spotlight`）与说明卡片（`#tour-card`）坐标的 5 步交互式新手引导。
- `showGameOverModal(winnerPlayer)` / `resetGameSession()`：页内终局积分榜结算面板与无刷新内存状态重置。

### 3.1 静态数据与坐标投影（Section 1）

- `NODES`：50 座城市的对象数组，字段包含 `{ id, name, x, y, score, isStart, icon, sticker, stickerOffset, region, wx, wz }`。
- `EDGES`：83 条铁轨连接数组，字段包含 `{ u, v, color, dashed }`。
- `mapTo3D(x, y)` / `mapToCanvas(x, y)`：将 `[80..860, 110..940]` 的棋盘平面坐标映射到 Three.js 世界坐标 `[-37..37]` 与 4096×4096 贴图画布坐标。
- `lngLatToCanvas(lng, lat, provinceName)`：带台湾省、海南省局部锚定校准的经纬度投影函数，确保官方 GeoJSON 省界与 50 座城市坐标严丝合缝。

### 3.2 4K 离屏纹理渲染引擎（Section 2）

- `renderFullBoardCanvas(canvas, geoJsonData)`：主渲染入口，依次绘制松石蓝渐变海洋、邻国大陆、中国大陆底板、台湾与海南本岛及岛内分界（`drawTaiwanAndHainanWithProvinceBorders`）、34 省 GeoJSON 分省重彩图斑、沙漠湖泊、森林动物、定制矢量地标（`drawVectorPotala` / `drawVectorBeijingGreatWall` 等）、8 角航海罗盘（`drawOriginalCompass`）及右下角标准《南海诸岛》十段线附图框（`drawStandardSouthChinaSeaInset`）。

### 3.3 WebGL 场景、相机与射线拾取（Section 3 ~ 5）

- `camState` & `updateCameraPosition()`：手写内联球面坐标阻尼摄像机。
- `toggle2D3DMode()`：一键切换 2D 平面正射俯视（锁定旋转，左键直接平移，压扁棋子与骰子，隐藏悬浮晶石）与 3D 沙盘视图。
- `build3DRailsAndStations()`：构建 3D 车站圆台、彩色圆柱铁轨、木枕木以及半径 `0.95` 的透明铁轨拾取碰撞盒（`userData.isEdge = true`）。
- `applyEdgeSubtleHighlight(mesh, active)`：控制铁轨悬停或路径预览时保持自身原色微增 28%（约 1pt），绝不泛白。
- `setPawnsGhostMode(uId, vId)`：当鼠标悬停在某段铁轨上时，将该铁轨两端城市的棋子透明度设为 `0.28` 以防遮挡。

### 3.4 游戏状态机与全路径寻路算法（Section 6 ~ 7）

- `computeReachable(startNode, tickets)`：受限深度无环全路径枚举算法，返回每个可达城市的所有合法路线方案及途经得分统计。
- `openRoutePicker(targetNodeId, routes)`：当存在多条可行路线时弹出方案选择框，支持悬停实时预览。
- `executeNodeMove(routeObject, onAllStepsDone)`：逐城分步跳跃状态机，处理单步车票扣减、抛物线插值动画、途经公共目标即时拿分、10 分达标号角与回家胜利判定。
- `editPlayerName(idx)` / `toggleAI(idx)` / `toggleAllAIAutoRun()`：处理玩家自定义改名、真人/AI 身份切换以及全员 AI 时的暂停保护与手动启停。

---

## 4. GitHub 仓库推送与 GitHub Pages 部署指南

由于当前云端沙箱环境出于安全隔离策略禁用了外网连接且未挂载个人 GitHub 凭证，无法在容器内直接执行远端 `git push`。下载 `discover_china_3d.html` 与 `discover_china_handover.md` 到本地后，可通过以下任一方式快速推送并上线：

### 方式一：本地命令行一键推送（推荐）

将下载的两个文件放入本地项目文件夹（建议将主页命名为 `index.html` 以便直接启用 GitHub Pages），在终端执行：

```bash
git init
cp discover_china_3d.html index.html
git add index.html discover_china_3d.html discover_china_handover.md
git commit -m "feat: add Web Audio synthesizer, onboarding tour, and rules modal"
git branch -M main
git remote add origin https://github.com/<你的GitHub用户名>/<你的仓库名>.git
git push -u origin main
```

### 方式二：GitHub 网页端直接拖拽上传（零命令行）

1. 在 GitHub 新建仓库后，点击页面上的 **Add file ➔ Upload files**。
2. 将 `discover_china_3d.html`（可重命名为 `index.html`）与 `discover_china_handover.md` 直接拖入浏览器窗口，点击 **Commit changes** 即可完成上传。
3. 进入仓库 **Settings ➔ Pages**，在 **Build and deployment** 下选择 `Deploy from a branch`（分支选 `main` / `/ (root)`），保存后约 1 分钟即可获得可公开分享的在线游玩链接。
