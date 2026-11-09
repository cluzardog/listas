# Plan — Demo local + Cloudflare Tunnel (primera intención)

- **Estado:** BORRADOR — primera intención, **no ejecutado**
- **Fecha:** 2026-11-09
- **Ubicación:** fuera del repositorio (`C:\Users\cluzardo\.opencode\plan`)
- **Proyecto:** CampMed — `D:\Angular2026\ExamenFinalAngular`
- **Objetivo:** mostrar la aplicación al usuario en una reunión de Zoom/Teams, con la app corriendo en la PC del desarrollador, sin cambios en el repositorio, sin tarjeta, sin dominio propio y sin Swagger.

---

## 1. Resultado esperado

Una URL pública temporal `https://<aleatorio>.trycloudflare.com` que abre la app Angular desde la PC del desarrollador. El usuario remoto puede mirar por pantalla compartida y, si quiere, hacer clics desde su propia PC. Al terminar, se cierran los procesos y no queda nada en la nube.

**Modo principal:** compartir pantalla en Zoom/Teams.
**Modo secundario:** pasar la URL para que el usuario haga clics remotos.

---

## 2. Decisiones ya aprobadas (de la conversación)

| Tema | Decisión |
|------|----------|
| Alternativa elegida | **C1: local + Cloudflare Quick Tunnel** (mínimo cambio) |
| Alternativa A (Pages + Render + Neon) | Solo si la PC no puede quedar encendida (no es el caso) |
| Alternativa B (Oracle) | **Descartada** para esta demo (2–3 días de montaje, tarjeta, riesgo de reclaim) |
| Dominio propio | **No se usa**. La URL del túnel es temporal y desechable |
| Duración | 2–3 días; luego se destruye |
| Usuario remoto | Reunión Zoom/Teams; al inicio solo mira, pero puede pedir hacer clics |
| Cambios en el repo | **Ninguno**. Solo flags de CLI y archivos fuera del repo |
| Base de datos | PostgreSQL 18 local con data precargada; se restaura si se ensucia |
| Swagger | No se necesita; todo se prueba desde la app |
| URL estable | No se necesita; puede cambiar entre demos |

---

## 3. Alcance

### Incluido
- Levantar API .NET 10, frontend Angular 22 y PostgreSQL 18 en la PC.
- Exponer **solo el puerto 4200** (frontend) mediante Cloudflare Quick Tunnel.
- Probar desde otra red (celular con datos móviles) y desde un navegador distinto.
- Ensayo general tipo reunión (pantalla compartida + URL).
- Reset de la base local al terminar, si fue necesario.

### Fuera de alcance
- Despliegue en Render, Neon, Oracle, Azure, AWS o cualquier cloud.
- Docker, Dockerfile, `environment.production.ts`, CORS, `/health`.
- Dominio propio, DNS, certificados, Cloudflare Access.
- Cambios en `appsettings.json`, `Program.cs`, `angular.json` o cualquier archivo del repo.
- Documentación dentro de `docs/` (se hará cuando se decida la tecnología de producción).

---

## 4. Arquitectura de la demo

```text
PC del usuario remoto
  │  https://<aleatorio>.trycloudflare.com
  ▼
Cloudflare (TLS + túnel saliente)
  │
  ▼
PC del desarrollador
  ├─ cloudflared → http://localhost:4200
  ├─ Angular dev server (npm start)
  │     └─ /api  →  proxy.conf.json  →  http://localhost:5143
  ├─ API .NET 10 (Kestrel)
  └─ PostgreSQL 18 (localhost:5432, base bdcampmed)
```

- Un solo origen para el navegador: `/api` se resuelve en el mismo host del túnel.
- No hay CORS porque el navegador nunca llama directo a `localhost:5143`.
- La base no se expone: solo sale el puerto 4200 por el túnel.

---

## 5. Estado actual del entorno (verificado el 2026-11-09)

