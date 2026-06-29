# Hyper-V (PowerShell)

Managing Hyper-V VMs from PowerShell — lifecycle, inspection, resource config, snapshots, cloning,
and full provisioning scripts. Run PowerShell **as Administrator**; the `Hyper-V` module ships with
the role/feature.

```powershell
Get-Command -Module Hyper-V | Measure-Object   # how many cmdlets the module exposes
Get-VMHost                                       # host-level settings (default paths, NUMA, etc.)
Import-Module Hyper-V                             # usually auto-loaded; force it if cmdlets are missing
```

- [Lifecycle (start / stop / restart)](#lifecycle)
- [Inspect / list VMs](#inspect--list-vms)
- [CPU & memory](#cpu--memory)
- [Disks (VHD)](#disks-vhd)
- [Snapshots (checkpoints)](#snapshots-checkpoints)
- [Clone a VM](#clone-a-vm)
- [Provisioning script (parameterized)](#provisioning-script-parameterized)
- [TODO / enhancements](#todo--enhancements)

---

## Lifecycle

```powershell
Start-VM   -Name "TestVM_01"               # power on
Stop-VM    -Name "TestVM_01"               # graceful shutdown (asks the guest OS nicely)
Stop-VM    -Name "TestVM_01" -TurnOff      # hard power-off (pulls the plug — risk of FS corruption)
Restart-VM -Name "TestVM_01"               # graceful restart
Restart-VM -Name "TestVM_01" -Force        # don't prompt

# bulk: act on many VMs at once
Get-VM | Where-Object {$_.State -eq 'Running'} | Stop-VM            # shut down everything running
Get-VM -Name Dev* | Start-VM                                        # start all "Dev*" VMs
```

> `Stop-VM` (graceful) needs Hyper-V Integration Services running in the guest. If a VM ignores it,
> `-TurnOff` is the equivalent of holding the power button — use only when the guest is hung.

---

## Inspect / list VMs

```powershell
Get-VM                                           # all VMs + state, CPU, memory, uptime
Get-VM -Name Dev*                                # all VMs whose name starts with "Dev"
Get-VM | Where-Object {$_.State -eq 'Running'}   # only running
Get-VM | Where-Object {$_.State -eq 'Off'}       # only powered-off

Get-VM -Name "ubuntu-server-1" | Format-List *   # EVERYTHING about one VM
Get-VM | Out-GridView                            # interactive sortable/filterable window (GUI)

# pick specific fields
Get-VM -Name "ubuntu-server-1" |
    Select-Object Name, ProcessorCount, AutomaticCheckpointsEnabled, CheckpointType

Get-VM -Name "ubuntu-server-1" |
    Select-Object Name, State, DynamicMemoryEnabled, CheckpointType, MemoryStartup | Format-Table
```

**Show each VM with its VHD path** (calculated property — `@{L="label"; E={expression}}`):

```powershell
Get-VM | Select-Object Name, State,
    @{L="CPU_Count";      E={$_.ProcessorCount}},
    @{L="RAM_Startup_GB"; E={$_.MemoryStartup / 1GB}},
    @{L="Dynamic_Memory"; E={$_.DynamicMemoryEnabled}},
    @{L="VHD_Location";   E={(Get-VMHardDiskDrive -VM $_).Path}} |   # -VM $_ : pass the object, not its name
    Format-Table
    # swap Format-Table for Out-GridView to get the clickable GUI version
```

> **Optimization (from the todo):** pass the VM object directly with `-VM $_` instead of
> `-VMName $_.Name`. It's faster and more reliable than extracting and re-resolving the name string.
> Doing the calculated property inside the **first** `Select-Object` (not a second piped one) saves a
> whole pipeline pass and the memory that goes with it.

---

## CPU & memory

```powershell
Set-VM -Name "ubuntu-server-1" -ProcessorCount 4          # set vCPU count

# force STATIC memory (Dynamic Memory off) — pin exactly 4 GB
Set-VMMemory -VMName "ubuntu-server-1" -DynamicMemoryEnabled $false -StartupBytes 4GB

# enable DYNAMIC memory with a floor and ceiling (balloons between 512 MB and 4 GB)
Set-VM -Name "ubuntu-server-1" -DynamicMemory `
    -MemoryMinimumBytes 512MB -MemoryMaximumBytes 4GB

Get-VMMemory -VMName "ubuntu-server-1" | Format-List      # verify memory config
```

> Memory must usually be changed while the VM is **off** (especially static↔dynamic switches).
> Dynamic Memory is great for density but bad for latency-sensitive workloads (DBs) — pin those static.

---

## Disks (VHD)

```powershell
Get-VMHardDiskDrive -VMName "ubuntu-server-1" | Select-Object Path   # where's the disk file?

# create / inspect VHDs directly
New-VHD -Path "C:\VMs\data.vhdx" -SizeBytes 50GB -Dynamic            # thin-provisioned (grows on use)
New-VHD -Path "C:\VMs\fixed.vhdx" -SizeBytes 50GB -Fixed             # full-size up front (faster I/O)
Get-VHD  -Path "C:\VMs\data.vhdx"                                    # size, type, fragmentation
Resize-VHD -Path "C:\VMs\data.vhdx" -SizeBytes 100GB                 # grow (then extend FS inside guest)

# attach an extra disk to a running/stopped VM
Add-VMHardDiskDrive -VMName "ubuntu-server-1" -Path "C:\VMs\data.vhdx"
```

---

## Snapshots (checkpoints)

```powershell
Get-VMSnapshot -VMName "ubuntu-server-1"                              # list checkpoints
Checkpoint-VM  -Name "ubuntu-server-1" -SnapshotName "before-upgrade" # take one
Restore-VMSnapshot -VMName "ubuntu-server-1" -Name "before-upgrade" -Confirm:$false  # roll back

Get-VMSnapshot -VMName "ubuntu-server-1" | Remove-VMSnapshot          # delete ALL checkpoints (merges back)

Set-VM -Name "ubuntu-server-1" -CheckpointType Disabled              # turn checkpoints off entirely
```

> **Gotcha:** checkpoints are **not backups** — they chain differencing disks (`.avhdx`) onto the
> parent VHD. Leaving them around bloats disk and slows I/O; the merge on delete can take a while.
> Never snapshot a domain controller or a clustered DB casually — rolling back can desync state.

---

## Clone a VM

Copy the VHD, point a fresh VM at the copy, then harden the config (static RAM, no checkpoints).

```powershell
# 1. copy the disk (backtick ` = line continuation in PowerShell)
Copy-Item "C:\Users\Admin\Desktop\Hyper-V\ubuntu-server-1.vhdx" `
          "C:\Users\Admin\Desktop\Hyper-V\ubuntu-server-2.vhdx"

# 2. create a Gen-2 VM on the copied disk
New-VM -Name "ubuntu-server-2" `
       -Generation 2 `
       -MemoryStartupBytes 4GB `
       -VHDPath "C:\Users\Admin\Desktop\Hyper-V\ubuntu-server-2.vhdx"

# 3. config while OFF
Stop-VM "ubuntu-server-2" -Force -ErrorAction SilentlyContinue
Set-VMMemory -VMName "ubuntu-server-2" `
    -DynamicMemoryEnabled $false -StartupBytes 4GB -MinimumBytes 4GB -MaximumBytes 4GB
Set-VM -Name "ubuntu-server-2" -ProcessorCount 2
Set-VM -Name "ubuntu-server-2" -CheckpointType Disabled

# 4. verify
Get-VM -Name "ubuntu-server-2" |
    Select-Object Name, State, ProcessorCount, DynamicMemoryEnabled, CheckpointType,
        @{L="VHD_Path"; E={(Get-VMHardDiskDrive -VM $_).Path}} | Format-List
```

> **Gotcha — clone collisions:** the copied guest keeps the original's hostname, machine-id, and
> SSH host keys. Boot it isolated and regenerate them (Linux: `hostnamectl`, `dbus-uuidgen`,
> `ssh-keygen -A`) or two VMs will fight over identity on the network. Gen-2 disks must be `.vhdx`.

---

## Provisioning script (parameterized)

Save as `New-HyperVVM.ps1` — one command to create a hardened, static-memory, no-checkpoint VM.

```powershell
param(
    [Parameter(Mandatory = $true)] [string]$VMName,
    [Parameter(Mandatory = $true)] [int]$MemoryGB = 4,
    [Parameter(Mandatory = $true)] [int]$CPUCount = 2,
    [Parameter(Mandatory = $true)] [string]$VHDPath,
    [ValidateSet("Generation1", "Generation2")] [string]$Generation = "Generation2"
)

$memoryBytes = $MemoryGB * 1GB                          # convert GB → bytes

Write-Host "Creating VM: $VMName" -ForegroundColor Cyan
$gen = if ($Generation -eq "Generation2") { 2 } else { 1 }
New-VM -Name $VMName -Generation $gen -MemoryStartupBytes $memoryBytes -VHDPath $VHDPath

# harden while OFF
Stop-VM -Name $VMName -Force -ErrorAction SilentlyContinue

Write-Host "Configuring CPU..."             -ForegroundColor Cyan
Set-VM -Name $VMName -ProcessorCount $CPUCount

Write-Host "Disabling checkpoints..."       -ForegroundColor Cyan
Set-VM -Name $VMName -CheckpointType Disabled

Write-Host "Forcing STATIC memory..."       -ForegroundColor Cyan
Set-VMMemory -VMName $VMName `
    -DynamicMemoryEnabled $false `
    -StartupBytes $memoryBytes -MinimumBytes $memoryBytes -MaximumBytes $memoryBytes

Write-Host "Verification:" -ForegroundColor Green
Get-VM -Name $VMName |
    Select-Object Name, State, ProcessorCount, DynamicMemoryEnabled, CheckpointType | Format-List
Get-VMMemory -VMName $VMName | Format-List
```

Run it:

```powershell
.\New-HyperVVM.ps1 `
    -VMName "ubuntu-server-2" `
    -MemoryGB 4 `
    -CPUCount 2 `
    -VHDPath "C:\Users\Admin\Desktop\Hyper-V\ubuntu-server-2.vhdx"
```

> If the script is blocked by execution policy:
> `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` (per-session, safest).

---

## TODO / enhancements

- [ ] Retarget everything for **Windows Server 2025**.
- [x] Pass `-VM $_` directly into `Get-VMHardDiskDrive` (faster + more reliable than the `.Name` string) — applied in [Inspect](#inspect--list-vms).
- [x] **Single-select pass:** fold the calculated property into the initial `Select-Object` to drop a pipeline stage and cut memory overhead — applied above.
- [ ] Optimize the pipeline further and build a **full end-to-end automation script** like the one in [Linux/KVM.md](../Linux/KVM.md) (cloud-image download → provision → first-boot config), the Hyper-V equivalent of the KVM `virt-install` automation.
