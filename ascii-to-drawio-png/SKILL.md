---
name: ascii-to-drawio-png
description: 将 ASCII / Mermaid 流程图转换为有设计感的 draw.io 图形并导出 PNG（默认 16:9 横向、白底黑字、卡片化）。触发词：ASCII转图/mermaid转图/流程图转PNG/ascii to drawio/mermaid to png/转换流程图/drawio导出。
---

# 流程图转 Draw.io PNG

将 ASCII art 或 Mermaid 流程图转为 draw.io XML，按统一设计标准导出 PNG（默认 16:9 横向、白底黑字、卡片化），并可选替换源 Markdown 中的代码块。

## 触发条件

- 用户提供 ASCII 流程图或 Mermaid 代码块（在 md 文件中，或直接贴在对话里），要求转 drawio / PNG
- 提到：ASCII转图 / mermaid转图 / 流程图转PNG / drawio导出 / 转换流程图 等

## 输入形式（两种）

1. **md 文件内的代码块**：代码围栏后无语言标注 / 标注 `text`（ASCII art），或标注 `mermaid`
2. **对话内直接贴的代码块**：用户把 ASCII / mermaid 直接贴在消息里，无源文件 → 输出到当前工作目录（或指定目录）的 `assets/`，**跳过"替换源 md"步骤**

若未给路径也未贴代码，先问一句要处理哪个文件 / 要不要直接贴。

---

## 设计标准（默认风格，务必遵守，避免反复纠正）

- **白底黑字**：背景透明或纯白；主字体 `#111111`。**禁止灰字配灰底**（低对比看不清）
- **字号下限**：节点标题 ≥ 16px、正文 ≥ 13px、序号徽标 ≥ 16px。**不要用 ≤11px 正文**
- **卡片化节点**：`fillColor=#ffffff;rounded=1;arcSize=8~10;shadow=1` + 彩色 `strokeColor`；高亮节点 `strokeWidth=2`，普通 `strokeWidth=1`
- **可选 accent bar**：节点左侧 8px 彩色竖条（同色 fillColor 窄矩形）做状态标识
- **序号徽标**：`ellipse;fillColor=<主题色>;strokeColor=none;fontColor=#ffffff;fontSize≥16;fontStyle=1`
- **不要加主标题/副标题装饰**：只画流程图本身。分区标签（swimlane header）属结构、可保留
- **HTML 富文本分层**：`<b style="font-size:16px;color:#111">标题</b><br><span style="font-size:13px;color:#333">细节</span>`

## 布局与比例（默认 16:9 横向 —— 最强诉求）

用户明确偏好 16:9 长方形，**不要太宽也不要太高**：

- **目标比例**：内容包围盒 宽:高 ≈ 1.7~1.8（16:9 = 1.78）
- ⚠️ **关键机制**：draw.io CLI 导出会**自动 crop 到内容包围盒**，`pageWidth/pageHeight` **不决定**最终尺寸。最终图片比例 = 所有 cell 的包围盒比例 → 要 16:9 必须**规划节点坐标使包围盒宽:高 ≈ 1.78**
- **节点多时横向展开**：多列 / 蛇形 / U 型 / 泳道，避免一直向下堆叠成窄长条
- **忠实方向但控高**：mermaid `flowchart LR` → 横向多列；`flowchart TD` → 纵向单列若超 ~6 节点，改成双列蛇形控制高度
- 导出后用 `file xxx.png` 确认像素，比例偏离 1.6~1.9 → 回炉调坐标重导

## 配色语义（按角色，不随机）

| 角色 | strokeColor | 适用 |
|------|-------------|------|
| 数据接入/源 | `#6c8ebf` 蓝 | pipe、source、输入消息 |
| 解析/转换 | `#9673a6` 紫 | parser、udf、转换工具 |
| 共识/核心 | `#ff8c00` 橙 (2px) | raft、hub 进程、关键调度 |
| 写入/成功 | `#4a9050` 绿 (2px) | 内存表、终态产出 |
| 过滤/校验 | `#d79b00` 琥珀 | filter、校验、中间产物 |
| 终点/扣款 | `#b85450` 红 | 最终动作、警告 |
| 基础设施 | `#8a97a5` 灰 | 队列、脚本、本地文件 |

- mermaid `style` 高亮节点 → **忠实保留**其 fill/stroke/stroke-width
- 连线颜色随上游节点主题色过渡
- **虚线边** `dashed=1;dashPattern=6 4;endArrow=open;endFill=0` 用于校验/反馈等非主流程，与实线数据流区分

