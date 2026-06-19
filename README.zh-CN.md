<div align="center">

# ClipSkills

**专为 AI Agent 打造的视频剪辑技能 —— 把"点哪个按钮"重写为"做什么决策"。**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Release](https://img.shields.io/github/v/release/appergb/ClipSkills?color=blue)](https://github.com/appergb/ClipSkills/releases)
[![Stars](https://img.shields.io/github/stars/appergb/ClipSkills?style=social)](https://github.com/appergb/ClipSkills/stargazers)
![Claude Agent Skills](https://img.shields.io/badge/Claude-Agent%20Skills-D97757?logo=anthropic&logoColor=white)
![Skill validated](https://img.shields.io/badge/skill-validated-brightgreen)

![FFmpeg](https://img.shields.io/badge/FFmpeg-tested%208.1.1-007808?logo=ffmpeg&logoColor=white)
![Remotion](https://img.shields.io/badge/Remotion-v4-0B84F3?logo=react&logoColor=white)
![Whisper](https://img.shields.io/badge/Whisper-faster--whisper-412991?logo=openai&logoColor=white)
![Python](https://img.shields.io/badge/Python-librosa-3776AB?logo=python&logoColor=white)
![Palmier Pro](https://img.shields.io/badge/Palmier%20Pro-MCP-111111)
![Markdown](https://img.shields.io/badge/format-Markdown%20Skill-000000?logo=markdown&logoColor=white)

[English](README.md) · **简体中文**

</div>

---

> 照片/截图等素材后续补充；当前 README 为纯文字版。

## 这是什么

`ClipSkills` 提供一个符合 [Claude Agent Skills 规范](https://claude.com/claude-code) 的标准技能 **`clip-skills`**，让 AI Agent 真正具备视频剪辑能力：能**答疑、能反问、能自主出初剪**，并把决策落地到真实软件。

核心设计：**判断逻辑（怎么剪、为什么、好坏怎么判）与软件操作（用什么剪）彻底分离**，每个剪辑软件用独立子 Agent 执行，避免上下文污染。

## 为什么"专为 Agent 打造"

传统教程教人**点哪个按钮**；Agent 需要的是**在什么情况下、依据什么、做什么决定、输出什么**。`clip-skills` 的每条规则都按此重写：

| 人类做法 | clip-skills 里的 Agent 决策 |
|:---|:---|
| 看波形、手动 Ctrl+B 切 | 取带时间戳字幕 → 按规则识别气口/卡顿/重复 → 输出切点列表 |
| 双时间线肉眼捞 B-roll | 语义检索素材库 → 输出"字幕段 ↔ 候选 B-roll"匹配表 |
| 拖色轮靠眼睛 | 用示波器量化曝光/白平衡 → 输出参数增量 |
| 凭手感踩点 | 检测节拍/重音时间戳 → 在节拍点切换 |
| 反复试错重跟踪 | 判断二维/平面、选占比最大帧、标失败帧处理 → 输出跟踪指令 |

`SKILL.md` 是总入口（三种模式、反问清单、音频驱动流水线、软件路由、知识索引、行为红线）；具体决策在 `references/` 按需读取——这是 Skill 规范的**渐进式披露**。

## 结构

```
ClipSkills/                         # 本仓库（项目名）
├── README.md                       # English
├── README.zh-CN.md                 # 简体中文（本文件）
├── LICENSE                         # MIT（仅覆盖原创表达，见"声明"）
└── clip-skills/                    # 技能本体（文件夹名 = 技能名，kebab-case）
    ├── SKILL.md                    # 入口：name/description + 总调度 + 索引
    ├── LICENSE.txt
    └── references/                 # 按需加载的捆绑资源
        ├── 01..12-*.md             # 12 册软件无关知识内核
        ├── editor-palmier-pro.md   # 执行指南 —— Palmier Pro（MCP）
        ├── editor-code-video.md    # 执行指南 —— FFmpeg / Remotion / Whisper+librosa
        └── editor-other-nle.md     # 执行指南 —— 剪映 / Premiere / 达芬奇
```

12 册知识内核：视频分类与剪辑逻辑 · 音频驱动剪辑 · 素材 A/B 分类与替换 · 声音与音乐衔接 · 节奏与结构 · 调色 · 关键帧与运动 · 变速与帧率 · 抠像蒙版与合成 · 跟踪与动效包装 · 多机位与采访 · 导出与交付。

## 经验来源

ClipSkills 吸取**各大剪辑博主与专业后期**的实战经验并系统化为可执行决策规则。课程化主干内化自**影视飓风《剪辑全能必修课》**（教学软件剪映专业版 / CapCut Desktop），并交叉验证通用剪辑理论与官方文档（FFmpeg、Remotion、Whisper、librosa）。"时钟理论 / 波峰制"等创作者经验已明确标注为**经验法则**而非普适数据。

## 搭配工具与环境

`clip-skills` 是一组 **Markdown 技能文件**（不含可执行代码）。要发挥全部能力，按需搭配：

- **Agent 运行环境（必需）**：[Claude Code](https://claude.com/claude-code) 或任何兼容 Agent Skills 规范的框架。
- **落地执行路径（三选一或按场景路由）**：
  - **Palmier Pro（MCP 全自动）** —— macOS + [Palmier Pro](https://www.palmier.io/)（编辑器与 MCP 免费）。连接：`claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp`
  - **代码流水线（最可复现）** —— `ffmpeg`（≥6，命令在 8.1.1 实测）、Node + [Remotion](https://www.remotion.dev/)、Python `faster-whisper`（转写）、`librosa`（节拍检测）。
  - **其他 NLE（操作清单）** —— 剪映 / Premiere / 达芬奇。剪映无公开 API → Agent 产出"导演级操作清单 + 参数"；达芬奇/Premiere 可脚本化。
- **可选增强**：ElevenLabs（TTS）、fal.ai（音乐/音效/图像）、VideoDB —— 缺素材时补齐。

> 红线：本技能**不臆造软件能力** —— Palmier 工具运行时用 `ToolSearch` 实查，FFmpeg/Remotion 参数以官方文档为准。

## 安装

```bash
# 方式一 —— .skill 安装包（从 Releases 下载）
unzip -o clip-skills.skill -d ~/.claude/skills/

# 方式二 —— 源码
git clone https://github.com/appergb/ClipSkills.git
cp -R ClipSkills/clip-skills ~/.claude/skills/
```

验证：对 Agent 说"帮我把这条口播按音频铺上 B-roll"，应触发 `clip-skills`。

## 关键约定（红线）

- **黑场不自动裁**：未覆盖的硬切/空缺标注交用户，不删主轨。
- **先反问后动手**：变量没齐不硬剪。
- **耗算力操作后置**：抠像/跟踪/补帧/变速放粗剪定稿后做，做完先导出备份。
- **经验法则要标注**：时钟理论/波峰制等已标明非普适数据。
- **taste 层交回用户**：Agent 清掉重复劳动，最终创作取舍由人定。

## 声明与许可

采用 **MIT 许可**（见 [LICENSE](LICENSE)）。MIT 仅覆盖本仓库的**原创表达**（技能结构、决策规则、编排、配方、文档文字），**不**主张其所系统化的底层剪辑技法——该知识致谢影视飓风课程与广大创作者、版权归各原作者所有。本仓库为独立的学习/研究用途二次创作，与任何课程或厂商无隶属或背书关系。

---

<div align="center">
<sub>ClipSkills · 让 Agent 从"会用工具"走向"会构思结构"。</sub>
</div>
