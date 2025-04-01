# Secure MFA Resets with Passkeys and Entra Governance

Provides a process using Entra access packages, logic apps, and corresponding conditional access policies
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
This will vary based on what pieces of your implementation you use but it involves at least 1 logic app

## Directions

All of these steps as far as I'm aware can be done with Graph Powershell SDK however these directions will be a mix a both.

1) In the Azure or Entra portal navigate to Identity Governance and create a dedicated Catalog which will contain your access packages and custom extensions (aka logic apps) 

2) Once catalog is created create your custom extension with the following properties
- Extension Type: Request workflow (triggered when an access package is requested, approved, granted, or removed)
- Launch and continue
- Create logic app "Yes" 

3) After the custom extension/logic app is created it needs to be granted the access to make authentication changes and teams chat creation. In the logic app's settings choose identity and create a "System assigned" identity. Record its ID

4) Using the Graph Powershell SDK determine the custom extension's ID [Credit]
- 


## Acknowledgements

- [Awesome Readme Templates](https://awesomeopensource.com/project/elangosundar/awesome-README-templates)
- [Awesome README](https://github.com/matiassingers/awesome-readme)
- [How to write a Good readme](https://bulldogjob.com/news/449-how-to-write-a-good-readme-for-your-github-project)

## Acknowledgements

- [Awesome Readme Templates](https://awesomeopensource.com/project/elangosundar/awesome-README-templates)
- [Awesome README](https://github.com/matiassingers/awesome-readme)
- [How to write a Good readme](https://bulldogjob.com/news/449-how-to-write-a-good-readme-for-your-github-project)
