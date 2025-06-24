# Exploring Windows Coercion via Remote GetAdminSection Calls

## Introduction

Modern Windows environments expose a wide variety of RPC interfaces, many of which are overlooked in typical security assessments. Some of these interfaces, originally designed for remote administration, can be abused to provoke outbound authentication attempts from a Windows machine, a technique often referred to as **coercion**. This method plays a critical role in relay attacks, credential harvesting, and lateral movement scenarios.

In this blog post, we focus on one such interface: the `MC-IISA` protocol, which underpins remote administration for Internet Information Services (IIS). By interacting with this protocol over a named pipe via SMB, it's possible to trigger a remote call to `GetAdminSection`, a function that, when crafted carefully, will cause the target machine to attempt authentication to a destination of our choosing.

Our test environment consists of a Windows Server (192.168.0.1) and an attacker machine (192.168.0.2) running Responder to capture incoming authentication attempts. By establishing an RPC session to the `MC-IISA` interface and calling `GetAdminSection` with specific parameters, we can force the target to contact the attacker's IP, effectively coercing it into revealing authentication material that can be captured or relayed.

In the sections that follow, we’ll walk through how this interaction works under the hood, how to trigger it programmatically, and why this seemingly harmless RPC function can become a powerful tool in the hands of an attacker.

## In production...
