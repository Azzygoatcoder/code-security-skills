---
name: code-security-audit
description: Use when the user wants to audit a codebase for security vulnerabilities, perform a security review, do penetration testing, run a 代码审计 or 安全审查, check for security issues, or verify that security fixes have been applied. Triggers on phrases like "audit this repo", "security review", "find vulnerabilities", "安全审计", "代码审计", "再审计", "verify fixes", "安全扫描".
---

# Code Security Audit

## Overview

Systematic security audit of any codebase using parallel domain exploration. Launch independent review agents across three security domains simultaneously, then synthesize findings into a structured audit report with severity ratings and concrete remediation steps.

**Core principle:** Coverage through parallelization. Three focused agents catch more than one broad agent — each domain has different grep patterns, different mental models, and different blind spots.

## Slash Commands

| Command | Action |
|---------|--------|
| `/audit` | Full security audit — explore → verify → report (Phase 1–3) |
| `/reaudit` | Verify previous audit fixes were applied (Phase 4) |
| `/reaudit mark-fixed <ID>` | Lightweight: mark a finding as fixed (update status annotation only, no file read) |
| `/reaudit mark-deferred <ID>` | Mark a finding as structurally deferred |
| `/reaudit status` | Show current fix progress (count by status from annotations) |
| `/code-security-audit` | Same as `/audit` (canonical name) |

## When to Use

Trigger when the user asks to:
- Audit a repository or codebase for security issues
- Perform a security review or vulnerability assessment
- Check if security fixes were properly applied (re-audit)
- Find vulnerabilities before a release or deployment

Do NOT use for:
- Reviewing a single PR diff (use `/security-review` or manual review)
- Checking one specific function for bugs (use systematic-debugging)
- General code review for style/architecture (use `/code-review`)

## Audit Workflow

### Phase 1 — Parallel Exploration

Launch **3 Explore agents simultaneously** (single message, parallel tool calls). Each agent covers one security domain.

**Agent A: Secrets & Credentials**
Prompt template:
```
Explore the codebase at <path> for security issues related to secrets, credentials, and sensitive data handling:
1. Hardcoded API keys, tokens, passwords in source files
2. .env files, config files that might contain secrets
3. How credentials are stored, accessed, and written
4. Any logging or error messages that might leak sensitive data
5. Check .gitignore — are sensitive file patterns properly excluded?
6. Check git history for accidentally committed secrets
For each finding, assign a stable category prefix:
  SECRET (leaked keys/tokens), DEP (file permissions/temp files)
Report findings with specific file paths, line numbers, and code snippets.
```

**Agent B: Input Validation & Injection**
Prompt template:
```
Explore the codebase at <path> for security issues related to input validation, injection, and unsafe data handling:
1. Command injection: shell=True, os.system, subprocess with user input in command strings
2. SQL injection if database queries exist
3. Path traversal: user input used in file paths without validation
4. Insecure deserialization: pickle, yaml.load, marshal
5. XSS: innerHTML, document.write, unsanitized user data in HTML context
6. SSRF: user-controlled URLs in HTTP clients without allowlist validation
7. Unsafe eval/exec usage
8. Template injection (SSTI)
For each finding, assign a stable category prefix:
  SSRF (SSRF), PATH (path traversal), CMD (command injection),
  XSS (cross-site scripting), CODE (eval/exec)
Report findings with specific file paths, line numbers, and code snippets.
```

**Agent C: Auth, Cryptography & Dependencies**
Prompt template:
```
Explore the codebase at <path> for security issues related to:
1. Authentication and authorization: missing auth checks, weak auth mechanisms
2. Cryptography: weak algorithms (MD5, SHA1, DES), hardcoded keys, improper TLS
3. Session management: insecure cookies, missing HttpOnly/Secure/SameSite
4. CSRF protections on state-changing endpoints
5. Dependency security: unpinned versions, known-vulnerable packages, missing lockfile
6. File permissions: credential files with weak permissions
7. Temp file handling: predictable names, missing cleanup
For each finding, assign a stable category prefix:
  AUTH (missing/weak auth), CRYPTO (weak crypto/TLS),
  DEP (dependencies/file-perms/temp-files)
Report findings with specific file paths, line numbers, and code snippets.
```

**Category-Prefixed IDs are stable** — they don't shift when new findings are added across audits. Use the prefix table in the report template (Phase 3).

