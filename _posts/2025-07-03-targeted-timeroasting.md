# Targeted Timeroasting: Weaponizing Users and Computers with Controlled UAC Flags

Do you know **NTP**, or its Microsoft Windows variant, **MS-SNTP**? Well, I racked my brain trying to figure out what it was, and I found it really interesting. In this article, we will focus exclusively on the **MS-SNTP** protocol. We will explore how this protocol could potentially be exploited to steal authentication or identification credentials. 

First, we will examine how the **MS-SNTP** protocol works and how it is used within **Active Directory** infrastructures. We will also highlight its importance in such environments. Next, we will explore how the **MS-SNTP** protocol can be abused by proposing a variation of an existing attack. We will develop a proof-of-concept exploit (`PoC`) and then suggest possible **remediations** to protect against this attack.

## What is the MS-SNTP protocol, and how does it work?

"*The **Simple Network Time Protocol** (abbreviated as **SNTP**) is a protocol from the Internet protocol suite used for synchronizing system clocks across networks. The current version, **SNTPv4**, supports both **IPv4** and **IPv6** networks and is described in [RFC 4330](https://datatracker.ietf.org/doc/html/rfc4330).*"

Its main purpose is to maintain accurate time synchronization across networked devices, ensuring consistent timestamps and coordinated operations. In **Active Directory** infrastructures, its use is similar: it ensures time synchronization across all computers within a Windows domain. This synchronization is especially important for **Kerberos authentication**, which, during the **pre-authentication** phase, checks the timestamp provided by the client machine. 

![image](https://github.com/user-attachments/assets/3d5e8fe6-477b-45a6-be24-be886235781e)

If there is a time difference greater than 5 minutes between the client and the **domain controller**, Kerberos pre-authentication fails. **MS-SNTP** ensures that all systems within an **Active Directory** domain are synchronized to a reliable time source.
