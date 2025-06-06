---
title: "Access Package Assignment Request without bypassing approval"
date: 2025-05-13
classes: wide
categories:
  - Access Packages
---
**Important:** Update: Looks like this method is currently not working. It will still auto-approve. I will log a ticket on github.
{: .notice--info}


How can you add a user to an access package using PowerShell without bypassing approval? Great question! Today, I attempted this using the Microsoft [documentation](https://learn.microsoft.com/en-us/graph/api/entitlementmanagement-post-accesspackageassignmentrequests?view=graph-rest-beta&tabs=http) but I kept encountering the following error: `No valid policy was found in the request`. This is the code that I was trying:
```powershell
$params = @{
	requestType = "UserAdd"
	accessPackageAssignment = @{
		targetId = "bc78db22-3bbd-4488-1121-35660bfa7989"
		assignmentPolicyId = "0aba116d-fab3-41a0-2205-c93b2a6ed59c"
		accessPackageId = "36c81c22-13e5-4a8e-9efb-e1b98d708bd8"
	}
    justification = "User needs this Access package"
}

New-MgBetaEntitlementManagementAccessPackageAssignmentRequest -BodyParameter $params
```
I initially thought that `AdminAdd` would always bypass approval, requiring us to use `UserAdd` instead. However, when reviewing my browser's network logs while manually assigning a user to the access package without bypassing approval I noticed that this was done using `requestType = "AdminAdd"`. Creating the request was challenging because simply changing `requestType = "UserAdd"` to `requestType = "AdminAdd"` works, but it bypassed the approval flow. 

## Now how do we solve this? 
We need to add the `IsApprovalRequired` parameter to our `$params` variable:
```powershell
$params = @{
    requestType = "AdminAdd"  
    accessPackageAssignment = @{
        targetId = "bc78db22-3bbd-4488-1121-35660bfa7989"
        assignmentPolicyId = "0aba116d-fab3-41a0-2205-c93b2a6ed59c"
        accessPackageId = "36c81c22-13e5-4a8e-9efb-e1b98d708bd8"
    }
    justification = "User needs this Access package"
    parameters = @(
        @{
            name = "IsApprovalRequired"
            value = "true"
        }
    )
}

New-MgBetaEntitlementManagementAccessPackageAssignmentRequest -BodyParameter $params
```
