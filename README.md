# Get-AbrAzNetAppFiles

Development repo for adding Azure NetApp Files (ANF) reporting to the [AsBuiltReport.Microsoft.Azure](https://github.com/AsBuiltReport/AsBuiltReport.Microsoft.Azure) module. The code here will be contributed upstream via PR.

## Current Version: v0.1.0

Single consolidated function `Get-AbrAzNetAppFiles` covering ANF Accounts, Capacity Pools, Volumes, Snapshots, Snapshot Policies, and Backup Policies with three InfoLevel tiers:

| InfoLevel | Content |
|-----------|---------|
| **1** | Summary tables for NetApp Accounts, Capacity Pools, and Volumes |
| **2** | Per-account detail (Active Directory joins, encryption config); per-pool detail (service level, QoS, size, utilization); volumes grouped by pool |
| **3** | Per-volume vertical detail (protocols, mount targets, quota, snapshot/backup policy, export policy rules, throughput), plus snapshot & backup policy summary tables |
| **4** | Per-volume snapshots list, volume quota rules |

### Health Checks

| Check | Condition | Style |
|-------|-----------|-------|
| Pool Near Capacity | Utilized > 85% of pool size | Warning |
| Volume Near Quota | Usage > 85% of usage threshold | Warning |
| AD Join Unhealthy | Account has AD config with Status != "InUse" | Warning |
| No Snapshot Policy | Volume has no snapshot policy attached | Info |
| Platform-Managed Key | Account encrypted with Microsoft.NetApp key (not customer-managed) | Info |
| No Backup Protection | Volume has no backup policy attached | Info |

### Azure Cmdlets Used

- `Get-AzNetAppFilesAccount`, `Get-AzNetAppFilesActiveDirectory`
- `Get-AzNetAppFilesPool`
- `Get-AzNetAppFilesVolume`
- `Get-AzNetAppFilesSnapshot`, `Get-AzNetAppFilesSnapshotPolicy`
- `Get-AzNetAppFilesBackupPolicy`
- `Get-AzNetAppFilesVolumeQuotaRule`

### Not yet covered (deferred to a future release)

- Volume Groups (SAP HANA)
- Subvolumes
- Backup Vaults (new feature)
- Cross-region replication (CRR) relationships

## Language Support

This module supports the AsBuiltReport internationalization framework (v1.5.0+). Language strings are externalized in `Language/<culture>/MicrosoftAzure.psd1`.

**Supported languages:**

| Culture | Language |
|---------|----------|
| en-US | English (US) |
| en-GB | English (UK) |
| es-ES | Spanish (Spain) |
| fr-FR | French (France) |
| de-DE | German (Germany) |

Non-English cultures ship with English placeholder strings pending native translation. Translations welcome via PR. Regional variants (e.g., es-MX, fr-CA) will automatically fall back to their base language.

When contributing to the upstream module, merge the `GetAbrAzNetAppFiles` section from each language file into the parent module's corresponding `Language/<culture>/MicrosoftAzure.psd1`.

## Integration

To integrate into the installed `AsBuiltReport.Microsoft.Azure` module:

1. Copy `Src/Private/Get-AbrAzNetAppFiles.ps1` to module's `Src/Private/`
2. Merge language strings from each `Language/<culture>/MicrosoftAzure.psd1` into parent module's corresponding language files
3. Add `"NetAppFiles" = "Get-AbrAzNetAppFiles"` to `$SectionFunctionMap` in the orchestrator
4. Add `"NetAppFiles"` to `$DefaultSectionOrder` in the orchestrator
5. Add `NetAppFiles` entries to module JSON: `SectionOrder`, `InfoLevel`, `HealthCheck`

## Requirements

- PowerShell 7+ (`pwsh`)
- `Az.NetAppFiles` module (>= 1.0.0)
- `AsBuiltReport.Microsoft.Azure` v0.2.0+