| Componente | Estado | Nota |
|------------|--------|------|
| Node.js | `v24.22.1` | Compatible con Angular 22 |
| npm | `11.19.1` | — |
| .NET SDK | `10.0.403` | Proyecto `net10.0` |
| PostgreSQL | Servicio `postgresql-x64-18` **Running** | Base `bdcampmed` |
| `cloudflared` | **No instalado** | Instalar con `winget install --id Cloudflare.cloudflared` |
| `node_modules` del frontend | **No existe** | Primer `npm ci` (puede tardar 3–5 min) |
| `bin` del backend | **No existe** | Primer `dotnet build` (1–2 min) |
| Puertos 4200 / 5143 | Libres | Verificar antes de arrancar |
| Backup más reciente | `bdcampmed_202611082300.backup` = **0 bytes** | **No usar**. Último válido: `bdcampmed_202611072330.backup` (492 KB) |
| Usuario admin de datos iniciales | `admin` / `Admin123` | Cambiar la clave antes de exponer el túnel |

> La clave `Admin123` está documentada en `database/datos/02_datos_iniciales.sql`. Todos los usuarios de `05_datos_prueba.sql` comparten el mismo hash. Para una URL pública, aunque sea aleatoria, se recomienda cambiar la clave de `admin`.

---

## 6. Etapas

### Etapa 0 — Preparación (30–45 min)

- [ ] Instalar `cloudflared`:
  ```powershell
  winget install --id Cloudflare.cloudflared
  cloudflared --version
  ```
- [ ] Verificar variables AES en el entorno de usuario:
  ```powershell
  [Environment]::GetEnvironmentVariable("CAMPMED_KEY","User")
  [Environment]::GetEnvironmentVariable("CAMPMED_JWT_KEY","User")
  ```
- [ ] Verificar que PostgreSQL 18 esté corriendo y que la base tenga datos:
  ```powershell
  Get-Service postgresql-x64-18
  & "C:\Program Files\PostgreSQL\18\bin\psql.exe" -h localhost -U postgres -d bdcampmed -c "select count(*) from usuario;"
  ```
- [ ] Verificar login con `admin` / `Admin123` en `http://localhost:4200` (cuando el frontend esté arriba).
- [ ] Cambiar la clave de `admin` a una clave de demo fuerte:
  ```powershell
  # 1. Generar hash
  dotnet run --project backend\CampMedTools -- --hash "CampMedDemo2026!"
  # 2. Actualizar en la BD (pegar el hash generado)
  # UPDATE USUARIO SET PASSWORD_HASH = '<hash>' WHERE LOGIN = 'admin';
  ```
- [ ] Evitar que la PC se duerma:
  ```powershell
  powercfg /change standby-timeout-ac 0
  powercfg /change hibernate-timeout-ac 0
  ```
- [ ] Instalar dependencias del frontend (solo la primera vez):
  ```powershell
  cd frontend\campmed-web
  npm ci
  ```
- [ ] Compilar la API (solo la primera vez):
  ```powershell
  dotnet build backend\CampMedApi
  ```
- [ ] Comprobar que los puertos estén libres:
  ```powershell
  Get-NetTCPConnection -LocalPort 4200,5143 -ErrorAction SilentlyContinue
  ```

### Etapa 1 — Ensayo local (15 min)

- [ ] Terminal 1 — API:
  ```powershell
  $env:CAMPMED_KEY     = [Environment]::GetEnvironmentVariable("CAMPMED_KEY","User")
  $env:CAMPMED_JWT_KEY = [Environment]::GetEnvironmentVariable("CAMPMED_JWT_KEY","User")
  dotnet run --project backend\CampMedApi
  ```
- [ ] Terminal 2 — Frontend:
  ```powershell
  cd frontend\campmed-web
  npm start
  ```
- [ ] Abrir `http://localhost:4200`, iniciar sesión y recorrer: campaña → pacientes → atención → receta → entrega → reportes.
- [ ] Revisar DevTools → Network: las llamadas `/api/v1/...` deben responder 200 y no debe haber errores de CORS.
- [ ] Verificar que una mutación (crear/editar algo) persista al refrescar la grilla.

