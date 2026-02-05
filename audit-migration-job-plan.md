# Blog Author Details Migration Plan

## Overview
This document describes the audit-and-migration trilogy to find and fix historical orphaned
`BlogAuthorDetailsModel` records left behind by an earlier wipeout process. It is intended
as an implementation and operational guide for Oppia engineers and release operators.

Summary:
- Historical bug: `BlogAuthorDetailsModel` records were not updated/deleted when users were
  pseudonymized/deleted. Blog posts were pseudonymized (author_id `uid_*` → `pid_*`) or
  left with `pid_*` values, but author-detail records still referenced old `uid_*` values.
- Goal: Provide a production-safe Beam-based audit job, a migration job to fix issues, and a
  validation job to verify correctness.

Status for this iteration: Documentation only. No code changes are applied yet.

---

## Background & Problem Statement

When users were wiped out historically:
- `UserSettingsModel` was removed for deleted users.
- `BlogPostModel.author_id` was pseudonymized (new value `pid_*`).
- `BlogAuthorDetailsModel` was NOT updated or removed and may still contain `author_id` like `uid_*`.

Consequences:
- `BlogAuthorDetailsModel.id` format = `author_{author_id}` (e.g., `author_uid_abc123`).
- Orphaned author detail models referencing old user IDs do not match blog posts' `author_id`.
- Broken reference chain: blog posts → author id (pid_*) → author model (author_pid_*) missing.


## Problem Categories (to detect)

1. Orphaned Models (High priority)
   - `BlogAuthorDetailsModel` exists with `author_id` using old format (`uid_*`) or mismatched
     value that does not correspond to any `UserSettingsModel` and does not match blog posts.
   - Fix: Update `BlogAuthorDetailsModel.author_id` and model `id` to use current pseudonymized `pid_*`.

2. Missing Models (Medium priority)
   - Blog posts exist with `author_id` (active `uid_*` or pseudonymized `pid_*`) but no
     corresponding `BlogAuthorDetailsModel` exists at id `author_{author_id}`.
   - Fix: Create `BlogAuthorDetailsModel` with fallback data.

3. Deleted User Posts (Informational)
   - Blog posts with pseudonymized `pid_*` author IDs. Models may or may not exist. Track counts.

4. Valid Models (Baseline)
   - `BlogAuthorDetailsModel.author_id` matches `BlogPostModel.author_id` and `UserSettingsModel`
     exists (when relevant). No action required.


## Job Trilogy (High-Level)
All jobs MUST follow the JobTrilogy pattern described in the request and be implemented using
Oppia's Beam + `ndb_io` utilities (no direct `get()`/`put()` etc.).

Files to create (final implementation):
```
core/jobs/batch_jobs/
├── blog_author_details_audit_jobs.py
├── blog_author_details_migration_jobs.py
├── blog_author_details_validation_jobs.py
└── blog_author_details_audit_jobs_test.py
```

The trilogy:
- Audit: Read-only; report counts, sample IDs (max 10 per category) and totals.
- Migrate: Make changes (update or create models). Log each change. Provide detailed counts.
- Validate: Confirm migration removed all issues; report PASS/FAIL.


## Beam Pipeline Design

High-level approach:
- Read all `BlogPostModel` and `BlogAuthorDetailsModel` using `ndb_io.GetModels(query)`.
- Key blog posts by `author_id` and author models by `author_id`.
- CoGroupByKey to correlate posts and author models by author_id.
- For groups where an author model exists but uses old `uid_*` ID that doesn't match posts' `author_id`,
  classify as orphaned — determine mapping to target `pid_*` if possible (see "Determining mappings").
- Where blog posts exist without author models: missing models.
- Where `pid_*` posts exist: deleted user posts (informational).

Important: Label every transform clearly (see Template restrictions).

Example pipeline skeleton (Audit job):

