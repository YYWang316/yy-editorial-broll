---
name: yy-editorial-broll
description: 半调纸拼贴风信息动画（HyperFrames）。给一份口播逐字稿，从中挑出适合做成信息动画的段落（数字冲击、流程结构、清单收敛、对比翻转、结果高光），经用户确认后出静态样帧，再确认后产出默认 1920×1080、无音轨的 H.264 mp4 动画。用户说「拼贴信息动画」「collage info」「gbro-collage-info」「信息动画挑段落」「逐字稿做信息动画」「半调纸拼贴动画」，或给逐字稿/分镜并想要这种米色纸场+卡纸剪片+定格动效的信息动画时，必须使用此 skill。本 skill 只做无角色的信息图动画（纯 HTML/CSS/GSAP，本机渲染）；情绪独白、隐喻类段落不做，建议归实拍或角色/隐喻动画。
compatibility: 需要 Node >= 22、hyperframes@0.7.56、Chrome（headless 截样帧）、ffmpeg/ffprobe。全流程本机渲染，不调用任何图片生成模型。首次使用先跑 scripts/check_setup.sh 自检环境。
---

# gbro Collage Info（半调纸拼贴 · 信息动画）

把逐字稿里"讲信息"的段落，做成 editorial 半调纸拼贴风的信息图动画：米色纸场、卡纸剪片、单一点色、半调截图、stop-motion 定格。

效果参考：仓库 README 里的 GIF 演示与末帧一览图。

## 画幅与字幕安全区优先级

默认保持 **1920×1080 = 横屏 horizontal 16:9**，用于当前主要的 YouTube 横屏工作流；**1080×1920 = 竖屏 vertical 9:16**，只有用户明确指定时才使用。

安全区取值优先级：**explicit item/project safe-zone requirement → approved visual brief → fallback upper-2/3 information area / bottom-1/3 safe area**。先读取当前 item 的明确要求，否则继承项目要求；其后才采用已批准 visual brief 的安全区 / 构图要求。只有以上均未给出时，才使用上方 2/3 信息区、底部 1/3 安全区的保守 fallback。

因此 1920×1080 的 y ≤ 720 / bottom 360px 不是固定硬限制。例如 brief 指定 `bottom 180px caption-safe` 时，内容区下沿为 y=900，应允许使用 y=900 以上的完整空间，不得强行预留 360px 或把 y=720–900 的合法内容判为失败。可用空间不等于必须填满，继续保持批准的层级与留白。将最终安全区及其来源写入 production spec，贯穿样帧、动画和 QA；不改变审批 Gate、viewer-clean 或 silent-master 规则。

## 三道确认门

```text
Gate 1  选段 + 主动效分配 → 用户确认
Gate 2  静态样帧（headless Chrome 截图 + 拼图）→ 用户确认
Gate 3  HyperFrames 无音轨动画 + 成片画面/技术 QA → 交付
```

每道门都停下来等确认，不跳门。用户在 Gate 1 改名单、在 Gate 2 改构图，都比在 Gate 3 改成片便宜。

## Gate 1：选段

读逐字稿（或分镜脚本，只取其中标为动画的段），按下面的判据挑段落。

**适合做信息动画的段落**（一条至少命中一类）：

| 类型 | 信号 | 对应主动效 |
|---|---|---|
| 数字冲击 | 台词里有具体数字（7 条、10 句、2 分钟、12 类） | 数字撞入 / 计时压缩 |
| 流程结构 | 步骤、决策树、分类收敛（30 个→5 大类） | 从底部生长 / 抽屉抽出 |
| 清单收敛 | 多个候选收敛到一个结果 | 推焦收敛 |
| 对比翻转 | 正面 vs 痛点、以前 vs 现在 | 三联卡翻面 |
| 结果高光 | 观众最想要的东西落地的那句 | 推焦 + 点色描边 |
| 概念封装 | 抽象论点第一次被看见（钩子落点） | 吸入 / 汇聚 |
| 系列标题卡 | "第一个…第二个…第三个…"的章节口 | 同模板换色派生 |

**不适合、要拦下来的**：界面操作演示（归录屏）、需要实拍质感的（归实拍）、情绪独白和隐喻（归角色/隐喻动画，不在本 skill 范围）、在截图上画重点（归录屏+后期描边）。

**数量与去重**：

- 默认精选 6–10 条独立设计；用户指定数量就按用户的。
- 每条分配一个**不重复的主动效**（吸入 / 堆叠 / 撞入 / 景深 / 生长 / 抽屉 / 推焦收敛 / 翻面）。两条段落同形时合并或派生，不做两条同质动画。
- 主动积极找派生机会：系列标题卡从一个模板换色导出；相邻的收拢/翻面/汇聚三段可以做成一条长 composition 分段切。落选段落写明去处（录屏/合并/派生），不是丢掉。

**产出**：一张表（段落原文落点 / 入选理由 / 主动效 / 需要的截图锚点），加落选段的降级说明。停，等确认。

## Gate 2：样帧

确认名单后，每条先做静态 hero frame（不写动画代码）：