### Etapa 2 — Túnel y prueba remota (15–20 min)

- [ ] Reiniciar el frontend con el host del túnel permitido:
  ```powershell
  npm start -- --allowed-hosts .trycloudflare.com --no-live-reload
  ```
  > El flag es necesario porque el dev server de Angular/Vite bloquea hostnames desconocidos. No modifica archivos.
- [ ] Terminal 3 — Túnel:
  ```powershell
  cloudflared tunnel --url http://localhost:4200
  ```
- [ ] Copiar la URL `https://<aleatorio>.trycloudflare.com`.
- [ ] Abrir la URL desde el celular **con datos móviles** (no Wi-Fi) y verificar:
  - [ ] Carga la app con candado HTTPS.
  - [ ] Login funciona.
  - [ ] Una pantalla con datos y una acción (clic) funcionan.
  - [ ] No hay errores en consola ni pantalla en blanco.
- [ ] Medir el tiempo de la primera carga y de la primera llamada al API.

### Etapa 3 — Ensayo general tipo reunión (15–20 min)

- [ ] Simular Zoom/Teams con un segundo dispositivo o un compañero:
  - [ ] Compartir pantalla y, al mismo tiempo, pegar la URL en el chat.
  - [ ] Probar la URL en una ventana de incógnito.
  - [ ] Probar en otro navegador (Edge/Chrome).
- [ ] Ejecutar el guion de demo (sección 8) de principio a fin.
- [ ] Anotar ajustes: latencia, tamaño de letra, pantallas que conviene mostrar, datos que faltan.

### Etapa 4 — Día de la demo

**T-30 minutos:**
- [ ] API, frontend y túnel arriba.
- [ ] URL probada desde el celular.
- [ ] Login de `admin` probado.
- [ ] Terminales visibles y sin actualizaciones de Windows pendientes.
- [ ] Tener a mano el usuario/clave de demo.

**Durante:**
- [ ] Compartir pantalla primero.
- [ ] Pegar la URL en el chat: «si quieres entrar y hacer clic, este es el enlace».
- [ ] Dejar que el usuario navegue; observar y asistir.
- [ ] Si el usuario ensucia datos, no corregir en vivo; anotar y resetear después.

**Contingencia:**
- [ ] Si el túnel no responde: reiniciar `cloudflared` (genera URL nueva) y volver a pegarla.
- [ ] Si el dev server bloquea el host: verificar el flag `--allowed-hosts`.
- [ ] Si la red del usuario bloquea `trycloudflare.com`: volver a compartir pantalla.
- [ ] Si la API falla: revisar la terminal 1, reiniciar `dotnet run`.
- [ ] Si todo falla: demo 100 % por pantalla compartida; la app local sigue funcionando.

### Etapa 5 — Cierre

- [ ] `Ctrl+C` en las tres terminales (API, frontend, túnel).
- [ ] Verificar que no queden procesos:
  ```powershell
  Get-Process cloudflared,node,dotnet -ErrorAction SilentlyContinue
  ```
- [ ] Si se ensució la base, restaurar el backup válido o resetear:
  ```powershell
  # Opción A: restaurar backup válido (detener la API primero)
  & "C:\Program Files\PostgreSQL\18\bin\pg_restore.exe" -h localhost -U postgres -d bdcampmed --clean --if-exists "database\backup_database\bdcampmed_202611072330.backup"

  # Opción B: reset + datos iniciales + datos de prueba
  # 03_reset.sql → 02_datos_iniciales.sql → 05_datos_prueba.sql
  ```
- [ ] Registrar feedback del usuario para la siguiente demo.
- [ ] **No hay nada que borrar en la nube.**

---

## 7. Criterios de aceptación

