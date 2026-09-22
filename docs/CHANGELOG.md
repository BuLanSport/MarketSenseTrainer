# 开发记录

## 2026-09-22 — 文件名英文化

为便于跨平台 / 命令行 / 版本控制使用，中文文件名全部改为英文。项目根目录亦由 `盘感训练器/` 改为 `MarketSenseTrainer/`。

| 原名 | 新名 | 备注 |
|---|---|---|
| `盘感训练器.html` | `index.html` | 符合静态站入口惯例，双击行为不变 |
| `使用说明.txt` | `README.txt` | |
| `docs/开发记录.md` | `docs/CHANGELOG.md` | |
| `game-data.js` | `game-data.js` | 本就是英文，未动 |

**关键约束**：`index.html` 内唯一的外部引用是 `src="game-data.js"`，该文件名未变，因此 HTML 内容**零改动**（git 记录重命名相似度 100%）。

---

## 版本 v1.0 — 2026-09-22

首个可玩版本。

### 交付内容

| 文件 | 大小 | 说明 |
|---|---|---|
| `index.html` | 46.7 KB | 单文件游戏本体，内联 CSS + JS |
| `game-data.js` | 319.1 KB | 27 只 A 股 × 250 根日 K |
| `README.txt` | 3.8 KB | 玩家向说明 |

### 数据层

- 数据源：`westock-data-skillhub`（前复权日 K）。
- **单次拉取上限 250 根日 K**，因此每只股票取 250 根。
- 区间：2025-09-11 → 2026-09-22。
- 股票池 27 只，覆盖金融 / 消费 / 科技 / 新能源 / 医药 / 周期 / 农业。
- 字段结构：`[日期, 开盘, 最高, 最低, 收盘, 成交量]`。

### 游戏设计

- **逐日推进**：玩家点一次「过」前进一天，不是自动播放。
- **匿名标的**：显示为 `SIM-XXXX`，结算后才揭示真实代码与名称。
  - 理由：知道标的会让玩家调用记忆而非判断，练的是背题不是盘感。
- **三动作**：买 / 卖 / 过。单一持仓，全仓进出，不支持分批。
- **零未来函数**：任何时点只能看到已发生的 K 线。

### 三轮修复（按发生顺序）

#### 修复 1：择时评分基准口径错误（最严重）

**症状**：随机乱点居然跑出 **+1.10% 的 alpha**，且只有 41.2% 的概率跑输"开局躺平"。

**病根**：原基准是「开局买入持有」。但随机策略 75% 的时间处于空仓，而**空仓在一只不涨的票上会自动跑赢满仓**。于是评分系统系统性地奖励"少动手"这个伪技巧，而不是奖励判断力。

**修复**：改用**暴露匹配基准（exposure-matched benchmark）**：

```javascript
var totalDays = S.idx - WARMUP + 1;
var holdDays = 0;
S.trades.forEach(function(t){
  var end = (t.exitIdx !== null) ? t.exitIdx : S.idx;
  holdDays += (end - t.entryIdx);
});
var exposure = totalDays > 0 ? clamp(holdDays / totalDays, 0, 1) : 0;
var baseMatch = baseRet * exposure;   // 按持仓占比折算的基准
var alpha     = ret - baseMatch;      // 择时超额
```

**验证**（三项自洽检验全部通过）：

| 检验 | 期望 | 实测 |
|---|---|---|
| 全程观望 alpha | 0.00% | ✅ 0.00% |
| 买入持有 alpha | ≈ 0 | ✅ ≈ 0（最大偏离 0.365%） |
| 随机交易 alpha（1080 样本） | 0，正负对称 | ✅ 均值 −0.05%，负占比 51.6% |

#### 修复 2：K 线随进度被压扁

**症状**：推进越深，K 线越细，最后形态完全看不清。

**病根**：`render()` 用 `S.bars.slice(0, S.idx+1)` 把**全量历史**塞进固定宽度图 → 单根宽度 = 图宽 ÷ K线数。这是把"无未来函数"错误实现成了"显示全部历史"。

**修复**：改为**滑动窗口**，只渲染最近 N 根：

