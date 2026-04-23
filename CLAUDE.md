# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Development repo for adding Azure NetApp Files (ANF) reporting to the [AsBuiltReport.Microsoft.Azure](https://github.com/AsBuiltReport/AsBuiltReport.Microsoft.Azure) module. This is **not** a standalone module — code here will be contributed upstream via PR to `AsBuiltReport/AsBuiltReport.Microsoft.Azure` (dev branch).

| Item | Detail |
|------|--------|
| GitHub repo | `https://github.com/cse-gh/Get-AbrAzNetAppFiles` (primary) |
| Gitea mirror | `https://git.ixian.co/ixian/Get-AbrAzNetAppFiles` |
| Upstream | `https://github.com/AsBuiltReport/AsBuiltReport.Microsoft.Azure` |
| PR target | `dev` branch |
| Current version | **0.1.0** |
| PowerShell | 7+ required (`pwsh`, not `powershell.exe`) |

## Commands

```powershell
# Lint
Invoke-ScriptAnalyzer -Path .\Src\ -Recurse -ReportSummary

# Test module loads (after copying files to installed module path)
Import-Module AsBuiltReport.Microsoft.Azure -Force

# Generate test report (requires Azure auth)
$cred = Get-Credential
New-AsBuiltReport -Report Microsoft.Azure -Target '<subscription-id>' -Credential $cred -MFA -Format HTML -OutputFolderPath .\TestOutput

# Install modules (use AllUsers to avoid OneDrive sync issues on Windows)
Install-Module <name> -Scope AllUsers
```

## Architecture

### Single Consolidated Function

The repo contains one function — `Get-AbrAzNetAppFiles` (`Src/Private/Get-AbrAzNetAppFiles.ps1`) — that covers all ANF resource types: NetApp Accounts, Capacity Pools, Volumes, Snapshots, Snapshot Policies, and Backup Policies. This is a deliberate design choice; the function will be added to the upstream module's `Src/Private/` directory.

### Internationalization

All user-facing strings are externalized in `Language/<culture>/MicrosoftAzure.psd1` files (en-US, en-GB, es-ES, fr-FR, de-DE). Each file contains a `GetAbrAzNetAppFiles` key accessed via:

```powershell
$LocalizedData = $reportTranslate.GetAbrAzNetAppFiles
Section -Style Heading4 $LocalizedData.Heading { ... }
```

**Never hardcode user-facing text.** All labels, headings, status values, and warning messages must use `$LocalizedData.*` lookups. When adding new strings, add them to all five language files.

### InfoLevel Tiered Rendering

Content is gated by `$InfoLevel.NetAppFiles` (0=off, 1-4 = increasing detail):

| Level | Rendering |
|-------|-----------|
| 1 | Summary tables for accounts, pools, volumes |
| 2 | Per-account vertical detail (AD + encryption); per-pool vertical detail with utilization; volumes grouped by pool |
| 3 | Per-volume vertical detail (protocols, mount targets, quota, data protection, export policy); snapshot & backup policy summary tables |
| 4 | Per-volume snapshot list; volume quota rules |

### Health Checks (all implemented)

| Check | Condition | Style | Config key |
|-------|-----------|-------|------------|
| Pool Near Capacity | UsedSize / Size > 0.85 | Warning | `PoolCapacity` |
| Volume Near Quota | UsedSize / UsageThreshold > 0.85 | Warning | `VolumeQuota` |
| AD Join Unhealthy | ActiveDirectories[].Status != "InUse" | Warning | `AdHealth` |
| No Snapshot Policy | Volume.DataProtection.Snapshot.SnapshotPolicyId is null | Info | `SnapshotPolicy` |
| Platform-Managed Key | Account.Encryption.KeySource == "Microsoft.NetApp" | Info | `CustomerManagedKey` |
| No Backup Protection | Volume.DataProtection.Backup.BackupPolicyId is null | Info | `BackupProtection` |

All gated by `$Healthcheck.NetAppFiles.<key>`.

## Get-AbrAz* Function Pattern

Every resource function in the upstream module follows this structure — **new code must conform**:

```powershell
function Get-AbrAzResourceType {
    [CmdletBinding()]
    param ()
    begin {
        Write-PScriboMessage "ResourceType InfoLevel set at $($InfoLevel.ResourceType)."
    }
    process {
        Try {
            if ($InfoLevel.ResourceType -gt 0) {
                $resources = Get-AzSomeResource | Sort-Object Name
                if ($resources) {
                    Write-PscriboMessage "Collecting Azure ResourceType information."
                    Section -Style Heading4 'Resource Type' {
                        # Build [Ordered]@{} PSCustomObject array
                        # Render via Table @TableParams
                    }
                }
            }
        } Catch {
            Write-PScriboMessage -IsWarning $($_.Exception.Message)
        }
    }
    end {}
}
```

