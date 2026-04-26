 # Azure AD Connect — Hybrid Identity Migration (350 Users)

  ## Overview
  Migrated an on-premises Active Directory environment to hybrid identity using Azure AD Connect — enabling single sign-on across on-prem and Microsoft 365 / Azure services for 350 users.

  **Scope:** 350 users · Microsoft 365 E3 licenses · Exchange Online · Microsoft Intune
  **Environment:** Windows Server [2022] · Azure AD (Entra ID) · AAD Connect
  **Duration:** [2] weeks

  ---

  ## Problem

  Organization used cloud-only Microsoft 365 accounts disconnected from on-premises AD. Users managed two separate identities and passwords — one for domain login, one for M365.

  **Consequences:**
  - No single sign-on — users had separate passwords for domain and M365
  - No centralized identity management — AD and Azure AD were completely separate
  - Intune MDM could not enforce conditional access based on AD group membership
  - Manual onboarding: IT had to create accounts in two places for every new hire

  ---

  ## Solution Architecture

  ```
  On-Premises AD  ──────►  Azure AD Connect  ──────►  Azure AD (Entra ID)
  (Source of Truth)         (Sync Engine)              (Cloud Identity)
                                  │
                           Password Hash Sync
                           (PHS — selected over ADFS
                            for resilience, no federation
                            infrastructure required)
  ```

  **Sync method chosen:** Password Hash Synchronization (PHS)
  **Reason:** No ADFS infrastructure needed, resilient (cloud auth works if on-prem goes down), sufficient for organization's compliance requirements.

  ---

  ## Migration Process

  ### Phase 1 — Pre-Migration Audit

  - Verified all user UPNs matched verified domain in Azure AD
  - Checked for duplicate `proxyAddresses` and `mail` attributes that would cause sync conflicts
  - Ensured AD OU structure was clean before sync scope was defined

  ```powershell
  # Find duplicate proxyAddresses
  Get-ADUser -Filter * -Properties proxyAddresses |
      Group-Object { $_.proxyAddresses } |
      Where-Object { $_.Count -gt 1 }
  ```

  ### Phase 2 — AAD Connect Installation

  - Installed AAD Connect on dedicated server (not a DC)
  - Configured OU filtering — excluded service accounts and disabled objects OU from sync scope
  - Enabled Password Hash Sync + Password Writeback
  - Staged sync (dry run) — reviewed all objects in Synchronization Service Manager before enabling live sync

  ### Phase 3 — Identity Matching (Critical Step)

  Before enabling sync, had to match existing cloud-only M365 accounts to on-premises AD accounts to prevent duplicate identity creation.

  **Method used:** `ms-DS-ConsistencyGuid` (ImmutableID) hard-match

  ```powershell
  # Convert ObjectGUID to Base64 ImmutableID
  $user = Get-ADUser -Identity "username"
  $immutableID = [System.Convert]::ToBase64String($user.ObjectGUID.ToByteArray())

  # Set ImmutableID on existing Azure AD user (via MSOnline)
  Set-MsolUser -UserPrincipalName "user@domain.com" -ImmutableId $immutableID
  ```

  ### Phase 4 — The Immutable ID Incident

  **Problem encountered:** After enabling sync, multiple users failed to match and AAD Connect created duplicate cloud accounts. Investigation revealed these users had been recreated in on-premises AD at some point (e.g., after leaving and
  rejoining), so their `ObjectGUID` had changed — but their Azure AD accounts retained the old ImmutableID.

  **Root cause:** AAD Connect uses `ms-DS-ConsistencyGuid` (or `ObjectGUID` as fallback) for matching. When AD accounts were recreated, new GUIDs were generated — mismatch with existing Azure AD ImmutableIDs.

  **Fix:**
  1. Identified all affected users by comparing ImmutableIDs in Azure AD vs. current AD ObjectGUIDs
  2. Cleared ImmutableID on Azure AD side (`Set-MsolUser -ImmutableId $null`) — forcing soft-match mode
  3. Re-ran delta sync — AAD Connect matched on UPN (soft-match fallback)
  4. Verified no duplicate accounts remained in Azure AD

  ```powershell
  # Clear ImmutableID to allow soft-match re-sync
  Set-MsolUser -UserPrincipalName "user@domain.com" -ImmutableId ""
  # Then force delta sync
  Start-ADSyncSyncCycle -PolicyType Delta
  ```

  **Impact:** Zero data loss. Affected users experienced a brief sync delay (~2 hours) but no access interruption.

  ### Phase 5 — Post-Migration Validation

  - Verified all 350 users synced with no errors in AAD Connect Health dashboard
  - Confirmed SSO working: domain-joined machines auto-authenticated to M365 without second prompt
  - Conditional Access policies applied correctly based on synced group membership
  - Intune enrollment working for hybrid-joined devices

  ---

  ## Results

  | Before | After |
  |--------|-------|
  | Two separate identities per user | Single identity, one password |
  | Manual dual account creation for new hires | AD account creation auto-provisions M365 |
  | No conditional access based on AD groups | Full conditional access enforcement |
  | Separate Intune and AD management | Hybrid Azure AD join — unified device management |

  - 350 users migrated — zero accounts lost or duplicated (after immutable ID fix)
  - Onboarding time reduced: new hire provisioning from [X] minutes to [X] minutes
  - Security improved: conditional access now enforced at identity layer

  ---

  ## Tools Used
  - Azure AD Connect (AAD Connect / Microsoft Entra Connect)
  - Azure AD Connect Health (monitoring)
  - PowerShell: `MSOnline` module, `ADSync` module, `ActiveDirectory` module
  - Azure Portal → Entra ID → Users (sync validation)
  - Synchronization Service Manager

  ---

  ## Key Lesson
  Always audit ImmutableID alignment **before** enabling live sync — especially in environments where AD accounts have been recreated. A pre-migration script comparing Azure AD ImmutableIDs against current AD ObjectGUIDs prevents the
  hard-match failure scenario entirely.

  ---
