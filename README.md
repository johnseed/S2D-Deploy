# S2D-Deploy
Deploy Storage Spaces Direct on Windows Server

## Plan  
- Server : 1 domain controller + n nodes
- Network Adapter : RDMA support (optional)

## Troubleshoot
- Max volume size 64TB : don't use WindowsAdminCenter to create volume, powershell `New-Volume -UseMaximumSize`
- RAID not supported : Change RAID Personality to JBOD
- mount destination unreachable : install keyutils cifs-utils
- mount cifs error 5 input/output error : upgrade system, e.g. CentOS 7.6 upgrade to 7.9

## Deploy
[Reference](https://learn.microsoft.com/en-us/windows-server/storage/storage-spaces/deploy-storage-spaces-direct?source=docs). 
install system, create domain controller, join domain. then
```powershell
$ServerList = "win-node1.s2d.com", "win-node2.s2d.com", "WIN-node3.s2d.com"

# Install roles and features
$FeatureList = "Failover-Clustering", "Data-Center-Bridging", "RSAT-Clustering-PowerShell", "Hyper-V", "Hyper-V-PowerShell", "FS-FileServer" # Hyper-V is optional if deploying with virtual machines
Invoke-Command ($ServerList) {
    Install-WindowsFeature -Name $Using:Featurelist
}

# Clean drives
Invoke-Command ($ServerList) {
    Update-StorageProviderCache
    Get-StoragePool | ? IsPrimordial -eq $false | Set-StoragePool -IsReadOnly:$false -ErrorAction SilentlyContinue
    Get-StoragePool | ? IsPrimordial -eq $false | Get-VirtualDisk | Remove-VirtualDisk -Confirm:$false -ErrorAction SilentlyContinue
    Get-StoragePool | ? IsPrimordial -eq $false | Remove-StoragePool -Confirm:$false -ErrorAction SilentlyContinue
    Get-PhysicalDisk | Reset-PhysicalDisk -ErrorAction SilentlyContinue
    Get-Disk | ? Number -ne $null | ? IsBoot -ne $true | ? IsSystem -ne $true | ? PartitionStyle -ne RAW | % {
        $_ | Set-Disk -isoffline:$false
        $_ | Set-Disk -isreadonly:$false
        $_ | Clear-Disk -RemoveData -RemoveOEM -Confirm:$false
        $_ | Set-Disk -isreadonly:$true
        $_ | Set-Disk -isoffline:$true
    }
    Get-Disk | Where Number -Ne $Null | Where IsBoot -Ne $True | Where IsSystem -Ne $True | Where PartitionStyle -Eq RAW | Group -NoElement -Property FriendlyName
} | Sort -Property PsComputerName, Count

# Validate the cluster(execute on node)
Test-Cluster -Node win-nod1.s2d.com, win-node2.s2d.com, win-node3.s2d.com -Include "Storage Spaces Direct", "Inventory"

# Create the cluster
New-Cluster -Name SASCluster -Node win-nod1.s2d.com, win-node2.s2d.com, win-node3.s2d.com -NoStorage

Enable-ClusterStorageSpacesDirect
New-Volume -UseMaximumSize
mountvol S: \\?\Volume{891f956c-bbc9-41dc-94fc-2fdbd58213a2}\

```
