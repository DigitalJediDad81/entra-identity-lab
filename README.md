# Entra Identity Lab — casa.lab / juanlab.cloud

A hands-on identity administration portfolio covering hybrid Active Directory/Entra ID federation, workload identity concepts, and Azure-native credential-free authentication patterns — built as both an SC-300 study lab and a live architecture demo.

## Why this exists

Identity administration is easy to describe in the abstract and hard to actually demonstrate. This repo is the record of building it for real: a themed AD environment with intentional attack paths, two parallel cloud federation methods running side by side, and a working proof that an Azure workload can authenticate to a protected resource without ever touching a stored secret.

## Contents

| Doc | What it covers |
|---|---|
| [`azure-entra-star-wars-lab-setup.md`](./azure-entra-star-wars-lab-setup.md) | The foundational build: AD seeded with a themed org structure and intentional weaknesses, dual federation (Entra Connect + Okta AD Agent) into a single Entra ID tenant, SAML SSO to Cloudflare Zero Trust, and Azure Arc onboarding with scoped telemetry collection |
| [`workload-identity-lab.md`](./workload-identity-lab.md) | App registration vs. service principal, delegated vs. application Graph permissions, and a full walkthrough of a system-assigned managed identity authenticating to Key Vault with zero stored credentials — including the real troubleshooting log (RBAC data-plane vs. control-plane, provider registration, CLI extension gaps) |

## Architecture

```
Proxmox VE (2x mini PCs)
   │
   ├── Windows DCs + Linux hosts, domain: casa.lab
   │      │
   │      ├──► Entra Connect ──► Entra ID (juanlab.cloud) ──► Microsoft 365
   │      │
   │      └──► Okta AD Agent ──► Okta Workforce ──SAML──► Cloudflare Zero Trust
   │
   └──► Azure Arc ──► Azure Monitor Agent ──► Data Collection Rules ──► Sentinel

Azure (separate, cloud-native):
   App Registration → Service Principal → Delegated + Application Graph permissions
   Automation Account → System-Assigned Managed Identity → RBAC → Key Vault (no secrets)
```

## What's demonstrated

- **Hybrid identity**: on-prem AD synced two ways into the cloud (Entra Connect and Okta), proving out both the standard enterprise pattern and a secondary IAM platform
- **Federated SSO**: SAML from Okta through to a Zero Trust proxy, with group-based and email-based access policy enforcement
- **Workload identity fundamentals**: the application object / service principal split, delegated vs. application permission consent flows, and RBAC vault access models
- **Credential-free authentication**: a working managed identity pulling a Key Vault secret with no client secret, certificate, or password anywhere in the code or configuration

## Notes on documentation style

Both docs keep the troubleshooting record intact rather than presenting a cleaned-up "it just worked" narrative — unresolved issues (like a SAML group-matching quirk in Cloudflare) and real errors hit along the way (RBAC permission gaps, CLI extension limitations, provider registration failures) are documented alongside the fixes. That's intentional: it's a more honest and more useful record, both for revisiting the lab later and for interview conversations.

## About

Built by Juan Rojas — Information Security Manager, ~20 years across IAM, PAM, vulnerability management, and cloud security (Azure/M365/Entra ID). This lab supports SC-300 certification prep and a consulting practice ([Intihuatana Cyber LLC](#)) focused on Microsoft-stack security for SMBs.
