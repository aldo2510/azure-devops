# Guía: Crear e instalar un Agente de Azure DevOps en Windows como **Servicio**

Esta guía en **Markdown** te lleva de cero a tener un agente autoalojado
de Azure DevOps ejecutándose como **servicio de Windows**.

------------------------------------------------------------------------

## 1) Requisitos previos

-   **Windows** Server/Pro (x64) con acceso a Internet.
-   **.NET Framework 4.6.2+** (normalmente ya instalado en Windows
    Server 2016+).
-   **Cuenta de servicio** local o de dominio (recomendado) con:
    -   Permiso de *Log on as a service* (se concede durante la
        instalación si usas `--runAsService`).
    -   Lectura/escritura sobre el directorio del agente (p. ej.
        `C:\azagent`).
    -   Acceso a los repos/artefactos (si usas rutas UNC, dale permisos
        adecuados).
-   **PAT (Personal Access Token)** en Azure DevOps con alcance **Agent
    Pools (Read & manage)** y **Deployment Groups si aplica**.
-   **Nombre de organización** de Azure DevOps:
    `https://dev.azure.com/<ORGANIZACION>`.
-   **Nombre del Pool** donde registrarás el agente (p. ej. `Default` o
    uno dedicado).

> 💡 Seguridad: guarda el PAT de forma segura. Puedes revocarlo tras
> configurar el agente si habilitas **Auto‑update**.

------------------------------------------------------------------------

## 2) Crear PAT (Token personal)

1.  Entra a **Azure DevOps** → **User settings** (icono de usuario) →
    **Personal access tokens**.
2.  **New Token** → Nombre descriptivo (p. ej. `agent-win-srv01`) →
    Organization: tu organización.
3.  **Scopes**: marca **Agent Pools (Read & manage)**. Agrega otros
    scopes solo si los necesitas.
4.  Crea el token y **cópialo** (solo se muestra una vez).

------------------------------------------------------------------------

## 3) Preparar la carpeta del agente

``` powershell
# Ejecutar en PowerShell como Administrador
New-Item -ItemType Directory -Force -Path C:\azagent | Out-Null
cd C:\azagent
```

------------------------------------------------------------------------

## 4) Descargar el binario del agente (Windows x64)

> El agente se publica como `.zip` versionado. Sustituye `<VERSION>` por
> la versión estable que estés usando en tu organización.

``` powershell
# Ejemplo con variables (ajusta la versión)
$version = "<VERSION>"            # ej. 3.240.1
$zip     = "vsts-agent-win-x64-$version.zip"
$uri     = "https://download.agent.dev.azure.com/agent/$version/$zip"

Invoke-WebRequest -Uri $uri -OutFile $zip
Expand-Archive -Path $zip -DestinationPath . -Force
Remove-Item $zip
```

> Si tu servidor no tiene salida directa a Internet, descarga el `.zip`
> en otro equipo y cópialo a `C:\azagent`.

------------------------------------------------------------------------

## 5) Configuración **interactiva** (rápida)

``` cmd
cd C:\azagent
config.cmd
```

Responde a los prompts: 
- **Server URL**:
`https://dev.azure.com/<ORGANIZACION>` 
- **Authentication type**:
`PAT` - **PAT**: pega tu token 
- **Agent Pool**: el pool deseado (p. ej.
`Default`) 
- **Agent name**: nombre único (p. ej. `win-srv01`)\
- **Work folder**: `_work` (recomendado) 
- **Run as service?**: `Y`\
- **User account** para el servicio: `DOMINIO\svc-azdo` o `.\svc-azdo`
(si local)\
- **Password**: contraseña de la cuenta

Inicia el servicio (si no se inicia automáticamente):

``` cmd
svc start
```

> Si prefieres **no** ejecutarlo como servicio en la configuración,
> también puedes:
>
> ``` cmd
> svc install
> svc start
> ```

------------------------------------------------------------------------

## 6) Configuración **desatendida** (scriptable / IaC)

> Ideal para automatización con PowerShell/Ansible/Chocolatey/WinRM,
> etc.

``` cmd
cd C:\azagent
config.cmd --unattended ^
  --url https://dev.azure.com/<ORGANIZACION> ^
  --auth pat ^
  --token <TU_PAT> ^
  --pool "<POOL>" ^
  --agent "<NOMBRE_AGENTE>" ^
  --work "_work" ^
  --acceptTeeEula ^
  --runAsService ^
  --windowsLogonAccount "<DOMINIO\\svc-azdo>" ^
  --windowsLogonPassword "<CONTRASEÑA>"

svc start
```

Parámetros útiles adicionales: - `--replace` para reutilizar el nombre
si ya existía. - `--addDeploymentGroupTags` y opciones de Deployment
Group si lo usas para Release Management clásico.

