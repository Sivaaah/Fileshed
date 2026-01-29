# Security Audit Report: Fileshed Tool for OpenWebUI

## Identification

| Field | Value |
|-------|-------|
| **Filename** | `Fileshed.py` |
| **Version** | `1.0.3` |
| **Author** | Fade78 (with Claude Opus 4.5) |
| **License** | MIT |
| **Required OpenWebUI Version** | 0.4.0+ |
| **Lines of Code** | 8,556 |

---

## Executive Summary

Fileshed is a comprehensive persistent file storage tool designed for OpenWebUI with group collaboration capabilities. It provides shell command execution, file manipulation, SQLite database access, network downloads, and group-based file sharing. The tool demonstrates significant security awareness with multiple defense layers, but the inherent complexity and extensive attack surface introduce risks that must be carefully managed.

---

## Analysis Axis 1: Architecture and Separation of Concerns

### Design

The codebase employs a well-structured three-tier architecture:

1. **`Tools` class** — Public API exposed to the LLM (all `shed_*` methods)
2. **`_FileshedCore` class** — Internal implementation hidden from the LLM
3. **`_OpenWebUIBridge` class** — Isolated abstraction for OpenWebUI internal API

**Key Strengths:**
- Clean separation between public API and internal logic prevents LLM from invoking internal methods directly
- Comments explicitly warn LLM about method naming conventions (`shed_*` only)
- Singleton pattern for `_OpenWebUIBridge` avoids repeated imports
- Zone-based abstraction (uploads/storage/documents/group) provides logical isolation
- `ZoneContext` dataclass encapsulates zone-specific configuration cleanly
- Valves are properly defined inside the `Tools` class as required by OpenWebUI

**Weaknesses/Risks:**
- The codebase is extremely large (8,500+ lines), which increases cognitive load for auditors and risk of latent bugs
- Heavy reliance on Python's name mangling and OpenWebUI's tool discovery mechanism for hiding internal methods—this is a convention, not an enforcement mechanism
- Some exception handlers use bare `except Exception:` which masks root causes and could hide security-relevant failures
- The `_format_response` pattern standardizes output but generic error messages reduce forensic capability

**Assessment:** ★★★★☆

---

## Analysis Axis 2: LLM-Facing Attack Surface

### Exposure Model

All public methods accept user/LLM-controllable parameters. The tool explicitly documents in header comments which functions the LLM may call.

**Key Strengths:**
- Exhaustive docstrings with usage examples guide LLM toward correct usage
- Parameter validation occurs early with descriptive error messages
- The `allow_zone_in_path` parameter (default `False`) prevents common LLM mistakes like duplicating zone names in paths
- Contextual help system (`FUNCTION_HELP`) provides recovery guidance when errors occur
- Methods that require specific workflows (e.g., locked edit) explicitly document prerequisites

**Weaknesses/Risks:**
- The LLM can invoke ~40+ public methods, creating a large attack surface
- Complex parameter combinations (e.g., `shed_patch_text` with `position`, `line`, `pattern`, `regex_flags`, `match_all`) increase misuse risk
- The `allow_zone_in_path=True` override can be weaponized by a manipulated LLM to confuse path resolution
- No rate limiting on individual function calls—a hallucinating LLM could invoke thousands of operations
- The `shed_exec` function exposes ~80+ shell commands, each with its own argument semantics

**Assessment:** ★★★☆☆

---

## Analysis Axis 3: Filesystem Access and Path Isolation

### Chroot Model

The tool implements virtual chroot isolation per zone with multiple validation layers.

**Key Strengths:**
- `_validate_relative_path()` performs:
  - Unicode NFC normalization (prevents path confusion attacks)
  - Virtual path resolution to detect `..` escapes
  - Zone prefix duplication detection
  - Control character rejection
- `_resolve_chroot_path()` provides:
  - Symlink detection at each path component
  - Final path resolution with `relative_to()` verification
  - Symlink target validation (must stay within chroot)
- User directories use validated UUIDs preventing path injection
- Conversation IDs are sanitized before use in paths

**Weaknesses/Risks:**
- **Symlink TOCTOU**: Between symlink check and file operation, a race condition could theoretically allow escape (mitigated by operational context)
- `Path.resolve()` behavior varies by Python version—edge cases with non-existent paths
- Hard links are blocked (`ln` forbidden), but files could still reference same inode via external means
- The `allow_zone_in_path=True` flag, while documented, could be abused if LLM is manipulated

**Assessment:** ★★★★☆

---

## Analysis Axis 4: Command Execution and Input Validation

