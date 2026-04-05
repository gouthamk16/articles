---
layout: post
title: "Building a Personal Second Brain with Claude Code and Obsidian"
date: 2026-04-05 10:00:00 +0000
categories: tools ai knowledge-management
---

There is a specific kind of frustration that comes from knowing you have read something relevant before but being unable to find it, reconstruct it, or even confirm it existed. You saved the PDF. You bookmarked the article. You clipped the thread. But when you actually need it, all you have is a folder of disconnected files and the vague memory that you once understood this.

This post is about the system I built to fix that, using Claude Code and Obsidian. It is not a productivity framework. It is an engineering project, and it has tradeoffs worth explaining.

---

## The Problem: Information In, Knowledge Never Out

The failure mode is consistent: capture is easy, synthesis is hard, and hard things do not happen automatically.

Over time you accumulate a `Downloads/` folder full of PDFs, a browser full of bookmarks, a notes app with half-written thoughts, and a Pocket queue you have not touched in eight months. The information is technically "saved." But it is not knowledge. It has never been connected to anything else you know, never been summarized at a level you can actually use, and never been indexed in a way that lets you retrieve it by concept rather than by filename.

Manual note-taking is the canonical solution and it does not work, not sustainably. The friction of switching from reading to writing, maintaining structure, enforcing consistency, and keeping links alive is too high. Most people maintain active notes for a few weeks after reading a book about it, then stop.

The root issue is that synthesis is cognitive work, and you are already doing cognitive work when you capture something. Asking yourself to do both at once, every time, is a losing bet.

---

## Inspiration: Karpathy's Approach

Andrej Karpathy described the idea publicly: use an LLM to maintain a personal wiki. You drop raw material into a folder. The LLM reads everything, extracts concepts, writes articles, and keeps them linked. You never touch the wiki directly. You just read it.

The key insight is the division of labor. The human is good at knowing what is worth capturing, bad at maintaining structure under time pressure. The LLM is bad at knowing what is worth capturing (it has no context on your life), good at synthesis and consistent formatting. So you split the work accordingly: the human owns `raw/`, the LLM owns `wiki/`.

This removes the friction at the point where it kills most systems: you no longer need to synthesize at capture time. You drop a file and move on. The synthesis happens later, automatically, at a time when no cognitive resources are at stake.

---

## Architecture Overview

The vault has two zones.

`raw/` is append-only. The user dumps files here: PDFs, images, `.txt` files containing URLs, Web Clipper `.md` files, email dumps. Nothing is processed here. There is no preprocessing step, no metadata required, no folder structure to maintain.

`wiki/` is Claude-maintained. It contains `concepts/` (atomic articles about individual ideas), `sources/` (one note per file in `raw/`), `topics/` (Map of Content files, one per domain), `slides/` (Marp-formatted presentations), and `index.md` (the master entry point).

`manifest.json` sits at the vault root and tracks the state of the last compilation pass. It stores a `compiled_at` ISO 8601 UTC timestamp and a `files` map of `sha256` hashes keyed by path. This is the delta tracker: on any given night, Claude reads the manifest, compares it against the current contents of `raw/`, and only processes files that are new or changed.

Obsidian is the read interface. It renders the `wiki/` markdown, shows the graph of links between concepts, and lets you run Dataview queries over frontmatter. You open it to read. You never open it to write.

The whole thing runs on a nightly scheduled task that fires at 2:15am, processes the delta, and leaves the wiki updated by morning.

---

## The raw/ Folder: Frictionless Capture

The design principle for `raw/` is that capture must have zero friction or it will not happen consistently.

Accepted formats:
- `.txt` files containing a single URL, which Claude fetches during compilation
- `.txt` files with multiple URLs or pasted email threads, which Claude parses and fetches individually
- `.md` files from the Obsidian Web Clipper browser extension, which carry the full article HTML already converted to markdown
- `.pdf` files, which Claude reads directly using vision-based PDF reading
- Images (`.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`), which Claude processes using vision

