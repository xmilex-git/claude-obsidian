---
type: source
title: "Nate Herk LLM Wiki Transcript"
source_type: transcript
author: "Nate Herk"
date_published: 2026-04-07
url: "https://youtube.com/@nateherk"
confidence: high
key_claims:
  - "LLM wiki makes knowledge compound like interest — nothing is re-derived on every query"
  - "Hot cache (~500 words) enables cross-project context without crawling the full wiki"
  - "One article can generate 15-25 wiki pages with full cross-references"
  - "One user dropped token usage by 95% switching from inline context files to wiki"
  - "Obsidian is the IDE, Claude is the programmer, the wiki is the codebase"
  - "Index file is enough at small scale (~100 sources) — no RAG infrastructure needed"
created: 2026-04-07
updated: 2026-04-07
tags:
  - source
  - llm-wiki
  - obsidian
  - karpathy
status: mature
related:
  - "[[LLM Wiki Pattern]]"
  - "[[Hot Cache]]"
  - "[[Compounding Knowledge]]"
  - "[[Andrej Karpathy]]"
  - "[[index]]"
  - "[[sources/_index]]"
sources:
  - "[[.raw/nate-herk-llm-wiki-transcript.md]]"
---

# Nate Herk LLM Wiki Transcript

Raw source: [[.raw/nate-herk-llm-wiki-transcript.md]]

Nate Herk demonstrates the [[LLM Wiki Pattern]] in practice. He shows two live vaults: one for his YouTube transcript archive (36 videos) and one personal second brain. He breaks down Andrej Karpathy's original post and shows a 5-minute setup workflow.

---

## Key Takeaways

**The core insight**: normal AI chats are ephemeral. The wiki makes knowledge compound. Every source ingested, every question answered, every analysis filed — all of it stays and grows richer over time.

**The stack is simple**: Claude Code + Obsidian + a folder of markdown files. No vector databases, no embeddings, no infrastructure. Just files and Claude.

**The hot cache**: a ~500-word file (`wiki/hot.md`) that captures recent context. In an executive assistant setup, this prevented having to crawl dozens of wiki pages at the start of each session. See [[Hot Cache]].

**Cross-project referencing**: other Claude Code projects can read this vault by pointing at it in their CLAUDE.md. Nate's executive assistant reads from his herk-brain vault. Token usage dropped significantly compared to inline context files.

**At scale**: the index file alone is sufficient for hundreds of pages. Vector RAG only becomes necessary at millions of documents.

---

## Obsidian as IDE

Obsidian is just a markdown viewer with graph visualization. The graph view shows which pages are hubs (many connections) and which are orphans (none). Real-time — you can watch the wiki grow as Claude creates pages.

The key Obsidian features used:
- Graph view — visualize the knowledge structure
- Backlinks — follow connections between pages
- Dataview — query pages by frontmatter
- Web Clipper — send articles directly to `.raw/` from any browser

---

## Workflow Demonstrated

1. Install Obsidian, create a vault
2. Paste Karpathy's LLM wiki idea into Claude Code
3. Claude scaffolds the structure (raw/, wiki/, CLAUDE.md, index, log)
4. Drop a source into `.raw/` using Web Clipper
5. Tell Claude: "ingest this"
6. Claude reads, creates 15-25 wiki pages, cross-references everything
7. Query the wiki for insights

The ingest for one article (AI 2027) took 10 minutes and created 23 pages: 1 source, 6 people, 5 organizations, 1 AI systems page, multiple concepts, plus an analysis.

---

## Entities Mentioned

- [[Andrej Karpathy]] — originated the LLM wiki pattern
- Nate Herk — demonstrated the pattern in this video

---

## Connections

See [[LLM Wiki Pattern]] for the full architecture.
See [[Compounding Knowledge]] for the core insight on why this works.
See [[Hot Cache]] for the session context mechanism.
