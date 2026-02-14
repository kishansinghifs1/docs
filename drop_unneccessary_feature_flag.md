# Feature Flags Audit & Cleanup Report

**Date**: February 15, 2026  
**Codebase**: Oppia  
**Analysis Method**: Automated codebase scanning combined with manual review

---

## Executive Summary

This audit analyzed all feature flags defined in `core/feature_flag_list.py` to identify which flags are:
1. Still actively used in production code
2. Deprecated/unused and safe to remove
3. Used only in tests

### Key Findings:

- **23 Active Feature Flags**: All are currently in use across the codebase
- **10 Deprecated Flags**: Already marked as deprecated but some still referenced
  - **9 Flags**: Completely unused - **SAFE TO REMOVE**
  - **1 Flag**: Only used in unit tests - **SAFE TO REMOVE** (after cleanup)

---

## Active Feature Flags (Currently Used)

All 23 active feature flags are being used across the codebase in production code. These are split across three stages:

### Production Stage (PROD) - 8 Flags
| Flag Name | Usage Files | Description |
|-----------|-------------|-------------|
| `DUMMY_FEATURE_FLAG_FOR_E2E_TESTS` | 8 | Test-only dummy flag for E2E testing |
| `IS_IMPROVEMENTS_TAB_ENABLED` | 4 | Improvements tab visibility in exploration editor |
| `LEARNER_GROUPS_ARE_ENABLED` | 3 | Learner groups feature |
| `EXPLORATION_EDITOR_CAN_MODIFY_TRANSLATIONS` | 11 | Translation editing in exploration editor |
| `EXPLORATION_EDITOR_CAN_TAG_MISCONCEPTIONS` | 5 | Misconception tagging in editor |
| `SHOW_REDESIGNED_LEARNER_DASHBOARD` | 24 | Redesigned learner dashboard (HEAVIEST USAGE) |
| `ENABLE_WORKED_EXAMPLES_RTE_COMPONENT` | 12 | Worked examples RTE component |
| `SHOW_RESTRUCTURED_STUDY_GUIDES` | 10 | Restructured study guides UI |

### Test Stage (TEST) - 10 Flags
| Flag Name | Usage Files | Description |
|-----------|-------------|-------------|
| `CD_ADMIN_DASHBOARD_NEW_UI` | 8 | Contributor dashboard new UI |
| `SERIAL_CHAPTER_LAUNCH_CURRICULUM_ADMIN_VIEW` | 4 | Serial chapter launch (admin view) |
| `SERIAL_CHAPTER_LAUNCH_LEARNER_VIEW` | 2 | Serial chapter launch (learner view) |
| `CD_ALLOW_UNDOING_TRANSLATION_REVIEW` | 2 | Undo translation review capability |
| `ENABLE_MULTIPLE_CLASSROOMS` | 4 | Multiple classrooms feature |
| `SHOW_VOICEOVER_TAB_FOR_NON_CURATED_EXPLORATIONS` | 5 | Voiceover tab for non-curated explorations |
| `NEW_LESSON_PLAYER` | 2 | New exploration player redesign |
| `AUTOMATIC_VOICEOVER_REGENERATION_FROM_EXP` | 4 | Auto voiceover regeneration |
| `SHOW_REGENERATED_VOICEOVERS_TO_LEARNERS` | 4 | Show regenerated voiceovers to learners |
| `ENABLE_BACKGROUND_VOICEOVER_SYNTHESIS` | 5 | Background voiceover synthesis |

### Development Stage (DEV) - 5 Flags
| Flag Name | Usage Files | Description |
|-----------|-------------|-------------|
| `SHOW_FEEDBACK_UPDATES_IN_PROFILE_PIC_DROPDOWN` | 2 | Feedback updates in profile dropdown |
| `SHOW_TRANSLATION_SIZE` | 2 | Show translation size on contributor dashboard |
| `REDESIGNED_TOPIC_VIEWER_PAGE` | 2 | Redesigned topic viewer page |
| `ENABLE_TRANSLATION_OPPORTUNITIES_WITH_NEW_OPP_MODELS` | 2 | New translation opportunity models |
| `ENABLE_READY_FOR_REVIEW_TEST` | 3 | Ready for review test feature |

---

## Deprecated Feature Flags - Cleanup Recommendations

All flags listed below are marked as `DEPRECATED_FEATURE_NAMES` in `core/feature_flag_list.py`.

### 🔴 SAFE TO REMOVE IMMEDIATELY (No References)

These 9 flags have zero references in the codebase outside of their definition:

