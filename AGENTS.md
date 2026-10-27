# AGENTS.md — 像素屋 开发者说明书（AI 工具优先读这份）

> 本文件供 AI 编码工具（OpenCode / Claude Code 等）与人类开发者快速上手。**改了架构或数字后请同步更新本文件。**

## 项目是什么

零构建单页网站「像素屋 · PIXEL HOUSE」——打发无聊的像素风大杂烩：小游戏 + 每日内容 tab。
纯原生 HTML/CSS/JS，无框架、无打包器、无外部运行时依赖（不接任何 API）。

- 线上主站：https://weiyan06.github.io/wuliao/ （GitHub Pages）
- 国内镜像：https://gitee.com/weiyan06/wuliao/ （双端同步推送）
- 桌面备份仓库：`C:\Users\95639\Desktop\ai\wuliao`（每次上线后 ff-only 同步）
- 开发工作区：OpenCode worktree（detached HEAD，push 时用 `HEAD:refs/heads/master`）

## 文件布局

| 文件               | 体量     | 内容                                                                 | 加载方式                                    |
| ---------------- | ------ | ------------------------------------------------------------------ | --------------------------------------- |
| `index.html`     | ~353KB | 全部 HTML + CSS + 逻辑（巨型 `<script>` 在 `<body>` 末尾）                    | 首屏                                      |
| `data/site.js`   | 117KB  | CONTENT（随机内容）/ SPEAKS(100) / DISHES(60) / STORIES(40) / VOCAB(220) | 首屏同步前载                                  |
| `data/movies.js` | 30KB   | MOVIES(95部) 片库                                                     | 首屏同步前载                                  |
| `data/tang.js`   | 1.7MB  | TANG300(320首) + TANG_S2T/TANG_T2S 繁简映射 + TANG_FOLD                 | **懒加载**，首次进诗词 tab 时 `ensureTang(cb)` 注入 |

巨型 script 的内部分区用 `/* ---------- 标题 ---------- */` 注释分隔，顺序大致是：
工具函数 → sfx/VIEWS/setTab → 游戏（GAMES.xxx = {…}）→ 各内容 tab 逻辑（poems/speak/food/story/vocab/movies）→ `/* ---------- 启动 ---------- */` 收尾。

## tab 架构（新增 tab 的五步）

11 个 tab：小游戏(32款) / 随机内容 / 唐诗三百首(320) / 每日练嘴(100) / 每日菜谱(60) / 睡前故事(40) / 每日背单词(220词=44天×5) / 电影推荐(95) / 生活妙招(79条) / 随手记(便签+Gist同步) / 互动玩具(32个)。

1. `<nav class="tabs">` 加 `<button class="tab-btn" data-tab="NAME" type="button">标题</button>`
2. `const VIEWS = { …, NAME:'view-NAME' }` 加一项
3. `<main>` 里加 `<section id="view-NAME" class="view">…`（内部结构惯例：`#xxxLobby` 大厅 + 可选 `#xxxView` 详情页 `.game-view.hidden`）
4. 新数据放 `data/*.js` 并在巨型 `<script>` **之前**用 `<script src="data/xxx.js"></script>` 引入（大数据走懒加载模式，参考 `ensureTang`）
5. 巨型 `<script>` 里加逻辑段 `/* ---------- 标题 ---------- */`；详情页模式参考 `openPoem/openMovie`（大厅加 `.hidden`、详情去 `.hidden`、`$('#xxBack')` 返回时重渲染）
- 切换入口：`setTab(name)`；`#NAME` hash 可直达；默认 tab = games
- 大厅卡片点开详情 / chips 筛选 / 分页，全部是上面 tab 的成熟模式，照抄改名即可

## 全局工具箱（巨型 script 顶层，可直接用）

