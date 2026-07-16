# serez-pack

[Serez-Code](https://serezcode.org) application packager: ship your app as a **self-contained**
bundle where the `sz.exe` runtime travels **inside** the package — the end user does not need
Serez-Code installed. Written in pure `.sz` (dogfooding); it never touches the core.

## Install & use

```powershell
sz install serez-pack
```

```powershell
# Self-contained folder (no extra toolchain needed)
sz packages\serez-pack\pack.sz entry=my_app.sz name=MyApp format=folder

# .msi installer (auto-installs WiX if missing)
sz packages\serez-pack\pack.sz entry=my_app.sz name=MyApp format=msi

# .exe installer (Burn bundle wrapping the .msi)
sz packages\serez-pack\pack.sz entry=my_app.sz name=MyApp format=exe

# An app that uses serez-ui
sz packages\serez-pack\pack.sz entry=my_app.sz name=MyApp serez-ui=..\serez-ui serez-json=serez.json format=msi
```

> ⚠️ Options are `key=value` **without dashes** — `sz.exe` rejects unknown `--flags`.

## Formats

| `format=` | Output | Toolchain |
|-----------|--------|-----------|
| `folder`  | `<out>\<AppName>\` — folder that runs with the embedded `sz.exe` | none |
| `msi`     | `<out>\<AppName>.msi` — installs into `Program Files\<AppName>` | WiX 5 (auto-installed) |
| `exe`     | `<out>\<AppName>Setup.exe` — Burn bundle that installs the .msi | WiX 5 (auto-installed) |

## Options

| Option        | Default                | Description |
|---------------|------------------------|-------------|
| `entry=`      | (required)             | Path to the app's `.sz` entry file |
| `name=`       | (required)             | Name of the app / package |
| `out=`        | `dist`                 | Output directory |
| `sz=`         | the executing `sz.exe` | Runtime to embed |
| `serez-ui=`   | (empty)                | Path to serez-ui (if the app imports it) |
| `serez-json=` | (empty)                | `serez.json` with the app's permissions |
| `format=`     | `folder`               | `folder` \| `msi` \| `exe` |

## How it works

The output folder contains `sz.exe` (embedded runtime), your entry `.sz`, the app's `serez.json`
(permissions) and any bundled libraries (e.g. `serez-ui/`). The runtime resolves imports from the
app's own directory and the `sz.exe` directory, so the package runs anywhere — the same layout
backs the folder, the `.msi` and the `.exe` bundle.

For `.msi`/`.exe`, WiX 5 is a **managed dependency**: if missing, serez-pack auto-installs it
with `dotnet tool install --global wix` (needs the .NET SDK). Include a `"version"` field in your
app's `serez.json` — the runtime needs it to load permissions (serez-pack warns if it's missing).

## Documentation

- **[serez-pack reference](https://serezcode.org/docs/serez-pack)** — options, formats and
  troubleshooting, on the Serez-Code site.
- **[Package your app](https://serezcode.org/guides/packaging)** — step-by-step tutorial from a
  working app to a distributable installer.
