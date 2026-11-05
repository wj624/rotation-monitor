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
- **轮动信号**：以「创业板指 ÷ 红利低波」归一化比值对照固定阈值切换（`< BUY_THRESHOLD=0.20` 持创业板、`> SELL_THRESHOLD=0.40` 持红利低波）；阈值在 `generate_page.py` 第 21–22 行与 `main.py` 第 24–25 行各写一份，必须同步改

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
| 创业板指（399006）日线 | a-stock-api Worker（腾讯行情） | `GET /api/kline?symbol=sz399006&start=2010-01-01` |
| 红利低波（H30269）日线 | a-stock-api Worker（中证指数官网） | `GET /api/csindex?symbol=H30269&start=20160803` |

- Worker 基址：`https://a-stock-api.wj4020410.workers.dev`（详见 [a-stock-api](https://github.com/wj624/a-stock-api)）。
- **H30269 必须走 `/api/csindex`**：它是中证自定义指数，腾讯没有日线，走 `/api/kline` 会拿到空数组。
- 2026-10 之前用的是 AKShare（`stock_zh_index_daily` 等），已全部替换；换源后**口径未变**，历史读数可比。
- Worker 侧自带 KV 缓存（1 小时）；此外 `main.py` 还有进程内缓存（`_cache` + `CACHE_TTL = 300` 秒），5 分钟内复用同一份结果。

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
| 抓数 + 计算 + 渲染 | `.github/workflows/generate_page.py`（全流程都在这一个文件里） |
| Web 服务入口 | `main.py`（FastAPI，本地或 Render 用，**不参与** Actions 出图） |
| 构建产物 | `public/index.html`（图表已内联），**由工作流自动提交回 main** |
| 调度 | `daily_update.yml`，`cron: "0 16 * * *"`（UTC 16:00 = 北京 00:00） |
| 兜底 | `watchdog.yml`，工作日北京时间 15:00–23:40 每 20 分钟检查一次页面是否已刷新（现已指向本仓库 `wj624/rotation-monitor`） |
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

【已修复：watchdog 现已指向本仓库】
- .github/workflows/watchdog.yml 原先检查的是 **上游仓库 wwww1998/rotation-monitor** 的页面，
  并试图 dispatch **上游仓库** 的 workflow。这是复制上游代码留下的残留，GITHUB_TOKEN 也无权
  触发别人仓库，因此对本仓库的兜底当时并未生效（这正是原「已知问题」，修之前先确认，别当新 bug）。
  现已把其中的 wwww1998/rotation-monitor 改成本仓库 wj624/rotation-monitor，兜底恢复生效。

【收尾】
- 改了数据源/口径/阈值，必须同步更新 README 的「一、监控什么 / 三、数据来源 / 四、自动运行」。
```

## 免责声明

本看板仅用于数据观察与学习，**不构成任何投资建议**。轮动信号为动量口径的历史统计，
不代表未来走势，请独立判断。

---

## 附录 · 2026-11-05 全量整改记录（给维护者 / 给 AI 读）

> **这一节是什么**：AI 维护助手在 2026-11-05 对账号下 6 个仓库做完一次「大范围彻底修复」后追加的说明。目的只有一个——**让下一次维护（不管人或 AI）能看懂每个改动为什么做、哪些绝对不能回退**。
>
> **怎么读**：改任何代码之前，先读本 README 前文的「维护须知 / 铁律 / 给 AI 的维护提示词」，再读本附录。**两者冲突时以前文铁律为准**；本附录只解释本次改动，不新增、不覆盖任何设计口径。
>
> **本次原则**：用户要求「稳定、基本无维护、占用空间少、不自动生成垃圾」，并明确「修改前先看 README 的设计思路与维护须知」。因此本次只修**确定性 bug、静默失败、危险工作流行为、文档与代码不一致、依赖版本失控、运行时垃圾文件**；**没有做架构改造，没有删任何数据资产，没有改任何口径/阈值/单位/cron/构建顺序**。
>
> **本次改动范围**：6 仓共 40 个文件、0 删除；逐仓白名单精确提交，全程无 `--force`、无 rebase 冲突。**上线后的串行补测又追加了一次 pandas 3.x 兼容性修复（3 仓 6 文件，见 F 节），所以最终线上状态以 F 节为准，A 表里凡是写 `pandas<4` 的地方都已收紧为 `pandas<3`**。

### A. 本次改了什么（逐文件）

| 文件 | 原始缺陷（为什么必须改） | 本次修法 | 依据 / 约束 |
|---|---|---|---|
| `.github/workflows/generate_page.py`（10 处） | `run_backtest*` 返回 3 值但 2 处调用按 2 值解包（**必崩**）；主流程 7 处 `print("ERROR") + return`（列名无法识别、数据不足）**静默失败**——CI 显示成功却不更新页面；最大回撤写死 `-37.4%`；页脚数据来源写 AKShare | 3 值返回与 2 处调用点解包对齐；7 处改 `raise RuntimeError`；最大回撤用真实 `max_dd` 计算；页脚改「自建 a-stock-api Worker」 | 「任一指数取不到数据应中止并保留上一版页面，绝不能用 0 或前一日数据顶替」；`generate_page.py` 在 `.github/workflows/` 下 |
| `.github/workflows/generate_page.py`（补测二次修复） | `resample('M')` 在 pandas 3.1.1 抛 `ValueError: 'M' is no longer supported`，第一次补测整轮失败 | 改 `resample('ME')`（pandas 2.3.3 / 3.0.0 语义一致，本地实测输出一致） | 见 F 节；**不要改回 `'M'`** |
| `requirements.txt` | **缺 `matplotlib`**，但 `generate_page.py` 会 import，README 给的本地命令必然失败；其余依赖无上界 | 补 `matplotlib>=3.7,<4`；pandas 锁 `>=2.0.0,<3` 并加注释说明 3.x 移除 `resample('M')`、Copy-on-Write、字符串 dtype 变更；其余加上界 | 五、本地运行；见 F 节 |
| `.github/workflows/daily_update.yml` | 无数据有效性守卫、Python 版本未固定；`pip install` 行完全无上界，补测时装到 pandas 3.1.1 | 插入守卫；固定 Python 3.11；pip install 改 `pandas>=2.0.0,<3` `numpy<3` `matplotlib<4` `requests<3`，与 requirements.txt 同步（见 F 节） | 阈值两份同步的设计；线上跑的是 workflow 的 pip install |
| `.github/workflows/watchdog.yml`（2 处） | 硬编码**上游仓库** `wwww1998/rotation-monitor`，对本仓兜底无效（README §7.3 已标为已知残留） | 改指本仓 `wj624/rotation-monitor` | §7.3 明确授权修改；`curl` 已有 `--max-time 60`，早期报告误判「无超时」，本次未动 |
| `.github/workflows/monthly-snapshot.yml` | 每月自动 force 重写历史 | **停用 `schedule`**，保留 `workflow_dispatch`；guard 增加 `*.json` 校验；回滚 SHA `retention-days` 90 → 365 | 不删文件、不改主体 |
| `EDGEONE_DEPLOY.md` | 仓库名仍是上游；`portal.html` 应为 `public/index.html`；「14 次/天」与「5 个看板仓库」等数字错误 | 逐项订正（每日 1 次；watchdog 是工作日每 20 分钟检查、最多 27 次检查；看板数按同文清单为 4 个） | 部署文档 |
| `README.md` | 策略描述、数据来源（AKShare）、缓存描述过时；§7.3 已知问题未标已修复；「约 59 KB / 3 MB」等易变数字 | 改成比值阈值 `BUY 0.20 / SELL 0.40`、来源 `a-stock-api Worker`、缓存 `_cache + CACHE_TTL=300`；§7.3 标注已修复并保留原因说明；删过时数字 | 七、给 AI 的维护提示词 7.1 / 7.2 |

### B. 本次坚决没改（硬约束，改了就是破坏设计）

- **`BUY_THRESHOLD = 0.20` / `SELL_THRESHOLD = 0.40` 两份（`generate_page.py` 与 `main.py`）必须同步，本次未改数值**。
- **`generate_page.py` 在 `.github/workflows/` 下**（不在仓库根目录）。
- **`public/index.html` 是每天工作流提交回 `main` 的产物，供 EdgeOne Pages 检出**：别手改、别按「产物不入库」处理。
- **H30269 走 `/api/csindex`，399006 走 `/api/kline`**；任一指数取不到数据应中止并保留旧页面。
- `watchdog.yml` 的 `curl` 已有 `--max-time 60`，未改。
- **pandas 必须 `<3`、`resample` 用 `'ME'`**（F 节），且 requirements.txt 与 daily_update.yml 两处同步 —— 第一次补测失败的根因就是漏改 workflow 的 pip install 行。

### C. 明令作废的旧建议（别再走回头路）

第一轮「只读体检报告」里提出过若干架构建议，**读完各仓 README 的维护须知后已全部作废**，本次一个都没采纳：

- ❌ **「产物不入库 / 数据与页面不进 git」**：本仓产物是 Pages / EdgeOne Pages / GAS 的**数据源本身**，从仓库检出，必须入库。
- ❌ **「删 `cache/` / 改回 `actions/cache`」**：`cache/` 是断网降级与历史累积的命根子。
- ❌ **「删数据资产 / 删摘要文件 / 清理 `echarts.min.js` / 删 `akshare` / 删 `westock-data`」**：均被 README 明确点名保护。
- ❌ **「把 `except: pass` 全改抛错」**：局部容错（TTL 状态、日志编码、临时文件清理、上游回退链）是设计，不是缺陷；本次只把**主流程**的静默失败改成显式失败。
- ❌ 早期报告曾把 GAS 脚本里的 `RECEIVER_EMAIL` 硬编码列为 bug —— **不是 bug**，它是 `CONFIG` 里明确列出的默认配置项，本次未改。

> 若下次 AI 读到 `wj624-仓库体检与整改思路.md` 或 `wj624-整改落地手册.md` 里的 P1/P2 建议，请以本附录为准：**那两份是整改前的中间产物，已作废**。


### D. 给 AI 的维护边界（可复制这段）

```
你在维护 wj624/rotation-monitor。动手前：
1. 先读 README 七、给 AI 的维护提示词（7.2 三个坑），再读本附录。
2. 绝对不要：把 public/index.html 当「产物不入库」清掉；手改 public/index.html；
   只改一处阈值（generate_page.py 与 main.py 两份必须同步）；用 0 或前一日数据顶替缺失指数；
   把 generate_page.py 挪出 .github/workflows/；恢复 monthly-snapshot 自动 schedule。
3. 两个指数必须都取到数据才算比值；任一为空就中止并保留上一版页面。
4. 本附录 A 表已修的 3 值解包、7 处静默失败、写死 -37.4%、watchdog 指向上游，别回退。
5. pandas 锁 <3、resample 用 'ME' 不要改回 'M'；改依赖必须 requirements.txt 与 daily_update.yml 两处同步（F 节）。
6. 改了数据源/口径/阈值，同步 README「一、监控什么 / 三、数据来源 / 四、自动运行」+ 两份阈值常量。
7. 线上写操作：精确 git add、中文 commit、git pull --rebase、绝不 force。
```
### E. 验证与回滚

| 项目 | 说明 |
|---|---|
| 本地验证 | 补丁脚本在从原仓库 `git archive HEAD` 导出的**干净副本**上端到端重跑：59/59 替换、2 处追加、2 个新建 `.gitignore`、5 个 `monthly-snapshot.yml` 降级；`.py` 用 `ast.parse`、`worker.js` 用 `node --check`、`.yml` 用 `yaml.safe_load`，语法自检 26 项全 PASS |
| README 硬约束复核 | 61 项，实际 61/61 通过（初检 5 项 FAIL 均为检查规则误报） |
| 线上内容核验 | GitHub API 独立核验：commit 主题、变更文件集合与白名单逐仓一致、**0 删除**、关键修复内容、数据资产 12/12 在线 |
| 线上实测 | 2026-11-05 串行手动触发 4 个核心 workflow 并逐个核验：4/4 全部 success、每仓产物 commit 均 removed=0、关键数据资产完好。第一次补测在 rotation-monitor 暴露 pandas 3.x 移除 resample('M') 的问题，双保险修复后重跑通过。明细与最终线上 HEAD 见下方「F. 补测实测记录」 |
| 令牌 / 垃圾 | 交付文件 `ghp_` / `github_pat_` 0 命中；仓库内不生成 `__pycache__` |

**回滚**：优先 `git revert <整改 commit>` 生成反向 commit 再 push（保留历史、无需 force）。整改前基线 `54e04c85`，整改 commit `b5f01afc`（`git revert b5f01afc`）；pandas 3.x 二次修复 commit `ee80a439`（见 F 节，一并保留）。**本仓所有 workflow 都是 `schedule + workflow_dispatch`，没有 `push` 触发**，所以整改 commit 当时没有触发任何 run。**2026-11-05 第一次手动 dispatch 补测（run 41989043660）因 pandas 3.x 失败**，按 F 节双保险修复后重跑（run 41990306988）**success**，产物 commit `1c6dd808`、0 删除、`public/index.html` 完好。

**下次改代码前的三行自检**（沿用前文铁律）：
1. 这个文件是源码还是产物？产物只能靠重跑生成，不能手改。
2. 改动会不会破坏「数据资产 / 口径 / 阈值 / 单位 / cron / 构建顺序」？会就停手。
3. 改完要跑什么验证、要同步 README 哪几节？写进 commit body。

### F. 补测实测记录（2026-11-05，含 pandas 3.x 兼容性二次修复）

整改 commit 上线后，用户要求对 4 个「`push` 不触发、只在 `schedule` / `workflow_dispatch` 跑」的核心抓数流水线做**串行手动触发补测**：一个跑完、核验通过，再触发下一个，避免并发写 `main` 冲突。

**第一次补测在 `rotation-monitor` 失败**，日志关键行：

```
ValueError: 'M' is no longer supported for offsets. Please use 'ME' instead.
  File ".github/workflows/generate_page.py", line 130, in ...
    monthly = bd.set_index('date')['close_cyb'].resample('M').last()
```

根因：pandas 3.1.1 于 2026-11-04 发布，移除了 `resample('M')` 别名，并默认开启 Copy-on-Write、改变字符串 dtype。**整改前依赖本无上界**；本轮整改把上界写成 `<4` 过宽，且 `daily_update.yml` 的 `pip install` 行仍是完全无上界 —— 这是策略失误，已按下面「双保险」纠正。

**双保险修复（3 仓 6 文件）**：

1. **代码**：`rotation-monitor/.github/workflows/generate_page.py` 的 `resample('M')` → `resample('ME')`（pandas 2.3.3 与 3.0.0 下语义一致，均为月末重采样，本地实测两种版本输出一致）。
2. **依赖**：所有 pandas 约束收紧为 `<3`，且 **`requirements.txt` 与 workflow 的 `pip install` 行两处同步** —— 线上实际执行的是 workflow 里的 pip install，不是 requirements.txt，只改一处等于没改。
3. **二次 commit**：`rotation-monitor` `ee80a439`、`earl-etf-dashboard` `af8b5237`、`national-team-etf-flow` `b0ef1c6c`。

**修复后重跑，4/4 全部通过**：

| 仓库 / workflow | run id | 结论 | 产物 commit（前 → 后） | 删除 | 关键资产 |
|---|---|---|---|---|---|
| `rotation-monitor` / `daily_update.yml` | 41990306988 | success | `ee80a439` → `1c6dd808` | 0 | `public/index.html` 完好 |
| `us-world-economy-dashboard` / `update.yml` | 41990418511 | success | `e9ee6da9` → `0b18f069` | 0 | `cache/` · `index.html` · `llms.txt` · `snapshot.md` · `version.txt` 完好 |
| `earl-etf-dashboard` / `update.yml` | 41990472730 | success | `af8b5237` → `abdee20d` | 0 | `cache/` · `version.txt` · `llms.txt` · `latest.json` · `index.html` 完好 |
| `national-team-etf-flow` / `daily-refresh.yml` | 41990810766 | success | `b0ef1c6c` → `5110f0f1` | 0 | `_etf_shares_cache.json` · `echarts.min.js` · `index.html` 完好 |

> `a-stock-api`（Worker 部署 + 13 个冒烟端点）与 `etf-lsi`（daily report）由 `push` 触发，整改当时即 `success`，不在上表。

**写本附录时各仓 `main` 的线上实况**（均为短 SHA，位数不影响定位）：

| 仓库 | 整改 commit | pandas 二次修复 | 补测产物 / 当前 HEAD |
|---|---|---|---|
| `a-stock-api` | `daea02a` | — | `daea02a` |
| `us-world-economy-dashboard` | `e9ee6da` | — | `0b18f06` |
| `earl-etf-dashboard` | `75255bd` | `af8b523` | `abdee20` |
| `national-team-etf-flow` | `016a401` | `b0ef1c6` | `5110f0f` |
| `rotation-monitor` | `b5f01af` | `ee80a43` | `1c6dd80` |
| `etf-lsi` | `606958f` | — | `8e8f74f`（自动产物 commit 会继续前移，属正常） |

> **给 AI 的硬约束**：这 3 个仓的 pandas **必须锁 `<3`**，不要「简化」成 `<4` 或去掉上界；`resample('ME')` 不要改回 `'M'`。改依赖时**必须同时改 `requirements.txt` 和 workflow 的 `pip install` 行** —— 第一次补测就是这样崩的。另外这 6 仓**没有单元测试套件**，上述「实测」指真实 workflow 执行 + 产物核验，改完代码仍应手动 dispatch 一次确认。

### G. 后续策略调整 · 按体积自动清理（2026-11-05 同日）

> **本仓情况**：本仓有 `monthly-snapshot.yml`，已按本节改制；`KEY_FILE` 为 `main.py`，不要改错。
>
> **优先级说明**：A / B / D 节记录的是当天**第一版**整改状态，原文保留不改以便追溯决策过程；关于「每月清理」的部分自本节起**以本节为准**：
> - A 表里的「停用 `schedule`」= 停用**旧版「每月 1 号无条件 force 重写 main」**的 schedule；
> - D 节「绝对不要恢复 monthly-snapshot 自动 schedule」= 不要恢复**上面那个旧版**；
> - 本节新增的「按体积体检 schedule」是**有意保留的新机制，不要删、不要改回每月无条件执行**。

#### 为什么要调整

用户指出：当初设计每月清理，是为了防止「每日产物入库」让仓库长期运行后体积膨胀。第一版整改只从「安全 / 可回滚」角度停用了 schedule，**没有回应「空间封顶」这个原始诉求**，等于把自动能力关掉了。实测后确认这个担心成立：

| 实测项 | 数值 |
|---|---|
| 6 仓当时合计体积 | 约 67 MB |
| 最大单个文件 | 4.4 MB（GitHub 单文件硬限 100 MB） |
| 仓库年龄 | 2~6 天（全部建于 2026-10-31 ~ 11-04） |
| 稳定后估算增长 | 约 1~2 MB/天（前几天叠加了整改提交，偏高） |
| 放着不管碰线时间 | 大半年 ~ 2 年到 1 GB 建议线 |
| GitHub 官方口径 | 理想 < 1 GB、强烈建议 < 5 GB，**不是免费版封顶** |
| 6 仓可见性 | 全部 **public**，Actions 分钟数免费不限量 |

结论：不会立刻被封，但会变慢、可能收到 GitHub 提醒邮件，所以「自动封顶」确实需要保留——只是不能再用「每月无条件强推」这种不可回滚的方式。

#### 改成什么

`.github/workflows/monthly-snapshot.yml`：

- `on.schedule` 改为**每周一 UTC 03:00（北京 11:00）只做一次「体积体检」**：读一次仓库 `size`，**低于 500 MB 就直接结束，一个字节都不改、绝不碰 `main` 历史**。
- 仅当 `size >= 500 MB（512000 KB）` 时才执行剪断。按 67 MB 起步、1~2 MB/天估算，约 8 个月~1 年才第一次真正动刀；清完回到约 10 MB 再重新累积，永远远低于 1 GB。
- `workflow_dispatch` 保留，新增 `force` 布尔输入（**默认 false**）：手动触发默认「只体检、不改动」，勾选 `force` 才忽略阈值立即清理。
- 原有护栏**全部保留**：`README.md` / `KEY_FILE` 存在、`git ls-files >= 8`、JSON 合法性校验、root commit 用 Git Data API（`parents: []`）、不走 `git push --force`、不触发 push 工作流、旧 HEAD 记入 artifact `retention-days: 365`、回滚命令写进日志、`concurrency` 互斥。
- **针对「按体积触发」新增一道护栏**：所有被跟踪文件必须 < 50 MB。体积突然暴涨通常是误提交大文件，绝不能把这种异常树固化成「唯一的 root commit」。
- guard 的 JSON 校验从「只查根目录 `*.json`」扩展到「`git ls-files -z '*.json'` 的全部被跟踪 JSON」。
- 体积读取优先 `gh api repos/... --jq .size`，失败回落 `github.event.repository.size`；两者都取不到时**安全跳过、不清理**。

#### 给 AI 的维护边界（追加，不替代 D 节）

1. **不要**把 `on.schedule` 改回「每月 1 号无条件执行」，不要删掉体积判断或改掉 `THRESHOLD_KB`。
2. **不要**把 `force` 默认值改成 `true`，不要去掉手动入口「默认只体检」的语义。
3. **不要**删掉 50 MB 单文件护栏或全量 JSON 校验；它们是「按体积自动」特有的防误固化措施。
4. 想立刻清理：Actions 页手动 dispatch 并**勾选 force**；只想体检：直接 dispatch 不勾选。
5. 清理后 `main` 变成 root commit，**外部克隆 / fork 需要重新 clone**；回滚命令在 run 日志与 artifact 里，保留 365 天。
6. 阈值 500 MB 可调，但**不要调到 1 GB 以上**（失去提前封顶意义），也**不要调到 100 MB 以下**（频繁剪历史）。
7. 判断是否需要清理看的是仓库 `size`（`gh api repos/<owner>/<repo> --jq .size`，单位 KB），不是文件个数或提交数。
8. 线上写操作照旧：精确 `git add`、中文 commit、`git pull --rebase`、**绝不 force**。

#### 验证与回滚

- 验证：改完用 `yaml.safe_load` 校验；在 Actions 手动 dispatch（**不勾选 force**）应看到 `below threshold: nothing to do, main history untouched`，且 `main` HEAD 不变、工作区无改动。
- 回滚本策略：`git revert <本策略 commit>` 即可回到「仅手动入口」的第一版状态；数据资产与其它 workflow 均不受影响。
