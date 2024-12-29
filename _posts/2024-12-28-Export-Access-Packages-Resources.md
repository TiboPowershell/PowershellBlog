---
title: "Export Access Package Resource Groups"
date: 2024-12-28
classes: wide
categories:
  - Access Packages
---

Access Packages are very useful but currently lack robust reporting options. Imagine working in a large environment with over 300 Access Packages, and needing to identify which ones are responsible for adding people to a PIM group (PIM_Security_Sentinel_Reader).
Unfortunately, there isn't an out-of-the-box solution for this query. In the first part of this series, we will write a script that exports all Access Packages across all Catalogs and shows their corresponding groups. We will then export this data to Excel, allowing us to use the filter option for easier analysis.

## Script
If you don't want to read through this, feel free to download the [script](https://github.com/TiboPowershell/PowershellScripts/blob/main/AccessPackageReporting/ExportAccessPackageResources.ps1) directly.

## Result
We want to achieve the following, allowing us to filter on GroupDisplayname: 

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/excel.png)

Filtering on GroupDisplayname we can see that PIM_Security_Sentinel_Reader is a resource of the following Access Packages:

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/excel2.png)

You might be thinking, "This information can't be too hard to retrieve, right?" Unfortunately, there are two issues that make this more challenging than it should be.
1. **Indirect Group Addition:** Groups are not directly added to Access Packages. Instead, they are added to the Catalog. Within the Access Package, a correlation is made between the Access Package and the resource groups in the Catalog. This leads to another potential issue: if a group's display name is changed after being added to an Access Package, the Access Package will still show the old name. Consequently, when retrieving the group via PowerShell, it will also display the old name. A workaround for this is to go to the corresponding catalog and refresh all groups from the origin, a feature currently in preview. Another workaround (this is the one we will be using) is to create a Dictionary of all Groups (id, Displayname) and use this to get the correct Display Name by using the OriginID and getting the value from our Dictionary.
1. **Active Assignment Requirement:** There is a command that can easily retrieve groups given the Access Package ID. However, this only works when there is an active assignment for the Access Package. This means that any Access Packages that are not currently assigned (perhaps they are new or old) will not appear in the report. The command I’m referring to is: `Get-MgBetaEntitlementManagementAccessPackageAssignment`

## Prerequisites
To run this script, we will need to download several PowerShell modules and we also need to create an App Registration which will be used to connect to Graph.

### Powershell Modules
- Microsoft.Graph.Beta: `Install-Module Microsoft.Graph.Beta -Repository PSGallery -Force`
- ImportExcel: `Install-Module -Name ImportExcel -RequiredVersion 7.8.4`
  
### App Registration
#### Permissions
You will need to create a new app registration with the following **Application** permissions:
- EntitlementManagement.Read.All
- Group.Read.All
![alt]({{ site.url }}{{ site.baseurl }}/assets/images/permissions.png)

**Important:** Don't forget to grant admin consent!
{: .notice--info}

#### Certificates & secrets
We will also need to create a client secret or a certificate, which the application will use to authenticate itself. After creating the client secret, make sure to note its value, as it will be required later. Along with the client secret, we will also need the Tenant ID and the Client ID (App ID). 
![alt]({{ site.url }}{{ site.baseurl }}/assets/images/secret.png)

