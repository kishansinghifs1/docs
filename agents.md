# LibreHealth EHR Laravel — Agent Context

This file captures the current state of the codebase as of the last full traversal. It is intended to prevent re-traversing the entire repository for common questions. **Keep it updated** as work progresses.

## 1. Project Overview

| Item | Detail |
|------|--------|
| **Project** | `librehealth/ehr` |
| **Laravel version** | **12.62.0** |
| **PHP requirement** | `^8.2` |
| **Frontend stack** | Inertia.js 2.0 + **Vue 2.6** + Laravel Mix 6 + Tailwind CSS 3 |
| **State/i18n** | Vuex 3 (`resources/app/store/theme.js`), Vue I18n 8 |
| **Working dir** | `/data/lh-ehr-laravel` |
| **Routing pattern** | Server-side Laravel routes return `Inertia::render('PageName')`; Inertia resolves `resources/app/pages/${name}.vue`. Named routes exposed via Ziggy (`@routes` in Blade). |

## 2. Directory Map

```
/data/lh-ehr-laravel
├── app/
│   ├── Console/Commands/           # Artisan commands incl. SecurityScoreCommand (uses exec)
│   ├── Http/Controllers/           # Auth, Dashboard, Admin, Installer, Pages
│   ├── Http/Middleware/            # Localization, HandleInertiaRequests, SelectedPatient, EHRInstaller, etc.
│   ├── Http/Resources/             # PatientResource, PatientCollection
│   ├── Models/                     # 80+ Eloquent models incl. User, Patient, Facility, Role, Permission
│   ├── Providers/                  # AppServiceProvider, AuthServiceProvider, etc.
│   ├── Security/Reporting/         # CLI security-report generator
│   └── Support/helper.php          # Misc helpers with type issues
├── bootstrap/
├── config/
├── database/
│   ├── factories/
│   ├── migrations/                 # ~130 migrations
│   └── seeders/                    # Includes hardcoded default passwords
├── resources/
│   ├── app/                        # Vue 2 SPA source
│   │   ├── app.js                  # JS entry
│   │   ├── assets/sass/app.scss    # SCSS entry
│   │   ├── components/             # Reusable Vue components
│   │   ├── global-components/      # EhrInput, EhrLabel, EhrButton, etc.
│   │   ├── layouts/                # DashboardLayout, AuthLayout, StaticPageLayout, InstallerLayout
│   │   ├── locales/                # i18n translation files
│   │   ├── pages/                  # Inertia page components
│   │   ├── shared/includes/        # Header, Footer, FlashMessages, Table/*, MobileNav
│   │   ├── store/                  # Vuex store (theme module only)
│   │   └── utils/                  # Helper utilities
│   ├── files/ehr_installer.json
│   └── views/                      # Blade root + vendor overrides
├── routes/
│   ├── web.php                     # All web routes
│   ├── api.php                     # Single /api/user route
│   ├── channels.php
│   └── console.php
├── public/                         # Compiled assets gitignored; must run npm build
├── tests/
└── .env                            # Contains real APP_KEY and DB_PASSWORD in workspace
```

## 3. Backend Routes Status

### Route files
- `/data/lh-ehr-laravel/routes/web.php` — all web routes
- `/data/lh-ehr-laravel/routes/api.php` — single `/api/user` route
- `/data/lh-ehr-laravel/routes/channels.php` — `App.User.{id}` channel
- `/data/lh-ehr-laravel/routes/console.php` — `inspire` command only

### Registered route groups

