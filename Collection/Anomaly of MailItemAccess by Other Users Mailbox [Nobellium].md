# Anomaly of MailItemAccess by Other Users Mailbox [Solorigate].
This query looks for users accessing multiple other user's mailboxes or accessing multiple folders in another users mailbox

Query is inspired by:
https://github.com/Azure/Azure-Sentinel/blob/master/Hunting%20Queries/OfficeActivity/AnomolousUserAccessingOtherUsersMailbox.yaml
## Query
```
// Adjust this value to exclude historical activity as known good
let LookBack = 30d;
// Adjust this value to change hunting timeframe
let TimeFrame = 14d;
// Adjust this value to alter how many mailbox (other than their own) a user needs to access before being included in results
let UserThreshold = 1;
// Adjust this value to alter how many mailbox folders in other's email accounts a users needs to access before being included in results.
let FolderThreshold = 5;
let relevantMailItems = materialize (
    CloudAppEvents
    | where Timestamp > ago(LookBack)
    | where ActionType == "MailItemsAccessed"
    | where RawEventData['ResultStatus'] == "Succeeded"
    | extend UserId = tostring(RawEventData['UserId'])
    | extend MailboxOwnerUPN = tostring(RawEventData['MailboxOwnerUPN'])
    | where tolower(UserId) != tolower(MailboxOwnerUPN)
    | extend Folders = RawEventData['Folders']
    | where isnotempty(Folders)
    | mv-expand parse_json(Folders)
    | extend foldersPath = tostring(Folders.Path)  
    | where isnotempty(foldersPath)
    | extend ClientInfoString = RawEventData['ClientInfoString']   
    | extend MailBoxGuid = RawEventData['MailboxGuid']   
    | extend ClientIP = iif(IPAddress startswith "[", extract("\\[([^\\]]*)", 1, IPAddress), IPAddress)
    | project Timestamp, ClientIP, UserId, MailboxOwnerUPN, tostring(ClientInfoString), foldersPath, tostring(MailBoxGuid)    
);
let relevantMailItemsBaseLine = 
    relevantMailItems
    | where Timestamp between(ago(LookBack) ..  ago(TimeFrame))    
    | distinct MailboxOwnerUPN, UserId;
let relevantMailItemsHunting = 
    relevantMailItems
    | where Timestamp between(ago(TimeFrame) .. now())
    | distinct ClientIP, UserId, MailboxOwnerUPN, ClientInfoString, foldersPath, MailBoxGuid; 
relevantMailItemsBaseLine 
    | join kind=rightanti relevantMailItemsHunting
    on MailboxOwnerUPN, UserId
    | summarize FolderCount = dcount(tostring(foldersPath)), 
                UserCount = dcount(MailBoxGuid), 
                foldersPathSet = make_set(foldersPath), 
                ClientInfoStringSet = make_set(ClientInfoString), 
                ClientIPSet = make_set(ClientIP), 
                MailBoxGuidSet = make_set(MailBoxGuid), 
                MailboxOwnerUPNSet = make_set(MailboxOwnerUPN)  
            by UserId
    | where UserCount > UserThreshold or FolderCount > FolderThreshold
    | extend Reason = case( 
                            UserCount > UserThreshold and FolderCount > FolderThreshold, "Both User and Folder Threshold Exceeded",
                            FolderCount > FolderThreshold and UserCount < UserThreshold, "Folder Count Threshold Exceeded",
                            "User Threshold Exceeded"
                            )
    | sort by UserCount desc
```
## Category
This query can be used to detect the following attack techniques and tactics ([see MITRE ATT&CK framework](https://attack.mitre.org/)) or security configuration states.
| Technique, tactic, or state | Covered? (v=yes) | Notes |
|------------------------|----------|-------|
| Initial access |  |  |
| Execution |  |  |
| Persistence |  |  | 
| Privilege escalation |  |  |
| Defense evasion |  |  | 
| Credential Access |  |  | 
| Discovery |  |  | 
| Lateral movement |  |  | 
| Collection | V |  | 
| Command and control |  |  | 
| Exfiltration | |  | 
| Impact |  |  |
| Vulnerability |  |  |
| Misconfiguration |  |  |
| Malware, component |  |  |

## Contributor info
**Contributor:** Stefan Sellmer
**GitHub alias:** @stesell
**Organization:** Microsoft 365 Defender
**Contact info:** stesell@microsoft.com