# 科技红轮动策略 · 创业板指 vs 红利低波 指数轮动监控

数据经自建的 [a-stock-api](https://github.com/wj624/a-stock-api) Worker 取数（不再直连 AKShare），对 **创业板指（399006）** 与 **红利低波（H30269）** 做相对强弱轮动监控，
输出一份自包含 HTML 看板，图形与数据全部内联，断网也能看。

- **在线看板（点开即用）**：https://wj624.github.io/rotation-monitor/
- **看板中心（已作为第 5 个面板集成）**：https://wj-dashboards.pages.dev/
- **原始项目（上游 wwww1998）**：https://wwww1998.github.io/rotation-monitor/
- **原始源码**：https://github.com/wwww1998/rotation-monitor
- **本仓库源码**：https://github.com/wj624/rotation-monitor

**兄弟看板（同一套监控体系）**：

- ETF + LSI：https://etf-lsi.pages.dev/
- 国家队 ETF 资金流：https://wj624.github.io/national-team-etf-flow/
- EarlETF 图表周刊：https://wj624.github.io/earl-etf-dashboard/
- 美股 / 世界经济温度计：https://wj624.github.io/us-world-economy-dashboard/

**一句话原理**：用两指数的**相对强弱（动量）**判断风格切换——创业板相对红利低波走强时持有创业板（进攻），
转弱则切到红利低波（防御），本质是「追强弃弱」的动量轮动。

> 本仓库直接复制自 [wwww1998/rotation-monitor](https://github.com/wwww1998/rotation-monitor)，保留原作者署名。
> 改动集中在「怎么让它稳定、省资源地每天自动跑下去」，监控口径与原文一致。

---

## 一、监控什么

- **创业板指（399006）**：高弹性成长风格代表
- **红利低波（H30269）**：低波动、高股息的防御风格代表
- **轮动信号**：两指数近 N 日相对涨跌幅（动量），谁强持谁

## 二、页面有哪些内容

打开在线看板，从上到下依次是：

| 模块 | 说明 |
| --- | --- |
| **相对强弱曲线** | 创业板指 ÷ 红利低波的归一化比值，上行 = 成长占优 |
| **轮动信号状态** | 当前持有哪一头、信号触发阈值 |
| **两指数 K 线对照** | 创业板指 / 红利低波各自走势，悬停看当日值 |
| **每日数据明细** | 最新收盘价、涨跌幅、相对强弱值 |

## 三、数据来源

| 数据 | 来源 | 取数方式 |
| --- | --- | --- |
| 创业板指（399006）日线 | 腾讯行情（经 Worker 代理） | `GET /api/kline?symbol=sz399006&start=2010-01-01` |
| 红利低波（H30269）日线 | 中证指数官网（经 Worker 代理） | `GET /api/csindex?symbol=H30269&start=20160803` |

- Worker 基址：`https://a-stock-api.wj4020410.workers.dev`（详见 [a-stock-api](https://github.com/wj624/a-stock-api)）。
- **H30269 必须走 `/api/csindex`**：它是中证自定义指数，腾讯没有日线，走 `/api/kline` 会拿到空数组。
- 2026-10 之前用的是 AKShare（`stock_zh_index_daily` 等），已全部替换；换源后**口径未变**，历史读数可比。
- Worker 侧自带 KV 缓存（1 小时），本仓库不再额外缓存。

## 四、自动运行

由 GitHub Actions 工作流 `.github/workflows/daily_update.yml` 驱动，在 `ubuntu-latest` + Python 3.10 上：
装依赖 → 跑 `generate_page.py` → 把 `public/index.html` 提交回仓库并部署到 GitHub Pages。

**当前调度：每天 1 次**（cron 写的是 UTC，北京时间 = UTC+8）。

```yaml
- cron: "0 16 * * *"    # 16:00 UTC，每天一次
```

- **手动触发**：仓库 → Actions → 每日更新监控数据 → Run workflow。
- 运行后 `public/index.html` 自动提交回 `main`，GitHub Pages 重新部署。

## 五、本地运行

```bash
pip install -r requirements.txt
python main.py        # 本地 Web 服务，访问 http://localhost:8000
# 或仅生成页面：
python .github/workflows/generate_page.py
```

## 六、文件说明

```
.github/workflows/daily_update.yml    GitHub Actions 定时刷新工作流
.github/workflows/generate_page.py    抓取 + 计算 + 生成看板的全部逻辑
main.py                               Render / 本地 Web 服务入口
requirements.txt                      依赖
public/index.html                     已发布的自包含看板（工作流自动生成并提交）
README.md                            本文件
```

## 七、给 AI 的维护提示词

> 把 §7.3 的代码块整段复制给 AI，它就能在不了解背景的情况下接手维护。
> §7.1 / §7.2 是给人看的「哪里容易改漏」，建议先过一眼。

### 7.1 仓库速览

| 项 | 内容 |
| --- | --- |
| 用途 | 创业板指 vs 红利低波 的相对强弱轮动监控，产出**单文件自包含 HTML** 看板 |
| 抓数 + 计算 + 渲染 | `.github/workflows/generate_page.py`（约 59 KB，全流程都在这一个文件里） |
| Web 服务入口 | `main.py`（FastAPI，本地或 Render 用，**不参与** Actions 出图） |
| 构建产物 | `public/index.html`（约 3 MB，图表已内联），**由工作流自动提交回 main** |
| 调度 | `daily_update.yml`，`cron: "0 16 * * *"`（UTC 16:00 = 北京 00:00） |
| 兜底 | `watchdog.yml`，工作日北京时间 15:00–23:40 每 20 分钟检查一次页面是否已刷新 |
| 依赖 | `fastapi uvicorn pandas numpy requests`（无 akshare） |
| 密钥 | **不需要任何 Secret**（Worker 免鉴权，Pages 用内置 token） |

### 7.2 改代码前必须知道的三个坑

1. **主逻辑不在根目录。** `generate_page.py` 放在 `.github/workflows/` 里面（历史遗留）。在根目录 `grep` 是找不到的。
2. **策略阈值有两份，必须同步改。** `generate_page.py` 第 21–22 行（`BUY_THRESHOLD = 0.20` / `SELL_THRESHOLD = 0.40`）与 `main.py` 第 24–25 行各写了一套。只改一处 → 网页版和 Web 服务版会给出**互相矛盾**的轮动信号，而且不报错。
3. **`public/index.html` 是产物，别手改。** 它每天被工作流覆盖并提交回仓库，供 EdgeOne Pages 检出。要改样式/文案请改 `generate_page.py` 里的模板字符串。

### 7.3 提示词（复制这段给 AI）

```text
你在维护 GitHub 仓库 wj624/rotation-monitor —— 一个「创业板指(399006) vs 红利低波(H30269)
相对强弱轮动」监控看板。先读 README.md（尤其是「三、数据来源」与「七、给 AI 的维护提示词」）。

【结构】
- 全部逻辑在 .github/workflows/generate_page.py（注意：藏在 .github/workflows/ 下，不在根目录）：
  抓数 → 算相对强弱/回测 → matplotlib 出图转 base64 → 渲染成自包含 public/index.html。
- main.py 是 FastAPI Web 服务（本地/Render 用），不参与 Actions 出图。
- .github/workflows/daily_update.yml 每天 UTC 16:00 跑一次；watchdog.yml 做当日兜底。

【数据】
- 唯一数据源是自建 Worker：https://a-stock-api.wj4020410.workers.dev
- H30269（红利低波）必须用 /api/csindex?symbol=H30269&start=20160803
  —— 它是中证自定义指数，腾讯没有日线，走 /api/kline 会拿到空数组。
- 399006 用 /api/kline?symbol=sz399006&start=2010-01-01
- 不要再引入 akshare 或直连腾讯/中证；取数失败时应整体失败或沿用旧页面，不要用半截数据出图。

【改动的铁律】
1. 策略阈值有两套：generate_page.py 第 21-22 行、main.py 第 24-25 行（BUY 0.20 / SELL 0.40）。
   要改就两处一起改，否则网页版与 Web 服务版信号会不一致且不报错。
2. 不要手改 public/index.html，它是构建产物，每天会被覆盖。
3. 两个指数必须都取到数据才能算比值。任一为空就应当中止并保留上一版页面，
   绝不能用 0 或前一日数据顶替（会比停更更危险）。
4. 看板遵循 A 股配色：涨红跌绿。

【验证】
- 本地：pip install -r requirements.txt && python .github/workflows/generate_page.py
  然后打开 public/index.html，确认「更新于」是今天、两条曲线都有值、信号状态与比值自洽。
- 云端：Actions 里手动跑「每日更新监控数据」，看日志无异常、且 public/index.html 有新提交。
- 注意：沙盒/部分 CI 网络访问不到 *.workers.dev，验证 Worker 可改用
  https://github.com/wj624/a-stock-api 的 README §9（读 KV 缓存值）或让用户在浏览器里打开端点 URL。

【已知问题（修之前先确认，别当新 bug）】
- .github/workflows/watchdog.yml 目前检查的是 **上游仓库 wwww1998/rotation-monitor** 的页面，
  并试图 dispatch **上游仓库** 的 workflow。这是复制上游代码留下的残留，GITHUB_TOKEN 也无权
  触发别人仓库，因此对本仓库的兜底实际无效。若要修，把其中的 wwww1998/rotation-monitor
  改成本仓库 wj624/rotation-monitor 即可。

【收尾】
- 改了数据源/口径/阈值，必须同步更新 README 的「一、监控什么 / 三、数据来源 / 四、自动运行」。
```

## 免责声明

本看板仅用于数据观察与学习，**不构成任何投资建议**。轮动信号为动量口径的历史统计，
不代表未来走势，请独立判断。
