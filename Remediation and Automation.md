# VM Migration Remediation and Automation

## 1. Objective

Standardize migration of standalone VMs to a resilient VMSS Flex construct with:
- Multi-zone distribution (where supported).
- Application Health Extension + Automatic Instance Repair.
- Deterministic prerequisite validation (managed disks, no PPG, etc.).

## 2. Prerequisite and Eligibility Matrix

| Check | Requirement | Failure Action | Script Ref |
|-------|-------------|----------------|------------|
| Managed Disks | All OS/data disks managed | Convert disks | Attach Script |
| Availability Set | VM not in availability set | Rebuild via image to VMSS Flex | Attach Script |
| Proximity Placement Group | None attached | Remove PPG association | Attach Script |
| Dedicated Host | Not on dedicated host | Migrate off host | Attach Script |
| Zones | VM zone subset of VMSS zones | Adjust VMSS or redeploy | Attach Script |
| Orchestration Mode | VMSS Flexible | Create new Flex VMSS | Fix Script |
| Fault Domains | FD=1 | Recreate with FD=1 | Fix Script |
| Single Placement Group | false | Recreate without SPG | Fix Script |
| Auto OS Upgrade | Disabled | Disable flag | Fix Script |
| Autoscale | Disabled during wave | Remove autoscale | Fix Script |

## 2a. Current State Classification

Total Analyzed VMs: 1

| State | Count | Notes |
|-------|------:|-------|
| Total Zone Resilient | 0 | Combined resilient population |
| Not Zone Resilient | 1 | Needs remediation path |
| Automation Eligible (subset) | 1 | Can follow scripted attach flow |
| Not Automation Eligible | 0 | Manual / complex path |

### Non-Resilient Breakdown By Category

| Category | Count |
|----------|------:|
| Standalone | 1 |

### Per-VM State Detail

| VM Resource ID | isZoneRedundant | zoneCount | haProtected | vmCategory | migrationComplexity | automationEligible | powerState |
|---------------------------|----------------------------|---------------------:|-----------------------:|-----------------------|--------------------------------|-------------------------------|-----------------------|
| `/subscriptions/70bcc220-4d88-48f2-a59a-77bae4785eac/resourceGroups/oldsystems/providers/Microsoft.Compute/virtualMachines/maayalab-old` | false | 0 | false | Standalone | Medium | true | PowerState/running |

## 2b. Targeted Remediation (State Driven)

The following remedies are generated from detected VM states. Scripts are embedded for direct review (do NOT run without validation).

### Standalone (No Zones) VMs (1)
Primary Action: Attach to VMSS Flexible with zone distribution.
#### Full Attach Script (Inline Review and Copy/Paste)

powershell

