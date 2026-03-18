# Obsidian Integration Review: second-brain for Knowledge Base Adaptation

**Repository Reviewed:** [earlyaidopters/second-brain](https://github.com/earlyaidopters/second-brain)
**Date:** 2026-03-18

---

## Executive Summary

The `second-brain` project connects Obsidian (a local markdown note-taking app) with Claude Code to create a persistent, context-aware AI assistant. Knowledge compounds over sessions because Claude reads the vault before each interaction. This review evaluates which patterns are worth adapting for a knowledge base project and flags areas that need modification.

---

## Architecture Overview

```
User's Obsidian Vault
├── CLAUDE.md          ← "Brain of the brain" (user profile + vault rules)
├── memory.md          ← Append-only session log (cross-session context)
├── inbox/             ← Drop zone for unsorted files
├── daily/             ← Daily notes (YYYY-MM-DD.md)
├── projects/          ← Active work with briefs and status
├── research/          ← Synthesis and reference notes
├── archive/           ← Completed work (never deleted)
└── [role-specific]/   ← Generated during /vault-setup (e.g., clients/, decisions/)
```

Four Claude Code slash commands drive the workflow:
- `/vault-setup` — Interactive interview that scaffolds a personalized vault
- `/daily` — Morning standup: reads context, surfaces priorities
- `/tldr` — End-of-session capture: saves decisions, updates memory.md
- `/file-intel` — Converts external docs (PDF, DOCX, PPTX, XLSX) into vault-ready markdown via Gemini

---

## Strengths Worth Adopting

### 1. Context Compounding via CLAUDE.md + memory.md
The two-file system is elegant: `CLAUDE.md` holds static identity/rules while `memory.md` accumulates session-by-session learnings. This pattern is directly portable to any knowledge base project — it gives AI assistants persistent memory without a database.

**Adaptation:** Use `CLAUDE.md` as a project manifest that describes your knowledge base structure, conventions, and domain context. Use `memory.md` to track evolving decisions and discovered patterns.

### 2. Inbox-Zero Folder Pattern
The five-folder structure (`inbox/`, `daily/`, `projects/`, `research/`, `archive/`) enforces a clear lifecycle: capture → process → organize → archive. This prevents the common failure mode of flat, unsorted note dumps.

**Adaptation:** Map these folders to your knowledge domain. For a technical knowledge base, consider:
- `inbox/` → raw captures, bookmarks, snippets
- `reference/` → processed, structured documentation
- `projects/` → active investigations or learning tracks
- `templates/` → reusable note structures
- `archive/` → completed or superseded content

### 3. Document-to-Markdown Pipeline
`process_docs_to_obsidian.py` converts PDFs, DOCX, PPTX into structured markdown with YAML frontmatter (tags, source, date, type). This is highly valuable for ingesting external knowledge sources into a searchable vault.

**Adaptation:** The script uses Gemini for summarization (300-600 words per doc, YAML frontmatter generation). You could:
- Swap Gemini for Claude API if preferred
- Adjust the frontmatter schema to match your taxonomy
- Modify compression targets based on your use case (the 300-600 word limit may be too aggressive for detailed technical docs)

### 4. Slash Command Workflow Automation
The skills system (`/daily`, `/tldr`, `/file-intel`, `/vault-setup`) turns repetitive knowledge management tasks into one-command operations. This dramatically reduces friction.

**Adaptation:** Design custom slash commands for your specific workflows:
- `/ingest [source]` — Process and file new content
- `/review` — Surface notes that need updating or linking
- `/synthesize [topic]` — Generate summaries across related notes
- `/export [format]` — Publish vault content to other formats

---

## Weaknesses and Gaps

### 1. No Graph/Linking Strategy
Despite using Obsidian (whose killer feature is the knowledge graph), the project has **no explicit linking conventions**. There are no `[[wiki-link]]` patterns, no backlink strategies, no MOC (Map of Content) files. The vault is organized by folders only, missing Obsidian's most powerful feature.

**Recommendation:** Define linking conventions:
- Use `[[wiki-links]]` to connect related concepts
- Create MOC files for major topics (index pages that link to all related notes)
- Use tags in YAML frontmatter for cross-cutting concerns
- Leverage Obsidian's graph view to discover gaps in your knowledge base

### 2. Gemini Dependency for Core Functionality
The document processing pipeline requires a Google API key and Gemini. This adds an external dependency and potential cost for what could be done locally or with different models.

**Recommendation:** Make the LLM provider configurable. Abstract the summarization step behind an interface so you can swap between Gemini, Claude, local models (Ollama), or even rule-based extraction.

### 3. No Version Control Strategy for the Vault
The project doesn't address how to version-control an Obsidian vault. `.obsidian/` workspace files change constantly, binary attachments bloat repos, and merge conflicts in markdown can corrupt notes.

**Recommendation:**
- Add a comprehensive `.gitignore` for `.obsidian/workspace.json`, `.obsidian/workspace-mobile.json`, and cache files
- Use git-lfs for image/PDF attachments if version-controlling them
- Consider the Obsidian Git plugin for automated commits
- Or use a separate sync solution (Syncthing, iCloud) for the vault itself

### 4. No Search or Retrieval Beyond File Reading
Claude Code reads files sequentially. For large vaults (1000+ notes), this won't scale — context windows fill up before all relevant notes are loaded.

**Recommendation:** Implement a retrieval layer:
- Use embeddings + vector search to find relevant notes before loading them
- Build a local index (e.g., with SQLite FTS5) for fast keyword search
- Create a `/search [query]` command that pre-filters before sending to Claude

### 5. No Obsidian Plugin Integration
The system doesn't leverage Obsidian's plugin ecosystem. Plugins like Dataview, Templater, and QuickAdd could automate much of what the scripts do manually.

**Recommendation:** Evaluate these Obsidian plugins for your knowledge base:
- **Dataview** — Query notes like a database (filter by tags, dates, status)
- **Templater** — Dynamic templates with date insertion, prompts, etc.
- **QuickAdd** — Macro system for rapid note creation workflows
- **Periodic Notes** — Enhanced daily/weekly/monthly note management

### 6. Single-User Design
The system is built for one person. There's no concept of shared knowledge, access control, or collaborative editing.

**Recommendation:** If your knowledge base needs multi-user support:
- Use a shared git repo with branch-per-user for drafts
- Establish a review process for merging into the main vault
- Consider Obsidian Publish or a static site generator for read-only sharing

---

## Adaptation Roadmap

### Phase 1: Foundation
- [ ] Set up an Obsidian vault with the five-folder structure
- [ ] Create a `CLAUDE.md` tailored to your knowledge domain
- [ ] Initialize `memory.md` for session tracking
- [ ] Add a `.gitignore` covering Obsidian workspace/cache files

### Phase 2: Workflow Automation
- [ ] Port or rewrite the `/vault-setup` skill for your domain
- [ ] Implement `/daily` and `/tldr` equivalents
- [ ] Set up the document processing pipeline (choose LLM provider)
- [ ] Define YAML frontmatter schema for your content types

### Phase 3: Knowledge Graph
- [ ] Establish `[[wiki-link]]` conventions
- [ ] Create MOC (Map of Content) files for major topics
- [ ] Set up tag taxonomy in frontmatter
- [ ] Configure Obsidian graph view filters

### Phase 4: Scale
- [ ] Add retrieval/search layer for large vaults
- [ ] Implement embedding-based similarity search
- [ ] Build custom Obsidian plugins or Dataview queries
- [ ] Set up automated backup and sync

---

## Key Files to Study

| File | Purpose | Adaptation Value |
|------|---------|-----------------|
| `CLAUDE.md` | User profile + vault rules | High — template for your own project manifest |
| `memory.md` | Cross-session context log | High — directly reusable pattern |
| `skills/vault-setup/SKILL.md` | Interactive vault scaffolding | Medium — rewrite for your domain |
| `skills/daily/SKILL.md` | Morning standup workflow | Medium — adapt to your review cadence |
| `skills/tldr/SKILL.md` | Session summary + routing | High — the auto-routing logic is clever |
| `scripts/process_docs_to_obsidian.py` | Doc-to-markdown converter | High — core ingestion pipeline |
| `scripts/process_files_with_gemini.py` | Folder analysis + summaries | Medium — useful for bulk processing |
| `setup.sh` / `setup.ps1` | One-command installation | Low — too opinionated, write your own |

---

## Conclusion

The `second-brain` project provides a solid starting framework for an AI-augmented Obsidian knowledge base. Its strongest contributions are the **context compounding pattern** (CLAUDE.md + memory.md), the **inbox-zero folder structure**, and the **document ingestion pipeline**. Its main gaps are the lack of a **linking/graph strategy**, **scalable retrieval**, and **provider flexibility**. Adapting this for a knowledge base project should focus on adopting the workflow patterns while building out the missing graph and search capabilities that make Obsidian truly powerful.
