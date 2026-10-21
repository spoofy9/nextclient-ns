**Русский** · [English](README.en.md)

# CS 1.6 NextClient — своя nonSteam-сборка

Вводный гайд: как из двух архивов (**base** + **patch**) собрать рабочий nonSteam-клиент
CS 1.6 NextClient, направить его на **свой** game-coordinator и запаковать в установщик.

Полный исходник нашего боевого установщика (`engine.iss` — валидация пути, реестр, скин, сплеш,
firewall, пост-проверка целостности, мультикопийность) **в комплект не входит**. Здесь описан
сам флоу и приведён рабочий скелет `.iss` с тем, без чего клиент работает криво: предпосев
реестра, per-copy ключ с хешем папки и проверки пути установки.

> Для кого: владелец сервера / паблика, который хочет раздавать игрокам свой клиент
> со своим списком серверов, а не ставить официальный.

---

## 1. Содержимое раздачи

| Архив | Что внутри | Когда менять |
|---|---|---|
| `base.zip` | инвариантный контент игры: `cstrike/`, `cstrike_hd/`, `valve/`, `platform/`, движковые DLL (`hw.dll`, `libcef.dll`, `icudt.dll`, `Mss32.dll`, …) | почти никогда |
| `2.5.4.zip` | файлы версии NextClient: `cstrike.exe`, `next_lib.dll`, `next_engine_mini.dll`, `nitro_api2.dll`, `FileSystem_Proxy.dll`, `steam_api.dll`, `cstrike/cl_dlls/*`, `platform/defaults/*`, локализации | при каждом апдейте клиента |

| Файл | Размер | SHA256 |
|---|---|---|
| [`base.zip`](../../releases/latest) | 303 МиБ | `dfa5bcdaca4e910ac66c30299068eb5b65aa6e4416531d8f5f180c7405108931` |
| [`2.5.4.zip`](2.5.4.zip) | 22 МиБ | `eda4cdfbf8e92b91d98e8fe0e9890daa22b38ebb61ff4e5eb434e3d60231cdcb` |

> Ссылки относительные: `2.5.4.zip` — файл в репозитории, `base.zip` — ассет в
> [Releases](../../releases/latest) (303 МиБ не влезают в лимит GitHub на файл в репозитории — 100 МиБ).
> При переносе/переименовании репозитория править README не нужно.

Проверить, что архив скачался целым:

```powershell
Get-FileHash .\base.zip -Algorithm SHA256 | Format-List
```

**Правило:** base — одна под любой патч; патч накладывается **поверх** базы и перетирает
одноимённые файлы. Base в одиночку не запускается, патч в одиночку — тоже.

### Что нужно из софта

- архиватор (zip Windows распаковывает сам)
- **Inno Setup 6 Unicode** (с ISPP) — скачать: <https://jrsoftware.org/isdl.php>.
  Дефолтный путь к компилятору: `C:\Program Files (x86)\Inno Setup 6\ISCC.exe`

---

## 2. Схема сборки

```
base/            patch/             → итоговое дерево (staging)
├── cstrike/     ├── cstrike.exe    ├── cstrike.exe        ← из патча
├── valve/       ├── next_lib.dll   ├── next_lib.dll       ← из патча
├── hw.dll       ├── cstrike/       ├── hw.dll             ← из базы
└── platform/    └── platform/      ├── cstrike/           ← база + файлы патча сверху
                                    └── platform/
```

Порядок слоёв: **base → patch → (свой кастом)**. Для каждого пути в итоговом дереве остаётся
версия из **последнего** слоя, где этот файл есть.

---

## 3. Шаг 1. Склеить дерево

Распакуй `base.zip` и `2.5.4.zip` в отдельные папки, затем склей в `assembled`:

```powershell
# структура рабочей папки
#   .\client\base\           — распакованный base.zip
#   .\client\patch\2.5.4\    — распакованный 2.5.4.zip
#   .\assembled\             — результат (его и пакуем)

Remove-Item -Recurse -Force .\assembled -ErrorAction SilentlyContinue
robocopy .\client\base        .\assembled /E /NFL /NDL /NJH /NJS
robocopy .\client\patch\2.5.4 .\assembled /E /NFL /NDL /NJH /NJS
```

