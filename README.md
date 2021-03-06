# Limiting the exposures of credentials when connecting to servers

Microsoft guidance from [here](https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material). Explains the credential exposure associated with using different administrative tools for remote administration. 

## Administration methods

| Connection   method                              | Logon type            | Reusable credentials on   destination | Comments                                                                                                                                          |
|--------------------------------------------------|-----------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Log on at console                                | Interactive           | v                                     | Includes   hardware remote access / lights-out cards and network KVMs.                                                                            |
| RUNAS                                            | Interactive           | v                                     |                                                                                                                                                   |
| RUNAS /NETWORK                                   | NewCredentials        | v                                     | Clones   current LSA session for local access, but uses new credentials when   connecting to network resources.                                   |
| Remote Desktop (success)                         | RemoteInteractive     | v                                     | If the   remote desktop client is configured to share local devices and resources,   those may be compromised as well.                            |
| Remote Desktop (failure - logon type was denied) | RemoteInteractive     | -                                     | By   default, if RDP logon fails credentials are only stored very briefly. This   may not be the case if the computer is compromised.             |
| Net use * \\SERVER                               | Network               | -                                     |                                                                                                                                                   |
| Net use * \\SERVER /u:user                       | Network               | -                                     |                                                                                                                                                   |
| MMC snap-ins to remote computer                  | Network               | -                                     | Example:   Computer Management, Event Viewer, Device Manager, Services                                                                            |
| PowerShell WinRM                                 | Network               | -                                     | Example:   Enter-PSSession server                                                                                                                 |
| PowerShell   WinRM with CredSSP                  | NetworkClearText      | v                                     | New-PSSession   server                                                                                                                            |
|                                                  |                       |                                       | -Authentication   Credssp                                                                                                                         |
|                                                  |                       |                                       | -Credential   cred                                                                                                                                |
| PsExec without explicit creds                    | Network               | -                                     | Example:   PsExec \\server cmd                                                                                                                    |
| PsExec   with explicit creds                     | Network + Interactive | v                                     | PsExec   \\server -u user -p pwd cmd                                                                                                              |
|                                                  |                       |                                       | Creates   multiple logon sessions.                                                                                                                |
| Remote Registry                                  | Network               | -                                     |                                                                                                                                                   |
| Remote Desktop Gateway                           | Network               | -                                     | Authenticating   to Remote Desktop Gateway.                                                                                                       |
| Scheduled task                                   | Batch                 | v                                     | Password   will also be saved as LSA secret on disk.                                                                                              |
| Run tools as a service                           | Service               | v                                     | Password   will also be saved as LSA secret on disk.                                                                                              |
| Vulnerability scanners                           | Network               | -                                     | Most   scanners default to using network logons, though some vendors may implement   non-network logons and introduce more credential theft risk. |

## Logon types 

| Logon   type                          | #  | Authenticators accepted                   | Reusable credentials in LSA   session                                 | Examples                                                                                               |
|---------------------------------------|----|-------------------------------------------|-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Interactive   (a.k.a., Logon locally) | 2  | Password,   Smartcard, other              | Yes                                                                   | Console   logon;  RUNAS; Hardware remote control solutions   (such as Network KVM or Remote Access / Lights-Out Card in server)   IIS Basic Auth (before IIS 6.0)|                                           |
| Network                               | 3  | Password,  NT Hash,  Kerberos   ticket    | No   (except if delegation is enabled, then Kerberos tickets present) | NET USE; RPC calls;  Remote registry;      IIS integrated Windows auth;      SQL Windows auth;    |
| Batch                                 | 4  | Password   (usually stored as LSA secret) | Yes                                                                   | Scheduled   tasks                                                                                      |
| Service                               | 5  | Password   (usually stored as LSA secret) | Yes                                                                   | Windows   services                                                                                     |
| NetworkCleartext                      | 8  | Password                                  | Yes                                                                   | IIS   Basic Auth (IIS 6.0 and newer);   Windows   PowerShell with CredSSP                                         |
| NewCredentials                        | 9  | Password                                  | Yes                                                                   | RUNAS   /NETWORK                                                                                       |
| RemoteInteractive                     | 10 | Password,   Smartcard,     other               | Yes                                                                   | Remote   Desktop (formerly known as "Terminal Services")                                               |

## Domain Admin accounts 

The DA group should be added to the following user rights in Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\User Rights Assignments:
- Deny access to this computer from the network
- Deny log on as a batch job
- Deny log on as a service
- Deny log on locally
- Deny log on through Remote Desktop Services user rights

https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-f--securing-domain-admins-groups-in-active-directory

https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/initially-isolate-tier-0-assets-with-group-policy-to-start/ba-p/1184934

## Remediation 

* What is the attacker's objective? 
* What system(s) does the attacker need to access to achieve those goals? 
* What account(s) would the attacker need to access those systems? 
* On what systems are the credentials for those accounts exposed?  
* How are those systems protected? 
* How could the attacker gain access to those systems?
* What other attack paths exist for an attacker to compromise those credentials? 
* How can we mitigate the attacker gaining access to those accounts? 

## Remote desktop Restricted Admin mode

> To prevent credentials from being stored on the remote machine, Restricted Admin changes the Remote Desktop protocol so that it uses network logons rather than an interactive logons for authentication. With this protection, establishing an RDP session will not require the associated password to be provided; instead the NT hash or Kerberos ticket of the user will be used for authentication.

> Restricted Admin for both clients and servers has been backported to Windows 7 and Server 2008 but remains disabled by default on most standard Windows builds due to a couple of caveats surrounding its use.

> We consider Restricted Admin mode an effective control to prevent the spread of administrative credentials across the network, thus making lateral movement more difficult, particularly in environments where tiering is not yet in place or has not yet been fully implemented. 

https://labs.f-secure.com/blog/undisable/#:~:text=Restricted%20Admin%20mode%20for%20RDP,enabling%20many%20lateral%20movement%20techniques.&text=Because%20it%20uses%20network%20rather,the%20Remote%20Desktop%20protocol%20itself.