1. 写单页 HTML（默认 1920×1080 横屏 16:9；用户明确指定其他格式时按其要求），引用 skill 自带的 `assets/collage.css`，放在项目 `style-plates/` 下。同步设置 CSS 的 `--canvas-width`、`--canvas-height` 与 composition 尺寸，并按上节优先级设置 `--caption-safe-bottom`（`--safe-top` 自动派生）；按有效内容区布局，不拉伸已确认构图。
2. headless Chrome 截图（macOS 路径如下；Linux 用 `google-chrome` / `chromium` 可执行名）：

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --disable-gpu --hide-scrollbars --force-device-scale-factor=1 \
  --window-size=1920,1080 --screenshot=<名>.png "file://<绝对路径>.html"
```

3. 自己先看图 QA（读 PNG）：留白够不够、点色是否只有一处、信息是否遵守按优先级解析后的实际安全区、撕边地层是否只在底部；不得把 fallback 上方 2/3 当作所有项目的验收边界。按 production.md 的 viewer-clean 规则检查正式画面。不合格先改再给用户。
4. 拼 contact sheet 发给用户确认。

风格规则读 [`references/style-system.md`](references/style-system.md)。核心六条：纸场底不用渐变；一件信息=一张卡纸剪片；真实截图=半调剪片、全帧只一处点色批注；一帧一个点色 + 一个出画大色块；3–6 个大组大留白；按已解析的要求保留字幕安全区（仅无指定时 fallback 为底部 1/3），只放纸场与撕边地层，不显示安全区标签或辅助线。

**截图锚点纪律**：需要真实界面截图的位置，没拿到截图前一律用 `.shot` 抽象半调剪片（灰条+网点），**绝不画假界面**；不是原文就不框选放大。拿到截图后换 `<img class="shot-img">` 一行即可。

## Gate 3：动画

样帧确认后，搭 HyperFrames 项目并渲染。动效与按需拟音预览规则读 [`references/motion-grammar.md`](references/motion-grammar.md)，必须读 [`references/production.md`](references/production.md) 的渲染命令、FINAL VIEWER-FACING CLEANUP 与 FINAL TECHNICAL QA。

Authoring annotations are not production content. Final renders must be viewer-clean.

Default delivery is a silent editing master. Audio-bearing previews are opt-in only.

1. 用 `scripts/new_project.sh <项目名>` 建项目（自动拷 collage.css / gsap，写 hyperframes.json；默认不拷 sfx）。
2. 每条一个 `compositions/<段号>-<名>.html`，把样帧 DOM 原样搬进去，再加 paused GSAP timeline。样帧就是动画的收帧——动画结束时的画面必须和确认过的样帧一致。
3. 正式 composition 不含音频元素或 cue；制作标注单独标记为 authoring-only，生产模式隐藏或移除。保留已批准的观众叙事文字，不按词表盲删。
4. `lint` → 直接渲染 `<ASSET-ID>-<descriptive-name>-SILENT.mp4` → 检查实际成片首帧、代表性运动帧、末帧（推荐 contact sheet）→ ffprobe 核验 → 两项 QA 通过后交付，并问是否要拷贝到指定目录。任一制作标注泄漏或意外音轨均为 QA FAIL，修复后重新渲染并复查。

交付默认：1920×1080 / H.264 / yuv420p / 30fps（或请求的 fps）/ 精确请求时长（未指定时 5s）/ NO AUDIO STREAM。音效、对白、音乐与最终声音设计留到剪辑台。仅用户明确要求听音效、测试声音设计或听运动节奏时另做声音预览；默认不生成 sfx.wav、AAC 混音或冗余预览 MP4，不先做带音轨版本再剥离。

每条默认交付 `prompt.txt`、`source/` 或等效可编辑源（compositions + assets 等）、`first-frame.png`、`last-frame.png`、`qa/` 或 QA 记录/contact sheet、`<ASSET-ID>-<descriptive-name>-SILENT.mp4`。

## 色本

固定四组「纸场 + 点色」，按内容功能段分组使用，一个功能段锁一组色：

| 纸场 | 点色 | 默认给 |
|---|---|---|
| 奶白 `#EEE7D6` | 深绿 `#1F5142` | 第一个功能段 |
| 冷灰白 `#E6E7E3` | 墨蓝 `#18334D` | 第二个功能段 |
| 暖石灰 `#D8D0BC` | 珊瑚 `#E0783C` | 第三个功能段 |
| 暖石灰 `#D8D0BC` | 芥末 `#D6A92B` | 钩子与结尾（首尾闭环） |

（可选集成）同一支片子里若还有其他拼贴风动画条目且带有自己的色本定义，继承其纸场/点色，让全片看起来是一家人；没有就用上表默认。功能段少于四段时优先保住"钩子结尾=芥末"的闭环。

## 写作纪律

- 屏幕事实文字来自逐字稿：数字、名称、原话；已批准 visual brief 明确要求的观众叙事标签也应保留。台词 / brief 没给的内容（比如第 4 个候选的文案、5 大类的名字）用空白卡占位，**不替内容编数据**。
- 中文短句，每帧一个论点；mono 小字仅用于已批准的叙事编号 / 路径 / 单位。制作编号、文件路径和段落 ID 属于 authoring annotations，不得出现在成片。
