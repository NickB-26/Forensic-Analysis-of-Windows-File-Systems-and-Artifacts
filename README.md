# 🛡️ Forensic Analysis of Windows File Systems and Artifacts

A hands-on file system forensics project — imaging a Windows machine, loading it into Autopsy, and pulling apart the **MFT**, **Prefetch**, and **Shellbag** artifacts to reconstruct what happened on it.

---

## 🎯 Project Overview

I wanted to get more comfortable with the artifacts you actually reach for when triaging a Windows host. Imaging the drive, getting it into Autopsy, parsing the MFT to see file activity, looking at Prefetch to see what ran, and pulling Shellbags to see which folders the users opened.

This project covers five exercises built around three core artifacts, following the full workflow: **image → load → parse → filter → correlate** — with a fair bit of fighting with tools along the way.

---

## 📋 Pre-requisites

- Basic familiarity with Windows and the Command Prompt
- A Windows machine (physical or virtual) with administrative privileges
- Enough free space on a secondary drive for the disk image (50 GB+ depending on the source)

---

## 🖥️ Lab Environment

- **Operating System:** Windows 11 (64-bit) — physical desktop
- **Source drive:** `C:\` partition
- **Working directory:** `D:\Project 3`
- **Image destination (acquisition):** `E:\C partition image` — removable USB disk, used because I didn't have enough free space on the local drives at the time
- **Image location (analysis):** `D:\Project 3\C disk image\` — copied here after acquisition so I wasn't running Autopsy off USB

---

## 🧰 Tools Used

| Tool | Purpose |
|---|---|
| **FTK Imager 4.7.3.81** | Imaging tool — used to acquire a bit-for-bit copy of the `C:\` partition in .E01 format |
| **Autopsy 4.23.1** | Forensics platform — for loading the image and navigating the file system |
| **MFTECmd** | Eric Zimmerman's `$MFT` parser |
| **PECmd** | Eric Zimmerman's Prefetch parser |
| **SBECmd** | Eric Zimmerman's Shellbags parser |
| **Timeline Explorer** | Eric Zimmerman's CSV viewer — (handles the output from all three parsers better than MS Excel) |
| **Command Prompt (admin)** | For running the command-line tools |

---

## 🔬 Exercises

### Exercise 1 — Creating a Forensic Image using FTK Imager

**🎯 Objective:** Acquire a forensically sound image of the `C:\` partition in .E01 format.

**Steps:**

1. **Download FTK Imager** — From Exterro at [go.exterro.com](https://go.exterro.com). You have to fill in a contact form before it gives you the installer link.


<p align="center">
  <img src="https://github.com/user-attachments/assets/4e8bbe62-606c-4254-b0ee-bf446f7d7af3" alt="FTK Imager 4.7 download page" width="700"><br>
  <em>Figure 1: FTK Imager 4.7 download page</em>
</p>

2. **Install FTK Imager** — InstallShield wizard, default installation path (`C:\Program Files\AccessData`). The "Add Installation File path in Windows Defender Exclusion List" option is ticked by default — I left it on because Defender has a habit of interfering with imaging.

<p align="center">
  <img src="https://github.com/user-attachments/assets/70871bc9-5eb7-499c-bd33-cb244ad54f39" alt="FTK Imager installation path" width="700"><br>
  <em>Figure 2: Installing FTK Imager</em>
</p>



3. **Create the disk image** — **File → Create Disk Image**, picked **Physical Drive** as the source, then the `C:\` partition. For the output I went with **E01** (compressed, splittable, stores hashes inside the image).

<p align="center">
  <img src="https://github.com/user-attachments/assets/e34ba595-1460-4bf9-b11e-74bcf173bffa" alt="FTK Imager Select Image Destination dialog with C partition image filename" width="700"><br>
  <em>Figure 3: Starting aquisition</em>
</p>





4. **Verify the image** — When the image finishes, FTK Imager automatically re-reads the output, computes MD5 and SHA1, and compares them against the hashes it stored during acquisition. Got a **Match** on both with no bad blocks, which is what you want to see.

<p align="center">
  <img src="https://github.com/user-attachments/assets/0be1e22f-5217-4a0f-8e61-9539d9289c56" alt="FTK Imager Drive Image Verify Results showing matching MD5 and SHA1 hashes" width="700"><br>
  <em>Figure 4: Verification — MD5 and SHA1 both match, no bad blocks</em>
</p>


**Expected Output:** A verified E01 image with matching hashes.

> 💡 **Note:** I imaged to `E:\` (removable disk) first because I was running out of disk space but cleared enough disk space eventually therefore I copied the E01 across to `D:\Project 3\C disk image\` before starting analysis so I don't run it from USB. The hashes are stored inside the E01, so verification against the original acquisition hashes still worked after being copied.

---

### Exercise 2 — Analysing the File System with Autopsy

**🎯 Objective:** Load the image into Autopsy and have a look around the file system.

**Steps:**

1. **Download and install Autopsy** — Grabbed the 64-bit Windows build from [autopsy.com/download](https://www.autopsy.com/download) and installed it to the default path.

<p align="center">
  <img src="https://github.com/user-attachments/assets/2d027803-ee82-4ba4-9367-f152a21cf0e2" alt="Autopsy Setup Select Installation Folder dialog" width="700"><br>
  <em>Figure 5: Installing Autopsy 4.23.1</em>
</p>


2. **Create a new case** — Named it `Case 1`, base directory `D:\Project 3`, single-user.

<p align="center">
  <img src="https://github.com/user-attachments/assets/e60ab597-aa77-4651-8d54-48db995c4bdd" alt="Autopsy New Case Information dialog" width="700"><br>
  <em>Figure 6: New case setup</em>
</p>


3. **Select host** — Let Autopsy generate the host name from the data source.

<p align="center">
  <img src="https://github.com/user-attachments/assets/357c3bf4-0f40-4899-bccb-d9e308a2e5e4" alt="Autopsy Add Data Source Select Host dialog" width="700"><br>
  <em>Figure 7: Host selection</em>
</p>

4. **Configure ingest modules** — Left most of the defaults on. **Recent Activity**, **Hash Lookup**, **File Type Identification**, **Extension Mismatch Detector**, and **Encryption Detection** are the ones I particularly wanted. Skipped **Keyword Search** since I had no specific terms in mind.

<p align="center">
  <img src="https://github.com/user-attachments/assets/b929572f-2ef8-4f29-8ff1-740bb1069674" alt="Autopsy Configure Ingest dialog with default ingest modules selected" width="700"><br>
  <em>Figure 8: Ingest module selection</em>
</p>




5. **Wait for ingest** — Autopsy goes through every file on the image, builds its database, and runs the modules. On a full system image this takes a while.


<p align="center">
  <img src="https://github.com/user-attachments/assets/b303305f-2c69-4f56-9bd1-0b8dd0d26384" alt="Autopsy Select Data Source dialog with E01 image path" width="700"><br>
  <em>Figure 9: Ingest in progress</em>
</p>


6. **Navigate the file system** — Once ingest is done, the tree on the left shows the whole Windows directory structure. The folders I was most interested in were `Users\` (per-user data, registry hives, browser stuff), `Windows\System32\config\` (system registry), `Windows\Prefetch`, and `$Recycle.Bin`.

<p align="center">
  <img src="https://github.com/user-attachments/assets/24323c77-e414-4771-805f-ac464696c74d" alt="Autopsy Installed Programs view showing Windows 10 Pro and installed software" width="700"><br>
  <em>Figure 10: Installed Programs view </em>
</p>




7. **Check Installed Programs** — Under **C partition image  → Program Files**, Autopsy lists software pulled from the registry. Quick way to confirm the OS build and see what's on the box.

<p align="center">
  <img src="https://github.com/user-attachments/assets/d961bba5-01f7-4953-8364-b571a045152c" alt="Autopsy Installed Programs view showing Windows 10 Pro and installed software" width="700"><br>
  <em>Figure 11: Installed Programs view — confirms the OS build and installed software</em>
</p>




**Expected Output:** A navigable file system in Autopsy, with key directories identifiable and ingest artifacts populated.

---

### Exercise 3 — Extracting and Analysing the Master File Table

**🎯 Objective:** Pull `$MFT` out of the image, parse it with MFTECmd, and review the results in Timeline Explorer.

**Background:** The Master File Table is the index NTFS uses to track every file on the volume. Every file — including deleted ones whose records haven't yet been overwritten — has an MFT entry containing its name, parent path, size, timestamps, and attribute flags. Parsing it gives you a near-complete inventory of file activity on the disk.

**Steps:**

1. **Locate and extract `$MFT`** — In Autopsy I navigated to the root of the NTFS volume. `$MFT` sits alongside the other NTFS metadata files (`$Boot`, `$LogFile`, `$Bitmap` etc.). Right-click → **Extract File(s)** → saved to `D:\Project 3\MFT`.

<p align="center">
  <img src="https://github.com/user-attachments/assets/61b9ba44-045b-4f79-b11c-98c2f64988f1" alt="Autopsy file system view with $MFT highlighted and File(s) extracted confirmation" width="700"><br>
  <em>Figure 12: Extracting <code>$MFT</code> from the NTFS root</em>
</p>




2. **Download MFTECmd** — From Eric Zimmerman's tools page at [ericzimmerman.github.io](https://ericzimmerman.github.io). Timeline Explorer is on the same page so I grabbed that too while I was there.

<p align="center">
  <img src="https://github.com/user-attachments/assets/5809a6b6-418b-4089-967c-6b086a561f82" alt="Eric Zimmerman tools page showing MFTECmd in the forensic tools list" width="700"><br>
  <em>Figure 13: MFTECmd on Eric Zimmerman's tools page</em>
</p>


3. **Parse the `$MFT`** — Opened Command Prompt as admin, switched to the working directory, and ran MFTECmd with `--csv` pointing at an output folder:
   ```cmd
   cd /d "D:\Project 3\MFT"
   MFTECmd.exe -f "D:\Project 3\MFT\$MFT" --csv "D:\Project 3\MFT\parsed"
   ```

<p align="center">
  <img src="https://github.com/user-attachments/assets/16b5fb8f-fca7-411d-b80c-dfe57d54908f" alt="MFTECmd successful parse output showing 258,742 FILE records found in 25.6 seconds" width="700"><br>
  <em>Figure 14: 258,742 FILE records, 256,344 free, parsed in 25.6 seconds</em>
</p>

   The output is a timestamped CSV (`20260518152406_MFTECmd_$MFT_Output.csv`) with one row per MFT record. Each row has the four NTFS timestamps (`$STANDARD_INFORMATION` created/modified, `$FILE_NAME` created/modified), parent path, file size, and attribute flags.

4. **Open the parsed CSV in Timeline Explorer** — My first instinct was to open it in Excel. Bad idea — Excel choked on the row count and reformatted the timestamps to a US locale, which made sorting unreliable. Timeline Explorer is built for this kind of output and handled it without a fuss.

5. **Apply filters** — 625,000+ rows is too much to skim. I used the filter bar at the bottom of Timeline Explorer to narrow it down: `Extension IN .exe, .Exe, .EXE` AND `Parent Path IN` a couple of specific folders I was curious about.


<p align="center">
  <img src="https://github.com/user-attachments/assets/a42cb683-bc1a-4b5e-a53f-419c9ee73ccf" alt="Timeline Explorer showing filtered MFT output with executables under Python and Downloads paths" width="700"><br>
  <em>Figure 15: Filtered down to executables under the Python install paths and the user's Downloads folder</em>
</p>

6. **Review the timeline** — A few things jumped out from the filtered view:
   - Python 3.14 was installed on **2026-05-10** — `pip.exe`, `pip3.exe`, `pip3.14.exe`, and `idle.exe` all clustered around 00:18–00:28.
   - `WFA.exe` (the Windows File Analyzer I'd downloaded for Exercise 4) was created in `Downloads\WFA` on **2026-05-12 at 14:07** by user `nyco8`.
   - On **2026-05-07** there's activity under the `WsiAccount` profile creating `MicrosoftWindows.DesktopStickers` and `ActionsMcpHost.exe`.
   - Scrolling right, `WFA.exe` has the **Has Ads** flag set — that's the alternate data stream, almost certainly the `Zone.Identifier` Windows attaches to anything downloaded from the internet.

<p align="center">
  <img src="https://github.com/user-attachments/assets/6d2dbfe3-5fb4-4813-ac36-73a639aecac1" alt="Timeline Explorer view scrolled to show File Name, Extension, and timestamp columns" width="700"><br>
  <em>Figure 16: Same view scrolled across — file names, sizes, and all four MFT timestamps</em>
</p>

**Expected Output:** A parsed CSV of every MFT record on the volume, filterable in Timeline Explorer.

> 💡 **Tip:** The MFT has **four** timestamps per file — `$STANDARD_INFORMATION` (created/modified) and `$FILE_NAME` (created/modified). If they disagree, that's a sign of timestomping; an attacker tweaking `$STANDARD_INFORMATION` to mess with timelines while leaving `$FILE_NAME` (harder to reach) untouched. Timeline Explorer shows all four side by side, which makes the check straightforward.

---

### Exercise 4 — Analysing Windows Prefetch Files

**🎯 Objective:** Pull the Prefetch (`.pf`) files out of the image and figure out what's been run on the system, how often, and when.

**Background:** Prefetch is a Windows feature that speeds up application launch by recording metadata about every executable that runs — the path, a run count, the last eight run times, and what files the executable touched in its first 10 seconds. For an investigator that translates to "what ran on this machine, when, how many times" — one of the first questions you want answered in an incident.

**Steps:**

1. **Extract the Prefetch folder from Autopsy** — Navigated to `C partition image.E01 → Windows → Prefetch`. Right-click → **Extract File(s)**. The folder had 440 entries.

<p align="center">
  <img src="https://github.com/user-attachments/assets/09684c48-7e5c-44b5-947a-390b544ce903" alt="Autopsy Save dialog targeting the Prefetch folder" width="700"><br>
  <em>Figure 17: Extracting the Prefetch folder — 440 entries</em>
</p>



2. **Windows File Analyzer** — Initially, I wanted to use WFA to visualize the extracted files. I've downloaded it from [mitec.cz/wfa.html](https://www.mitec.cz/wfa.html) and pointed it at the extracted folder. It came back with **"Prefetch is disabled on this machine."**

<p align="center">
  <img src="https://github.com/user-attachments/assets/831b0c25-79bb-4313-bea9-5bd5ba54c8b3" alt="Windows File Analyzer warning dialog stating Prefetch is disabled on this machine" width="700"><br>
  <em>Figure 18: WFA insisting Prefetch is disabled — it isn't</em>
</p>


   That's misleading. Prefetch is clearly enabled on the source — I'd just extracted 440 `.pf` files from the image. The actual issue is that WFA hasn't been updated since around 2010, when Prefetch files were uncompressed. Windows 10/11 uses a compressed format (MAM/Xpress Huffman) and WFA doesn't recognise the header bytes, so it gives up and reports the folder as disabled rather than parsing it.

3. **Switch to PECmd** — Eric Zimmerman's PECmd is the modern replacement. It handles the compressed format and outputs CSVs that Timeline Explorer reads natively. Same download page as MFTECmd.

<p align="center">
  <img src="https://github.com/user-attachments/assets/a6b0a63b-1728-42b0-966c-6e1c542964ca" alt="Eric Zimmerman forensic tools page with PECmd row highlighted" width="700"><br>
  <em>Figure 19: PECmd on the tools page</em>
</p>



4. **Parse the Prefetch folder with PECmd** — `-d` for directory mode, `--csv` for the output:
   ```cmd
   cd /d "D:\Project 3\PECmd"
   PECmd.exe -d "D:\Project 3\Prefetch\Prefetch" --csv "D:\Project 3\Prefetch\parsed"
   ```

<p align="center">
  <img src="https://github.com/user-attachments/assets/a16ecf08-bada-463e-acb6-d0b118c41240" alt="PECmd command line output showing 414 of 432 files parsed successfully in 26.4 seconds with failed files listed" width="700"><br>
  <em>Figure 20: 414 of 432 files parsed in 26.4 seconds; 18 failures listed</em>
</p>

   PECmd got 414 of the 432 files. The 18 that failed were mostly Git temp files (`GIT-2.54.0-ARM64.TMP-*.pf`, `GIT-BASH.EXE-*.pf`), plus a couple of others — `CYGWIN-CONSOLE-HELPER.EXE`, `MOUSOCOREWORKER.EXE`, `RUNTIMEBROKER.EXE`, and two `SVCHOST` entries flagged as corrupt. The error message — `Invalid signature! Should be 'SCCA'` — means those files don't start with the SCCA/MAM magic bytes that a real Prefetch file should. Either they're corrupt, or they're not actually Prefetch files at all (more likely for the Git temp ones).

5. **Open the parsed CSV in Timeline Explorer** — Opened `20260518155409_PECmd_Output.csv` and sorted by **Run Count** descending.

<p align="center">
  <img src="https://github.com/user-attachments/assets/c44bd514-4365-4034-b527-332f283449a7" alt="Timeline Explorer showing parsed Prefetch CSV sorted by Run Count, with both MFT and Prefetch tabs visible" width="700"><br>
  <em>Figure 21: Prefetch sorted by Run Count descending — the MFT tab from Exercise 3 is still open alongside</em>
</p>


   The top of the list is all OS housekeeping:

   | Executable | Run Count | What it is |
   |---|---|---|
   | `CONHOST.EXE` | 266 | Console host — launched by every cmd/PowerShell window |
   | `TASKHOSTW.EXE` | 261 | Generic host for Windows scheduled tasks |
   | `MICROSOFTEDGEUPDATE.EXE` | 218 | Edge's auto-updater |
   | `DLLHOST.EXE` | 218 | COM Surrogate |
   | `MOUSOCOREWORKER.EXE` | 197 | Windows Update orchestrator |
   | `SPPSVC.EXE` | 184 | Software Protection Platform (licensing) |
   | `MSEDGE.EXE` | 158 | Edge browser |
   | `SVCHOST.EXE` (various) | 140 / 137 / 97 | Service hosts — each hash is a different service grouping |
   | `SMARTSCREEN.EXE` | 130 | Defender SmartScreen reputation checks |
   | `WMIPRVSE.EXE` | 122 | WMI Provider Host |

   This is what a clean Windows 11 machine looks like. The point of seeing this isn't the result itself, it's so you know what's expected — so on a real case, the random `runonce.exe` from `%APPDATA%` with a run count of 3 actually stands out as weird.

**Expected Output:** A parsed CSV of all valid Prefetch files, sortable by run count and last run time.

> 💡 **What to look for in a real Prefetch list:**
> - Anything in `%TEMP%` or `%APPDATA%\Local\Temp`
> - Executables with random-looking names
> - Single-execution `.exe` files in `Downloads`
> - LOLBins (`certutil.exe`, `mshta.exe`, `regsvr32.exe`, `bitsadmin.exe`) running from unusual paths

> ⚠️ **Tool limitation discovered during testing:** Windows File Analyzer cannot parse modern (Windows 10/11) Prefetch because it hasn't been updated since around 2010. PECmd handles the compressed MAM/Xpress Huffman format properly and produces CSVs Timeline Explorer reads natively.

---

### Exercise 5 — Investigating Recent Activity using Shellbags

**🎯 Objective:** Pull each user's registry hives out, parse the Shellbags with SBECmd, and see which directories each user has opened in Explorer.

**Background:** Shellbags are registry keys Windows uses to remember per-folder view preferences — window size, position, sort order, icon vs detail view. They live in two hives per user: `NTUSER.DAT` (where they were stored on XP/Vista) and `UsrClass.dat` (where they moved to from Windows 7 onwards). What makes them useful in forensics is they **persist after the folder is gone** — deleted folders, unmounted USB drives, and offline network shares all leave Shellbag traces behind even when nothing else does.

**Steps:**

1. **Identify which user profiles to extract from** — The `Users\` folder of the image had nine entries. Six were template/system profiles (`All Users`, `Default`, `Default User`, `Public`, plus a couple of Windows-internal ones) — no point extracting those. That left two real profiles: `nyco8` (the main interactive user) and `WsiAccount` (the secondary one I'd already noticed in the MFT timeline running OS components on 2026-05-07).

2. **Extract the hives** — For each profile I extracted:
   - `NTUSER.DAT` + `NTUSER.DAT.LOG1` + `NTUSER.DAT.LOG2` (from the profile root)
   - `UsrClass.dat` + `UsrClass.dat.LOG1` + `UsrClass.dat.LOG2` (from `AppData\Local\Microsoft\Windows`)

   The `.LOG1` and `.LOG2` files matter. They're the registry's transaction logs — SBECmd uses them to replay any pending changes that hadn't been committed to the main hive at the moment the image was taken. Skip the logs and you risk missing the most recent Shellbag updates.

<p align="center">
  <img src="https://github.com/user-attachments/assets/a20d7f7b-d7c1-491a-98fb-2d5b8aa5c200" alt="Autopsy Save dialog showing extracted NTUSER.DAT, UsrClass.dat and their log files for the nyco8 user" width="700"><br>
  <em>Figure 22: Extracting <code>NTUSER.DAT</code>, <code>UsrClass.dat</code>, and the transaction logs for <code>nyco8</code></em>
</p>



3. **Organise the hives by user** — SBECmd only takes a directory of hives (`-d`), not a single file, and when you point it at a folder containing multiple users' hives it produces one combined CSV with no column saying which hive each row came from. So to keep `nyco8`'s data separate from `WsiAccount`'s, I put each user's hives in their own subdirectory and ran SBECmd twice:

   ```
   D:\Project 3\Shellbags
   ├── hives
   │   ├── nyco8\          ← NTUSER.DAT + UsrClass.dat + logs
   │   └── WsiAccount\     ← NTUSER.DAT + UsrClass.dat + logs
   └── parsed
       ├── nyco8\          ← SBECmd output for nyco8
       └── WsiAccount\     ← SBECmd output for WsiAccount
   ```

4. **Parse the hives with SBECmd** — Downloaded SBECmd from the same Eric Zimmerman tools page. Then ran it once per user:
   ```cmd
   cd /d "D:\Project 3\SBECmd"
   SBECmd.exe -d "D:\Project 3\Shellbags\hives\nyco8" --csv "D:\Project 3\Shellbags\parsed\nyco8"
   SBECmd.exe -d "D:\Project 3\Shellbags\hives\WsiAccount" --csv "D:\Project 3\Shellbags\parsed\WsiAccount"
   ```

<p align="center">
  <img src="https://github.com/user-attachments/assets/e1f8843f-72e3-46a1-aa38-00235ec583ac" alt="SBECmd command line output showing 73 Shellbags found for nyco8" width="700"><br>
  <em>Figure 23: SBECmd output — 73 Shellbags for <code>nyco8</code></em>
</p>




<p align="center">
  <img src="https://github.com/user-attachments/assets/ae56a025-3142-4f9f-a54d-5f06529f5701" alt="SBECmd command line output showing 73 Shellbags found for nyco8, showing no of directories and file types" width="700"><br>
  <em>Figure 24: SBECmd output — 73 Shellbags for <code>nyco8</code></em>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/bb8dffe7-776b-410a-99dd-af759e2ba4b9" alt="SBECmd command line output showing 73 Shellbags found for nyco8, showing no of directories and file types" width="700"><br>
  <em>Figure 25: SBECmd output — 0 Shellbags for <code>WsiAccount</code></em>
</p>


   The counts alone are useful:
   - **`nyco8`** — 73 Shellbags in `UsrClass.dat`, 0 in `NTUSER.DAT`. That split is normal on Windows 10/11; the action is all in `UsrClass`.
   - **`WsiAccount`** — 0 in either hive.

   0 Shellbags for `WsiAccount` is itself a finding. It means that account has never opened a folder in Windows Explorer. Combined with what I saw in the MFT (the account only ever appeared launching OS components), I think it's safe to call `WsiAccount` a non-interactive system account rather than a person. In a real case, that's the kind of thing that lets you cross an account off the "investigate further" list.

5. **Look at `nyco8`'s Shellbags in Timeline Explorer** — Opened `D:\Project 3\Shellbags\parsed\nyco8\UsrClass.csv` and sorted by **LastInteracted** descending.

<p align="center">
  <img src="https://github.com/user-attachments/assets/65bf9593-b518-44ad-9585-7063ec0cf94e" alt="Timeline Explorer showing nyco8 Shellbags sorted by Last Interacted with paths to project folders" width="700"><br>
  <em>Figure 26: <code>nyco8</code>'s Shellbags, sorted by Last Interacted descending</em>
</p>


   The pattern is project work in OneDrive-synced folders. Some of what stood out:

   | Last Interacted | Absolute Path | What it tells me |
   |---|---|---|
   | 2026-05-13 00:18 | `Desktop\This PC` | Browsed This PC root |
   | 2026-05-13 00:15 | `Desktop\This PC\C:` | Browsed `C:\` |
   | 2026-05-13 00:13 | `Desktop\OneDrive\Desktop\GitHub project instructions` | Opened the GitHub instructions for these projects |
   | 2026-05-12 14:45 | `Desktop\Downloads\WFA` | Opened the WFA folder — matches what I saw in MFT |
   | 2026-05-11 14:41 | `Desktop\OneDrive\Immagini\Screenshots\Project 2 Volatility` | Screenshots from a previous project |
   | 2026-05-10 00:37 | `Desktop\OneDrive\Desktop\Volatility\volatility3-develop\...` | Multiple subfolders opened inside the Volatility 3 source tree (configuration, framework, volatility3) — all within a 6-minute window |
   | 2026-05-09 23:43 | `Desktop\ControlPanelHome\User Accounts\Credential Manager` | Opened Credential Manager — worth flagging in any real investigation |
   | 2026-05-07 14:14 | `Desktop\OneDrive\Desktop\Project 1 v.1\Log Parser 2.2 from x86` | Log Parser from an earlier project |

   A few things you can say about `nyco8` just from this:
   - The account is **interactive** — 73 folder views across multiple days
   - The user works on cybersecurity training projects involving Volatility and Log Parser — visible from folder names alone, no need for file contents
   - Everything is in OneDrive (`Desktop\OneDrive\Desktop\...`), so there's a cloud copy of all of it
   - **Credential Manager** was opened on 2026-05-09 at 23:43 — not necessarily suspicious here, but in any real investigation that's the kind of detail you'd note in the timeline (Credential Manager is where Windows stores saved passwords)
   - The activity is **bursty** rather than spread out — the Volatility cluster on 2026-05-10 is all within six minutes between 00:31 and 00:37

**Expected Output:** Per-user parsed CSVs showing every folder each profile has opened in Explorer, with first- and last-interaction timestamps.

> 💡 **What to look for in the output:**
> - Folders the user opened that no longer exist on disk (removed USB drives, deleted directories, offline shares)
> - Unusual late-night activity bursts
> - Access to sensitive paths (`Credential Manager`, password managers, sensitive shares)
> - Accounts with **zero** Shellbags — likely non-interactive system accounts rather than real users

> ⚠️ **SBECmd limitation discovered during testing:** SBECmd has no single-file mode — only `-d` (directory) and `-l` (live registry). When pointed at a folder containing multiple users' hives, it produces one combined CSV with no column identifying the source user. The workaround is to put each user's hives in their own subdirectory and run SBECmd once per user.

---

## 🎓 Lessons Learned

### Technical insights
- **Hash verification is what makes an image forensically usable.** FTK Imager computes MD5 and SHA1 during acquisition and re-computes them after, and a Match on both is the difference between an image you can defend in a report and a copy of the disk. Without it you've just got a file.
- **The MFT has four timestamps per file, not one.** `$STANDARD_INFORMATION` and `$FILE_NAME` each store their own created and modified times. If they disagree, that's a sign of timestomping — an attacker tweaking the `$STANDARD_INFORMATION` times to mess with timelines while leaving `$FILE_NAME` (which is harder to reach) untouched. Timeline Explorer shows all four side by side, which makes the check straightforward.
- **The parsing is the easy bit; filtering is where the work is.** MFTECmd handed me 625,882 rows. The interesting stuff was 10 of them. Knowing what to filter for — executables in user-writable paths, recent creations in `%TEMP%`, anything in Downloads — matters more than the parsing.
- **Prefetch shows you what *ran*, MFT shows you what *exists*.** Two different questions. A binary sitting on disk doesn't mean it was used; a Prefetch entry proves it actually launched, and the run count tells you how often. That separation is useful in triage.
- **Knowing what "normal" looks like is half the job.** The top-20 Prefetch entries on this clean machine are all Microsoft-signed OS components — `CONHOST`, `TASKHOSTW`, `SVCHOST`, `DLLHOST`, the usual suspects. Having that baseline in your head is what lets you spot a `runonce.exe` from `%APPDATA%` with a run count of 3 as something to look at.
- **Shellbags outlive the folder.** Even after the folder is deleted, the drive is unmounted, or the share is offline, the Shellbag entry stays in the registry. That's why they're so useful — they record that the user opened something, even when nothing else does.
- **Absence can be evidence.** `WsiAccount` having zero Shellbags isn't a parsing failure, it's a finding — the account never used Explorer. Combined with the MFT showing only OS components running under that account, I'd call it a system account rather than a person.
- **Modern Windows splits Shellbags between two hives.** Old Windows put them in `NTUSER.DAT`; Windows 7 onwards put the interesting ones in `UsrClass.dat`. Parse only one and you miss most of what's there. All 73 of `nyco8`'s Shellbags came from `UsrClass.dat`; `NTUSER.DAT` had nothing.

### SOC analyst mindset
- **Image first, always.** Even on a known-clean test machine. Building the habit of imaging before touching anything is the only way the same workflow stays valid when the target is a real incident host that you can't afford to change.
- **The MFT punches well above its weight.** A single ~500 MB file with metadata on every file that's ever existed on the volume, deleted ones included (as long as their record hasn't been overwritten). For triage you often get more out of parsing the MFT than carving the file system.

### Challenges & how I overcame them
- **Ran out of local disk space during imaging.** The `C:\` image was bigger than the free space I had on my internal drives, so I imaged straight to a USB disk (`E:`) and copied the E01 over to `D:\` afterwards. The hashes are stored inside the E01, so verification in Autopsy still works against the original acquisition hashes — but it's an extra step that wouldn't happen on a properly provisioned forensic workstation. **Lesson:** check disk space before you start, and aim for at least 2× the source size free on a dedicated drive.
- **MFTECmd failed with a file lock error on the first run.** First attempt threw `System.IO.IOException: The process cannot access the file because another process has locked a portion of the file` partway through writing the CSV. The `$MFT` itself was fine — it was the output CSV that was locked, because OneDrive was syncing the folder MFTECmd was writing to. Moved the output to a path on `D:\` that isn't OneDrive-synced, re-ran, and it finished cleanly in 25.6 seconds with all 258,742 records. **Lesson:** keep working directories off OneDrive entirely. Cloud sync and forensic tooling don't mix.
- **Excel is the wrong tool for MFT output.** First instinct was to open the CSV in Excel. It choked on the row count. Timeline Explorer is much better and handled the same file instantly. Wasted about ten minutes thinking the data was wrong when it was just the display.
- **WFA can't read modern Prefetch.** The brief says use WFA, but WFA told me "Prefetch is disabled on this machine" on a system that clearly has Prefetch enabled (440 `.pf` files extracted from the image). WFA hasn't been updated in a long time. I switched to PECmd — actively maintained, handles the modern format, and produces CSVs Timeline Explorer reads natively. PECmd parsed 414 of 432 files in 26 seconds. **Lesson:** Check if a tool works on the actual artifact format before committing time to it.
- **Some `.pf` files won't parse and that's normal.** PECmd flagged 18 files as `Invalid signature! Should be 'SCCA'`(the magic bytes at the start of a valid Prefetch file). Files that fail this are either corrupt entries (two `SVCHOST` files PECmd called "corrupt and did not parse completely") or files that share the `.pf` extension but aren't actually Prefetch. Worth flagging in a report rather than ignoring; corrupt Prefetch occasionally points at anti-forensics activity (an attacker wiping their tracks).
- **SBECmd doesn't have a single-file mode.** I assumed it would, like MFTECmd and PECmd do (`-f`). Got "Unrecognized command or argument" on my first few attempts. Turns out SBECmd only accepts `-d` (directory of hives) or `-l` (live registry) — there's no way to point it at a single file. Workaround was to give each user's hives their own subfolder and run `-d` per user. **Lesson:** even within the same toolset by the same author, the flags aren't always consistent. When something doesn't behave, run the tool with no arguments and read the help output.

### What I would do differently next time
- **Image to a properly prepared drive.** I imaged to an ordinary NTFS USB disk. In a real case you'd want a freshly wiped, write-blocked drive set aside specifically for evidence — partly for cleanliness, partly so there's no question about cross-contamination.
- **Build a saved filter set in Timeline Explorer.** Rather than starting from the full 625k-row view every time and narrowing down, I'd set up a few standard filters in advance — executables in user-writable paths, anything created in the last 24 hours, anything under `%TEMP%` or `%APPDATA%\Local\Temp` — and apply them on first open. A small bit of setup that pays off on every subsequent case.

---

## 🔗 Related Projects

- 🛡️ [Analyzing Windows Registry for Evidence of Malicious Activity](#) — companion project parsing registry hives for autoruns, USB history, and recently-opened files. Together these projects cover the major Windows host-based forensic artifacts end-to-end.

---

## 📚 References

- [Eric Zimmerman's Forensic Tools](https://ericzimmerman.github.io)
- [Autopsy User Documentation](https://sleuthkit.org/autopsy/docs/user-docs/)
- [FTK Imager User Guide](https://www.exterro.com/ftk-imager)
- [MFTECmd Documentation (GitHub)](https://github.com/EricZimmerman/MFTECmd)
- [PECmd Documentation (GitHub)](https://github.com/EricZimmerman/PECmd)
- [SBECmd / ShellBags Explorer (GitHub)](https://github.com/EricZimmerman/ShellBagsExplorer)
- [Windows File Analyzer (MiTeC)](https://www.mitec.cz/wfa.html)
