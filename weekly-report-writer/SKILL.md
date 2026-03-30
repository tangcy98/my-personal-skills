---
name: weekly-report-writer
description: Transform casual progress notes into professionally formatted weekly reports by learning from your past reports. Use when you want to convert informal notes about this week's work into a polished report that matches your team's existing format and style. Provide your casual progress notes and a past weekly report as format reference. The skill analyzes structure, language conventions, and formatting of past reports, then applies the same style to polish your new content.
---

# Weekly Report Writer

## Overview

This skill has two modes of operation:

### Mode 1: Real-time Progress Tracking (Throughout the Week)
- **You provide**: Brief progress updates whenever something noteworthy happens
- **Skill does**: Accumulates and organizes updates in persistent memory (`memory/weekly-progress-YYYY-MM-DD.md`)
- **Benefit**: No need to collect everything at the end of the week

### Mode 2: Automated Report Generation (On Report Day)
- **You provide**: A past report (format reference) + command to generate report
- **Skill does**: 
  1. Reads accumulated progress from memory
  2. Analyzes past report's structure and style
  3. Aggregates, deduplicates, and organizes progress notes
  4. Generates final polished report matching your format
- **Output**: Ready-to-submit Redoc document

**The key difference:** Instead of collecting everything and formatting it all at once, you capture progress throughout the week, and the skill synthesizes it all on report day.

## Workflow

### Phase 1: Progress Tracking (Monday-Friday)

**Throughout the week**, whenever you have noteworthy progress:

```
"今天完成了 xxx 设计评审，算法对齐完毕"

or

"修了那个性能问题，DAP 提升了 0.3%"

or

just

"被动消费模型对齐了，可以周一开实验"
```

**What happens:**
- Skill automatically appends to `memory/weekly-progress-YYYY-MM-DD.md`
- Each entry is timestamped and categorized
- No formatting needed—just raw progress updates

### Phase 2: Report Generation (Report Day)

**On your report day**, provide:
1. Past report (URL or content) as format reference
2. Command: "Generate weekly report" or "生成周报"

**What happens:**
- Skill reads `memory/weekly-progress-YYYY-MM-DD.md` (accumulated progress)
- Analyzes past report structure and style
- Aggregates all progress notes
- Removes duplicates
- Organizes into one-level and two-level structure
- Generates final polished Redoc report

### Processing Logic

#### During the Week: Progress Capture

When you share progress updates:

1. **Parse input** — Extract:
   - Main topic/theme
   - Key facts and metrics
   - Status (completed, in-progress, blocked)
   - Related context or dates

2. **Append to memory** — Store in `memory/weekly-progress-YYYY-MM-DD.md`:
   ```
   - [Timestamp/Date] [Topic]: [Raw progress note with metrics/status]
   ```
   Example:
   ```
   - Mon 3.17: 模型训练 - 完成了模型准召对齐，准确 90%、召回 85%
   - Wed 3.19: 扶持策略 - 新标签实验启动，DAP +0.08%
   ```

3. **Accumulate** — Keep appending throughout the week (no deduplication yet)

#### On Report Day: Report Generation

When you request report generation:

1. **Analyze past report** — Extract:
   - **One-level structure** (main sections): What are the primary topics?
   - **Two-level structure** (subsections): How are items nested?
   - **Language tone, phrasing patterns, formatting**

2. **Read memory file** — Extract all progress entries from `memory/weekly-progress-YYYY-MM-DD.md`

3. **Aggregate & deduplicate** — Combine related updates, remove duplicates

4. **Map content to structure** — Critical step:
   - **Semantic understanding**: Understand that "model deployed" belongs under "Model Training"
   - **Preserve hierarchy**: Use past report's structure as template
   - **Only include mentioned sections**: If user didn't mention a one-level section all week, don't include it
   - **⚠️ CRITICAL: Do NOT auto-fill**: Never copy past report content for sections not mentioned
   - **Check for new categories**: If something doesn't fit, ask clarifying questions

5. **Transform** — Rewrite aggregated progress in past report's style:
   - Same one-level and two-level structure
   - Same tone and language patterns
   - Same conciseness (1-2 sentences per two-level item)
   - Same phrasing style (metrics-driven, use of connectors like "、")
   - Same formatting

6. **Output** — Generate and upload to Redoc

### Output: Your Formatted Report

You get a report ready to submit that:
- ✓ Matches your team's actual style
- ✓ Contains this week's real accomplishments
- ✓ Maintains your voice/persona
- ✓ Is consistent week-to-week

## How to Use

### Mode 1: Throughout the Week (Progress Tracking)