| Group | URI prefix | Middleware | Notes |
|-------|------------|------------|-------|
| Public pages | `/`, `/about`, `/contact`, `/version`, `/acknowledge-license-cert`, `/manual/installation-guide` | `web`, `localization` | `/version` and `/manual/...` point to **non-existent** controller methods |
| Auth | `/login`, `/logout`, `/forgot-password`, `/password/reset/{token}`, `/reset-password` | `web`, `localization`, `RedirectIfAuthenticated` (guest routes) | No rate limiting |
| Installer | `/install/*` | `web`, `localization` | GET-only; no actual DB install endpoint; `ehr.install` middleware not applied |
| Locale | `/lang/{lang}` | `web`, `localization` | |
| Dashboard | `/dashboard/*` | `web`, `localization`, `role:super_admin\|admin\|user` | Many stubs / missing methods; no explicit `auth` middleware |
| Admin | `/secure.portal/admin`, `/secure.portal/admin/horizon/*` | `web`, `localization`, `role:super_admin\|admin` | Horizon has own auth |
| Laratrust | `/dashboard/administration/manage/roles-permissions/*` | `role:super_admin\|admin` | Package CRUD for roles/permissions |

### Middleware aliases defined in `app/Http/Kernel.php`
- `auth`, `guest`, `localization`
- `forbid-banned-user` — **defined but never applied**
- `ehr.install` — **defined but never applied**
- `select.patient` — applied to patient sub-routes
- `json.response` — **not included in api group**

## 4. Missing / Broken Controller Methods

### Methods referenced by routes but NOT IMPLEMENTED

| Route | Controller | Missing method | File / Line |
|-------|------------|----------------|-------------|
| `dashboard/profile` | `App\Http\Controllers\Dashboard\DashboardController` | `profile` | `routes/web.php:103` |
| `dashboard/settings` | `App\Http\Controllers\Dashboard\DashboardController` | `settings` | `routes/web.php:104` |
| `dashboard/users/load/data` | `App\Http\Controllers\Dashboard\User\UserController` | `getUserData` | `routes/web.php:123` |
| `version` | `App\Http\Controllers\PagesController` | `showVersion` | `routes/web.php:53` |
| `manual/installation-guide` | `App\Http\Controllers\PagesController` | `getInstallationManual` | `routes/web.php:58` |

### Controllers with EMPTY / PLACEHOLDER methods

| File | Methods | Lines | Note |
|------|---------|-------|------|
| `app/Http/Controllers/Dashboard/User/UserController.php` | `index`, `create`, `store`, `show`, `edit`, `update`, `destroy` | 16, 26, 37, 48, 59, 71, 82 | All bodies are just `//`; declare `Response` return type but return `null` → **500 TypeError** |
| `app/Http/Controllers/Dashboard/Facility/FacilityController.php` | `store`, `edit`, `update`, `destroy` | 77, 127, 139, 150 | Empty bodies |
| `app/Http/Controllers/Dashboard/Patient/PatientController.php` | `store`, `edit`, `update`, `destroy` | 83, 185, 197, 208 | Empty bodies |
| `app/Http/Controllers/Dashboard/Patient/PatientAppointmentController.php` | `index` | 21 | Returns `User` data instead of appointments |
| `app/Http/Controllers/Dashboard/Patient/PatientHistoryController.php` | `index` | 13 | Fetches `$history` but never passes it to view |
| `app/Http/Controllers/Dashboard/Patient/PatientController.php` | `patientDocuments`, `patientReports`, `patientTransactions`, `patientLedger` | 243, 254, 265, 276 | All just redirect to patient show page |

### Controllers that exist but are NOT referenced by any route
- `app/Http/Controllers/Auth/RegisterController.php`
- `app/Http/Controllers/Auth/ConfirmPasswordController.php`
- `app/Http/Controllers/Auth/VerificationController.php`

## 5. Critical Runtime Errors

