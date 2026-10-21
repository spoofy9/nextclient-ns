[Русский](README.md) · **English**

# CS 1.6 NextClient — build your own nonSteam client

A starter guide: how to take two archives (**base** + **patch**), assemble a working nonSteam
CS 1.6 NextClient, point it at **your own** game coordinator and wrap it into an installer.

The full source of our production installer (`engine.iss` — path validation, registry, skin,
splash, firewall, post-install integrity check, multi-copy support) is **not included**. What you
get here is the flow itself plus a working `.iss` skeleton containing the parts the client really
needs: registry pre-seed, a per-copy key hashed from the install path, and install-path checks.

> Who this is for: a server/community owner who wants to hand players their own client with
> their own server list instead of the official one.

---

## 1. What's in the pack

| Archive | What's inside | How often it changes |
|---|---|---|
| `base.zip` | invariant game content: `cstrike/`, `cstrike_hd/`, `valve/`, `platform/`, engine DLLs (`hw.dll`, `libcef.dll`, `icudt.dll`, `Mss32.dll`, …) | almost never |
| `2.5.4.zip` | files of this NextClient version: `cstrike.exe`, `next_lib.dll`, `next_engine_mini.dll`, `nitro_api2.dll`, `FileSystem_Proxy.dll`, `steam_api.dll`, `cstrike/cl_dlls/*`, `platform/defaults/*`, localizations | on every client update |

| File | Size | SHA256 |
|---|---|---|
| [`base.zip`](../../releases/latest) | 303 MiB | `dfa5bcdaca4e910ac66c30299068eb5b65aa6e4416531d8f5f180c7405108931` |
| [`2.5.4.zip`](2.5.4.zip) | 22 MiB | `eda4cdfbf8e92b91d98e8fe0e9890daa22b38ebb61ff4e5eb434e3d60231cdcb` |

> Links are relative: `2.5.4.zip` lives in the repository, `base.zip` is a
> [Releases](../../releases/latest) asset (303 MiB exceeds GitHub's 100 MiB per-file limit for
> repository content). Renaming or moving the repository needs no README changes.

Verify your download:

```powershell
Get-FileHash .\base.zip -Algorithm SHA256 | Format-List
```

**Rule:** one base fits any patch; the patch is applied **on top** of the base and overwrites
files with the same path. Base alone won't run, patch alone won't either.

### Software you need

- any unzip tool (Windows handles zip natively)
- **Inno Setup 6 Unicode** (with ISPP) — download: <https://jrsoftware.org/isdl.php>.
  Default compiler path: `C:\Program Files (x86)\Inno Setup 6\ISCC.exe`

---

## 2. How assembly works

```
base/            patch/             → final tree (staging)
├── cstrike/     ├── cstrike.exe    ├── cstrike.exe        ← from patch
├── valve/       ├── next_lib.dll   ├── next_lib.dll       ← from patch
├── hw.dll       ├── cstrike/       ├── hw.dll             ← from base
└── platform/    └── platform/      ├── cstrike/           ← base + patch files on top
                                    └── platform/
```

Layer order: **base → patch → (your custom content)**. For every path, the final tree keeps the
version from the **last** layer that contains that file.

---

## 3. Step 1. Merge the tree

Unpack `base.zip` and `2.5.4.zip` into separate folders, then merge them into `assembled`:

```powershell
# working folder layout
#   .\client\base\           — unpacked base.zip
#   .\client\patch\2.5.4\    — unpacked 2.5.4.zip
#   .\assembled\             — result (this is what gets packed)

Remove-Item -Recurse -Force .\assembled -ErrorAction SilentlyContinue
robocopy .\client\base        .\assembled /E /NFL /NDL /NJH /NJS
robocopy .\client\patch\2.5.4 .\assembled /E /NFL /NDL /NJH /NJS
```

The second `robocopy` overwrites base files with patch files — that's the overlay.

Clean the junk so players don't get it:

```powershell
Remove-Item -Recurse -Force .\assembled\crashes, .\assembled\htmlcache -ErrorAction SilentlyContinue
Get-ChildItem .\assembled -Include *.log,*.mdmp -Recurse | Remove-Item -Force
```

Sanity check — the root of `assembled` must contain:
`cstrike.exe`, `hw.dll`, `next_lib.dll`, `next_engine_mini.dll`, `libcef.dll`, `nitro_api2.dll`.

> At this point you can already run `assembled\cstrike.exe` and test the client before packing.

### If you already have your own CS 1.6 build

You don't need `base.zip` — the patch applies straight on top of your build:

1. Delete the **old launchers** from the game folder: `cs.exe`, `hl.exe` and the old `cstrike.exe` —
   otherwise players will keep starting the old engine instead of NextClient.
2. Unpack `2.5.4.zip` into the game folder, **overwriting** files.
3. Repoint every **shortcut** (desktop, Start menu, autostart) at `cstrike.exe` from the patch —
   that's the new launcher.

Note: the patch overwrites `cstrike/cl_dlls/*` (`client_mini.dll`, `GameUI.dll`), `platform/*`
and the client's root DLLs — your own edits to those files will be lost. Custom content
(maps, models, sprites, sounds, `*.wad`) is left alone.

Steps 2–5 (domain, keys, registry, packing) are the same.

---

## 4. Step 2. Your domain — `platform/defaults/backend.json`

This is the **only file** that holds your game coordinator address:

```
assembled\platform\defaults\backend.json
```

```json
{
  "payload" :
  {
    "addresses" : ["http://client.example.com/gc"]
  }
}
```

What matters:

- The protocol is written **explicitly, as `http://`** — that's what the stock config uses and what
  the client expects.
- The path is `/gc`: every API method is a single `POST` to that endpoint.
- `addresses` is an **array**: list several addresses separated by commas — you're not limited to
  a single backend. Put the main one first, the rest act as fallbacks/mirrors. An IP works instead
  of a domain.
- The file comes from the **patch layer**, so edit it **after** merging (inside `assembled`), or
  once inside your own patch folder — otherwise the next overlay restores the official address.
- Leave the default address (`client.cs16nextclient.com`) and the client talks to the official
  coordinator: running your own backend is optional.

Several addresses look like this:

```json
{
  "payload" :
  {
    "addresses" :
    [
      "http://client.example.com/gc",
      "http://gc2.example.com/gc",
      "http://1.2.3.4/gc"
    ]
  }
}
```

Language and launch parameters sit next to it, in `platform/defaults/user_game_config.ini`:

```ini
language english
launch_parameters "-noforcemaccel -noforcemparms -noforcemspd"
steam_mode 1
```

The distribution tag is the `partner.id` file in the game root (any string, e.g. `myserver-main`).
The client sends it to the backend in the `partner` header — that's how per-distribution stats
are counted.

---

## 5. Step 3. Encryption keys

Traffic between client and coordinator is encrypted with **AES-128-CBC**: the request body carries
`iv` and `data`, and the key never travels over the network — both sides know it in advance.

Two keys are used:

| Key | Purpose |
|---|---|
| `AES_CLIENT_KEY` | the main client key |
| `AES_LAUNCHER_KEY` | requests coming through the launcher (marked by the `kind` header) |

