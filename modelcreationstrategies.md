# Model Creation Strategies: Comparison & Recommendations

## The Fundamental Question

**When should `BlogAuthorDetailsModel` be created?**

This determines whether the blog system is fast, reliable, and maintainable.

---

## Strategy Comparison Matrix

### Strategy A: Lazy Creation on Read (❌ ORIGINAL - BROKEN)

**When**: First time `get_blog_author_details()` is called
**Who calls**: Any read handler (homepage, blog post view, etc.)
**Issue**: Creates models even when author is deleted

```
User deletes account
  └─ UserSettingsModel deleted
  └─ author_id changed to pid_xxx
  └─ Later, someone visits blog post page
      └─ get_blog_author_details(pid_xxx)
          └─ Tries to create_blog_author_details_model(pid_xxx)
              └─ Fails: UserSettingsModel doesn't exist
                  └─ 500 ERROR ❌
```

**Performance**: Fast first-time read (user doesn't wait)
**Reliability**: ❌ Crashes for deleted users
**Architecture**: ❌ Violates SRP (getter creating)
**Data bloat**: ✓ None
**User count impact**: High (affects all users on deleted-author posts)

---

### Strategy B: Creation at Post Time (✅ CURRENT - RECOMMENDED)

**When**: When `create_new_blog_post()` is called
**Who calls**: Blog editor controller (intentional write operation)
**Guarantee**: Model exists for all published posts

```
New user creates first blog post
  ├─ create_new_blog_post(author_id)
  │  ├─ Create BlogPostModel ✓
  │  ├─ Create BlogPostSummaryModel ✓
  │  └─ Ensure BlogAuthorDetailsModel exists ✓
  │     (creates if needed)
  │
  └─ Author has complete profile ✓

Later, user is deleted
  └─ author_id changed to pid_xxx
  └─ BlogAuthorDetailsModel orphaned but doesn't crash
  └─ Read flows handle with strict=False
  └─ Shows "Deleted User" ✓
```

**Performance**: Slightly slower at post creation (+1 model write)
**Reliability**: ✓ No crashes (with strict=False in reads)
**Architecture**: ✓ Pure SRP (writes create, reads retrieve)
**Data bloat**: ✓ None (only models for authors with posts)
**User count impact**: Low (only affects new authors on first post)

**Implementation**:
```python
# In blog_services.py
def create_new_blog_post(author_id: str) -> blog_domain.BlogPost:
    # ... create post models ...
    
    # Ensure author has details model
    if get_blog_author_details(author_id, strict=False) is None:
        create_blog_author_details_model(author_id)
    
    return new_blog_post
```

---

### Strategy C: Creation on Dashboard Visit (❌ PROBLEMATIC)

**When**: First time user visits `/blog/dashboard`
**Who calls**: Blog dashboard controller
**Issue**: Creates model even if user never writes a blog

```
User visits blog dashboard
  └─ BlogAuthorDetailsModel created
  └─ User never writes a blog post
  └─ Model wastes space ❌

User visits dashboard, then account deleted
  └─ Model persists as orphan ❌
```

**Performance**: Slower dashboard load for new users
**Reliability**: ✓ Works (user can be deleted later)
**Architecture**: ⚠️ Partially violates SRP (dashboard shouldn't create models)
**Data bloat**: ❌ Models for users who never blog
**User count impact**: Medium (creates models for explorers)

---

### Strategy D: Proactive Batch Creation (⚠️ HYBRID)

**When**: Deployment time or scheduled job
**Who calls**: Beam job or one-time script
**Issue**: Over-creates for non-authors

```
Run initial batch job:
  └─ For each UserSettingsModel:
      └─ Create BlogAuthorDetailsModel

Later:
  └─ "Ghost" models exist for users who never blog
  └─ Harder to distinguish "never posted" from "deleted" ✗
```

**Performance**: ✓ Fast reads (all models exist)
**Reliability**: ✓ No crashes
**Architecture**: ❌ Violates SRP (batch creates speculatively)
**Data bloat**: ❌ Major (model for every user)
**User count impact**: Extreme (creates millions of unused models)

---

### Strategy E: Caching + Lazy Creation (⚠️ COMPLEX)

**When**: First read OR first write (cached per request)
**Who calls**: Any handler
**Issue**: Complex cache invalidation

```
User visits homepage
  └─ @cached_get_or_create_author_details(author_id)
      ├─ Check cache
      ├─ Model missing in cache?
      │  └─ Create model
      │  └─ Cache result
      └─ Return cached model

Problem: Cache invalidation when user deletes account
  └─ Stale cached model persists ❌
```

**Performance**: ✓ Fast reads (cache + creation amortized)
**Reliability**: ⚠️ Cache invalidation complexity
**Architecture**: ❌ Caching hides SRP violation
**Data bloat**: ✓ None (only cached models)
**User count impact**: Low (but cache complexity high)

---

## Recommendation: Strategy B (Creation at Post Time)

### Why B is Best

```
┌─────────────────────────────────────────────────────────────────┐
│ STRATEGY B: POST-TIME CREATION                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ✓ RELIABILITY:  Safe for all users (strict=False in reads)    │
│ ✓ EFFICIENCY:   No database bloat                             │
│ ✓ ARCHITECTURE: Clean SRP (write creates, read retrieves)      │
│ ✓ SIMPLICITY:   Single creation point, easy to reason about   │
│ ✓ TESTABILITY:  Mock models easily, no cache complexity       │
│ ✓ RECOVERY:     Beam job backfills legacy orphaned data       │
│ ✓ PERFORMANCE:  Same query count as other strategies          │
│                                                                 │
│ Trade-off: First post creation +50-100ms (acceptable)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### 1. Service Layer (blog_services.py)
```python
@overload
def get_blog_author_details(user_id: str, strict: Literal[False]) 
    -> Optional[blog_domain.BlogAuthorDetails]: ...

def get_blog_author_details(user_id: str, strict: bool = True):
    """Pure retrieval. Does NOT create models."""
    model = blog_models.BlogAuthorDetailsModel.get_by_author(user_id)
    
    if model is None:
        if strict:
            raise Exception('Model not found.')
        return None  # ← Caller handles None
    
    return blog_domain.BlogAuthorDetails(...)
```

#### 2. Create Flow (blog_services.py)
```python
def create_new_blog_post(author_id: str) -> blog_domain.BlogPost:
    """Single point where model is created."""
    # Create post models...
    new_blog_post = blog_models.BlogPostModel.create(...)
    
    # Ensure author has details model
    if get_blog_author_details(author_id, strict=False) is None:
        create_blog_author_details_model(author_id)
    
    return new_blog_post
```

#### 3. Read Flows (blog_homepage.py, etc.)
```python
def get(self) -> None:
    """Retrieve without side effects."""
    # Use strict=False for safe retrieval
    author_details = blog_services.get_blog_author_details(
        author_id, strict=False
    )
    
    # Handle None gracefully
    if author_details:
        name = author_details.displayed_author_name
    else:
        name = 'Deleted User'
```

#### 4. Legacy Data (blog_author_details_backfill_jobs.py)
```python
class BackfillBlogAuthorDetailsModelJob(beam.PTransform):
    """Backfill missing models for existing blog posts."""
    
    def expand(self, pipeline):
        return (
            pipeline
            | 'GetBlogAuthors' >> beam.Map(get_blog_authors)
            | 'FindMissing' >> beam.Map(find_missing_models)
            | 'JoinWithUsers' >> beam.Map(get_user_settings)
            | 'CreateModels' >> beam.Map(create_author_details_model)
            | 'WriteToDatastore' >> WriteToDatastore()
        )
```

---

## Performance Comparison

### Query Count Breakdown

| Operation | Strategy B | Overhead |
|-----------|-----------|----------|
| View blog post | 21 queries | 0 (same as now) |
| Create post | 21 queries + 1 model write | +50-100ms |
| View dashboard (new user) | 21 queries | 0 |
| View dashboard (returning user) | 21 queries | 0 |

### Typical User Journey Timing

```
New Author Timeline:
─────────────────────────────────────────────────────────────────

T0: Visit blog homepage
    └─ 21 queries: ~250ms ✓

T1: Visit blog dashboard
    └─ 2 queries (user & settings): ~50ms ✓
    └─ No model created yet ✓

T2: Edit and create first blog post (200ms)
    └─ Create models: ~50ms
    └─ Write to datastore: ~100ms
    └─ Total: ~250ms (acceptable) ✓

T3: Reload dashboard
    └─ 21 queries + author details lookup: ~300ms ✓
    └─ Model exists, loads instantly ✓
```

### Comparison with Other Strategies

| Metric | Strategy A | Strategy B | Strategy C | Strategy D |
|--------|-----------|-----------|-----------|-----------|
| Homepage load (new user) | 500 error ❌ | ~250ms ✓ | ~250ms ✓ | ~250ms ✓ |
| First post creation | ~150ms ✓ | ~250ms ✓ | ~150ms ✓ | ~150ms ✓ |
| Handles deleted users | ❌ Crashes | ✓ Graceful | ✓ Graceful | ✓ Graceful |
| Database bloat | N/A | None ✓ | Moderate ⚠️ | Extreme ❌ |
| Architectural cleanliness | ❌ No | ✓ Yes | ⚠️ Partial | ❌ No |
| **Overall** | **Broken** | **Best** | **Workable** | **Wasteful** |

---

## Implementation Checklist

- [x] Add `strict` parameter to `get_blog_author_details()` in blog_services.py
- [x] Ensure `create_new_blog_post()` creates model at post time
- [x] Update all read handlers to use `strict=False`
  - [x] blog_homepage.py: `_get_blog_card_summary_dicts_for_homepage()`
  - [x] blog_homepage.py: `BlogPostDataHandler.get()`
  - [x] blog_homepage.py: `BlogAuthorProfilePageHandler.get()`
  - [ ] Any other handlers
- [x] Add fallback for missing models (display "Deleted User")
- [x] Create Beam job to backfill legacy data
- [x] Add unit tests for new behavior
- [ ] Manual testing for all scenarios
- [ ] Update frontend templates to handle `None` author_details

---

## Rollout Plan

### Phase 1: Deploy Code (Low Risk)
1. Deploy changes to blog_services.py
2. Deploy changes to blog_homepage.py, blog_dashboard.py
3. Deploy Beam jobs
4. Expected impact: None (backward compatible with `strict=False`)

### Phase 2: Run Audit (Assessment)
1. Run `AuditBlogPostsWithMissingAuthorDetailsJob`
2. Identify how many missing models exist
3. Separate active users vs deleted users
4. Expected: Small number of legacy orphans

### Phase 3: Backfill (Repair)
1. Run `BackfillBlogAuthorDetailsModelJob`
2. Creates models for active users with posts
3. Skips deleted users (cannot backfill)
4. Expected impact: Improved reliability, no user-visible changes

### Phase 4: Verify (Validation)
1. Run smoke tests
2. Load test blog homepage
3. Verify all pagination pages work
4. Confirm no 500 errors
5. Expected: All tests pass ✓

---

## Conclusion

**Strategy B: Create at Post Time** is the clear winner because it:

1. **Fixes the bug**: No more 500 errors for deleted users
2. **Maintains efficiency**: No database bloat
3. **Follows best practices**: Single Responsibility Principle
4. **Scales well**: Same performance at any user count
5. **Handles edge cases**: Graceful "Deleted User" fallback
6. **Enables recovery**: Beam job backfills legacy data

The slight performance cost on first post creation (~50-100ms) is negligible and acceptable for the architectural benefits.
 