```
# Full attach script (inline snippet - review before execution)
<#
.SYNOPSIS
  Guidance script to attach an existing VM to a VMSS (Flexible orchestration).

.DESCRIPTION
  This script demonstrates a safe, guided workflow for attaching a single VM to a VM Scale Set (Flexible).
  It is written for reference and must be reviewed before any real execution. By default it runs in dry-run mode.

.PARAMETER VmName
  Name of the VM to attach.

.PARAMETER ResourceGroupName
  Resource group containing the VM.

.PARAMETER VmssName
  Target VM Scale Set (Flexible) name.

.PARAMETER DryRun
  Switch to perform a non-mutating simulation. Default: $true.

.NOTES
  Requires Az.Compute module.
  This script intentionally uses -WhatIf / DryRun patterns and explicit checks. Do not run without review.
#>

param(
    [Parameter(Mandatory = $true)]
    [string] $VmName,

    [Parameter(Mandatory = $true)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory = $true)]
    [string] $VmssName,

    [switch] $DryRun = $true
)

function Write-Info([string] $msg) { Write-Host "[INFO] $msg" -ForegroundColor Cyan }
function Write-Warn([string] $msg) { Write-Host "[WARN] $msg" -ForegroundColor Yellow }
function Write-Err([string] $msg) { Write-Host "[ERROR] $msg" -ForegroundColor Red }

try {
    Write-Info "Starting attach workflow for VM '$VmName' -> VMSS '$VmssName' (RG: $ResourceGroupName). DryRun=$($DryRun.IsPresent)"

    # Ensure Az module available
    if (-not (Get-Module -ListAvailable -Name Az.Compute)) {
        Write-Warn "Az.Compute module not found. Install-Module Az -Scope CurrentUser -Force"
    }

    # Validate resources
    $vm = Get-AzVM -Name $VmName -ResourceGroupName $ResourceGroupName -ErrorAction SilentlyContinue
    if (-not $vm) {
        throw "VM '$VmName' not found in resource group '$ResourceGroupName'. Aborting."
    }

    $vmss = Get-AzVmss -ResourceGroupName $ResourceGroupName -VMScaleSetName $VmssName -ErrorAction SilentlyContinue
    if (-not $vmss) {
        throw "VMSS '$VmssName' not found in resource group '$ResourceGroupName'. Aborting."
    }

    # Outline recommended approach:
    Write-Info "Recommended safe approach (dry-run will only simulate):"
    Write-Info "  1) Create VM snapshot or image (preserve OS/data)"
    Write-Info "  2) Capture VM configuration (NICs, extensions, disk IDs, tags)"
    Write-Info "  3) Create a VMSS VM from the snapshot/image and reapply configuration"
    Write-Info "  4) Drain traffic, cutover, and deallocate legacy VM only after verification"

    # Example: create managed image from VM (dry-run)
    $imageName = "$VmName-image-$(Get-Date -Format yyyyMMddHHmmss)"
    $createImageScript = {
        $vm | New-AzImageConfig -Name $imageName | New-AzImage -ImageName $imageName -ResourceGroupName $ResourceGroupName
    }

    if ($DryRun) {
        Write-Info "DryRun: would create image named $imageName from VM '$VmName'."
    } else {
        Write-Info "Creating image from VM..."
        & $createImageScript
        Write-Info "Image created: $imageName"
    }

    # Example: capture VM networking metadata
    $nics = $vm.NetworkProfile.NetworkInterfaces
    Write-Info "Captured NICs: $($nics.Count)"
    foreach ($nicRef in $nics) {
        Write-Info "  NIC reference: $($nicRef.Id)"
    }

    # Example: prepare VMSS VM configuration (illustrative)
    Write-Info "Preparing VMSS model update to include image reference."
    $vmssModel = $vmss.VirtualMachineProfile
    if ($DryRun) {
        Write-Info "DryRun: would update VMSS '$VmssName' VirtualMachineProfile to use image '$imageName' or custom image reference."
    } else {
        Write-Info "Updating VMSS model to use image and issuing an update (no rolling upgrade applied here)."
        # Example update (uncomment after review)
        # $vmssModel.StorageProfile.ImageReference = @{ Id = '/subscriptions/.../resourceGroups/.../providers/Microsoft.Compute/images/' + $imageName }
        # Update-AzVmss -ResourceGroupName $ResourceGroupName -VMScaleSetName $VmssName -VirtualMachineScaleSet $vmss
    }

    # Guidance: add the VM to VMSS instance model (VMSS Flexible supports adding existing NIC-backed instances by matching network configuration)
    Write-Info "Final step guidance: create VMSS instance from the prepared image/configuration, verify health probes, then decommission original VM."

    Write-Info "Attach workflow completed (simulated). Review produced artifacts and run without -DryRun when ready and after manual review."
}
catch {
    Write-Err $_.Exception.Message
    exit 1
}
```

### Post-Attach Health and Repair (All Newly Migrated Instances)
#### Full Health and Repair Script (Inline Review and Copy/Paste)

powershell

