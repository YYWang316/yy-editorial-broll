# 生产环境 · 渲染 · QA

## 环境与项目结构

- hyperframes 需 Node ≥ 22，版本锁定 `hyperframes@0.7.56`，避免 latest 漂移。
- 样帧截图用 headless Chrome（见 SKILL.md Gate 2）；成片核验用 ffmpeg / ffprobe。
- 默认项目目录 `~/hyperframes-projects/<YYYY-MM-DD>-collage-info-<主题>/`，可用 `HYPERFRAMES_PROJECTS_DIR` 改根目录。
- `scripts/new_project.sh` 创建 hyperframes.json、package.json、meta.json、assets/（collage.css + gsap）、style-plates/、compositions/、renders/、qa/。默认不拷贝 sfx，不建音频 cue。
- compositions/ + assets/ + 配置文件是等效可编辑生产源；保留逐条 prompt.txt、DESIGN.md（配色与截图锚点）和 STORYBOARD.md（节拍）。

## composition 与直接无音轨渲染

Authoring annotations are not production content. Final renders must be viewer-clean.

Default delivery is a silent editing master. Audio-bearing previews are opt-in only.

默认根节点示例（时长和 fps 以请求为准）：

```html
<div id="root" data-composition-id="sNN" data-start="0" data-duration="5"
     data-width="1920" data-height="1080" data-fps="30">
  <div class="clip" id="scene-NN"></div>
</div>
```

- CSS 画布尺寸与 composition 一致；1920×1080 是默认横屏 16:9，1080×1920 是明确指定时的竖屏 9:16。安全区按 **明确 item/project 要求 → approved visual brief → 上方 2/3 / 底部 1/3 fallback** 解析，记录数值与来源。y ≤ 720 / bottom 360px 仅为无更高优先级要求时的保守指导。
- 例如 1920×1080 + `bottom 180px caption-safe`：在引入 collage.css 后用 `:root { --caption-safe-bottom: 180px; }` 覆盖 fallback，派生 `--safe-top: 900px`；允许使用 y=900 以上完整空间，不把 y=720–900 判为越界，也不因 fallback 另加留白。使用相同变量作用域同步画幅 / 安全区，不能让源布局仍硬编码 y=720。非底部矩形或多个安全区按 brief 的实际区域处理，不强行转成底部 1/3。
- 资源路径写根相对 `assets/...`；GSAP 本地引用。内层 .clip 有稳定 id。
- 正式入口不得含 `<audio>`、音频 cue、带音轨的嵌入媒体或音频合成逻辑。不要把文件名后缀 SILENT 当作静音证明。
- `collage.css` 默认隐藏 `[data-authoring-only]` / `.authoring-only` / `.shot .ph`；调试专用的 `html[data-view="authoring"]` 不得出现在生产渲染入口。
- 不改动已批准样帧的视觉语言与叙事内容；动画最终构图仍等于批准样帧。

```bash
cd <项目>
npx --yes hyperframes@0.7.56 lint
npx --yes hyperframes@0.7.56 render \
  -c compositions/<名>.html \
  -o renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4 -q standard --quiet
```

无音频输入直接渲染 silent master；日志 audioCount 应为 0 / hasAudio 为 false（若提供），最终以 ffprobe 为准。lint 真错误须修复；默认流程不存在“忽略 audio lint，只要有 AAC 即可”的例外。

如果渲染器无法直接满足编码参数，使用其无音频图像序列直接编码 H.264；不要先输出带音轨视频。以下示例为 1920×1080、30fps、精确 5s / 150 帧，其他请求同步替换 fps、N 与输入帧尺寸：

```bash
ffmpeg -v error -framerate 30 -start_number 0 -i "frames/%06d.png" \
  -frames:v 150 -map 0:v:0 -an -c:v libx264 -pix_fmt yuv420p \
  -movflags +faststart "renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4"
```

必须从实际渲染结果抽帧；例如已由 ffprobe 核实总帧数为 150 的文件：

