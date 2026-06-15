# Handover — Windows students can't launch `observathon-sim.exe`

**Status:** ✅ FIXED. The durable `--onedir` rebuild is implemented in `tools/build_binaries.py` and shipped via CI tag `observathon-v7`; the Windows practice binary on the `observathon-v6-practice` release is now an onedir folder (no `%TEMP%` DLL self-extraction → error 998 can't occur). The workarounds in §3 remain valid for any older copy.
**Run path (Windows):** unzip the folder and run `bin\practice\observathon-sim\observathon-sim.exe` from inside it — keep the folder intact.

---

## 1. The symptom

A Windows student sees, on launch:

```
\observathon-sim.exe
[PYI-26284:ERROR] Failed to load Python DLL 'C:\Users\<user>\AppData\Local\Temp\_MEI143842\python312.dll'.
LoadLibrary: Invalid access to memory location.
```

This is Windows error **998 (ERROR_NOACCESS)**.

## 2. Root cause (confirmed)

The binaries are PyInstaller **`--onefile`**. On launch the exe self-extracts `python312.dll`
(and everything else) into `%TEMP%\_MEIxxxxxx\`, then `LoadLibrary("python312.dll")`. Error 998
means the DLL was unpacked but the OS refused to map it executable — i.e. **something interfered
with the temp file mid-load** (antivirus), or the temp/exe location isn't a normal runnable spot
(zip-preview temp, OneDrive-synced folder, locked-down `%TEMP%`).

**It is environmental to that machine, not a build defect.** Verified:
- CI builds Windows natively (`windows-latest`) — not a cross-compile artifact.
- UPX is already Linux-only in the build — not a UPX/AV interaction.
- Binaries ship as per-phase **zips**, not via git (`student/bin/` is gitignored) — no git line-ending corruption.
- Other students run the same exe fine — local to this machine/setup.

**Do NOT tell students to "run from source."** The `observathon_sim` package lives in `instructor/`,
which is deliberately withheld (hidden faults, datasets, answer keys). The exe is the only student path.

## 3. Send this to the affected student (most -> least likely to fix)

1. **Extract the zip fully and run from a plain local path.** Not from inside the Explorer
   "zip preview", and not from a OneDrive / synced Desktop folder. Copy to `C:\observathon\` and run there.
2. **Unblock + antivirus.** Right-click `observathon-sim.exe` -> Properties -> **Unblock**. Then add a
   Windows Defender exclusion for that folder **and** for `%TEMP%`, or briefly turn off real-time
   protection, and re-run.
3. **Clear stale temp / check disk.** Delete leftover `C:\Users\<user>\AppData\Local\Temp\_MEI*`
   folders; make sure the disk isn't full.
4. **Redirect the extraction dir** if `%TEMP%` is locked down (corporate AppLocker/DEP). In `cmd`:
   ```
   set TMP=C:\obs_tmp && set TEMP=C:\obs_tmp && bin\practice\observathon-sim.exe --help
   ```
5. **Re-download the zip** — a truncated extraction corrupts the DLL.

Steps 1 and 2 resolve the large majority of these.

## 4. Durable fix (do this if more than one Windows student is blocked)

Rebuild **Windows** binaries as **`--onedir`** instead of `--onefile`. Onedir keeps `python312.dll`
in a folder next to the exe and loads it directly — **no `%TEMP%` self-extraction, so the entire
error-998 failure mode disappears.** Zips are already the distribution format, so shipping a folder
costs nothing operationally. Trade-off: students run the exe from inside its folder rather than a
lone file (keep the folder intact; don't move the exe out).

### Code change (in `tools/build_binaries.py`)
- In `_pyi(...)`, make the mode OS-conditional: use `--onedir` on Windows (`os.name == "nt"`),
  keep `--onefile` on macOS/Linux.
- In `_build_all(...)`, the copy step at the end copies a single file per binary; for onedir the
  output is `dist/<name>/` (a folder). Update the Windows branch to copy the whole folder
  (`shutil.copytree`) into `student/bin/<phase>/<name>/` instead of one `.exe`.
- Re-verify `tools/verify_binaries.py` handles a folder path on Windows.

### Ship it
1. Land the build change.
2. Re-tag to trigger CI: `git tag observathon-v7 && git push origin observathon-v7`
   (workflow triggers on `observathon-*`).
3. Download the `observathon-binaries-windows-latest` artifact, zip the per-phase folders,
   redistribute the Windows zip(s). macOS/Linux zips are unchanged.

**Note:** the Windows artifact must be built by CI — it can't be produced from the macOS dev box
(PyInstaller can't cross-compile).

## 5. Verification checklist before declaring fixed
- [ ] Affected student confirms launch after step 1/2.
- [ ] (If onedir shipped) a Windows machine launches `observathon-sim.exe --help` from the
      extracted folder with no DLL error.
- [ ] `observathon-sim` runs a practice batch and writes `run_output.json`.
- [ ] `verify_binaries.py` passes on the new Windows build.
