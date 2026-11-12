# Entra ID Conditional Access Baseline

A layered Conditional Access baseline for Microsoft 365 tenants, built and operated across a multi-tenant MSP client portfolio. Policies are grouped by control area and written as Microsoft Graph `conditionalAccessPolicy` JSON bodies.

## Policy Index

Prefixes: **ID** identity and authentication, **DV** device trust, **NW** network and location, **SS** session lifetime, **RK** risk-based (Entra ID P2).

| ID | Policy | Control |
|---|---|---|
| ZT-DV01 | Windows: Managed Devices Only | Block, device filter |
| ZT-DV02 | macOS: Registered Devices Only Off-Network | Block, device filter |
| ZT-DV03 | Windows: No Browser Access from Unmanaged PCs | Block, device filter |
| ZT-DV04 | Deny Unrecognized Device Platforms | Block |
| ZT-ID01 | Deny Legacy Protocol Sign-ins | Block |
| ZT-ID02 | Privileged Admins: Fresh MFA on Every Sign-in | MFA strength, every-time frequency |
| ZT-ID03 | Baseline MFA for Workforce Identities | MFA strength |
| ZT-ID04 | MFA Gate for Azure Resource Management | MFA strength |
| ZT-ID05 | MFA Gate for Security Info Changes | MFA strength (user action) |
| ZT-ID06 | Device Join and Registration: MFA Unless on Trusted Network | MFA strength (user action) |
| ZT-ID07 | External Identities Must Use MFA | MFA |
| ZT-NW01 | Geo-Fence: Deny Sign-ins from Non-Approved Countries | Block, named location |
| ZT-RK01 | Risky Sign-ins: Step-Up MFA | MFA strength (Entra ID P2) |
| ZT-RK02 | High-Risk Users: Forced Remediation | Risk remediation (Entra ID P2) |
| ZT-SS01 | Contractor Sessions Expire Daily | MFA strength, 1-day frequency |
| ZT-SS02 | Unmanaged Devices: Daily Browser Sessions, No Persistence | Session |
| ZT-SS03 | Unmanaged PCs: Daily Desktop App Reauthentication | MFA, 1-day frequency |
| ZT-SS04 | Mobile Apps: Weekly Reauthentication | Session |
| ZT-SS05 | Trusted Network and Managed Device: 30-Day Persistent Sessions | Session |
| ZT-SS06 | Off-Network: Weekly Reauthentication | MFA strength, 7-day frequency |

## Placeholders

Replace before import. Graph requires object IDs, not display names.

| Placeholder | Replace with |
|---|---|
| `{{BREAK_GLASS_GROUP_ID}}` | Emergency access accounts group (excluded from every policy) |
| `{{ALL_USERS_GROUP_ID}}` | Licensed users group |
| `{{CONTRACTORS_GROUP_ID}}` | Contractor accounts group |
| `{{P2_RISK_USERS_GROUP_ID}}` | Users licensed for Entra ID P2 risk policies |
| `{{OFFICE_NAMED_LOCATION_ID}}` | ID of the location created from `named-locations/office-trusted-ip.json` |
| `{{ALLOWED_COUNTRIES_NAMED_LOCATION_ID}}` | ID of the location created from `named-locations/allowed-countries.json` |
| `{{OFFICE_PUBLIC_IP}}` | Office egress IP |

Well-known IDs kept as-is: `62e90394-69f5-4237-9190-012177145e10` (Global Administrator role template), `797f4846-ba00-4fd7-ba43-dac1f8f63013` (Windows Azure Service Management API), `00000002-0000-0ff1-ce00-000000000000` (Office 365 Exchange Online), `00000000-0000-0000-0000-000000000002` (built-in MFA authentication strength).

> [!NOTE]
> **Design trade-off: administrator network exception (ZT-DV01, ZT-DV03)**
> Both policies exclude a single trusted location: the managing provider's office network. This lets technicians administer client tenants from provider workstations, which the client tenant sees as unmanaged devices. The exception applies only to sign-ins coming from that office IP. The provider's own tenant separately blocks personal and unmanaged devices, so a personal device on the office network can't authenticate to the provider's admin tooling either. For tighter scoping, the exclusion could be limited to a dedicated admin group or replaced with device-based trust for privileged roles.

## Deployment

All templates ship as `enabledForReportingButNotEnforced`. Create named locations first, replace placeholders, import, then review sign-in logs in report-only mode before enforcing.

```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.ConditionalAccess","Policy.Read.All"
Get-ChildItem .\policies\*.json | ForEach-Object {
    Invoke-MgGraphRequest -Method POST `
        -Uri "https://graph.microsoft.com/beta/identity/conditionalAccess/policies" `
        -Body (Get-Content $_.FullName -Raw) -ContentType "application/json"
}
```

> Verify break-glass accounts can sign in before moving any policy out of report-only.
