# LibreHealth EHR — MU Workflow Security Mapping

## Overview

This document maps all target Meaningful Use (MU) certification workflows in the LibreHealth EHR
system, identifies their components, and catalogs security-relevant surfaces for automated CVSS
scoring and integration testing.

---

## 1. Authentication & Session Management Workflow

| Component          | File                                                        | Security Surface                |
|--------------------|-------------------------------------------------------------|---------------------------------|
| Login Controller   | `app/Http/Controllers/Auth/LoginController.php`             | Credential handling, session    |
| Password Reset     | `app/Http/Controllers/Auth/ForgotPasswordController.php`    | Token generation, email enum    |
| Reset Execution    | `app/Http/Controllers/Auth/ResetPasswordController.php`     | Token validation, password set  |
| Auth Middleware    | `app/Http/Middleware/Authenticate.php`                      | Session validation              |
| CSRF Middleware    | `vendor/.../VerifyCsrfToken.php`                            | CSRF protection                 |
| Throttle           | `vendor/.../ThrottleRequests.php`                            | Rate limiting (NOT on login)    |

### Known Vulnerabilities
- **No rate limiting on login route** — brute-force possible
- **Password reset always returns success** — `ForgotPasswordController` logic bug with `||`
- **Password history model is empty stub** — password reuse not enforced
- **Banned user middleware registered but not applied**

### MU Relevance
- **MU Criterion**: Authentication, Access Control, and Authorization (§170.315(d)(1))
- **Impact**: Unauthorized access to PHI if authentication is bypassed

---

## 2. Patient Demographics & Management Workflow

| Component               | File                                                                       | Security Surface           |
|--------------------------|----------------------------------------------------------------------------|----------------------------|
| Patient Controller       | `app/Http/Controllers/Dashboard/Patient/PatientController.php`             | CRUD, data exposure        |
| Patient History          | `app/Http/Controllers/Dashboard/Patient/PatientHistoryController.php`      | IDOR risk                  |
| Patient Appointments     | `app/Http/Controllers/Dashboard/Patient/PatientAppointmentController.php`  | Debug code exposure        |
| Patient Selection        | `app/Http/Middleware/SelectedPatient.php`                                  | Cookie-based auth          |
| Patient Model            | `app/Models/Patients/Patient.php`                                          | Mass assignment            |
| Patient Factory          | `database/factories/Patients/PatientFactory.php`                           | Test data generation       |

### Known Vulnerabilities
- **IDOR in PatientHistoryController** — uses `Patient::find($id)` without ownership check
- **Patient CRUD methods are empty stubs** — no input validation exists
- **Cookie-based patient selection** — tamper risk despite encryption
- **`dd()` debug calls in PatientAppointmentController** — information disclosure

### MU Relevance
- **MU Criterion**: Demographics (§170.315(a)(5)), Patient Selection (§170.315(a)(1))
- **Impact**: Unauthorized access/modification of patient PHI

---

## 3. User & Role Management Workflow

| Component           | File                                                          | Security Surface            |
|---------------------|---------------------------------------------------------------|-----------------------------|
| User Controller     | `app/Http/Controllers/Dashboard/User/UserController.php`      | CRUD (all stubs)            |
| User Model          | `app/Models/User.php`                                         | Laratrust, mass assignment  |
| Role Model          | `app/Models/Role.php`                                         | `$guarded = []`             |
| Permission Model    | `app/Models/Permission.php`                                   | `$guarded = []`             |
| Laratrust Config    | `config/laratrust.php`                                        | RBAC configuration          |
| Admin Controller    | `app/Http/Controllers/Admin/AdminController.php`              | Admin panel access          |

### Known Vulnerabilities
- **All UserController methods are empty stubs** — no authorization logic
- **Role/Permission models use `$guarded = []`** — mass assignment of privileges
- **No authorization policies or gates defined** — only role-based middleware
- **No user activity logging implemented** — audit models are stubs

### MU Relevance
- **MU Criterion**: Role-Based Access Control (§170.315(d)(1)), Audit Logging (§170.315(d)(2))
- **Impact**: Privilege escalation, unauthorized role changes

---

## 4. Clinical Documentation Workflow

| Component              | File                                            | Security Surface         |
|------------------------|-------------------------------------------------|--------------------------|
| Encounter Model        | `app/Models/Encounter.php`                      | Empty stub               |
| Prescription Model     | `app/Models/Prescription.php`                   | Empty stub               |
| Immunization Model     | `app/Models/Immunization.php`                   | Clinical data            |
| Form Models (22 types) | `app/Models/Forms/Form*.php`                    | Clinical form data       |
| Document Model         | `app/Models/Document.php`                       | File handling            |
| Drug Models            | `app/Models/Drugs/Drug*.php`                    | Medication data          |

