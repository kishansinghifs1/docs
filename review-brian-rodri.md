# Blog Author Changes — Requested vs Implemented

Date: 2026-02-07

## Overview
- Problem: Public blog pagination crashed for later pages when authors were deleted.
- Goal: Fix the crash and prevent similar regressions by centralizing absent-author handling and avoiding model creation during dashboard GETs.

## Requested Changes
- Make `get_blog_author_details` a pure getter with a `strict` flag.
- Avoid creating `BlogAuthorDetailsModel` on read (dashboard GET) to prevent blocking latency.
- Provide a consistent fallback for deleted/missing authors across views.
- Ensure author details exist when posts are published (create at publish time).
- Update controllers to consume the domain-level fallback instead of local fallback dicts.
- Update tests and E2E wait utilities to remove flakiness.

## Implemented Changes

- Domain
  - Added `BlogAuthorDetails.create_default_author_details_for_user(user_id)` to produce a default/error-state domain object (display name: "Blog Author").
  - Updated `get_blog_author_details(user_id, strict=True)` to:
    - Return the domain object when the model exists.
    - If missing and `strict=True`: raise a descriptive Exception including `user_id`.
    - If missing and `strict=False`: return the default domain object (never return `None`).

- Services
  - Ensured `publish_blog_post` creates the `BlogAuthorDetailsModel` at publish time if missing.
  - `update_blog_author_details` will create the model if missing before applying updates.

- Controllers
  - Removed local fallback dicts and bare `assert` checks in `blog_dashboard` and `blog_homepage` controllers.
  - Controllers now call `get_blog_author_details(..., strict=False)` where appropriate and use `author_details.displayed_author_name` directly.
  - No creation runs during dashboard GET requests.

- Tests & E2E
  - Updated backend unit and controller tests to reflect the new service contract (strict=False returns a default object).
  - Adjusted exception expectations and removed redundant `None` assertions in tests.
  - Added `DEFAULT_WAIT_TIME_MSECS_FOR_BLOG_OPERATIONS` and made test click/wait helpers accept optional timeouts to reduce flaky E2E failures during publish latency.

## Files Touched (representative)
- `core/domain/blog_domain.py` — added default author factory.
- `core/domain/blog_services.py` — changed `get_blog_author_details`, moved creation to publish/update.
- `core/controllers/blog_dashboard.py` — removed fallback creation-on-GET and local fallback dicts.
- `core/controllers/blog_homepage.py` — consume domain fallback; removed local fallbacks.
- `core/tests/...` — updated unit and controller tests.
- `core/tests/webdriverio_utils/waitFor.js` & `action.js` & `BlogPages.js` — E2E wait-time improvements.

## Validation Performed
- Ran targeted backend tests for affected suites (blog services, homepage, dashboard) — all passed locally.
- Fixed linter and MyPy issues introduced by contract changes.
- Scoped E2E timing change to blog operations to reduce flakiness; full E2E suite run is recommended.

## Rationale
- Centralizing the absent-author fallback in domain code ensures a single source of truth and removes duplicate fallback logic across controllers.
- Creating the author model at publish/update time prevents blocking work during read-only dashboard GET requests and avoids introducing latency for viewers.

## Next Steps / Recommendations
- Run the full E2E suite to validate browser flows end-to-end.
- Plan an audit/migration job to reconcile historical posts missing `BlogAuthorDetailsModel` (Apache Beam or similar).

---
If you want this saved elsewhere or formatted differently (release notes, changelog entry, or PR description), tell me where and I will adapt it.