**Anytime you have progress**, just tell me:

```
"Completed model training, accuracy 90%, recall 85%"

or

"New label experiment launched, DAP +0.08%"

or even just

"推全了外流混排 midnv 队列独立"
```

**What I do:**
- Append to `memory/weekly-progress-2026-03-16.md` (auto-created)
- Keep it raw, unformatted
- Accumulate all week

### Mode 2: On Report Day (Report Generation)

**When ready to generate report**, provide:

```
"Generate weekly report based on this past report:
[paste past report URL or content]"
```

**What I do:**
1. Read all progress from `memory/weekly-progress-2026-03-16.md`
2. Analyze past report structure
3. Aggregate and organize your progress
4. Generate final polished Redoc report
5. Send you the link

### Real Example Workflow

**During the week:**
```
Mon: "模型部署完毕，日均刷 500w 作者"
Wed: "新标签流量策略实验筹备，最快本周内启动"
Thu: "活人UGC贡献度重新迭代，周内出结论"
Fri: "被动促产模型完成，本周内开实验"
```

**On report day:**
```
"生成周报，参考这个过往报告：https://docs.xiaohongshu.com/doc/xxx"
```

**Output:**
```
[Fully formatted, aggregated report ready to submit]
```

No manual aggregation needed—the skill does it all!

## Advanced Usage

### Custom Report Day Command

If you want to specify report parameters:

```
"生成周报，参考这个过往报告：[URL]
要求：more concise / more detailed / add more metrics"
```

The skill will generate according to your preferences while still learning from the past report structure.

### For Different Report Types

If you submit different report formats to different audiences:
- Weekly to manager (formal, detailed)
- Team standup (casual, brief)
- Executive summary (high-level metrics)

Just provide the corresponding past report when requesting generation, and the skill will adapt.

## Critical Processing Checklist

### During the Week (Progress Capture)
- [ ] Identify the main topic/theme from user's update
- [ ] Extract key metrics or status indicators
- [ ] Append with timestamp to `memory/weekly-progress-YYYY-MM-DD.md`
- [ ] Keep original format (no transformation at this stage)
- [ ] Do NOT deduplicate or aggregate yet

### On Report Day (Report Generation)

Before transforming your notes into final report, the skill must:

1. **Structure Analysis**
   - [ ] Identify all one-level headings (main topics/sections)
   - [ ] For each one-level section, identify all two-level items (subsections)
   - [ ] Determine if any one-level sections are optional or context-dependent

2. **Semantic Mapping**
   - [ ] Understand that "deployment" typically maps to "Model Training", not as standalone topic
   - [ ] Understand that "experiment/test initiative" maps to "Support Strategy", not as standalone topic
   - [ ] For each fact in your input, determine which one-level section it belongs to
   - [ ] For each fact, determine which two-level item it fills

3. **New Content Handling**
   - [ ] If you mention something that doesn't fit the past report's structure, ASK for clarification
   - [ ] Ask: "Should this be a new two-level item under [one-level section]?" or "Should this be a new one-level section?"
   - [ ] Never assume; always clarify structural placement

4. **Output Validation**
   - [ ] Only include one-level sections that the user explicitly mentioned or clearly implied
   - [ ] Do NOT include one-level sections from the past report that the user did not mention
   - [ ] Do NOT copy/paste past report content for sections the user didn't discuss
   - [ ] Two-level items are only populated if you provided corresponding input
   - [ ] Language style matches past report (conciseness, phrasing patterns, tone)
   - [ ] No new sections added without explicit approval
   - [ ] Metrics and data are included when available
   - [ ] Always verify: "Did the user mention this section?" before including it

## Tips for Best Results

### What Makes a Good Past Report Reference

- **Complete** — Full report, not just excerpts
- **Well-written** — You're happy with how it reads
- **Representative** — Shows the style you want to match (one-level and two-level structure clearly visible)
- **Recent** — From same year/timeframe (language conventions evolve)

### For Consistent Results

- Use the same past report as reference for multiple weeks
- Or gradually introduce new past reports as your style evolves
- The skill will maintain consistency over time

### When You Don't Have a Past Report

No worries! You can:
- Ask a colleague to share one (with their permission)
- Provide a "style guide" example (like a company template)
- Use any writing sample that matches your target tone
- Even describe: "I want it to be more casual/formal/technical"

### For Different Report Types

If you submit different report formats:
- Weekly to manager (detailed, formal)
- Team standup (casual, bullet points)
- Executive summary (high-level, metrics-focused)

Just provide the matching past report for each context, and the skill will adapt.
