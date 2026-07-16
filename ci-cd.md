# Security Engine Architecture & CI/CD Pipeline Reference

This document provides a technical reference for the self-contained Security Assessment and Trend Analysis Engine implemented in the LibreHealth EHR project. 

---

## 1. Core Architecture (Layered Model)

The framework is structured as a pipeline of four distinct layers, decoupling raw data aggregation from CVSS scoring, clinical context analysis, and build-gate enforcement.

```
┌─────────────────────────────────────────────────────────┐
│              Layer 1: Finding Normalization             │
│   (Parses Composer Audit, PHPStan, & OWASP ZAP to JSON) │
└────────────────────────────┬────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│               Layer 2: CWE Weakness Mapping             │
│    (Translates tool outputs into standard CWE classes)  │
└────────────────────────────┬────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│        Layer 3: CVSS v3.1 Scoring & PHI Classifier      │
│  (Calculates base/adjusted scores & identifies exposure)│
└────────────────────────────┬────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                Layer 4: Security Gates                  │
│    (Enforces quality thresholds, blocks compilation)    │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Layer-by-Layer Technical Breakdown

### Layer 1: Finding Normalization (`FindingNormalizer.php`)
* **Purpose**: Resolves the lack of structure across different tool report formats.
* **Mechanism**: Reads input files and compiles them into a unified array of `Finding` objects.
* **Parser Implementations**:
  * `parseComposerAudit(string $path)`: Parses lockfile advisories. Identifies package names, vulnerable version bounds, CVE codes, links, and severity strings.
  * `parsePhpStan(string $path)`: Iterates through files with static analysis failures. Parses file path relative bounds, line numbers, and rule messages.
  * `parseZap(string $path)`: Extracts web alerts, target HTTP methods, parameter names, attacker payloads, vulnerability descriptions, solution recommendations, and target URLs.

---

### Layer 2: CWE Weakness Mapping (`CweMapper.php`)
* **Purpose**: Normalizes diverse vulnerability definitions into Common Weakness Enumeration (CWE) standards.
* **Ruleset**:
  * **Composer Audit**: If a package is marked as abandoned, it maps to `CWE-1104` (Use of Unmaintained Third-Party Components). Active vulnerabilities default to `CWE-937` (Vulnerable Component).
  * **PHPStan**: Uses token keyword checks on error strings:
    * Contains `null` $\to$ `CWE-476` (Null Pointer Dereference).
    * Contains `undefined` / `does not exist` $\to$ `CWE-398` (Code Quality Indicator).
    * Contains `deprecated` $\to$ `CWE-477` (Use of Obsolete Function).
    * Default $\to$ `CWE-703` (Improper Exception/Error Handling).
  * **OWASP ZAP**: Matches alert titles and description strings against a static regex-free dictionary (e.g., `clickjacking` $\to$ `CWE-1021`, `cross site scripting` $\to$ `CWE-79`, `sql injection` $\to$ `CWE-89`).

---

### Layer 3: CVSS v3.1 Scoring & PHI Classification
* **Formula Implementation (`CvssScorer.php`)**: Implements the official FIRST CVSS v3.1 specification. Computes Exploitability, Impact, and Base Risk scores.
* **Metrics Lookup (`FindingMetricsMapper.php`)**: Maps the resolved CWE to a CVSS metric vector. Examples:
  * `CWE-89` (SQLi) $\to$ `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` (Base Score: **10.0**)
  * `CWE-79` (XSS) $\to$ `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N` (Base Score: **6.1**)
* **PHI Classification (`PhiClassifier.php`)**:
  * Classifies vulnerabilities by inspecting files, tables, and controllers:
    * **Critical PHI** (e.g. `patients`, `records`, `prescriptions`, `billing`): Targets clinical and financial logs.
    * **Moderate PHI** (e.g. `addresses`, `contacts`, `facilities`): Targets demographic and logistical records.
    * **Non-PHI**: Static assets, public configurations, or infrastructure.
* **Duality Risk Adjustment**:
  * If a vulnerability occurs within a file path or context classified as **Critical** or **Moderate PHI**, the severity metrics (such as Confidentiality `C` and Integrity `I`) are escalated to `High`, adjusting the CVSS score to reflect medical data exposure risks.

---

### Layer 4: Security Gates (`SecurityGate.php`)
* **Purpose**: Evaluates the compiled scored findings against security policies to determine whether code is fit for deployment.
* **Enforced Threshold Rules**:
  1. **Maximum Score Limit**: The build fails if any vulnerability reaches an adjusted CVSS score $\ge 8.5$.
  2. **PHI Leakage Gate**: The build fails if any vulnerability affecting a **Critical** or **Moderate PHI** component reaches an adjusted CVSS score $\ge 7.0$.
* **Exit Protocol**: If any of the above conditions are violated, the command prints the offending files/lines, halts execution, and exits with code **1** to cancel runner execution.

---

## 3. Artisan Command Reference

```bash
php artisan security:score [options]
```

### Options

| Option | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `--composer-audit` | String | `composer-audit-report.json` | Path to the JSON report generated by `composer audit`. |
| `--phpstan` | String | `phpstan-report.json` | Path to the JSON report generated by PHPStan. |
| `--zap` | String | `zap_report.json` | Path to the JSON report generated by OWASP ZAP. |
| `--output-findings` | String | `findings.json` | Path to save compiled raw findings (Layer 1 output). |
| `--output-scored` | String | `scored_findings.json` | Path to save CVSS-scored findings (Layer 3 output). |
| `--output-html` | String | `security-report.html` | Path to save the interactive HTML dashboard. |
| `--history` | String | `security_trends.json` | Path to save/append the historical trends ledger. |
| `--fail-on-gate` | Boolean | `true` | Set to `false` to prevent exit code 1 on gate failures. |

---

## 4. GitLab CI/CD Pipeline Integration

### Pipeline Config (`.gitlab-ci.yml`)
The runner executes these jobs sequentially to feed reports to the scoring command:

1. **`build-ci-image`**:
   * Uses `Dockerfile.ci` to compile a runner environment containing PHP 8.3 CLI, composer, git, and Java dependencies.
2. **`static-scanning` (Parallel Jobs)**:
   * `composer-audit`: Runs `composer audit --format=json > composer-audit-report.json`.
   * `static-scanning-larastan`: Runs PHPStan analysis and pipes results to `phpstan-report.json`.
3. **`dynamic-scanning`**:
   * Runs `php artisan migrate:fresh --seed` to populate test fixtures.
   * Boots PHP web server in background: `php -S 127.0.0.1:8000`.
   * Runs OWASP ZAP baseline scan to inspect headers and paths, generating `zap_report.json`.
4. **`cvss-scoring`**:
   * Consumes all three JSON outputs and executes `php artisan security:score`.
   * Archives `security-report.html` as a GitLab build artifact.

### Caching and Artifacts
* The pipeline caches the `vendor/` directory across jobs using:
  ```yaml
  cache:
    key: "$CI_COMMIT_REF_SLUG"
    paths:
      - vendor/
  ```
* All intermediate reports (`composer-audit-report.json`, `phpstan-report.json`, `zap_report.json`) are configured as transient job artifacts (`expire_in: 1 week`) so the scoring job can fetch and consume them via runner dependencies.
