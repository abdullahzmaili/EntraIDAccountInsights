# Entra ID Account Insights – Detailed Instructions

This guide explains how to set up, run, configure, and interpret the results of `EntraIDAccountInsights.ps1`. For a fast start, see `QUICKSTART.md`. For a high-level overview, see `README.md`.

---

## 1. Prerequisites

### 1.1 Software

| Requirement | Notes |
|-------------|-------|
| Windows PowerShell 5.1+ | Bundled with Windows 10/11 and Windows Server. |
| PowerShell 7+ (optional) | Required only for `-ParallelAnalysis`. Install from the Microsoft PowerShell releases. |
| `Microsoft.Graph.Authentication` | Auto-installed on first run. |
| `Microsoft.Graph.Reports` | Auto-installed on first run. |
| `ExchangeOnlineManagement` | Auto-installed when unified audit logs / mailbox settings are requested. |

The script calls `Install-Module ... -Scope CurrentUser -Force -AllowClobber` for any missing modules, so you need permission to install modules for your user profile.

### 1.2 Accounts & Permissions

You need **two identities** (they can be the same account if it has all rights):

1. **Admin account** (`-EntraIDAdmin`) used to authenticate to Microsoft Graph and, optionally, Exchange Online / Security & Compliance.
2. **Target user** (`-UserToAnalyze`) — the account you want to investigate.

Delegated Microsoft Graph scopes requested at sign-in:

- `AuditLog.Read.All`
- `Directory.Read.All`
- `SecurityAlert.Read.All`
- `UserAuthenticationMethod.Read.All`

For unified audit logs and mailbox settings, the admin account must be able to connect to Exchange Online and the Security & Compliance (IPPS) endpoint, and unified audit logging must be enabled in the tenant.

### 1.3 Execution policy

If script execution is blocked, allow it for the current session only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

---

## 2. Running the Script

### 2.1 Interactive mode

```powershell
.\EntraIDAccountInsights.ps1
```

You will be prompted for:

- Output folder (created if it doesn't exist).
- Target user UPN.
- Number of days to search (default 30).
- Working hours (`HH:mm` start/end) used for off-hours detection.
- Whether to open the report at the end.

### 2.2 Non-interactive mode

Provide `-UserToAnalyze` (and typically `-EntraIDAdmin` and `-Output`) to skip most prompts:

```powershell
.\EntraIDAccountInsights.ps1 `
    -EntraIDAdmin admin@contoso.onmicrosoft.com `
    -UserToAnalyze user@contoso.com `
    -Output C:\Investigations\user `
    -StartDate '2025-12-01' -EndDate '2025-12-31' `
    -StartTime '09:00' -EndTime '17:00' `
    -Open
```

### 2.3 Authentication flow

1. The script ensures required modules are present.
2. It checks for an existing Graph session. If the session is missing required scopes, it reconnects.
3. An interactive sign-in window prompts for the admin credentials and consent.
4. After Phase 1 completes, **all cloud connections are closed** and Phase 2 runs offline.

---

## 3. Parameter Reference

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `-Output` | No | Prompted | Destination folder for CSVs and the HTML report. |
| `-Open` | No | Off | Open the HTML report automatically when done. |
| `-EntraIDAdmin` | No | Current session / prompted | Admin UPN for interactive authentication. |
| `-UserToAnalyze` | No | Prompted | Target user UPN to analyze. |
| `-StartTime` | No | Prompted | Working-hours start (`HH:mm`). Pair with `-EndTime`. |
| `-EndTime` | No | Prompted | Working-hours end (`HH:mm`). Must differ from start. |
| `-StartDate` | No | None | Log date-range start (date string or Unix epoch seconds/ms). |
| `-EndDate` | No | None | Log date-range end. |
| `-daysToSearch` | No | 30 | Days of history to retrieve. |
| `-ExcludeUnifiedAudit` | No | Off | Skip unified audit logs (no Exchange/Compliance connection). |
| `-ParallelAnalysis` | No | Off | Concurrent analysis (PowerShell 7+ only). |
| `-ConfigPath` | No | `default-config.json` | Path to a custom threshold configuration file. |
| `-LoadFunctionsOnly` | No | Off | Internal switch: dot-source functions without executing the tool. |

**Notes**

- `-StartTime`/`-EndTime` are hour-granular; the minutes are informational.
- Sign-in and Defender queries honor `-daysToSearch` with no 30-day cap; directory audit logs are limited by tenant retention.
- If `-UserToAnalyze` is supplied, the menu banner is skipped and analysis runs directly.

---

## 4. Configuration File (`default-config.json`)

Place `default-config.json` beside the script, or point to a custom file with `-ConfigPath`. Values not present in the file fall back to the built-in defaults below.

### 4.1 Detection thresholds (built-in defaults)

| Setting | Default | Meaning |
|---------|---------|---------|
| `BruteForceThreshold` | 10 | Failures (error 50126) within the window to flag brute-force. |
| `BruteForceWindowMinutes` | 10 | Sliding window for brute-force. |
| `PasswordSprayThreshold` | 10 | Failures within the window to flag password-spray. |
| `PasswordSprayWindowMinutes` | 60 | Sliding window for password-spray. |
| `AccountLockoutThreshold` | 3 | Lockout events (error 50053) to flag. |
| `FailedSignInsThreshold` | 10 | Failed/interrupted sign-ins to flag. |
| `OffHoursThreshold` | 10 | Off-hours successful sign-ins to flag. |
| `MultipleIPsThreshold` | 3 | Unique IPs in 24h to flag. |
| `MultipleLocationsThreshold` | 2 | Distinct locations baseline. |
| `MultipleCountriesThreshold` | 3 | Distinct countries within a 24h window to flag. |
| `MultipleDevicesThreshold` | 3 | Distinct operating systems to flag. |
| `AuditHighRiskThreshold` | 4 | High-risk audit events to flag. |
| `AuditBulkThreshold` | 10 | Bulk audit events to flag. |
| `IPWindowHours` | 24 | Window for multiple-IP analysis. |
| `PasswordChangeWindowMinutes` | 5 | Correlate sign-in after password change. |
| `ImpossibleTravelSpeedKmh` | 900 | Speed threshold for impossible travel. |
| `ImpossibleTravelMaxHours` | 12 | Max time between sign-ins to evaluate travel. |
| `MassDownloadHourThreshold` | 50 | File downloads in 1h to flag. |
| `MassDownload24hThreshold` | 200 | File downloads in 24h to flag. |
| `MassEmailDeleteHourThreshold` | 50 | Email deletions in 1h to flag. |
| `MailItemsAccessedThreshold` | 100 | MailItemsAccessed bind events to flag. |

### 4.2 Graph API retry settings

| Setting | Default | Meaning |
|---------|---------|---------|
| `GraphMaxRetries` | 3 | Retries on HTTP 429/503/504. |
| `GraphInitialRetryDelaySec` | 2 | Initial backoff delay. |
| `GraphMaxRetryDelaySec` | 60 | Maximum backoff delay. |

### 4.3 Example JSON structure

```json
{
  "detection": {
    "bruteForce":        { "threshold": 10, "windowMinutes": 10 },
    "passwordSpray":     { "threshold": 10, "windowMinutes": 60 },
    "impossibleTravel":  { "maxSpeedKmh": 900, "maxHours": 12 },
    "massDownload":      { "hourlyThreshold": 50, "dailyThreshold": 200 },
    "massEmailDeletion": { "hourlyThreshold": 50 },
    "mailItemsAccessed": { "threshold": 100 }
  },
  "analysis": {
    "multipleLocationThreshold": 2,
    "multipleIPWindowHours": 24,
    "passwordChangeWindowMinutes": 5
  },
  "graphApi": {
    "maxRetries": 3,
    "initialRetryDelaySeconds": 2,
    "maxRetryDelaySeconds": 60
  }
}
```

---

## 5. Output & Report

### 5.1 CSV exports

Written to `-Output` with a shared `yyyyMMdd_HHmmss` timestamp. Depending on data availability, you may see:

- `Signinlogs_<user>_<ts>.csv`
- `AzureAD_DirectoryAuditLogs_<user>_<ts>.csv`
- `DefenderAlerts_<user>_<ts>.csv`
- `UnifiedLogs_<user>_<ts>.csv`
- `EmailForwarding_<user>_<ts>.csv`, `InboxRules_<user>_<ts>.csv`, `JunkEmailConfiguration_<user>_<ts>.csv`, `MailboxProtocols_<user>_<ts>.csv`
- `SendAs_<user>_<ts>.csv`, `FullAccess_<user>_<ts>.csv`
- `EmailSendandDeleteEvents_<user>_<ts>.csv`, `MailItemsAccessed_<user>_<ts>.csv`
- `MFADetails_<user>_<ts>.csv`
- `RolesAndGroupsMembership_<user>_<ts>.csv`

### 5.2 HTML report

`Entra_ID_Account_Insights_Report_<ts>.html` is self-contained and includes:

- **Executive Summary** with flagged/clean/total indicator counts and top findings.
- **Sign-In / Audit / Unified Audit metric** cards (each opens a detail modal).
- **Account & Email Settings** metrics (MFA, mailbox protocols, forwarding, rules, permissions).
- **Microsoft Defender Alerts** (all and affected-user views).
- **Sign-In Indicators** and **Audit Indicators** of suspicious behavior.
- **MITRE ATT&CK** mapping matrix with per-technique modals.
- **Security Recommendations** filtered by priority (Critical/High/Medium/Low).

Report interactions: click stat cards/indicators to open modals, sort tables by clicking headers, export any indicator's data to CSV, and use the sidebar to jump between sections. The report is print-friendly.

### 5.3 Re-running analysis offline

Because Phase 2 reads only from CSVs, you can re-generate the report from a prior export folder without reconnecting — useful for tuning thresholds via `-ConfigPath` or re-reviewing evidence.

---

## 6. Interpreting Indicators

- **Attention Needed = Yes** means the indicator's threshold was met — it warrants review, not an automatic conclusion of compromise.
- Correlate multiple indicators (e.g., Impossible Travel + Token Theft + Mailbox Forwarding Rule) for stronger signal.
- Use the exported CSVs to pivot on IPs, applications, sessions, and timestamps.
- Review the **Recommendations** section and apply relevant controls (MFA, Conditional Access, token protection, etc.).

---

## 7. Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Module install fails | Run PowerShell as your user with internet access; ensure `-Scope CurrentUser` install is permitted. |
| Missing scopes / re-auth loop | Consent to **all** requested permissions at sign-in; an admin may need to grant consent. |
| No sign-in logs returned | No activity in range, logs purged, or insufficient audit retention. |
| Defender alerts skipped | `SecurityAlert.Read.All` not consented, or feature unavailable in tenant. |
| Unified audit empty / errors | Exchange/Compliance connection failed, unified audit logging disabled, or Gateway timeouts (the script retries automatically). |
| `-ParallelAnalysis` ignored | Requires PowerShell 7+ (`pwsh`). On 5.1 it runs sequentially. |
| Throttling (HTTP 429) | The script honors `Retry-After` and backs off automatically. |

---

## 8. Security Notes

- Interactive delegated auth; no secrets stored in plaintext.
- UPN and output-path inputs are validated (traversal-safe).
- HTML output is encoded to reduce XSS risk.
- Sensitive fields (MFA secrets, Temporary Access Pass) are redacted in CSVs.
- Treat all exports as sensitive; store and share them per your data-handling policy.

---

## 9. Disclaimer

This tool is for security monitoring and initial analysis only and does not replace a formal incident investigation. If indicators of compromise are found, engage qualified SOC/security professionals. Use is at your own risk.