### Known Vulnerabilities
- **All clinical models are stubs** — no validation or access control
- **No controller endpoints for clinical data** — workflow not yet implemented
- **Document model exists but no upload/download controller** — file handling untested

### MU Relevance
- **MU Criterion**: CPOE (§170.315(a)(1)), Medication List (§170.315(a)(7)), Problem List (§170.315(a)(6))
- **Impact**: Corruption or unauthorized access to clinical records

---

## 5. Facility Management Workflow

| Component            | File                                                               | Security Surface         |
|----------------------|--------------------------------------------------------------------|--------------------------|
| Facility Controller  | `app/Http/Controllers/Dashboard/Facility/FacilityController.php`   | Partial CRUD             |
| Facility Model       | `app/Models/Facilities/Facility.php`                               | Data integrity           |
| Facility User        | `app/Models/Facilities/FacilityUser.php`                           | User-facility binding    |
| Facility Factory     | `database/factories/Facilities/FacilityFactory.php`                | Test data                |

### Known Vulnerabilities
- **`store`/`edit`/`update`/`destroy` are stubs** — no input validation
- **No facility-level authorization** — any authenticated user can view facilities
- **No multi-tenancy isolation** — facilities not used for data scoping

### MU Relevance
- **MU Criterion**: Facility Demographics (§170.315(a)(5))
- **Impact**: Incorrect facility data could affect clinical decision-making

---

## 6. Dashboard & Reporting Workflow

| Component             | File                                                            | Security Surface         |
|-----------------------|-----------------------------------------------------------------|--------------------------|
| Dashboard Controller  | `app/Http/Controllers/Dashboard/DashboardController.php`        | Data aggregation         |
| Flow Board Controller | `app/Http/Controllers/Dashboard/FlowBoardController.php`        | User data exposure       |
| Calendar Controller   | `app/Http/Controllers/Dashboard/Calendar/CalendarController.php`| Scheduling data          |

### Known Vulnerabilities
- **FlowBoard exposes `access_control` field** — internal security data leaked to frontend
- **Dashboard returns aggregate counts without per-facility scoping**
- **No audit logging on data access** — cannot track who viewed what

### MU Relevance
- **MU Criterion**: Audit Logging and Monitoring (§170.315(d)(2))
- **Impact**: Cannot demonstrate compliance with audit requirements

---

## 7. API & External Integration Workflow

| Component        | File                       | Security Surface            |
|------------------|----------------------------|-----------------------------|
| API Routes       | `routes/api.php`           | Single endpoint             |
| CORS Config      | `config/cors.php`          | Wide-open origins           |
| Sanctum          | (installed, not active)    | Token auth not configured   |

### Known Vulnerabilities
- **CORS allows all origins** (`*`) — any domain can make requests
- **Sanctum middleware commented out** — API uses basic token auth only
- **Single API endpoint** — but if expanded, security posture is weak

### MU Relevance
- **MU Criterion**: Interoperability (§170.315(g)(7)-(10))
- **Impact**: Unauthorized API access from any origin

---

## 8. System Installation & Configuration Workflow

| Component            | File                                                   | Security Surface            |
|----------------------|--------------------------------------------------------|-----------------------------|
| Installer Middleware | `app/Http/Middleware/EHRInstaller.php`                  | Installation gate           |
| Installer Config     | `config/ehr_installer.php`                              | System config               |
| Installer Controller | `app/Http/Controllers/Installer/InstallerController.php`| Setup flow                  |

### Known Vulnerabilities
- **EHR Installer middleware not in global stack** — post-installation, setup routes may be accessible
- **File-based installation check** — can be manipulated by file system access

### MU Relevance
- **MU Criterion**: System Security (§170.315(d)(12)-(13))
- **Impact**: Re-installation or reconfiguration by unauthorized users

---

## Summary: Security Priority Matrix

| Workflow                      | Risk Level | Implementation Status | Test Priority |
|-------------------------------|------------|-----------------------|---------------|
| Authentication & Sessions     | 🔴 Critical | Partial               | P0            |
| Patient Demographics          | 🔴 Critical | Partial               | P0            |
| User & Role Management        | 🟠 High     | Stubs                 | P1            |
| Clinical Documentation        | 🟠 High     | Stubs                 | P1            |
| Facility Management           | 🟡 Medium   | Partial               | P2            |
| Dashboard & Reporting          | 🟡 Medium   | Implemented           | P2            |
| API & External Integration    | 🟡 Medium   | Minimal               | P2            |
| System Installation           | 🟢 Low      | Implemented           | P3            |