------------------------------------------------------------------------

## 7) Validación rápida

``` powershell
# Estado del servicio
Get-Service vstsagent* | Select-Object Name, Status

# Log del agente
Get-Content -Path .\_diag\*.log -Tail 200 -Wait
```

En Azure DevOps → **Project settings** → **Agent Pools** → tu pool →
**Agents** debe verse el agente **Online**.

------------------------------------------------------------------------

## 8) Firewall / Proxy (opcional)

-   Permite salida a dominios de Azure DevOps y Azure (puertos 80/443).\
-   Si hay proxy:
    -   Establece variables de entorno del sistema
        `VSO_AGENT_HTTP_PROXY` y/o
        `VSO_AGENT_HTTP_PROXY_USERNAME/PASSWORD` **antes** de configurar
        el agente.
    -   O usa `--proxyurl`, `--proxyusername`, `--proxypassword` con
        `config.cmd`.

Ejemplo (PowerShell, proxy simple):

``` powershell
[Environment]::SetEnvironmentVariable("VSO_AGENT_HTTP_PROXY", "http://proxy.miempresa.local:8080", "Machine")
```

------------------------------------------------------------------------

## 9) Actualizaciones del agente

El agente **se auto‑actualiza** cuando Azure DevOps lo requiere. Si
necesitas forzar una reinstalación:

``` cmd
# Detener servicio y quitar configuración
svc stop
config.cmd remove --unattended --auth pat --token <TU_PAT>

# (Opcional) Borrar carpeta y repetir instalación con una versión nueva
```

------------------------------------------------------------------------

## 10) Desinstalar / mover

``` cmd
svc stop
svc uninstall
config.cmd remove
# Luego elimina la carpeta C:\azagent si no la reutilizarás
```

------------------------------------------------------------------------

## 11) Consejos y buenas prácticas

-   Usa **cuentas de servicio dedicadas** con los mínimos privilegios.
-   Separa pools por **tipo de carga** (builds, despliegues, PR,
    self‑hosted runners con GPU, etc.).
-   Monitorea los **logs** en `C:\azagent\_diag` y el **Event Viewer** →
    *Windows Logs* → *Application*.
-   Si compilas .NET/Node/Java, instala previamente los **build
    tools/SDKs** necesarios.
-   Para limpiar espacio, automatiza `C:\azagent\_work\_temp` y
    pipelines con `checkout: self clean: true`.

------------------------------------------------------------------------

## 12) Ejemplo completo (PowerShell, desatendido)

> Ajusta ORGANIZACION, POOL, NOMBRE, CUENTA y VERSION.

``` powershell
$org      = "<ORGANIZACION>"
$pool     = "Default"
$agent    = "win-srv01"
$svcUser  = "DOMINIO\\svc-azdo"
$svcPass  = "<CONTRASEÑA>"
$pat      = "<TU_PAT>"
$version  = "<VERSION>"   # ej. 3.240.1

New-Item -ItemType Directory -Force -Path C:\azagent | Out-Null
cd C:\azagent

$zip = "vsts-agent-win-x64-$version.zip"
$uri = "https://vstsagentpackage.azureedge.net/agent/$version/$zip"
Invoke-WebRequest -Uri $uri -OutFile $zip
Expand-Archive -Path $zip -DestinationPath . -Force
Remove-Item $zip

cmd /c "config.cmd --unattended --url https://dev.azure.com/$org --auth pat --token $pat --pool $pool --agent $agent --work _work --acceptTeeEula --runAsService --windowsLogonAccount $svcUser --windowsLogonPassword $svcPass"

cmd /c "svc start"
```

------------------------------------------------------------------------

## 13) Problemas comunes

-   **401/403 al registrar**: revisa el *scope* del PAT y la URL de la
    organización.
-   **Servicio no inicia**: credenciales de la cuenta, permisos en
    carpeta, antivirus bloqueando `Agent.Listener.exe`.
-   **Detección de capacidades fallida**: faltan SDKs/herramientas en el
    host.
-   **Detrás de proxy**: define variables `VSO_AGENT_HTTP_PROXY*` o
    flags de proxy antes de `config.cmd`.

------------------------------------------------------------------------

## 14) Referencias rápidas (comandos)

``` cmd
# Iniciar / detener / estado del servicio
svc start
svc stop
sc query type= service state= all | findstr /I vstsagent

# Reconfigurar
config.cmd remove
config.cmd --help

# Logs
more _diag\Agent_*.log
```

------------------------------------------------------------------------

> **Listo**: con esto tendrás el agente instalado como servicio y
> visible en tu Pool.
