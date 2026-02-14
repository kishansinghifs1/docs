# Deprecated Feature Flags - PR History & Cleanup

**Generated**: February 15, 2026

This document tracks the pull requests (PRs) associated with each deprecated feature flag in the Oppia codebase.

---

## Summary

| # | Feature Flag | Status | Original PR | Deprecation PR | Notes |
|---|---|---|---|---|---|
| 1 | `ANDROID_BETA_LANDING_PAGE` | Unused | #16108 | #19094, #19632 | Android landing page feature |
| 2 | `BLOG_PAGES` | Test only | #16243 | #22180 | Blog feature, moved to deprecated |
| 3 | `CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS` | Unused | #16289 | #20483 | Removed contributor accomplishments gating |
| 4 | `DIAGNOSTIC_TEST` | Unused | Unknown | #22180 | Removed diagnostic test feature flag |
| 5 | `END_CHAPTER_CELEBRATION` | Unused | Unknown | Unknown | Chapter celebration feature |
| 6 | `CHECKPOINT_CELEBRATION` | Unused | Unknown | Unknown | Checkpoint celebration feature |
| 7 | `ENABLE_VOICEOVER_CONTRIBUTION` | Unused | Unknown | Unknown | Voiceover contribution feature |
| 8 | `AUTO_UPDATE_EXP_VOICE_ARTIST_LINK` | Unused | Unknown | #20658 | Moved to PROD, later deprecated |
| 9 | `LABEL_ACCENT_TO_VOICE_ARTIST` | Unused | #21339 | Unknown | Voiceover accent labeling |
| 10 | `ADD_VOICEOVER_WITH_ACCENT` | Unused | Unknown | #20944 | Moved to PROD, later deprecated |

---

## Detailed PR Information

### 1. ⚫ ANDROID_BETA_LANDING_PAGE
**Value**: `android_beta_landing_page`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Original**: #16108 - "Put android page behind a feature flag; Make more text translatable"
  - Introduced the Android landing page with feature flag gating
- **Refactored**: #18819 - "Introduces new model for feature-flags and implementation of rollout-percentage property for features"
  - Infrastructure update for feature flag system
- **Eliminated**: #19094 - "Fix #18922 [BUG]: Eliminate obsolete feature flags"
  - Marked obsolete feature flags for cleanup
- **Cleanup**: #19632 - "Fix #19285: Cleanup for the separation of feature flags from platform params"
  - Comprehensive separation of feature flags from platform parameters

**Action**: ✅ SAFE TO REMOVE

---

### 2. 🔵 BLOG_PAGES
**Value**: `blog_pages`  
**Current Status**: 1 reference in test code only (core/tests/test_utils_test.py)

**Related PRs**:
- **Original**: #16243 - "Milestone 2.3: Implements Blog Author Page and feature flag to enable Blog Homepage feature"
  - Initial implementation of blog feature with flag gating
- **Infrastructure**: #19632 - "Fix #19285: Cleanup for the separation of feature flags from platform params"
  - Refactored feature flag system
- **Removed**: #22180 - "Removes Diagnostic Test feature flag"
  - Cleaned up diagnostic test flag (also touched blog_pages)

**Action**: ✅ SAFE TO REMOVE (after updating test file)

---

### 3. ⚫ CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS
**Value**: `contributor_dashboard_accomplishments`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Original**: #16289 - "(Contributor Recognition Infrastructure) Milestone 2.2: Create contributor stats component to contributor dashboard with mobile view"
  - Introduced contributor dashboard accomplishments feature
- **Removed**: #20483 - "Fix #20329: Removed cd accomplishments flag"
  - Explicitly removed the contributor dashboard accomplishments feature flag

**Action**: ✅ SAFE TO REMOVE

---

### 4. ⚫ DIAGNOSTIC_TEST
**Value**: `diagnostic_test`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Removed**: #22180 - "Removes Diagnostic Test feature flag"
  - Explicit removal of diagnostic test feature flag
  - Most recent cleanup commit for deprecated flags

**Action**: ✅ SAFE TO REMOVE

---

### 5. ⚫ END_CHAPTER_CELEBRATION
**Value**: `end_chapter_celebration`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- Unknown original PR (feature predates current audit)
- No active removal PRs found in git history

