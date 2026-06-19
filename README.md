# ClipSkills · 专为 AI Agent 打造的剪辑技能

> 把"人怎么点按钮"重写为"Agent 怎么做决策"——一个让 AI Agent 真正具备视频剪辑能力的标准 **Agent Skill**。
> 能**答疑、能反问、能自主出初剪**，并把剪辑决策落地到真实软件。

本仓库提供一个符合 [Claude Agent Skills 规范](https://claude.com/claude-code) 的标准技能 **`clip-skills`**：判断逻辑（怎么剪、为什么这么剪、怎么判断好坏）与软件操作（用什么剪）彻底分离，每个剪辑软件用独立子 Agent 执行，避免上下文污染。

```
ClipSkills/                         ← 本仓库（项目名）
├── README.md                       ← 你正在看的说明
└── clip-skills/                    ← 技能本体（文件夹名 = 技能名）
    ├── SKILL.md                    ← 技能入口：name/description + 总调度逻辑 + 知识索引
    └── references/                 ← 按需加载的捆绑资源（渐进式披露）
        ├── 01-视频分类与剪辑逻辑.md  … 12-导出与交付.md   （12 册软件无关知识内核）
        ├── editor-palmier-pro.md   （Palmier Pro 执行指南）
        ├── editor-code-video.md    （FFmpeg / Remotion / Whisper+librosa）
        └── editor-other-nle.md     （剪映 / Premiere / 达芬奇）
```

> 技能名为 `clip-skills`（kebab-case 小写）——这是 Agent Skill 规范对 `name` 的硬性要求（仅允许小写字母/数字/连字符），仓库名 `ClipSkills` 为项目展示名。

---

## 为什么"专为 Agent 打造"

传统教程教人**点哪个按钮**；Agent 需要的是**在什么情况下、依据什么、做什么决定、输出什么结果**。`clip-skills` 的每一条知识都按这个范式重写：

| 人类做法 | clip-skills 里的 Agent 决策 |
|:---|:---|
| 看波形找气口、手动 Ctrl+B 切 | 取带时间戳字幕 → 按规则识别气口/卡顿/重复 → 输出切点列表 |
| 双时间线"筛子法"肉眼捞 B-roll | 用字幕语义检索素材库 → 输出"字幕段 ↔ 候选 B-roll"匹配表 |
| 拖色轮、看示波器调色 | 用示波器/直方图量化判断曝光与白平衡 → 输出参数增量 |
| 凭手感踩点、卡鼓点 | 检测音频节拍/重音时间戳 → 在节拍点对齐切换 |
| 肉眼找跟踪特征点、反复重跟 | 判断二维/平面、选占比最大帧、标失败帧处理 → 输出跟踪指令 |

`SKILL.md` 是总入口（三种工作模式、反问清单、音频驱动流水线、软件路由、知识索引、行为红线）；具体决策在 `references/` 的 12 册按需读取——这就是 Skill 规范的**渐进式披露**（metadata → SKILL.md 正文 → 按需 references）。

## 经验来源

ClipSkills 吸取了**各大剪辑博主与专业后期的实战经验**，并将其系统化为可执行的决策规则，覆盖：音频驱动剪辑与口播精剪、蒙太奇/景别/匹配剪辑/节奏结构、J/L-cut/侧链/卡点/音效、一级调色（Log/LUT/示波器/S曲线/色彩克隆）、关键帧与缓动、变速与补帧、抠像/蒙版/混合模式合成、二维/平面跟踪与文字数字动效包装、多机位导播切换、导出交付与平台画幅。

> **致谢与说明**：本技能的课程化知识主干系统内化自【影视飓风】《剪辑全能必修课》（教学软件为剪映专业版 / CapCut Desktop），并交叉验证了通用剪辑理论与官方文档（FFmpeg、Remotion、Whisper、librosa 等）。其中"时钟理论 / 波峰制 / 踩梯子"等为影视飓风的创作者经验框架，已在分册中明确标注为"经验法则"而非普适数据。本仓库为学习/研究用途的二次创作（决策逻辑与 Skill 工程为原创），非任何课程或软件的官方产物；剪辑经验版权归各原创作者所有。

---

## 需要搭配的工具与环境

`clip-skills` 本身是一组 **Markdown 格式的 Skill 文件**（`SKILL.md` + `references/`），不含可执行代码。要发挥全部能力，按需搭配：

### 1) Agent 运行环境（必需）

- **[Claude Code](https://claude.com/claude-code)** 或任何兼容 Agent Skills 规范（`SKILL.md` + frontmatter）的 Agent 框架。技能靠 `description` 触发、靠子 Agent 分派执行。
- 知识内核纯文本即可使用（答疑、出剪辑方案）；要"自主动手"则需下面任一落地路径。

### 2) 落地执行路径（三选一或按场景路由）

| 路径 | 适用 | 依赖 |
|:---|:---|:---|
| **Palmier Pro（MCP 全自动）** | 让 Agent 直接操作时间线 | macOS + [Palmier Pro](https://www.palmier.io/)（开源、编辑器与 MCP 免费）；先开 App 再 `claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp` |
| **代码流水线（最可复现）** | 程序化剪辑、批量、动效 | `ffmpeg`（≥6，本套命令在 8.1.1 实测）、Node + [Remotion](https://www.remotion.dev/)、Python `faster-whisper`（转写）、`librosa`（节拍检测） |
| **其他 NLE（操作清单）** | 剪映·CapCut / Premiere / 达芬奇 | 对应软件本体；剪映无公开 API → Agent 产出"导演级操作清单 + 参数"，达芬奇/Premiere 可经脚本自动化 |

### 3) 可选增强

- 生成层：**ElevenLabs**（TTS 旁白）、**fal.ai**（音乐/音效/图像）、**VideoDB**（生成与索引）——缺素材时补齐。
- 相关 Skill：`remotion-video-creation`、`fal-ai-media`、`videodb`、`manim-video`。

> 红线：本技能**不臆造软件能力**——Palmier 工具运行时用 `ToolSearch` 实查，FFmpeg/Remotion 参数以官方文档为准，其他软件接口先查再用。

---

## 安装 / 激活

```bash
git clone https://github.com/appergb/ClipSkills.git
# 把技能本体放进 skills 目录（全局级 ~/.claude/skills/ 或项目级 <项目>/.claude/skills/）
cp -R ClipSkills/clip-skills ~/.claude/skills/
```

Palmier Pro 用户再：先打开 App（起本机 MCP server）→ `claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp`。

**验证**：对 Agent 说一句"帮我把这条口播按音频铺上 B-roll"，应触发 `clip-skills` 进入流程。

---

## 关键约定（红线）

- **黑场不自动裁**：未覆盖的硬切/空缺标注位置交用户手动处理，不删主轨（人机分界）。
- **先反问后动手**：变量没齐不硬剪。
- **耗算力操作后置**：抠像/跟踪/补帧/变速放粗剪定稿后做，做完先导出备份。
- **经验法则要标注**：时钟理论/波峰制/时长策略等创作者经验已标明非普适数据。
- **taste 层交回用户**：Agent 清掉重复劳动，最终创作取舍由人定。

---

*ClipSkills · 让 Agent 从"会用工具"走向"会构思结构"。*
