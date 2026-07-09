# Workload Identity Lab: App Registrations, Service Principals & Managed Identities

**Author:** Juan Rojas
**Context:** SC-300 (Microsoft Identity and Access Administrator) retake prep — Plan and Implement Workload Identities domain
**Environment:** Azure (portal + CLI), built while primary Proxmox lab hosts were offline due to a storm

## Objective

Build hands-on proof of the core workload identity concepts tested on SC-300:

1. Register an application and understand the app object vs. service principal split
2. Assign both delegated and application (app-only) Graph permissions and observe the admin consent difference between them
3. Stand up a system-assigned managed identity and use it to authenticate to Key Vault with **zero stored credentials**
4. Compare the RBAC-based Key Vault access model against the legacy access policy model

## Architecture

```
sc300-lab-app (App Registration)
  └── Application object (global template)
        └── Service Principal (local instance, tenant-scoped)
              ├── Delegated permission: User.Read
              └── Application permission: User.Read.All (admin-consented)

sc300-lab-automation (Automation Account)
  └── System-assigned Managed Identity
        └── RBAC role: Key Vault Secrets User (scoped to vault)
              └── Runbook: Test-ManagedIdentityKeyVault
                    └── Connect-AzAccount -Identity → Get-AzKeyVaultSecret
```

## Part 1 — App Registration & Service Principal

Registered an app in Entra ID (**App registrations → New registration**), named `sc300-lab-app`, single tenant.

**Key concept confirmed:** the app registration creates two distinct objects, not one:

| Object | Scope | Where it lives |
|---|---|---|
| Application object | Global (one per app, across all tenants that use it) | App registrations |
| Service principal | Local (one per tenant where the app is used) | Enterprise applications |

The service principal is created automatically in the home tenant the moment the app is registered — it's the object that permissions, role assignments, and Conditional Access policies actually attach to, not the application object itself.

## Part 2 — Delegated vs. Application Permissions

Under **API permissions → Add a permission → Microsoft Graph**:

- Added **delegated** permission `User.Read` — acts on behalf of a signed-in user; token scoped to that user's context
- Added **application** permission `User.Read.All` — acts as the app itself, no signed-in user; required clicking **Grant admin consent** before it would activate

**Key concept confirmed:** application permissions always require admin consent because they grant the app standing access independent of any user's own permissions — this is the mechanism the exam tests when asking why an app-only Graph call fails until an admin explicitly consents.

## Part 3 — Managed Identity → Key Vault (No Secret)

### Setup

```bash
RG="sc300-lab-rg"
LOCATION="eastus"
KV="sc300lab<random>"
AUTOMATION="sc300-lab-automation"

az group create --name $RG --location $LOCATION

az keyvault create \
  --name $KV \
  --resource-group $RG \
  --location $LOCATION \
  --enable-rbac-authorization true

az keyvault secret set \
  --vault-name $KV \
  --name "sc300-test-secret" \
  --value "if-you-can-read-this-managed-identity-worked"
```

```bash
az automation account create \
  --name $AUTOMATION \
  --resource-group $RG \
  --location $LOCATION

AUTOMATION_ID=$(az automation account show \
  --name $AUTOMATION --resource-group $RG --query id -o tsv)

az resource update --ids $AUTOMATION_ID --set identity.type=SystemAssigned

PRINCIPAL_ID=$(az automation account show \
  --name $AUTOMATION --resource-group $RG --query identity.principalId -o tsv)
```

```bash
KV_ID=$(az keyvault show --name $KV --resource-group $RG --query id -o tsv)

az role assignment create \
  --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

### Runbook (`Test-ManagedIdentityKeyVault`, PowerShell)

```powershell
Connect-AzAccount -Identity

Write-Output "Connected using managed identity. No secret was stored or passed."

try {
    $secret = Get-AzKeyVaultSecret -VaultName "<KV_NAME>" -Name "sc300-test-secret" -AsPlainText -ErrorAction Stop
    Write-Output "Retrieved secret value: $secret"
}
catch {
    Write-Output "ERROR retrieving secret:"
    Write-Output $_.Exception.Message
}

Write-Output "Context used:"
Get-AzContext | Format-List
```

### Result

```
Connected using managed identity. No secret was stored or passed.

Retrieved secret value: if-you-can-read-this-managed-identity-worked

Context used:
Account            : MSI@50342
Environment        : AzureCloud
Subscription       : f709aa07-f633-4716-9d8e-99b0926a83b6
Tenant             : 0652cfae-5900-4d18-9ac8-9f1ff468baf4
```

`Account: MSI@50342` confirms the authentication happened via the platform-managed identity token — no client secret, no certificate, no stored credential anywhere in the script or the Automation Account.

## Troubleshooting Log (kept intentionally — this is the part that shows real understanding, not just a working script)

| Issue | Root Cause | Fix |
|---|---|---|
| `MissingSubscriptionRegistration` on Key Vault create | `Microsoft.KeyVault` resource provider not registered on the subscription | `az provider register --namespace Microsoft.KeyVault` |
| `Forbidden` / `ForbiddenByRbac` on `secret set` | RBAC-enabled vault separates control-plane (management) access from data-plane (secrets) access; creating the vault only grants the former | Assigned self `Key Vault Secrets Officer` at the vault scope |
| `unrecognized arguments: --assign-identity [system]` | The `automation` CLI extension (preview) doesn't support the `--assign-identity` flag yet | Created the account without the flag, then patched identity on with `az resource update --set identity.type=SystemAssigned` |
| `Invalid URI: The hostname could not be parsed` | Left the `<YOUR_KV_NAME>` placeholder literally in the `-VaultName` parameter instead of substituting the real vault name | Replaced with actual `$KV` value |
| Secret came back blank with no visible error (before adding try/catch) | `Get-AzKeyVaultSecret` failed silently inside the Automation sandbox rather than throwing to stdout | Wrapped the call in `try { } catch { }` with `-ErrorAction Stop` to surface the real exception |

## Exam-Relevant Takeaways

- **App object vs. service principal**: permissions, roles, and Conditional Access always target the service principal, not the application object.
- **Delegated vs. application permissions**: application permissions require admin consent because they grant standing, user-independent access.
- **RBAC vault model vs. access policy model**: RBAC vaults use standard Azure role assignments (`Key Vault Secrets Officer`, `Key Vault Secrets User`, etc.) scoped like any other resource; access policy model vaults use a vault-level policy list instead. Control-plane access ≠ data-plane access under the RBAC model — a classic exam trap.
- **Least privilege in practice**: the human admin (me) was assigned `Secrets Officer` (read/write); the managed identity was scoped to `Secrets User` (read-only) — matching the actual need of each principal.
- **System-assigned managed identity**: eliminates the need to manage, rotate, or store any credential; the identity's lifecycle is tied to the resource it's attached to (deleting the Automation Account deletes the identity).

## Next Steps

- Repeat the exercise using a **user-assigned** managed identity to compare lifecycle differences (shared across multiple resources, independent lifecycle)
- Assign a Graph application permission directly to the managed identity's service principal (via Graph API, not RBAC) and call Graph instead of Key Vault
- Add Conditional Access for workload identities (Entra Workload ID Premium feature) scoped to this app's service principal, and test blocking access from an unapproved location