## Let’s start with the script
### Connect to Graph
We will need to fill in our Tenant ID, Client ID and Client Secret. We also define a path for our Excel export.
```powershell
$Global:TenantId = "<TenantID>"
$Global:ClientId = "<ClientID>"
$Global:clientSecret = "<ClientSecret>"
$ExportToExcelPath = "<Path/To/Excel/export.xlsx>"

$SecuredPasswordPassword = ConvertTo-SecureString -String $clientSecret -AsPlainText -Force
$ClientSecretCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $clientId, $SecuredPasswordPassword

#Connect MgGraph
Connect-MgGraph -TenantId $tenantId -ClientSecretCredential $ClientSecretCredential
```
{: #code-example-1}

### Build a dictionary of groups
We will start by building a function to create a dictionary of all our groups and their corresponding IDs. Although we can retrieve all group display names in each catalog using the `Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageResource` command, we will use the `originId` property to match each group with its corresponding display name in our dictionary. This approach ensures that the group display names are accurate and correctly mapped.
```powershell
function Get-GroupsDictionary {
    $groupDictionary = @{}
    $groups = Get-MgBetaGroup -All -Property Id, DisplayName
    
    foreach ($group in $groups) {
        $groupDictionary[$group.Id] = $group.DisplayName
    }

    return $groupDictionary
}
```
### Get the Access Package Resources
This part can be a bit tricky. To determine which resources (groups) belong to an Access Package, we need to focus on the following:
`$accessPackage.AccessPackageResourceRoleScopes.Id`
This value will look something like:
`99249363-d5d9-4794-bb30-5a346ad98c20_7f9cb442-f7f7-4ae4-9375-690f05ef0b79.`
You might be wondering, "How does this relate to a group?" Believe me, I asked myself the same question until I noticed the underscore separating two IDs. According to [Microsoft documentation](https://learn.microsoft.com/en-us/graph/api/resources/accesspackageresourcerolescope?view=graph-rest-1.0), the AccessPackageResourceRoleScopes property refers to both a role within a resource and a scope for that resource.
- **First ID**: Represents the Role.
- **Second ID**: Represents the Scope.
For our purposes, we will only need the **first ID** to proceed.

Now, where can we use this ID? We can use it to search for the corresponding resource in the `$resources.AccessPackageResourceRoles`. If we request the `OriginId` property for the matching resource, you'll see something like this:
`Member_144505d1-837a-4a86-8143-2aa775e4cf97`.

Once again, notice the underscore separating two values:
- The part before the underscore (`Member`) indicates whether the user will be added as a **member** or **owner** of the group.
- The part after the underscore (`144505d1-837a-4a86-8143-2aa775e4cf97`) is the **ID of the group**.
We can then use this group ID to search in our dictionary for the correct display name.

So, what do we need to do in this function? Here’s the plan:
- Loop through all catalogs.
- Retrieve all resources within each catalog.
- Loop through each access package within the catalog.
- Match the **Role ID** with the **AccessPackageResourceRoles ID**.
- Use the group ID to retrieve the corresponding group display name from the dictionary.
- Create an array of custom objects that contains all the necessary information.

```powershell
function Get-ResourcesFromAccessPackages{
    param (
        [hashtable]$GroupDictionary
    )

    $accessPackageCatalogs = Get-MgBetaEntitlementManagementAccessPackageCatalog -All
    $exportList = @()
    $totalCatalogs = $accessPackageCatalogs.Count
    foreach ($catalog in $accessPackageCatalogs) {
        Write-Host "[$($accessPackageCatalogs.IndexOf($catalog) + 1)/$($totalCatalogs)][Catalog: $($catalog.DisplayName)]"

        ##get resource from catalog
        $resources = Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageResource -AccessPackageCatalogId $catalog.Id -ExpandProperty *
        ## get all access packages within this resource
        $accessPackages = Get-MgBetaEntitlementManagementAccessPackage -CatalogId $catalog.Id -ExpandProperty AccessPackageResourceRoleScopes

        $totalAccessPackagesInCatalog = $accessPackages.count
        foreach($accessPackage in $accessPackages){
            Write-Host "`t[$($accessPackages.IndexOf($accessPackage) + 1)/$($totalAccessPackagesInCatalog)][Access Package: $($accessPackage.DisplayName)]"
            $roleIDs = $accessPackage.AccessPackageResourceRoleScopes.Id | ForEach-Object {($_ -split '_')[0]} 
            foreach($roleID in $roleIDs){
                ##match the roleIDs with $resources.AccessPackageResourceRoles.ID to get the origin ID (we split it this with underscore since this value is prefixed with Member or Owner)
                $matchedRole = (($resources.AccessPackageResourceRoles | Where-Object {$_.id -eq $roleID}).OriginId -split '_')[1]
                ##match this with our GroupDictionary (to make sure we get the correct name)
                $exportList += [PSCustomObject][Ordered]@{
                    Catalog = $catalog.DisplayName
                    CatalogID = $catalog.id
                    AccessPackage = $accessPackage.DisplayName
                    AccessPackageID = $accessPackage.Id
                    GroupDisplayname = $GroupDictionary[$matchedRole]
                    GroupID  = $matchedRole
                }
            }
        }
    }
    return $exportList
}
```
### Export the results
Now, all that’s left is to call the two functions and export the result. This final step will consolidate all the information into a single output, ready for use.
```powershell
$groupsDictionary = Get-GroupsDictionary
$getAllAccessPackagesWithResources = Get-ResourcesFromAccessPackages -GroupDictionary $groupsDictionary
##export values to Excel or Csv 
Export-Excel -Path $ExportToExcelPath -InputObject $getAllAccessPackagesWithResources  -WorksheetName "AccessPackageResources" -TableStyle Light1 -TableName "Results"
```

## Putting it all together
```powershell
$Global:TenantId = "<TenantID>"
$Global:ClientId = "<ClientID>"
$Global:clientSecret = "<ClientSecret>"
$ExportToExcelPath = "<Path/To/Excel/export.xlsx>"

