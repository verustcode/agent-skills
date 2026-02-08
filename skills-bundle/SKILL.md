---
name: skills-bundle
description: Use when user describes a complex project goal, wants to set up multiple skills at once, asks "what skills do I need for X", or needs to manage installed skill bundles. Triggers on multi-skill setup, skill combination, project bootstrapping, or batch skill management.
---

# Skills Bundle

## Overview

Decompose a user's project goal into multi-dimensional skill requirements, search the skill ecosystem, then **intelligently filter results against user intent** before recommending. Uses a Search → Inspect → Analyze pipeline to avoid irrelevant recommendations.

**Core problem:** When a user says "I want to build X", naive keyword search returns skills that match the keyword but not the intent. For example, searching "ui" for a game project returns web UI skills instead of game UI skills. This skill solves that by adding a context-aware filtering step.

## When to Use

- User describes a complex project goal (e.g., "build a React mobile app")
- User asks "what skills do I need for X"
- User wants to batch install/manage/clean up skills
- User wants to bootstrap a new project with the right skill set

**Not for:** Finding a single specific skill — just use `npx skills find` directly.

## Core Pipeline

```
User Goal → Decompose → Search → Inspect → Analyze & Filter → Present → Install
```

## Workflow

### Step 1: Decompose Requirements

Understand the user's project goal and decompose into two layers:

**Layer 1 — Essential:** Skill domains the project cannot do without — derived from the user's explicit requirements and the hard dependencies of the project type (e.g., a game project inherently needs a game engine, asset management, etc.).
**Layer 2 — Extended:** Not strictly required, but significantly improves project quality (e.g., code style, documentation generation). Only expand dimensions relevant to the project type — a CLI tool doesn't need "UI design".

Present the decomposition to the user for confirmation. **Always respond in the user's language** — if the user writes in Chinese, all output (decomposition, tables, explanations) should be in Chinese; if in English, respond in English.

For each layer, generate **context-aware search keywords**. Keywords should be specific to the project domain, not generic:

| Bad (generic) | Good (context-aware) |
|--------------|---------------------|
| `ui` | `game-ui`, `game-hud` |
| `assets` | `game-assets`, `sprite`, `pixel-art` |
| `testing` | `game-testing`, `playwright` |
| `performance` | `webgl-performance`, `game-optimization` |

**Rule: Always prefix generic keywords with the project domain when a generic term is ambiguous.** If the domain-prefixed search returns no results, fall back to the generic keyword but mark it for careful filtering in Step 3.

### Step 2: Multi-round Search

**CRITICAL: You MUST run these commands yourself via your Shell/terminal tool.** Do NOT guess skills from memory. Do NOT skip this step.

Run `npx skills find <keyword>` for each keyword from Step 1. Run searches in parallel when possible (chain with `&&`).

Collect all `owner/repo@skill` entries and their URLs from the output. Deduplicate by skill name. **Per keyword, only take the top 5 results** (or all if fewer than 5) — `npx skills find` already returns results in relevance order.

### Step 3: Inspect & Analyze (Sub-agent Filtering)

This is the critical step that prevents irrelevant recommendations.

**For each unique repository found in Step 2**, launch a sub-agent (Task tool) to:

1. Fetch the skill's detail page at `https://skills.sh/<owner>/<repo>/<skill>` using WebFetch
2. Read the skill's description, content, and **Weekly Installs** count (shown at the bottom of the page)
3. **Score relevance** against the user's original goal on a 1-5 scale, considering both content fit and community adoption:
   - 5: Directly addresses a core need, well-maintained (high installs)
   - 4: Strongly relevant, covers an important aspect
   - 3: Somewhat relevant, could be useful
   - 2: Tangentially related, different domain
   - 1: Irrelevant despite keyword match
   - **Adoption bonus:** When two skills cover similar functionality, prefer the one with significantly higher Weekly Installs — it signals better quality, maintenance, and community validation.
   - **Low adoption caution:** A skill with very few installs (< 100/week) should be noted but not automatically excluded — it may be new or niche.
4. Return: skill name, source (`owner/repo@skill`), one-line description, weekly installs, relevance score, and a brief reason

**Sub-agent prompt template:**

```
Analyze whether this skill is relevant for the user's project goal.

User's goal: "<user's original request>"
Project type: "<detected project type, e.g. 'browser-based 2D game'>"

Skill to evaluate: <owner/repo@skill>
Skill URL: https://skills.sh/<owner>/<repo>/<skill>

Fetch the skill URL and read its full description and Weekly Installs count. Then score its relevance (1-5) against the user's goal, factoring in community adoption. Return a JSON object:
{
  "name": "<skill-name>",
  "source": "<owner/repo@skill>",
  "description": "<one-line description from the skill page>",
  "weekly_installs": <number>,
  "score": <1-5>,
  "reason": "<why this score, mention installs if it influenced the decision>"
}
```

