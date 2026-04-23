# claude-obsidian — Claude + Obsidian Wiki Vault

This folder is both a Claude Code plugin and an Obsidian vault.

**Plugin name:** `claude-obsidian`
**Skills:** `/wiki`, `/wiki-ingest`, `/wiki-query`, `/wiki-lint`
**Vault path:** This directory (open in Obsidian directly)

## What This Vault Is For

Primary scope: **Documenting the CUBRID relational database source tree** at `/Users/song/DEV/cubrid/` (Mode B — GitHub / codebase wiki). Captures modules, components, data flows, decisions, and dependencies.

Secondary scope: A small seed cluster of pages about the LLM Wiki pattern itself (how this vault works) lives under `wiki/concepts/`, `wiki/entities/`, and `wiki/comparisons/`. These predate the CUBRID scope and are retained as meta-documentation.

## Mode B Conventions (CUBRID)

Structure under `wiki/`:
- `modules/` — one page per top-level CUBRID directory (broker, src, cs, pl_engine, ...)
- `components/` — subsystems (optimizer, page buffer, lock manager, ...)
- `decisions/` — ADRs (`NNNN-short-title.md`)
- `dependencies/` — one page per external / bundled library
- `flows/` — request paths, lifecycles, recovery sequences

Hub pages at `wiki/` root:
- [[Architecture Overview]] · [[Tech Stack]] · [[Data Flow]] · [[Dependency Graph]] · [[Key Decisions]]

Source of truth for CUBRID: `/Users/song/DEV/cubrid/` — **never write to the source tree**, only read.

## Vault Structure

```
.raw/           source documents — immutable, Claude reads but never modifies
wiki/           Claude-generated knowledge base
_templates/     Obsidian Templater templates
_attachments/   images and PDFs referenced by wiki pages
```

## How to Use

Drop a source file into `.raw/`, then tell Claude: "ingest [filename]".

Ask any question. Claude reads the index first, then drills into relevant pages.

Run `/wiki` to scaffold a new vault or check setup status.

Run "lint the wiki" every 10-15 ingests to catch orphans and gaps.

## Cross-Project Access

To reference this wiki from another Claude Code project, add to that project's CLAUDE.md:

```markdown
## Wiki Knowledge Base
Path: /path/to/this/vault

When you need context not already in this project:
1. Read wiki/hot.md first (recent context, ~500 words)
2. If not enough, read wiki/index.md
3. If you need domain specifics, read wiki/<domain>/_index.md
4. Only then read individual wiki pages

Do NOT read the wiki for general coding questions or things already in this project.
```

## Plugin Skills

| Skill | Trigger |
|-------|---------|
| `/wiki` | Setup, scaffold, route to sub-skills |
| `ingest [source]` | Single or batch source ingestion |
| `query: [question]` | Answer from wiki content |
| `lint the wiki` | Health check |
| `/save` | File the current conversation as a structured wiki note |
| `/autoresearch [topic]` | Autonomous research loop: search, fetch, synthesize, file |
| `/canvas` | Visual layer: add images, PDFs, notes to Obsidian canvas |

## MCP (Optional)

If you configured the MCP server, Claude can read and write vault notes directly.
See `skills/wiki/references/mcp-setup.md` for setup instructions.
