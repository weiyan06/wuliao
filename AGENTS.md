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

| 文件               | 体量     | 内容                                                                                    | 加载方式                                       |
| ---------------- | ------ | ------------------------------------------------------------------------------------- | ------------------------------------------ |
| `index.html`     | ~552KB | 全部 HTML + CSS + 逻辑（巨型 `<script>` 在 `<body>` 末尾）                                       | 首屏                                         |
| `data/site.js`   | 122KB  | CONTENT（随机内容，含歇后语 45 条）/ SPEAKS(100) / DISHES(60) / STORIES(40) / VOCAB(220)          | 首屏同步前载                                     |
| `data/daily.js`  | 9.4KB  | ALMANAC_YI(38) / ALMANAC_JI(36) / HISTORY(89 条带日期) / HISTORY_FALLBACK(22)             | 首屏同步前载（`renderDaily` 有 `typeof` 守卫）        |
| `data/movies.js` | 30KB   | MOVIES(95部) 片库                                                                        | 首屏同步前载                                     |
| `data/gsfx.js`   | 130KB  | GSFX(23个游戏事件音效 mp3 base64，Kenney CC0 来源)                                              | **懒加载**，首次交互 / 进游戏时 `ensureAudioData()` 注入 |
| `data/tang.js`   | 1.7MB  | TANG300(320首) + TANG_S2T/TANG_T2S 繁简映射 + TANG_FOLD                                    | **懒加载**，首次进诗词 tab 时 `ensureTang(cb)` 注入    |
| `data/nes.js`    | 165KB  | jsnes 1.2.1（NES 模拟器 UMD，Apache-2.0）+ NES_ROMS[1]（Falling，MIT，ROM base64）+ NES_LICENSE | **懒加载**，首次进复古游戏厅时 `ensureNES(cb)` 注入       |
| `data/nesjs-core.js` | 266KB | @nesjs/core 2.7.0（备用 NES 模拟器，MIT，130+ mapper）——jsnes 打不开的 mapper(25/74/162/246) 自动切它重试 | **懒加载**，首次玩增强引擎游戏时 `ensureNESCore(cb)` 注入 |
| `licenses/`      | 12KB   | jsnes Apache-2.0 / Falling MIT 授权原文 + 说明                                              | 静态文件，不上页面                                  |

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
- `ensureNES(cb) / ensureNESCore(cb) / nesBoot(rom) / stopNes()`：复古游戏厅（**双引擎**：jsnes 主 + @nesjs/core 备用）。`ensureNES` 幂等注入 `data/nes.js`（jsnes）；`ensureNESCore` 幂等注入 `data/nesjs-core.js`（@nesjs/core 2.7.0，MIT，130+ mapper），仅玩增强引擎游戏才下载；引擎选择 `nesPickEngine(mapper)`：mapper 25/74/162/246 → nescore（重装机兵汉字正常、忍着神龟3、封神榜、塞尔达三神之力），其余 → jsnes；**mapper74 黑屏自动回退**：nescore 起跑 100 帧无画面自动重建 jsnes（保护天使之翼2，它在 nescore 黑屏但 jsnes 正常）；`S.engine` + `S.core` 双实例，`nesLoop` 按引擎调 `S.nes.frame()`/`S.core.runFrame()`，按键 `nesBtnDown/Up` 统一（两引擎按钮码一致 A=0,B=1,…），存档按引擎分开；`setTab('nes')` 预先拉取，`setTab` 非 nes 时自动 `stopNes()`（清 rAF + 关 AudioContext）；ROM 对象形如 `{id,name,author,license,desc,b64|bin}`，用户载入的放 `nesUserRoms`；「载入 ROM」支持 .nes / .zip（zip 用浏览器原生 `DecompressionStream('deflate-raw')` 纯前端解包，零依赖）+ **多选批量导入**；命名规则：**有 zip 用 zip 文件名，没有 zip 用 .nes 文件名**（过一遍 `nesPrettyName`：点号→空格、剥结尾地区/语言标记）；**卡带架**（v3.7.2）：导入的 ROM 存本机 IndexedDB `wuliao-nes-carts`（`nesRack*` 函数族），刷新/重启都在，大厅「我的卡带架」卡片可玩/改名/删除/清空；模拟器就绪时 `nesRackLoad()` 自动把卡带架灌回 `nesUserRoms`（rom.id=`rack<key>` 稳定，旧存档可重新挂上）；**按键**（v3.7.4）：`nesKeys`（localStorage `nes.keys`）可自定义双人键位，P1 默认 方向键+Z/X+Enter/右Shift、P2 默认 WASD+K/J+U/I，大厅「按键设置」卡片和游戏内「按键」按钮（自动暂停）进 `#nesKeysView` 改键（点框按键即绑、抢占式去重、Esc 取消），虚拟手柄固定玩家1
- 模拟器要点：画面 `256×240` canvas 直接 `putImageData`（CSS `image-rendering:pixelated` 放大）、rAF 按 `1000/60.0988` 累加限帧、声音走 `onAudioSample` → Float32Array 累积 → `AudioBufferSourceNode` 排程（`A.next` 落后/超前都重对齐），音量跟随全局 `muted`（gain 0/0.3）；存档是 `toJSON()` → gzip(`CompressionStream`) → base64 存 `nes.state`（≈130KB）
- `dailySeed() / seededPick(arr, seed, n)`：每日卡片（黄历宜忌 / 历史上的今天）的稳定随机；`renderDaily()` 启动时渲染一次，`data/daily.js` 缺失时静默跳过
- 每日内容 seed 公式：`new Date()` → `seed = y*372 + (m+1)*31 + day`（poems daily、vocab Day 制、黄历/历史同源；词汇 epoch `V_EPOCH = 2026-10-24`）