**Parallelism:** Launch sub-agents concurrently to inspect skills in parallel.

**Filtering threshold:** Only keep skills with score >= 3. Present score 5 and 4 as "Recommended", score 3 as "Optional".

### Step 4: Present Results

Group filtered results by layer (Essential / Extended) and by confidence:

```
## Recommended Skills for: <user's goal>

### Essential
| # | Skill | Description | Installs/wk | Relevance | Reason |
|---|-------|------------|-------------|-----------|--------|
| 1 | owner/repo@skill | ... | 1.8K | ★★★★★ | ... |
| 2 | owner/repo@skill | ... | 500 | ★★★★☆ | ... |

### Extended
| # | Skill | Description | Installs/wk | Relevance | Reason |
|---|-------|------------|-------------|-----------|--------|
| 3 | owner/repo@skill | ... | 2.1K | ★★★★☆ | ... |

### Optional
| # | Skill | Description | Installs/wk | Relevance | Reason |
|---|-------|------------|-------------|-----------|--------|
| 4 | owner/repo@skill | ... | 80 | ★★★☆☆ | ... |
```

**Filtered out (not shown to user):** Skills with score < 3 are silently dropped. If the user asks why something was excluded, explain the relevance mismatch.

Ask user to select by number (e.g., "1,2,3"), "all essential", or "all".

### Step 5: Generate Install Commands

Generate copy-paste-ready commands based on user selection:

```bash
# Essential
npx skills add <owner/repo@skill> -g -y
npx skills add <owner/repo@skill> -g -y

# Extended
npx skills add <owner/repo@skill> -g -y
```

### Step 6: Generate Bundle Config

After installation, generate `.skills-bundle.json` in the project root:

```json
{
  "name": "project-name",
  "created": "2026-02-08",
  "description": "User's project goal",
  "skills": {
    "essential": [
      { "name": "skill-name", "source": "owner/repo@skill", "reason": "Why this skill", "score": 5 }
    ],
    "extended": [
      { "name": "skill-name", "source": "owner/repo@skill", "reason": "Why this skill", "score": 4 }
    ]
  }
}
```

## Bundle Management

### View Installed Skills

```bash
npx skills ls -g          # List global skills
npx skills ls             # List project-level skills
npx skills ls -a cursor   # Filter by agent
```

### Install from Config

Read `.skills-bundle.json`, extract all `source` fields, and generate `npx skills add <source> -g -y` commands.

### Uninstall Bundle

Read `.skills-bundle.json`, extract all `name` fields, and generate:

```bash
npx skills remove <name1> <name2> ... -g -y
```

### Check for Updates

```bash
npx skills check    # Check for available updates
npx skills update   # Update all installed skills
```

## Quick Reference

| Action | Command |
|--------|---------|
| Search skills | `npx skills find <keyword>` |
| Install skill | `npx skills add <owner/repo@skill> -g -y` |
| List installed | `npx skills ls -g` |
| Remove skills | `npx skills remove <name1> <name2> -g -y` |
| Check updates | `npx skills check` |
| Update all | `npx skills update` |

## Red Flags — STOP If You Catch Yourself Doing These

- "I cannot run `npx skills find`" — **Yes you can.** Use your Shell/terminal tool.
- "Based on known/installed skills..." — **Wrong.** You must search the ecosystem.
- Showing skill recommendations without running `npx skills find` — **Start Step 2 over.**
- Recommending skills without inspecting their descriptions — **Start Step 3 over.** You must verify each skill's actual content against the user's goal.
- Recommending a skill just because its name matches a keyword — **Wrong.** A skill named "ui-components" is not useful for a game project unless its description confirms game UI support.

## Common Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| Guess skills from memory | **Run** `npx skills find` for every keyword |
| Search one keyword and stop | Use two-layer model for multi-round search |
| Recommend without inspecting skill content | **Fetch and read** each skill's description via sub-agent |
| Use generic keywords that return wrong-domain results | Use domain-prefixed keywords; fall back to generic only with careful filtering |
| Mix essential and extended without distinction | Separate into Essential / Extended layers so user can choose |
| Install without keeping records | Generate `.skills-bundle.json` |
| Skip relevance filtering | Always run Step 3 — it's the most important step |
