# Blog Author Details Migration Jobs - Line-by-Line Documentation

## Overview

This document provides detailed line-by-line explanations of the beam jobs in `core/jobs/batch_jobs/blog_author_details_migration_jobs.py`.

These jobs solve a critical issue: **deleted users' blog posts cause 500 errors during homepage pagination** because the rendering code tries to auto-create a `BlogAuthorDetailsModel` for a deleted user and fails.

The solution: Create placeholder `BlogAuthorDetailsModel` entries (with name "Deleted User") for truly-deleted authors so pagination doesn't crash.

---

## Part 1: Module Setup & Imports (Lines 1-60)

### File Header & Documentation (Lines 1-34)

```python
# coding: utf-8
#
# Copyright 2026 The Oppia Authors. All Rights Reserved.
...
```

- **Line 1-15**: Standard Apache 2.0 license header required by Oppia
- **Line 17-34**: Module-level docstring explaining the problem and solution
  - Lines 19-22: Describes the core issue (deleted user posts cause 500 errors)
  - Lines 24-27: Explains what the audit job does (finds orphaned authors)
  - Lines 29-32: Explains what the migration job does (creates fallback models)
  - Lines 34-37: Critical note: active users without models are NOT flagged (runtime handles them)

### Imports Section (Lines 36-60)

```python
from __future__ import annotations

from core import utils
from core.jobs import base_jobs
from core.jobs.io import ndb_io
from core.jobs.transforms import job_result_transforms
from core.jobs.types import job_run_result
from core.platform import models

import apache_beam as beam
```

- **Line 38**: `from __future__ import annotations` — enables forward references for type hints
- **Lines 40-48**: Import core Oppia dependencies:
  - `utils`: Utility functions for hashing and random number generation
  - `base_jobs`: Base class for all beam jobs
  - `ndb_io`: I/O transforms for reading NDB models into Beam pipelines
  - `job_result_transforms`: Transforms to count and format results
  - `job_run_result`: Data structure for job output messages
  - `models`: Registry to load domain models dynamically
- **Line 50**: `import apache_beam as beam` — Apache Beam library for parallel processing

### Type Stubs & Model Loading (Lines 52-60)

```python
MYPY = False
if MYPY:  # pragma: no cover
    from mypy_imports import base_models
    from mypy_imports import blog_models
    from mypy_imports import datastore_services
    from mypy_imports import user_models

(base_models, blog_models, user_models) = models.Registry.import_models(
    [models.Names.BASE_MODEL, models.Names.BLOG, models.Names.USER]
)
datastore_services = models.Registry.import_datastore_services()
```

- **Lines 52-57**: Type stub imports for mypy static type checking
  - MYPY is False at runtime, so these imports are skipped
  - They exist only for static analysis and IDE autocompletion
  - Pragma no cover excludes from test coverage

- **Lines 59-61**: Dynamic model loading (runtime)
  - `models.Registry.import_models()` loads domain models at runtime (avoids circular imports)
  - Loads: `BaseModel`, `BlogModel`, `UserModel` domain classes
  - This pattern is used throughout Oppia for safe dynamic importing

- **Line 62**: Load datastore services for NDB context management
  - `datastore_services` provides `get_ndb_context()` to manage NDB operations

### Constants (Lines 64-65)

```python
DELETED_USER_FALLBACK_AUTHOR_NAME = 'Deleted User'
DELETED_USER_FALLBACK_AUTHOR_BIO = ''
```

- **Lines 64-65**: Module-level constants used when creating placeholder models
  - `DELETED_USER_FALLBACK_AUTHOR_NAME = 'Deleted User'`: Display name shown when author is deleted
  - `DELETED_USER_FALLBACK_AUTHOR_BIO = ''`: Empty bio for deleted user models

---

## Part 2: AuditBlogAuthorDetailsForDeletedUsersJob Class (Lines 67-191)

### Class Definition & Docstring (Lines 67-88)