## localStorage 键位表（**禁止改名，会丢用户数据**；共 71 个）

- 每日内容：`vocab.count/vocab.marks/vocab.streak/vocab.rev`、`story.stamp/story.count/story.favs/story.fsz`、`food.favs/food.spins`、`speak.favs/speak.streak`、`poem.favs`、`movie.favs/movie.rolls`、`life.favs/life.best`、`notes.list/notes.token/notes.gist/notes.fold`
- 复古游戏厅：`nes.keys`（自定义按键映射 {p1:{UP…},p2:{…}}，值为 e.code）
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
6. **第三方代码/资源**：只收「许可证明确允许再分发」的（当前：jsnes 1.2.1 Apache-2.0、Falling ROM MIT、@nesjs/core 2.7.0 MIT），一律内联进 `data/*.js` 并在 `licenses/` 留授权原文；**禁止内置任何商业 ROM**，用户的 `.nes`/`.zip` 只能本机读、不上传不进仓库；「卡带架」功能（v3.7.2）可把用户自备 ROM 存进**本机 IndexedDB**（库名 `wuliao-nes-carts`，只存在用户自己设备上，网站/仓库零 ROM 字节）

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
- **模拟器数据必须懒加载**：`data/nes.js` 165KB，`setTab('nes')` 才注入；否则首屏 gzip 直接 +34KB
- **jsnes 画面四边默认被裁**：`ppu.clipToTvSize` 初值 `true`，`endFrame()` 把上下左右各 8px 清零（TV 过扫描）。症状是「状态栏右边少一块」，**但不报错、其余画面看着全对** —— 排查画面缺角先查它，别去怀疑 mapper / 滚动 / 镜像；已在 `nesNewEmu` 置 `false`，改模拟器时别改回去
- **jsnes `nmiCounter` 太小 → 游戏卡在等 vblank、整屏灰**：`endScanline` case 261 置 vblank 标志后，要等 `nmiCounter` 个 PPU dot 才触发 NMI；原值 9 ≈3 CPU 周期，比游戏主循环 `LDA $2002 / BPL` 一轮 7 周期还短，NMI 处理程序每帧抢先把 $80 读走 → 游戏永远等不到 vblank → PPU 一个字节都没初始化（调色板/nametable/CHR 全 0）→ **整屏灰**（NES 调色板 $00 就是灰，所以是灰不是黑）。辨识特征：`cpu.REG_PC` 连续几百帧钉死在同一地址 + `ppu.vramMem[0x3F00..0x3F1F]` 全 0。已改 `60`；这是**全局 NMI 时序**，再动它必须连带回归 MMC3 分屏游戏