- `$ / $$`：querySelector / queryAll
- `store.get(key, def) / store.set(key, val)`：localStorage 封装（自动加 `wuliao.` 前缀）
- `sfx.click / point / pop / win / lose / bonk`：Web Audio 音效（受「音效」开关控制，全局 `muted`）
- `escHtml(s)`：HTML 转义（用户输入/数据上屏必须过一遍）
- `copyFallback(text, done)`：复制兜底（file:// 下 clipboard API 可能被拒）
- `initPager(pagerSel, cardSel, perPage)`：分页器；**动态渲染的卡片每次 innerHTML 后要重调一次**
- `makeCanvas(w, h)`：像素画布工厂（返回 `{cv, ctx}`，自动加 `.pix-canvas`）
- 每日内容 seed 公式：`new Date()` → `seed = y*372 + (m+1)*31 + day`（poems daily、vocab Day 制同源；词汇 epoch `V_EPOCH = 2026-10-24`）

## localStorage 键位表（**禁止改名，会丢用户数据**；共 55 个）

- 每日内容：`vocab.count/vocab.marks/vocab.streak/vocab.rev`、`story.stamp/story.count/story.favs/story.fsz`、`food.favs/food.spins`、`speak.favs/speak.streak`、`poem.favs`、`movie.favs/movie.rolls`、`life.favs/life.best`、`notes.list/notes.token/notes.gist/notes.fold`
- 游戏最佳分：`snake.best`、`g2048.best`、`mine.best`、`tetris.best`、`simon.best`、`breakout.best`、`pong.best`、`flap.best`、`reflex.best`、`guess.best`、`maze.best`、`memory.best`、`rps.best`、`slide.best`、`mole.best`、`jumper.best`、`fishy.best`、`spider…`（`*.best` 后缀为惯例）
- 玩具/设置：`muted`、`lamp.on/lamp.hue/lamp.bright/lamp.spread`、`beat.bpm`、`rhythm.best`

## 硬性约束

1. **零构建**：不引入打包器、框架、CDN；字体/图标全内联（DotGothic16 + 系统等宽 + inline SVG）
2. **零外部运行时依赖**：不接 API（决策记录：TMDB key 无法藏 + 大陆网络不稳 + 双端一致，数据一律内置 `data/*.js`）。**例外**（2026-10 决策）：随手记的 Gist 同步走 `api.github.com`——用户自有 token + 私有 gist，数据自控、可撤销，断网自动降级本地模式
3. **单文件时代风格**：新代码复用既有模式（原生 JS、`$/$$`、`store`、`sfx`、`initPager`），不搞新范式
4. `index.html` 是 **CRLF 行尾**；多行文本锚点匹配必须用 `\r\n`
5. 文件全 UTF-8；PowerShell 脚本写盘用 `UTF8Encoding($false)`（无 BOM），含中文的 ps1 用 `-File` 执行时字符串会花——用 `[IO.File]::ReadAllText` + IndexOf 重写法处理

## 开发-测试-部署流程（每轮改动照此执行）

1. **改代码**（见上面「新增 tab 五步」/「工具箱」）
2. **浏览器实测**：`browser.preview({path:'index.html'})` 打开 → 检查零 JS 报错 → 主功能链路走通 → **全部 tab 快速回归**（逐个 click 看 `.active`）
3. `git add … && git commit -m "中文描述" --no-verify`
4. **双端推送**：`git push gitee HEAD:refs/heads/master` 和 `git push origin HEAD:refs/heads/master`（GitHub master 曾被 curl PUT 搞成分叉，需要时 `push -f` 对齐本地）
5. **等 Pages 构建**：`gh api repos/weiyan06/wuliao/pages/builds/latest --jq '.status'` 轮询到 `built`
6. **线上抽查**：`curl --compressed` 验 index 版本号 + 新数据文件 200
7. **同步桌面仓库**：`git -C C:\Users\95639\Desktop\ai\wuliao fetch --all && git merge --ff-only origin/master`
8. **升版本**：index.html 顶栏 `.tagline` 的 `PIXEL HOUSE vX.Y`（功能+1 小版本，修复 +0.0.1）；大改动顺手回来更新本文件的数字

## 血泪坑清单（别重蹈覆辙）

- **PowerShell**：`.Replace(string,string)` 会被重载仲裁坑（报"无法转 char"）→ 一律 `IndexOf + Remove + Insert`
- **多区间删除必须严格按位置降序**执行，否则中途 shift 全错位（切割脚本教训）
- **锚点匹配**：先 `IndexOf` 探测真实字节再改；跨多行锚点注意 CRLF；行内 `"}];` 收口和独立行 `];` 都存在过
- **浏览器工具**：tab id 会失效，先 `browser.tabs.list` 拿当前 id；`evaluate` 里**没有 `setTimeout`**，长任务分步或用 `browser.wait`；本地文件必须走 `browser.preview`（不能 file:// open）
- **诗词 tab**：懒加载钩子挂在 `setTab('poems')` 分支（boot 时不渲染）；诗卡类名是 `.poem-card`（不是 `.speak-card`）；按钮 id 是 `poemRnd`（不是 poemRand）
- **TANG300 收口**是行内的 `"}];`，不是独立行 `];`（拆数据脚本踩过）
- 控制台输出中文乱码是 PowerShell 显示问题，文件本身没事——用 `grep` 工具核对中文

## 版本历史（摘要）

- v2.7 睡前故事 / v2.8 每日背单词（ECDICT 词库）/ v2.9 数据拆分优化（2.14MB→首屏 102KB）
- v2.9.1 五子棋落子修复（round→floor）
- v3.0 电影推荐 tab（95 部片库 + 掷骰荐片机）
- v3.0.1 掷骰子改 5 颗 + 闪烁动画 + 「最大」总和统计；destroy 清定时器修复
- v3.0.2 骰子 3+2 两排 + 真实骰子配色（1/4 红点）
- v3.0.3 修 1 点大红点偏位（box-shadow 外扩不碰布局）
- v3.1.1 修复 Gist 连接 422（noteGh 漏传 body + Content-Type 头）
- **v3.1.2 Gist 同步自愈：缓存 gist 404 时自动清缓存→重新发现→认领现存 gist（防分叉）**
- v3.1.3 token 体验：本机 token 回显输入框（明文可复制）+ `?t=` URL 一次点击配新设备（吸收后清地址栏）
- v3.1.4 内置混淆 token：新设备零配置自动连接+开机自动同步；断开用 `notes.token='!off'` 记忆
- v3.1.5 同步设置面板可折叠：默认收起（有 token 时），`notes.fold` 记忆手动展开/收起
- v3.1.6 全站改名：像素快乐星球/BOREDOM KILLER → **像素屋 / PIXEL HOUSE**（存储键 wuliao.* 与仓库地址不变）
- **v3.2.0 生活妙招 tab（79 条小常识：每日主推 + 分类浏览/收藏/搜索 + 真假 5 连测）**
- v3.2.1 掷骰子接入真实骰子音效：`data/dsfx.js`（34KB mp3 → base64 内联，不新增网络请求），受静音开关控制
- **v3.2.2 骰子声画同步：声音是节拍器——`sfxDice(onEnd)` 用 mp3 的 `ended` 事件驱动回合落定（声音放多久动画摇多久）；静音/无声退回 1.2s 兜底；连掷中途重掷截断上一个声音（5s 硬截止兜底）**
- v3.2.3 掷骰子「摇一摇」：`devicemotion` 重力传感器触发（阈值 delta≥20+1.2s 冷却防连掷）；iOS 走 `requestPermission()`（需 https+手势授权，拒绝给设置指引），安卓 Chrome 默认已授权；开关仅触摸设备显示，状态记内存不落库
- v3.2.4 掷骰子手机振动：掷出 `[30,60,30]` 短节奏 + 落定 `15ms` 轻震，走全局 `vib(p)`（iOS/电脑无 `navigator.vibrate` 自动静默）；与音效开关独立
- v3.2.5 骰子滚动全程震感（`RATTLE_VIB` 25ms/70ms 磙动节奏 ≈2.4s，settle 掐断 + 15ms 轻震收尾）；掷骰子卡片挪到小游戏第一位
- v3.2.6 掷骰子沉浸模式：进入时隐藏顶栏+tab 导航、`html.dice-lock` 锁滚动；返回/切 tab 自动恢复；其他游戏不受影响 ← 当前