```python
class AuditBlogAuthorDetailsForDeletedUsersJob(base_jobs.JobBase):
    """Job that audits published blog post summary models for deleted
    author_ids that have no BlogAuthorDetailsModel.

    An author_id is considered a deleted-user orphan only when ALL of
    the following are true:
    1. The author_id appears in at least one published BlogPostSummary.
    2. No BlogAuthorDetailsModel exists for that author_id.
    3. No (non-deleted) UserSettingsModel exists for that author_id,
       meaning the user has been deleted from the platform.

    Authors who are still active (have a UserSettingsModel) but simply
    lack a BlogAuthorDetailsModel are NOT flagged — the runtime code
    handles that case automatically.
    """

    DATASTORE_UPDATES_ALLOWED = False
```

- **Line 67**: Class inherits from `base_jobs.JobBase` — Oppia's beam job framework
- **Lines 68-85**: Detailed docstring explaining:
  - Lines 68-70: What the job does (audits for orphaned authors)
  - Lines 72-78: The three conditions for an orphan:
    1. Author appears in published blog posts
    2. No BlogAuthorDetailsModel exists
    3. No UserSettingsModel exists (user is deleted)
  - Lines 80-82: Critical: active users without models are NOT flagged
- **Line 84**: `DATASTORE_UPDATES_ALLOWED = False` — **audit jobs never write to datastore**

### Run Method Signature (Lines 86-93)

```python
    def run(self) -> beam.PCollection[job_run_result.JobRunResult]:
        """Returns a PCollection of audit results.

        Returns:
            PCollection. A PCollection of JobRunResult instances reporting
            the orphaned author_ids and related counts.
        """
```

