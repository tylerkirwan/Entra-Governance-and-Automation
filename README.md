# :lock: Secure MFA Resets with Passkeys and Entra Governance:lock:

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
    style="width:45%; max-width:300px; margin-top:-60px;"
  />
</div>

## :shield:Purpose and Goals:shield:

- Reduce likelihood of social engineering a helpdesk member by adding approval stages, visibility, and reporting capabilities to MFA resets
- Privileged roles are not required for helpdesk personnel 
- Streamlines MFA resets via Entra Governance access packages and saves time with your support teams and end users
- Enrolls new MFA methods into phish resistance by default
- No exchanging or sharing of passwords
- End user, manager, helpdesk, and security are all aware and can be audited or integrated into SIEM/SOAR with reporting capabilities
## FAQ & Prerequisites 

#### What licensing is required?

Security E3 and or Entra P2

#### :key:What access is needed?

Privileged Role Administrator and Privileged Authentication Administrator or Global Admin (for setup)

#### What Resources?
This will vary based on what pieces of your implementation you use but it involves at least 1 logic app. Optional: Azure Communication Services for delivering email and additional logic app for temporary access cleanup.

## :open_book:Directions:open_book:
Broken into 4 sections. Part 3 requires the most time to setup.
### :envelope:Part 1 - Create Access Package and Assign Permissions:envelope:

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
9) Create or assign the likey temporary security group you will grant as part of the access package. This security group is used to apply a specific conditional access policy for passkey enrollment within Microsoft Authenticator. 

10) Create the Access Package Policy. Customize as you see fit however it's strongly suggested to utilize the multistage approvals. I also expire the access package quickly (more on this in Part 4). Below is how I configured the policy:


-  Pictures of policy
### :gear:Part 2 - Apply logic app json :gear:
---
Use [LAmfa.json](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/LAmfa.json) This however is not the full logic app, the connection and communication actions need to be added by the designer. Part 3 will explain the communication pieces that I found easiest to add through the logic app designer.

[Designer](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/fullwithoutcomm.png)


---
### :telephone_receiver:Part 3 - Apply logic app Communication Actions:telephone_receiver:
1) In the logic app designer there will be 4 actions to add. Ensure you have set up Azure Communication Services if you plan on following exactly
- Create a Chat
- Send Message to requestor and manager
- Send Message to mfa reset channel
- Email (Requires Azure Communication Services)

#### Create a Chat
Members to add needs:
```plaintext
body('Parse_Manager_Response')?['mail']
body('Get_target_information_from_EM')?['Email']
```
Chat title:
```plaintext
body('Get_target_information_from_EM')?['DisplayName']
```
![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/createachat.png)

---
#### Send Message to Requestor and Manager
```plaintext
replace(body('Create_a_chat')?['id'], '\n', '')
body('Parse_TAP_Response')?['temporaryAccessPass']
```
![Send Teams Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/teamsmessagerequestormanager.png)

---
#### Send Message to mfa Reset Channel
```plaintext
body('Parse_Manager_Response')?['mail']
body('Get_target_information_from_EM')?['Email']
body('Parse_TAP_Response')?['temporaryAccessPass']
```
![Send Teams Channel](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/teamschannel.png)
	
---
#### Email
```plaintext
body('Parse_Manager_Response')?['mail']
body('Get_target_information_from_EM')?['Email']
![Create Chat](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/sendemail.png)
```
![Email](https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/sendemail.png)
---

#### Full Designer View
How the full logic app looks with communication added

<div style="display: flex; align-items: flex-start;">
  <img 
    src="https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/logicapptop.png?raw=true" 
    alt="Top" 
    style="width:45%; max-width:300px; margin-right:10px;"
  />
  <img 
    src="https://github.com/tylerkirwan/Entra-Governance-and-Automation/blob/main/images/logicappbottom.png?raw=true" 
    alt="Bottom" 
    style="width:45%; max-width:300px; margin-top:-60px;"
  />
</div>

### :key:Part 4 - Conditional Access Policy(s) :key:

This part will depend on your org's requirements and policies. I've used this MFA reset flow as a way to register FIDO strength authentication within mobile devices with Microsoft Authenticator (the easy way) and then shortly after passkey enrollment enforce device compliance. 

1) Create an authentication strength that includes the security key phish resistance method (check box Authenticator) AND Temporary Access Pass method.
2) Create a dedicated Security Group such as "CA_temp_Passkey Enrollment" (applied to and granted as part of the access package in Part 1)
3) The conditions of the conditional access policy should be scoped to iOS and Android operating systems
4) Set the policy to GRANT and require the authentication strength that was created in step 1 (don't require device compliance unless all the mobile devices are being registered/enrolled into MDM, the devices will fail to add a passkey within the authenticator app if unenrolled)
5) I've set the security group "CA_temp_Passkey Enrollment" as an exclusion to my base policy requiring device compliance. When the access package expires within the hour the member will be removed from the security group and back to enforcing the base device compliance policy.

<small><strong><em>Note: The only requirement is enforcing phish resistance strength that includes temporary access pass. All other configurations should be adjusted to meet your org's unique requirements and accepted risk policies. :warning:Also consider restricting the security group with Entra Administrative units; the MI will need to be included.</em></strong></small>


## :star:Acknowledgements:star:
This logic app was built off of much of what Nathan provided for rolling out passkeys with temporary access packages. Along with numerous other resources provided by Merill and Joshua Fernando. Could not have finished this flow without resources like these. Also check out Nathan's operational groups for ways to manage the auth strength policies with conditional access
- [Nathan Mcnulty](https://github.com/nathanmcnulty)
- [Entra News](https://entra.news/)
