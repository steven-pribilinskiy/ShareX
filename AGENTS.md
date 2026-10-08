# Agent notes for this fork

This is a personal fork (`steven-pribilinskiy/ShareX`) whose build is the owner's daily-driver
ShareX. Upstream (`ShareX/ShareX`) has pull requests disabled, so nothing here is staged for a PR.

## Deploy without asking

After any change to this repo that affects the app, **build and deploy it to the live install
without asking for confirmation**: quit the running ShareX, swap the build, relaunch it `-silent`.
The only reason to hold off is a recording in progress (`Get-Process ffmpeg` returns something) —
then wait for it to finish rather than asking.

- Live install: `%LOCALAPPDATA%\Programs\ShareX-custom\` (the official install was removed;
  there is no rollback copy in `C:\Program Files\ShareX`).
- Autostart: scheduled task `ShareX delayed start`, pointing at the live install.
- Config: `C:\Users\steve\OneDrive\Documents\ShareX\` (shared; never let a `Portable` marker
  file survive next to the deployed `ShareX.exe`, or settings fork away from it).

## Build and deploy

```powershell
Set-Location C:\Users\steve\projects\forks\ShareX
dotnet restore --runtime win-x64 ShareX.sln
dotnet build ShareX\ShareX.csproj -c Release -p:Platform=x64 --self-contained true --no-restore
dotnet build ShareX.Setup\ShareX.Setup.csproj -c Release -p:Platform=x64 --no-restore

# Run Setup through the dotnet host: its output folder has a stray hostfxr.dll, so the
# .exe looks for .NET only beside itself and fails with 0x80008096.
dotnet .\ShareX.Setup\bin\Release\win-x64\ShareX.Setup.dll -silent -job CreatePortable

# CreatePortable wipes Output\ShareX-portable and the DownloadTools job no longer exists,
# so carry ffmpeg and exiftool over from the live install before deploying (/MIR would delete them).
$live = "$env:LOCALAPPDATA\Programs\ShareX-custom"; $out = "Output\ShareX-portable"
Copy-Item "$live\ffmpeg.exe", "$live\exiftool.exe" $out
Copy-Item "$live\exiftool_files" $out -Recurse -Force
Copy-Item "$live\Licenses\ExifTool_license.txt" "$out\Licenses\"

& "$env:USERPROFILE\.claude\skills\sharex\scripts\Deploy-CustomShareX.ps1" `
  -SourceDir (Resolve-Path $out) -OfficialExe "$live\ShareX.exe"
```

Build the `ShareX` project, not `ShareX.sln` — the solution build fails on RuntimeIdentifier
assets (`NETSDK1047`) for projects that are not part of the app.

Verify after deploying: the new process runs from the live install with `-silent`, the
`Portable` marker is gone, and the ShareX log shows
`Personal path: C:\Users\steve\OneDrive\Documents\ShareX`.

## Branches

- `fix/nvenc-max-dimension` — the deployed branch, rebased on upstream `develop`.
- Rebase from upstream with `git fetch origin && git rebase origin/develop`, then
  `git push --force-with-lease fork <branch>`.
