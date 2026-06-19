---
name: editor-code-video
description: >
  用代码绘制/合成视频（程序化剪辑）。当需要用代码生成动态图形、字幕动效、数据可视化动画，或对素材做批量裁剪/拼接/转码/叠加/混音/调色时用本技能。
  按需求路由到 FFmpeg（剪切/拼接/转码/叠加/混音/套 LUT/响度/侧链/转场/检测）、Remotion（React 程序化视频与字幕动效）、
  Whisper+librosa（转写出 SRT 与节拍时间戳，对接音频驱动流水线），或 Manim/videodb。建议在独立子 Agent 中加载，避免与时间线类软件上下文混淆。
---

# 代码绘制视频执行技能

把剪辑决策落地为**代码渲染**而非 GUI 操作。适合：动态图形/包装、字幕与数字动效、数据可视化、批量/自动化合成、端到端"音频驱动剪辑"。

> 命令均在本机 FFmpeg 8.1.1（libx264/libx265）实测；落地前仍以 `ffmpeg -h filter=<name>` 与官方文档复核**你的版本**——**先核实再执行，绝不臆造参数**。

## 一、选工具（路由）

| 需求 | 用 | 去查的技能/文档 |
|:---|:---|:---|
| 剪切/拼接/转码/裁切/叠加/混音/抽音轨/套 LUT/响度/侧链/转场/场景·静音检测 | **FFmpeg** | 见下 §三 cookbook |
| 把音频转写成带时间戳 SRT、检测节拍卡点 | **Whisper + librosa** | 见下 §五 |
| React 组件化、可参数化的程序化视频（动效、字幕条、片头包装、数据动画） | **Remotion** | skill `remotion-video-creation`，见下 §四 |
| 数学/几何/算法/图示的精确动画、讲解类可视化 | **Manim** | skill `manim-video` |
| 对已有视频库做检索/索引/片段抽取 | **videodb** | skill `videodb` |
| 生成缺失的素材/配音/音乐/音效 | **fal.ai / ElevenLabs / VideoDB** | skill `fal-ai-media`、`videodb` |

## 二、把课程逻辑映射到代码

| 知识内核决策 | 代码落地 |
|:---|:---|
| 音频驱动切点（`02`） | 用 SRT 时间戳驱动 Remotion `<Sequence>` 的 from/durationInFrames；或 FFmpeg 按时间戳切段 |
| 无字幕兜底定切点（`02`） | FFmpeg `silencedetect`/`silenceremove`（气口/死气）+ `select='gt(scene,0.3)'`（镜头边界） |
| 卡点/节拍对齐（`04`/`08`） | `librosa.beat.beat_track` 得节拍秒 → 换算视频帧号 → 在节拍帧切换/触发动效 |
| 贴 B-roll 覆盖（`03`） | FFmpeg `overlay=...:enable='between(t,a,b)'`；Remotion 上层 `<Sequence>` 叠组件 |
| BGM 让位人声（`04`） | FFmpeg `sidechaincompress`（人声触发压低 BGM）+ `amix` |
| 关键帧/运动（`07`） | Remotion `useCurrentFrame()`+`interpolate(...,{clamp})`，只动 transform/opacity；FFmpeg `zoompan` 仅缓慢推拉 |
| 变速（`08`） | FFmpeg `setpts`（视频）+`atempo`（音频）；曲线变速需分段或 Remotion；补帧 `minterpolate=mi_mode=mci` |
| 抠像/合成（`09`） | FFmpeg `chromakey`/`colorkey`+`overlay`+`blend`；AI 人物抠像用 rembg/RVM 或外接达芬奇/Premiere |
| 字幕/数字动效（`10`） | Remotion 组件做文字进出场/滚动/数字跳动；按帧驱动 |
| 调色/S曲线/LUT（`06`） | FFmpeg `curves`/`lut3d`/`eq`/`colorbalance`；Remotion 用 CSS filter |
| 导出/画幅（`12`） | FFmpeg `libx264 -crf`/`-b:v`、`-c:a aac -b:a`、`crop+scale` 重构画幅 |
| 黑场红线 | 不自动用代码"硬接"填补；输出待补位置清单交回用户 |

## 三、FFmpeg 命令 cookbook（已实测，可直接改用）

**① 抽音轨给 Whisper（16kHz 单声道）**
```bash
ffmpeg -i in.mp4 -vn -ac 1 -ar 16000 -c:a pcm_s16le audio.wav
```

**② 按时间切段（精度 vs 速度）**
```bash
# 精确（重编码，逐帧准）：跨段切割推荐 -ss + -t（显式时长，规避 -to 参考点版本差异）
ffmpeg -ss 00:01:30 -i in.mp4 -t 30 -c:v libx264 -crf 18 -c:a aac seg.mp4
# 极快（-c copy，但只能对齐到最近关键帧、起点不精确）
ffmpeg -ss 00:01:30 -to 00:02:00 -i in.mp4 -c copy seg_fast.mp4
```
> 坑：`-ss` 放 `-i` 前 + `-c copy` 起点会落到最近关键帧（首段偏差/黑帧）；要精确必须重编码。

