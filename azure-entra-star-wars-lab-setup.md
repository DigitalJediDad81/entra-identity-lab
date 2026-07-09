# Hybrid Identity Lab: Azure/Entra Setup & Star Wars-Themed Resource Onboarding

**Author:** Juan Rojas
**Environment:** Home lab — 2x Proxmox VE 9.1.9 mini PCs (Intel i5-14600, 32GB RAM, 1TB NVMe each)
**Tenant:** `juanlab.cloud` (Entra ID, Microsoft 365 Business Basic)
**Domain:** `casa.lab` (Active Directory, migrated from earlier `lab.local`)

## Purpose

Build a realistic hybrid identity environment — on-prem AD, cloud identity federation, SSO, and cloud-native monitoring — that doubles as both a red/blue-team detection lab and a live architecture demo for interviews and consulting pitches. The Star Wars theming isn't cosmetic: it makes the environment memorable and easy to narrate ("Kylo Ren is AS-REP roastable") while every object and misconfiguration maps to a specific, intentional detection or attack path.

## Architecture Overview

```
Proxmox VE (2x mini PCs)
   │
   ├── Windows DCs (WINDC1, WINDC2) ──┐
   │                                    │
   ├── Linux hosts (linuxcore1/2) ─────┤
   │                                    │
   └── Domain: casa.lab                │
         │                             │
         ├──► Entra Connect ──► Entra ID (juanlab.cloud) ──► Microsoft 365
         │
         └──► Okta AD Agent ──► Okta Workforce ──SAML──► Cloudflare Zero Trust ──► sso-test.juanlab.cloud
         │
         └──► Azure Arc ──► Azure Monitor Agent ──► Data Collection Rules ──► Log Analytics ──► Sentinel
```

Two parallel identity paths were deliberately built side by side: **Entra Connect** (the standard hybrid identity path most enterprises use) and **Okta AD Agent + delegated auth** (to gain hands-on Okta federation experience for IAM breadth). Both sync from the same source of truth in AD.

---

## Part 1 — Active Directory Foundation

Domain controllers seeded with a Star Wars-themed environment via an idempotent PowerShell script (`seed-casa-lab.ps1`), organized into five faction OUs:

| OU | Members |
|---|---|
| Rebellion | Luke, Leia, Han, Chewbacca, Lando, Wedge, Mon Mothma, Ackbar, Jyn, Cassian |
| Empire | Palpatine, Vader, Tarkin, Krennic, Thrawn, Kylo Ren |
| JediOrder | Yoda, Obi-Wan, Mace Windu, Qui-Gon, Ahsoka, Kanan, Ezra |
| BountyHunters | Boba, Jango, Bossk, Dengar, Cad Bane |
| Smugglers | Greedo, Hondo, Maz, Sana |

**Service accounts:** `svc-r2d2`, `svc-c3po`, `svc-bb8`, `svc-k2so`, `svc-ig88`, `svc-hk47`

**Security groups:** `JediCouncil`, `SithLords`, `RebelLeaders`, `AllRebels`, `AllImperials`, `FileShare-Tatooine`, `FileShare-Hoth`, `VPN-Users`

**Pre-staged computer objects:** Servers (TATOOINE, CORUSCANT, HOTH, NABOO, ENDOR, DAGOBAH), Workstations (FALCON, XWING, TIE-FIGHTER, SLAVE-1, GHOST, RAZOR-CREST)

### Intentional weaknesses (by design, for detection engineering practice)

| Weakness | Accounts | Feeds detection for |
|---|---|---|
| Kerberoastable SPNs | `svc-c3po`, `svc-r2d2`, `svc-k2so`, `svc-hk47` | Kerberoasting rule (T1558.003) |
| AS-REP roastable (no preauth) | `svc-ig88`, Kylo Ren, Boba Fett | AS-REP Roasting rule (T1558.004) |
| DES-only encryption | Cad Bane, Greedo, `svc-k2so` | Weak-crypto detection |
| Sensitive info in description fields | Han Solo, Krennic, `svc-bb8`, `svc-hk47` | Credential-exposure hunting queries |
| Nested privilege escalation paths | JediCouncil → Domain Admin, SithLords → Enterprise Admin, `svc-k2so` → Domain Admin | BloodHound path-finding practice |
| Passwords never expire | All `svc-*`, plus Han, Vader, Palpatine, Yoda, Kylo | Stale-credential hygiene findings |

Every misconfiguration is commented in the seed script with the exact detection it's meant to trigger — the script functions as living documentation, not just setup automation.

---

## Part 2 — Cloud Identity Federation

### Entra Connect (primary hybrid identity path)

Installed Entra Connect on a dedicated sync server, syncing all `casa.lab` users and groups to the `juanlab.cloud` Entra ID tenant. This is the standard hybrid identity pattern most enterprises run, and the one most directly relevant to the SC-300 exam and day-to-day M365/Entra admin work.

### Okta AD Agent (secondary path, for IAM platform breadth)

Installed the Okta AD Agent on the same sync server using a dedicated `svc-okta` service account, configured for delegated authentication (Okta validates passwords against AD in real time rather than syncing password hashes). Notable troubleshooting during setup:

- Cloud Shell sessions are ephemeral and silently discard in-progress script edits — moved to a persistent editing environment for anything multi-step
- The agent installer requires signing in with the **domain** account (`LAB\labadmin`), not a local Windows account — using the local account causes a silent authentication mismatch
- Installer ships as both MSI and EXE; the two aren't interchangeable in all deployment contexts
- `juan.admin@juanlab.cloud` needed an actual mailbox before certain admin flows would complete — resolved by adding an M365 Business Basic license

All 32 lab users imported into Okta from AD with delegated authentication confirmed working.

### SAML SSO — Okta to Cloudflare Zero Trust

Configured a test relying-party application (`sso-test.juanlab.cloud`) to prove out full SSO from AD → Okta → Cloudflare Zero Trust via SAML. DNS hosting for `juanlab.cloud` was moved from GoDaddy to Cloudflare to support this.

**Validated working:**
- Group attributes (e.g., `AllRebels`) correctly present in the SAML assertion
- Access policy enforcement by group and by email
- Faction-based access control proven end-to-end: Rebellion members granted access, Vader denied

**Known issue (kept as a documented gap, not silently worked around):** Cloudflare's SAML Group selector failed to match the `AllRebels` group name despite the attribute being confirmed present and correctly formatted in the SAML assertion (verified via Okta's SAML trace tool). Workaround: switched enforcement to an email-based allow-list. Suspected UI/version-specific matching quirk on Cloudflare's side — flagged for deeper troubleshooting later rather than treated as resolved.

### Break-glass account

A dedicated break-glass admin (`breakglass@juanlab.onmicrosoft.com`) was created and excluded from all Conditional Access and federation dependencies **before** any federation changes were made — standard practice to guarantee tenant access survives a misconfigured SSO or sync failure.

---

## Part 3 — Azure Arc Onboarding

To feed telemetry into Sentinel, all on-prem `casa.lab` machines needed to become Azure-managed resources first — Azure Monitor Agent only deploys to Azure VMs or Arc-enabled servers.

### Onboarding sequence (per machine)

```powershell
# Generated from: Azure Portal → Azure Arc → Machines → Add a single server → Generate script
# Subscription: PAYG | Resource Group: rg-sentinel | Region: East US 2 | Connectivity: Public endpoint

# Script downloads and installs the Connected Machine agent (azcmagent),
# then runs azcmagent connect with tenant/subscription/RG/region pre-filled.
# Requires interactive device-code sign-in as juan.admin (Owner role).
```

Verified via:
```powershell
azcmagent show
# Agent Status: Connected
```

or in-portal: Azure Arc → Machines → status = **Connected** (green).

All four hosts (2 Windows DCs, 2 Linux hosts) onboarded this way. Arc itself carries no cost — billing only begins with Log Analytics data ingestion, which the Sentinel trial covers at lab scale.

### Data Collection Rules — scoped, not "collect everything"

Rather than ingesting full Security/Syslog streams, DCRs were scoped tightly to the event IDs and facilities that feed the five detection rules:

- `dcr-casalab-security` → Windows Event IDs **4624, 4625, 4768, 4769, 4776** only
- `dcr-casalab-linux-syslog` → `auth`, `authpriv` facilities only

Associating a DCR with an Arc machine auto-deploys AMA — no separate agent installation step required.

### Verification layers (the actual go/no-go gate used before enabling Defender for Servers)

| Layer | Check | Tool |
|---|---|---|
| Arc connectivity | All 4 machines show `Connected` | Azure Resource Graph / portal |
| AMA extension | Extension installed and running | Resource Graph / per-machine Extensions blade |
| DCR association | Each machine associated with the correct DCR | `az monitor data-collection rule association list` |
| Data flowing | Heartbeat table shows recent rows tagged Azure Monitor Agent; SecurityEvent has both DCs; Syslog has both Linux hosts | Log Analytics KQL |

**Key lesson:** heartbeat alone is a false sense of security. AMA can heartbeat successfully while the actual event tables stay empty if the DCR isn't associated or the data-source filter is misconfigured — heartbeat confirms the agent is alive, not that telemetry is flowing. All four layers have to be green before trusting the pipeline.

**Legacy agent conflict check:** the old Log Analytics (MMA) agent was retired industry-wide in August 2024. If a machine's `Category` field shows "Direct Agent" instead of "Azure Monitor Agent," that's leftover MMA double-reporting and needs to be removed to avoid duplicate/confusing telemetry.

---

## Outcome

With this foundation in place — AD seeded, dual identity federation working, all four hosts Arc-connected with scoped telemetry flowing — the environment became the platform for the five MITRE-mapped Sentinel detection rules (Kerberoasting, AS-REP Roasting, Password Spray, SSH Brute Force, Linux Account Lifecycle Abuse), each validated against real attack traffic generated in this exact lab.

## Interview / Portfolio Framing

> "I built a hybrid identity lab from the ground up — Active Directory seeded with realistic organizational structure and intentional attack paths, federated two ways (Entra Connect and Okta) into a single Entra ID tenant, SSO'd out to a Zero Trust proxy, and Arc-onboarded every host so I could feed real telemetry into Sentinel. I didn't stop at 'it works' — I built the verification layers to prove data was actually flowing, not just that agents were installed, and I documented the one thing that didn't work cleanly (the Cloudflare SAML group matching) instead of hiding it."

That last sentence is deliberate — willingness to name an unresolved issue, rather than presenting a lab as flawlessly complete, reads as more credible in a technical interview than a story with no rough edges.