| Location | Problem | Severity |
|----------|---------|----------|
| `app/Http/Middleware/HandleInertiaRequests.php:102` | `decrypt(Cookie::get('ehr_patient'))` throws `DecryptException` when cookie missing → **crashes every Inertia request** | **Critical** |
| `app/Http/Middleware/SelectedPatient.php:27` | Same `decrypt(null)` crash; also fails when optional `pid` is null | **High** |
| `app/Http/Controllers/Dashboard/User/UserController.php` | All resource methods return no value against typed `Response` → 500 | **High** |
| `app/Http/Middleware/HandleInertiaRequests.php:45` | Calls `Debugbar()`; undefined when debugbar not loaded | **High** |
| `config/app.php:208` | `Barryvdh\LaravelIdeHelper\IdeHelperServiceProvider` registered unconditionally; it is require-dev | **High** |
| `app/Http/Controllers/Dashboard/DashboardController.php` | Missing `profile`/`settings` methods → `BadMethodCallException` | **High** |
| `app/Http/Controllers/PagesController.php` | Missing `showVersion`/`getInstallationManual` → `BadMethodCallException` | Medium |
| `app/Http/Controllers/Dashboard/Facility/FacilityController.php:58` | `create()` renders `Patient/Create` instead of `Facility/Create` | Medium |
| `app/Http/Controllers/Dashboard/Patient/PatientController.php:127,150,151` | Uses `$patient->deleted_at` for `created_at` and swaps timestamps | Medium |
| `app/Http/Controllers/Dashboard/Calendar/CalendarController.php:27,36` | Typo `$facility->update_at` / `$user->update_at` returns null | Low |
| `app/Http/Controllers/Dashboard/FlowBoardController.php:19-20` | Uses `fname`/`lname` but users table has `first_name`/`last_name` | Low |
| `app/Models/User.php:115-118` | `notifications()` relation points to `Notification` facade instead of `DatabaseNotification` | Medium |
| `app/Models/Address.php:26,34,44` | `belongsTo(..., 'id')` uses wrong foreign key | Medium |
| `app/Models/User.php:82-85` | `currency()` relation expects `currency_id` but users table has string `currency` | Medium |
| `app/Support/helper.php:65` | `redirectWithoutInertia()` declares return type `string` but returns `Response` | Low |
| `app/Support/helper.php:75` | `generatePID()` declares return type `string` but returns `int` | Low |
| `app/Http/Controllers/Auth/LoginController.php:78` | Override of `login()` does not call `session()->regenerate()` → session fixation | Medium |
| `app/Http/Controllers/Auth/ForgotPasswordController.php:59` | Status check always truthy → errors never shown | Medium |
| `app/Http/Controllers/Auth/RegisterController.php:67-71` | Creates user with `name` column which does not exist | Medium |
| `app/Support/Installer/RequirementsChecker.php:46` | Apache module check uses wrong variable | Low |
| `app/Http/Middleware/EHRInstaller.php:19,25,39,53` | Property mismatch (`$installerFilePath` vs `$installerFile`); `File::put` overwrites log | Medium |

## 6. Frontend Issues

### Missing Inertia pages
- `resources/app/pages/Admin/Index.vue` — referenced by `AdminController@index`
- `resources/app/pages/Patient/History.vue` — referenced by `PatientHistoryController@index`
- `resources/app/pages/User/` — entire user CRUD page directory missing

### Frontend runtime errors / bugs

| File | Issue | Severity |
|------|-------|----------|
| `resources/app/pages/Facility/Create.vue:38` | Arrow function `this` undefined; wrong route name `patients.index` | **High** |
| `resources/app/shared/includes/MobileNav.vue` | Calls non-existent store methods (`keyExist`, `getStorage`, `addStorage`); undefined `id` variable; dead marketplace routes | Medium |
| `resources/app/shared/includes/SearchInput.vue:27` | Navigates to non-existent `search` route | Medium |
| `resources/app/pages/Patient/Create.vue:401` | Wrong route name `patients.index` | Medium |
| `resources/app/pages/Facility/Show.vue:49` | Wrong route name `facilities.index` | Medium |
| `resources/app/pages/Patient/Profile.vue:6` | Uses `patient.username`, `patient.thumbnail` not supplied by controller | Low |
| `resources/app/shared/includes/Table/Detail.vue` | Uses `data.thumbnail`, `data.user.thumbnail`, `auth.profile_image` never provided | Low |
| `resources/views/app.blade.php:9` | `@yield('title')` incompatible with Inertia; titles broken | Low |