**③ 拼接**
```bash
# 同编码/分辨率/帧率/音频参数（同源切段）→ concat demuxer，最快、无重编码
for f in segments/*.mp4; do echo "file '$PWD/$f'"; done > list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy out.mp4
# 异构源（分辨率/帧率/编码不同）→ concat filter，必须先 scale+setsar+fps 统一、必然重编码
ffmpeg -i a.mp4 -i b.mp4 -filter_complex \
"[0:v]scale=1920:1080,setsar=1,fps=30[v0];[1:v]scale=1920:1080,setsar=1,fps=30[v1];\
[v0][0:a][v1][1:a]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" -c:v libx264 -crf 18 -c:a aac out.mp4
```
> 坑：异构源用 demuxer + `-c copy` 会音画错乱。

**④ B-roll 按时间段覆盖**
```bash
ffmpeg -i main.mp4 -i broll.mp4 -filter_complex \
"[1:v]scale=1920:1080,setsar=1[bl];[0:v][bl]overlay=enable='between(t,10,14)'[v]" \
-map "[v]" -map 0:a -c:a copy out.mp4
```

**⑤ 画幅重构（横转竖/方，宽高必须偶数）**
```bash
ffmpeg -i in.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920:flags=lanczos,setsar=1" -c:a copy vertical.mp4
ffmpeg -i in.mp4 -vf "crop=ih:ih,scale=1080:1080:flags=lanczos,setsar=1"   -c:a copy square.mp4
```
> 坑：crop 表达式可能算出奇数宽（如 405→404 不可控）；务必随后 `scale` 到明确偶数尺寸或用 `scale=-2`，并加 `setsar=1` 防变形。libx264+yuv420p 要求宽高均偶数。

**⑥ 响度归一（EBU R128，两遍法更准）**
```bash
# 一遍法（快）
ffmpeg -i in.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy out.mp4
# 两遍法：第1遍测量出 JSON（在 stderr），把 measured_* 回填第2遍
ffmpeg -i in.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11:print_format=json -f null -
ffmpeg -i in.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11:measured_I=…:measured_TP=…:measured_LRA=…:measured_thresh=…:offset=…:linear=true -ar 48000 -c:v copy out.mp4
```
> 坑：loudnorm 默认是 `I=-24:LRA=7:TP=-2`，**必须显式给目标**（语音/通用 I=-16、YouTube I=-14、广播 I=-23）；measured_* 必须来自该文件实测，不能照抄示例。

**⑦ BGM 让位人声（侧链压缩 ducking）**
```bash
ffmpeg -i video_with_voice.mp4 -i bgm.mp3 -filter_complex \
"[0:a]asplit=2[voice_sc][voice_mix];\
[1:a][voice_sc]sidechaincompress=threshold=0.03:ratio=8:attack=20:release=300:makeup=1[bgm_ducked];\
[voice_mix][bgm_ducked]amix=inputs=2:duration=first:normalize=0[aout]" \
-map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 192k out.mp4
```
> 坑：sidechaincompress 第一输入=被压的 BGM、第二=人声侧链（接反会压人声）；用 `asplit` 把人声复制两份（一份触发、一份进 amix）；`amix` 设 `normalize=0` 防整体衰减。阈值/makeup 是线性幅度非 dB。

**⑧ 淡入淡出 / 转场**
```bash
# 音频淡入淡出（淡出 st 需手动算 = 总时长-淡出时长）
ffmpeg -i in.mp4 -af "afade=t=in:st=0:d=1,afade=t=out:st=29:d=1" -c:v copy out.mp4
# 画面交叉转场 xfade（两路须同分辨率/SAR/帧率；offset≈第一段时长-转场时长）
ffmpeg -i a.mp4 -i b.mp4 -filter_complex \
"[0:v]scale=1920:1080,setsar=1,fps=30[v0];[1:v]scale=1920:1080,setsar=1,fps=30[v1];\
[v0][v1]xfade=transition=fade:duration=1:offset=4[v];[0:a][1:a]acrossfade=d=1[a]" \
-map "[v]" -map "[a]" -c:v libx264 -crf 18 -c:a aac out.mp4
```

**⑨ 调色：LUT / S曲线 / 白平衡曝光**
```bash
ffmpeg -i in.mp4 -vf lut3d=look.cube -c:a copy graded.mp4           # 套 3D LUT（先确认色彩空间匹配）
ffmpeg -i in.mp4 -vf "curves=master='0/0 0.25/0.18 0.75/0.82 1/1'" -c:a copy out.mp4   # S 曲线
ffmpeg -i in.mp4 -vf "eq=brightness=0.06:contrast=1.12:saturation=1.1" -c:a copy out.mp4  # 曝光/对比/饱和
ffmpeg -i in.mp4 -vf "colorbalance=rm=-0.1:bm=0.1" -c:a copy out.mp4   # 白平衡/色偏（中间调去暖偏冷）
```

