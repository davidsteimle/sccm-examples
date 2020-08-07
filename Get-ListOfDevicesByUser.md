# SCCM: Getting a List of Devices by User

There was an issue raised in our enterprise about systems with a particular software having issues with a system upgrade.

I found that we had a deployment in place to install the software. The target collection was user based. I wanted to find the systems associated with these users in an attempt to see how many systems might have/need the software.

I performed the below actions in the SCCM Powershell client:

```powershell
# Get all users from a user targetting collection as an array
$Users = Get-CMCollectionMember -CollectionName $CollectionName | `
    Select-Object -ExpandProperty SMSID

# Build a generic list
$Results = New-Object "System.Collections.Generic.List[PSObject]"

# Traverse array and get device affinity based on username (SMSID)
$($Users).ForEach(
    {
        $Query = [psobject]@{
            User = $PSItem
            SerialNumber = $(Get-CMUserDeviceAffinity -UserName $PSItem | `
                Select-Object -ExpandProperty ResourceName)
        }
        # Add $Query object to $Results list
        $Results.Add($Query)
    }
)

# Build a generic list
$SystemQuery = New-Object "System.Collections.Generic.List[PSObject]"

# Traverse list and get desired properties from each system
$Results.SerialNumber.ForEach(
    {
        Write-Host $PSItem
        $Result = Get-CMDevice -Name $PSItem | `
            Select-Object -Property Name,DeviceOS,DeviceOSBuild,PrimaryUser
        # Write some output so you can see it running
        Write-Host $Result.DeviceOSBuild
        # Add result to $SystemQuery list
        $SystemQuery.Add($Result)
    }
)

# Writing out to JSON because my query took a long time
$SystemQuery | `
    ConvertTo-Json | `
    Out-File C:\Users\steimle\bin\$CollectionName.json -Force

# Getting a count of list entries
$SystemQuery.Count
```

I could then convert to CSV and share with interested parties who love Excel. I did find that some entries with missing data did not ``ConvertTo-Csv`` well, so I imported the JSON file with ``ConvertFrom-Json`` and then converted that new object to CSV.

A sample of the results looks like:
 
```
Name       DeviceOS                                               DeviceOSBuild    PrimaryUser
----       --------                                               -------------    -----------
System0001 Microsoft Windows NT Workstation 10.0                  10.0.14393.3808  dom\user02
System0002 Microsoft Windows NT Workstation 10.0                  10.0.14393.3808  dom\user03
System0003 Microsoft Windows NT Workstation 10.0                  10.0.10240.18638 dom\user07
System0004 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.14393.3504  dom\user05
System0005 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.14393.3630  dom\user09
System0007 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.10240.18638 dom\user10
System0006 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.14393.3808  dom\user23
System0010 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.14393.3750  dom\user01
System0008 Microsoft Windows NT Workstation 10.0 (Tablet Edition) 10.0.14393.3808  dom\user04
```