### Orphan pages (exist but no route renders them)
- `resources/app/pages/Fees.vue`
- `resources/app/pages/Inventory.vue`
- `resources/app/pages/Auth/VerifyAccount.vue`
- `resources/app/pages/Facility/Create.vue`
- `resources/app/pages/Error/Index.vue`

### XSS sinks (`v-html` / `{!! !!}`)

| File | Unsafe construct |
|------|------------------|
| `resources/app/shared/includes/FlashMessages.vue:17-18,38-39,59-60,80-81,102-103` | `v-html="item.title"` / `v-html="item.text"` |
| `resources/app/pages/Auth/RequestPasswordForm.vue:14` | `v-html="form.errors.email"` |
| `resources/app/pages/Auth/VerifyAccount.vue:10,20,30` | `v-html` on flash error and i18n strings |
| `resources/app/global-components/EhrInput.vue:49` | `v-html="description"` |
| `resources/app/global-components/EhrLabel.vue:3` | `v-html="label"` |
| `resources/app/shared/includes/Table/Label.vue:7,17` | `v-html` on status/role data |
| `resources/views/vendor/translation/notifications.blade.php:12` | `{!! Session::get('error') !!}` |
| `resources/views/vendor/translation/forms/text.blade.php:15` | `{!! $error !!}` |

## 7. Security Vulnerabilities

### Critical
| Issue | File(s) | Details |
|-------|---------|---------|
| `.env` contains real secrets | `.env` | `APP_KEY=base64:...` and `DB_PASSWORD=passw0rd` present in workspace |
| App crashes on missing patient cookie | `HandleInertiaRequests.php:102` | With `APP_DEBUG=true`, stack traces leak on every request |
| Installer routes unprotected | `routes/web.php:65-80`, `EHRInstaller.php` | `ehr.install` middleware never applied; `/install/*` stays public |

### High
| Issue | File(s) | Details |
|-------|---------|---------|
| Mass assignment over-permissive | `app/Models/User.php:32`, `Role.php`, `Permission.php`, `Team.php` | `$fillable` includes `password`, `deleted_at`, timestamps; Laratrust models use `$guarded = []`; `Patient` has no guard |
| Disabled/banned users can log in | `Auth/LoginController.php` | Does not check `active`, `authorized`, `banned_at`/`banned_until` |
| No login rate limiting | `routes/web.php:39-49` | Brute-force possible |
| Dev-only code in production stack | `config/app.php:208`, `HandleInertiaRequests.php:45` | IdeHelper and Debugbar loaded unconditionally |
| IDOR / missing per-resource auth | `PatientController`, `FacilityController`, `UserController` | Any role-holding user can access any record by ID |
| JS dependency vulnerabilities | `package-lock.json` | `npm audit`: 89 vulnerable packages (7 critical, 32 high) |

### Medium
| Issue | File(s) | Details |
|-------|---------|---------|
| Dashboard routes lack explicit `auth` | `routes/web.php:96` | Only `role` middleware; guests may get odd redirects |
| `forbid-banned-user` never applied | `app/Http/Kernel.php:75` | Ban package not enforced |
| API not forced to JSON | `app/Http/Kernel.php:73` | `json.response` not in api group |
| Session config insecure defaults | `config/session.php` | `secure` falls back false, `encrypt => false` |
| CSRF bypass in testing env | `VerifyCsrfToken.php:26` | `inExceptArray()` returns true in `testing` env |
| Open redirect via locale | `LocaleController.php:25` | `redirect()->back()` follows Referer |
| TrustHosts disabled | `app/Http/Kernel.php:21` | Commented out |
| Unvalidated datatable page length | `PatientController.php:220` | `$length` passed directly to `paginate()` |

## 8. API / Auth Status

- Only API route: `GET /api/user` in `routes/api.php:17`
- API guard uses plain `token` driver (`config/auth.php:44-48`)
- Sanctum installed but not configured (`EnsureFrontendRequestsAreStateful` commented out in api middleware group)
- `AuthServiceProvider` has no policy mappings
- `VerificationController` exists but no verification routes registered