### Shell Command Model

`shed_exec` wraps `subprocess.run()` with extensive pre-execution validation.

**Key Strengths:**
- **Whitelist system**: Commands must be in `WHITELIST_READONLY` or `WHITELIST_READWRITE`
- **Blacklist enforcement**: Dangerous commands (shells, interpreters, privilege escalation) explicitly blocked
- **Dangerous argument pattern detection**: Regex blocks `; & | && || > >> < << $( ${` and backticks
- **Per-command validation**:
  - `find -exec` blocked
  - `awk system()` and `getline` pipes blocked
  - `tar --absolute-names` blocked
  - `ln` entirely forbidden
- **Resource limits**: Memory (`RLIMIT_AS`) and CPU time (`RLIMIT_CPU`) constraints via `preexec_fn`
- **Timeout enforcement** with clamping to admin-configured maximum
- **Output truncation** prevents context pollution

**Weaknesses/Risks:**
- The whitelist includes powerful commands: `ffmpeg`, `pandoc`, `magick/convert`, `git` with network capabilities
- Pattern matching for dangerous arguments may have edge cases (e.g., encoded characters, unusual shells)
- `sed -e` expressions are complex to validate—current logic may miss sophisticated injection patterns
- `awk`/`jq` allow pipe character internally, but the `COMMANDS_ALLOWING_PIPE` exception could be exploited
- The `regex_flags` parameter in `shed_patch_text` accepts arbitrary Python regex flags (`re.DOTALL`, etc.)
- No sandboxing beyond resource limits—commands run in the OpenWebUI process context

**Assessment:** ★★★☆☆

---

## Analysis Axis 5: Network Access and Data Exfiltration

### Network Control Model

Network access is governed by the `network_mode` valve (admin-controlled).

**Key Strengths:**
- **Three-tier model**: `disabled` (default), `safe` (download only), `all` (full access)
- **Safe mode protections**:
  - `curl`/`wget` forbidden POST/PUT options (`CURL_FORBIDDEN_GET_OPTS`)
  - `ffmpeg` output protocols blocked (`FFMPEG_OUTPUT_PROTOCOLS`)
  - `git push` requires `all` mode
- **URL pattern detection** blocks network access in arguments when disabled
- Git network operations (`clone`, `fetch`, `pull`, `push`) are categorized separately

**Weaknesses/Risks:**
- `network_mode=all` enables full data exfiltration—this is an **intentional feature** but represents high risk if misconfigured
- `ffmpeg` with `all` mode can stream to arbitrary endpoints
- DNS exfiltration is not blocked (commands could leak data via DNS queries)
- Even in `safe` mode, downloaded content could contain malicious payloads (e.g., malicious archives, polyglots)
- The `pandoc`, `convert`, and `ffprobe` input URL allowance could be exploited for SSRF against internal services
- No TLS certificate validation enforcement for downloads

**Assessment:** ★★★☆☆

---

## Analysis Axis 6: Concurrency, Locking, and State Management

### Locking Model

The tool implements advisory file locking for the "locked edit" workflow.

**Key Strengths:**
- **Atomic lock creation**: Uses `os.open()` with `O_CREAT | O_EXCL` for race-free lock acquisition
- **Lock ownership validation**: Prevents other conversations from hijacking locks
- **Lock expiration**: `lock_max_age_hours` valve enables automatic cleanup
- **Maintenance function**: `shed_maintenance()` cleans expired locks and orphan editzones
- **Conversation isolation**: Locks are per-conversation, preventing cross-conversation conflicts

**Weaknesses/Risks:**
- Advisory locking only—other processes or direct filesystem access can bypass
- **Lock file corruption handling**: If lock file is corrupted during `O_CREAT | O_EXCL` race, fallback to `write_text()` is non-atomic
- Lock path calculation uses string manipulation—edge cases with special characters possible
- No deadlock detection for complex multi-file operations
- Editzone cleanup in `finally` blocks may fail silently if permissions change mid-operation

**Assessment:** ★★★★☆

---

## Analysis Axis 7: Data Processing and Persistence

### SQLite and CSV Processing

**Key Strengths:**
- **Parameterized queries**: `shed_sqlite` uses `?` placeholders preventing SQL injection
- **Table name validation**: Regex `^[a-zA-Z_][a-zA-Z0-9_]*$` blocks injection via table names
- **Read-only mode**: `sqlite_readonly` valve restricts to SELECT only
- **Row limits**: Default 10 rows for SELECT, hard cap at 10,000 even with `limit=0`
- **CSV auto-detection**: Smart encoding and delimiter detection
- **Column name sanitization**: Non-alphanumeric characters replaced with underscores