```javascript
var VISIBLE_BARS = 70;
function render(){
  var all = S.bars.slice(0, S.idx+1);
  var start = Math.max(0, all.length - VISIBLE_BARS);
  var bars  = all.slice(start);
  // MA 必须用全量历史算完再切窗，否则窗口左端的 MA 会错
  var ma5All = [], ma20All = [];
  for(var i=0;i<all.length;i++){ /* 全量计算 */ }
  var ma5  = ma5All.slice(start);
  var ma20 = ma20All.slice(start);
  // 标记点/阴影/成交量做 -start 索引平移，并判断是否落在窗内
}
```

补充：K 线加 `barWidth:'62%'`，并新增 40 / 70 / 110 根三档切换。

**验证**：窗口填满后单根宽度锁定 **17.1px**，第 21 天与第 103 天的宽度波动比 **1.0000x**（完全一致）。

#### 修复 3：布局按手机设计，电脑上过小

**症状**：大屏上缩成一条窄柱，K 线看不清。

**病根**：`.app{max-width:520px}` 锁死宽度。

**修复**：改为 CSS Grid 响应式双列：

```css
.app{
  max-width:1680px;
  display:grid;
  grid-template-columns:1fr 340px;
  grid-template-areas:"topbar topbar" "stats stats" "chart side" "log side";
}
@media(max-width:900px){
  .app{grid-template-columns:1fr;
    grid-template-areas:"topbar" "stats" "chart" "side";}
}
```

配套：K 线高度 `clamp(360px, 52vh, 620px)`；新增成交量副图（双 grid / 双 xAxis / 双 yAxis / 4 series）；加 dataZoom；字号按 `window.innerWidth` 动态调整；结算弹层 460px → 780px。

### 评分体系

| 维度 | 算法 |
|---|---|
| 买点评分 | `(区间最高 − 买入价) / (区间最高 − 区间最低) × 100`，窗口取买入后 `LOOKAHEAD` 根 |
| 卖点评分 | `100 − (卖出价 − 区间最低) / (区间最高 − 区间最低) × 100` |
| 扛回撤舒适度 | `100 + 最大浮亏% × 5`，clamp 到 [0,100] |
| 择时超额 | 见修复 1 的暴露匹配公式 |

### 跨局累计

- `localStorage` key：`pangan_trainer_cum_v1`。
- 幂等标志 `S._saved` 防止同一局重复写入。
- `newGame()` 中重置 `S._saved = false`。

### 验证方法（无浏览器环境）

本机 `agent-browser` 需下载约 500 MB Chromium、`npm install jsdom` 被沙箱拦截，因此自建了一套验证方案：

**Node + 最小 DOM stub 执行真实游戏代码。**

- 用 `node --check` 校验内联 JS 语法。
- 用 DOM stub（`mkEl` / `_ev` 事件表 / `document.getElementById` 映射）加载真实游戏脚本，跑 500 局模拟，零异常。
- 用真实玩家路径连打 8 局 / 12 局，校验累计记录条数正确。
- 用宽度恒定测试验证滑动窗口（波动比 1.0000x）。

### 踩坑记录

| 问题 | 原因 | 修复 |
|---|---|---|
| 累计统计首局不显示 | `showReview()` 里先渲染后写档 | 改为先 `saveCum()` 再渲染 |
| 同一局记录写两次 | `revealAll` → `btnBack` 重新触发 `finish(true)` | 加 `S._saved` 幂等标志 |
| 多局连打测试只记 1 条 | 测试脚本点了开始页的 `btnStart`，它不关 review 遮罩 | 改用真实路径 `btnAgain` |
| DOM stub 无法触发点击 | `addEventListener:()=>{}` 是空函数 | 改为 `el._ev={}` + 真实记录回调 |
| Node 报 `.format is not a function` | JS 里误用 Python 的 `.format()` | 改用 `String.padEnd()` |

**教训：测试必须走真实用户路径。**

### 环境备注

- PowerShell 在本机不回显 stdout → 命令结果一律 `Out-File` 落盘再读。
- PowerShell `Remove-Item` 删不掉工作区内临时文件（静默失败）→ 清理改用 Python `os.remove`。
- Bash 工具 coreutils 缺失（`ls`/`dirname`/`head` 全不可用）→ 文件操作一律 PowerShell 或专用工具。
