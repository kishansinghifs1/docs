# Porting the Lua Test Suite to Lunatik

This document describes how to approach porting the upstream Lua test suite to Lunatik.

## 1. What Lunatik changes versus user-space Lua

Before porting, align expectations with Lunatik's kernel constraints:

- no `io` and `os` libraries;
- no floating-point support (numbers are integers-only);
- some `math.*`, `debug.debug`, and `package.cpath` identifiers are unavailable;
- `require` cannot dynamically load C modules at runtime;
- `package.path` and `_VERSION` are kernel-specific.

These constraints are part of Lunatik's contract and should drive test selection/adaptation.

## 2. Relevant repository areas

- `lua/`: Lunatik's Lua core (adapted Lua source);
- `lunatik_core.c`, `lunatik_obj.c`, `lunatik_aux.c`: runtime/object model, file loading, and error handling;
- `lib/*.c`: Lunatik kernel bindings (`linux`, `socket`, `rcu`, `thread`, crypto, etc.);
- `lib/util.lua`: lightweight pass/fail test helper (`util.test`);
- `tests/`: current Lunatik tests/examples of the existing test style;
- `Makefile`: how tests are installed into `/lib/modules/lua/tests`.

## 3. Porting strategy

### Phase A: Scope and classify the upstream Lua tests

Classify upstream test files into buckets:

1. **Run unchanged**: pure-language semantics not relying on blocked features.
2. **Run with small edits**: mostly portable but need path/module adjustments.
3. **Kernel-adapted**: need replacement shims for unavailable facilities.
4. **Out-of-scope**: fundamentally dependent on `io`, `os`, float behavior, or dynamic C loading.

Produce a manifest table with columns:

- upstream file;
- category;
- adaptation notes;
- expected status (pass/skip/xfail);
- rationale.

### Phase B: Create a Lunatik-compatible test harness

Use `lib/util.lua` conventions (`util.test`, `pcall`, string diagnostics) and add a dedicated runner script under `tests/lua_suite/` to execute all selected tests and print machine-parseable summaries.

Recommended output format per case:

- `PASS\t<test_name>`
- `FAIL\t<test_name>\t<error>`
- `SKIP\t<test_name>\t<reason>`

At the end, print totals for pass/fail/skip.

### Phase C: Port Lua-side tests

- Start with core language semantics (`table`, `string`, coroutines, control-flow, metamethods except `__div`/`__pow` cases that imply float behavior).
- Replace file/process dependencies with in-memory helpers or explicit skips.
- Keep ported scripts isolated in `tests/lua_suite/` to simplify maintenance.

### Phase D: Port the C-API test portion as a kernel module

Create a dedicated module (for example `lib/luatestapi.c`) that exports Lua-callable functions used by the adapted test scripts.

Implementation guidelines:

- register with `luaL_newlib` and `luaL_requiref`, similar to existing Lunatik modules;
- expose only deterministic, kernel-safe functions;
- avoid sleeping operations unless runtime context explicitly allows it;
- return Lunatik error names (`EINVAL`, etc.) consistently for assertion stability.

Build integration:

- add object to `Kbuild` as optional `CONFIG_LUNATIK_TESTAPI`;
- add corresponding config entry in `Kconfig`;
- include module in install path and config generation as needed.

### Phase E: Automate execution and reporting

- Add launcher scripts under `tests/lua_suite/`.
- Add make targets to install/uninstall suite files.
- Document exact invocation commands with `lunatik run`/`spawn`.
- Generate a concise text report from test output.

## 4. Architecture test report requirements (x86_64 and arm64)

For the expected project deliverable, keep two report artifacts:

- `report/x86_64.md`
- `report/arm64.md`

Each report should include:

- kernel version and Lunatik commit;
- enabled Lunatik modules/config options;
- test matrix: pass/fail/skip/xfail counts;
- notable incompatibilities (with links to manifest entries);
- crashes/oops/panics if any (with dmesg excerpts).

## 5. Suggested milestone plan (90h/175h friendly)

1. **Milestone 1**: test inventory + compatibility manifest.
2. **Milestone 2**: harness + first green subset (language-only tests).
3. **Milestone 3**: kernel C-API support module and adapted C API tests.
4. **Milestone 4**: CI/manual execution scripts and dual-arch reports.
5. **Milestone 5**: cleanup, docs, and upstream-ready patch set.

## 6. Risks and how to mitigate

- **Kernel context differences**: annotate tests as sleepable/non-sleepable and run in proper context.
- **Non-determinism**: avoid timing-sensitive tests or add retries/timeouts.
- **Version drift with upstream Lua tests**: pin an upstream suite revision in the manifest.
- **False negatives from unsupported features**: use explicit `SKIP` with rationale rather than failing.

## 7. Definition of done

A solid completion should provide:

- adapted Lua test scripts with clear skip policy;
- Lunatik C-API testing library/module;
- reproducible test runner and instructions;
- results/report for both x86_64 and arm64.
