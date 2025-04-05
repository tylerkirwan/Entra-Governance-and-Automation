# Secure MFA Resets with Passkeys and Entra Governance

Provides a process and automation for a self-service reset of MFA in a passwordless (or future passwordless) environment. The process institutes an approval process where the manager requests on behalf the individual needing an MFA reset (Or is the 1st stage of the approval process). IT or Security is the secondary stage approver.

This flow utilizes Entra access packages, logic apps, and corresponding conditional access policies to transition or maintain phish resistant MFA. 
<div style="display: flex; align-items: flex-start;">
  <img 
    src="https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/approvalflow.png?raw=true" 
    alt="Approval Flow" 
    style="width:45%; max-width:300px; margin-right:10px;"
  />
  <img 
    src="https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/approvallogic.png?raw=true" 
    alt="Approval Logic" 
    style="width:45%; max-width:300px; margin-top:-20px;"
  />
</div>

## Purpose and Goals

- Reduce likelihood of social engineering a helpdesk member by adding approval stages, visibility, and reporting capabilities to MFA resets
- Privileged roles don't need to be granted to helpdesk even temporally
- Streamline MFA resets via Entra Governance access packages and save time with your support teams and end users
- Enroll new MFA methods into phish resistance by default
- No exchanging or sharing of passwords
- Cleanup of expired TAPs (Optional)
## FAQ & Prerequisites 

#### What licensing is required?

Security E3 and or Entra P2

#### What access is needed?

Privileged Role Administrator and Privileged Authentication Administrator or Global Admin

#### What Resources?
This will vary based on what pieces of your implementation you use but it involves at least 1 logic app. Optional: Azure Communication Services for delivering email

## Directions
Broken into 4 sections. The first 2 sections are the most time consuming.
### Part 1 - Create Access Package and Assign Permissions

All of these steps as far as I'm aware can be done with Graph Powershell SDK however these directions will be a mix a both.

1) In the Azure or Entra portal navigate to Identity Governance and create a dedicated Catalog which will contain your access packages and custom extensions (aka logic apps) Record the catalog's ID to be used later.

2) Once catalog is created create your custom extension with the following properties
- Extension Type: Request workflow (triggered when an access package is requested, approved, granted, or removed)
- Launch and continue
- Create logic app "Yes" 

3) After the custom extension/logic app is created it needs to be granted the access to make authentication changes and teams chat creation. In the logic app's settings choose identity and create a "System assigned" identity. Record its ID

4) Using the Graph Powershell SDK determine the custom extension's ID 
and assign the ID from the catalog that was created created in step 1. [Credit]
```powershell
$catalogId = <YOUR ID>
```

```powershell
$customExtensionId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageCustomWorkflowExtension -AccessPackageCatalogId $catalogId | Out-GridView -PassThru).Id
```
5) Select and record the ID of the extension
6) Assign variables to the IDs collected and permissions to the system managed identity created earlier. Ensure the account assigning these permissions has at least Privileged Role Administrator
```powershell
$permissions =  @("UserAuthenticationMethod.ReadWrite.All", "Chat.ReadWrite.All", "Directory.Read.All")
```
7) Assign the managed ID variable you recorded in step 4 and assign it the permissions.
```powershell
$MIObjectId = "YOUR ID"
Connect-MgGraph -Scopes @("AppRoleAssignment.ReadWrite.All", "EntitlementManagement.ReadWrite.All")
$permissions | ForEach-Object {
	$PermissionName = $_
	$GraphSP = Get-MgServicePrincipal -Filter "startswith(DisplayName,'Microsoft Graph')" | Select-Object -first 1 #Graph App ID: 00000003-0000-0000-c000-000000000000
	$AppRole = $GraphSP.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
	New-MgServicePrincipalAppRoleAssignment -AppRoleId $AppRole.Id -ServicePrincipalId $MIObjectId -ResourceId $GraphSP.Id -PrincipalId $MIObjectId
}
```
8) Finally, create the access package you will use. 
```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "2Direct Add: MFA Resets"
  description = "2Directly reset MFA and assign tap"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id
```
9) At this point you can add the security group you will use as part of the access package. This group will be used for conditional access. Additionally, you can create the Access Package Policy. Customize as you see fit however it's strongly suggested to utilize the multistage approvals. Below is how I configured the policy:
-  Pictures of policy
### Part 2 - Apply logic app json and Communication Actions
---
LAmfa.json is not the full code, the connection and communication actions need to be added by the designer. Part 3 will explain the communication pieces that I found easiest to add through the logic app designer.

---
1) In the logic app designer there will be 4 actions to add. Ensure you have set up Azure Communication Services if you plan on following exactly
- Create a Chat
- Send Message to requestor and manager
- Send Message to mfa reset channel
- Email (Requires Azure Communication Services)

#### Create a Chat
Members to add needs:
body('Parse_Manager_Response')?['mail']
body('Get_target_information_from_EM')?['Email']

Chat title:
body('Get_target_information_from_EM')?['DisplayName']

![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/createachat.png)

---
#### Send Message to Requestor and Manager

replace(body('Create_a_chat')?['id'], '\n', '')
body('Parse_TAP_Response')?['temporaryAccessPass']

![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/teamsmessagerequestormanager.png)

---
#### Send Message to mfa Reset Channel

body('Parse_Manager_Response')?['mail']
body('Get_target_information_from_EM')?['Email']
body('Parse_TAP_Response')?['temporaryAccessPass']

![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/teamschannel.png)
	
---
#### Email


![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/sendemail.png)

---

#### Full Designer View
How the full logic app looks with communication added

![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/sendemail.png)

## Acknowledgements

- [Nathan Mcnulty](https://github.com/nathanmcnulty)