**Critical rules:**
- Check `$InfoLevel` before executing (0 = disabled)
- No parameters — all context from global variables set by the orchestrator
- Never return values — render directly via PScribo (`Section`, `Paragraph`, `Table`, `BlankLine`)
- Wrap in Try/Catch with `Write-PScriboMessage -IsWarning`
- Use `[Ordered]@{}` for predictable column order
- Cache API calls when iterating (avoid Azure throttling)

## Global Variables (set by orchestrator)

| Variable | Type | Purpose |
|----------|------|---------|
| `$InfoLevel` | Hashtable | Detail level per resource (0=off, 1-4) |
| `$Options` | Hashtable | `ShowSectionInfo`, `ShowTags` |
| `$Report` | Hashtable | `ShowTableCaptions` |
| `$Healthcheck` | Hashtable | Boolean flags per resource type |
| `$AzSubscription` | Object | Current subscription being processed |
| `$AzLocationLookup` | Hashtable | Location code → display name |
| `$AzSubscriptionLookup` | Hashtable | Subscription ID → name |
| `$reportTranslate` | Hashtable | Localized string data per function |

## PScribo Quick Reference

```powershell
Section -Style Heading4 'Title' { ... }                    # TOC section
Section -Style NOTOCHeading5 -ExcludeFromTOC 'Title' { }   # Detail section (no TOC)
Paragraph "text"                                            # Text block
BlankLine                                                   # Spacing
$data | Table @TableParams                                  # Render table
$data | Set-Style -Style Warning -Property 'Status'         # Health check highlighting
Write-PScriboMessage "msg"                                  # Console logging (not in report)
```

Table config: `List = $false` for horizontal summary, `List = $true` for vertical key-value (use `ColumnWidths = 40, 60`). `ColumnWidths` must sum to 100.

## ANF-Specific Gotchas

- **Resource names are compound**: `Get-AzNetAppFilesPool` returns `.Name` as `"account/pool"`, volumes return `"account/pool/volume"`. Use `.Name.Split('/')[-1]` for the display name and `.Id.Split('/')[4]` for the resource group.
- **Sizes are in bytes**: `Size`, `UsageThreshold`, `UsedSize`, `ProvisionedAvailabilityZone` free-vs-total — all bytes. Convert with `/1GB` or `/1TB`.
- **Protocol types is an array**: `$volume.ProtocolTypes` is `string[]`; join with `, ` for display.
- **Policy references are resource IDs**: `DataProtection.Snapshot.SnapshotPolicyId` and `DataProtection.Backup.BackupPolicyId` are full ARM IDs — extract policy name with `.Split('/')[-1]`. `$null` means no policy attached.
- **Active Directory lives on the account**: `$account.ActiveDirectories` is an array; a healthy join has `Status = "InUse"`. An account may have zero AD configs (NFS-only volumes).
- **Mount targets are on the volume**: `$volume.MountTargets[].IpAddress` / `.SmbServerFqdn`.

## Integration Checklist (for upstream PR)

1. Copy `Src/Private/Get-AbrAzNetAppFiles.ps1` to module's `Src/Private/`
2. Merge `GetAbrAzNetAppFiles` string block from each `Language/<culture>/MicrosoftAzure.psd1` into parent module's language files
3. Add `"NetAppFiles" = "Get-AbrAzNetAppFiles"` to `$SectionFunctionMap` in orchestrator
4. Add `"NetAppFiles"` to `$DefaultSectionOrder` in orchestrator
5. Add `NetAppFiles` entries to module JSON: `SectionOrder`, `InfoLevel`, `HealthCheck`

## Parent Module Reference

Key reference files in the upstream `AsBuiltReport.Microsoft.Azure` module:
- `Src/Public/Invoke-AsBuiltReport.Microsoft.Azure.ps1` — orchestrator
- `Src/Private/Get-AbrAzAvailabilitySet.ps1` — simplest function (InfoLevel 1 only)
- `Src/Private/Get-AbrAzVirtualMachine.ps1` — complex example (nested resources, InfoLevel 2+, health checks)
- `Src/Private/Get-AbrAzDesktopVirtualization.ps1` — closest structural analog (multi-tier, multiple resource types in one function)