class AuditBlogAuthorDetailsJob(base_jobs.JobBase):
    def run(self) -> beam.PCollection[job_run_result.JobRunResult]:
        blog_posts = (
            self.pipeline
            | 'Get all blog posts' >> ndb_io.GetModels(
                blog_models.BlogPostModel.get_all()
            )
        )

        author_models = (
            self.pipeline
            | 'Get all author models' >> ndb_io.GetModels(
                blog_models.BlogAuthorDetailsModel.get_all()
            )
        )

        keyed_posts = (
            blog_posts
            | 'Key posts by author' >> beam.Map(lambda p: (p.author_id, p))
        )

        keyed_models = (
            author_models
            | 'Key models by author_id' >> beam.Map(
                lambda m: (m.author_id, m)
            )
        )

        joined = ({'posts': keyed_posts, 'models': keyed_models}
                  | 'CoGroup posts and models' >> beam.CoGroupByKey())

        categorized = (
            joined
            | 'Categorize groups' >> beam.FlatMap(self._categorize_group)
        )

        # Aggregate counts, sample ids, format output exactly as required.

Notes on required I/O primitives:
- Use `ndb_io.GetModels(query)` for reads.
- Use `pcoll | ndb_io.PutModels()` for writes in migration job.
- NEVER call `model.get()`, `model.put()`, `ndb.put_multi()`, `ndb.delete_multi()` or
  `datastore_services.query_everything()`.


## Determining mappings (uid_* → pid_*)

There are two possibilities for mapping old `uid_*` author IDs to their current pseudonymized
`pid_*` equivalents:

1. If the pseudonymization mapping is stored in another persistent model or can be derived
   deterministically from blog content or logs, use that. (Preferred if available.)
2. If no mapping is available, derive mapping by inspecting all blog posts: for an `author_uid_X`
   author model, look for blog posts whose `author_id` is `pid_*` and whose historical link(s)
   indicate they belong to the same person. If this is impossible, fall back to one of:
   - If there is a single `pid_*` value used across posts that were authored by the same person,
     assume that `pid_*` is the target.
   - Otherwise, log as ambiguous and skip automatic update — surface to operators.

Operationally we assume we can identify mapping for the majority of orphaned models by
correlating `BlogPostModel` records and author model ids.

Implementation detail (safe default):
- Audit job should report orphaned models and candidate target `pid_*` values (if derivable).
- Migration job should update only when the mapping is unambiguous (one-to-one); otherwise,
  write an error and skip (or create a fallback model keyed to the pid_* used by posts).


## Helper functions (recommended)

def _is_pseudonymized(author_id: str) -> bool:
    return author_id.startswith('pid_')

def _extract_author_id_from_model_id(model_id: str) -> str:
    return model_id.replace('author_', '', 1)

def _create_fallback_author_model(blog_post: BlogPostModel) -> BlogAuthorDetailsModel:
    return blog_models.BlogAuthorDetailsModel(
        id=f'author_{blog_post.author_id}',
        author_id=blog_post.author_id,
        displayed_author_name='Blog Author',
        author_bio=''
    )


## Audit Job: Responsibilities & Output

Responsibilities:
- Read-only analysis of all blog posts and author models.
- Detect counts and provide up to 10 sample IDs per category.
- Output the textual report exactly in the requested format.

Output format: (exact template)
```
===== Blog Author Details Audit Report =====
Total blog posts analyzed: 1,523
Total author models found: 847

ISSUES DETECTED:
├─ Orphaned models (old author_id): 234
│  └─ Sample IDs: ['author_uid_abc123', 'author_uid_def456', ...]
│
├─ Missing models (no author details): 89
│  └─ Sample blog post IDs: ['post_xyz789', 'post_uvw012', ...]
│
└─ Posts from deleted users: 347
   └─ Sample author IDs: ['pid_aaa111', 'pid_bbb222', ...]

BASELINE:
└─ Valid models (correct state): 1,189

RECOMMENDATION: 
Run MigrateBlogAuthorDetailsJob to fix 323 total issues.
```

Safety: The audit job must make zero writes.


## Migration Job: Responsibilities & Safety

Responsibilities:
- Update orphaned `BlogAuthorDetailsModel.author_id` and model `id` (rename/replace) to match the
  target `pid_*` when mapping is unambiguous.
- Create missing `BlogAuthorDetailsModel` with fallback data for `pid_*` or `uid_*` author_ids.
- Log every change and return a structured summary.

Safety rules (mandatory):
- Only run after Audit job has been reviewed and accepted by an engineer.
- Batch updates, respect production limits (discuss >10k modifications with the team).
- If `author model id` must be changed (id includes author id), implement as "create new model with
  correct id, copy fields, write with PutModels, then optionally delete old model" — but prefer
  creating new models and leaving old ones for manual review when in doubt.
