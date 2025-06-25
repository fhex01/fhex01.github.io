# Technical Analysis of ESC16 on ADCS

## Introduction

The `ESC16` vulnerability highlights a critical misconfiguration in **Active Directory Certificate Services (ADCS)**, where a **Certification Authority (CA)** is globally configured to omit the `szOID_NTDS_CA_SECURITY_EXT` security extension (`OID 1.3.6.1.4.1.311.25.2`) from all issued certificates. Introduced with the May 2022 security update (`KB5014754`), this extension is essential for **strong certificate-to-SID binding**, allowing **Domain Controllers (DCs)** to reliably associate certificates with user or computer accounts based on their Security Identifier (SID).

When this extension is **disabled at the CA level**, either intentionally (to bypass compatibility issues) or due to the CA being **unpatched**, every certificate it issues lacks the required SID binding. This causes all templates to behave as if they had the `CT_FLAG_NO_SECURITY_EXTENSION` flag set, reminiscent of the `ESC9` vulnerability, but applied **globally**.

The result is a weakened authentication model. Unless DCs are configured in **Full Enforcement mode** (`StrongCertificateBindingEnforcement = 2`), they fall back to **legacy mapping methods** such as UPN or DNS name matching, reopening attack paths similar to those exploited by `Certifried` (`CVE-2022-26923`).

This article provides a **technical deep dive** into `ESC16`: its underlying mechanics, how it can be detected (with `Certipy`), methods of exploitation (including in combination with `ESC6`), and the key mitigations necessary to restore a secure ADCS environment.

## In progress..
