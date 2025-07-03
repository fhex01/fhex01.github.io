# Targeted Timeroasting: Weaponizing Computers with Controlled UAC Flags

In this article, we will explore how the behavior of the **NTP protocol**, or more specifically, its Microsoft extension, **MS-SNTP**, can be abused to obtain *crackable hashes* for accounts. And not just any accounts: we’re talking about **user accounts**, not **machine accounts**. We'll dive deep into the internals of the **MS-SNTP extension** and into the depths of this still-overlooked attack vector. 

First, we'll take a look at what the **NTP protocol** actually is. Its full name, **Network Time Protocol**, refers to the mechanism that allows systems to *synchronize* their clocks over a network. Time synchronization is critical in many areas of computing, particularly in **Kerberos authentication**, where even slight discrepancies can lead to failed logins or other issues. Understanding how *NTP* works is essential before we dive into its Microsoft-specific implementation, *MS-SNTP*, and how it can be exploited.

![image](https://github.com/user-attachments/assets/5c89ac10-90a0-404a-9353-901649988a41)

In a Windows domain environment, time synchronization follows a strict hierarchy to ensure consistency across all systems. At the top of this hierarchy is the **PDC Emulator**, which acts as the authoritative time source for the domain. Referred to as "Logical Stratum 1," it either synchronizes with an external time source (such as GPS or a hardware clock) or operates independently if configured as the root of trust.

Other **Domain Controllers (DC01)** within the domain synchronize their time with the PDC Emulator at regular intervals. In this configuration, synchronization is handled by the Windows Time service (`w32time`), which by default performs time synchronization approximately every hour. This communication is referred to as an **NTP Sync**, and in the context of a domain, it is secured using Microsoft’s extension of NTP: **MS-SNTP**. This extension integrates Kerberos authentication into the time sync process, ensuring that only trusted domain members can participate in and verify the time hierarchy.

Workstations such as **WS01** and **WS02** synchronize their clocks with their respective domain controllers. When there is a significant difference between the local clock and the DC's time, the client will perform an **NTP Step**, an immediate correction to bring the local time into alignment. This typically occurs at system startup or when a workstation first joins the domain. If the time offset is small, a gradual adjustment (slew) is used instead, though this is not depicted in the diagram.

Overall, the diagram illustrates how **time flows downward** from a trusted root, the PDC Emulator, through the domain controllers and ultimately to the workstations. The note at the bottom highlights a key security point: **all time synchronization within the domain is protected using MS-SNTP**, ensuring authenticity and preventing time-based attacks that could undermine services like Kerberos.

## Working on...