- **`evaluate` 里别拿页面自身的进度计数当循环条件**（如 `while(nesSession.frames<k) nes.frame()`）：暂停/未跑起来时该计数不增长 → **死循环焊死主线程**，刷新的 load 事件都排队发不出去，只能关标签重开。长任务一律**固定次数 + 分步**，跑之前先想清楚退出条件
- **jsnes `Tile.opaque` 是「每行」标志，不是每像素**：`setScanline` 只在该行有透明像素时写 `false`，默认是 `undefined` —— 拿它统计「tile 有没有图形」会得到全 0 的错误结论（误判成图案表全空、画面该黑屏）。要看图形用 `tile.pix`
- **CHR-RAM 游戏（`vromCount===0`）别自己从 `rom.vrom` 拷银行**：`load1kVromBank` 在 `vromCount===0` 时直接 return，游戏是通过 `$2007` 写 `ptTile` 的；mapper 里再拷一次会把游戏写好的图形抹掉
- **MMC5 的 PRG 模式切换有前置依赖**：`m5prgApply` 里用 `prg_set[]` 门住「本槽由哪个寄存器控制」，**该寄存器还没被写过就保留 loadROM 开机映射**；否则游戏一写 `$5100`，还没写过的槽会被切成全 0 的 RAM → 代码区变 BRK，3 帧内 CPU 跑飞到 $0007 崩（症状：`pc` 钉在低地址 + 非法操作码）
- **MMC5 `$5204` 的 in-frame（bit6）必须现算**：用常驻 `in_frame` 标志它整帧不清，游戏 `BIT $5204 / BVC` 的等待瞬间通过、时序全错；按 jsnes 扫描线编号用 `scanline∈[20,260]` + `$2001` 渲染开关判定（`clockIrqCounter` 只在 20~260 被调，261→-1 是 vblank 起点）
- **`data/nes.js` 是 CRLF 单行巨文件**：多行替换先把 `
` 归一化成 `
` 再写回；**替换串里不要再手写 `
`**，会被写回逻辑二次转义成 `
`（`node --check` 照过，但字节脏了）。mapper 实例就是 `nes.mmap`（原型上挂 `m5*` 方法），没有 `.m` / `.mapper`

## 版本历史（摘要）

