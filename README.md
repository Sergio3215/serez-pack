# serez-pack

**v1.0.0** · [Serez-Code](../Serez-code) application packager to a **self-contained** bundle: the `sz.exe` runtime travels **inside** the package, so the end user **does not need to install Serez-Code**. Written in pure `.sz` (dogfooding); **does not touch** the serez-code core.

## Installation

```powershell
sz install serez-pack
```

It is installed in `./packages/serez-pack/`. Run it with:

```powershell
sz packages\serez-pack\pack.sz entry=my_app.sz name=MyApp format=msi
```

Or use it directly from the repo (cloned): `sz pack.sz entry=… name=…`.

## Formats

| `format=` | Output | Toolchain | Status |
|-----------|--------|-----------|--------|
| `folder`  | `<out>\<AppName>\` executable with `sz.exe app.sz` | — (none) | ✅ |
| `msi`     | `<out>\<AppName>.msi` (installs in `Program Files\<AppName>`) | WiX | ✅ |
| `exe`     | `<out>\<AppName>Setup.exe` (Burn bundle that installs the .msi) | WiX | ✅ |

## How it works

```
app.sz + serez-ui + sz.exe  ──pack.sz──▶  <AppName>/      (self-contained folder)
                                              sz.exe       embedded runtime
                                              app.sz       your app (entry)
                                              serez.json   permissions (optional)
                                              serez-ui/    index.sz + src/ (if used)
```

The runtime resolves `import "serez-ui"` by looking into, among others, the **app's own directory and the `sz.exe` directory** — therefore, with `serez-ui/` next to `sz.exe`, the import resolves without relying on the working directory. This applies equally to the folder, the `.msi` (Program Files) and the `.exe` bundle.

## Usage

> ⚠️ Options are passed as `key=value` **without dashes**: `sz.exe` rejects any unknown `--flag` and would abort before executing the script.

```powershell
$sz = "..\Serez-code\target\release\sz.exe"

# Self-contained folder (no extra toolchain)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=folder

# .msi installer (auto-installs WiX if missing)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=msi

# .exe installer (Burn bundle wrapping the .msi)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=exe

# With an app using serez-ui
& $sz pack.sz entry=my_app.sz name=MyApp serez-ui=..\serez-ui serez-json=..\serez-ui\serez.json format=msi
```

### Options

| Option        | Default                     | Description |
|---------------|-----------------------------|-------------|
| `entry=`      | (required)                  | Path to the app's `.sz` (entry) |
| `name=`       | (required)                  | Name of the app / package |
| `out=`        | `dist`                      | Output directory |
| `sz=`         | the executing `sz.exe`      | Runtime to embed |
| `serez-ui=`   | (empty)                     | Path to serez-ui repo (if imported by the app) |
| `serez-json=` | (empty)                     | `serez.json` with the app's permissions |
| `format=`     | `folder`                    | `folder` \| `msi` \| `exe` |

## Prerequisites

- **Folder (`folder`)**: none.
- **`.msi` / `.exe`**: **WiX** as a **managed dependency**. Declared in `serez.json`:
  ```json
  "dependencies": { "wix": "5" }
  ```
  If not installed, serez-pack **auto-installs** it with `dotnet tool install --global wix` (requires the **.NET SDK**, which you already have). It is intentionally pinned to **WiX 5**: v6+ requires accepting the OSMF EULA. For `.exe` it also auto-adds the Burn extension pinned to the same version. If `wix` is **not** declared and is missing, `msi`/`exe` will warn with a clear message.

> The `.msi` installs into `Program Files\<AppName>` (requires elevation during install) and creates a shortcut that launches `sz.exe app.sz`. The `.exe` is a Burn bundle that installs that `.msi`.

> Note: 7-Zip SFX was discarded — generating its autorun requires concatenating binaries, which is unfeasible in pure `.sz` without touching the core. WiX assembles the installer from files on disk, without binary manipulation.

## Structure

```
serez-pack/
  pack.sz              orchestrator (options parsing → staging → format)
  src/
    strutil.sz         string & path helpers (incl. replaceAll, strIndexOf)
    args.sz            key=value options reading
    fsutil.sz          copyFile (binary-safe) + copyTree
    staging.sz         builds the self-contained folder
    detect.sz          locates WiX / dotnet
    deps.sz            WiX as dependency: declaresDep + auto-install
    wxs.sz             generates .wxs (MSI Product + Bundle) from templates
    msi.sz             wix build for .msi and .exe bundle (+ Burn extension)
  templates/
    product.wxs.template   MSI template (placeholders @...@)
    bundle.wxs.template    Burn bundle template
  apps/
    hello_pack.sz      minimal demo (no serez-ui)
    ui_smoke.sz        demo importing serez-ui
  serez.json           dependencies: { wix: 5 } · permissions: OS, File, Env
```
