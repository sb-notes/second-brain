# Second Brain v2 — Architecture

Personal knowledge and state management system for a Claude-powered second brain spanning multiple project contexts. Notion is the write surface and human-readable UI. GitHub is the read mirror. Sync is one-way only — Notion to GitHub — on a schedule. This document is the canonical spec for the v2 file system and is referenced by every project's custom instructions and every Skill that reads or writes state.

## File tiers

Every project is split into four file types. Each has a distinct purpose, a distinct load rule, and a distinct token budget. Keeping these boundaries clean is the entire point of v2 — it is what makes the system fast, cheap, and maintainable.

**Live state file** — named `<project>-live.md`. Loaded on every new thread. Contains only the current snapshot, active priorities, open items, and pointers to warm files for deeper content. Target token budget: 300 to 600 tokens. If a live file grows beyond 600 tokens, content is pruned down into warm files during the next `update` run. The live file never contains historical narrative; history belongs in the changelog.

**Warm specialist files** — named `<project>-<topic>.md`, for example `<project>-subject-a.md` or `<project>-protocols.md`. Loaded on demand when the conversation touches the topic. Contains durable facts, reference material, logs, and reusable strategies for a single clean theme. Target budget: 1,000 to 3,000 tokens per file. Typically three to seven warm files per project. The live file carries a table of contents so Claude knows which warm file to reach for.

**Cold archive** — named `<project>-archive.md`. Loaded only on explicit request. Contains superseded decisions, old narratives, and anything the live and warm files no longer need but should not be lost. No token budget. Read rarely, grows slowly.

**Changelog** — named `<project>-changelog.md`. Append-only chronological log, newest entry at top. One entry per meaningful change, one to two lines each, stating what changed and why. Loaded only on explicit request such as "what changed recently in this project?". Not loaded during normal Q&A or update cycles. Centralising the changelog per project (rather than per file) keeps warm files tight.

## Frontmatter schema

Every markdown file begins with YAML frontmatter:

```yaml
---
project: <project-slug>
file: <project-slug>-live
tier: live
schema_version: 2
last_updated: YYYY-MM-DD
token_budget: 600
---
```

The `schema_version` field lets us migrate the format later without guessing what shape old files are in. The `last_updated` field gives Skills a cheap staleness check. The `token_budget` field makes pruning rules mechanical rather than judgement-based.

## Anchor pattern for append-only writes

Files that support append operations — changelog files and any warm file with an explicit session-log zone — carry a fixed anchor comment at the append point:

```markdown
<!-- APPEND_ANCHOR -->
```

Skills performing an append write use `str_replace` with `old_str` equal to the anchor and `new_str` equal to the anchor followed by the new content. The anchor never moves, so no fetch-before-write is needed. This is what collapses the three-call Notion write pattern into a single call and is the mechanical win that makes v2 affordable.

Live state files are *not* appended to. They are rewritten whole, because they are small enough (under 600 tokens) for Claude to regenerate in one write operation without needing to match partial content.

## De-duplication rule

A fact lives in exactly one file. Other files may reference it by name — "see `<project>-subject-a.md` for current medications" — but must never restate it. When a fact moves between tiers (for example, a resolved issue dropping from live into an archive), it is deleted from the origin in the same write. This rule is the single biggest defence against drift and is enforced by every Skill at write time.

## Anonymisation rules

The repo operates under **defence-in-depth anonymisation**. Multiple independent layers protect identity; no single layer is trusted alone.

**Layer one — core-name codenames.** State files use codenames in place of real names for core family members. The codename map is *not stored in this repo*. It lives in a secured Notion page referenced only by project-level custom instructions, and it is injected into Skills at write time via local context, never committed to version control. The principle is simple: if this repo were downloaded by a stranger tomorrow, they would see codenames throughout with no key to unlock them.

**Layer two — third-party scrub.** Any file synced to the public mirror must also scrub third-party real names (healthcare providers, therapists, accountants, colleagues, neighbours), specific addresses and postcodes, employer names, children's nursery and school names, and any other identifier that could connect the content to a real person. Third parties who appear repeatedly enough to need stable reference receive their own third-party codenames (for example `therapist-c`, `accountant-m`) stored in the same out-of-repo map as layer one. Anything that cannot be cleanly scrubbed stays Notion-only and is excluded from the sync.

**Layer three — hosting identity.** The GitHub account or organisation hosting the repo must not be publicly linked to the real operator. A neutral organisation name or a dedicated burner account is required. The personal handle of the operator must not appear in the repo URL, commits, or any file.

**Layer four — commit hygiene.** Git history is forever. Scrub passes happen *before* commit, never *after*, because rewriting history on a public repo is unreliable and any push of real content has to be assumed permanently cached somewhere. When in doubt, the file does not get pushed.

Any Skill that writes to a state file applies codename substitution using the out-of-repo map before the write. Any Skill that reads a state file passes content back to conversation unchanged; Claude translates codenames back to real names in spoken context using the same out-of-repo map.

## Custom instructions template

Every project's Claude custom instructions follow the same tight template, well under 200 words:

```
You are [persona] for [project name].

On thread start, fetch the live state file from GitHub:
https://raw.githubusercontent.com/sb-notes/second-brain/main/<project>-live.md

Load warm files on demand when the conversation touches their topic.
The live file contains a table of contents pointing to warm files.

When I say "update", run the update Skill.
When I say "archive <topic>", run the archive Skill for that topic.

Codename rule: apply the project codename map at write time only;
use real names in conversation. The map is defined in the project's
private context and is never written to the public mirror.
```

All procedural logic lives in Skills, not in custom instructions. Custom instructions identify the project, point at the live file, list the command verbs, and stop.

## The `update` command contract

Every project's `update` Skill follows the same four-stage contract, so they're interchangeable and maintainable:

1. **Identify** new information in the conversation thread that belongs in state files.
2. **De-duplicate**: for each new fact, check whether it already exists in a warm file and update in place rather than creating a duplicate.
3. **Route** new content to the correct tier — live for current priorities and snapshots, warm for durable facts, archive for superseded material.
4. **Log**: append a one-line entry to `<project>-changelog.md` describing the delta and reason, dated today.

Every write applies both codename substitution and the third-party scrub rules from the anonymisation section. The Skill reports what it wrote and where, one line per file, and nothing else.

## Versioning and migration

The repo root contains this file (`ARCHITECTURE.md`) and a `VERSION` file containing the current `schema_version`. When the architecture changes, bump the version in both places and provide a migration Skill that walks old files through the upgrade. Files carrying an older `schema_version` in their frontmatter are flagged for review but continue to work under legacy rules until migrated.

## What is explicitly out of scope for v2

V2 does not include: two-way sync between Notion and GitHub, automatic cross-project fact synchronisation, multi-user access, programmatic conflict resolution, or an API layer for external agents to write directly to state files. All of those are plausible later. V2 is scoped to solving token bleed, reliability, maintainability, and anonymisation for a single-user Notion-and-GitHub setup.