- Use `ndb_io.PutModels()` to persist changes.
- Do not perform `ndb.delete_multi()` directly; if deletion is required, use an ID-marking strategy or
  a Beam Delete transform that uses allowed utilities (e.g., `ndb_io.DeleteModels()`), following
  Oppia's deletion safety policies.

Migration job output format (required template):
```
===== Blog Author Details Migration Report =====
Models updated: 234
Models created: 89
Total changes: 323

UPDATED MODELS (author_id corrected):
├─ author_uid_abc123 → author_pid_xyz789
├─ author_uid_def456 → author_pid_uvw012
└─ ... (234 total)

CREATED MODELS (new fallback models):
├─ author_pid_aaa111 (for blog post: post_123)
├─ author_pid_bbb222 (for blog post: post_456)
└─ ... (89 total)

ERRORS: 0

Status: SUCCESS ✅
Next step: Run ValidateBlogAuthorDetailsJob
```

Error handling:
- For each attempted update/create, return a per-model `JobRunResult` line. Collect errors and
  include them in the final report (count + list).

Reversibility:
- Keep backups/snapshots before migration (see Execution Plan below).
- Migration should be idempotent: re-running should not produce additional changes if already applied.


## Validation Job: Responsibilities

Responsibilities:
- Confirm no orphaned models remain.
- Confirm all blog posts have a corresponding `BlogAuthorDetailsModel` with matching `author_id`.
- Report PASS/FAIL and counts in required format.

Output template:
```
===== Blog Author Details Validation Report =====
Status: PASS ✅

All checks passed:
├─ Orphaned models: 0 (was 234)
├─ Missing models: 0 (was 89)
└─ All blog posts have valid author models: YES

Migration was successful. No further action needed.
```


## Execution Plan (step-by-step)

Pre-requisites:
- Take datastore backup / snapshot (required for rollback).
- Run Audit job on a staging or backup copy and review results.
- Create a Jira/PR with migration plan and get sign-off from the team lead.

Step 0 — Dry run & audit (READ-ONLY):
- Run `AuditBlogAuthorDetailsJob` in dataflow/runner in a dry-run, collect full report.
- Save full CSV/JSON of candidate orphaned rows to GCS for manual inspection.

Step 1 — Small-batch migration (smoke test):
- Prepare migration mapping for a small deterministic subset (e.g., 100 authors) where mapping is
  unambiguous.
- Run `MigrateBlogAuthorDetailsJob` restricted to that subset (pass job parameter / filter).
- Validate with `ValidateBlogAuthorDetailsJob`.

Step 2 — Rollout in batches:
- If smoke test passes, run migration in batches (10k models per batch threshold is a recommended
  limit; consult infra team if >10k required).
- After each batch, run validation for that batch.

Step 3 — Full migration:
- Run migration for all remaining candidates.
- Run final validation job; confirm PASS.

Step 4 — Post-migration:
- Monitor dashboards and logs for anomalies.
- Retain backups for rollback period.

Commands (example local/job runner):

```bash
# Audit (local or cloud runner as configured in Oppia jobs paradigm)
python -m scripts.run_beam_job --job_class=AuditBlogAuthorDetailsJob --output_gcs=gs://my-bucket/reports/audit.json

# Migration (after audit review)
python -m scripts.run_beam_job --job_class=MigrateBlogAuthorDetailsJob --mapping_gcs=gs://my-bucket/reports/audit_candidates.json

# Validation
python -m scripts.run_beam_job --job_class=ValidateBlogAuthorDetailsJob --report_gcs=gs://my-bucket/reports/validation.json
```