**Customize the prompts** based on the codebase language and framework. Add language-specific patterns (e.g., for Python add `os.popen`, for JS add `eval`, for Go add `text/template` without escaping).

### Phase 2 — Deep-Dive Verification

After receiving all three agent reports:

1. **Read the flagged files** yourself. Agents provide summaries but you must verify each finding is real.
2. **Filter false positives**. Not everything an agent flags is exploitable. Check context: is user input actually reachable? Is the vulnerable function guarded?
3. **Deduplicate overlapping findings**. Agents A, B, and C may flag the same issue independently. Merge findings that:
   - Share the same file AND lines are within 15 of each other → same root cause
   - Share the same vulnerability category AND same file → likely the same issue
   - Keep the most detailed description; note both agent sources in the merged entry
4. **Assign severity** to each confirmed (and deduplicated) finding:

| Severity | Criteria |
|----------|----------|
| **Critical** | Remote unauthenticated access to secrets, arbitrary code execution, complete system compromise |
| **High** | Authenticated access to others' data, privilege escalation, injection with direct impact |
| **Medium** | Information disclosure, missing hardening, defense-in-depth gaps |
| **Low** | Best-practice violations with minimal direct risk, cosmetic issues |

5. **Assign a stable category-prefixed ID** to each finding using this table:

| Prefix | Category | Typical patterns |
|--------|----------|------------------|
| `SSRF` | SSRF / URL injection | Unsanitized URL in HTTP client, Playwright navigation |
| `PATH` | Path traversal | `../` in file paths, missing `is_relative_to` |
| `AUTH` | Missing/weak authentication | Unguarded endpoint, missing auth check |
| `CMD` | Command injection | `shell=True`, `os.system`, MATLAB -batch |
| `XSS` | Cross-site scripting | `innerHTML`, unsanitized HTML output |
| `SECRET` | Secrets/credentials leak | Hardcoded keys, tokens in logs/URL |
| `CRYPTO` | Weak cryptography | MD5/SHA1, `verify=False`, hardcoded keys |
| `DEP` | Dependencies / files / config | Unpinned deps, missing lockfile, temp files, file perms |
| `STATE` | Shared mutable state | Global state without isolation |

Number sequentially within each prefix: `SSRF-1`, `SSRF-2`, `PATH-1`, etc.

### Phase 3 — Report Compilation

Write the audit report to an agreed location. Use this exact structure:

```
# [Project Name] Security Audit Report

**Date:** [date]
**Audit commit:** [git rev-parse HEAD]
**Scope:** [what was reviewed]
**Methodology:** Three parallel reviews covering (1) secrets & credentials,
  (2) input validation & injection, (3) authentication, cryptography & dependencies

## Executive Summary
One paragraph: total findings by severity, most critical issues, overall risk posture.

## Risk Distribution
Table: | Severity | Count | Finding IDs |

## Critical Findings
One subsection per finding:

### SSRF-1 — [title]
<!-- AUDIT:STATUS=open SEVERITY=critical FILE=path/to/file.py LINES=123-145 -->

- **Finding:** title
- **Severity:** Critical
- **File:** path (line numbers)
- **Description:** what and why it matters
- **Vulnerable code:** fenced code block
- **Remediation:** concrete fix with corrected code

## High Findings
[Same format with category-prefixed IDs]

## Medium Findings
[Same format with category-prefixed IDs]

## Low Findings
[Same format with category-prefixed IDs]

## Cross-Cutting Recommendations
Themes spanning multiple findings.

## Remediation Priority Matrix
Table: | ID | Finding | Effort | Impact | Priority (P1-P4) |

## Appendix
Commit hash, verification guidance per finding.
```

**Status annotation format** (the `<!-- AUDIT:... -->` line after each finding title):

```
<!-- AUDIT:STATUS=<status> SEVERITY=<severity> FILE=<path> LINES=<start>-<end> [COMMIT=<hash>] -->
```

| Field | Required | Values |
|-------|----------|--------|
| `STATUS` | Yes | `open` → `fixed` → `deferred` → `not-fixed` → `partial` |
| `SEVERITY` | Yes | `critical` / `high` / `medium` / `low` |
| `FILE` | Yes | Relative path from repo root |
| `LINES` | Yes | Line range (e.g., `123-145`) |
| `COMMIT` | On fix | Hash of the commit that resolved this finding |

