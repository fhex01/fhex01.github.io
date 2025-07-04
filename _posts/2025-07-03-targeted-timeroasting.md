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

A Microsoft extension (MS-SNTP) adds a MAC authenticator calculated from the hash of a computer account, along with a **Relative Identifier (RID)** in the request. So, an attacker can send NTP requests specifying RIDs linked to machine accounts and receive hashes in response, which can then be cracked offline. The goal would be to send a time synchronization request to the Domain Controller, including the RID of the targeted machine account. The Domain Controller will then generate the MAC based on the provided RID, therefore using the **targeted machine account**.

![image](https://github.com/user-attachments/assets/98fe2782-8fa3-4220-974a-09df869e637e)

Above, you can see a **Wireshark** capture showing the process, along with a **failure**, of a time synchronization request using the **`NTP`** protocol (**`MS-SNTP`**). You can observe that in the response sent by the **NTP server** (**the DC in our case**), the **MAC** is indeed present. Why didn’t the first attempt work? Because it couldn’t generate a **MAC**, since we had deliberately provided an incorrect **`RID`** for testing purposes.

> Info: The **MAC** (Message Authentication Code) is a cryptographic signature added to the **`NTP`** response to ensure the **integrity** of the **NTP message** and the **authenticity** of the sender (the **Domain Controller**).