Второй `robocopy` затирает файлы базы файлами патча — это и есть overlay.

Почистить мусор, чтобы не тащить его игрокам:

```powershell
Remove-Item -Recurse -Force .\assembled\crashes, .\assembled\htmlcache -ErrorAction SilentlyContinue
Get-ChildItem .\assembled -Include *.log,*.mdmp -Recurse | Remove-Item -Force
```

Проверка, что склейка удалась — в корне `assembled` должны лежать:
`cstrike.exe`, `hw.dll`, `next_lib.dll`, `next_engine_mini.dll`, `libcef.dll`, `nitro_api2.dll`.

> На этом этапе папку `assembled` уже можно запустить (`cstrike.exe`) и проверить клиент
> до упаковки.

### Если у вас уже есть своя сборка CS 1.6

`base.zip` качать не нужно — патч накатывается прямо поверх вашей сборки:

1. Удалить из папки игры **старые лаунчеры**: `cs.exe`, `hl.exe` и старый `cstrike.exe` —
   иначе игроки будут запускать старый движок, а не NextClient.
2. Распаковать `2.5.4.zip` в папку игры **с заменой** файлов.
3. Перевести **ярлыки** (рабочий стол, меню «Пуск», автозапуск) на `cstrike.exe` из патча —
   это и есть новый лаунчер.

Учти: патч перетирает `cstrike/cl_dlls/*` (`client_mini.dll`, `GameUI.dll`), `platform/*`
и корневые DLL клиента — свои правки в этих файлах потеряются. Кастомный контент
(карты, модели, спрайты, звуки, `*.wad`) патч не трогает.

Шаги 2–5 (домен, ключи, реестр, упаковка) — те же.

---

## 4. Шаг 2. Свой домен — `platform/defaults/backend.json`

Это **единственный файл**, в котором задаётся адрес твоего game-coordinator:

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

Что важно:

- Протокол указывается **явно и в формате `http://`** — так в оригинальном конфиге, так ждёт клиент.
- Путь — `/gc`: все методы API идут одним `POST` на этот эндпоинт.
- `addresses` — **массив**: адреса перечисляются через запятую, одним бэкендом ограничиваться
  не обязательно. Основной ставь первым, остальные — резерв/зеркала. Вместо домена можно
  указать IP.
- Файл приезжает из **слоя патча**, поэтому правь его **после** склейки (в `assembled`)
  либо один раз в своей папке патча — иначе следующий overlay вернёт официальный адрес.
- Оставишь адрес по умолчанию (`client.cs16nextclient.com`) — клиент будет работать
  с официальным координатором: свой бэкенд поднимать не обязательно.

Несколько адресов — так:

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

Язык и параметры запуска — рядом, в `platform/defaults/user_game_config.ini`:

```ini
language russian
launch_parameters "-noforcemaccel -noforcemparms -noforcemspd"
steam_mode 1
```

Метка источника раздачи — файл `partner.id` в корне игры (произвольная строка, например
`myserver-main`). Клиент отдаёт её бэкенду заголовком `partner` — по ней считается статистика
по раздачам.

---

## 5. Шаг 3. Ключи шифрования

Трафик между клиентом и координатором шифруется **AES-128-CBC**: в теле запроса приходят
`iv` и `data`, ключ по сети не передаётся — он заранее известен обеим сторонам.

Используются два ключа:

| Ключ | Для чего |
|---|---|
| `AES_CLIENT_KEY` | основной ключ клиента |
| `AES_LAUNCHER_KEY` | запросы, идущие через лаунчер (признак — заголовок `kind`) |

