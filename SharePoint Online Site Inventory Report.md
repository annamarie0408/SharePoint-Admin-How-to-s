# SharePoint Online Site Inventory Report

A PowerShell script that connects to SharePoint Online (via PnP PowerShell + Microsoft Entra ID app registration) and exports a CSV report of **every site collection** in your tenant, including:

- Site Title & URL
- Site Owners (Microsoft 365 Group owners *or* Site Collection Admins)
- Storage used (MB & GB)
- Total file count

Authentication uses a **certificate-based Entra ID app registration**, so the script runs unattended with no interactive login prompts — great for scheduled tasks / automation.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **PowerShell 7+** | Required by the PnP.PowerShell module (Windows PowerShell 5.1 is *not* supported by current PnP.PowerShell versions) |
| **PnP.PowerShell module** | Installed from the PowerShell Gallery |
| **SharePoint / Entra ID admin rights** | You need to be a Global Admin or SharePoint Admin to register the app and grant consent |
| **Local certificate store access** | `Register-PnPEntraIDApp` will generate and store a self-signed cert in your user's certificate store |

### Install PowerShell 7

If you don't already have it:

```powershell
winget install --id Microsoft.PowerShell --source winget
```

Then launch PowerShell 7 specifically by opening **`pwsh`** (not `powershell`) going forward.

### Install the PnP.PowerShell module

Run this inside PowerShell 7 (`pwsh`):

```powershell
Install-Module -Name PnP.PowerShell -Scope CurrentUser -Force
```

> If you hit an execution policy error, run:
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

Verify it installed correctly:

```powershell
Get-InstalledModule PnP.PowerShell
```

---

## Step 1: Register the Entra ID App

This creates an app registration in Microsoft Entra ID, generates a self-signed certificate, and grants it the SharePoint and Graph permissions the script needs.

```powershell
Register-PnPEntraIDApp `
    -ApplicationName "SPO-Reporting-App" `
    -Tenant "yourtenant.onmicrosoft.com" `
    -SharePointApplicationPermissions "Sites.FullControl.All" `
    -GraphApplicationPermissions "Group.Read.All" `
    -Store CurrentUser
```

**What this does:**
- Creates an app registration named `SPO-Reporting-App` in Entra ID
- Generates a self-signed certificate and stores the private key in your **CurrentUser** certificate store (`Cert:\CurrentUser\My`)
- Requests **application-level** (not delegated) permissions:
  - `Sites.FullControl.All` (SharePoint) — needed to read all site collections, libraries, and site collection admins
  - `Group.Read.All` (Microsoft Graph) — needed to read Microsoft 365 Group owners for group-connected (Team) sites

**After running this command:**

1. It will output a **Client ID (Application ID)** and a **certificate thumbprint** — copy both, you'll need them below.
2. Because these are *application* permissions, an admin must grant consent. Sign in to the [Azure Portal](https://portal.azure.com) → **Entra ID** → **App registrations** → find `SPO-Reporting-App` → **API permissions** → click **Grant admin consent for [tenant]**.
3. Confirm the certificate exists locally:

```powershell
Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*SPO-Reporting-App*" }
```

> ⚠️ **Security note:** Treat the Client ID + certificate thumbprint like credentials. Don't commit them to a public repo. See the [Keeping secrets out of source control](#-keeping-secrets-out-of-source-control) section below.

---

## ⚙️ Step 2: Configure the script

Update the variables at the top of `Export-SPOSiteReport.ps1` with your own values:

```powershell
$AdminUrl    = "https://yourtenant-admin.sharepoint.com"
$CSVPath     = "C:\Temp\SharepointData.csv"
$ClientID    = "<your-client-id-guid>"
$Thumbprint  = "<your-certificate-thumbprint>"
$Tenant      = "yourtenant.onmicrosoft.com"
```

| Variable | Where to get it |
|---|---|
| `$AdminUrl` | Your SharePoint Admin Center URL |
| `$CSVPath` | Wherever you want the output CSV saved |
| `$ClientID` | Output from `Register-PnPEntraIDApp` (also visible in Azure Portal → App registrations) |
| `$Thumbprint` | Output from `Register-PnPEntraIDApp` (also visible via `Get-ChildItem Cert:\CurrentUser\My`) |
| `$Tenant` | Your tenant's `.onmicrosoft.com` domain |

---

## Step 3: The script

Save this as `Export-SPOSiteReport.ps1`.

```powershell
# 1. Define Variables
$AdminUrl    = "https://yourtenant-admin.sharepoint.com"
$CSVPath     = "C:\Temp\SharepointData.csv"
$ClientID    = "<your-client-id-guid>"
$Thumbprint  = "<your-certificate-thumbprint>"
$Tenant      = "yourtenant.onmicrosoft.com"