**⑩ 场景检测 / 静音检测（无字幕时定切点）**
```bash
# 场景切换（FFmpeg 8.x 用 -fps_mode vfr，旧的 -vsync vfr 已弃用）
ffmpeg -i in.mp4 -vf "select='gt(scene,0.3)',showinfo" -fps_mode vfr -f null - 2>&1 | grep showinfo
# 静音检测（找气口/死气；结果在 stderr）
ffmpeg -i in.mp4 -af silencedetect=noise=-30dB:d=0.5 -f null - 2>&1 | grep silence
# 直接删除静音段
ffmpeg -i in.mp4 -af "silenceremove=stop_periods=-1:stop_duration=0.5:stop_threshold=-30dB" out.mp4
```

## 四、Remotion 工作流要点（已核 v4 文档）

1. 把转写得到的 `segments`（已换算成帧）作为 **props** 传入；每个保留段一个 `<Sequence from durationInFrames>`，B-roll/字幕作为上层子组件叠加。
2. `<Sequence>` 内 `useCurrentFrame()` **从该段 from 重新归零**——字幕进出场用"段内相对帧"。
3. `durationInFrames` 省略时默认 **Infinity**（子组件挂到合成结束）；要限制时长**必须显式传**，否则多段全程叠着。`from` 自 v3.2.36 起可选（默认 0）。
4. 动效用 `interpolate(frame,[in],[out],{extrapolateLeft:'clamp',extrapolateRight:'clamp'})`——**默认 `extend` 会外推**（opacity>1、scale 异常），动效几乎都要 `clamp`；**只动 transform/opacity**。
5. BGM：经典 `<Audio>` 从 `'remotion'` 导入（无需装包；核心包较新版本已更名为 `<Html5Audio>`，`<Audio>` 为旧别名仍可用）；新 `<Audio>` 从 `'@remotion/media'` 导入（实验性、需 `npm i @remotion/media`，client-side/web-renderer 场景须用它）。`src` 用 `staticFile('public下文件')` 或 URL；`volume` 可传按帧函数。版本命名以当时官方文档为准。
6. 渲染（位置参数语序固定）：`npx remotion render <entry-point?> <composition-id> <output?>`，entry 可省（自动探测）、output 省则进 `out/`。批量出片用 `--props=props.json`。
7. 细节查 skill `remotion-video-creation`。

## 五、可复现代码栈：转写 + 节拍（对接 `02`/`04`/`08`）

```python
# 转写：faster-whisper → segments=[{text,start,end}]（知识02产物）
from faster_whisper import WhisperModel
model = WhisperModel("large-v3", device="cpu", compute_type="int8")
seg_iter, info = model.transcribe("aroll.wav", language="zh", word_timestamps=True, vad_filter=True)
segments = [{"text": s.text.strip(), "start": s.start, "end": s.end} for s in seg_iter]  # 生成器，必须迭代才转写
# 词级文本属性是 word.word（不是 .text）；SRT 毫秒用逗号 HH:MM:SS,mmm
```
```python
# 节拍：librosa → 节拍秒 → 视频帧号（知识04/08卡点）
import librosa
y, sr = librosa.load("bgm.mp3", sr=None, mono=True)
tempo, beats = librosa.beat.beat_track(y=y, sr=sr, hop_length=512, units="time")  # 直接得"秒"
bpm = float(tempo[0]) if hasattr(tempo, "__len__") else float(tempo)  # 0.10+ tempo 是 ndarray，要取标量
beat_frames = [round(t * fps) for t in beats]      # 秒→视频帧（用 round 防累积漂移）
downbeats   = beat_frames[::4]                     # 每4拍≈强拍（踩节拍一）；全部=踩节拍二
```
> 坑：librosa 的"帧"是 STFT 分析帧（hop_length 决定），与视频 fps 帧是两码事——先 `frames_to_time`/`units='time'` 得秒，再乘 fps。`beat_track` 给全局平均 BPM，变速曲目会漂，强卡点人工校锚点。秒↔帧统一用 round，fps 必须与 Remotion Composition 一致。

## 六、查找指引（信息不足时去查什么）

- 该工具的官方文档（经 Context7 或对应 skill：`remotion-video-creation`/`manim-video`/`videodb`/`fal-ai-media`）。
- 滤镜/API 的精确参数（FFmpeg `ffmpeg -h filter=`、Remotion/librosa/faster-whisper 文档），并注意版本差异（如 `-fps_mode` 取代 `-vsync`、`@remotion/media` 仍实验性）。
- **绝不臆造参数**：不确定就先查、先跑小样验证；`measured_*` 等需实测的值不能照抄示例。