(Adjust commands to Oppia's job runner conventions; the above are illustrative.)


## Rollback Plan

1. Always ensure a datastore backup/snapshot is taken and verified before any migration run.
2. Since migration will create/put models, rollback involves restoring the datastore snapshot
   or reversing updates by re-creating previous models from audit artifacts.
3. For changes that are limited in scope (a single batch), roll back by running a reverse job that:
   - Reads the migration report and recreates deleted/old models and reverts `author_id` values.
4. Communicate rollback window to stakeholders, and do not purge the backups until rollback window
   closes.


## Success Criteria

- Audit job identifies all problem categories and provides sample IDs.
- Migration job updates or creates models such that:
  - No `BlogAuthorDetailsModel` remains with `author_id` that is inconsistent with `BlogPostModel.author_id`.
  - All blog posts have a corresponding `BlogAuthorDetailsModel` at `author_{author_id}`.
- Validation job reports PASS and zero remaining issues.
- No datastore errors; no data loss related to blog content.


## Estimated Runtime

Estimate depends on data volume. Example guidance:
- Read throughput: a few seconds per thousand models in Beam (varies by runner).
- For 100k blog posts + 50k author models expect minutes to low-hours depending on cluster size.
- Plan for staging run to measure real runtime; schedule maintenance window if migration >1 hour.


## Risk Assessment

Top risks:
- Ambiguous mappings (uid_* → pid_*): incorrect mapping could misattribute authorship.
- Large-scale writes causing quota/exhaustion issues or affecting live performance.
- Bugs in migration logic causing unintended model overwrites.

Mitigations:
- Audit first; only migrate when mappings are unambiguous.
- Perform small smoke tests; run migrations in batches.
- Require sign-off and backups before full run.


## Testing Plan

Automated unit tests (must implement):
- `AuditBlogAuthorDetailsJobTests` covering:
  - Empty datastore (0 posts)
  - All valid models
  - Orphaned model detected
  - Missing model detected
  - Deleted user posts counted
  - Multiple issues simultaneously
- Migration job tests:
  - Update orphaned models successfully
  - Create missing models
  - Skip ambiguous mappings and report them
- Validation job tests:
  - PASS/FAIL conditions

Tests should use `job_test_utils.JobTestBase` as Oppia jobs usually do.

Example test runner command:

```bash
python -m scripts.run_backend_tests --test_target core.jobs.batch_jobs.blog_author_details_audit_jobs_test.AuditBlogAuthorDetailsJobTests
```


## Sample Data & Dry-run Evidence (what to collect)

- From Audit run: JSON/CSV with the list of candidate orphaned model IDs and suggested `pid_*` mapping.
- From Migration run: detailed log of created/updated model IDs and any errors.
- From Validation run: PASS/FAIL report.

Collect and attach these artifacts to the migration PR.


## Implementation Notes & Coding Guidelines

- Use `ndb_io.GetModels` and `ndb_io.PutModels` for reads and writes.
- Give explicit labels to transforms, e.g., `'Key posts by author'`, `'CoGroup posts and models'`,
  `'Filter orphaned models'`, etc.
- Follow Oppia conventions: tests must reach 100% coverage for modified code; avoid `Any` types.
- Keep migrations idempotent.
- Use small, well-defined helper functions for categorization and safety-wrapped update calls.


## Example Categorization Logic (pseudocode)

1. CoGroup posts and author models by author_id.
2. For each key `aid`:
   - posts = group['posts']
   - models = group['models']

   Cases:
   - If models exist and model.author_id != aid: this indicates a model recorded with an old author_id
     (often `uid_*`) — mark as ORPHANED. Attempt to find target `pid_*` by inspecting posts.
   - If no model exists and posts exist: MISSING — create fallback model.
   - If `aid` is `pid_*`: increment "deleted user posts" counter.
   - Else: VALID.


## Reporting & Operator Guidance

Before running migration in production:
- Attach the Audit report to the PR and ask reviewer to confirm threshold for automatic fixes.
- If many ambiguous mappings are present, request manual triage instead of wholesale migration.

Recommended run checklist for operator:
- Backup datastore, verify backup integrity.
- Run `AuditBlogAuthorDetailsJob` and review sample IDs.
- Run smoke `MigrateBlogAuthorDetailsJob` for a small subset.
- Run validation; if PASS, continue in batches until complete.


## Appendix: Templates for Job Output (copy into job source for consistent formatting)

- Audit Report template: See section 'Audit Job: Responsibilities & Output'.
- Migration Report template: See section 'Migration Job: Responsibilities & Safety'.
- Validation Report template: See section 'Validation Job: Responsibilities'.


## Next Steps (for implementers)

1. Implement `AuditBlogAuthorDetailsJob` and tests (follow test template in this doc).
2. Run audit on staging/backup datastore and iterate until report quality is sufficient.
3. Implement `MigrateBlogAuthorDetailsJob` with batch mode and thorough logging.
4. Implement `ValidateBlogAuthorDetailsJob` and tests.
5. Execute smoke-test and then full migration following the plan above.


---

If you want, I can now create the job stub files and unit test skeletons in
`core/jobs/batch_jobs/` consistent with this plan. Which step should I take next?