# 2. Connect using certificate — no prompts, no browser
Write-Host "Connecting to SharePoint Admin Center..." -ForegroundColor Cyan
$AdminConn = Connect-PnPOnline -Url $AdminUrl -ClientId $ClientID -Thumbprint $Thumbprint -Tenant $Tenant -ReturnConnection

# 3. Retrieve all active site collections
Write-Host "Fetching site collections..." -ForegroundColor Cyan
$Sites = Get-PnPTenantSite -Connection $AdminConn | Where-Object {
    $_.Template -ne "RedirectSite#0" -and $_.Template -ne "SPSMSITEHOST#0"
}

$Report = @()
$Counter = 1

# 4. Loop through each site
foreach ($Site in $Sites) {
    Write-Host "[($Counter/$($Sites.Count))] Processing: $($Site.Url)" -ForegroundColor Yellow

    $TotalFileCount = 0
    $OwnersList = @()

    try {
        $SiteConn = Connect-PnPOnline -Url $Site.Url -ClientId $ClientID -Thumbprint $Thumbprint -Tenant $Tenant -ReturnConnection

        # File count
        $Libraries = Get-PnPList -Connection $SiteConn | Where-Object { $_.BaseTemplate -eq 101 -and $_.Hidden -eq $false }
        if ($Libraries) {
            $TotalFileCount = ($Libraries | Measure-Object -Property ItemCount -Sum).Sum
        }

        # --- Owners ---
        # Get the underlying M365 GroupId, if any (Team Sites are group-connected)
        $SiteInfo = Get-PnPSite -Connection $SiteConn -Includes GroupId
        if ($SiteInfo.GroupId -and $SiteInfo.GroupId.Guid -ne '00000000-0000-0000-0000-000000000000') {
            # Group-connected site (Team Site) — pull the M365 Group owners
            try {
                $GroupOwners = Get-PnPEntraIDGroupOwner -Identity $SiteInfo.GroupId.Guid -Connection $AdminConn
                $OwnersList += $GroupOwners | ForEach-Object { $_.Email -or $_.UserPrincipalName }
            }
            catch {
                Write-Host "  Could not retrieve group owners: $($_.Exception.Message)" -ForegroundColor DarkYellow
            }
        }
        else {
            # Classic/communication site — pull Site Collection Admins directly
            $SCAdmins = Get-PnPSiteCollectionAdmin -Connection $SiteConn | Where-Object { $_.PrincipalType -eq "User" }
            $OwnersList += $SCAdmins | ForEach-Object { $_.Email }
        }
    }
    catch {
        Write-Host "Skipping $($Site.Url) due to access restrictions: $($_.Exception.Message)" -ForegroundColor Red
        $TotalFileCount = "Access Denied / Error"
        $OwnersList = @("Access Denied / Error")
    }

    $SizeInMB = [Math]::Round($Site.StorageUsageCurrent, 2)
    $SizeInGB = [Math]::Round(($Site.StorageUsageCurrent / 1024), 2)
    $OwnersString = ($OwnersList | Where-Object { $_ } | Select-Object -Unique) -join "; "

    $Report += [PSCustomObject]@{
        "Site Title"       = $Site.Title
        "Site URL"         = $Site.Url
        "Owners"           = $OwnersString
        "Storage Used MB"  = $SizeInMB
        "Storage Used GB"  = $SizeInGB
        "Total File Count" = $TotalFileCount
    }

    $Counter++
}

# 5. Export results
$OutputFolder = Split-Path -Path $CSVPath -Parent
if (-not (Test-Path $OutputFolder)) {
    New-Item -Path $OutputFolder -ItemType Directory -Force | Out-Null
}

$Report | Export-Csv -Path $CSVPath -NoTypeInformation
Write-Host "Report successfully exported to: $CSVPath" -ForegroundColor Green

Disconnect-PnPOnline
```

---

## Step 4: Run it

1. Open **PowerShell 7** (`pwsh`), not Windows PowerShell.
2. Navigate to the folder containing the script.
3. Run:

```powershell
.\Export-SPOSiteReport.ps1
```

You'll see progress output for each site as it's processed, and a final CSV at the path you configured.

---

## Troubleshooting

| Issue | Likely cause / fix |
|---|---|
| `The term 'Register-PnPEntraIDApp' is not recognized` | You're on an older PnP.PowerShell version — run `Update-Module PnP.PowerShell` |
| `Access Denied` errors for specific sites | The app hasn't been granted admin consent, or the site has sharing/permission restrictions blocking app-only access |
| Script hangs or times out on large tenants | Very large tenants (thousands of sites) may need throttling/retry logic — consider adding `Start-Sleep` between sites or batching |
| `Group.Read.All` errors on owner lookups | Confirm admin consent was granted for the Graph permission, not just SharePoint |
| Certificate not found when connecting | Make sure you're running as the same Windows user account that ran `Register-PnPEntraIDApp` (cert is stored in that user's `CurrentUser` store) |

---

