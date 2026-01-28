# Blog Author Details Model Creation Strategy

## The Scenario: First-Time User Experience

### Timeline of Events

**New User Journey**:
```
1. User registers on Oppia
   └─ UserSettingsModel created ✓
   └─ BlogAuthorDetailsModel NOT created ✗

2. User browses blog homepage
   └─ Tries to view blog posts
   └─ Code calls get_blog_author_details(user_id, strict=True) [DEFAULT]
   └─ Model doesn't exist
   └─ CRASH! ❌ (if user is an author somewhere)

3. User starts writing a blog post
   └─ create_new_blog_post() is called
   └─ BlogAuthorDetailsModel gets created here ✓

4. User visits dashboard
   └─ get_blog_author_details(user_id, strict=False)
   └─ Model exists now ✓
```

### The Loading Time Problem

When a new user visits the blog page for the first time:

**Current Flow** (if model doesn't exist):
```
User clicks blog link
  └─ _get_blog_card_summary_dicts_for_homepage() runs
      └─ For EACH blog post:
          └─ get_blog_author_details(author_id) [STRICT=TRUE by default]
          └─ If author_id belongs to deleted user
             └─ Exception raised
             └─ 500 error + CRASH
          └─ If author_id belongs to new author who never posted
             └─ Same crash
```

**Problem**: Every blog post requires a model to exist. If even ONE post's author is missing a model → entire page crashes.

---

## The Real Issue: Model Creation Timing

### Current Problems

#### Problem 1: `get_blog_author_details()` Still Crashes

Line 85 in [blog_homepage.py](core/controllers/blog_homepage.py#L85):
```python
author_details = blog_services.get_blog_author_details(
    summary_dict['author_id']  # NO strict=False! Uses default strict=True
)
```

**This should be**:
```python
author_details = blog_services.get_blog_author_details(
    summary_dict['author_id'], strict=False  # ← Safe for missing models
)
```

#### Problem 2: What Counts as a "Blog Author"?

Users can be blog authors in these scenarios:
1. **They created a blog post** → BlogAuthorDetailsModel should exist
2. **They are mentioned as author in someone else's post** → Should they have a model?
3. **They deleted their account** → Model can't be created (pseudonymized)
4. **They never created a post but appear in data** → Legacy/orphaned case

---

## Recommended Solution

### Phase 1: Fix Immediate Crashes (Read Safety)

**Change all read flows to use `strict=False`**:

```python
# In _get_blog_card_summary_dicts_for_homepage()
author_details = blog_services.get_blog_author_details(
    summary_dict['author_id'], strict=False  # Safe default for reads
)

# Handle None case
if author_details is not None:
    displayed_author_name = author_details.displayed_author_name
else:
    displayed_author_name = 'Deleted User'  # or 'Unknown Author'
```

**Affected files**:
- [blog_homepage.py](core/controllers/blog_homepage.py) - Lines 85, and similar in other handlers
- [blog_dashboard.py](core/controllers/blog_dashboard.py) - Dashboard GET
- Any other controller accessing blog author details

### Phase 2: Strategic Model Creation

**Option A: Create on First Dashboard Visit** (PROACTIVE)
```python
# core/controllers/blog_dashboard.py
@acl_decorators.can_access_blog_dashboard
def get(self) -> None:
    author_details = blog_services.get_blog_author_details(
        self.user_id, strict=False
    )
    
    # First time visiting dashboard? Create the model
    if author_details is None:
        try:
            blog_services.create_blog_author_details_model(self.user_id)
            author_details = blog_services.get_blog_author_details(
                self.user_id
            )
        except Exception:
            # User deleted their account or other issue
            pass
```

**Pros**:
- User model exists before they write their first post
- Dashboard loads faster on subsequent visits
- Better UX for new authors

**Cons**:
- Violates Single Responsibility Principle
- Creates models even for users who never write blogs
- Database bloat with unused models

**Option B: Keep Creation at Post Time** (CURRENT - RECOMMENDED)
```python
# core/domain/blog_services.py - create_new_blog_post()
def create_new_blog_post(...):
    # ... create blog post model, rights, summary ...
    
    # Ensure author has a details model
    if get_blog_author_details(author_id, strict=False) is None:
        create_blog_author_details_model(author_id)
```

**Pros**:
- Single Responsibility: Models created at write time only
- No database bloat
- Cleaner architecture

**Cons**:
- First post creation takes slightly longer (one extra model creation)
- Dashboard shows `None` for new users who haven't posted yet

**Option C: Lazy Creation with Caching** (HYBRID)
```python
# core/domain/blog_services.py
@functools.lru_cache(maxsize=1024)
def get_or_create_blog_author_details(user_id: str):
    author_details = get_blog_author_details(user_id, strict=False)
    
    if author_details is None:
        # Only create once per user per application lifetime
        create_blog_author_details_model(user_id)
        author_details = get_blog_author_details(user_id, strict=False)
    
    return author_details
```

**Pros**:
- Model created on first access (any handler)
- Cached in memory for performance
- Single Responsibility with pragmatism

**Cons**:
- Cache invalidation complexity
- Still creates models speculatively

---

## Performance Analysis

### Loading Time Impact

| Scenario | Option A | Option B | Option C | Current (Crashes) |
|----------|----------|----------|----------|-------------------|
| New user views blog page | ✓ Fast | ✓ Fast | ✓ Fast | ❌ 500 Error |
| New user views dashboard | ✓ +1 model write | ✓ Fast | ✓ +1 model write | ✓ Fast |
| New user creates post | ✓ Fast | ✓ +1 model write | ✓ Fast | ✓ +1 model write |
| 100 blog posts loaded | ✓ Fast | ✓ Fast | ✓ Fast | ❌ Crashes if 1+ missing |

### Database Calls (per blog page load with 10 posts)

**Before Fix**:
```
get_blog_post_summaries()        1 query
get_user_settings() × 10          10 queries
get_blog_author_details() × 10    10 queries (+ failures for missing)
Total:                            21 queries (possibly crashes)
```

**After Fix with Option B (Recommended)**:
```
get_blog_post_summaries()        1 query
get_user_settings() × 10          10 queries
get_blog_author_details(strict=False) × 10    10 queries
Total:                            21 queries ✓ (safe, no crashes)
```

---

## Recommendation

### Adopt Option B: Create at Blog Post Time

**Rationale**:
1. **Single Responsibility**: Clear, predictable model lifecycle
2. **No speculation**: Don't create models that might never be used
3. **Measurable**: Can track exactly when models are created (on publish)
4. **Safe reads**: All read flows use `strict=False` with fallback
5. **Legacy handled**: Beam jobs backfill for existing orphaned posts

### Implementation Steps

1. **Immediate**: Fix all read flows to use `strict=False`
   - [blog_homepage.py](core/controllers/blog_homepage.py#L85)
   - [blog_dashboard.py](core/controllers/blog_dashboard.py)
   - Any other blog-related handler

2. **Verify**: Ensure `create_new_blog_post()` creates model on first post
   - [blog_services.py](core/domain/blog_services.py)

3. **Backfill**: Run Beam jobs for orphaned data
   - `BackfillBlogAuthorDetailsModelJob` creates models for users with existing posts
   - Deleted users remain as "Deleted User" (acceptable)

4. **Test**: All new/existing users can browse blog without crashes
   - Blog homepage loads for all users
   - New users can create first blog post
   - Dashboard works for everyone

---

## FAQ

### Q: What about users who co-authored a post?
**A**: Only the `author_id` field gets a model. Co-authors don't need dedicated models; they're attributes of the post.

### Q: Should dashboard auto-create the model?
**A**: No. Dashboard should handle `None` gracefully. Model is created when user publishes first post.

### Q: What's the performance impact?
**A**: Negligible. One extra model creation per user on first post (batched with post creation). Read flows don't change performance.

### Q: How long should loading time be?
**A**: Blog homepage should load in <500ms with 10 posts (assumes caching and indexing are optimized).
- If loading is slow even with fix, investigate: blog post search queries, user settings lookups, or database indexing.

### Q: Should we delete models for users who delete their blogs?
**A**: No. Keep the model but mark `is_active = False` if needed. This preserves historical data.

---

## Summary

| Aspect | Current | After Fix |
|--------|---------|-----------|
| First-time blog page visit | ❌ Crashes | ✓ Safe (models missing shown as "Deleted User") |
| Loading time | N/A (crashes) | ~500ms for 10 posts |
| Database calls | 21+ (+ errors) | 21 (no errors) |
| Model creation | Lazy (buggy) | At post creation (clean) |
| SRP Compliance | ❌ No | ✓ Yes |

**Next Steps**: Update [blog_homepage.py](core/controllers/blog_homepage.py#L85) to use `strict=False`.
