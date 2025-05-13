---
title: "Access Package Assignment Request without bypassing approval"
date: 2025-05-13
classes: wide
categories:
  - Access Packages
---
How can you add a user to an access package using PowerShell without bypassing approval? Great question! Today, I attempted this using the Microsoft [documentation](https://learn.microsoft.com/en-us/graph/api/entitlementmanagement-post-accesspackageassignmentrequests?view=graph-rest-beta&tabs=http) but I kept encountering the following error: `No valid policy was found in the request` .This is the code that I was trying:
```powershell
$params = @{
	requestType = "UserAdd"
	accessPackageAssignment = @{
		targetId = "bc78db22-3bbd-4488-1121-35660bfa7989"
		assignmentPolicyId = "0aba116d-fab3-41a0-2205-c93b2a6ed59c"
		accessPackageId = "36c81c22-13e5-4a8e-9efb-e1b98d708bd8"
	}
    justification = "test"
}

New-MgBetaEntitlementManagementAccessPackageAssignmentRequest -BodyParameter $params -Verbose
```

