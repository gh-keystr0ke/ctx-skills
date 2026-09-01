# Onboarding a repository onto ctx

Run this only when the user has asked to add ctx to a repository, or explicitly asked for product-context onboarding/mining. Never run `ctx init` unprompted.

There are two ways to get from zero to a usable `.context/` baseline: hand-author a handful of high-value documents, or mine what already exists (Git history, code comments, GitLab, or referenced Jira issues) and have an agent propose candidates for human (or agent) review. They compose — most real onboardings do a little of both.

## 0. Prerequisites

- Rust 1.88+ and Git installed, and the `ctx` binary on `PATH` (or built via `cargo install --locked --path crates/ctx-cli` from the `ctx` source tree).
- A Git repository with at least one commit. `ctx index` always requires configured source to be committed; `.context/`/`.ctx-candidates/` require commits too unless they are redirected to a plain, non-Git context store.
- Decide `languages` for `.ctx/config.toml` up front: `python`, `rust`, `go`, and `goose` (SQL migrations) are the built-in modules. Unsupported or empty language sets fail during repository discovery.
- **Onboarding a repository you don't own or can't push into?** Run `ctx context-store set <path>` (add `--git` for the same commit-before-index guarantee this checkout's `.context/` normally has) *before* `ctx init`, so `.context/`/`.ctx-candidates/` are scaffolded at that redirected location instead of inside the checkout — nothing below writes into the repository itself. See `commands.md` for behavior and `authoring-context.md` for committed-versus-local paths.

## 1. Initialize

```bash
ctx init
```

Creates `.ctx/config.toml`, `.context/{features,requirements,invariants,decisions}/`, and local SQLite storage, and adds the local-only `.ctx/*` paths to the repository-local Git exclude file. Safe to re-run.

Adjust `.ctx/config.toml`'s `languages` and `[paths]` for the repository's real layout (see `authoring-context.md` § configuration) before the first index — changing it later is reconciled automatically but an accurate first pass avoids a redundant re-scan.

## 2a. Manual bootstrap (small, high-trust documents)

Best when the repository is small, or when you want a handful of very deliberate, high-value documents rather than broad coverage immediately.

1. Identify the 3–10 highest-value flows (the ones where "what does this affect" questions actually matter — payment, auth, data-destructive operations, public APIs).
2. Author one Requirement or Invariant per flow under `.context/`, per `authoring-context.md`'s schema, with exact `implementation`/`tests` symbol links.
3. `git add .ctx/config.toml .context && git commit` — or, if `.context/` was redirected (see Prerequisites), commit inside that separate location instead; nothing to add in this checkout.
4. `ctx index`
5. `ctx status --json` — confirm `health` moved from `needs_context` and check `unmapped_intents` is empty. You don't need full coverage up front; `ctx status` tells you what's mapped and what isn't as you go.

## 2b. Fully automated onboarding (mine existing knowledge)

Best when the repository has real history worth mining: meaningful commit messages, doc comments/docstrings, an active GitLab project, or referenced Jira issues. This gets you from zero to a populated, evidence-backed `.context/` baseline without hand-authoring anything — at the cost of it being agent-judgment, not human-authored, until spot-checked.

```bash
# 1. Commit configuration and build the initial structural index
git add .ctx/config.toml
git commit -m "chore: configure ctx"
ctx index

# 2a. Default all-artifacts pipeline (legacy behavior)
# Ingest source material; --since narrows only `git` to newer commits.
ctx ingest git
ctx ingest code-comments --reconcile
ctx ingest gitlab              # only if [gitlab] is configured; CTX_GITLAB_TOKEN is optional for public projects
ctx ingest jira                # run after git/gitlab; referenced Jira issues plus the legacy one-hop expansion

# 3a. Have an AI agent propose typed candidates from all ingested material
ctx enrich --agent claude      # or --agent codex / --agent antigravity — needs that CLI authenticated on PATH

# OR use the strict business-linked pipeline for steps 2 and 3:
# GitLab fetches only MRs selected by current Git; Jira and enrichment stay Jira-anchored.
ctx ingest git
ctx ingest code-comments --reconcile
ctx ingest gitlab --scope business-linked
ctx ingest jira --scope business-linked --related-depth 0
ctx enrich --scope business-linked --related-depth 0 --agent claude

# 4a. Human review, one candidate at a time
ctx verify --knowledge
ctx verify --knowledge --accept <FINGERPRINT> --id REQ-SUB-014 --author <name>
ctx verify --knowledge --reject <FINGERPRINT> --author <name>

# 4b. OR: bulk-review via a second independent agent instead of a human
ctx verify --knowledge --auto --agent claude --id-prefix SUB

# 5. Commit accepted documents and candidate-queue updates before indexing
git add .context .ctx-candidates
git commit -m "docs: add mined product context"

# 6. Index and check health
ctx index
ctx status --json
```

Notes:

- `ctx ingest`/`ctx enrich` never write to `.context/` directly — they produce Git-tracked pending candidates under `.ctx-candidates/`. Nothing becomes a real, indexed document until step 4 accepts it.
- `ctx ingest code-comments --reconcile` treats comments/docstrings successfully analyzed from HEAD as a complete snapshot. It removes stored entries absent from that snapshot, but only after the full replacement snapshot has been read and analyzed successfully, so a source/analyzer failure cannot erase the last good state. Re-running it is idempotent.
- Struct/type-level doc comments are legitimate signal, not noise — a repository's domain modeling is often expressed through type declarations as much as function comments. Don't dismiss a candidate just because its source comment was attached to a struct rather than a function.
- `ctx enrich` shells out to a real, already-authenticated agent CLI and analyzes one bounded artifact neighborhood at a time — expect it to take real wall-clock time (and, for hosted models, real cost) proportional to the number of ingested artifacts. It's fine to run it in the background and check back.
- `--auto`'s decisions are always recorded as agent-made — `ctx explain` renders them "Auto-verified", not human-reviewed. Report this honestly to the user; it is a strong starting baseline, not equivalent to a human having read every document. For a first-time onboarding of an unfamiliar codebase, spot-check a sample of the resulting `.context/*.yaml` documents (`ctx explain <id>`) before treating the baseline as trustworthy, and mention to the user that this is a good moment to skim.
- A likely restatement of an already-active document is left pending unless `--force` — this is deduplication working as intended, not a bug to route around by default.
- If GitLab or Jira isn't configured or the team doesn't use it, skip that ingest source — `git` and `code-comments` alone still produce a useful candidate set for most repositories. When the team's branches/commits already carry ticket keys (e.g. `PSI-1122-fix`), `ctx ingest jira` alone is often the highest-value addition: the ticket key becomes an issue artifact's `external_id`, so `ctx-core`'s existing deterministic ticket-key linking resolves those branches/commits to the issue with no extra step.

### Business-linked scope and existing artifact stores

Use `business-linked` when Jira issues are the business anchors and the repository should not accumulate unrelated issues/MRs merely because they belong to the configured GitLab/Jira projects. This is opt-in: `all` remains the default for ingest and enrich. A strict run must use the scope consistently for `gitlab`, `jira`, and `enrich`, and use the same `--related-depth` for Jira ingestion and enrichment (`0` is the default and retains no Jira `RelatedIssue` hops).

The scope is evidence-based, not project-based. GitLab selects MRs only when current Git contains a literal `!IID`, matches the MR source branch, or matches its head/merge/squash commit SHA; it does not fetch GitLab issues in this mode. Jira keys come only from current Git plus those selected MRs/comments. Old Jira/GitLab artifacts already in the local store cannot make themselves relevant.

Scoped ingestion does not delete artifacts stored by earlier all-artifacts runs. If the user asks to clean the existing store, preview the exact plan first:

```bash
ctx artifacts prune --scope business-linked --related-depth 0 --json
```

Review every planned identity/reason and any warning about pending candidates. Candidate files and `.context/` are deliberately not rewritten. Only with explicit user authorization, apply the identical scope/depth:

```bash
ctx artifacts prune --scope business-linked --related-depth 0 --apply
```

Apply removes exactly the dry-run set plus its local artifact links and analysis rows in one transaction. Re-running the same prune is idempotent. Use `-v` for reason counts or `-vv` for identities with reasons when human-readable output is preferable to the complete JSON plan.

## 3. Iterate to `ready`

After either path, `ctx status --json`'s `health` field tells you what's left:

| `health` | Means | Next step |
| --- | --- | --- |
| `needs_index` | `index_state` isn't `current` | `ctx index` (working tree clean) |
| `needs_context` | No product documents indexed yet | Author or mine some (2a/2b above) |
| `needs_mappings` | An active Requirement/Invariant/Decision has no implementation/test link | Add exact `implementation`/`tests` symbols to the documents in `unmapped_intents` |
| `needs_attention` | Stale claims or schema divergences exist | Investigate the named items in `stale_claims`/`schema_divergences` |
| `ready` | Fully current and mapped | Onboarding is done; ongoing work follows `SKILL.md`'s per-task pipeline |

`ctx status`'s own `suggested_actions` array names the exact next command — prefer it over guessing.

## Optional: multiple repositories on the same team

If this repository is one of several services owned by the same team, see `federation.md` once each individual repository has its own `.context/` baseline — federation shares a bounded, `public`-only slice of that baseline across sibling checkouts, it isn't a substitute for step 1–3 in each repository.
