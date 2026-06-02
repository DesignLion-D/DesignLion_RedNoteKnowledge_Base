---
title: "用 Obsidian + Claude 打造第二大脑 · 配套资料"
article: article/2026-05-28-second-brain-obsidian-claude/index.md
type: resource
created: 2026-05-28
---

## 说明
本资料是「用 Obsidian + Claude 打造第二大脑」的配套内容，收录文中提及的工具入口和可直接复用的 Prompt 模板。

---

## 工具入口

**Obsidian**
本地优先的 Markdown 笔记软件，这套系统的载体。免费，无厂商锁定，所有笔记都是普通 .md 文件，可被任何 AI 直接读取。
https://obsidian.md

---

## 可复用的 Skill Prompt 模板

以下四个文件直接复制到你的 Obsidian vault 的 `05-skills/` 文件夹即可使用。

---

**ingest.md（入库技能）**

```
# Ingest Skill

## Purpose
把一个新源头处理进 wiki：提取核心想法，
与既有知识建立连接，更新相关 wiki 条目，
并在需要的地方创建新条目。

## Process
1. 先完整读取源头，再做任何操作
2. 识别出值得保留的 3-7 个核心想法
3. 对每个想法：
   a. 检查是否已有 wiki 条目覆盖了它
   b. 如果有，用新信息或新角度更新条目
   c. 如果没有，创建新条目
4. 对每个新条目，找至少 2 个与现有 wiki 条目的连接，
   并添加双向链接
5. 用创建的新条目更新 wiki/index.md
6. 在 wiki/hot.md 里添加一行记录，注明入库了什么

## Quality Standard
入库一个源头后，wiki 应该明显更丰富。
如果入库一个源头没有创造出至少一个之前不明显的新连接，
就再仔细找找。

## What Not to Do
不要创建一个只是回顾源头的摘要文件。
wiki 里存的是想法，不是摘要。提取想法并整合进去。
源头文件本身在 03-sources/ 里就是记录。
```

---

**synthesize.md（综合技能）**

```
# Synthesize Skill

## Purpose
围绕特定问题或话题，组合多个 wiki 条目的知识，
生成一份综合文档。这不是检索，而是推理。

## Process
1. 确定要综合的话题或问题
2. 读取所有相关 wiki 条目（通过标签和链接识别）
3. 寻找：
   - 跨多个源头出现的模式
   - 源头之间的张力或矛盾
   - 问题没有被回答的空白
   - 尚未被明确建立的连接
4. 写一份综合文档：
   - 第一段清晰陈述核心洞察
   - 用具体 vault 条目的证据展开论证
   - 诚实地呈现张力所在
   - 在证据支持某个立场时明确表态
5. 保存到 06-output/ 并从相关 wiki 条目链接过去

## Quality Standard
一份综合文档应该包含至少一个洞察，
这个洞察不存在于任何单个源头，
而是从组合中涌现出来的。
如果它只是多个源头的摘要，就不是综合。
```

---

**connect.md（连接技能）**

```
# Connect Skill

## Purpose
读遍整个 vault，浮现尚未被明确链接的
想法之间的非明显连接。

## Process
1. 读取 wiki/index.md，了解全貌
2. 读取每个 wiki 条目，注意它在主题上
   靠近但尚未连接的内容
3. 对于每一个发现的非明显连接：
   a. 在两个条目里都添加链接
   b. 在每个条目里写一句话解释
      这个连接揭示了什么
4. 整理一份简报：连接了什么、为什么重要

## Quality Standard
只添加真正有启发性的连接。
不是所有条目都有关系。
一个需要勉强解释的连接不值得建立。
```

---

**热缓存更新 Prompt（每次对话结束时使用）**

```
在我们结束这次对话之前，请用以下信息更新 wiki/hot.md：

1. SESSION SUMMARY（3-5 句话）：我们做了什么、完成了什么
2. KEY DECISIONS（要点）：做出的、未来对话应该知道的决定
3. OPEN THREADS（要点）：提出但没有解决的问题，下次继续
4. VAULT CHANGES（要点）：今天在 vault 里添加、更新或连接了什么
5. NEXT SESSION PRIORITY：下次最重要的一件事是什么

用这个新条目替换整个文件，不要追加。
每个条目只反映最近一次对话。
```