**Where to get them:** the client's author issues the keys — Telegram
**[@eto_bylo_letom](https://t.me/eto_bylo_letom)**. They are not in the client's open files and
are not part of this pack.

**Where they go:** the keys are needed **on your server side** (config / environment variables of
your `/gc` implementation) to decrypt requests and encrypt responses. Keys are never written into
the game files.

**API documentation:** <https://api.cs16nextclient.com/2.4.0> — request/response format, headers
(`branch`, `buildversion`, `partner`, `uid`, `launchgamecount`, `kind`, `self-hosted`, and behind
Cloudflare `cf-connecting-ip`, `cf-ipcountry`) and every method:

`branch_list`, `config`, `favorites`, `favorite_add`, `favorite_remove`, `gamemenu`,
`server_list`, `internet_refresh`, `session_end`, `user_activity`, `user_log`, `launcher_update`.

The minimum for a live client is `config`, `server_list`, `favorites`, `gamemenu`, `branch_list`;
the rest (`user_*`, `session_end`, `internet_refresh`) is telemetry — an empty success response is
fine. Game addresses in responses must be in `IP:PORT` format.

---

## 6. Step 4. Registry: what the installer must write

The client reads from the registry part of what it later sends to the backend as headers
(`branch`, `uid`, `launchgamecount`), and the engine reads its startup config from there. Everything
lives in `HKCU` (the installer runs without UAC, under `PrivilegesRequired=lowest`, so it writes
into the real player's profile).

### 6.1 Engine config pre-seed

`HKCU\Software\Valve\Half-Life\Settings` — so the game starts on the first try instead of dying
while initializing video:

| Value | Type | Data |
|---|---|---|
| `EngineDLL` | string | `hw.dll` |
| `ScreenBPP` | dword | `0x20` (32) |
| `EngineD3D` | dword | `0` |
| `CrashInitializingVideoMode` | dword | `0` |
| `hdmodels` | dword | `0` |
| `addons_folder` | dword | `1` |

### 6.2 Per-copy key with the folder hash

The important part. The build's key is tied to the **install path**:

```
HKCU\Software\Valve\Half-Life\nextclient\<hash>
     hash = first 8 characters of the MD5 of the full install path ({app})
```

| Path | Value | Written by | What it is |
|---|---|---|---|
| `nextclient\global` | `uid` (string) | installer | player GUID, **shared** per machine; sent in the `uid` header |
| `nextclient\global` | `LaunchGameCount` (dword) | the game itself | launch counter; sent in the `launchgamecount` header |
| `nextclient\<hash>` | `branch` (string) | installer | build branch (`main` or your own); sent in the `branch` header, used by `branch_list` |
| `nextclient\<hash>` | `AppId` (string) | installer | id of this copy |
| `nextclient\<hash>` | `PartnerId` (string) | the client itself | the `partner.id` it picked up |

Why the hash: a player may have **several copies** of the client in different folders (yours, the
official one, another community's). Hashing the path makes the key unique per copy — branches and
AppIds never clash, and reinstalling one copy doesn't break another. The installer uses the same
key to detect that **our** client is already installed in a folder.

`uid` is generated once and **reused**: read the existing one from `nextclient\global` first, and
only create a new GUID if it's empty. Otherwise every reinstall shows up as a new player in your
stats.

---

## 7. Step 5. Pack it into `setup.exe`

Put the file next to the `assembled` folder, save it as **UTF-8 with BOM** (otherwise non-Latin
messages break) and compile.

```iss
; minimal.iss — NextClient installer skeleton. UTF-8 with BOM!
#define ClientName  "CS 1.6 NextClient"
#define Version     "2.5.4"
#define Branch      "main"
#define SrcStage    "assembled"      ; the merged tree from step 1
#define RunExe      "cstrike.exe"

[Setup]
; AppId is derived from the install folder -> multiple copies never clash
AppId={code:GetAppId}
AppName={#ClientName}
AppVersion={#Version}
AppPublisher=My Server
DefaultDirName={sd}\games\CS 1.6 - NextClient
DefaultGroupName=Counter Strike 1.6 NextClient
DisableProgramGroupPage=yes
UsePreviousAppDir=no
UsePreviousLanguage=no
; no UAC: HKCU is written into the real player's profile
PrivilegesRequired=lowest
OutputDir=out
OutputBaseFilename=CS_1.6_NextClient_setup
Compression=lzma2
SolidCompression=no
WizardStyle=modern
UninstallDisplayIcon={app}\cstrike\game.ico

[Languages]
Name: "english"; MessagesFile: "compiler:Default.isl"
Name: "russian"; MessagesFile: "compiler:Languages\Russian.isl"

[CustomMessages]
english.NotValidInstallDir=The selected folder path contains invalid characters.%nUse only Latin letters, digits and . _ - ( )
russian.NotValidInstallDir=Путь содержит недопустимые символы.%nИспользуйте только латиницу, цифры и . _ - ( )
english.PreventInstallWrongDir=Cannot install to this location (Program Files / Windows / drive root). Choose another folder.
russian.PreventInstallWrongDir=Сюда устанавливать нельзя (Program Files / Windows / корень диска). Выберите другую папку.
english.DirHasCS=This folder already contains Counter-Strike. Choose another folder.
russian.DirHasCS=В этой папке уже есть Counter-Strike. Выберите другую папку.
english.DirNotEmpty=The selected folder is not empty. Choose an empty or a new folder.
russian.DirNotEmpty=Выбранная папка не пустая. Выберите пустую или новую папку.

[Files]
; one entry = one line: Inno Setup has no line-continuation character
Source: "{#SrcStage}\*"; DestDir: "{app}"; Excludes: "*.log,*.mdmp,crashes\*,htmlcache\*"; Flags: recursesubdirs createallsubdirs ignoreversion

[Icons]
Name: "{group}\{#ClientName}";       Filename: "{app}\{#RunExe}"; WorkingDir: "{app}"; IconFilename: "{app}\cstrike\game.ico"
Name: "{autodesktop}\{#ClientName}"; Filename: "{app}\{#RunExe}"; WorkingDir: "{app}"; IconFilename: "{app}\cstrike\game.ico"

[Registry]
; --- engine config pre-seed: the game starts on the first try ---
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "EngineDLL";                  ValueType: string; ValueData: "hw.dll"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "ScreenBPP";                  ValueType: dword;  ValueData: "$20"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "EngineD3D";                  ValueType: dword;  ValueData: "$0"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "CrashInitializingVideoMode"; ValueType: dword;  ValueData: "$0"; Flags: uninsdeletevalue
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "hdmodels";                   ValueType: dword;  ValueData: "$0"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "addons_folder";              ValueType: dword;  ValueData: "$1"
; --- per-copy key: nextclient\<hash>, hash = MD5(install path)[:8] ---
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\global";          ValueName: "uid";    ValueType: string; ValueData: "{code:GetUid}"
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\{code:GetHash}";  ValueName: "branch"; ValueType: string; ValueData: "{#Branch}"; Flags: uninsdeletekey
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\{code:GetHash}";  ValueName: "AppId";  ValueType: string; ValueData: "{code:GetAppId}"

[Run]
Filename: "{app}\{#RunExe}"; Description: "Launch the game"; Flags: nowait postinstall skipifsilent

[UninstallDelete]
; wiping the whole folder is safe — we only install into an empty one
Type: filesandordirs; Name: "{app}\*"
Type: dirifempty;     Name: "{app}"

[Code]
function CoCreateGuid(var g: TGUID): Integer;
  external 'CoCreateGuid@ole32.dll stdcall';
function StringFromGUID2(var g: TGUID; lpsz: String; cchMax: Integer): Integer;
  external 'StringFromGUID2@ole32.dll stdcall';

var
  gUid: String;

// ---- install-folder hash -> registry key and AppId --------------------------
function GetDirHash(): String;
var d: String;
begin
  // {app} is not known yet on an early call (AppId is evaluated at startup)
  try
    d := ExpandConstant('{app}');
  except
    d := '';
  end;
  if d <> '' then Result := Copy(GetMD5OfString(d), 1, 8) else Result := '';
end;

function GetHash(Param: String): String;
begin
  Result := GetDirHash();
end;

function GetAppId(Param: String): String;
var h: String;
begin
  h := GetDirHash();
  if h <> '' then Result := 'CS16NC-' + h else Result := 'CS16NC';
end;

// ---- uid: read the existing one, generate only when empty ------------------
function GetUid(Param: String): String;
var g: TGUID; s: String; n: Integer;
begin
  if gUid = '' then begin
    RegQueryStringValue(HKCU, 'Software\Valve\Half-Life\nextclient\global', 'uid', gUid);
    if (gUid = '') and (CoCreateGuid(g) = 0) then begin
      SetLength(s, 100);
      n := StringFromGUID2(g, s, 100);
      if n > 0 then begin
        SetLength(s, n - 1);                 // drop the trailing null
        gUid := Copy(s, 2, Length(s) - 2);   // strip the curly braces
      end;
    end;
  end;
  Result := gUid;
end;

// ---- install path checks ---------------------------------------------------
function IsAllowedPath(S: String): Boolean;
var i: Integer; c: Char;
begin
  // Latin letters/digits/space and . _ - ( ) : \ only — anything else breaks the engine
  Result := True;
  for i := 1 to Length(S) do begin
    c := S[i];
    if not ( ((c >= 'A') and (c <= 'Z')) or ((c >= 'a') and (c <= 'z'))
          or ((c >= '0') and (c <= '9'))
          or (c = ' ') or (c = '_') or (c = '-') or (c = '.')
          or (c = ':') or (c = '\') or (c = '(') or (c = ')') ) then begin
      Result := False;
      Exit;
    end;
  end;
end;

function IsForbiddenLocation(Dir: String): Boolean;
var u: String;
begin
  u := Uppercase(Dir);
  Result := (Pos('PROGRAM FILES', u) > 0)
         or (Pos(Uppercase(ExpandConstant('{win}')), u) = 1)
         or (Length(Dir) <= 3);   // drive root "C:\"
end;

function IsCSInDir(Dir: String): Boolean;
begin
  Result := DirExists(Dir + '\cstrike')
         or FileExists(Dir + '\hw.dll')
         or FileExists(Dir + '\cstrike.exe')
         or FileExists(Dir + '\cs.exe');
end;

function IsDirEffectivelyEmpty(Dir: String): Boolean;
var FR: TFindRec; nm: String;
begin
  Result := True;
  if not DirExists(Dir) then Exit;
  if FindFirst(Dir + '\*', FR) then begin
    try
      repeat
        if (FR.Name = '.') or (FR.Name = '..') then Continue;
        nm := Lowercase(FR.Name);
        if (nm = 'desktop.ini') or (nm = 'thumbs.db') then Continue;
        Result := False;
        Break;
      until not FindNext(FR);
    finally
      FindClose(FR);
    end;
  end;
end;

function NextButtonClick(CurPageID: Integer): Boolean;
var dir: String;
begin
  Result := True;
  if CurPageID = wpSelectDir then begin
    dir := WizardForm.DirEdit.Text;
    if not IsAllowedPath(dir) then begin
      MsgBox(ExpandConstant('{cm:NotValidInstallDir}'), mbError, MB_OK);
      Result := False; Exit;
    end;
    if IsForbiddenLocation(dir) then begin
      MsgBox(ExpandConstant('{cm:PreventInstallWrongDir}'), mbError, MB_OK);
      Result := False; Exit;
    end;
    if IsCSInDir(dir) then begin
      MsgBox(ExpandConstant('{cm:DirHasCS}'), mbError, MB_OK);
      Result := False; Exit;
    end;
    if not IsDirEffectivelyEmpty(dir) then begin
      MsgBox(ExpandConstant('{cm:DirNotEmpty}'), mbError, MB_OK);
      Result := False; Exit;
    end;
  end;
end;

function PrepareToInstall(var NeedsRestart: Boolean): String;
var dir: String;
begin
  // safety net + covers /SILENT, where NextButtonClick is never called
  Result := '';
  dir := ExpandConstant('{app}');
  if not IsAllowedPath(dir) then
    Result := ExpandConstant('{cm:NotValidInstallDir}')
  else if IsForbiddenLocation(dir) then
    Result := ExpandConstant('{cm:PreventInstallWrongDir}')
  else if IsCSInDir(dir) then
    Result := ExpandConstant('{cm:DirHasCS}')
  else if not IsDirEffectivelyEmpty(dir) then
    Result := ExpandConstant('{cm:DirNotEmpty}');
end;
```

Compile:

```powershell
& "C:\Program Files (x86)\Inno Setup 6\ISCC.exe" minimal.iss
# result: .\out\CS_1.6_NextClient_setup.exe
```

### Why these path checks

| Check | What happens without it |
|---|---|
| Latin-only path | the client won't start or crashes — the engine chokes on non-Latin and special characters in the path |
| not `Program Files` / `Windows` / drive root | under `lowest` there's no write access to the game folder → the game can't save configs and the updater fails |
| no other CS in the folder | files get mixed with a foreign build and can't be separated later |
| folder is empty | uninstall takes foreign files with it (our `[UninstallDelete]` wipes the whole folder) |

What the skeleton does **not** do (our production installer does, but a working client doesn't
need it):

- doesn't auto-pick a free folder (`CS 1.6 - NextClient 2`, ` 3`, … when the default is taken);
- doesn't add a firewall rule for `cstrike.exe` — under `PrivilegesRequired=lowest` that needs a
  separate elevated step, otherwise Windows asks the player itself on the first connect;
- doesn't set a safe screen resolution, splash or skin;
- doesn't verify integrity after installing (in case an antivirus ate some files);
- doesn't overlay the client language from the wizard choice — the language is baked into
  `user_game_config.ini` back in step 2.

---

## 8. Pre-release checklist

- [ ] the root contains `cstrike.exe`, `hw.dll`, `next_lib.dll`, `libcef.dll`, `nitro_api2.dll`
- [ ] `platform/defaults/backend.json` → your domain, `http://` protocol, `/gc` path
- [ ] `partner.id` — your own tag
- [ ] `platform/defaults/user_game_config.ini` — the language you want
- [ ] `crashes/`, `htmlcache/`, `*.log`, `*.mdmp` cleaned out
- [ ] the client runs from `assembled` and the server list arrives
- [ ] your backend speaks AES-128-CBC with the keys you were given
- [ ] the installer was tested against a non-Latin path and `Program Files` — it must refuse both
- [ ] after installing, `HKCU\...\Half-Life\nextclient\<hash>` holds `branch`

---

## 9. Common problems

| Symptom | Cause |
|---|---|
| Client won't start / crashes on launch | non-Latin or special characters in the install path |
| Game doesn't save settings | installed into `Program Files` — no write access under `lowest` |
| `libcef.dll` / `hw.dll` disappeared, game won't launch | antivirus ate files of an unsigned build — add the folder to exclusions |
| Game minimizes on the first connect | no firewall rule for `cstrike.exe` — allow it in the Windows prompt or add the rule manually |
| Empty server list | wrong address in `backend.json`, wrong protocol/path, or the backend failed to decrypt the request (wrong key) |
| Backend sees a new player after every reinstall | `uid` is overwritten instead of reused — read the existing one from `nextclient\global` |
| A second copy of the client breaks the first | the registry key is written without the folder hash |
| SmartScreen complains about `setup.exe` | the installer isn't signed — expected for a self-build |
| The official domain came back after a patch update | `backend.json` was overwritten by the patch layer — edit it after merging |

---

## 10. Links

- Game Coordinator API documentation: <https://api.cs16nextclient.com/2.4.0>
- Encryption keys, API questions: Telegram [@eto_bylo_letom](https://t.me/eto_bylo_letom)
- Official client site: <https://cs16nextclient.com>
- Inno Setup 6 (download): <https://jrsoftware.org/isdl.php>
