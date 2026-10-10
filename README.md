# sjtu-hw-check

一个 [Claude Code](https://claude.com/claude-code) skill：**在手写作业原稿切片上用红笔补订正批注**。

原稿一个字都不动 —— 在切片右侧接一条白边，红字（黑体中文 + MathJax 数学公式）贴在那条白边上，
再从切口拉一条红色折线，精确指到出错的那一行。

```
第14题.png（手写切片）                第14题.png（交付图）
┌──────────────┐                    ┌──────────────┬────────────────────┐
│  …           │                    │  …           │                    │
│  =x²/2  ●────┼────────────────────┼──            │ ● 订正：少写了一个 │
│              │                    │              │   因子 x，应为 x³/2│
└──────────────┘                    └──────────────┴────────────────────┘
```

分工是刻意的：**「哪一行错了、红字怎么写」由 Claude 读图判断**，
脚本只负责几何与渲染（切图、量行、排版、出图）。所以批注是能独立读懂的完整订正，
而不是一个「此处有误」的标记。

## 依赖

| 需要 | 说明 |
|---|---|
| Python 3.9+ | `numpy`、`pillow`；用 `prepare.py`/`cut.py` 分题还需要 `pymupdf`<br>（`python -m pip install numpy pillow pymupdf`） |
| Node.js 18+ | 渲染 MathJax 用；首次运行 `redpen.py` 会自动在本目录 `npm install`（约 75MB） |
| Chrome 或 Edge | 无头截图用。找不到时用 `--chrome <路径>` 或环境变量 `CHROME_PATH` |

## 安装

```bash
git clone <本仓库> ~/.claude/skills/sjtu-hw-check
# Windows: %USERPROFILE%\.claude\skills\sjtu-hw-check
```

装好后在 Claude Code 里说「**在手写切片上标红笔订正**」「**原稿上改**」「**批改图**」即会触发。

## 快速开始

设 `S=~/.claude/skills/sjtu-hw-check`。

```bash
# ① 把作业 PDF 按大题切成手写切片（如果已经切好，跳过）
python -I $S/scripts/prepare.py 作业.pdf 切图1
#    读 切图1/候选_*.png 挑出每道大题的首行 → 写 边界.json
python -I $S/scripts/cut.py 作业.pdf 切图1 边界.json --out 切图1/按题分区

# ② 量某一行的位置
python -I $S/scripts/locate.py lines 14 --slices 切图1/按题分区 切图2/按题分区
python -I $S/scripts/locate.py band  14 650 780 x2 --slices 切图1/按题分区 切图2/按题分区
#    再用 Read 看一眼 ./_bands/第14题_650_780x2.png，确认就是那一行

# ③ 写 notes.py（照抄 scripts/notes.example.py 的头）

# ④ 合成交付图
python -I $S/scripts/redpen.py --notes notes.py
```

`redpen.py` 会把 `notes.py` 里没提到的块**原样也出一份**，所以交付目录里的张数与切片数一致，
整套可以对照着看。

## notes.py 的格式

```python
SLICES = [r"切图1/按题分区", r"切图2/按题分区"]   # 按顺序编号：1..N
OUT    = r"图片_手写订正"

NOTES = {
    14: [                                          # 键 = 块号
        (60,  r"订正：这里不能停。$\sqrt{1-3u+2u^2}=1-\frac32u-\frac18u^2+\cdots$，"
              r"故原式 $\to\frac32$。"),
        (717, r"订正：$n\sqrt n\cdot\frac{x}{\sqrt n}=nx$，少写了一个因子 $x$，"
              r"结果应为 $\frac{x^3}2$。"),
    ],
}
```

- **键** = 块号，`SLICES` 里的目录按顺序接起来编号（1 起）
- **值** = `[(y, 红字 HTML), ...]`，`y` 是切片图内部的纵坐标，取 `locate.py` 量出来的 `y0..y1` 中点最稳
- 红字是 **HTML**，不是 Markdown —— 加粗写 `<b>…</b>`
- 数学写 `$...$`，交给 MathJax；用 `r"..."` 原始字符串

改完直接重跑：HTML 没变的条目会跳过截图，只重做改动过的那几条（几秒钟）。

## 目录结构

```
sjtu-hw-check/
├── SKILL.md                 给 Claude 看的工作流
├── README.md
├── scripts/
│   ├── prepare.py           ① PDF → 标尺图 / 候选对照表 / 行坐标.json
│   ├── cut.py               ② 按边界切出每题一张手写切片
│   ├── hw_geom.py           几何内核（prepare/cut 共用）
│   ├── locate.py            ③ 量切片里每一行的位置 / 裁横带出来看
│   ├── redpen.py            ④ 合成交付图（红字排版 + 折线指示）
│   ├── snip.js              无头 Chrome 把红字 HTML 截成透明 PNG
│   ├── package.json         mathjax + puppeteer-core
│   └── notes.example.py     订正数据表模板
└── .gitignore
```

`prepare.py` / `cut.py` / `hw_geom.py` 与同作者的 `sjtu-hw-split` skill 同源，
这里内联一份，好让本仓库自足。

## 常用参数

```bash
python -I $S/scripts/redpen.py --notes notes.py 14 19   # 只重做这几块（调试）
--font '"SimSun","SimHei",sans-serif'                   # 换宋体
--css 26 --max-w 520                                    # 红字小一点、窄一点
--gap 46 --pad-top 26 --margin-max 900                  # 右边距的留白
--color 204,0,0                                         # 红笔颜色
--work <目录>                                           # 中间产物（_redpen/）放哪，默认订正表旁边
```

## 相关 skill

- `sjtu-hw-split` —— 只要按题切开、不批注
- `sjtu-hw-to-latex` —— 把作业识别重排成 LaTeX 图片

## 许可

MIT，见 [LICENSE](LICENSE)。
