# LastActivityView
I've rebuild LastActivityView from Nirsoft from scratch and improved the overall application by adding a Signature checker, and some more artifacts aswell as regedit keys known as "Run once keys". 
 LastActivityView

A NirSoft LastActivityView (v1.37) style list in black, with Sigcheck-style signature verification added. An In Instance / Unsigned filter bar replaces the menu bar and toolbar; the original's list view, Quick Filter, status bar and dialogs (Properties, Find, Choose Columns, Advanced Options) stay, with Signature and Signer columns. Native Rust + egui, one ~4 MB exe with no runtime and no install. The original is closed-source freeware, so there is no `legacy-*` folder; nothing of it is reused.

## Window

- **Filter bar** (top): two toggle buttons that combine with each other and with Quick Filter.
  - **In Instance** (Ctrl+I): only actions since this PC booted. Rows with no time are hidden. Hover it to see the boot time.
  - **Unsigned** (Ctrl+U): only executables whose signature is not trusted: unsigned, tampered, invalid, expired or revoked.
- **Quick Filter bar** (Ctrl+Q, hidden by default): text plus the original's three drop-downs: all columns or visible columns only; one string, any or all words, any or all comma-separated strings; show or hide the matches.
- **List**
  - Columns: Action Time, Description, Filename, Full Path, **Signature**, **Signer**, File Extension.
  - Click a header to sort and click again to reverse. Drag a header edge to resize. Right-click a header for Choose Columns or Auto Size.
  - Double-click a row (or Alt+Enter) for Properties. It also shows the details that have no column: More Information (run count, renamed-file note, friendly name), User and Data Source.
  - Hovering a cut-off cell shows its full text. Hovering a signature explains it and names the signer.
- **Right-click a row**: Save Selected Items, Copy Selected Items, Copy Files List, HTML Report (All / Selected), Choose Columns, Auto Size Columns, Open Folder In Explorer, Properties.
  - View submenu: Find, Use Quick Filter, Show Grid Lines, Show Tooltips, Mark Odd/Even Rows.
  - Options submenu: Save File Encoding, Show Time In GMT, Show Milliseconds In Time, Add Header Line To CSV/Tab-Delimited File, Advanced Options, About.
  - Then Run as Administrator (when not elevated) and Refresh.
- **Status bar**: "N item(s), N Selected", what is loading or the last save, then the signature progress. Red warnings (hijacked signature checks, partial read) sit on the right; hover them for details.
- **Dates and times** use your Windows regional format, like the original.


Save formats are the original's seven: Text, Tab Delimited, Tabular, HTML Horizontal, HTML Vertical, XML and CSV. Only the visible columns are saved, in their on-screen order. HTML reports open in the default browser from `%TEMP%`. Settings are not saved between runs (nothing is written next to the exe on the machine being checked).

## Sources

| Description | Source | Time shown |
|---|---|---|
| Run .EXE file | `C:\Windows\Prefetch\*.pf` (MAM / XPRESS-Huffman decompressed via `ntdll`; SCCA v17 / 23 / 26 / 30 / 31) | Each of the last 8 run times (one row each); run count in More Information |
| Run .EXE file | BAM / DAM: `HKLM\SYSTEM\CurrentControlSet\Services\{bam,dam}\State\UserSettings\<SID>` | Last execution |
| Run .EXE file | MUICache: `UsrClass.dat\Local Settings\...\Shell\MuiCache` | None stored (friendly name + company shown) |
| Run .EXE file | Compatibility Assistant: `NTUSER.DAT\...\AppCompatFlags\Compatibility Assistant\{Store,Persisted}` | None stored |
| Select file in open/save dialog-box | `ComDlg32\OpenSavePidlMRU\<ext>` | Newest entry per extension = key last write; the rest have no time |
| Select folder in open/save dialog-box | `ComDlg32\LastVisitedPidlMRU` (with the program that used the dialog) | Newest entry = key last write |
| Open file or folder | `%APPDATA%\Microsoft\Windows\Recent\*.lnk` | Shortcut modified (last opened); created = first opened in More Information |
| Open file or folder | `Explorer\RecentDocs` (names only; survives clearing the Recent folder) | Newest overall and newest per extension = key last write |
| View Folder in Explorer | ShellBags: `BagMRU` under UsrClass.dat, `Shell` and `ShellNoRoam` | Newest child = parent key last write, else the folder's own key |

- Every profile in `ProfileList` is read (Properties shows whose). Registry sources need the user's hive to be loaded (signed-in user); a signed-out user gets only their Recent shortcuts, and the status bar says so.
- Prefetch paths are mapped to drive letters through the volume serial and given their real on-disk case. A path on a volume that is no longer attached keeps its `\VOLUME{..}` form. When the name Windows recorded differs from the only executable image in the file (renamed copy or hard link), More Information says so.
- Shell item lists (open/save, ShellBags, shortcuts) are length-checked before the shell sees them, then named with `SHGetPathFromIDListEx` (else the display path, e.g. `This PC\Phone\DCIM`).
- Not read (unlike the original): event logs (start / shutdown / sleep, crashes, BSOD, network, logon, installer, restore points, scheduled task runs) and the Uninstall keys.
- Times are kept to the millisecond in UTC, shown in local time (or GMT), sorted on UTC.

## Signatures

Same method as SrumParser / Sysinternals Sigcheck, for `.exe .com .dll .sys .scr .cpl` paths only: embedded Authenticode via `WinVerifyTrust`, otherwise Windows catalogs. The PE Authenticode digest is also recomputed independently, so a patched file or a copied signature shows **Tampered** even when the PE SIP registry entry is hijacked (that hijack is detected and shown as a red warning). Offline: cached CRLs only, nothing uploaded. Results are cached per path + size + modified time in `%LOCALAPPDATA%\LastActivityView\sigcache.tsv`. Network drives are skipped.

| Signature | Colour | Meaning |
|---|---|---|
| Signed | green | Valid embedded or catalog signature (hover: which, and the signer) |
| Unsigned / Tampered / Revoked | red | No signature · signature does not match the file (patched or SigThief copy) · certificate revoked |
| Invalid | orange | Signature does not verify (untrusted or broken chain) |
| Expired | amber | Certificate expired and no timestamp |
| Not Found | grey | The file is gone from that path (ran, then deleted or moved) |

The Unsigned button (Ctrl+U) lists the red, orange and amber rows; Quick Filter on `tampered` or `not found` narrows to one kind.

## Why it is small and fast

egui on OpenGL (glow), no embedded fonts (Segoe UI and the icon font are loaded from `%WINDIR%\Fonts`), Win32 bound with `raw-dylib`, static CRT, column-wise rows with only visible rows laid out, times formatted once per row, signatures resolved once per distinct path on background threads, `opt-level = "s"` + fat LTO + `panic = "abort"`.