- **v3.9.0 复古游戏厅双引擎：jsnes 主引擎 + @nesjs/core 备用引擎，重装机兵汉字正常 + 再救 3 款**：核心痛点（重装机兵 mapper74 对话中文花屏）此前 v3.8.3 的「跳过 CHR bank8/9」修复被用户实测否定（「花的更厉害了」）后回退，根治改为引入第二引擎 @nesjs/core 2.7.0（github.com/taiyuuki/nesjs，MIT，130+ mapper，纯 JS）内联进 `data/nesjs-core.js`（266KB，懒加载，`licenses/nesjs-core-MIT.txt` 留原文）。**引擎规则** `nesPickEngine(mapper)`：mapper 25/74/162/246 → nescore（jsnes 打不开：忍着神龟3/重装机兵/封神榜/塞尔达三神之力），其余 → jsnes；**mapper74 黑屏自动回退**：nescore 起跑 100 帧无画面自动重建 jsnes（天使之翼2 在 nescore 黑屏、jsnes 正常 → 回退保住）。**实测（无头 Chrome 全部画面验证）**：重装机兵 nescore 标题「重装机兵」汉字清晰（截图 OCR 可读，@nesjs/core 的 mapper74 CHR-RAM 处理正确）、忍着神龟3/封神榜/塞尔达三神之力 nescore 出画面、天使之翼2 nescore 黑屏→自动回退 jsnes 出画面、热血曲棍球/新纪录/格斗 nescore 出画面、天神之剑(195)/快打旋风/忍者龙剑传3/Falling jsnes 无回归、12 tab 零报错。**实现**：`S.engine`（'jsnes'|'nescore'）+ `S.core` 双实例，`nesLoop` 帧循环按引擎调 `S.nes.frame()`/`S.core.runFrame()`，按键 `nesBtnDown/Up` 统一（jsnes `buttonDown/Up` 与 nescore `getGamepad().setButton` 的按钮码一致 A=0,B=1,SELECT=2,START=3,UP=4,DOWN=5,LEFT=6,RIGHT=7），音频 nescore 走 `setAudioInterface({outputSample,flushFrame})`，存档按引擎分开（jsnes `toJSON` gzip / nescore `createBinarySaveState` 二进制，格式不互通但可分别存/读），重置按引擎重建实例。**已知遗留**：热血时代剧(191)/松鼠大作战2(192) 两引擎都不支持（Unsupport Mapper Type），仍跑不起来；热血足球 jsnes 黑屏待查。index.html 升 v3.9.0