$SecuredPasswordPassword = ConvertTo-SecureString -String $clientSecret -AsPlainText -Force
$ClientSecretCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $clientId, $SecuredPasswordPassword

#Connect MgGraph
Connect-MgGraph -TenantId $tenantId -ClientSecretCredential $ClientSecretCredential

function Get-GroupsDictionary {
    $groupDictionary = @{}
    $groups = Get-MgBetaGroup -All -Property Id, DisplayName
    
    foreach ($group in $groups) {
        $groupDictionary[$group.Id] = $group.DisplayName
    }

    return $groupDictionary
}

## NOTE we do this because if you want to use $Assignments = Get-MgBetaEntitlementManagementAccessPackageAssignment it will only work for access packages that have a user assigned to them
function Get-ResourcesFromAccessPackages{
    param (
        [hashtable]$GroupDictionary
    )

    $accessPackageCatalogs = Get-MgBetaEntitlementManagementAccessPackageCatalog -All
    $exportList = @()
    $totalCatalogs = $accessPackageCatalogs.Count
    foreach ($catalog in $accessPackageCatalogs) {
        Write-Host "[$($accessPackageCatalogs.IndexOf($catalog) + 1)/$($totalCatalogs)][Catalog: $($catalog.DisplayName)]"

        ##get resource from catalog
        $resources = Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageResource -AccessPackageCatalogId $catalog.Id -ExpandProperty *
        ## get all access packages within this resource
        $accessPackages = Get-MgBetaEntitlementManagementAccessPackage -CatalogId $catalog.Id -ExpandProperty AccessPackageResourceRoleScopes

        $totalAccessPackagesInCatalog = $accessPackages.count
        foreach($accessPackage in $accessPackages){
            Write-Host "`t[$($accessPackages.IndexOf($accessPackage) + 1)/$($totalAccessPackagesInCatalog)][Access Package: $($accessPackage.DisplayName)]"
            $roleIDs = $accessPackage.AccessPackageResourceRoleScopes.Id | ForEach-Object {($_ -split '_')[0]} 
            foreach($roleID in $roleIDs){
                ##match the roleIDs with $resources.AccessPackageResourceRoles.ID to get the origin ID (we split it this with underscore since this value is prefixed with Member or Owner)
                $matchedRole = (($resources.AccessPackageResourceRoles | Where-Object {$_.id -eq $roleID}).OriginId -split '_')[1]
                ##match this with our GroupDictionary (to make sure we get the correct name)
                $exportList += [PSCustomObject][Ordered]@{
                    Catalog = $catalog.DisplayName
                    CatalogID = $catalog.id
                    AccessPackage = $accessPackage.DisplayName
                    AccessPackageID = $accessPackage.Id
                    GroupDisplayname = $GroupDictionary[$matchedRole]
                    GroupID  = $matchedRole
                }
            }
        }
    }
    return $exportList
}

$groupsDictionary = Get-GroupsDictionary
$getAllAccessPackagesWithResources = Get-ResourcesFromAccessPackages -GroupDictionary $groupsDictionary
##export values to Excel or Csv 
Export-Excel -Path $ExportToExcelPath -InputObject $getAllAccessPackagesWithResources  -WorksheetName "AccessPackageResources" -TableStyle Light1 -TableName "Results"
```