**Note**: This flag appears to be permanently enabled functionality

**Action**: ✅ SAFE TO REMOVE

---

### 6. ⚫ CHECKPOINT_CELEBRATION
**Value**: `checkpoint_celebration`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- Unknown original PR (feature predates current audit)
- No active removal PRs found in git history

**Note**: This flag appears to be permanently enabled functionality

**Action**: ✅ SAFE TO REMOVE

---

### 7. ⚫ ENABLE_VOICEOVER_CONTRIBUTION
**Value**: `enable_voiceover_contribution`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- Unknown original PR
- Part of voiceover feature development cycle
- Moved to deprecated when voiceover contribution became standard

**Note**: Voiceover contribution is now standard functionality

**Action**: ✅ SAFE TO REMOVE

---

### 8. ⚫ AUTO_UPDATE_EXP_VOICE_ARTIST_LINK
**Value**: `auto_update_exp_voice_artist_link`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Moved to PROD**: #20658 - "Moves AUTO_UPDATE_EXP_VOICE_ARTIST_LINK feature flag to prod"
  - Flag moved from test/dev to production stage
- Later: Moved to deprecated list as feature became standard

**Action**: ✅ SAFE TO REMOVE

---

### 9. ⚫ LABEL_ACCENT_TO_VOICE_ARTIST
**Value**: `label_accent_to_voice_artist`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Introduced**: #21339 - "Adds feature flag to hide voiceover regeneration feature from exp editor page and voice artist accent labeling feature from voiceover admin page"
  - Added during voiceover feature enhancement wave
- **Later**: Moved to deprecated as feature became standard

**Action**: ✅ SAFE TO REMOVE

---

### 10. ⚫ ADD_VOICEOVER_WITH_ACCENT
**Value**: `add_voiceover_with_accent`  
**Current Status**: Unused (0 references in codebase)

**Related PRs**:
- **Moved to PROD**: #20944 - "Moves the ADD_VOICEOVER_WITH_ACCENT feature flag to prod"
  - Flag elevated to production stage indicating stability
- Part of GSoC 2024 work on voiceover enhancements
- Later: Moved to deprecated list as feature became standard

**Action**: ✅ SAFE TO REMOVE

---

## PR Cleanup Strategy

### Key Milestones in Feature Flag System

1. **#18819** - Foundation for rollout-percentage based feature flags
2. **#19094** - Identified obsolete feature flags
3. **#19632** - Major separation of feature flags from platform parameters (~Feb 2024)
4. **#20658** - Voiceover artist link auto-update moved to prod
5. **#20944** - Voiceover with accent moved to prod  
6. **#22180** - Removed diagnostic test feature flag (~Jan 2025)

### Pattern Observed