- **Line 86**: Method signature for the job's main execution
  - `self`: Instance method (has access to `self.pipeline`, `self.DATASTORE_UPDATES_ALLOWED`)
  - **Return type**: `beam.PCollection[job_run_result.JobRunResult]`
    - PCollection = Parallel Collection (Beam's distributed data structure)
    - Contains JobRunResult objects (output messages)
- **Lines 88-92**: Docstring explaining return value

### Step 1: Get Published Blog Post Author IDs (Lines 94-108)

```python
        # Step 1: Get all published BlogPostSummaryModel author_ids as
        # keyed pairs for CoGroupByKey.
        blog_post_author_pairs = (
            self.pipeline
            | 'Get all BlogPostSummaryModels'
            >> ndb_io.GetModels(
                blog_models.BlogPostSummaryModel.get_all(
                    include_deleted=False
                )
            )
            | 'Filter published summaries'
            >> beam.Filter(
                lambda model: model.published_on is not None
            )
            | 'Key by author_id from summaries'
            >> beam.Map(lambda model: (model.author_id, None))
            | 'Deduplicate author_id pairs'
            >> beam.Distinct()
        )
```

**Purpose**: Extract unique author_ids from published blog posts

**Line-by-line breakdown**:

1. **Lines 94-96**: Comment explaining the step
2. **Line 97**: `blog_post_author_pairs = (` — Start building a PCollection via chaining operations

3. **Line 98**: `self.pipeline` — Beam pipeline object (entry point for all Beam operations)

4. **Lines 99-102**: First transform: **read all BlogPostSummaryModels from datastore**
   - Line 100: `'Get all BlogPostSummaryModels'` — unique name for this pipeline stage (for debugging)
   - Line 101: `>>` — Beam's pipe operator (chain transforms)
   - Line 102-104: `ndb_io.GetModels()` — reads models from NDB datastore
   - Line 105-107: `blog_models.BlogPostSummaryModel.get_all(include_deleted=False)` — queries all non-deleted blog post summaries

5. **Lines 109-113**: Second transform: **filter to only published posts**
   - Line 110: `'Filter published summaries'` — stage name
   - Line 112: `beam.Filter()` — keep only items matching the condition
   - Line 113: Lambda checks `model.published_on is not None` (published posts have a date)

6. **Lines 114-116**: Third transform: **extract author_ids as (key, value) pairs**
   - Line 115: `'Key by author_id from summaries'` — stage name
   - Line 116: `beam.Map()` — transform each model
   - Line 116: Outputs tuple `(model.author_id, None)` — author_id is the key, value is ignored
   - Purpose: Prepare for `CoGroupByKey` which requires key-value pairs

7. **Lines 117-119**: Fourth transform: **deduplicate author_ids**
   - Line 118: `'Deduplicate author_id pairs'` — stage name
   - Line 119: `beam.Distinct()` — remove duplicates (if same author wrote multiple published posts, keep only one)

**Output**: PCollection of `(author_id, None)` tuples for authors with published posts

### Step 2: Get Existing BlogAuthorDetailsModel Author IDs (Lines 121-128)

```python
        # Step 2: Get all existing BlogAuthorDetailsModel author_ids as
        # keyed pairs.
        existing_author_detail_pairs = (
            self.pipeline
            | 'Get all BlogAuthorDetailsModels'
            >> ndb_io.GetModels(
                blog_models.BlogAuthorDetailsModel.get_all(
                    include_deleted=False
                )
            )
            | 'Key by author_id from details'
            >> beam.Map(lambda model: (model.author_id, None))
        )
```

**Purpose**: Extract unique author_ids that already have a BlogAuthorDetailsModel

**Line-by-line breakdown**:

1. **Lines 121-123**: Comment and variable assignment start
2. **Line 124**: `self.pipeline` — new pipeline branch (independent of Step 1)
3. **Lines 125-129**: Read all existing BlogAuthorDetailsModels
   - `ndb_io.GetModels()` with `BlogAuthorDetailsModel.get_all(include_deleted=False)`
   - Gets all non-deleted author detail models
4. **Lines 130-131**: Transform to `(author_id, None)` pairs
   - Extract only the author_id from each model

**Output**: PCollection of `(author_id, None)` tuples for authors who have a BlogAuthorDetailsModel

### Step 3: Get Existing Active UserSettingsModel IDs (Lines 133-144)

```python
        # Step 3: Get all existing (non-deleted) UserSettingsModel IDs
        # as keyed pairs. This lets us distinguish truly-deleted users
        # from active users who just lack a BlogAuthorDetailsModel.
        existing_user_pairs = (
            self.pipeline
            | 'Get all UserSettingsModels'
            >> ndb_io.GetModels(
                user_models.UserSettingsModel.get_all(
                    include_deleted=False
                )
            )
            | 'Key by user_id from user settings'
            >> beam.Map(lambda model: (model.id, None))
        )
```

**Purpose**: Extract unique user_ids of all active (non-deleted) users

**Line-by-line breakdown**:

1. **Lines 133-136**: Comment explaining the significance (distinguishes deleted vs. active)
2. **Line 137**: `existing_user_pairs = (` — third independent pipeline branch
3. **Line 138**: `self.pipeline` — new branch
4. **Lines 139-143**: Read all active UserSettingsModels
   - `UserSettingsModel.get_all(include_deleted=False)` — gets all non-deleted users
   - A user is "deleted" if their UserSettingsModel is gone
5. **Lines 144-145**: Transform to `(user_id, None)` pairs
   - Extract user_id (matches author_id in the user system)

**Output**: PCollection of `(user_id, None)` tuples for active users

### Step 4: Three-Way CoGroupByKey Join (Lines 147-170)

```python
        # Step 4: Three-way CoGroupByKey to find author_ids that:
        # (a) have published blog posts, (b) have no
        # BlogAuthorDetailsModel, AND (c) have no UserSettingsModel
        # (i.e. the user is truly deleted).
        orphaned_author_ids = (
            {
                'blog_post_authors': blog_post_author_pairs,
                'existing_author_details': existing_author_detail_pairs,
                'existing_users': existing_user_pairs,
            }
            | 'CoGroup by author_id' >> beam.CoGroupByKey()
            | 'Filter deleted-user orphaned author_ids'
            >> beam.Filter(
                lambda item: (
                    len(list(item[1]['blog_post_authors'])) > 0
                    and len(list(item[1]['existing_author_details'])) == 0
                    and len(list(item[1]['existing_users'])) == 0
                )
            )
            | 'Extract orphaned author_id'
            >> beam.Map(lambda item: item[0])
        )
```

**Purpose**: Find author_ids that satisfy ALL three orphan conditions

**Key Concept**: `CoGroupByKey()` is like SQL's FULL OUTER JOIN on a key

**Line-by-line breakdown**:

1. **Lines 147-151**: Comment explaining the three conditions

2. **Lines 152-161**: Create input dictionary for CoGroupByKey
   - `{` — dictionary mapping logical names to PCollections
   - `'blog_post_authors': blog_post_author_pairs` — authors with published posts
   - `'existing_author_details': existing_author_detail_pairs` — authors with detail models
   - `'existing_users': existing_user_pairs` — active users
   - Purpose: Label each stream so we can distinguish them after grouping

3. **Line 162**: `beam.CoGroupByKey()` — **the core operation**
   - Meaning: "Group by key across all three PCollections"
   - Output format: `(author_id, {'blog_post_authors': [...], 'existing_author_details': [...], 'existing_users': [...]})`
   - For each author_id, shows which groups it appears in
   - Example outputs:
     - `('author1', {'blog_post_authors': [None], 'existing_author_details': [None], 'existing_users': [None]})` — author has posts, details model, and is active → NOT orphaned
     - `('pid_deleted', {'blog_post_authors': [None], 'existing_author_details': [], 'existing_users': []})` — author has posts but no details model and is deleted → **ORPHANED**

4. **Lines 163-170**: Filter to keep only orphans
   - Line 165: `lambda item:` — processes each CoGroupByKey result
   - Line 166: `item[0]` = author_id key
   - Line 167: `item[1]` = grouped values dictionary

5. **Lines 166-169**: Three AND conditions (all must be true):
   - **Line 166**: `len(list(item[1]['blog_post_authors'])) > 0`
     - Convert to list and check length > 0
     - Means: author has at least one published post ✓
   - **Line 167**: `len(list(item[1]['existing_author_details'])) == 0`
     - Length == 0
     - Means: author has NO BlogAuthorDetailsModel ✓
   - **Line 168**: `len(list(item[1]['existing_users'])) == 0`
     - Length == 0
     - Means: author has NO UserSettingsModel (is deleted) ✓

6. **Lines 171-172**: Extract just the author_id key
   - `beam.Map(lambda item: item[0])` — discard the grouped values, keep only the key
   - Output: just author_ids that are orphans

**Output**: PCollection of orphaned author_ids (deleted users with published posts but no model)

### Step 5: Report Results (Lines 174-191)

```python
        # Step 5: Report results.
        orphaned_author_id_results = (
            orphaned_author_ids
            | 'Report orphaned author_ids'
            >> beam.Map(
                lambda author_id: job_run_result.JobRunResult.as_stdout(
                    f'ORPHANED AUTHOR ID: {author_id}'
                )
            )
        )

        orphaned_count_results = (
            orphaned_author_ids
            | 'Count orphaned author_ids'
            >> job_result_transforms.CountObjectsToJobRunResult(
                'ORPHANED AUTHOR IDS COUNT'
            )
        )

        total_blog_author_count_results = (
            blog_post_author_pairs
            | 'Count total blog post author_ids'
            >> job_result_transforms.CountObjectsToJobRunResult(
                'TOTAL BLOG POST AUTHOR IDS COUNT'
            )
        )

        return (
            orphaned_author_id_results,
            orphaned_count_results,
            total_blog_author_count_results,
        ) | 'Combine audit results' >> beam.Flatten()
```

**Purpose**: Format and output audit results

**Line-by-line breakdown**:

1. **Lines 174-181**: Report each orphaned author_id
   - Line 177-181: `beam.Map()` — transform each author_id to a result message
   - Line 179-180: `JobRunResult.as_stdout()` — creates output message
   - Output: "ORPHANED AUTHOR ID: author_id" for each orphan

2. **Lines 183-188**: Count total orphaned author_ids
   - Line 186-188: `CountObjectsToJobRunResult()` — custom transform to count and format
   - Output: "ORPHANED AUTHOR IDS COUNT SUCCESS: N"

3. **Lines 190-194**: Count total blog post authors
   - Uses `blog_post_author_pairs` (all authors with published posts, not just orphans)
   - Output: "TOTAL BLOG POST AUTHOR IDS COUNT SUCCESS: N"

4. **Lines 196-200**: Combine and return
   - Tuple of three result PCollections
   - `beam.Flatten()` — merge three separate streams into one
   - Return value: combined PCollection of JobRunResult messages

---

## Part 3: MigrateBlogAuthorDetailsForDeletedUsersJob Class (Lines 193-351)

### Class Definition & Docstring (Lines 193-211)

```python
class MigrateBlogAuthorDetailsForDeletedUsersJob(base_jobs.JobBase):
    """Job that creates BlogAuthorDetailsModel entries for deleted users.

    For each published blog post summary whose author_id has no
    corresponding BlogAuthorDetailsModel AND no UserSettingsModel (the
    user was deleted), this job creates a new BlogAuthorDetailsModel with
    a fallback display name and empty bio. This prevents 500 errors
    during blog homepage pagination when the rendering code encounters
    posts by deleted authors.

    Authors who still have a UserSettingsModel are skipped — the runtime
    code in blog_services.get_blog_author_details() handles creating
    their BlogAuthorDetailsModel using the user's actual display name.
    """

    DATASTORE_UPDATES_ALLOWED = True
```

- **Line 193**: Inherits from `base_jobs.JobBase`
- **Lines 194-210**: Docstring explaining:
  - Lines 194-196: What the job does (creates fallback models)
  - Lines 198-202: The problem it solves (prevents 500 errors)
  - Lines 204-206: Why active users are skipped
- **Line 209**: `DATASTORE_UPDATES_ALLOWED = True` — **this job DOES write to datastore** (unlike Audit)

### _create_author_details_model Method (Lines 211-261)

```python
    def _create_author_details_model(
        self,
        author_id: str,
    ) -> blog_models.BlogAuthorDetailsModel:
        """Creates a new BlogAuthorDetailsModel for the given author_id.

        When DATASTORE_UPDATES_ALLOWED is True, the model is persisted
        directly via model.put() inside the NDB context. This avoids
        using ndb_io.PutModels() which relies on WriteToDatastore and
        its ramp-up throttling that can exceed the dev server's
        gunicorn timeout.

        Args:
            author_id: str. The author_id of a deleted user who has no
                BlogAuthorDetailsModel.

        Returns:
            BlogAuthorDetailsModel. The newly created model instance.
        """
```

**Purpose**: Create and optionally persist a BlogAuthorDetailsModel for a deleted user

**Key Design**: Uses direct `model.put()` instead of `ndb_io.PutModels()` to avoid WriteToDatastore timeout

**Line-by-line breakdown**:

1. **Lines 211-215**: Method signature
   - `def _create_author_details_model(self, author_id: str):`
   - Takes instance (`self`) and author_id string
   - Returns BlogAuthorDetailsModel

2. **Lines 216-226**: Docstring explaining:
   - Why use direct `put()`: avoids WriteToDatastore ramp-up throttling that exceeds gunicorn 60s timeout
   - This is critical for dev server performance

### Generate Unique ID (Lines 228-233)

```python
        # Generate a unique instance ID for the new model, matching
        # the pattern used by
        # BlogAuthorDetailsModel.generate_new_instance_id.
        instance_id = utils.convert_to_hash(
            str(utils.get_random_int(base_models.RAND_RANGE)),
            base_models.ID_LENGTH,
        )
```

**Purpose**: Create a unique identifier for the new model

**Line-by-line breakdown**:

1. **Lines 228-230**: Comment explaining the ID generation strategy
2. **Line 231**: `instance_id = ` — store the generated ID
3. **Line 232-234**: Generate the ID
   - `utils.get_random_int(base_models.RAND_RANGE)` — generate random number in valid range
   - `str(...)` — convert to string
   - `utils.convert_to_hash(..., base_models.ID_LENGTH)` — hash to create fixed-length unique ID
   - Follows same pattern as BlogAuthorDetailsModel's auto-ID generation

### Create and Persist Model (Lines 236-249)

```python
        # Construct the NDB model inside a context and persist it
        # directly via model.put(), following the pattern established
        # in exp_migration_jobs._migrate_exploration_snapshot_model.
        with datastore_services.get_ndb_context():
            model = blog_models.BlogAuthorDetailsModel(
                id=instance_id,
                author_id=author_id,
                displayed_author_name=DELETED_USER_FALLBACK_AUTHOR_NAME,
                author_bio=DELETED_USER_FALLBACK_AUTHOR_BIO,
            )
            model.update_timestamps()
            if self.DATASTORE_UPDATES_ALLOWED:
                model.put()
        return model
```

**Purpose**: Construct the model, add timestamps, optionally persist it

**Key Design**: Persistence happens inside NDB context but only if `DATASTORE_UPDATES_ALLOWED`

**Line-by-line breakdown**:

1. **Lines 236-238**: Comment referencing the pattern from exp_migration_jobs

2. **Line 239**: `with datastore_services.get_ndb_context():`
   - Enter NDB context (required for all NDB model operations)
   - Context manager ensures proper cleanup

3. **Lines 240-245**: Create BlogAuthorDetailsModel instance
   - `id=instance_id` — unique identifier generated above
   - `author_id=author_id` — link to the deleted user
   - `displayed_author_name=DELETED_USER_FALLBACK_AUTHOR_NAME` — "Deleted User" fallback name
   - `author_bio=DELETED_USER_FALLBACK_AUTHOR_BIO` — empty bio for deleted users

4. **Line 246**: `model.update_timestamps()`
   - Sets creation and update timestamps on the model
   - Happens inside NDB context (required)

5. **Lines 247-248**: Conditional persistence
   - `if self.DATASTORE_UPDATES_ALLOWED:` — check the job flag
   - `model.put()` — persist to datastore (direct call, not via PutModels)
   - Only executes in Migrate job (flag is True)
   - Skipped in AuditMigrate job (flag is False)

6. **Line 249**: `return model`
   - Return the model whether or not it was persisted
   - Audit jobs return unpersisted models (for counting)
   - Migrate jobs return persisted models

**Key Innovation**: This pattern (direct `put()` inside map callback) avoids `WriteToDatastore`'s ramp-up throttling that can exceed the dev server's gunicorn 60-second timeout.

### Run Method (Lines 251-351)

The run method for Migration is identical to Audit in steps 1-4, but adds steps 5-6 for creation and persistence.

#### Steps 1-4: Same as Audit (Lines 260-328)

Same three-way join to find orphaned author_ids. See Audit job explanation above.

#### Step 5: Create BlogAuthorDetailsModel (Lines 330-335)

```python
        # Step 5: Create BlogAuthorDetailsModel for each orphaned
        # author_id.
        new_author_details_models = (
            orphaned_author_ids
            | 'Create BlogAuthorDetailsModel'
            >> beam.Map(self._create_author_details_model)
        )
```

**Purpose**: For each orphaned author_id, create a BlogAuthorDetailsModel

**Line-by-line breakdown**:

1. **Lines 330-332**: Comment explaining step
2. **Line 333**: `new_author_details_models = (` — start pipeline
3. **Line 334**: `orphaned_author_ids` — input from step 4 (stream of author_ids)
4. **Line 335**: `beam.Map(self._create_author_details_model)`
   - For each author_id, call `_create_author_details_model(author_id)`
   - Persists directly inside the callback if `DATASTORE_UPDATES_ALLOWED` is True
   - Output: stream of BlogAuthorDetailsModel instances

**Output**: PCollection of BlogAuthorDetailsModel objects (created and optionally persisted)

#### Step 6: Report Migration Results (Lines 337-351)

```python
        # Step 6: Report migration results.
        migration_results = (
            new_author_details_models
            | 'Report migrated author_ids'
            >> beam.Map(
                lambda model: job_run_result.JobRunResult.as_stdout(
                    f'MIGRATED AUTHOR ID: {model.author_id}'
                )
            )
        )

        migration_count_results = (
            new_author_details_models
            | 'Count migrated author_ids'
            >> job_result_transforms.CountObjectsToJobRunResult(
                'MIGRATED AUTHOR DETAILS COUNT'
            )
        )

        return (
            migration_results,
            migration_count_results,
        ) | 'Combine migration results' >> beam.Flatten()
```

**Purpose**: Format and output migration results

**Line-by-line breakdown**:

1. **Lines 337-346**: Report each created model
   - Extract `model.author_id` from each created model
   - Output message: "MIGRATED AUTHOR ID: author_id"

2. **Lines 348-353**: Count total migrated models
   - `CountObjectsToJobRunResult()` — count and format
   - Output: "MIGRATED AUTHOR DETAILS COUNT SUCCESS: N"

3. **Lines 355-358**: Combine and return results
   - Return tuple of both result streams
   - `beam.Flatten()` — merge into single output

---

## Part 4: AuditMigrateBlogAuthorDetailsForDeletedUsersJob Class (Lines 360-375)

```python
class AuditMigrateBlogAuthorDetailsForDeletedUsersJob(
    MigrateBlogAuthorDetailsForDeletedUsersJob
):
    """Audit job for MigrateBlogAuthorDetailsForDeletedUsersJob.

    This job performs the same detection logic as the migration job but
    does not write any data to the datastore. It is used to verify the
    migration plan before executing it.
    """

    DATASTORE_UPDATES_ALLOWED = False
```

**Purpose**: Test version of Migration job that doesn't write to datastore

**Line-by-line breakdown**:

1. **Line 360-362**: Class inherits from `MigrateBlogAuthorDetailsForDeletedUsersJob`
   - Inherits all methods: `run()`, `_create_author_details_model()`
   - Only difference: the flag below

2. **Lines 363-368**: Docstring explaining:
   - Same detection logic as Migration
   - Does NOT write to datastore
   - Used for dry-run / verification

3. **Line 371**: `DATASTORE_UPDATES_ALLOWED = False`
   - **This is the only code difference**
   - Inherited `_create_author_details_model()` checks this flag
   - When False, models are created but NOT persisted
   - Allows dry-run to see what would be migrated without actually writing

**Clever Design**: Single flag controls whether a shared codebase audits or migrates. No code duplication needed.

---

## Data Flow Diagram

```
AUDIT JOB                          MIGRATE JOB
═════════════                      ════════════

1. Blog Post Authors    1. Blog Post Authors
   └─ (author_id, None)    └─ (author_id, None)

2. Existing Author      2. Existing Author
   Details              Details
   └─ (author_id, None)    └─ (author_id, None)

3. Active Users         3. Active Users
   └─ (user_id, None)      └─ (user_id, None)

   ↓                       ↓
   3-way CoGroupByKey      3-way CoGroupByKey
   ↓                       ↓
4. Filter Orphans       4. Filter Orphans
   └─ orphaned_ids         └─ orphaned_ids
                           ↓
                        5. Create Models
                           _create_author_details_model()
                           └─ model.put() if ALLOWED
                           ↓
   ↓                    6. Output Results
5. Output Results
   └─ Report counts        └─ Report counts
   └─ NO datastore         └─ Models persisted
      writes
```

---

## Key Architectural Decisions

### 1. Three-Way Join via CoGroupByKey

**Why not query each collection separately?**
- Inefficient: would require 3 separate datastore queries
- Beam optimization: CoGroupByKey groups by key, showing which collections contain each key
- Filtering on group sizes is elegant and efficient

**The filter logic**:
```python
len(list(item[1]['blog_post_authors'])) > 0      # Has posts
and len(list(item[1]['existing_author_details'])) == 0  # No model
and len(list(item[1]['existing_users'])) == 0    # No user
```
This is the "three-way AND" that defines an orphan.

### 2. Direct model.put() Instead of ndb_io.PutModels()

**Problem with PutModels**:
- Uses Beam's `WriteToDatastore` transform
- Has `RampupThrottlingFn` with 5-minute ramp intervals
- Causes total execution time > 60 seconds
- Exceeds gunicorn's `--timeout 60` on dev server

**Solution**:
- Call `model.put()` directly inside `beam.Map()` callback
- Happens immediately for each model
- Follows pattern from `exp_migration_jobs._migrate_exploration_snapshot_model()`
- Much faster on dev server (avoids throttling overhead)

### 3. Flag-Based Audit/Migrate Split

**Problem**: Need two jobs with same logic (audit and migrate)

**Solution**: Single `DATASTORE_UPDATES_ALLOWED` flag
- Audit: flag = False → creates models but doesn't persist
- Migrate: flag = True → creates and persists models
- AuditMigrate: inherits, sets flag = False for dry-run

**Benefit**: No code duplication, single source of truth

### 4. Active Users NOT Flagged

**Important Design**: Active users without a BlogAuthorDetailsModel are NOT orphans

**Why?**
- Runtime code in `blog_services.get_blog_author_details()` auto-creates the model
- Uses user's real display name (not "Deleted User")
- Only truly-deleted users (no UserSettingsModel) need the fallback model

**Checking this condition**:
```python
and len(list(item[1]['existing_users'])) == 0    # MUST be empty
```
This ensures the UserSettingsModel doesn't exist.

---

## Execution Example

### Input Scenario

```
BlogPostSummaryModels (published):
├─ post1: author_id = "uid_active"
├─ post2: author_id = "pid_deleted"
└─ post3: author_id = "uid_deleted_no_model"

BlogAuthorDetailsModels:
└─ detail1: author_id = "uid_active"

UserSettingsModels:
└─ user1: id = "uid_active"
```

### After Step 4 (CoGroupByKey):

```
Key: "uid_active"
  blog_post_authors: [None]         ✓ has post
  existing_author_details: [None]   ✓ has model
  existing_users: [None]            ✓ is active
  → NOT orphaned (has model)

Key: "pid_deleted"
  blog_post_authors: [None]         ✓ has post
  existing_author_details: []       ✓ no model
  existing_users: []                ✓ is deleted
  → IS ORPHANED ✓✓✓

Key: "uid_deleted_no_model"
  blog_post_authors: [None]         ✓ has post
  existing_author_details: []       ✓ no model
  existing_users: []                ✓ is deleted
  → IS ORPHANED ✓✓✓
```

### Audit Job Output

```
ORPHANED AUTHOR ID: pid_deleted
ORPHANED AUTHOR ID: uid_deleted_no_model
ORPHANED AUTHOR IDS COUNT SUCCESS: 2
TOTAL BLOG POST AUTHOR IDS COUNT SUCCESS: 3
```

### Migrate Job Output

```
MIGRATED AUTHOR ID: pid_deleted
MIGRATED AUTHOR ID: uid_deleted_no_model
MIGRATED AUTHOR DETAILS COUNT SUCCESS: 2
```

Plus: 2 new BlogAuthorDetailsModel entries persisted to datastore with name="Deleted User"

---

## Summary

This beam job solution elegantly solves the deleted-user blog pagination problem:

1. **Audit**: Identifies orphaned author_ids (3-way CoGroupByKey)
2. **Migrate**: Creates fallback models for them (direct put() for speed)
3. **AuditMigrate**: Dry-run version (flag controls persistence)

The key innovations:
- Three-way join to precisely identify orphans
- Direct `model.put()` to avoid WriteToDatastore timeout
- Flag-based code reuse for audit/migrate split

Result: Blog homepage pagination works even when authors are deleted.
 