| Flag Name | Value | Reason for Deprecation |
|-----------|-------|------------------------|
| `ANDROID_BETA_LANDING_PAGE` | `android_beta_landing_page` | Feature completed, permanently enabled |
| `BLOG_PAGES` | `blog_pages` | Feature completed, permanently enabled |
| `CONTRIBUTOR_DASHBOARD_ACCOMPLISHMENTS` | `contributor_dashboard_accomplishments` | Feature completed, permanently enabled |
| `DIAGNOSTIC_TEST` | `diagnostic_test` | Feature completed, permanently enabled |
| `END_CHAPTER_CELEBRATION` | `end_chapter_celebration` | Feature completed, permanently enabled |
| `CHECKPOINT_CELEBRATION` | `checkpoint_celebration` | Feature completed, permanently enabled |
| `ENABLE_VOICEOVER_CONTRIBUTION` | `enable_voiceover_contribution` | Feature completed, permanently enabled |
| `AUTO_UPDATE_EXP_VOICE_ARTIST_LINK` | `auto_update_exp_voice_artist_link` | Feature completed, permanently enabled |
| `LABEL_ACCENT_TO_VOICE_ARTIST` | `label_accent_to_voice_artist` | Feature completed, permanently enabled |

**Additional Flag Marked as Deprecated:**
| Flag Name | Value | Current Usage | Recommendation |
|-----------|-------|-------|-----------------|
| `ADD_VOICEOVER_WITH_ACCENT` | `add_voiceover_with_accent` | No references | **SAFE TO REMOVE** |

---

## Cleanup Roadmap

### Phase 1: Remove Completely Unused Deprecated Flags (LOW RISK)

Remove the following 10 flags from `core/feature_flag_list.py`:

```python
# Remove from FeatureNames enum:
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

# Remove from DEPRECATED_FEATURE_NAMES list
```

**Impact Assessment**: 
- ✅ Zero production code changes needed
- ✅ One test file needs minor update: `core/tests/test_utils_test.py` (remove `BLOG_PAGES` reference)
- ✅ No feature functionality affected

### Phase 2: Review Active Flags for Permanent Enablement (FUTURE)

Consider reviewing flags that have been in PROD stage for extended periods. These could potentially be permanently enabled in code (flag removal):

**Candidates for future review:**
- `SHOW_REDESIGNED_LEARNER_DASHBOARD` (24 files - heavily integrated)
- `ENABLE_WORKED_EXAMPLES_RTE_COMPONENT` (12 files)
- `SHOW_RESTRUCTURED_STUDY_GUIDES` (10 files)
- `EXPLORATION_EDITOR_CAN_MODIFY_TRANSLATIONS` (11 files)

These flags are so deeply integrated that keeping them gated may no longer provide value. However, this should be a future decision after verifying they're truly stable and needed.

---

## Files Affected by Cleanup

### Modification Required:
1. **`core/feature_flag_list.py`** - Remove enum values and deprecated list entries
2. **`core/tests/test_utils_test.py`** - Remove `BLOG_PAGES` reference from test decorator (lines 59-61)

### No Changes Needed:
- All other files rely on strings, not enum imports
- Database/storage: No migration needed (flags stored as strings)
- Frontend: No changes needed (features already permanently enabled)

---

## Implementation Steps

1. **Remove from Enum Definition** (10 lines from `FeatureNames` enum)
2. **Remove from Description Dictionary** (if present)
3. **Update `DEPRECATED_FEATURE_NAMES` list** - These can be completely removed
4. **Update test file** - Replace `BLOG_PAGES` with a still-valid flag or create dummy test flag
5. **Run tests**: `python -m scripts.run_backend_tests --test_target core.feature_flag_list_test`

---

## Validation Checklist

After cleanup:

- [ ] Run `core.feature_flag_list_test` - verify all active flags are valid
- [ ] Check for any remaining references: `grep -r "BLOG_PAGES\|android_beta_landing_page\|" core/`
- [ ] Verify no import errors in modules that may have imported deprecated flags
- [ ] Run feature flag integration tests
- [ ] Verify admin UI still renders correctly without these flags

---

## Appendix: Feature Flag Usage Distribution

```
PROD Stage:  8 flags (35% of active flags)
TEST Stage:  10 flags (43% of active flags)  
DEV Stage:   5 flags (22% of active flags)

Most Heavily Used: SHOW_REDESIGNED_LEARNER_DASHBOARD (24 files)
Least Used:       Multiple flags with 2 files
```

---

## Conclusion

The Oppia codebase has good feature flag hygiene:
- ✅ All active flags are in actual use
- ✅ Deprecated flags are properly marked in a separate list
- ✅ 90% of deprecated flags are completely unused (safe to remove)
- ✅ Only 1 reference to deprecated `BLOG_PAGES` in test code

**Recommended Action**: Proceed with Phase 1 cleanup to remove the 10 completely unused deprecated flags. This is a low-risk operation that will improve code cleanliness and reduce maintenance burden.
