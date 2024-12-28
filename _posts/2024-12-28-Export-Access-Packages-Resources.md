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

Filtering on GroupDisplayname we can see that PIM_Security_Sentinel_Reader is in the following Access Packages:

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/excel2.png)

You might be thinking, "This information can't be too hard to retrieve, right?" Unfortunately, there are two issues that make this more challenging than it should be.
1. **Indirect Group Addition:** Groups are not directly added to Access Packages. Instead, they are added to the Catalog. Within the Access Package, a correlation is made between the Access Package and the resource groups in the Catalog. This leads to another potential issue: if a group's display name is changed after being added to an Access Package, the Access Package will still show the old name. Consequently, when retrieving the group via PowerShell, it will also display the old name. A workaround for this is to go to the corresponding catalog and refresh all groups from the origin, a feature currently in preview. Another workaround (this is the one we will be using) is to create a Dictionary of all Groups (id, Displayname) and use this to get the correct Display Name by using the OriginID and getting the value from our Dictionary.
1. **Active Assignment Requirement:** There is a command that can easily retrieve groups given the Access Package ID. However, this only works when there is an active assignment for the Access Package. This means that any Access Packages that are not currently assigned (perhaps they are new or old) will not appear in the report. The command I’m referring to is: `Get-MgBetaEntitlementManagementAccessPackageAssignment`

## Prerequisites
To run this script, we will need to download several PowerShell modules and we also need to create an App Registration which will be used to connect to Graph

### Powershell Modules
- Microsoft.Graph.Beta: `Install-Module Microsoft.Graph.Beta -Repository PSGallery -Force`
- ImportExcel: `Install-Module -Name ImportExcel -RequiredVersion 7.8.4`
  
### App Registration
You will need to create a new app registration with the following **Application** permissions:
- EntitlementManagement.Read.All
- Group.Read.All

**Important:** Don't forget to grant admin consent!
{: .notice--info}