Google Drive syncs the `raw/` folder across phone and laptop. On mobile, the workflow is: screenshot something interesting, save to Drive. On desktop: drag and drop. The Web Clipper extension handles paywalled or JavaScript-heavy articles, where a raw URL would just return a login wall.

Nothing runs at capture time. No script validates the file, renames it, or tries to extract metadata. The file lands and sits there until the next compilation pass.

---

## AI-First Compilation: Why We Dropped Deterministic Scraping

The original pipeline was two steps: a Python scraper using `requests` and `html2text` to extract article text from URLs, plus `PyMuPDF` to extract text from PDFs, followed by a Claude pass to summarize the extracted text.

This was wrong for several reasons.

First, it added a failure surface. Scraping with `requests` fails on JavaScript-rendered pages, redirects, paywalled content, and a dozen other cases. Every failure mode had to be handled explicitly.

Second, it was the wrong abstraction. Format conversion (HTML to text, PDF to text) is not the same as understanding. The scraper produced a flat text dump. Claude then had to work with that dump rather than with the original source. For PDFs with figures, charts, or complex layouts, the text extraction lost most of what made the document interesting.

Third, it was two steps when one suffices. Claude can fetch URLs directly with the `WebFetch` tool, read PDFs natively, and process images with vision. There is no intermediate representation needed.

The current pipeline is a single step. For each file in the delta:

- `.txt` with a single URL: `WebFetch` the URL, read the full article
- `.txt` with multiple URLs or email dumps: read the file, extract all URLs, `WebFetch` each one
- `.md` Web Clipper clips: read directly, content is already there
- `.pdf`: read using the Read tool (Claude handles PDF natively)
- Images: view directly using vision

Multi-URL files work automatically because Claude can parse free-form text, find URLs, and handle each one. There is no regex required, no format to enforce.

Paywalled content gets a stub note in `wiki/sources/` with `status: stub` and a comment explaining that the Web Clipper should be used to capture the content manually. The manifest entry is still written, so the file is not retried every night.

---

## The manifest.json Delta System

The manifest exists to make nightly compilation cheap. Without it, a full recompile would touch every file in `raw/` every night, which grows linearly as the vault grows and wastes time re-summarizing things that have not changed.

The format is minimal:

```json
{
  "compiled_at": "2026-04-04T02:00:00+00:00",
  "files": {
    "raw/article.md": "sha256:a3f1...",
    "raw/paper.pdf": "sha256:9b2c..."
  }
}
```

On each compilation pass, Claude reads the manifest, lists all files in `raw/`, and computes the delta: files not present in `manifest.files` (new captures) or files whose current hash differs from the stored hash (modified files). Only those files get processed.

After compilation, the manifest is rewritten with the updated `compiled_at` timestamp and the current hashes for all files in `raw/`. If a file was deleted from `raw/`, it drops out of the manifest naturally on the next write.

The `compiled_at` timestamp is also what the staleness check uses.

---

## The Staleness Hook

`_meta/staleness_check.py` is a small script that compares file modification times in `raw/` against the `compiled_at` timestamp in `manifest.json`. If any file in `raw/` is newer than the last compilation, it prints a warning listing the uncompiled files.

```python
stale = [
    f.relative_to(VAULT)
    for f in RAW.rglob("*")
    if f.is_file() and datetime.fromtimestamp(f.stat().st_mtime, tz=timezone.utc) > compiled_at
]
```

This runs before every Claude Code session via a `UserPromptSubmit` hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python C:/Users/Goutham/Desktop/Goutham/brain/_meta/staleness_check.py",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

The hook fires on every prompt. If the wiki is stale, the warning surfaces immediately before you start asking questions. This prevents the failure mode where you query the brain about something you captured last night and get an answer that does not account for it, without knowing that the wiki is behind.

The script exits with code 1 if stale, 0 if fresh. The hook output appears inline in the session.

---

## Obsidian as the Frontend

Obsidian was chosen for three specific features: bidirectional links, the graph view, and Dataview.

Bidirectional links mean that when a concept article links to another concept, both notes automatically show the connection. You can navigate the knowledge graph without needing to maintain a central index manually.