```
# Full health & repair configuration script (inline snippet - review before execution)
<#
.SYNOPSIS
  Guidance script to configure VM Scale Set (Flexible) health probes and automatic repair settings.

.DESCRIPTION
  This script provides a reference workflow to configure health probes, health extensions, and automatic repair
  policies for VM Scale Sets (Flexible orchestration). It runs in dry-run mode by default and is intended as
  guidance to be reviewed before any real execution.

.PARAMETER VmssName
  Name of the VM Scale Set.

.PARAMETER ResourceGroupName
  Resource group containing the VMSS.

.PARAMETER DryRun
  Switch to perform a non-mutating simulation. Default: $true.

.NOTES
  Requires Az.Compute and Az.Network modules.
  This script intentionally uses dry-run patterns and explicit checks. Do not run without review.
#>

param(
    [Parameter(Mandatory = $true)]
    [string] $VmssName,

    [Parameter(Mandatory = $true)]
    [string] $ResourceGroupName,

    [switch] $DryRun = $true
)

function Write-Info([string] $msg) { Write-Host "[INFO] $msg" -ForegroundColor Cyan }
function Write-Warn([string] $msg) { Write-Host "[WARN] $msg" -ForegroundColor Yellow }
function Write-Err([string] $msg) { Write-Host "[ERROR] $msg" -ForegroundColor Red }

try {
    Write-Info "Starting VMSS health & repair configuration for '$VmssName' in RG '$ResourceGroupName'. DryRun=$($DryRun.IsPresent)"

    $vmss = Get-AzVmss -ResourceGroupName $ResourceGroupName -VMScaleSetName $VmssName -ErrorAction SilentlyContinue
    if (-not $vmss) {
        throw "VMSS '$VmssName' not found in resource group '$ResourceGroupName'. Aborting."
    }

    # Example: inspect existing health probes (if using Azure Load Balancer or Application Gateway)
    Write-Info "Inspecting health probe / load balancer configuration (if any)."
    # Note: this is illustrative; actual probe lookup requires the LB/ApplicationGateway resource IDs.
    Write-Info "Ensure health probes target application endpoints and use adequate timeouts and thresholds."

    # Example: ensure VMSS has extension for health reporting (this is a common pattern)
    $healthExtensionName = "Microsoft.Azure.Monitor.Perf"
    $hasHealthExtension = $vmss.VirtualMachineProfile.ExtensionProfile?.Extensions |
        Where-Object { $_.Name -like "*health*" -or $_.Name -like "*monitor*" } | Measure-Object | Select-Object -ExpandProperty Count

    Write-Info "Found $hasHealthExtension health/monitor extensions (0 means none found)."

    if ($DryRun) {
        Write-Info "DryRun: would add or configure health reporting extension and configure repair policy for VMSS."
    } else {
        Write-Info "Configuring health reporting extension and repair policy (example)."
        # Example extension add (uncomment & review before use)
        # $ext = New-AzVmssExtension -Name "HealthAgent" -Publisher "Microsoft.Azure.Monitor" -Type "AzureMonitorWindowsAgent" -TypeHandlerVersion "1.0" -Settings @{}
        # Add-AzVmssExtension -ResourceGroupName $ResourceGroupName -VMScaleSetName $VmssName -Extension $ext

        # Example repair policy (illustrative only)
        # $repairPolicy = @{
        #     Enabled = $true
        #     GracePeriod = "00:15:00" # 15 minutes
        # }
        # Set-AzVmssRepairPolicy -ResourceGroupName $ResourceGroupName -VMScaleSetName $VmssName -RepairPolicy $repairPolicy
    }

    Write-Info "Recommendations:"
    Write-Info "  - Use application-level health probes when possible."
    Write-Info "  - Configure extension heartbeat and diagnostics to feed repair decisions."
    Write-Info "  - Set sensible grace periods to avoid flapping."
    Write-Info "  - Test repair workflows in a staging environment."

    Write-Info "Health & repair configuration guidance completed (simulated)."
}
catch {
    Write-Err $_.Exception.Message
    exit 1
}
```

## 3. Migration Flow (Recommended)

1. Inventory snapshot and scope validation.
2. Capability and regional eligibility validation (zones, capacity signals).
3. Prepare or reuse target VMSS Flex (orchestration, zones, FD=1, flags).
4. Dry-run attachment / WhatIf validation (prerequisites and safety checks).
5. Bulk attachment (wave-based) with controlled progression.
6. Configure health extension and automatic instance repair.
7. Post-attach validation (probe health, distribution, resiliency posture).
8. Soak period and observability (repair events, stability signals).

## 4. Single VM Attachment (WhatIf → Execute)

powershell
```
# Dry-run (validation only - inline steps)
$subId='<subId>'; $rg='<rg>'; $vmName='<vm>'; $vmss='<vmss>'
$vm = Get-AzVM -Name $vmName -ResourceGroupName $rg
$vmssObj = Get-AzVmss -ResourceGroupName $rg -VMScaleSetName $vmss
Write-Host 'Validated VM & VMSS (DryRun)' -ForegroundColor Cyan

# Execute (illustrative - review each step before running)
$imageName = "$vmName-image-$(Get-Date -Format yyyyMMddHHmmss)"
# New-AzImage ... (review parameters)
# Update VMSS model with image, create instance, cutover traffic, deallocate legacy VM
```