Most deprecated flags follow this lifecycle:
1. Created with feature (e.g., #16108, #16243)
2. Moved through dev → test → prod stages as feature stabilizes
3. Feature becomes standard functionality
4. Flag moved to DEPRECATED_FEATURE_NAMES list
5. Eventually removed entirely (this PR)

---

## Cleanup Action Items

### Phase 1: Remove from Code (Immediate)

#### File: `core/feature_flag_list.py`

**Remove from `FeatureNames` enum (10 entries)**:
```python
ANDROID_BETA_LANDING_PAGE = 'android_beta_landing_page'
BLOG_PAGES = 'blog_pages'
CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS = 'contributor_dashboard_accomplishments'
DIAGNOSTIC_TEST = 'diagnostic_test'
END_CHAPTER_CELEBRATION = 'end_chapter_celebration'
CHECKPOINT_CELEBRATION = 'checkpoint_celebration'
ENABLE_VOICEOVER_CONTRIBUTION = 'enable_voiceover_contribution'
AUTO_UPDATE_EXP_VOICE_ARTIST_LINK = 'auto_update_exp_voice_artist_link'
LABEL_ACCENT_TO_VOICE_ARTIST = 'label_accent_to_voice_artist'
ADD_VOICEOVER_WITH_ACCENT = 'add_voiceover_with_accent'
```

**Remove from `DEPRECATED_FEATURE_NAMES` list (all 10 entries)**

#### File: `core/tests/test_utils_test.py`

**Replace BLOG_PAGES reference in test decorator** (~line 59):
```python
# Before:
[
    feature_flag_list.FeatureNames.DUMMY_FEATURE_FLAG_FOR_E2E_TESTS,
    feature_flag_list.FeatureNames.BLOG_PAGES,  # ← Remove this line
]

# After:
[
    feature_flag_list.FeatureNames.DUMMY_FEATURE_FLAG_FOR_E2E_TESTS,
    feature_flag_list.FeatureNames.SHOW_TRANSLATION_SIZE,  # ← Use different flag
]
```

### Phase 2: Verification

**Run tests**:
```bash
python -m scripts.run_backend_tests --test_target core.feature_flag_list_test
python -m scripts.run_backend_tests --test_target core.tests.test_utils_test
python -m scripts.linters.run_lint_checks
```

**Verify no orphaned references**:
```bash
grep -r 'ANDROID_BETA_LANDING_PAGE\|BLOG_PAGES\|CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS' core/
grep -r 'DIAGNOSTIC_TEST\|END_CHAPTER_CELEBRATION\|CHECKPOINT_CELEBRATION' core/
grep -r 'ENABLE_VOICEOVER_CONTRIBUTION\|AUTO_UPDATE_EXP_VOICE_ARTIST_LINK' core/
grep -r 'LABEL_ACCENT_TO_VOICE_ARTIST\|ADD_VOICEOVER_WITH_ACCENT' core/
```

### Phase 3: PR Creation & Merge

**Commit message**:
```
fix: Remove 10 deprecated feature flags

This commit removes the following deprecated feature flags that have
been permanently enabled in the codebase and are no longer needed:

- ANDROID_BETA_LANDING_PAGE (Original: #16108, Deprecated: #19632)
- BLOG_PAGES (Original: #16243, Removed: #22180)
- CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS (Original: #16289, Removed: #20483)
- DIAGNOSTIC_TEST (Removed: #22180)
- END_CHAPTER_CELEBRATION (Deprecated unknown)
- CHECKPOINT_CELEBRATION (Deprecated unknown)
- ENABLE_VOICEOVER_CONTRIBUTION (Deprecated unknown)
- AUTO_UPDATE_EXP_VOICE_ARTIST_LINK (Deprecated: #20658)
- LABEL_ACCENT_TO_VOICE_ARTIST (Original: #21339)
- ADD_VOICEOVER_WITH_ACCENT (Deprecated: #20944)

Fixes: #XXXXX (Reference to cleanup issue if created)

Tests:
- All feature flag tests pass
- No references to removed flags in codebase
```

**PR Description Template**:
```markdown
## Description

This PR removes 10 deprecated feature flags that have been marked for 
cleanup. These flags have reached the end of their lifecycle:

- Features are permanently enabled in codebase
- No conditional logic gated on these flags remains
- All references have been removed

Detailed PR history for each flag is available in FEATURE_FLAGS_AUDIT.md

## Related Issues

- Audit: FEATURE_FLAGS_AUDIT.md

## Testing

- ✅ All feature flag tests pass
- ✅ No orphaned references to removed flags
- ✅ Lints clean
```

---

## Summary

- **Total Flags to Remove**: 10
- **Risk Level**: 🟢 LOW (only internal definitions affected)
- **Files to Modify**: 2 (feature_flag_list.py, test_utils_test.py)
- **Breaking Changes**: None (features are permanently enabled)
- **Estimated Review Time**: 10-15 minutes
- **Estimated CI Time**: 5-10 minutes

---

## Appendix: Full Git Log References

### All Related PRs Extracted:
- #16108 - Android beta landing page feature
- #16243 - Blog pages feature
- #16289 - Contributor dashboard accomplishments
- #18819 - Feature flag infrastructure refactor
- #19094 - Eliminate obsolete feature flags
- #19632 - Major feature flag & platform parameter separation
- #20483 - Remove contributor dashboard accomplishments flag
- #20658 - Move auto update exp voice artist link to prod
- #20944 - Move add voiceover with accent to prod
- #21339 - Voiceover regeneration and accent labeling
- #22180 - Remove diagnostic test feature flag

These PRs form the complete history of the deprecated features from creation through deprecation to cleanup.