- **v3.8.3 修重装机兵中文字体花屏（mapper 74 CHR-RAM 支持）**：mapper 74 = MMC3 克隆 + 74LS138/139，硬件把 CHR-ROM 1KB bank **#8/#9 重定向到 2KB CHR-RAM**（NESdev Wiki INES_Mapper_074）。中文 hack 卡运行时通过 `$2007` 把汉字点阵写入 CHR-RAM，渲染时选 bank 8/9 读取。jsnes 的 `writeMem` 对 pattern 区**无条件**写 vramMem + ptTile（`patternWrite`），所以 `$2007` 写汉字本就生效；花屏根因是 h[74]=h[4] 的 MMC3 在选 bank 8/9 时 `load1kVromBank` 从 CHR-ROM 重新加载覆盖了游戏写入的字模。修复：`h[74].prototype.load1kVromBank=function(b,a){if(8===b||9===b)return;return h[0].prototype.load1kVromBank.apply(this,arguments)}`（选 bank 8/9 跳过 CHR-ROM 加载，保留 CHR-RAM）。确定性验证：写 0xAB 到 pattern 0x0800 → 选 bank 8 → 保留 0xAB（未修复会被 CHR-ROM 的 0x9B 覆盖）。回归：mapper74 四款（天使之翼2/热血新纪录/热血曲棍球/热血格斗）+ 原有 8 款 + 内置 Falling + 12 tab 零报错。**重装机兵中文对话字模修复（确定性测试验证机制，端到端待用户实测确认）**。index.html 升 v3.8.3
- **v3.8.2 复古游戏厅新增 mapper 74（MMC3 克隆）支持，4 款中文盗版卡复活**：`data/nes.js` 注册 `h[74]=h[4]` + `h[74].prototype=new h[4]`（与 v3.7.8 mapper195 同模式）。mapper 74 是 NESdev 记载的「MMC3 clone」，用于中文/盗版 hack，jsnes 原生不支持（loadROM 抛「mapper not supported」）。实测：天使之翼2 / 热血新纪录 / 热血曲棍球 / 热血格斗 从「跑不起来」变可玩（帧缓冲 1K~61K 非黑像素），原有 mapper 4/195 各代表款 + 内置 Falling 全部正常、12 tab 零报错。**已知遗留**：同批「重装机兵」（512K PRG 的 mapper74 盗版卡）能启动但黑屏——疑似用了 mapper 74 在选特定 CHR bank 时切 CHR-RAM 的特性，未处理；忍着神龟3(25)/塞尔达三神之力(162)/热血时代剧(191)/松鼠大作战2(192)/封神榜(246) 为未实现 mapper，仍跑不起来（用户暂缓）
- **v3.8.1 复古游戏厅手机端交互优化**：① 手机触摸·竖屏（`@media (pointer:coarse) and (orientation:portrait)`）下游戏画面置顶 + 虚拟手柄上移到屏幕中下（`#nesMount .nes-pad{margin:auto 0 clamp(16dvh,24dvh,200px)}`，避开底部地址栏），桌面端/横屏不受影响；② 虚拟手柄新增「连发」开关（点亮后 A/B 走 `nesTurbo` 连发，关闭时 `nesTurboRelease` 防卡键）+「A+B」组合键（按住同时触发 B+A）。无头 Chrome 390×844 触摸模拟实测：A 单次 / 连发 / 组合键三路互不冲突、画面置顶 + 手柄 59%~78% 屏幕中下、页面零报错。index.html 升 v3.8.1
- **v3.8.0 补齐 MMC5（mapper 5）：恶魔城传说 / 三国志II霸王的大陆 / 大航海时代 从「一开机就抛异常」变可玩**：`data/nes.js` 的 `h[5]` 从残缺桩（`exWrite/ClearIRQ/SetBank_CPU/SetBank_PPU/SetBank_SRAM` 只有调用没定义）换成完整实现（约 +150 行）。① **PRG 四模式**（`$5100` 0=32KB 用 `$5117` 通吃 / 1=16K×2 / 2=16+8+8 / 3=8KB×4，bank 对齐掩码 `&124/&126/&127`）+ `$5114-$5117` **bit7=1 选 ROM、0 选 RAM**、`$5117` 恒为 ROM（Pretendo mapper 005 文档原文核对）。② **开机 `$5100=3`（8KB×4）**：三国志II/大航海时代 从不写 `$5100`，却按 8KB 粒度写 `$5114/15/16`（0xE0/E1/F2/F9/FE/FF 都是 8KB 银行号），只有模式 3 让三个寄存器同时生效；Pretendo 也记着「Romance of the 3 Kingdoms 2 seems to expect $5100 = $03」。开机四窗全映射到末 8KB（对应「游戏预期 `$5117=$FF`」）。③ **CHR**：A 组 `$5120-$5127` / B 组 `$5128-$512F`（镜像 `$512C-$512F`）+ `$5130` 高 2 位；8x16 精灵下硬件是「背景吃 B 组、精灵吃 A 组」→ 在 `renderBgScanline` / `renderSpritesPartially` 开头调 `m5chrBg()/m5chrSpr()` 按阶段换银行，且**走私有装填 `m5chrLoad1k`**（`load1kVromBank` 会 `triggerRendering()`，渲染中换银行再触发渲染会跟精灵钩子互相打架甚至递归，实测把主线程跑满）。④ **`$5105` nametable 映射**：2bit 值 0/1/2/3 恰好等于 jsnes `nameTable` 下标 → 直接写 `ntable1[i]`，**复用闲置的 `nameTable[2]` 当 ExRAM nametable、`[3]` 当 Fill**，不扩数组、不破存档兼容；ExRAM 四种模式先只存不裁（避免 vblank 写入被清零）。⑤ **PRG-RAM**：`$5113` 选 8KB 页给 `$6000-$7FFF`，`$8000+` 落在 RAM 银行也回写 `prgRam`。⑥ **乘法 `$5205/$5206`、扫描线 IRQ**（挂 jsnes 每条可见线调一次的 `clockIrqCounter`：`scanline=20` 重置、21 起计数，恶魔城的 `$5203=8/14` 能触发）、`$5204` bit7=pending（读清零）、**bit6=in-frame 用扫描线区间现算**。⑦ `toJSON/fromJSON` 带完整 m5 状态。**实测**：恶魔城传说 进标题画面（`prg_size=2` 是它自己写的 `$5100`）、三国志II 主循环正常 + 按 START 切菜单（画面逐帧变化）、大航海时代 跑出剧情文本（nametable 里是「SAIL FOR THE COAST OF WEST AFRICA」，960 格逐格与帧缓冲比对 960:9 一致）。**已知缺口**：扩展音频 `$5000-$5015` 只存寄存器不发声（三款开机各只写一次 `$5010/$5015`，无音轨数据写入）；分屏 `$5200` 只存状态、未接 PPU 滚动（三款都写 `$5200=0` 即关闭）。回归：mapper 0/1/2/3/4/195 代表款（超级马里奥兄弟 / 塞尔达传说 / 魂斗罗 / 赤色要塞 / 冒险岛 / 天神之剑（水晶战记）/ 快打旋风 / 吞食天地II / 天神之剑（汉化））全部正常渲染，天神之剑（汉化）状态栏 y=224~239 仍是 16 行 × 256 全宽、右 8 列 128 个亮像素，12 个 tab 回归全过、控制台零报错