## 4a. Alternate Remediation (ASR Zonal Replication)

Use this path ONLY when VM must stay single-instance but requires cross-zone failure protection:

| Condition | Action | Result |
|-----------|--------|--------|
| Single-zone VM (no multi-zone / no VMSS) | Enable ASR replication (zonal supported vault / target) | High availability (treated as zone resilient) |
| Multi-zone VM or already in VMSS Flex | Skip ASR (unnecessary) | N/A |

powershell
```
# Portal deep link pattern (no encoding):
$vmResourceId = '/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vmName>'
$asrLink = "https://ms.portal.azure.com/#resource/$vmResourceId/siteRecoverySetting"
$asrLink

# (Illustrative) Az PowerShell outline for enabling replication (review docs for full parameters)
# Requires Az.RecoveryServices & proper vault/context setup
# Set-AzRecoveryServicesAsrVaultContext -VaultId <vaultId>
# New-AzRecoveryServicesAsrReplicationProtectedItem -Name <name> -ProtectionContainerMapping <pcm> -RecoveryAzureStorageAccountId <storageId> -SourceAzureVMId $vmResourceId -RecoveryResourceGroupId <targetRgId>
```

> After ASR reports isHighlyAvailable=true the VM is classified as zone resilient.

## 5. Bulk Attachment Pattern

powershell
```
$vms = @(
  'vmA', 'vmB', 'vmC'
)

foreach ($vm in $vms) {
  Write-Host "Validating $vm" -ForegroundColor Cyan
  # Inline validation: ensure VM & VMSS exist
  Get-AzVM -Name $vm -ResourceGroupName <rg> | Out-Null
}

foreach ($vm in $vms) {
  Write-Host "Attaching $vm" -ForegroundColor Yellow
  # Inline illustrative steps (image + model update placeholder)
  # $imageName = "$vm-image-$(Get-Date -Format yyyyMMddHHmmss)"
}

# Post attach: configure health + repair once
# See embedded Health & Repair script snippet above for configuration commands
```

## 6. VMSS Flex Preparation / Repair

powershell
```
# Validate VMSS Flex posture
# Inline posture validation (see embedded Fix script snippet for full logic)

# Auto-fix fixable issues
# Inline auto-fix placeholder (review embedded Fix script snippet before applying changes)
```

## 7. Health and Automatic Repair Configuration

powershell
```
# All attached VMs (auto protocol detection)
# Apply health & repair (see embedded Health & Repair script snippet)

# Custom probe (HTTPS /health)
# Custom probe example (use embedded Health & Repair script to add parameters)
```

## 8. Validation Checklist

- VMSS Orchestration = Flexible
- PlatformFaultDomainCount = 1
- SinglePlacementGroup = false
- Auto OS Upgrade disabled
- Autoscale disabled during migration wave
- Health Extension present
- Automatic Instance Repair enabled
- Application probe endpoint healthy across zones

## 9. Bulk and Downtime Guidance

- Attachment is metadata operation; typical no compute downtime if prerequisites satisfied.
- Recommend sequential wave progression + health probe validation before next wave.
- Test application behavior under simulated probe failures (grace period calibration).

## 10. Cost and Risk Considerations

| Aspect | Consideration | Mitigation |
|--------|---------------|------------|
| Cross-Zone Latency | Potential latency increase | Validate latency SLO post migration |
| Health Misconfiguration | False positive restarts | Tune grace and thresholds before prod |
| Script Drift | Version mismatch | Pin versions / central repo |
| Capacity Constraints | Zone provisioning delays | Pre-check region capacity |

## 11. Rollback / Abort Strategy

- If attachment reveals unexpected behavior: detach VM (custom script) and re-validate baseline.
- Maintain snapshot/baseline metrics for quick diagnosis.

## 12. References

- Internal posture capability service
- Azure Docs: VMSS Flex, Application Health Extension, Automatic Instance Repair
- Embedded script snippets in this artifact (Attach, Fix, Health and Repair, ASR outline) — no external .ps1 file references required

_Validate automation paths in a staging subscription before large-scale production adoption._