---

## 执行流程

### 步骤 1：识别图源

**ASCII**（md 内或对话内）：代码块无语言标注 / `text`；含 box-drawing 字符 `│┌┐└┘├┤┬┴─▼▲►◄→←↓↑`；排除含 import/include/#define/function/class/def 等关键字的真代码块。

**Mermaid**：代码块标注 `mermaid`，支持 `flowchart`/`sequenceDiagram`/`stateDiagram` 等。提取节点 id、label（含 `<br/>`）、边（`-->` / `-.->` / `&` 分叉合流）、`subgraph`、`style`。

用 Grep/Read 取文件内容；对话内贴的直接用。

### 步骤 2：确认范围

md 多图时列出候选（`[行 XX-YY] 描述`）询问全部/编号；**对话内单图直接做**，不必反复确认布局（先按默认 16:9 + 设计标准产出，再听反馈微调）。

### 步骤 3：生成 draw.io XML

**通用要求**：完整 `mxfile > diagram > mxGraphModel > root`；id 从 2 递增（0/1 给 root）；边用 `edgeStyle=orthogonalEdgeStyle;rounded=1`；中文节点宽 ≥ `字符数*14` 高 ≥ 40；严格按上面的【设计标准】【布局与比例】【配色语义】产出。

**Mermaid → drawio 映射**：
- `flowchart LR/TD` → 按 LSB 决定主排布方向，但**最终包围盒仍要 ≈16:9**（TD 超 6 节点改双列蛇形）
- `subgraph X["标题"]` → 分组容器 `rounded=1;arcSize=5~6;dashed=1;dashPattern=8 4;verticalAlign=top;spacingTop=6;fillColor=<浅色>;strokeColor=<中色>`，标题在顶部，子节点坐标落在容器范围内（`parent="1"` 不必真嵌套）
- `A --> B` → 实线 `endArrow=block;endFill=1;strokeWidth=2`
- `A -.label.-> B` → 虚线 `dashed=1;endArrow=open;endFill=0`，label 写在 edge value 上 + `labelBackgroundColor`
- `A --> B & C` / `A & B --> C` → 多条 edge，分叉点用上游节点的多个 exitX/entryX 错开避免重叠
- `style A fill:#x;stroke:#y;stroke-width:Npx` → 写进对应节点 style；保留 fill（作 fillColor 或浅底）、stroke、stroke-width

**节点样式模板**（卡片）：
```
style="rounded=1;arcSize=10;whiteSpace=wrap;html=1;fillColor=#ffffff;strokeColor=#6c8ebf;strokeWidth=1;shadow=1;fontSize=13;verticalAlign=middle;spacingLeft=14;align=left;"
```
**边样式模板**（实线）：
```
style="edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;endArrow=block;endFill=1;strokeColor=#6c8ebf;strokeWidth=2;"
```

### 步骤 4：保存 .drawio

源 md 同级（或工作目录）建 `assets/`，写入 `diagram_NN_简短描述.drawio`（NN 两位序号，描述英文小写下划线）。

```bash
mkdir -p <目录>/assets
```

### 步骤 5：导出 PNG

CLI：`/Applications/draw.io.app/Contents/MacOS/draw.io`

```bash
# 透明背景
.../draw.io --export --format png --transparent --scale 2 --output out.png in.drawio
# 白底（去掉 --transparent）
.../draw.io --export --format png --scale 2 --output out.png in.drawio
```

⚠️ **坑**：`--background "#ffffff"` 参数在当前 draw.io 版本会误报 `input file not found`，**不要用**；要白底就省略 `--transparent`。

导出后 **必须** `file out.png` 确认像素比例 ∈ [1.6, 1.9]，偏离则回步骤 3 调坐标重导。

报错排查：`ls /Applications/draw.io.app` 确认安装；Linux 加 `--no-sandbox`。

### 步骤 6：替换源 Markdown（仅 md 文件输入时）

用 Edit 把原代码块替换为（保留原文便于回溯）：

```markdown
<!-- Original diagram:
原始内容
-->
![图片描述](./assets/diagram_NN_xxx.png)
```

对话内贴的图：跳过此步。

---

## 注意事项

- 一次只处理一个 md 文件 /一张图
- 默认就按 16:9 + 设计标准产出，不要等用户纠正；产出后在回复里**报告实际像素和比例**
- 复杂拓扑（hub 多入多出、反馈回路）优先把核心节点放视觉中心，让边自然发散
- mermaid 高亮节点（`style`）一定忠实保留语义色
