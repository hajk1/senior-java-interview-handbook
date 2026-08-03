# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A pure-Markdown content repository: a senior-level Java interview handbook. There is no build system, package manager, linter, or test suite — every file is documentation. "Development" here means writing and editing chapter content, not code.

## Structure and conventions

Each chapter lives in a numbered top-level directory with a single `README.md` (e.g. `01-java-core/README.md`, `02-spring/README.md`, `03-database/README.md`). Existing chapters follow this exact internal shape — match it when adding or editing content:

1. **Title** — `# <Topic> — Senior Interview Guide`.
2. **Intro paragraph** describing the chapter's scope and answer philosophy.
3. **"How to answer" callout** (blockquote) stating the expected answer structure for that chapter.
4. **`## Contents`** — a numbered TOC linking to each `##` section via Markdown anchors.
5. **Numbered `##` sections** (e.g. `## 1. Language and object model`), each containing sequentially numbered `### N. Question?` entries. Question numbering is continuous across the whole chapter, not reset per section.
6. Each answer leads with the direct rule/answer, explains the mechanism, then gives a code example, trap, or production consequence — code blocks use fenced ` ```java ` (or the relevant language). Tables are used for comparisons (e.g. access modifiers, isolation levels).
7. Final section is **`## 10. Rapid revision`**: a flat numbered list of all questions in the chapter (as prompts, no answers), followed by a **`### Thirty-second summary`** paragraph and an **`## Official references`** list of authoritative external links.

This structure mirrors `README.md`'s own "Interview principles": *Rule → mechanism → example → trade-off*.

### Chapter status and branch workflow

The top-level `README.md` and `ROADMAP.md` are the source of truth for which chapters are complete, in progress, or planned — check them before assuming a chapter exists locally. New chapters are developed on dedicated branches named `agent/add-<topic>-chapter` (see `agent/add-microservices-chapter`, `agent/add-system-design-chapter`) before being merged, so a chapter referenced in `README.md`'s chapter table may not yet exist on the current branch. Confirm with `git branch -a` / `git log --all` if a directory the README references appears to be missing.

`templates/chapter-template.md` and `templates/question-template.md` exist but are currently empty placeholders — do not treat them as authoritative; follow the structure of an existing complete chapter (`01-java-core`, `02-spring`, or `03-database`) instead.

### Root-level files

- `README.md` — landing page: chapter table with question counts and status, study order, and contribution guidelines.
- `ROADMAP.md` — authoritative, granular tracking of what's done vs. planned per chapter, phased.
- `QUICK-REVISION.md`, `INTERVIEW-CHECKLIST.md`, `INTERVIEW-LOG.md` — currently empty; planned per the roadmap's Phase 6.
- `cheat-sheets/`, `assets/diagrams/`, `assets/images/` — currently empty; planned supporting material.

## Contributing content

When adding or editing a question (per `README.md`'s contribution guidance):

1. Lead with the direct answer.
2. Explain the underlying mechanism.
3. Include a realistic example or failure mode.
4. Call out behavior that varies by version, database, JVM, or framework implementation.
5. Prefer official documentation for version-sensitive claims, and add it to the chapter's `## Official references`.

Keep answers senior-level and interview-focused: precise contracts over folklore, general principles distinguished from implementation-specific behavior, and technical choices connected to latency, throughput, consistency, and failure recovery — not certification-style trivia.

When a chapter's question count or status changes, update both the chapter table in `README.md` and the corresponding row/checklist in `ROADMAP.md` so they stay consistent.
