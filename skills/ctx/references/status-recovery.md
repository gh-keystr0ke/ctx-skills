# Recovering from an unhealthy `ctx status`

Read this when `ctx status --json`'s `health` field is anything other than `ready` (or, on older builds with no `health` field, when `index_state` isn't `current`, or `unmapped_intents`/`stale_claims`/`schema_divergences` are non-empty). The goal is always the same: identify the one specific cause, fix only what your current task is responsible for, and never expand into a repository-wide cleanup nobody asked for.

## The general procedure

1. Run `ctx status --json` and read `health`, `notices`, and `suggested_actions` together — `notices` names the affected relationships/entities, `suggested_actions` names the exact next command. Don't guess a fix from the `health` string alone.
2. Classify the cause using the table below. More than one can be true at once (e.g. `needs_index` *and* `needs_mappings`); handle them independently, not as one blob.
3. Ask: **did my current task cause this, or did it predate my change?**
   - Caused by your diff (you added a symbol with no test/implementation mapping, your edit made a mapped symbol stale): fix it as part of the same task, before calling the task done.
   - Predates your task (stale claims from someone else's earlier change, `unmapped_intents` that were already unmapped): say so to the user, leave it alone, and continue your own bounded work. Fixing it anyway is scope creep, not thoroughness — unless the user explicitly asks for a health cleanup as its own task.
4. Re-run `ctx status --json` after any fix to confirm it actually resolved, rather than assuming.

## Cause-by-cause

### `index_state: "behind"` / `"not_indexed"` (`needs_index`)

The local SQLite index doesn't reflect `HEAD`. Fix: `ctx index`. This only works when the working tree is clean for the configured source/context paths that require a commit (`git status --short`) — `ctx index` deliberately refuses to run over uncommitted changes to those inputs. If you're mid-edit and not ready to commit, don't force it: either commit first, or skip indexing for now and rely on `ctx review`, which works fine over an uncommitted diff regardless of index freshness. `.context`/`.ctx-candidates` inputs only carry this commit requirement when their (possibly redirected — `ctx context-store show`) location is itself a Git repository.

### Uncommitted index inputs

If `uncommitted_index_inputs` (or the `git status --short` check above) shows dirty files inside configured source/context paths, that's *why* `ctx index` is refusing to run, not a separate problem. Commit those files (or stash them if they aren't part of your task) rather than looking for a way around the check.

### `unmapped_intents` (`needs_mappings`)

An active Requirement/Invariant/Decision has no `implementation`/`tests` link (Features are organizing parents and are intentionally excluded from this check). This means a product claim exists that `ctx impact`/`ctx review` can't connect to any code — it's invisible to blast-radius analysis until mapped.

- If your task just introduced this document, you're responsible for it: add the missing `implementation:`/`tests:` entries per `references/authoring-context.md`, and verify the symbol paths resolve (`ctx find <symbol>`) rather than hand-deriving them.
- If it predates your task, note it and move on unless it's directly relevant to what you're changing.

### `stale_claims` / `stale_semantic_edges`

A previously confirmed `implementation`/`tests` mapping's code changed since it was last verified — not necessarily wrong, just unconfirmed since the change. `ctx explain "<source> -> <target>"` shows why the relationship was made and what changed.

- **Do not** run `ctx verify --stale` in bulk to clear these automatically. It shells out to a real agent CLI (`claude`/`codex`/`agy`) per stale claim — real time and, for the agent CLIs, real money — and it's explicitly an opt-in, user-requested action (see the skill's "What not to do"). Surface the list of stale claims to the user and let them decide whether re-verification is worth running.
- If a stale claim is a direct, obvious consequence of your own diff (you renamed the exact symbol a document points at), it's reasonable to fix the mapping by hand in the same commit rather than waiting on `ctx verify --stale` — that's editing `.context/`, not running bulk AI verification.
- If the stale claims predate your task and aren't touched by it, report the count and move on.

### `schema_divergences`

A best-effort mismatch between an ORM model's declared shape and the migration history's declared shape (presence-only — it doesn't diff types/nullability). If your task touched either side, check whether the divergence is real (a genuinely missing migration or an out-of-sync model) or a known, accepted gap; report either way rather than silently ignoring it.

### `needs_context` (no `.context/` yet)

Structural graph only — no product documents indexed. This isn't a broken state; it means onboarding hasn't happened. Don't try to "fix" it mid-task by inventing product documents from thin air. If the user wants ctx set up properly, that's `references/onboarding.md`, not a status-recovery detour.

## What "bounded read-only work" means when you decide not to fix a pre-existing issue

Continuing "bounded and read-only-safe" after flagging a pre-existing health problem means: your own `ctx context`/`ctx impact`/`ctx review` calls still work (they don't require a clean bill of health, just a current-enough index for the parts you're querying), you still edit only the files your task requires, and you still run `ctx review` before your own commit. It does not mean silently trusting stale data as if it were fresh — say plainly, when reporting results to the user, that a given claim is stale or unmapped rather than presenting it with the same confidence as a clean one.
