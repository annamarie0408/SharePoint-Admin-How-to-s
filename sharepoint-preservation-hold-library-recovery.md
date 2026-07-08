# Recovering Deleted Files from SharePoint's Preservation Hold Library

> **TL;DR:** If SharePoint won't let you restore a deleted file from the Recycle Bin because "a file with that name already exists" — but nothing appears in the library — the file is likely sitting in the hidden **Preservation Hold Library**. Copy it out from there instead of trying to restore it.

---

## The Symptom

- A file was deleted from a SharePoint document library (often by an external/guest user).
- You go to the **Recycle Bin** to restore it.
- SharePoint throws an error like *"A file with the same name already exists"* — even though the library appears empty of that file.
- You are a **Site Owner**, so permissions aren't the obvious problem.

## Why This Happens

If your SharePoint site has any **retention policy**, **retention label**, or **litigation hold** applied, deleted or edited files aren't just moved to the Recycle Bin — a copy is *also* silently preserved in a hidden system library called the **Preservation Hold Library (PHL)**.

That hidden copy still holds an internal reference to the file's original name and path. When you try to restore the file from the Recycle Bin, SharePoint sees this hidden reference and throws a naming conflict, even though the file is invisible in every list, view, and search you'd normally check.

This is expected, by-design behavior — Microsoft built it this way so retained content can't be accidentally (or maliciously) destroyed before its retention period expires.

## Who Can Access It

| Role | Can view PHL? |
|---|---|
| Regular member (Edit permissions) | ❌ No |
| Site Owner | ✅ Usually |
| Site Collection Administrator | ✅ Yes, full access |

If your account doesn't show the library at all, you may have "Owner" rights on the library itself but not full **Site Collection Administrator** rights — ask your Microsoft 365/SharePoint admin to grant that or pull the file for you.

---

## How to Fix It

### 1. Navigate directly to the Preservation Hold Library

It won't appear in your site navigation, so go there directly:

- **Via URL:**
  ```
  https://<yourtenant>.sharepoint.com/sites/<yoursite>/PreservationHoldLibrary/Forms/AllItems.aspx
  ```
- **Via UI:** Gear icon (⚙️) → **Site Contents** → **Preservation Hold Library**

> The library is only created the first time content on the site is edited or deleted under a retention policy — so it won't exist until that's happened at least once.

### 2. Find your file

Look for the file by name and timestamp. It's normal to see **multiple copies** with odd number/letter suffixes — each preserved version of the file (before edits, before deletion, etc.) can appear as a separate entry. Use the **Modified** date to identify the version you need.

### 3. Copy it out — don't try to restore it

Select the file and use **Copy to** → choose the destination (usually the original library/folder).

⚠️ **Do not try to restore/move it back to the exact same path first** — that's what triggers the naming conflict. Copying it to the original library as a "new" file sidesteps this entirely. You can rename it back or clean up afterward if needed.

### 4. If you can't see a Copy/Download option

This usually means you have Owner rights but not full **Site Collection Administrator** rights. Loop in whoever manages your Microsoft 365 admin center — they can either grant you elevated access temporarily or retrieve the file on your behalf.

---

## Quick Reference

| Step | Action |
|---|---|
| 1 | Go to `.../PreservationHoldLibrary/Forms/AllItems.aspx` |
| 2 | Find the file by name + timestamp |
| 3 | Select **Copy to** → original location |
| 4 | No access? → escalate to a Site Collection Admin |

## Good to Know

- You **cannot edit or delete** files in the PHL while they're under an active retention policy or hold — this is intentional and protects against data loss.
- Files typically leave the PHL automatically once their retention period expires, at which point they move to the second-stage Recycle Bin.
- This same mechanism applies to **OneDrive** accounts too — just swap in your OneDrive URL pattern (`.../personal/<user>/PreservationHoldLibrary`).

---

### Related Reading
- [Microsoft Learn – Restore deleted items from the site collection recycle bin](https://learn.microsoft.com/en-us/sharepoint/restore-deleted-items-from-recycle-bin)
- [Microsoft Learn – Retention policies overview](https://learn.microsoft.com/en-us/purview/retention)
