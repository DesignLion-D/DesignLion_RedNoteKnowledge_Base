---
title: "9个可以直接复制的 Prompt 模板，把8小时工作日压到47分钟 · 配套资料"
article: article/2026-05-22-9-claude-cowork-prompt-templates/index.md
type: resource
created: 2026-05-22
---

## 说明
本资料是「9个可以直接复制的 Prompt 模板，把8小时工作日压到47分钟」的配套内容，收录文中全部9个可直接复用的 slash command 模板。括号内字段替换为你自己的数据源后即可使用。

---

## Prompt 模板

**模板1：每日情报简报 /morning-brief**
把隔夜 Gmail、Slack、日历、新闻源压缩成一页早报，10点前需要处理/可等待/噪音三区块。

```markdown
/morning-brief

ROLE: my chief of staff. you read everything overnight and prepare
a one-page briefing.

PULL:
- gmail: last 12 hours, inbox + sent (to see what i committed to)
- slack: DMs + @-mentions, last 12 hours, channels = [list]
- calendar: today + tomorrow morning, with attendee context
- polymarket: open positions, current P&L vs entry
- news: [3 RSS sources], last 12 hours, only items mentioning [keywords]

OUTPUT (markdown, one page max):
## Needs response before 10am
[bullets: sender + 1-line summary + suggested action]

## Can wait
[bullets, max 5]

## Noise — flagged so i know you saw it
[1 line per item, no detail]

TERMINATION: stop when all five sources are processed and the page
is under 400 words. Do not summarize the summary.
```

---

**模板2：竞品扫描 /competitor-scan**
抓取最多8个竞品的产品页、定价、博客、招聘、融资，与自家定位文档交叉比对。

```markdown
/competitor-scan

INPUTS:
- competitors: [comma-separated list, up to 8]
- my_positioning_doc: [Drive path]
- focus: [pricing | product | hiring | go-to-market | full]

WORKFLOW:
1. For each competitor, fetch: homepage, pricing page, last 5 blog
   posts, last 30 days of X posts from main handle, open job listings.
2. Compare positioning against my_positioning_doc.
3. Flag any competitor that has moved closer to my positioning
   in the last 60 days.
4. Flag any new hire or funding round (last 90 days).

OUTPUT (markdown):
## TL;DR (3 lines max)
## Per competitor
- pricing delta vs me
- positioning delta vs me (last 60 days)
- hiring signal
- funding signal
- one paragraph: what they would do that i would not see coming

TERMINATION: stop when each competitor section has all 5 fields filled.
Do not write an industry overview. Do not predict the future.
```

---

**模板3：邮件分类 + 起草回复 /triage**
每日三次自动分拣收件箱，匹配语气起草回复，以草稿形式存入 Gmail 待统一审阅。

```markdown
/triage

INPUTS:
- since: [last_run | morning | 24h]

WORKFLOW:
1. Pull all gmail messages since [since], excluding promotions and updates.
2. For each thread: needs_reply | informational | already_handled.
3. For needs_reply, draft a response using:
   - my last 5 sent emails to the same person (matches voice)
   - if no prior thread, default to 3-sentence acknowledgment + question
4. Save drafts in gmail with label "claude_draft", do not send.
5. Group informational by sender, summarize in 1 line each.

OUTPUT:
## Needs my eyes (drafts ready)
[list: sender + subject + 1-line on what the draft says]

## Informational (read-only)
[grouped by sender]

## Already handled / loops
[1 line each]

TERMINATION: stop when all unread threads are categorized.
Do not generate replies for threads you've already drafted.
```

---

**模板4：会议前摘要包 /meeting-prep**
会议前2小时自动运行，抓取参会者历史、共享文件、近期动态，生成1页简报+3个开放问题。

```markdown
/meeting-prep

INPUTS:
- meeting: [calendar event ID, or "next external meeting"]

WORKFLOW:
1. Identify external attendees (anyone outside my org).
2. For each: last 10 emails (any direction), shared docs,
   last linkedin update, last 30 days of X posts.
3. Find the last conversation we had and what was unresolved.
4. Generate 3 open questions based on what was unresolved.

OUTPUT (one page):
## Who's in the room
[per attendee: 2-line context]
## Last touchpoint
[when, what we agreed, what's unfinished]
## 3 open questions
## What's changed since
[only things that affect the meeting]

TERMINATION: stop at one page. If you can't fit something on one page,
drop it.
```

---

**模板5：周报 /weekly-status**
每周五4点自动抓取 Linear/Notion/Slack/日历，按受众（内部/客户/董事会）生成不同格式的状态报告。