**Weaknesses/Risks:**
- **`sqlite_readonly=False` (default)**: Allows arbitrary DDL/DML including `DROP`, `DELETE`, `UPDATE`
- Column names in CSV import use string substitution in `CREATE TABLE` (though sanitized)
- Pandas `read_csv` with `sep=None` and `engine='python'` has historical vulnerabilities
- Large CSV imports could cause memory exhaustion despite quota checks
- SQLite `ATTACH DATABASE` could potentially be abused if not blocked (not explicitly validated)

**Assessment:** ★★★☆☆

---

## Analysis Axis 8: ZIP Archive Processing

### ZIP Security Model

**Key Strengths:**
- **ZIP Slip prevention**: All archive members validated against path traversal before extraction
- **ZIP bomb protection**:
  - Maximum decompressed size: 500 MB
  - Maximum files: 10,000
  - Maximum compression ratio: 100:1
- **Absolute path blocking**: Members starting with `/` rejected
- **Rollback on failure**: Partially extracted files cleaned up

**Weaknesses/Risks:**
- Quine/recursive ZIP structures not explicitly detected (ratio check may not catch all cases)
- `zipfile.extractall()` is called after validation—if validation has gaps, extraction proceeds
- Symbolic link members in ZIP files are not explicitly handled (Python's zipfile handles them as regular files by default)
- No content scanning for malicious payloads in extracted files

**Assessment:** ★★★★☆

---

## Analysis Axis 9: Multi-User and Group Access Control

### Group Model

**Key Strengths:**
- **OpenWebUI Groups API integration**: Leverages existing authentication system
- **Membership validation**: `_check_group_access()` verifies user belongs to group
- **File ownership model**: Three modes (`owner`, `group`, `owner_ro`) with database-backed tracking
- **Per-group quotas**: Separate from user quotas
- **Ownership transfer**: `shed_group_chown()` with ownership verification
- **Group ID validation**: UUID format or name resolution with case-sensitivity enforcement

**Weaknesses/Risks:**
- Group membership checked at operation start—changes during long operations not re-verified
- Ownership database (`access_auth.sqlite`) is a single point of failure
- `new_owner` validation in `shed_group_chown` blocks dangerous characters but accepts any string up to 255 chars
- If OpenWebUI Groups API fails, errors are sometimes silently swallowed
- No audit logging for group file access

**Assessment:** ★★★★☆

---

## Analysis Axis 10: Git Integration Security

### Git Security Model

**Key Strengths:**
- **Hook neutralization**: `_neutralize_git_hooks()` removes hooks after clone and sets `core.hooksPath=/dev/null`
- **Subcommand whitelisting**: Only approved git operations allowed
- **Network operation separation**: `clone`/`fetch`/`pull` require `safe`, `push` requires `all`
- **Blacklist for dangerous operations**: `gc`, `prune`, `filter-branch` always blocked

**Weaknesses/Risks:**
- **Hook race condition**: Between clone completion and `_neutralize_git_hooks()`, a malicious hook could execute
- Git configs from cloned repos could include dangerous settings (e.g., `core.fsmonitor`)
- Submodule operations could trigger nested clones with their own hooks
- Git LFS hooks are not specifically handled
- `.git/config` in cloned repos could contain credential helpers or other dangerous settings

**Assessment:** ★★★☆☆

---

## Analysis Axis 11: Integration with OpenWebUI Internals

### API Integration Model

**Key Strengths:**
- **Bridge pattern**: `_OpenWebUIBridge` isolates version-specific imports
- **Lazy initialization**: API only loaded when needed
- **Fallback paths**: Multiple import paths tried for different OpenWebUI versions
- **Direct API calls**: Avoids HTTP self-requests preventing deadlocks

**Weaknesses/Risks:**
- Hard-coded paths (`/app/backend/data/uploads`) assume specific deployment structure
- If OpenWebUI internal API changes, operations fail with generic errors
- File upload path (`uploads_dir`) written to without additional validation beyond existence check
- `shed_link_create` copies files to OpenWebUI uploads directory—permissions inheritance unclear
- No verification that OpenWebUI database operations actually succeeded beyond null check

**Assessment:** ★★★☆☆

---

## Analysis Axis 12: Resistance to LLM Misuse

### Prompt Injection and Hallucination Resistance

**Key Strengths:**
- Clear documentation in code header about callable functions
- Parameter validation catches most obviously invalid inputs
- Helpful error messages guide LLM back to correct usage
- `__user__` and `__metadata__` are OpenWebUI-injected, not user-controllable
- Valves are admin-only, not exposed to user or LLM manipulation

**Weaknesses/Risks:**
- An adversarial prompt could potentially convince the LLM to:
  - Use `allow_zone_in_path=True` inappropriately
  - Construct malicious regex patterns for `shed_patch_text`
  - Request `network_mode` changes (which would require admin action, but social engineering risk)
  - Chain operations in unexpected ways
- No operation rate limiting or anomaly detection
- Complex parameter combinations increase hallucination-induced misuse probability
- LLM could be confused by similar function names (e.g., `shed_patch_text` vs `shed_lockedit_overwrite`)

**Assessment:** ★★★☆☆

---

## Analysis Axis 13: Error Handling and Information Disclosure

### Error Model

**Key Strengths:**
- **Structured errors**: `StorageError` class with code, message, details, hint
- **Contextual help**: Error responses include relevant function documentation
- **Sanitized output**: Internal paths use relative representation where possible

**Weaknesses/Risks:**
- Bare `except Exception:` blocks return generic messages hiding root cause
- Some stack traces could leak to stderr in subprocess operations
- File existence checks reveal path validity (oracle for enumeration)
- Error details sometimes include absolute paths (`storage_base_path`)
- Database errors return generic messages potentially hiding security-relevant failures

**Assessment:** ★★★☆☆

---

## Valve Security Analysis

All valves are **admin-controlled only** and cannot be modified by users or the LLM. This is a critical security property.

| Valve | Security Impact | Risk Level |
|-------|-----------------|------------|
| `storage_base_path` | Defines where all data lives; misconfiguration could expose system directories | High |
| `network_mode` | `all` enables data exfiltration | **Critical** |
| `max_file_size_mb` | DoS protection; too high allows resource exhaustion | Medium |
| `quota_per_user_mb` / `quota_per_group_mb` | Storage DoS protection | Medium |
| `exec_timeout_max` | Prevents infinite commands; too high enables DoS | Medium |
| `exec_memory_limit_mb` / `exec_cpu_limit_seconds` | Process resource limits | Medium |
| `sqlite_readonly` | When `True`, prevents database modification | High |
| `openwebui_api_url` | Misconfiguration could cause API calls to wrong server | Low |

---

## Final Overall Assessment

### Weighted Criteria

| Axis | Weight | Score | Weighted |
|------|--------|-------|----------|
| Architecture | 8% | 4/5 | 0.32 |
| LLM Attack Surface | 12% | 3/5 | 0.36 |
| Filesystem Isolation | 15% | 4/5 | 0.60 |
| Command Execution | 18% | 3/5 | 0.54 |
| Network Security | 15% | 3/5 | 0.45 |
| Concurrency/Locking | 6% | 4/5 | 0.24 |
| Data Processing | 8% | 3/5 | 0.24 |
| ZIP Security | 5% | 4/5 | 0.20 |
| Group Access | 5% | 4/5 | 0.20 |
| Git Security | 4% | 3/5 | 0.12 |
| OpenWebUI Integration | 2% | 3/5 | 0.06 |
| LLM Misuse Resistance | 2% | 3/5 | 0.06 |

**Total Weighted Score: 3.39 / 5**

---

## Final Rating: ★★★☆☆ (3.5 stars)

### Justification

Fileshed demonstrates **above-average security awareness** for a tool of its complexity. The implementation includes multiple defense layers: command whitelisting, path validation, network access controls, resource limits, and structured error handling. The architecture properly separates internal methods from the LLM-accessible API, and valves are correctly positioned as admin-only controls.

However, the **extensive attack surface** (80+ commands, 40+ public methods, file/network/database access combined) introduces inherent risk. The `network_mode=all` configuration enables data exfiltration by design. Command execution validation, while comprehensive, cannot guarantee coverage of all edge cases across dozens of Unix utilities. The Git hook neutralization has a theoretical race window. SQLite access with `sqlite_readonly=False` is permissive.

**Recommendation for deployment:**
1. **Set `network_mode=disabled` or `safe`** unless download/upload functionality is explicitly required
2. **Enable `sqlite_readonly=True`** for untrusted users
3. Consider **reducing the command whitelist** to only necessary commands for the deployment context
4. Implement **external monitoring/logging** for command execution patterns
5. Regularly **audit group membership** and file ownership records

The tool is **suitable for controlled environments** with trusted users and proper administrative configuration, but should not be deployed with default settings in high-security contexts.
