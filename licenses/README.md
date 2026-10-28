# 第三方授权

复古游戏厅用到的两个第三方作品，都允许再分发：

| 文件 / 数据 | 作品 | 作者 | 协议 |
| --- | --- | --- | --- |
| `data/nes.js` 内嵌的模拟器代码 | [jsnes](https://github.com/bfirsh/jsnes) 1.2.1（npm `jsnes@1.2.1` 的 `dist/jsnes.min.js`，未改动） | Ben Firshman 等 | Apache License 2.0 → [`jsnes-Apache-2.0.txt`](jsnes-Apache-2.0.txt) |
| `data/nes.js` 内嵌的 ROM（base64） | [Falling](https://github.com/xram64/falling-nes)（`falling.nes`，40,976 字节） | tragicmuffin | MIT，Copyright (c) 2018 tragicmuffin → [`falling-MIT.txt`](falling-MIT.txt) |

说明：

- 本站**不包含任何商业 ROM**（超级马里奥、魂斗罗等一律没有）。用户自己选择的 `.nes` 文件只在本机浏览器内存里运行，不上传、不写服务器。
- `data/nes.js` 文件头保留了这两份作品的来源与协议说明；完整协议原文放在本目录。
- 如果你要再分发本站，请一并保留本目录与 `data/nes.js` 的头部注释。