- [ ] La URL `https://<aleatorio>.trycloudflare.com` abre desde una red externa (datos móviles).
- [ ] El login con `admin` funciona y el JWT se envía en las llamadas.
- [ ] Se muestran al menos 5 pantallas: login, campaña, pacientes, atención/receta, entrega/reportes.
- [ ] Una mutación hecha por el usuario persiste en la base.
- [ ] No hay errores de CORS, contenido mixto ni pantalla en blanco.
- [ ] El usuario remoto puede hacer clics sin instalar nada.
- [ ] Al cerrar las terminales, la URL deja de responder.
- [ ] El repositorio queda sin cambios (`git status` limpio).

---

## 8. Guion de demo (7–10 minutos)

1. **Login** como `admin` → mostrar menú completo y selección de campaña.
2. **Campaña** → abrir la campaña de prueba y mostrar el contexto global.
3. **Pacientes** → buscar por documento, abrir uno y mostrar sus servicios.
4. **Atención/triaje** → registrar o abrir una consulta, mostrar la receta.
5. **Entrega de medicamentos** → mostrar stock, pendientes y entrega.
6. **Reportes/kardex** → mostrar una métrica o gráfico.
7. **Cierre** → preguntar al usuario qué flujo quiere ver con más detalle.

> Si el usuario pide hacer clics, repetir el mismo recorrido dejándolo manejar el mouse.

---

## 9. Seguridad para una URL pública temporal

- [ ] Cambiar la clave de `admin` (y de cualquier usuario que se vaya a mostrar).
- [ ] Exponer **solo el puerto 4200** por el túnel; nunca 5143 ni 5432.
- [ ] No abrir puertos en el router ni en el firewall de Windows; el túnel es saliente.
- [ ] No compartir la URL fuera de la reunión.
- [ ] Cerrar el túnel al terminar; no dejarlo corriendo de noche sin necesidad.
- [ ] No usar datos reales de pacientes en la demo; usar solo `05_datos_prueba.sql`.

---

## 10. Riesgos y mitigaciones

| Riesgo | Probabilidad | Mitigación |
|--------|--------------|------------|
| La PC se suspende | Media | `powercfg` + PC enchufada |
| El dev server bloquea el host del túnel | Alta | Flag `--allowed-hosts .trycloudflare.com` |
| WebSocket/HMR da ruido por el túnel | Media | `--no-live-reload` |
| La red del usuario bloquea `trycloudflare.com` | Media | Compartir pantalla |
| Internet del desarrollador se cae | Media | Hotspot móvil o demo por pantalla |
| El túnel cambia de URL | Baja | Reiniciar y pegar la nueva URL |
| La base queda con datos de la demo | Alta | Restaurar backup o reset después |
| El usuario entra con un navegador viejo | Baja | Probar Edge/Chrome; evitar IE |
| `UseHttpsRedirection` genera redirección rara | Baja | La API local no tiene puerto HTTPS configurado; verificar en el ensayo |
| Falta `cloudflared` | Confirmada | Instalarlo en Etapa 0 |

---

## 11. Limpieza / rollback

- Detener los tres procesos.
- Borrar el túnel (no hay recurso persistente).
- Si se cambió la clave de `admin`, decidir si se deja la nueva o se restaura el hash original.
- Si se agregó algún archivo fuera del repo (`start-demo.ps1`), puede quedarse para la próxima demo.
- El repo no se toca: `git status` debe quedar limpio.

---

## 12. Anexo A — `start-demo.ps1` (a crear fuera del repo, en Etapa 4)

> **No crear todavía.** Este script se implementará después de aprobar el plan, fuera del repositorio (por ejemplo, en `C:\Users\cluzardo\.opencode\plan\start-demo.ps1`).

```powershell
# start-demo.ps1 — levanta API, frontend y túnel en ventanas separadas.
# Uso: powershell -NoProfile -ExecutionPolicy Bypass -File .\start-demo.ps1

$ErrorActionPreference = 'Stop'
$repo  = 'D:\Angular2026\ExamenFinalAngular'
$front = Join-Path $repo 'frontend\campmed-web'

# 0. Prerrequisitos
if (-not (Get-Command cloudflared -ErrorAction SilentlyContinue)) {
    throw 'Falta cloudflared. Instala con: winget install --id Cloudflare.cloudflared'
}
if (-not (Test-Path (Join-Path $front 'node_modules'))) {
    throw "Falta node_modules. Ejecuta: cd `"$front`"; npm ci"
}