```markdown
/weekly-status

INPUTS:
- week: [current | last]
- audience: [internal | client | board]

WORKFLOW:
1. Pull all closed Linear issues this week, with PR titles and reviewers.
2. Pull Notion docs created or significantly edited this week.
3. Read Slack #status, #shipping, #design channels for the week.
4. Calendar: count external meetings, internal meetings, deep work blocks.

OUTPUT (markdown, audience-specific length):
- internal: 1 page
- client: 1.5 pages, soften failures, surface risks
- board: 2 pages, with metrics-vs-targets table

Sections:
## Shipped
## In flight
## Blocked / risks
## Metrics (board version only)
## Next week's focus

TERMINATION: stop at length for chosen audience. Do not invent metrics
not present in source data.
```

---

**模板6：文档审阅 + 问答 /doc-review**
对10页以上的 PDF 或文档生成结构化问答、核心论点、矛盾点和阅读建议。

```markdown
/doc-review

INPUTS:
- document: [file path or url]
- my_context: [Drive path with related notes, optional]
- depth: [skim | review | deep]

WORKFLOW:
1. Read the document in full.
2. Identify: thesis, key claims, supporting evidence, gaps in evidence.
3. If my_context provided: flag contradictions or alignments with my prior writing.
4. Generate Q&A: 5 questions a thoughtful reader would ask, with the answer
   from the document (or "not addressed").

OUTPUT:
## Thesis (1 line)
## Key claims (3-5 bullets, each with evidence quality)
## Inconsistencies / gaps
## Contradictions or alignments with my prior work (if context provided)
## Q&A (5 questions)
## Verdict: read in full | skim | skip

TERMINATION: stop at this structure. Do not summarize the document
linearly section by section.
```

---

**模板7：持仓审计 /poly-audit**
每日三次检查 Polymarket 持仓，结合近12小时新闻标记需要关注的仓位（可改造为任何需要定期监控的数据审计）。

```markdown
/poly-audit

INPUTS:
- wallet: [my polymarket wallet address]
- alert_threshold: [10% | 20% | 30%]

WORKFLOW:
1. Pull all open positions, entry price, current price, P&L.
2. For each: search news (last 12 hours) for related events.
3. Cross-reference with [my watchlist doc].
4. Flag any position where:
   - P&L is more than [threshold] from entry
   - News mentions the underlying event
   - Volume on the market increased >50% in the last 6 hours

OUTPUT:
## Needs my eyes
[per flagged position: market + reason + suggested action]
## Holding pattern
[1 line each]
## Closed since last audit
[1 line each, with realized P&L]

TERMINATION: stop after all positions are checked. Do not recommend
new positions. Do not predict market direction. Audit only.
```

---

**模板8：深度研究 /research-deep**
用5个并行子 Agent 分别抓取学术/新闻/X话语/官方文件/反对声音，协调者综合输出带引用的研究简报。

```markdown
/research-deep

INPUTS:
- topic: [free text]
- depth: [survey | brief | full]
- timebox: [30min | 60min | 120min]

WORKFLOW (uses sub-agents):
Spawn 5 parallel sub-agents:
- AGENT_A: academic sources (Google Scholar, arXiv, key journals)
- AGENT_B: news + trade press (last 12 months)
- AGENT_C: X discourse (top voices on topic, last 90 days)
- AGENT_D: primary documents (gov filings, company docs, public datasets)
- AGENT_E: contrarian view — the 3 strongest critics, what they argue

Each sub-agent returns: 5 sources + 3-line summary of what they found.

COORDINATOR then synthesizes:
## TL;DR (3 lines, written as if to a smart skeptic)
## Mainstream consensus
## Contrarian case
## Open questions where evidence is genuinely thin
## My recommendation given the evidence

TERMINATION: stop at timebox. One pass per sub-agent. If AGENT_A
returns 5 sources at minute 4 of a 60-minute timebox, do not spawn
a second pass.
```

---

**模板9：内容二次分发 /repurpose**
把一篇长文一次性适配为 X 帖子串、LinkedIn 帖、Newsletter 节选、Slack 分享、外联邮件，每个平台独立 hook。

```markdown
/repurpose

INPUTS:
- source: [file path or url of long-form piece]
- channels: [x | linkedin | newsletter | slack | email | all]
- tone: [my default | formal | casual]

WORKFLOW:
For each channel selected:
1. Identify the single strongest claim in the source.
2. Adapt to channel-specific constraints:
   - x: thread, 6-10 posts, each <280 chars, hook in post 1
   - linkedin: 1 post, 300 words, professional voice
   - newsletter: 2-paragraph excerpt + 1-line CTA back to full piece
   - slack: 4-line summary + link, written for #share channel
   - email: 60-word blurb + subject line + CTA

CONSTRAINTS:
- never use the same opening line across channels
- never quote more than 15 consecutive words from the source
- match my voice from [my last 10 posts on the channel, if available]

OUTPUT: one block per channel, ready to copy-paste.

TERMINATION: stop after all selected channels are filled. Do not
generate additional "bonus" versions.
```
