# Bug Fix Documentation: Blog Pagination Error (Issue #23323)

## Overview

**Issue**: Blog pagination broken beyond page 2; pages 3–8 inaccessible with 500 Internal Server Error showing "User not found"

**URL**: `https://www.oppia.org/blog?page_num=3`

**Root Cause**: `get_blog_author_details()` violated Single Responsibility Principle by auto-creating models during read operations, which failed for deleted users.

---

## Table of Contents

1. [Problem Analysis](#1-problem-analysis)
2. [Root Cause Investigation](#2-root-cause-investigation)
3. [Solution Design](#3-solution-design)
4. [Implementation Details](#4-implementation-details)
5. [Beam Jobs for Legacy Data](#5-beam-jobs-for-legacy-data)
6. [Files Changed](#6-files-changed)
7. [Testing](#7-testing)
8. [Future Considerations](#8-future-considerations)

---

## 1. Problem Analysis

### Symptoms

- Blog homepage loads fine (page 1)
- Clicking "Next" to page 2 works
- Pages 3-8 return 500 Internal Server Error
- Error message: "User not found"

### Reproduction Steps

1. Navigate to `https://www.oppia.org/blog`
2. Scroll to pagination
3. Click to go to page 3 or higher
4. Observe 500 error

### Error Trace

```
BlogHomepageDataHandler.get()
  └─ _get_blog_card_summary_dicts_for_homepage()
      └─ blog_services.get_blog_author_details(author_id)
          └─ create_blog_author_details_model(user_id)  # ← Problem
              └─ user_services.get_user_settings(user_id, strict=True)
                  └─ EXCEPTION: "User not found"
```

---

## 2. Root Cause Investigation

### The Problematic Code (Before Fix)

```python
# core/domain/blog_services.py (BEFORE)
def get_blog_author_details(user_id: str) -> blog_domain.BlogAuthorDetails:
    author_model = blog_models.BlogAuthorDetailsModel.get_by_author(user_id)

    if author_model is None:
        # PROBLEM: Auto-creates model during read operation!
        create_blog_author_details_model(user_id)
        author_model = blog_models.BlogAuthorDetailsModel.get_by_author(user_id)

    if author_model is None:
        raise Exception('Unable to fetch author details for the given user.')

    return blog_domain.BlogAuthorDetails(...)
```

### Why It Fails for Deleted Users

1. **User deletes their account** → `UserSettingsModel` is deleted
2. **Author ID is pseudonymized** → Changes from `uid_xxx` to `pid_xxx`
3. **Blog posts remain** → `BlogPostSummaryModel.author_id = "pid_xxx"`
4. **`BlogAuthorDetailsModel` may not exist** → Was never created (lazy initialization)
5. **Read flow triggers creation** → `create_blog_author_details_model("pid_xxx")`
6. **Creation requires `UserSettingsModel`** → `get_user_settings("pid_xxx", strict=True)`
7. **User doesn't exist** → Exception: "User not found"

### Database State for Deleted Users

```
┌─────────────────────────────────────────────────────────────────┐
│ BlogPostSummaryModel                                            │
│   id: "blog_post_123"                                           │
│   author_id: "pid_abc123"  ← Pseudonymized                      │
│   title: "My Blog Post"                                         │
├─────────────────────────────────────────────────────────────────┤
│ BlogAuthorDetailsModel for "pid_abc123"                         │
│   → DOES NOT EXIST (was never created before deletion)          │
├─────────────────────────────────────────────────────────────────┤
│ UserSettingsModel for "pid_abc123"                              │
│   → DOES NOT EXIST (user deleted their account)                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Pages 1-2 Work But 3+ Fail

Pages 1-2 happen to contain blog posts from active users (with existing models).
Pages 3+ contain at least one blog post from a deleted user without a model.

---

## 3. Solution Design

### Design Principles Applied

1. **Single Responsibility Principle**: `get_*` methods only retrieve, never create
2. **Explicit over Implicit**: Model creation at well-defined write points only
3. **Graceful Degradation**: Read flows handle missing data with fallbacks

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SOLUTION ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MODEL CREATION (Single Point):                                     │
│  ──────────────────────────────                                     │
│  create_new_blog_post(author_id)                                    │
│    └─ Creates BlogAuthorDetailsModel if not exists                  │
│    └─ This is the ONLY place model is created                       │
│                                                                     │
│  READ FLOWS (Pure Retrieval):                                       │
│  ───────────────────────────                                        │
│  get_blog_author_details(user_id, strict=False)                     │
│    └─ Returns model or None                                         │
│    └─ NO creation, NO side effects                                  │
│    └─ Caller shows "Deleted User" fallback if None                  │
│                                                                     │
│  LEGACY DATA (Beam Job):                                            │
│  ─────────────────────                                              │
│  BackfillBlogAuthorDetailsModelJob                                  │
│    └─ Backfills missing models for active users                     │
│    └─ Skips deleted users (cannot backfill)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow After Fix

**Read Flow (Viewing Blog Posts)**:
```
User visits blog homepage
  └─ _get_blog_card_summary_dicts_for_homepage()
      └─ get_blog_author_details(author_id, strict=False)
          ├─ Model exists → Return author details
          └─ Model is None → Caller shows "Deleted User"
              └─ NO CRASH ✓
```

**Write Flow (Creating Blog Post)**:
```
User creates blog post
  └─ create_new_blog_post(author_id)
      ├─ Create BlogPostModel
      ├─ Create BlogPostRightsModel
      ├─ Create BlogPostSummaryModel
      └─ Ensure BlogAuthorDetailsModel exists
          └─ Model guaranteed for all new posts
```

---

## 4. Implementation Details

### 4.1 blog_services.py Changes

#### Change A: Pure Getter with `strict` Parameter

```python
# AFTER (with overloads for type safety)
@overload
def get_blog_author_details(user_id: str) -> blog_domain.BlogAuthorDetails: ...

@overload
def get_blog_author_details(
    user_id: str, *, strict: Literal[True]
) -> blog_domain.BlogAuthorDetails: ...

@overload
def get_blog_author_details(
    user_id: str, *, strict: Literal[False]
) -> Optional[blog_domain.BlogAuthorDetails]: ...

def get_blog_author_details(
    user_id: str, strict: bool = True
) -> Optional[blog_domain.BlogAuthorDetails]:
    """Returns the blog author details for the given user id.

    This is a pure retrieval function and does NOT create models.
    """
    author_model = blog_models.BlogAuthorDetailsModel.get_by_author(user_id)

    if author_model is None:
        if strict:
            raise Exception('Unable to fetch author details for the given user.')
        return None  # ← Caller decides what to do

    return blog_domain.BlogAuthorDetails(...)
```

#### Change B: Model Creation at Blog Post Creation

```python
def create_new_blog_post(author_id: str) -> blog_domain.BlogPost:
    blog_post_id = get_new_blog_post_id()
    new_blog_post_model = blog_models.BlogPostModel.create(blog_post_id, author_id)
    blog_models.BlogPostRightsModel.create(blog_post_id, author_id)
    new_blog_post = get_blog_post_from_model(new_blog_post_model)
    new_blog_post_summary_model = compute_summary_of_blog_post(new_blog_post)
    _save_blog_post_summary(new_blog_post_summary_model)

    # NEW: Ensure BlogAuthorDetailsModel exists
    if get_blog_author_details(author_id, strict=False) is None:
        create_blog_author_details_model(author_id)

    return new_blog_post
```

### 4.2 blog_homepage.py Changes

#### Change A: Constant for Fallback Display Name

```python
DELETED_USER_DISPLAY_NAME: Final = 'Deleted User'
```

#### Change B: Handle Missing Models in Read Flows

```python
def _get_blog_card_summary_dicts_for_homepage(summaries):
    for summary in summaries:
        # Use strict=False to handle deleted users gracefully
        author_details = blog_services.get_blog_author_details(
            summary_dict['author_id'], strict=False
        )

        # Fallback for missing models
        if author_details:
            displayed_author_name = author_details.displayed_author_name
        else:
            displayed_author_name = DELETED_USER_DISPLAY_NAME

        # Build response dict...
```

#### Change C: Same Pattern for Single Blog Post View

```python
class BlogPostDataHandler:
    def get(self, blog_post_url):
        author_details = blog_services.get_blog_author_details(
            blog_post.author_id, strict=False
        )
        if author_details:
            displayed_author_name = author_details.displayed_author_name
        else:
            displayed_author_name = DELETED_USER_DISPLAY_NAME
```

### 4.3 blog_dashboard.py Changes

#### Dashboard GET: No Model Creation

```python
@acl_decorators.can_access_blog_dashboard
def get(self) -> None:
    # Use strict=False - user may not have created any blog posts yet
    author_details_obj = blog_services.get_blog_author_details(
        self.user_id, strict=False
    )
    author_details = (
        author_details_obj.to_dict() if author_details_obj else None
    )
    # Frontend handles None by showing default values
```

---

## 5. Beam Jobs for Legacy Data

### 5.1 AuditBlogPostsWithMissingAuthorDetailsJob

**Purpose**: Audit-only job to identify which blog posts are missing author details.

**Output**:
- `ACTIVE_USERS_MISSING_AUTHOR_DETAILS`: Count of active users that CAN be backfilled
- `DELETED_USERS_MISSING_AUTHOR_DETAILS`: Count of deleted users that CANNOT be backfilled
- `CAN_BACKFILL: uid_xxx`: Specific user IDs that can be backfilled
- `CANNOT_BACKFILL_DELETED_USER: pid_xxx`: Deleted users (will show "Deleted User" in UI)

**Usage**: Run this first to assess the scope of missing data.

### 5.2 BackfillBlogAuthorDetailsModelJob

**Purpose**: Creates missing `BlogAuthorDetailsModel` for active users.

**Process**:
1. Get all unique `author_id` values from `BlogPostSummaryModel`
2. Get all `author_id` values from existing `BlogAuthorDetailsModel`
3. Find missing author IDs
4. Join with `UserSettingsModel` (filters out deleted users automatically)
5. Create `BlogAuthorDetailsModel` for each active user
6. Write to datastore

**Output**:
- `BLOG_AUTHOR_DETAILS_MODELS_CREATED`: Count of models created
- `DELETED_USERS_SKIPPED`: Count of deleted users that couldn't be backfilled

### 5.3 Job Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BEAM JOB FLOW                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BlogPostSummaryModel ──┐                                           │
│  (all blog posts)       │                                           │
│                         ├─► Extract unique author_ids               │
│                         │                                           │
│  BlogAuthorDetailsModel ─┘   │                                      │
│  (existing models)           │                                      │
│                              ▼                                      │
│                    Find missing author_ids                          │
│                              │                                      │
│              ┌───────────────┴───────────────┐                      │
│              ▼                               ▼                      │
│       Active Users                    Deleted Users                 │
│       (uid_*)                         (pid_*)                       │
│              │                               │                      │
│              ▼                               ▼                      │
│    Join with UserSettingsModel        SKIP (cannot backfill)        │
│              │                                                      │
│              ▼                                                      │
│    Create BlogAuthorDetailsModel                                    │
│    (username, bio from UserSettings)                                │
│              │                                                      │
│              ▼                                                      │
│    Write to Datastore                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Files Changed

### New Files Created

| File | Purpose |
|------|---------|
| `core/jobs/batch_jobs/blog_author_details_backfill_jobs.py` | Beam jobs for auditing and backfilling |
| `core/jobs/batch_jobs/blog_author_details_backfill_jobs_test.py` | Unit tests for Beam jobs |

### Modified Files

| File | Changes |
|------|---------|
| `core/domain/blog_services.py` | - Added `strict` parameter to `get_blog_author_details()` (pure getter)<br>- Ensure model exists in `create_new_blog_post()` |
| `core/controllers/blog_homepage.py` | - Added `DELETED_USER_DISPLAY_NAME` constant<br>- Use `strict=False` in all read flows<br>- Handle `None` with "Deleted User" fallback |
| `core/controllers/blog_dashboard.py` | - Use `strict=False` for dashboard GET<br>- No lazy model creation |
| `core/jobs/registry.py` | - Registered new Beam jobs |

---

## 7. Testing

### Unit Tests Added

**blog_author_details_backfill_jobs_test.py**:

| Test | Description |
|------|-------------|
| `test_audit_finds_active_user_missing_author_details` | Audit identifies active users without models |
| `test_audit_finds_deleted_user_missing_author_details` | Audit identifies deleted users (pid_*) |
| `test_audit_does_not_report_users_with_existing_author_details` | Audit skips users with models |
| `test_audit_with_mixed_scenarios` | Audit handles mixed cases |
| `test_backfill_creates_model_for_active_user` | Backfill creates model for active user |
| `test_backfill_skips_deleted_user` | Backfill skips deleted users |
| `test_backfill_skips_user_with_existing_author_details` | Backfill doesn't duplicate |
| `test_backfill_with_empty_database` | Backfill handles empty DB |

### Manual Testing Scenarios

| Scenario | Expected Result |
|----------|-----------------|
| View blog homepage pages 1-8 | All pages load, deleted users show "Deleted User" |
| View single blog post by deleted user | Shows "Deleted User" as author |
| New user visits blog dashboard | Dashboard loads (author_details = None) |
| New user creates blog post | Model created, author details populated |
| Existing user creates another blog post | No duplicate model created |

---

## 8. Future Considerations

### Deployment Steps

1. **Deploy code changes first** - This fixes the 500 error immediately
2. **Run audit job** - Assess scope of missing data
3. **Run backfill job** - Create models for active users with missing data
4. **Deleted users** - Will continue showing "Deleted User" (by design)

### Potential Improvements

1. **Frontend handling**: Update Angular components to show appropriate UI when `author_details` is `None` on dashboard
2. **Monitoring**: Add metrics for missing author details occurrences
3. **Documentation**: Update developer docs about `BlogAuthorDetailsModel` lifecycle

### Why Not Delete Orphan Blog Posts?

Blog posts from deleted users are valuable content and should remain accessible.
The wipeout policy is `LOCALLY_PSEUDONYMIZE`, not `DELETE`, meaning:
- Content is preserved
- Author identity is anonymized
- "Deleted User" is the appropriate display

---

## Summary

| Aspect | Before | After |
|--------|--------|-------|
| `get_blog_author_details()` | Auto-creates model (side effect) | Pure getter with `strict` param |
| Model creation point | Multiple (lazy, on read) | Single (on blog post creation) |
| Read flows | Crash for deleted users | Graceful "Deleted User" fallback |
| Legacy data | Causes 500 errors | Handled by Beam backfill job |
| Single Responsibility | ❌ Violated | ✅ Followed |
| Pages 3+ on blog | ❌ 500 Error | ✅ Works |