- **v3.7.9 修 jsnes NMI 时序（赤色要塞开局整屏灰）**：`endScanline` case 261 置 vblank 标志后靠 `nmiCounter` 延时才触发 NMI，原值 `9` dot ≈3 CPU 周期，比游戏主循环 `LDA $2002 / BPL` 一轮 7 周期还短 → 每帧 NMI 处理程序（向量 $C292 一进来就 `LDA $2002`）抢先把 $80 吃掉，主循环 83900 次读取全拿到 $00 → 卡在 `$D0E0` 死等、PPU 从未被写过 → 调色板/nametable/CHR-RAM 全 0，整屏只剩 NES 调色板 $00 的灰。改 `nmiCounter=60` dot（最坏情况 24+9+12=45，扫参 24 能过但太脆）。「赤色要塞」（mapper 2 UxROM + CHR-RAM，即 Jackal）PC 脱困到 $C28F、调色板 0→30、CHR-RAM 0→4766、nametable 0→486。回归：mapper 0/1/2/3/4/195 各选代表（超级马里奥兄弟 / 塞尔达传说 / 魂斗罗 / 冒险岛 / 天神之剑（水晶战记）/ 快打旋风 / 吞食天地II / 天神之剑（汉化））+ 内置 Falling 全部正常渲染；天神之剑（汉化）状态栏 y=224~239 仍是 16 行 × 256 全宽、右 8 列 128 个亮像素（MMC3 IRQ 分屏未受 NMI 改动影响）；12 个 tab 回归全过、控制台零报错。**已知遗留（与本次无关）**：mapper 5 的「恶魔城传说 / 三国志II霸王的大陆 / 大航海时代」开机即抛 `papu.exWrite is not a function` / `this.SetBank_CPU is not a function` —— `h[5]` 是个残缺 MMC5 桩，`exWrite/ClearIRQ/SetBank_CPU/SetBank_PPU/SetBank_SRAM` 只有调用没有定义，自 v3.7.0 内联 jsnes 起就如此
- **v3.7.8 兼容 mapper 195 + 修画面四边被裁**：① `data/nes.js` 注册 `h[195]=h[4]` —— iNES 195 = Waixing FS303，本质是 MMC3 克隆（PRG/CHR/镜像/IRQ 寄存器与 MMC3 一致，多一块「按 PPU 写 CHR 地址选 CHR-RAM」的 GAL），直接复用 MMC3 实现即可；「天神之剑（汉化）」（flags6=0x32 / flags7=0xC0 → mapper195，PRG 32×16K + CHR 32×8K，电池存档）从 `loadROM` 抛「mapper not supported by JSNES」变为正常进游戏。② 修**画面不全**：jsnes 1.2.1 `ppu.clipToTvSize` 初值 `true`，`endFrame()` 会把最上/下/左/右各 8px **整列清零**（TV 过扫描裁边），症状是「状态栏右边像被截了一块」；`nesNewEmu` 里置 `false` 后，状态栏 y=224~239 十六行恢复 256 全宽（右 8 列亮像素 0→128，走默认值时整条为 0）。实测：汉化版 标题 → 过场 → 带状态栏的游玩画面渲染正常、FPS 60；12 个 tab 回归全过、控制台零报错。仍属 jsnes Apache-2.0 改动版，`licenses/` 不变，ROM 字节不进仓库。