```bash
ffmpeg -v error -i "renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4" \
  -vf "select=eq(n\\,0)" -frames:v 1 "first-frame.png"
ffmpeg -v error -i "renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4" \
  -vf "select=eq(n\\,149)" -frames:v 1 "last-frame.png"
```

另外抽取动作关键时刻用于 QA contact sheet。拼图标签在缩略图画布外。按需声音预览见 [motion-grammar.md](motion-grammar.md)，不得写回生产入口。

## 内容分层：production content 与 authoring / QA annotations

Authoring annotations are not production content. Final renders must be viewer-clean.

Production content 是向观众传达故事、事实或已批准视觉命题的内容。Authoring / QA annotations 只帮助创作者制作、定位或验收资产，可存在于 source HTML、独立 debug views、QA screenshots、contact sheets、设计文档和 development overlays，但必须在最终生产渲染中隐藏或移除。

非生产内容包括 asset / visual / chapter IDs、文件名、clip names、时长标签、帧号、timecodes、timeline positions、safe-zone / caption-safe labels、QA / debug labels、internal corner metadata、authoring notes / instructions、track names、first-frame / last-frame labels、仅供制作的 ESTABLISH / DEBUG / QA 等状态。已发现的泄漏例子：`OPEN / V02`、`CAPTION-SAFE FIELD`、`OPEN-V05 / 06.7 SEC`。这些只可写在画布外的制作记录或调试层。

按用途判断，不按单词黑名单删除叙事文字。`HYPOTHETICAL`、`TRADING HALT`、`PUBLIC STOCK`、`STILL RUNNING`，乃至上述状态词，在其确切措辞是已批准 viewer-facing brief 的内容时保留；不得自动把内部 ID 当作叙事标签。叙事中的时间或数值不等于渲染时长 / timecode。

HTML 中把制作元素显式标为 `data-authoring-only` / `.authoring-only`，生产模式默认隐藏或移除；调试模式须显式开启。最终渲染前关闭 authoring/debug 开关，检查伪元素、SVG/canvas 文字和已烘焙到图片中的标注，不能仅依赖 CSS 隐藏。图片生成提示词明确禁止绘制制作元数据；有标签的 QA 拼图不能作为正式首尾帧或视频参考输入。Contact sheet 的编号和时间信息放在画布缩略图外；独立调试截图须明确标明，不能替代成片抽帧 QA。

字幕安全区继续保留；`CAPTION SAFE`、`CAPTION-SAFE FIELD`、guides、outlines、shaded debug regions 均不得出现在最终 master。此处禁止的是制作辅助线，不是已批准的叙事框选或卡纸轮廓。

## 渲染合同：默认 silent master

Default delivery is a silent editing master. Audio-bearing previews are opt-in only.

正常生产只输出 `<ASSET-ID>-<descriptive-name>-SILENT.mp4`：

- H.264，默认 1920×1080；用户明确指定其他格式时按其要求。
- 请求的 fps，未指定时 30fps；精确请求时长，未指定时 5s。
- MP4 使用 yuv420p（适用时），NO AUDIO STREAM，音频流数量必须为 0；静音 AAC 也不合格。
- 在源头禁用音频：不在正式 HTML composition 加 SFX 或 audio cue elements，不默认生成 `sfx.wav`、AAC 混音或 `preview-with-sfx.mp4`；不以“先渲染带声音母版再 strip audio”为常规流程。
- 音效、对白、音乐和最终声音设计留到视频编辑器。只有用户明确要求听 SFX、测试声音设计或听运动节奏时才另做声音预览；普通预览请求不自动授权音频。声音预览单独命名，不替代 silent master。
- 将尺寸、fps、时长在制作前写入 brief / production spec 并贯穿首尾帧、composition、渲染和 QA。总帧数 N = duration × fps；若不能表示为整数帧，说明帧网格冲突并确定可实现的时长 / fps，不能静默取整后声称精确。

## Final Production QA（交付阻断检查）