This annotation enables lightweight `mark-fixed` (just edit this line) and diff-based incremental re-audit (compare `FILE` + `LINES` against `git diff`).

### Phase 4 — Re-Audit (When Fixes Are Applied)

When the user says fixes are applied and wants verification, run an **incremental** re-audit:

1. **Parse status annotations** — scan the report for all `<!-- AUDIT:STATUS=... -->` lines. Extract `FILE`, `LINES`, and current `STATUS` for each finding.

2. **Diff-filter to changed files only**:
   ```bash
   git diff --name-only <audit-commit>..HEAD
   ```
   Compare against each finding's `FILE` field. Only findings whose files changed are candidates for re-verification.

3. **For each changed-file finding** — read the flagged file at the flagged location, check if the fix is applied:
   - **Fixed**: Vulnerability pattern removed or guard added → update `STATUS=fixed COMMIT=<hash>`
   - **Not Fixed**: Original vulnerable code still present → update `STATUS=not-fixed`
   - **Partially Fixed**: Some but not all addressed → update `STATUS=partial`
   - **Deferred**: Structural/architectural issue acknowledged → keep `STATUS=deferred`

4. **For unchanged-file findings** — skip read, keep current STATUS. Add `NOTE=unchanged` if desired.

5. **Check for regressions** — did the diff introduce any new vulnerability patterns? Quick scan of added lines for common patterns (shell=True, innerHTML, requests.get with user input, etc.).

6. **Update the report** — add a Re-Audit section at the top:

```
## Re-Audit ([date])

**Diff range:** <audit-commit>..<current-commit>
**Files changed:** N

### Status Summary
Table: | Status | Count |
       | fixed | N |
       | not-fixed | N |
       | partial | N |
       | deferred | N |
       | unchanged | N |

### Fixed (N of total)
Table: | ID | Finding | Fix Verified At |

### Not Fixed (N of total)
Table: | ID | Finding | Status/reason |

### Partially Fixed (N of total)
Table: | ID | Finding | What's done vs. remaining |

### New Findings
Any issues introduced by the fixes.

### Verdict
One paragraph summary.
```

**Also update the in-line annotations** — when a finding is verified as fixed, change its `<!-- AUDIT:STATUS=... -->` line from `STATUS=open` to `STATUS=fixed COMMIT=<hash>`.

## Quick Reference: Vulnerability Categories

See `references/vulnerability-patterns.md` for grep patterns and remediation templates for each category. Read that file before starting Phase 2 to know what to look for.

Key categories covered:
- Path Traversal, CSRF/CORS, Command Injection, XSS, SSRF
- SQL Injection, Insecure Deserialization, SSTI, Eval Injection
- File Permissions, Plaintext Credentials, Environment Leakage
- Dependency Management, Subprocess Injection
- Auth Bypass, Session Weaknesses, Cryptographic Weaknesses
- Temp File Handling, Log Forging, Race Conditions

## Common Mistakes

| Mistake | Correction |
|---------|------------|
| Running a single broad agent | Three focused agents find different things. Always use all three. |
| Accepting agent findings without reading code | Agents summarize; you must verify each finding's reality and severity. |
| Skipping Phase 4 because "fixes look right" | Always verify every finding against current code. Fixes can be incomplete or introduce new issues. |
| Writing report before verifying | Phase 3 (write) only after Phase 2 (verify). |
| Using severity labels inconsistently | Follow the criteria table above. Critical = remote + unauthenticated + secret access or RCE. |
| Not including concrete remediations | Every finding must have a fix code snippet, not just "fix this." |
| Forgetting to check .gitignore and git history | Secrets accidentally committed are still in history even if removed from HEAD. |
| Using sequential IDs (N1, N2...) instead of category prefixes | Sequential IDs shift when new findings are inserted. Use stable category-prefixed IDs (SSRF-1, PATH-1, etc.). |
| Skipping status annotations in findings | Without `<!-- AUDIT:STATUS=... -->` lines, `mark-fixed` and incremental re-audit cannot work. |

## After the Audit

- **Report location:** Place the report at `docs/SECURITY_AUDIT.md` by default. Add it to `.gitignore` so it stays local.
- **Follow-up:** Offer to fix the highest-priority findings (P1 items).
- **Pattern collection:** After the audit, review `references/vulnerability-patterns.md`. Add any new patterns you discovered. This is how the skill evolves.