## 9. Database / Models

- ~130 migrations under `database/migrations/`
- 80+ models under `app/Models/`
- Key entities: `User`, `Patients/Patient`, `Patients/PatientFaceSheet`, `Patients/PatientHistory`, `Facilities/Facility`, `Encounter`, `CalendarEvent`, `Role`, `Permission`, `Team`
- Migrations use `config('platform_settings.default_currency')` which exists

## 10. Test Status

- Running `vendor/bin/phpunit --no-coverage`: **Tests: 18, Assertions: 1, Errors: 17**
- Feature/Security tests fail with `PDOException: SQLSTATE[HY000] [2002] Connection refused`
- Only Unit suite passes
- `phpunit.xml` hardcodes MySQL host `127.0.0.1` and credentials `root`/`root`

## 11. Recommended Action Plan

### Phase 1 — Stop crashes
1. Guard `decrypt(Cookie::get('ehr_patient'))` in `HandleInertiaRequests.php` and `SelectedPatient.php`
2. Fix `SelectedPatient.php` optional `pid` handling
3. Implement or disable `UserController` resource methods
4. Add or remove missing `profile`, `settings`, `showVersion`, `getInstallationManual` routes/methods
5. Create missing Vue pages (`Admin/Index.vue`, `Patient/History.vue`) or remove references

### Phase 2 — Security hardening
1. Rotate `.env` secrets; set `APP_DEBUG=false` for non-local
2. Remove/disable dev-only providers/helpers in production
3. Tighten `$fillable`/`$guarded` on `User`, `Role`, `Permission`, `Team`, `Patient`
4. Enforce user status checks in login; apply `forbid-banned-user`
5. Apply `ehr.install` middleware to installer routes
6. Add rate limiting to auth routes
7. Add `auth` middleware to dashboard group
8. Replace `v-html`/`{!! !!}` with safe interpolation where possible

### Phase 3 — Implement missing functionality
1. Full CRUD for `UserController`, `FacilityController`, `PatientController`
2. Implement `getUserData`, patient history, appointments, documents, reports, transactions, ledger
3. Fix `FacilityController::create()` rendering wrong page
4. Fix `PatientController` timestamp mapping
5. Implement `DashboardController::profile`/`settings` + Vue pages
6. Fix frontend route names and `MobileNav.vue` dead code

### Phase 4 — Logic bug cleanup
1. Fix `User::notifications()`, `User::currency()`, `Address` foreign keys
2. Fix `CalendarController` `update_at` typos
3. Fix `FlowBoardController` field names
4. Fix `ForgotPasswordController` status check
5. Fix `RequirementsChecker` variable bug
6. Fix `EHRInstaller` property/logic bugs
7. Fix `RegisterController` `name` column issue
8. Add `session()->regenerate()` to login
9. Validate referer in `LocaleController`

### Phase 5 — Testing & dependencies
1. Fix `phpunit.xml` to use SQLite or env-driven DB
2. Add feature tests for login, dashboard, patient list
3. Run `npm audit` / `composer audit`; triage and update
4. Build frontend (`npm install && npm run production`) and verify Inertia loads

## 12. How to Verify Route List

```bash
cd /data/lh-ehr-laravel
php artisan route:list
```

## 13. Notes for Future Agents

- **Do not commit `.env`**. It currently contains real credentials in the workspace.
- **Always guard cookie decryption**; the `ehr_patient` cookie is frequently absent.
- **The dashboard currently crashes on every request** until Phase 1 items are fixed.
- **Mass assignment models are dangerous**; do not rely on default Eloquent guarding.
- **Frontend uses Ziggy** for named routes; check `route('name')` calls against `php artisan route:list`.
- **Compiled assets are gitignored**; run `npm run dev`/`prod` after frontend changes.
- **No client-side SPA router exists**; navigation is Inertia visits to Laravel routes.
