<div align="center">

# ClipSkills

**A video-editing skill built for AI Agents — turning "which button to click" into "what decision to make."**

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

**English** · [简体中文](README.zh-CN.md)

</div>

---

> Photos / screenshots will be added later; this README is text-only for now.

## What is this

`ClipSkills` provides **`clip-skills`**, a standard [Claude Agent Skill](https://claude.com/claude-code) that gives an AI Agent real video-editing capability. It can **answer questions, ask clarifying questions back, and autonomously produce a first cut**, then land those decisions in real software.

Its core design: **decision logic (how to cut, why, how to judge quality) is fully separated from software operation (what to cut with)**, and each editing app runs in its own sub-agent to avoid context pollution.

## Why "built for Agents"

Traditional tutorials teach humans *which button to press*. An Agent needs *under what condition, based on what, what decision, what output*. Every rule in `clip-skills` is rewritten that way:

| Human workflow | The Agent decision in clip-skills |
|:---|:---|
| Eyeball the waveform, hand-cut with Ctrl+B | Take timestamped subtitles → flag breaths/stumbles/repeats by rule → output a cut list |
| Hunt for B-roll by eye on a dual timeline | Semantic search of the asset library → output a "subtitle-segment ↔ candidate B-roll" map |
| Drag color wheels by eye | Quantify exposure/white-balance with scopes → output parameter deltas |
| Tap beats by feel | Detect beat/accent timestamps → switch shots on the beat |
| Re-track planar tracks by trial and error | Pick 2D vs planar, choose the largest-area frame, handle failure frames → output a tracking spec |

`SKILL.md` is the entry point (three modes, clarification checklist, audio-driven pipeline, software routing, knowledge index, red lines). Detailed decisions live in `references/`, loaded on demand — Skill **progressive disclosure** (metadata → SKILL.md body → references as needed).

## Structure

```
ClipSkills/                         # this repo (project name)
├── README.md                       # English (this file)
├── README.zh-CN.md                 # 简体中文
├── LICENSE                         # MIT (original expression only — see Disclaimer)
└── clip-skills/                    # the skill (folder name = skill name, kebab-case)
    ├── SKILL.md                    # entry: name/description + orchestration + index
    ├── LICENSE.txt
    └── references/                 # bundled resources, loaded on demand
        ├── 01..12-*.md             # 12 software-agnostic knowledge modules
        ├── editor-palmier-pro.md   # execution guide — Palmier Pro (MCP)
        ├── editor-code-video.md    # execution guide — FFmpeg / Remotion / Whisper+librosa
        └── editor-other-nle.md     # execution guide — CapCut / Premiere / DaVinci
```

The 12 knowledge modules: video typing & edit logic · audio-driven editing · A/B-roll classification & replacement · sound & music transitions · pacing & structure · color grading · keyframes & motion · speed-ramp & frame rate · keying/masking/compositing · tracking & motion-graphics packaging · multicam & interviews · export & delivery.

## Knowledge & experience sources

ClipSkills distills the practical experience of **major editing creators and professional post-production**, systematized into executable decision rules. The course backbone is internalized from the **YingShiJuFeng (影视飓风) "Editing Mastery Course"** (taught on CapCut Desktop / 剪映专业版), cross-checked against general editing theory and official docs (FFmpeg, Remotion, Whisper, librosa). Creator heuristics such as "clock theory / wave-peak pacing" are explicitly labeled as *rules of thumb*, not universal data.

## Tools & environment

`clip-skills` is a set of **Markdown skill files** (no executable code). To unlock full capability, pair it with:

- **Agent runtime (required):** [Claude Code](https://claude.com/claude-code) or any framework supporting the Agent Skills spec (`SKILL.md` + frontmatter).
- **Execution paths (pick one or route by case):**
  - **Palmier Pro (full auto via MCP)** — macOS + [Palmier Pro](https://www.palmier.io/) (editor & MCP are free). Connect: `claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp`
  - **Code pipeline (most reproducible)** — `ffmpeg` (≥6; commands tested on 8.1.1), Node + [Remotion](https://www.remotion.dev/), Python `faster-whisper` (transcription), `librosa` (beat detection).
  - **Other NLEs (operation list)** — CapCut / Premiere / DaVinci. CapCut has no public API → the Agent emits a "director-grade operation list + parameters"; DaVinci/Premiere can be scripted.
- **Optional:** ElevenLabs (TTS), fal.ai (music/SFX/image), VideoDB — to generate missing assets.

> Red line: the skill **never fabricates software capabilities** — Palmier tools are discovered at runtime via `ToolSearch`; FFmpeg/Remotion params follow official docs.

## Install

```bash
# Option A — .skill package (from Releases)
unzip -o clip-skills.skill -d ~/.claude/skills/

# Option B — from source
git clone https://github.com/appergb/ClipSkills.git
cp -R ClipSkills/clip-skills ~/.claude/skills/
```

Verify: tell your Agent *"lay B-roll over this voiceover based on the audio"* — it should trigger `clip-skills`.

## Conventions (red lines)

- **Never auto-trim black gaps** — uncovered hard cuts/gaps are flagged for the user, never deleted from the main track.
- **Ask before cutting** when variables are missing.
- **Defer compute-heavy ops** (keying/tracking/interpolation/speed) until the rough cut is locked; export a backup afterward.
- **Label heuristics** — creator rules of thumb are marked as non-universal.
- **Taste is the human's** — the Agent clears repetitive work; final creative calls stay with you.

## Disclaimer & License

Licensed under **MIT** (see [LICENSE](LICENSE)). MIT covers the **original expression** here (skill structure, decision rules, orchestration, recipes, prose). It does **not** claim the underlying editing techniques, which are credited to the 影视飓风 course and the wider creator community and remain their owners' IP. This is an independent, educational re-creation, not affiliated with or endorsed by any course or vendor.

---

<div align="center">
<sub>ClipSkills · Take an Agent from "using tools" to "thinking in structure."</sub>
</div>
