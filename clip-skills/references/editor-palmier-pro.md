---
name: editor-palmier-pro
description: >
  用 Palmier Pro 真正动手剪辑。Palmier Pro 是 macOS 原生、为 AI 打造的开源剪辑器，自带 MCP 服务器，
  Agent 可经 MCP 直接读工程、切分/裁剪/排轨、调速度/不透明度/变换、生成并放置图像/视频/音频、导出。
  当目标软件是 Palmier Pro、需要把剪辑决策落地为真实时间线操作时用本技能。建议在独立子 Agent 中加载，避免与其他软件上下文混淆。
---

# Palmier Pro 执行技能

把 `jianji-knowledge` 的剪辑决策，落地为 Palmier Pro 时间线上的真实操作。本技能负责"怎么动手"，不负责"怎么判断"。

## 一、先决条件 & 连接

- **环境**：Palmier Pro 是 macOS 原生、为 AI 打造的开源视频编辑器（开源仓库 `github.com/palmier-io/palmier-pro`，Swift）。具体 macOS 版本/芯片要求**以官方文档为准**（面向较新 macOS / Apple Silicon）。**编辑器 + MCP 服务器免费、无需登录**；只有生成式 AI（视频/图片/音频/超清/Palmier chat）按 credits 计费、需登录订阅。
- **启用 MCP（务必先开 App 再连）**：先打开 Palmier Pro App（它在本机起 MCP server，默认端点 `http://127.0.0.1:19789/mcp`），再连接：
  - Claude Code：`claude mcp add --transport http palmier-pro http://127.0.0.1:19789/mcp`
  - 或在 App 内 `Help → MCP Instructions` 一键安装到 Cursor / Claude Desktop。
- **生成能力边界**：`get_timeline` 返回 `canGenerate=false` 时所有生成/超清工具会失败（需登录订阅）；但 MCP server 本身对外部 client 免费。
- **导出**：MP4（H.264/H.265/ProRes）与 Premiere/达芬奇可用的 NLE XML（以实际工具为准）。

## 二、第一步永远是"发现工具"（不要编工具名）

Palmier 的 MCP 工具会以 `mcp__palmier...__*` 形式出现，其确切名称/参数以运行时为准。**动手前先发现真实工具，再按其 schema 调用**：

```
ToolSearch("palmier timeline")        # 找 Palmier 的 MCP 工具
ToolSearch("select:<发现的工具名>")    # 载入其参数 schema
```

> 红线：**绝不臆造工具名或参数**。查不到就说明 MCP 没连上——提示用户在 App 内开启并一键安装，而不是假装能操作。

预期能力（以实际工具为准）：读取工程/时间线上下文、导入素材、在轨道上放置/移动片段、split/trim、调 speed/opacity/transform、生成并落地 AI 素材、导出。

## 三、自主出初剪的标准动作（对接 orchestrator 流水线）

输入＝orchestrator 传来的 `audio_timeline` + B-roll 匹配映射表 + 画幅/时长。

1. **建工程/对画幅**：新建或打开工程，设为目标画幅（横/竖屏）。
2. **铺 A-roll**：导入口播/旁白主素材到主轨；按 `audio_timeline` 的切点 split，删掉气口/卡顿段（保留段按 `[t_in,t_out]` 排好）。
3. **贴 B-roll**：按匹配映射表，把每段 B-roll 放到**上层轨道**对应 `[t_in,t_out]`；整条 B-roll 轨静音；硬切处用放大(transform 缩放)遮蔽。
4. **声音**：放 BGM 到音频轨，在节拍点对齐切换；按 `04` 做淡化/压音让位人声。
5. **调色**：加调整层/套 LUT，按 `06` 的参数增量调（先确认 Palmier 是否支持该操作，不支持则改导出到 Premiere/达芬奇处理并说明）。
6. **导出**：MP4 成片；需要交接专业 NLE 时导出 NLE XML。

## 四、黑场红线

- 流水线遗留的**黑场 / 无法覆盖的硬切**：在时间线上**标注位置清单**交回用户手动裁切，**不自动删主轨**。
- 大批量 split/替换前，先把"将执行的操作清单"回报 orchestrator/用户，留叫停机会。

## 五、生成式素材（可选）

- Palmier 支持经 MCP 在时间线上**生成并放置** B-roll、空镜、音效、配音；具体可用的生成模型清单**以官方文档/运行时 `ToolSearch` 为准**，不要把某一时刻的模型名写死（与本包"不臆造软件能力"红线一致）。
- 缺 B-roll 时可作为"万能空镜"来源（见 `03` 遮蔽硬切）；生成消耗 credits、需订阅，先与用户确认。

## 六、查找指引（信息不足时去查什么）

- App 内 `Help → MCP Instructions`：连接与工具清单。
- 官方文档 `palmier.io/docs`：生成、时间线编辑、chat、MCP、导出教程。
- GitHub `palmier-io/palmier-pro`：开源实现与 FAQ。
- 运行时 `ToolSearch` 列出的真实 MCP 工具与参数 = 最终依据。
