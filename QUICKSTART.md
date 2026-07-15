# Quick Start – Entra ID Account Insights

Get from zero to a report in a few minutes. For full details see `INSTRUCTIONS.md`; for an overview see `README.md`.

---

## 1. Prerequisites (30 seconds)

- Windows PowerShell 5.1+ (PowerShell 7+ optional, for `-ParallelAnalysis`).
- An admin account that can read audit logs / Defender alerts, and (optionally) connect to Exchange Online.
- Internet access to install modules and reach Microsoft Graph.

Required Graph scopes (consented at sign-in): `AuditLog.Read.All`, `Directory.Read.All`, `SecurityAlert.Read.All`, `UserAuthenticationMethod.Read.All`.

> Modules (`Microsoft.Graph.Authentication`, `Microsoft.Graph.Reports`, `ExchangeOnlineManagement`) are installed automatically on first run.

---

## 2. Allow the script to run (once per session)

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

---

## 3. Run it

### Easiest – interactive

```powershell
.\EntraIDAccountInsights.ps1
```

Answer the prompts (target user, output folder, days to search, working hours) and sign in when the window appears.

### Common – one line, opens the report

```powershell
.\EntraIDAccountInsights.ps1 `
    -EntraIDAdmin admin@contoso.onmicrosoft.com `
    -UserToAnalyze user@contoso.com `
    -Output C:\out `
    -Open
```

### Faster – skip unified audit logs (no Exchange connection)

```powershell
.\EntraIDAccountInsights.ps1 -UserToAnalyze user@contoso.com -Output C:\out -ExcludeUnifiedAudit -Open
```

### With a specific date range and working hours

```powershell
.\EntraIDAccountInsights.ps1 `
    -EntraIDAdmin admin@contoso.onmicrosoft.com `
    -UserToAnalyze user@contoso.com -Output C:\out `
    -StartDate '2025-12-01' -EndDate '2025-12-31' `
    -StartTime '09:00' -EndTime '17:00' -Open
```

---

## 4. What you get

In your `-Output` folder (timestamped):

- Raw CSVs: sign-in logs, directory audit logs, Defender alerts, unified audit logs, MFA, roles/groups, mailbox settings.
- **`Entra_ID_Account_Insights_Report_<timestamp>.html`** – the interactive report.

Open the HTML file to see:

- Executive summary (flagged vs. clean indicators)
- Sign-In / Audit / Unified Audit metrics
- Suspicious-behavior indicators + MITRE ATT&CK mapping
- Microsoft Defender alerts
- Prioritized security recommendations

---

## 5. Handy flags

| Flag | What it does |
|------|--------------|
| `-Open` | Opens the HTML report when finished. |
| `-ExcludeUnifiedAudit` | Skips Exchange/Compliance; faster run. |
| `-daysToSearch 60` | Look back 60 days (default 30). |
| `-ParallelAnalysis` | Faster analysis on PowerShell 7+ (`pwsh`). |
| `-ConfigPath .\my-config.json` | Use custom detection thresholds. |

---

## 6. Tips

- Re-running with a different `-ConfigPath` re-analyzes offline from existing CSVs — no need to reconnect.
- "Attention Needed = Yes" means *review this*, not *confirmed compromise*. Correlate several indicators.
- Exports may contain sensitive data — store and share responsibly.

---

## 7. Quick troubleshooting

- **Re-auth loop / missing scopes:** consent to all requested permissions at sign-in.
- **No sign-in data:** widen `-daysToSearch` or check tenant log retention.
- **Defender alerts skipped:** `SecurityAlert.Read.All` wasn't granted.
- **`-ParallelAnalysis` ignored:** run under PowerShell 7+ (`pwsh`).

> This tool supports triage only and does not replace a formal incident investigation.