The graph view renders the full link structure visually. Highly connected concepts are obvious at a glance. Orphan notes that have been created but never linked stand out immediately.

Dataview lets you query frontmatter across the vault using a SQL-like syntax. The `wiki/index.md` file contains Dataview blocks for things like "all notes with status: stub," "notes updated in the last 7 days," and "files in raw/ not yet compiled." These are live dashboards.

The vault structure:

```
raw/                    <- user writes here only
wiki/
  index.md              <- master entry point, Dataview dashboards
  concepts/             <- atomic concept articles
  sources/              <- one note per raw/ file
  topics/               <- MOC - <Domain>.md files
  slides/               <- Marp presentations
assets/                 <- images referenced in wiki articles
_meta/                  <- skills, scripts, config
manifest.json
```

Plugins installed: Dataview, Templater, Omnisearch, Obsidian Git, and `obsidian-mcp-server`.

`obsidian-mcp-server` is the bridge between Claude Code and the vault. It exposes 31 tools that Claude can call to read notes, write notes, search by text, get backlinks, list vault structure, update frontmatter, append to sections, and more. This is what lets Claude write to the vault without bypassing Obsidian's format expectations or breaking its internal link index.

The MCP server is configured in `.claude/settings.json` for project-level use and in `~/.claude/mcp.json` for global use (so `/brain` works from any project, not just from inside the vault directory).

---

## Claude Code Skills: The Operating Protocol

Five skill files live in `_meta/skills/`. Each is a protocol document that Claude reads before performing a specific operation. They define exactly how each operation should be done: which tools to call, in what order, with what constraints.

**brain-navigate.md** defines vault traversal discipline. Always start at `wiki/index.md`. Identify the relevant domain. Read the domain MOC. Follow links to concept articles. Use `get-backlinks` to find related articles not listed in the MOC. Never read notes one at a time in a loop; use `read-multiple-notes` for batches. Always scope `search-vault` to a specific directory; never search vault-wide unless the query genuinely spans domains.

**brain-compile.md** defines the compilation pipeline. Read `manifest.json` for the delta. List `raw/`. For each file in the delta, extract content using the appropriate tool. Produce a source note in `wiki/sources/`. For each concept extracted, check if a concept article exists and create or update accordingly. Update `wiki/index.md`. Write `manifest.json`.

**brain-query.md** defines how to answer questions from the brain. Check staleness first. Read `wiki/index.md`. Navigate to relevant MOCs. Search `wiki/concepts/` for specific terms. Expand via backlinks. Read full content of the relevant subset. Synthesize an answer with citations from frontmatter `source_url` or `source_file` fields. Never cite from Claude's general knowledge; only from sources that are in the vault.

**brain-lint.md** defines the health check pass: find broken links, check tag compliance, identify unknown tags, expand stubs that have source material, and find orphan notes that have no backlinks.

**brain-write.md** defines the frontmatter contract and section structure for every wiki article. Every note must have `title`, `type`, `tags`, `status`, `created`, `updated`, and optionally `source_url`, `source_file`, and `related`. Tags must come from `_meta/tag-registry.md`. Link concepts on first mention only. Never link generic terms like "model" or "system."

**_meta/CLAUDE.md** is the master operating rules file. Claude Code loads it automatically when the vault is the working directory. It defines the two-zone structure, the vault layout, which skill to load for each operation, the frontmatter contract, and the manifest format. It is the authoritative reference for all operating rules.

---

## Slash Commands: Two Levels

Each skill has a corresponding slash command. At the project level, `.claude/commands/brain/` contains five files: `compile.md`, `query.md`, `navigate.md`, `lint.md`, and `write.md`. Each is a one-liner:

```
Read `_meta/CLAUDE.md` for operating rules, then read and follow `_meta/skills/brain-compile.md` exactly.
```

This gives `/brain:compile`, `/brain:query`, `/brain:navigate`, `/brain:lint`, and `/brain:write` when working inside the vault directory.

At the global level, `~/.claude/commands/brain.md` gives `/brain` from any project. This delegates to the same skill files via an absolute path. The MCP server is configured in `~/.claude/mcp.json` so it is available in every Claude Code session, not just in the vault project.