- **v3.7.7 修 jsnes MMC3 IRQ（天神之剑能跑了）**：按 jsnes 官方 issue #595 + nesdev 标准修 `data/nes.js` Mapper4 三处——$C000=锁存/$C001=触发重装（原版写反）、`clockIrqCounter` 重写为「counter==0 或 reload 标志 → 从锁存重装，否则递减；减到 0 且使能才发 IRQ」（重装拍**不**递减，周期=latch+1 拍；初版把重装和递减写成逗号链导致提前 1 拍触发，忍者龙剑传3(汉) 会崩）、`irqReload` 进 toJSON/fromJSON 存档。实测：天神之剑/水晶战记（美版 Crystalis）✓、快打旋风 ✓、精选 27 款 MMC3 中 24 款 ✓ 无回归；超级马里奥兄弟3、忍者神龟3 新旧代码都崩（jsnes 固有缺陷，之前只验过 mapper 头从未真跑）；精选文件夹 68→71（+快打旋风、天神之剑（水晶战记）[美版 mapper4]、用户自加天神之剑（汉化）[mapper195 不支持，v3.7.8 已支持]），游戏清单.txt 补真机试跑备注

- **v3.7.6 修模拟器画面红蓝反色**：jsnes 帧缓冲实际输出 `0x00BBGGRR`（调色板 $16 红=0x0019BC 实证：低位字节才是 R），`nesFrame` 原来按 `0x00RRGGBB` 又交换了一次导致红蓝对调（游戏画面整体偏蓝/粉）；修复为直接透传 `0xFF000000 | v`（小端 u32 补 alpha 即 RGBA），并用 NES 标准调色板 64 色合成帧截图验证

- **v3.7.5 默认键位按用户习惯重设 + 真连发**：P1 = WASD 移动 · J=B(攻击) · K=A(跳跃) · U=连发攻击 · I=连发跳跃 · 数字1=确定 · 数字3=选择；P2 = 方向键 · 小键盘1(B)/2(A)/4(TB)/5(TA) · 小键盘Enter=确定 · 右Shift=选择；连发键为软件实现（按住时主循环每 55ms 翻转底层 A/B ≈18 次/秒，暂停/失焦/停止自动释放防卡键）；`nesKeysLoad` 检测 v3.7.4 旧结构（缺 TA/TB）整体换新默认，新结构只补缺失项（自定义保留）

- **v3.7.4 双人可自定义按键 + 卡带架清空**：`nesKeys` 键位映射（localStorage `nes.keys`，值=e.code），P1 默认 方向键+Z(B)/X(A)+Enter/右Shift、P2 默认 WASD+K(A)/J(B)+I(START)/U(SELECT)（4399 双人风格，双人分坐键盘两边）；大厅「按键设置 · 双人」卡片 + 游戏内「按键」按钮（自动暂停）进 `#nesKeysView`：点框→按键即绑（抢占式去重、Esc 取消、恢复默认），jsnes 双打 `buttonDown(2,…)` 接通，`S.down` 改为 `p:btn` 键；导入命名统一走 `nesPrettyName`（zip 文件名优先，点号→空格、剥 (J)(JU)(Chs) 等尾巴）；卡带架新增「🗑 清空卡带架」（确认后批量删 IndexedDB）

- **v3.7.3 zip 导入用 zip 文件名**：卡带架导入 .zip 时 ROM 名优先取 zip 文件名（去 .zip），整理过中文名的 zip 包直接在卡带架显示中文名

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