**Где взять:** ключи выдаёт автор клиента — Telegram **[@eto_bylo_letom](https://t.me/eto_bylo_letom)**.
В открытых файлах клиента их нет и в этот пак они не входят.

**Куда положить:** ключи нужны **на твоей серверной стороне** (в конфиге/переменных окружения
твоей реализации `/gc`), чтобы расшифровывать запросы и шифровать ответы. В файлы игры ключи
не прописываются.

**Документация API:** <https://api.cs16nextclient.com/2.4.0> — формат запроса/ответа, заголовки
(`branch`, `buildversion`, `partner`, `uid`, `launchgamecount`, `kind`, `self-hosted`,
за Cloudflare — `cf-connecting-ip`, `cf-ipcountry`) и все методы:

`branch_list`, `config`, `favorites`, `favorite_add`, `favorite_remove`, `gamemenu`,
`server_list`, `internet_refresh`, `session_end`, `user_activity`, `user_log`, `launcher_update`.

Минимум для живого клиента — `config`, `server_list`, `favorites`, `gamemenu`, `branch_list`;
остальное (`user_*`, `session_end`, `internet_refresh`) — телеметрия, можно отвечать пустым успехом.
Игровые адреса в ответах — строго в формате `IP:PORT`.

---

## 6. Шаг 4. Реестр: что должен записать установщик

Клиент берёт из реестра часть того, что потом уходит на бэкенд заголовками (`branch`, `uid`,
`launchgamecount`), а движок — свой стартовый конфиг. Всё лежит в `HKCU` (установщик работает
без UAC, под `PrivilegesRequired=lowest`, поэтому пишем в профиль реального игрока).

### 6.1 Предпосев конфига движка

`HKCU\Software\Valve\Half-Life\Settings` — чтобы игра завелась с первого запуска, а не
свалилась на инициализации видео:

| Значение | Тип | Данные |
|---|---|---|
| `EngineDLL` | string | `hw.dll` |
| `ScreenBPP` | dword | `0x20` (32) |
| `EngineD3D` | dword | `0` |
| `CrashInitializingVideoMode` | dword | `0` |
| `hdmodels` | dword | `0` |
| `addons_folder` | dword | `1` |

### 6.2 Per-copy ключ с хешем папки

Главное место. Ключ сборки привязан к **пути установки**:

```
HKCU\Software\Valve\Half-Life\nextclient\<hash>
     hash = первые 8 символов MD5 от полного пути установки ({app})
```

| Путь | Значение | Кто пишет | Что это |
|---|---|---|---|
| `nextclient\global` | `uid` (string) | установщик | GUID игрока, **общий** на машину; уходит заголовком `uid` |
| `nextclient\global` | `LaunchGameCount` (dword) | сама игра | счётчик запусков; уходит заголовком `launchgamecount` |
| `nextclient\<hash>` | `branch` (string) | установщик | ветка сборки (`main`, своя); уходит заголовком `branch`, используется в `branch_list` |
| `nextclient\<hash>` | `AppId` (string) | установщик | id этой копии |
| `nextclient\<hash>` | `PartnerId` (string) | сам клиент | подтянутый `partner.id` |

Зачем хеш: у игрока может стоять **несколько копий** клиента в разных папках (твоя, официальная,
другого паблика). Хеш пути делает ключ уникальным на копию — ветки и AppId не конфликтуют,
переустановка одной копии не ломает другую. По этому же ключу установщик понимает, что в папке
уже стоит **наш** клиент.

`uid` генерируется один раз и **переиспользуется**: сначала читаем существующий из
`nextclient\global`, и только если пусто — создаём новый GUID. Иначе каждая переустановка
будет новым игроком для статистики.

---

## 7. Шаг 5. Упаковка в `setup.exe`

Положи файл рядом с папкой `assembled`, сохрани **в UTF-8 with BOM** (иначе кириллица
в сообщениях сломается) и скомпилируй.

```iss
; minimal.iss — скелет установщика NextClient. UTF-8 with BOM!
#define ClientName  "CS 1.6 NextClient"
#define Version     "2.5.4"
#define Branch      "main"
#define SrcStage    "assembled"      ; готовое дерево из шага 1
#define RunExe      "cstrike.exe"

[Setup]
; AppId детерминированно считается от папки установки -> несколько копий не конфликтуют
AppId={code:GetAppId}
AppName={#ClientName}
AppVersion={#Version}
AppPublisher=My Server
DefaultDirName={sd}\games\CS 1.6 - NextClient
DefaultGroupName=Counter Strike 1.6 NextClient
DisableProgramGroupPage=yes
UsePreviousAppDir=no
UsePreviousLanguage=no
; без UAC: HKCU пишется в профиль реального игрока
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
; одна запись = одна строка: переносов строк Inno Setup не поддерживает
Source: "{#SrcStage}\*"; DestDir: "{app}"; Excludes: "*.log,*.mdmp,crashes\*,htmlcache\*"; Flags: recursesubdirs createallsubdirs ignoreversion

[Icons]
Name: "{group}\{#ClientName}";       Filename: "{app}\{#RunExe}"; WorkingDir: "{app}"; IconFilename: "{app}\cstrike\game.ico"
Name: "{autodesktop}\{#ClientName}"; Filename: "{app}\{#RunExe}"; WorkingDir: "{app}"; IconFilename: "{app}\cstrike\game.ico"

[Registry]
; --- предпосев конфига движка: игра заводится с первого раза ---
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "EngineDLL";                  ValueType: string; ValueData: "hw.dll"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "ScreenBPP";                  ValueType: dword;  ValueData: "$20"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "EngineD3D";                  ValueType: dword;  ValueData: "$0"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "CrashInitializingVideoMode"; ValueType: dword;  ValueData: "$0"; Flags: uninsdeletevalue
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "hdmodels";                   ValueType: dword;  ValueData: "$0"
Root: HKCU; Subkey: "Software\Valve\Half-Life\Settings"; ValueName: "addons_folder";              ValueType: dword;  ValueData: "$1"
; --- per-copy ключ: nextclient\<hash>, hash = MD5(путь установки)[:8] ---
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\global";          ValueName: "uid";    ValueType: string; ValueData: "{code:GetUid}"
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\{code:GetHash}";  ValueName: "branch"; ValueType: string; ValueData: "{#Branch}"; Flags: uninsdeletekey
Root: HKCU; Subkey: "Software\Valve\Half-Life\nextclient\{code:GetHash}";  ValueName: "AppId";  ValueType: string; ValueData: "{code:GetAppId}"

[Run]
Filename: "{app}\{#RunExe}"; Description: "Запустить игру"; Flags: nowait postinstall skipifsilent

[UninstallDelete]
; полный снос — безопасно, т.к. ставим только в пустую папку
Type: filesandordirs; Name: "{app}\*"
Type: dirifempty;     Name: "{app}"

[Code]
function CoCreateGuid(var g: TGUID): Integer;
  external 'CoCreateGuid@ole32.dll stdcall';
function StringFromGUID2(var g: TGUID; lpsz: String; cchMax: Integer): Integer;
  external 'StringFromGUID2@ole32.dll stdcall';

var
  gUid: String;

// ---- хеш папки установки -> ключ реестра и AppId ---------------------------
function GetDirHash(): String;
var d: String;
begin
  // {app} ещё не известен при раннем вызове (AppId считается на старте)
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

// ---- uid: читаем существующий, генерируем только если пусто ----------------
function GetUid(Param: String): String;
var g: TGUID; s: String; n: Integer;
begin
  if gUid = '' then begin
    RegQueryStringValue(HKCU, 'Software\Valve\Half-Life\nextclient\global', 'uid', gUid);
    if (gUid = '') and (CoCreateGuid(g) = 0) then begin
      SetLength(s, 100);
      n := StringFromGUID2(g, s, 100);
      if n > 0 then begin
        SetLength(s, n - 1);                 // отбросить завершающий null
        gUid := Copy(s, 2, Length(s) - 2);   // снять фигурные скобки
      end;
    end;
  end;
  Result := gUid;
end;

// ---- проверки пути установки -----------------------------------------------
function IsAllowedPath(S: String): Boolean;
var i: Integer; c: Char;
begin
  // только латиница/цифры/пробел и . _ - ( ) : \  — кириллица и спецсимволы ломают движок
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
         or (Length(Dir) <= 3);   // корень диска "C:\"
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
  // страховка + покрытие /SILENT, где NextButtonClick не вызывается
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

Компиляция:

```powershell
& "C:\Program Files (x86)\Inno Setup 6\ISCC.exe" minimal.iss
# результат: .\out\CS_1.6_NextClient_setup.exe
```

### Почему именно эти проверки пути

| Проверка | Если не сделать |
|---|---|
| только латиница в пути | клиент не стартует или крашится — движок не переваривает кириллицу и спецсимволы в пути |
| не `Program Files` / `Windows` / корень диска | под `lowest` нет прав на запись в игровую папку → игра не сохраняет конфиги, апдейтер не работает |
| в папке нет другого CS | файлы перемешаются с чужой сборкой, потом не разделить |
| папка пустая | снос при удалении зацепит чужое (у нас `[UninstallDelete]` чистит папку целиком) |

Что скелет **не делает** (в боевом установщике это есть, но для рабочего клиента не обязательно):

- не подбирает свободную папку автоматом (`CS 1.6 - NextClient 2`, ` 3`, … при занятой);
- не добавляет правило фаервола для `cstrike.exe` — под `PrivilegesRequired=lowest` для этого
  нужен отдельный шаг с повышением прав, иначе Windows спросит игрока сам при первом коннекте;
- не ставит безопасное разрешение экрана, сплеш и скин;
- не проверяет целостность после установки (вдруг антивирус выел файлы);
- не оверлеит язык клиента по выбору в мастере — язык зашит в `user_game_config.ini` на шаге 2.

---

## 8. Чеклист перед раздачей

- [ ] в корне есть `cstrike.exe`, `hw.dll`, `next_lib.dll`, `libcef.dll`, `nitro_api2.dll`
- [ ] `platform/defaults/backend.json` → свой домен, протокол `http://`, путь `/gc`
- [ ] `partner.id` — своя метка
- [ ] `platform/defaults/user_game_config.ini` — нужный язык
- [ ] `crashes/`, `htmlcache/`, `*.log`, `*.mdmp` вычищены
- [ ] клиент запущен из `assembled` и список серверов приходит
- [ ] бэкенд умеет AES-128-CBC с полученными ключами
- [ ] установщик проверен на пути с кириллицей и на `Program Files` — должен не пустить
- [ ] после установки в `HKCU\...\Half-Life\nextclient\<hash>` лежит `branch`

---

## 9. Частые проблемы

| Симптом | Причина |
|---|---|
| Клиент не стартует / крашится на старте | кириллица или спецсимволы в пути установки |
| Игра не сохраняет настройки | поставили в `Program Files` — под `lowest` там нет прав на запись |
| Пропал `libcef.dll` / `hw.dll`, игра не запускается | антивирус съел файлы неподписанного билда — добавить папку в исключения |
| Игра сворачивается при первом коннекте | нет правила фаервола для `cstrike.exe` — разрешить в запросе Windows или добавить правило руками |
| Пустой список серверов | неверный адрес в `backend.json`, не тот протокол/путь, или бэкенд не расшифровал запрос (не тот ключ) |
| Бэкенд видит нового игрока после каждой переустановки | `uid` перезаписывается вместо переиспользования — читай существующий из `nextclient\global` |
| Вторая копия клиента ломает первую | ключ реестра пишется без хеша папки |
| SmartScreen ругается на `setup.exe` | установщик не подписан — нормально для самосборки |
| После апдейта патча вернулся официальный домен | `backend.json` перетёрт слоем патча — правь после склейки |

---

## 10. Ссылки

- Документация Game Coordinator API: <https://api.cs16nextclient.com/2.4.0>
- Ключи шифрования, вопросы по API: Telegram [@eto_bylo_letom](https://t.me/eto_bylo_letom)
- Официальный сайт клиента: <https://cs16nextclient.com>
- Inno Setup 6 (скачать): <https://jrsoftware.org/isdl.php>