The practical consequence: if you are working on a coding project and want to pull in something from your knowledge base, you type `/brain` and ask your question without switching directories or loading a different project. The command has identical behavior in both contexts because both resolve to the same skill files and the same MCP server.

The single source of truth principle matters here. The skill files define the behavior. The commands are just entry points. If the compilation protocol changes, you update `brain-compile.md` once, and both the project-level and global commands immediately reflect the change.

---

## The Nightly Scheduled Task

The compilation pass runs automatically at 2:15am via a durable scheduled task created with the `mcp__scheduled-tasks__create_scheduled_task` tool. The task is disk-persisted, meaning it survives Claude Code restarts and does not depend on a running process to stay alive.

The scheduled agent receives the prompt: read `_meta/CLAUDE.md` for operating rules, then read and follow `_meta/skills/brain-compile.md` exactly. It executes the full protocol: reads the manifest, computes the delta, processes new files, writes source notes and concept articles, updates the index, and writes the updated manifest.

The 2:15am time is deliberate. Everything you captured during the day, whether at your desk or from your phone via Drive sync, gets compiled overnight. When you open Obsidian in the morning, the wiki reflects last night's captures. There is no manual step between capturing something and it appearing in your knowledge graph.

The laptop needs to be on for the task to run. This is an acknowledged tradeoff. A cloud-based solution would be more reliable but more complex and not free. For a personal knowledge base that grows incrementally, missing one night occasionally is acceptable.

---

## The Tag Registry and Frontmatter Contract

Every note in `wiki/` has the same frontmatter structure:

```yaml
---
title: Attention Mechanism
type: concept
tags: [domain/ml, type/concept, status/stable]
status: stable
created: 2026-03-12
updated: 2026-04-04
source_url: https://arxiv.org/abs/1706.03762
related: ["[[Transformer Architecture]]", "[[Self-Attention]]"]
---
```

Tags come from `_meta/tag-registry.md`, which defines three namespaces: `domain/`, `type/`, and `status/`. If a tag is not in the registry, it does not get used. If a new domain needs to be added, the registry is updated first, then the tag is used.

The status lifecycle is `stub -> draft -> stable -> outdated`. Stubs are created when a concept is first referenced but not yet populated. Draft notes have content but are incomplete or unreviewed. Stable notes are current and complete. Outdated notes have been flagged because a newer source contradicts or updates them.

This structure enables Dataview queries. You can ask "show me all stable machine learning concepts" or "show me all stubs in the finance domain" and get live results from frontmatter. The lint pass uses the same structure: `audit-tags` checks that required tag namespaces are present, `find-notes-by-tag` (with `matchExact: true`) finds notes with specific values.

Without controlled tags, the graph becomes unqueryable. Controlled tags are the difference between a collection of markdown files and a structured knowledge base.

---

## What It Enables

The end state is a system where the marginal cost of capturing something is near zero and the marginal cost of finding it later is also near zero.

Drop a PDF into `raw/`. Wake up the next morning. Open Obsidian. The paper has a source note in `wiki/sources/`, its key concepts have articles in `wiki/concepts/`, and those articles link to everything else in the vault that touches the same ideas. The index shows it was processed last night.

Query the brain from a coding project with `/brain: what do I know about attention mechanisms`. The agent checks staleness, navigates the ML domain MOC, reads the relevant concept articles, and synthesizes an answer with citations back to the source notes, which link back to the original raw files.

The wiki reflects exactly what you have captured. Not Claude's general knowledge, not a web search, not a hallucinated summary. What you chose to save, synthesized and linked.

Over time, the graph gets denser. A concept you first encountered in a 2024 paper gets connected to a blog post you saved in 2025 and a lecture slide you added last month. The connections are not ones you made manually; they emerged from compilation passes over time, each one adding links where the content warranted it.

This is the payoff: a knowledge base that compounds. The more you capture, the more connections exist, the more useful any individual query becomes. The system gets better the longer you use it, without requiring proportionally more maintenance effort.

The only requirement is that you keep dropping things into `raw/`.