必须检查实际最终 MP4 的第一帧、代表性运动帧（进入、落位、转场、可能出现叠层的时刻）及最后一帧，推荐制作 contact sheet。不能只检查源 HTML、Gate 2 静帧或距结尾 0.3s 的近末帧。最后一帧使用实际解码帧数 N 的索引 N−1。将检查时间 / 帧号、实际文件名与结论记录在 `qa/` 或 QA record，制作标签只在画布外。

### FINAL VIEWER-FACING CLEANUP

逐项验证最终渲染画布不存在可见的制作专用内容：

- [ ] asset ID / visual ID
- [ ] chapter ID
- [ ] filename / clip name
- [ ] render duration
- [ ] timecode / timeline position
- [ ] frame number
- [ ] safe-zone label
- [ ] caption-safe label
- [ ] QA/debug text / production-only status
- [ ] internal corner metadata / track name
- [ ] authoring instruction / note
- [ ] production-only guide / outline / shaded region
- [ ] first-frame / last-frame label
- [ ] 已批准的叙事文字保留，字幕安全区得到遵守

任一制作标注可见即 **QA FAIL**。从生产源隐藏或移除，重新渲染并再次检查首帧、运动帧、末帧后才可交付。若修复需要改动已批准的创意内容，按原 Gate 规则确认修订，不绕过审批。

### FINAL TECHNICAL QA

对最终文件运行 ffprobe（或等价工具），保存结果：

```bash
ffprobe -v error -count_frames \
  -show_entries stream=index,codec_type,codec_name,width,height,pix_fmt,r_frame_rate,avg_frame_rate,duration,nb_frames,nb_read_frames:format=duration \
  -of json "renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4"
ffprobe -v error -select_streams a -show_entries stream=index \
  -of csv=p=0 "renders/<ASSET-ID>-<descriptive-name>-SILENT.mp4"
```

- [ ] width / height 与请求相符（默认 1920×1080）
- [ ] codec_name = h264，pix_fmt = yuv420p（适用时）
- [ ] r_frame_rate / avg_frame_rate 对照请求 fps，核实恒定帧率
- [ ] stream / container duration 对照精确请求时长；仅允许容器时间基的表示误差，不能把短一帧 / 长一帧当作精确
- [ ] 尽可能核实 nb_read_frames / nb_frames = N，N / fps = 请求时长
- [ ] **audio stream count = 0**；第二条命令必须无流索引输出

任何意外音频流即 **QA FAIL**。修正音频生成或导出设置，从生产源重新渲染 / 正确导出，再跑完整 QA；不能接受带音轨文件，也不能将剥离音轨变成默认流水线。

## 默认完成视频交付包

每条保留：

```text
prompt.txt
source/                                  # 或等效可编辑生产源及依赖
first-frame.png
last-frame.png
qa/                                      # 或 QA record/contact sheet
<ASSET-ID>-<descriptive-name>-SILENT.mp4
```

不默认创建冗余 preview MP4。源文件、正式首尾帧和 QA 证据不作为临时文件清掉。

## 其他视觉 QA 与交付

- [ ] 末帧与 Gate 2 确认样帧一致；保留原审批行为。
- [ ] 内容区 / 安全区与当前 item/project 明确要求一致，否则继承 approved visual brief，仅两者缺省时使用 2/3 / 1/3 fallback；bottom 180px 的横屏按 y=900 验收。安全区只有纸场与地层，无制作标注。
- [ ] 每帧只有一处叙事点色批注，点色占比 <10%。
- [ ] 主动效全片不重复，落位有 hold，无 lint 真错误。
- [ ] 屏幕事实文字来自逐字稿，保留已批准 brief 的叙事标签；占位用空白卡，不编数据。
- [ ] 两项 Final Production QA 均通过，QA 记录指向实际交付文件。

成片可保存在项目 renders/；逐条附首尾帧、prompt、等效源与 QA 记录，发给用户并询问是否需复制到指定目录。
