# serez-pack

**v1.0.0** · Empaquetador de aplicaciones [Serez-Code](../Serez-code) a un paquete
**autocontenido**: el runtime `sz.exe` viaja **dentro** del paquete, así que el usuario final
**no necesita instalar Serez-Code**. Escrito en `.sz` puro (dogfooding); **no toca el core** de
serez-code.

## Instalación

```powershell
sz install serez-pack
```

Queda en `./packages/serez-pack/`. Ejecútalo con:

```powershell
sz packages\serez-pack\pack.sz entry=mi_app.sz name=MiApp format=msi
```

O úsalo directamente desde el repo (clonado): `sz pack.sz entry=… name=…`.

## Formatos

| `format=` | Salida | Toolchain | Estado |
|-----------|--------|-----------|--------|
| `folder`  | `<out>\<AppName>\` ejecutable con `sz.exe app.sz` | — (ninguno) | ✅ |
| `msi`     | `<out>\<AppName>.msi` (instala en `Program Files\<AppName>`) | WiX | ✅ |
| `exe`     | `<out>\<AppName>Setup.exe` (bundle Burn que instala el .msi) | WiX | ✅ |

## Cómo funciona

```
app.sz + serez-ui + sz.exe  ──pack.sz──▶  <AppName>/      (carpeta autocontenida)
                                              sz.exe       runtime embebido
                                              app.sz       tu app (entry)
                                              serez.json   permisos (opcional)
                                              serez-ui/    index.sz + src/ (si la usas)
```

El runtime resuelve `import "serez-ui"` mirando, entre otros, el **directorio de la propia
app y el del `sz.exe`** — por eso, con `serez-ui/` junto a `sz.exe`, el import resuelve sin
depender del directorio de trabajo. Vale igual para la carpeta, el `.msi` (Program Files) y
el bundle `.exe`.

## Uso

> ⚠️ Las opciones van como `clave=valor` **sin guiones**: `sz.exe` rechaza cualquier `--flag`
> desconocido y abortaría antes de ejecutar el script.

```powershell
$sz = "..\Serez-code\target\release\sz.exe"

# Carpeta autocontenida (sin toolchain extra)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=folder

# Instalador .msi (auto-instala WiX si falta)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=msi

# Instalador .exe (bundle Burn que envuelve el .msi)
& $sz pack.sz entry=apps\hello_pack.sz name=HelloPack out=dist format=exe

# Con una app que usa serez-ui
& $sz pack.sz entry=mi_app.sz name=MiApp serez-ui=..\serez-ui serez-json=..\serez-ui\serez.json format=msi
```

### Opciones

| Opción        | Por defecto                 | Descripción |
|---------------|-----------------------------|-------------|
| `entry=`      | (obligatoria)               | Ruta al `.sz` de la app (entry) |
| `name=`       | (obligatoria)               | Nombre de la app / del paquete |
| `out=`        | `dist`                      | Directorio de salida |
| `sz=`         | el `sz.exe` en ejecución    | Runtime a embeber |
| `serez-ui=`   | (vacío)                     | Ruta al repo serez-ui (si la app lo importa) |
| `serez-json=` | (vacío)                     | `serez.json` con los permisos de la app |
| `format=`     | `folder`                    | `folder` \| `msi` \| `exe` |

## Prerequisitos

- **Carpeta (`folder`)**: ninguno.
- **`.msi` / `.exe`**: **WiX** como **dependencia gestionada**. Se declara en `serez.json`:
  ```json
  "dependencies": { "wix": "5" }
  ```
  Si no está instalado, serez-pack lo **auto-instala** con `dotnet tool install --global wix`
  (necesita el **.NET SDK**, que ya traes). Se fija a **WiX 5** a propósito: la v6+ exige aceptar
  la EULA de OSMF. Para `.exe` también auto-añade la extensión Burn fijada a la misma versión.
  Si `wix` **no** está declarado y falta, `msi`/`exe` avisan con un mensaje claro.

> El `.msi` instala en `Program Files\<AppName>` (requiere elevación al instalar) y crea un acceso
> directo que lanza `sz.exe app.sz`. El `.exe` es un bundle Burn que instala ese `.msi`.

> Nota: el SFX de 7-Zip se descartó — generar su autorun exige concatenar binarios, algo
> inviable en `.sz` puro sin tocar el core. WiX ensambla el instalador desde los archivos del
> disco, sin manipulación binaria.

## Estructura

```
serez-pack/
  pack.sz              orquestador (parse de opciones → staging → formato)
  src/
    strutil.sz         helpers de string y rutas (incl. replaceAll, strIndexOf)
    args.sz            lectura de opciones clave=valor
    fsutil.sz          copyFile (binario-safe) + copyTree
    staging.sz         arma la carpeta autocontenida
    detect.sz          localiza WiX / dotnet
    deps.sz            WiX como dependencia: declaresDep + auto-instalación
    wxs.sz             genera los .wxs (MSI Product + Bundle) desde plantillas
    msi.sz             wix build del .msi y del bundle .exe (+ extensión Burn)
  templates/
    product.wxs.template   plantilla del MSI (placeholders @...@)
    bundle.wxs.template    plantilla del bundle Burn
  apps/
    hello_pack.sz      demo mínimo (sin serez-ui)
    ui_smoke.sz        demo que importa serez-ui
  serez.json           dependencies: { wix: 5 } · permisos: OS, File, Env
```