# 1. Variables AES (el proceso hijo las hereda)
$env:CAMPMED_KEY     = [Environment]::GetEnvironmentVariable('CAMPMED_KEY','User')
$env:CAMPMED_JWT_KEY = [Environment]::GetEnvironmentVariable('CAMPMED_JWT_KEY','User')
if (-not $env:CAMPMED_KEY -or -not $env:CAMPMED_JWT_KEY) {
    throw 'Faltan CAMPMED_KEY / CAMPMED_JWT_KEY en el entorno de usuario.'
}

# 2. API
Start-Process powershell -ArgumentList @(
    '-NoExit','-NoProfile','-Command',
    "Set-Location '$repo'; dotnet run --project backend\CampMedApi"
) | Out-Null

# 3. Frontend (con el host del túnel permitido)
Start-Sleep -Seconds 8
Start-Process powershell -ArgumentList @(
    '-NoExit','-NoProfile','-Command',
    "Set-Location '$front'; npm start -- --allowed-hosts .trycloudflare.com --no-live-reload"
) | Out-Null

# 4. Túnel
Start-Sleep -Seconds 8
Start-Process powershell -ArgumentList @(
    '-NoExit','-NoProfile','-Command',
    'cloudflared tunnel --url http://localhost:4200'
) | Out-Null

Write-Host ''
Write-Host 'Revisa la ventana del túnel y copia la URL https://*.trycloudflare.com' -ForegroundColor Cyan
Write-Host 'Prueba esa URL desde el celular con datos móviles antes de la reunión.' -ForegroundColor Cyan
```

---

## 13. Anexo B — Comandos de base de datos

### Restaurar el backup válido

```powershell
# Detener la API antes de restaurar.
& "C:\Program Files\PostgreSQL\18\bin\pg_restore.exe" `
  -h localhost -U postgres -d bdcampmed `
  --clean --if-exists `
  "database\backup_database\bdcampmed_202611072330.backup"
```

### Reset + datos iniciales + datos de prueba

```powershell
$psql = "C:\Program Files\PostgreSQL\18\bin\psql.exe"
Get-ChildItem database\reset\03_reset.sql,
              database\datos\02_datos_iniciales.sql,
              database\datos\05_datos_prueba.sql |
  ForEach-Object {
    & $psql -h localhost -U postgres -d bdcampmed -v ON_ERROR_STOP=1 -f $_.FullName
  }
```

### Cambiar la clave de un usuario

```powershell
# 1. Generar hash
dotnet run --project backend\CampMedTools -- --hash "CampMedDemo2026!"

# 2. Aplicar en la BD
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -h localhost -U postgres -d bdcampmed `
  -c "UPDATE USUARIO SET PASSWORD_HASH = '<hash>' WHERE LOGIN = 'admin';"
```

---

## 14. Punto de reanudación

1. Aprobar este plan.
2. Cambiar al agente de implementación (Build) para:
   - instalar `cloudflared`,
   - crear `start-demo.ps1` fuera del repo,
   - ejecutar la Etapa 1 y la Etapa 2,
   - dejar la URL de prueba documentada en este mismo archivo (sección 15).
3. Si la primera intención funciona, decidir si se formaliza como `docs/despliegue-preproduccion.md` cuando se elija la tecnología de producción.

---

## 15. Resultado de la primera prueba

_(Pendiente de ejecutar. Anotar aquí: fecha, URL probada, latencia, navegador, incidencias y ajustes.)_

- Fecha:
- URL:
- PC / red:
- Navegador remoto:
- Tiempo de primera carga:
- Incidencias:
- Ajustes para la próxima:
