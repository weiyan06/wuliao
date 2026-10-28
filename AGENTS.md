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
| `index.html`     | ~552KB | 全部 HTML + CSS + 逻辑（巨型 `<script>` 在 `<body>` 末尾）                    | 首屏                                      |
| `data/site.js`   | 122KB  | CONTENT（随机内容，含歇后语 45 条）/ SPEAKS(100) / DISHES(60) / STORIES(40) / VOCAB(220) | 首屏同步前载                       |
| `data/daily.js`  | 9.4KB  | ALMANAC_YI(38) / ALMANAC_JI(36) / HISTORY(89 条带日期) / HISTORY_FALLBACK(22) | 首屏同步前载（`renderDaily` 有 `typeof` 守卫） |
| `data/movies.js` | 30KB   | MOVIES(95部) 片库                                                     | 首屏同步前载                                  |
| `data/gsfx.js`   | 130KB  | GSFX(23个游戏事件音效 mp3 base64，Kenney CC0 来源)                     | **懒加载**，首次交互 / 进游戏时 `ensureAudioData()` 注入 |
| `data/tang.js`   | 1.7MB  | TANG300(320首) + TANG_S2T/TANG_T2S 繁简映射 + TANG_FOLD                 | **懒加载**，首次进诗词 tab 时 `ensureTang(cb)` 注入 |
| `data/nes.js`    | 156KB  | jsnes 1.2.1（NES 模拟器 UMD，Apache-2.0）+ NES_ROMS[1]（Falling，MIT，ROM base64）+ NES_LICENSE | **懒加载**，首次进复古游戏厅时 `ensureNES(cb)` 注入 |
| `licenses/`      | 12KB   | jsnes Apache-2.0 / Falling MIT 授权原文 + 说明                        | 静态文件，不上页面                            |

巨型 script 的内部分区用 `/* ---------- 标题 ---------- */` 注释分隔，顺序大致是：
工具函数 → sfx/VIEWS/setTab → 游戏（GAMES.xxx = {…}）→ 各内容 tab 逻辑（poems/speak/food/story/vocab/movies）→ `/* ---------- 启动 ---------- */` 收尾。

## tab 架构（新增 tab 的五步）

12 个 tab：小游戏(40款) / 互动玩具(36个) / 复古游戏厅(NES 模拟器：内置1款开源 ROM + 本地载入) / 随机内容(10栏目 + 黄历·历史两张每日卡片) / 唐诗三百首(320) / 每日练嘴(100) / 每日菜谱(60) / 睡前故事(40) / 每日背单词(220词=44天×5) / 电影推荐(95) / 生活妙招(80条) / 随手记(便签+Gist同步)。

1. `<nav class="tabs">` 加 `<button class="tab-btn" data-tab="NAME" type="button">标题</button>`
2. `const VIEWS = { …, NAME:'view-NAME' }` 加一项
3. `<main>` 里加 `<section id="view-NAME" class="view">…`（内部结构惯例：`#xxxLobby` 大厅 + 可选 `#xxxView` 详情页 `.game-view.hidden`）
4. 新数据放 `data/*.js` 并在巨型 `<script>` **之前**用 `<script src="data/xxx.js"></script>` 引入（大数据走懒加载模式，参考 `ensureTang`）
5. 巨型 `<script>` 里加逻辑段 `/* ---------- 标题 ---------- */`；详情页模式参考 `openPoem/openMovie`（大厅加 `.hidden`、详情去 `.hidden`、`$('#xxBack')` 返回时重渲染）
- 切换入口：`setTab(name)`；`#NAME` hash 可直达；默认 tab = games
- **内容 tab 首次进入才渲染**：`deferTabInit(name, fn)` 登记、`setTab` 里 `runTabInit(name)` 执行一次（speak/food/story/vocab/movies/life/random）；首屏只建游戏/玩具大厅，300+ 张隐藏卡不参与首屏
- 大厅卡片点开详情 / chips 筛选 / 分页，全部是上面 tab 的成熟模式，照抄改名即可

## 全局工具箱（巨型 script 顶层，可直接用）

- `$ / $$`：querySelector / queryAll
- `store.get(key, def) / store.set(key, val)`：localStorage 封装（自动加 `wuliao.` 前缀）
- `sfx.click / point / pop / win / lose / bonk`：Web Audio 音效（受「音效」开关控制，全局 `muted`）
- `escHtml(s)`：HTML 转义（用户输入/数据上屏必须过一遍）
- `copyFallback(text, done)`：复制兜底（file:// 下 clipboard API 可能被拒）
- `initPager(pagerSel, cardSel, perPage)`：分页器；**动态渲染的卡片每次 innerHTML 后要重调一次**
- `makeCanvas(w, h)`：像素画布工厂（返回 `{cv, ctx}`，自动加 `.pix-canvas`）
- `gsfx(name, throttleMs) / vibK(key)`：真实事件音效（`data/gsfx.js` 懒加载，未就绪时自动回退 `GSFX_FB` 合成音）+ 手机震动节奏（`VIBS`）；只在关键事件震，乐器类玩具保留合成 `tone()`
- `ensureAudioData()`：注入 `data/gsfx.js`（首次交互 / 进游戏 / 空闲时调用，幂等）；`deferTabInit(name, fn) / runTabInit(name)`：内容 tab 首访渲染
- `ensureNES(cb) / nesBoot(rom) / stopNes()`：复古游戏厅（jsnes）。`ensureNES` 幂等注入 `data/nes.js`；`setTab('nes')` 预先拉取，`setTab` 非 nes 时自动 `stopNes()`（清 rAF + 关 AudioContext）；ROM 对象形如 `{id,name,author,license,desc,b64|bin}`，用户载入的放 `nesUserRoms`；「载入 ROM」支持 .nes / .zip（zip 用浏览器原生 `DecompressionStream('deflate-raw')` 纯前端解包，零依赖）+ **多选批量导入**；**卡带架**（v3.7.2）：导入的 ROM 存本机 IndexedDB `wuliao-nes-carts`（`nesRack*` 函数族），刷新/重启都在，大厅「我的卡带架」卡片可玩/改名/删除；模拟器就绪时 `nesRackLoad()` 自动把卡带架灌回 `nesUserRoms`（rom.id=`rack<key>` 稳定，旧存档可重新挂上）
- 模拟器要点：画面 `256×240` canvas 直接 `putImageData`（CSS `image-rendering:pixelated` 放大）、rAF 按 `1000/60.0988` 累加限帧、声音走 `onAudioSample` → Float32Array 累积 → `AudioBufferSourceNode` 排程（`A.next` 落后/超前都重对齐），音量跟随全局 `muted`（gain 0/0.3）；存档是 `toJSON()` → gzip(`CompressionStream`) → base64 存 `nes.state`（≈130KB）
- `dailySeed() / seededPick(arr, seed, n)`：每日卡片（黄历宜忌 / 历史上的今天）的稳定随机；`renderDaily()` 启动时渲染一次，`data/daily.js` 缺失时静默跳过
- 每日内容 seed 公式：`new Date()` → `seed = y*372 + (m+1)*31 + day`（poems daily、vocab Day 制、黄历/历史同源；词汇 epoch `V_EPOCH = 2026-10-24`）

## localStorage 键位表（**禁止改名，会丢用户数据**；共 70 个）

- 每日内容：`vocab.count/vocab.marks/vocab.streak/vocab.rev`、`story.stamp/story.count/story.favs/story.fsz`、`food.favs/food.spins`、`speak.favs/speak.streak`、`poem.favs`、`movie.favs/movie.rolls`、`life.favs/life.best`、`notes.list/notes.token/notes.gist/notes.fold`
- 游戏最佳分：`snake.best`、`g2048.best`、`mine.best`、`tetris.best`、`simon.best`、`breakout.best`、`pong.best`、`flap.best`、`reflex.best`、`guess.best`、`maze.best`、`memory.best`、`rps.best`、`slide.best`、`mole.best`、`jumper.best`、`fishy.best`、`spider…`（`*.best` 后缀为惯例）
- v3.6 新游戏：`suika.best`、`picross.best`、`jigsaw.best`、`diff.best`、`typing.best`、`hanoi.best`（对象 `{3,4,5}` 按盘数存）、`g24.best`、`reversi.best`
- v3.6 新玩具/彩蛋：`muyu.count`、`qian.count`、`typewriter.count`、`slack.sec`（累计停留秒）、`slack.visits`（来访次数）
- 玩具/设置：`muted`、`lamp.on/lamp.hue/lamp.bright/lamp.spread`、`beat.bpm`、`rhythm.best`、`pet.save`（宠物存档：名字/皮肤/三围数值/照顾天数/阶段）
- v3.7 复古游戏厅：`nes.state`（单个模拟器存档 `{rom,name,at,data:'gz:…'}`，gzip 后 base64，容量不足时静默失败并提示）

## 硬性约束

1. **零构建**：不引入打包器、框架；字体走 CDN（display=swap + media=print 异步加载，不阻塞首屏），图标用 inline SVG
2. **零外部运行时依赖**：不接 API（决策记录：TMDB key 无法藏 + 大陆网络不稳 + 双端一致，数据一律内置 `data/*.js`）。**例外**（2026-10 决策）：随手记的 Gist 同步走 `api.github.com`——用户自有 token + 私有 gist，数据自控、可撤销，断网自动降级本地模式
3. **单文件时代风格**：新代码复用既有模式（原生 JS、`$/$$`、`store`、`sfx`、`initPager`），不搞新范式
4. `index.html` 是 **CRLF 行尾**；多行文本锚点匹配必须用 `\r\n`
5. 文件全 UTF-8；PowerShell 脚本写盘用 `UTF8Encoding($false)`（无 BOM），含中文的 ps1 用 `-File` 执行时字符串会花——用 `[IO.File]::ReadAllText` + IndexOf 重写法处理
6. **第三方代码/资源**：只收「许可证明确允许再分发」的（当前：jsnes 1.2.1 Apache-2.0、Falling ROM MIT），一律内联进 `data/*.js` 并在 `licenses/` 留授权原文；**禁止内置任何商业 ROM**，用户的 `.nes`/`.zip` 只能本机读、不上传不进仓库；「卡带架」功能（v3.7.2）可把用户自备 ROM 存进**本机 IndexedDB**（库名 `wuliao-nes-carts`，只存在用户自己设备上，网站/仓库零 ROM 字节）

## 开发-测试-部署流程（每轮改动照此执行）

1. **改代码**（见上面「新增 tab 五步」/「工具箱」）
2. **浏览器实测**：`browser.preview({path:'index.html'})` 打开 → 检查零 JS 报错 → 主功能链路走通 → **全部 tab 快速回归**（逐个 click 看 `.active`）
3. `git add … && git commit -m "中文描述" --no-verify`
4. **双端推送**：`git push gitee HEAD:refs/heads/master` 和 `git push origin HEAD:refs/heads/master`（GitHub master 曾被 curl PUT 搞成分叉，需要时 `push -f` 对齐本地）
5. **等 Pages 构建**：`gh api repos/weiyan06/wuliao/pages/builds/latest --jq '.status'` 轮询到 `built`
6. **线上抽查**：`curl --compressed` 验 index 版本号 + 新数据文件 200
7. **同步桌面仓库**：`git -C C:\Users\95639\Desktop\ai\wuliao fetch gitee && git merge --ff-only gitee/master`（GitHub 端历史可能因 `gh api` 提交分叉，以 gitee 为准）
8. **升版本**：index.html 顶栏 `.tagline` 的 `PIXEL HOUSE vX.Y`（功能+1 小版本，修复 +0.0.1）；大改动顺手回来更新本文件的数字

## 血泪坑清单（别重蹈覆辙）

- **PowerShell**：`.Replace(string,string)` 会被重载仲裁坑（报"无法转 char"）→ 一律 `IndexOf + Remove + Insert`
- **多区间删除必须严格按位置降序**执行，否则中途 shift 全错位（切割脚本教训）
- **锚点匹配**：先 `IndexOf` 探测真实字节再改；跨多行锚点注意 CRLF；行内 `"}];` 收口和独立行 `];` 都存在过
- **浏览器工具**：tab id 会失效，先 `browser.tabs.list` 拿当前 id；`evaluate` 里**没有 `setTimeout`**，长任务分步或用 `browser.wait`；本地文件必须走 `browser.preview`（不能 file:// open）
- **诗词 tab**：懒加载钩子挂在 `setTab('poems')` 分支（boot 时不渲染）；诗卡类名是 `.poem-card`（不是 `.speak-card`）；按钮 id 是 `poemRnd`（不是 poemRand）
- **TANG300 收口**是行内的 `"}];`，不是独立行 `];`（拆数据脚本踩过）
- 控制台输出中文乱码是 PowerShell 显示问题，文件本身没事——用 `grep` 工具核对中文
- **插入锚点**：8 款新游戏在 `const TOYS = {};` **之前**（GAMES 定义区末尾）、4 个新玩具在「像素宠物 · 果冻进阶版」段**之前**、每日卡片/摸鱼计时/Konami 在「启动」注释**之前**——插错位置会拿不到 `GAMES`/`TOYS` 或抢在初始化前执行
- **jsnes 1.2.1 的 `nes.reset()` 是坑**：reset 后状态不干净，实测能把 ROM 跑到非法操作码、CPU 死循环（页面卡死）。「重置」一律**重建 `new jsnes.NES()` + loadROM**（见 `nesNewEmu`）
- **jsnes 崩溃会走 `this.nes.stop()`**：1.2.1 里 `NES.prototype.stop` 不存在 → 每帧抛 `this.nes.stop is not a function` 刷屏。所以 `nesLoop` 里 `frame()` 必须 try/catch，出错就暂停并提示「点重置」
- **存档别直接存 `toJSON()`**：原始 JSON ≈1.9MB（ppu 的 vramMem/buffer/bgbuffer 全是 3 万级普通数组），必须 `CompressionStream('gzip')`+base64（≈130KB）再进 localStorage；`DecompressionStream` 兜底走 `raw:` 前缀
- **模拟器数据必须懒加载**：`data/nes.js` 156KB，`setTab('nes')` 才注入；否则首屏 gzip 直接 +34KB

## 版本历史（摘要）

- **v3.7.2 我的卡带架**：复古游戏厅 ROM 本机持久化——导入的 .nes/.zip 存 IndexedDB `wuliao-nes-carts`（只存用户设备，网站/仓库零 ROM 字节）；大厅新卡片「我的卡带架 · N 张」→ 管理视图（玩/改名/删除，`#nesRackView`）；「载入 ROM」支持一次多选批量导入（`nesImportFiles`，大厅导入自动开玩第一张、卡带架内导入只入架）；模拟器就绪时 `nesRackLoad()` 自动回灌 `nesUserRoms`，rom.id 改为稳定 `rack<key>`（本机存档跨会话可挂回）；卡带架不可用（隐私模式/空间不足）自动降级为会话内可玩
- **v3.7.1 载入 ROM 支持 zip 直载**：文件选择器收 .zip，纯前端解析 zip Central Directory + `DecompressionStream('deflate-raw')` 解包（浏览器原生 API，零依赖），取包内首个 .nes 直接开跑；无 .nes / 不支持的压缩方式 / zip64 等异常给友好提示；ROM 仍只进内存不落库

- **v3.7.0 复古游戏厅（第 12 个 tab）**：内联 jsnes 1.2.1（Apache-2.0）跑 NES/FC 游戏——内置 MIT 开源 ROM **Falling**（tragicmuffin）打开即玩 + 「载入 ROM」本机选 `.nes`（FileReader，不上传、只进内存）；256×240 `pixelated` 画面、rAF 按 NTSC 帧长限帧、Web Audio 采样排程（跟随全局静音）、键盘（方向键/WASD + X=A/Y=B… 实际 Z=B X=A + Enter=Start/Shift=Select）、粗指针设备出虚拟手柄（D-pad + A/B + SELECT/START）、暂停/重置/全屏、**单槽存读档**（`toJSON` → gzip → base64 → `nes.state`）；沉浸模式复用 `body.game-full`，切 tab 自动 `stopNes()` 清 rAF + 关 AudioContext；新增 `data/nes.js`（懒加载）与 `licenses/`（两份授权原文）；新键 `nes.state`（70 个）
- **v3.6.1 性能与稳定性补丁**：`data/gsfx.js` 改为懒加载（`ensureAudioData`，首次交互/进游戏/空闲注入，音效未就绪自动合成兜底）；内容 tab（speak/food/story/vocab/movies/life/random）改为首访渲染（`deferTabInit`）；每日卡片/背单词/妙招跨天自动刷新（`refreshDailyUi` + 零点定时器 + `visibilitychange` 补刷）；记忆色块/转盘/掷骰子的定时器在 `destroy` 里清理；修幸运转盘 `destroy` 前丢失的 `});` 造成的**全站脚本语法错误**（线上白屏级事故）；修掷骰子离场后音频 `ended` 回调写已销毁 DOM 的报错；字体 CSS 改 `media=print` 异步加载；补 SEO/OG meta
- **v3.6.0「全都坐上去」大礼包**：小游戏 32→40（合成大西瓜 / 数织 Picross / 像素拼图 / 找不同 / 打字高手 / 汉诺塔 / 24 点 / 黑白棋），互动玩具 32→36（电子木鱼 / 求签筒 / 白噪音 / 打字机），随机 tab 顶部新增「今日黄历（宜 4 忌 4）」+「历史上的今天」两张每日卡片（seed 同源，新增 `data/daily.js`），随机栏目 9→10（歇后语 45 条，`c-xie` 配色），页脚摸鱼计数器（`slack.sec/slack.visits`，按秒累计、10 秒落库、`pagehide` 兜底），隐藏 **Konami 秘技**（`↑↑↓↓←→←→BA` → 90 片彩带 + 全站色相流转 + toast，输入框内不触发）；新键 13 个：8 个游戏 `*.best` + `muyu.count/qian.count/typewriter.count/slack.sec/slack.visits`
- **v3.5.0 像素宠物「果冻」大扩展**：`pet.save` 存档（起名/6 色皮肤/离线衰减封顶 96h）+ 眼睛跟随指针/摸身体害羞 + 吐槽气泡/随机蹦跳摇头 + 睡觉模式（22:00~8:00 Zzz，喂食吵醒 6s）+ 成长阶段（照顾 5 天大果冻、15 天果冻王戴金冠）+ 清洁度第三数值 + 3 秒搓澡小游戏（点破泡泡加清洁）+ 随机食物喂食（蛋糕/苹果/辣椒 spicy 特效/面包）；新键 `pet.save`
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
- v3.2.6 掷骰子沉浸模式：进入时隐藏顶栏+tab 导航、`html.dice-lock` 锁滚动；返回/切 tab 自动恢复；其他游戏不受影响
- v3.2.7 沉浸模式推广到全部 32 个小游戏（`body.game-full` + `html.game-lock`，mountGame 挂/backToLobby·setTab 摘）；互动玩具当时未启用
- v3.4.2 图案库扩到 100 款：新增 50 款卡通人物/动物（`DOODLE_ART2`：火柴人/忍者/海盗/太空人/圣诞老人/狐狸/企鹅/恐龙/鲸鱼/独角兽/摩天轮…），`randDoodle` 合并两库抽取
- v3.4.1 像素涂鸦加「随机画一张」：50 款 16×16 字符画（`DOODLE_ART`，爱心/星星/猫脸/火箭…）居中画进 24×24 画布，`gsfx('sweep')`+提示名，相邻不重样（排除上一张）
- **v3.4.0 互动玩具沉浸模式 + 全玩具事件音效/震动**：`mountToy`/`backToToyLobby` 挂/摘 `body.game-full`+`html.game-lock`（含 scrollTo(0,0) 防返回按钮出视窗）；~20 个事件型玩具换 `gsfx`/`vibK`（捏泡泡/捏捏/史莱姆=squish、宠物喂食=chomp、烟花/气球爆/吹爆/雷=boom、摘星=coin 等），钢琴/鼓/鼓机/竖琴/指尖陀螺等乐器类玩具保留合成音
- v3.3.1 修沉浸模式滚动残留 bug：手机上在大厅深处点开游戏，隐藏顶栏/tab 后内容上移而滚动位置不变，返回按钮被顶出视窗且滚动锁死看不到 → 进入与退出时 `window.scrollTo(0,0)` 重置滚动
- **v3.3.0 全部 32 款小游戏接入真实音效 + 手机震动**：`data/gsfx.js`（23 个 Kenney CC0 ogg→mp3 base64，≈98KB 原始）+ `gsfx(名, 节流ms)` 引擎（含 `GSFX_FB` 合成兜底映射、`GSFX_TH` 节流、`gsfxPool` 防泄漏）+ `vibK(k)` 震动节奏库（`VIBS = {die:[60], boom:[80], win:[30,50,60], hurt:[40], tick:10, small:20}`，只在死亡/爆炸/大事件震）；win/lose 保留原 sfx 合成喇叭；掷骰子保持自带 sfxDice 链不重复接入
