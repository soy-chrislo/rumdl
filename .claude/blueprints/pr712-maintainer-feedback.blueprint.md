---
artifact: blueprint
generated: 2026-07-11T00:00:00Z
source_prd: .claude/prds/pr712-maintainer-feedback.prd.md
source_generated: 2026-07-11T00:00:00Z
chosen_approach: Seguir el patrón ya establecido en scripts/rumdl-action.sh (gate de herramienta con `command -v`, chequeo explícito de exit code) — sin nueva abstracción.
delivery_track: light
design_lane: express
linked_adr: none
status: approved
---

**Mode: feature — Lane: express** (las 4 condiciones se cumplen: un único enfoque natural ya fijado
por el maintainer, sin nueva abstracción/dependencia/runtime, no toca el contrato público de la
Action — inputs/outputs sin cambios—, alcance acotado a un archivo (+ uno en otra rama, gateado),
`delivery_track` obviamente `light`.)

## Feature: Implementar feedback CHANGES_REQUESTED de rvben en PR #712

### Linked ADR
none — la feature se queda dentro de la arquitectura actual (script bash, sin decisión de
tecnología nueva).

### Reuse scan
> **Nota de rama**: estas citas son válidas en `fix/action-install-without-pip` (la rama del PR
> #712). El worktree local está actualmente en `fix/restore-malformed-fixture`, donde
> `scripts/rumdl-action.sh` es la versión vieja pip-only (sin `sha256_of` ni `try_install_binary`).
> Ver Build sequence, paso 0.

- Ya existe: `scripts/rumdl-action.sh:64-70` (`sha256_of`, rama del PR) — patrón exacto de
  "chequear disponibilidad de herramienta con `command -v`, hacer fallback si no está" ya usado en
  el mismo script (ahí es `sha256sum` → `shasum -a 256`; acá es `curl` → pip). Referencia directa
  para el punto 1.
- Ya existe: `scripts/rumdl-action.sh:85-90` (rama del PR, patrón `curl_status=0; ... ||
  curl_status=$?; if [ "$curl_status" -ne 0 ]; then echo "::error::..."; exit 1; fi`) — el patrón
  ya establecido de chequeo explícito de exit code en este mismo script, referencia para el
  punto 2.
- Ya existe: `scripts/rumdl-action.sh:164-172` (rama del PR, sitio de llamada de
  `try_install_binary`: `echo "No prebuilt rumdl binary for RUNNER_OS=... — falling back to pip";
  install_via_pip`) — análogo más cercano aún al cambio del punto 1: mismo call site, mismo estilo
  de mensaje (echo plano, no `::warning::`). Resuelve además la Open Question del PRD sobre el
  wording del fallback: el precedente del propio script ya es echo plano, no hace falta preguntar.
- Gap: no hay gate de `curl` antes de `try_install_binary`; `mktemp -d`/`cd` sin chequeo explícito;
  el hash esperado no normaliza CRLF; no hay fallback de extracción a PowerShell.

### Chosen approach
Sigue el patrón ya establecido en el mismo archivo: gate de herramienta con `command -v` (igual
que `sha256_of` hace con `sha256sum`), y chequeo explícito de exit code (igual que el patrón
`curl_status`) para los dos puntos donde `set -e` no protege (confirmado por repro empírico en la
fase de discovery). Sin nueva abstracción ni dependencia.

### Design decisions
- Gate de curl fuera de `try_install_binary`, antes de invocarla, como **check anidado** (`if [ -n
  "$target_info" ]; then if command -v curl >/dev/null 2>&1; then ...; else echo "curl not found —
  falling back to pip"; install_via_pip; fi; else echo "No prebuilt rumdl binary for
  RUNNER_OS=..."; install_via_pip; fi`) — **no** un `&&` compuesto en la condición externa, para no
  colapsar los dos mensajes existentes/nuevos ("no prebuilt binary" vs. "curl not found") en uno
  solo → applies to: `scripts/rumdl-action.sh` (bloque final, `if [ -n "$target_info" ]; then ...
  fi`)
- Mensaje de fallback informativo y neutro (`echo "curl not found — falling back to pip"`), mismo
  estilo (echo plano) que el precedente ya citado en Reuse scan (líneas 164-172) — no
  `::warning::` ruidoso. Esto resuelve la Open Question del PRD con evidencia del propio repo, no
  hace falta preguntarle al usuario → applies to: `scripts/rumdl-action.sh`
- `|| exit 1` explícito solo en las 2 líneas identificadas por el repro empírico (`mktemp -d`,
  `cd`), sin tocar el resto del subshell — ya protegido por chequeos explícitos existentes o por
  ser la última instrucción del subshell → applies to: `scripts/rumdl-action.sh` (dentro de
  `try_install_binary`)
- `tr -d '\r'` insertado en el pipeline existente de `expected_hash`, sin alterar el resto de la
  línea → applies to: `scripts/rumdl-action.sh` (línea del `awk` sobre `$checksum_asset`)
- Fallback a `Expand-Archive`: chequear existencia de `/c/Windows/System32/tar.exe` antes de
  usarlo; si no existe, invocar PowerShell (`powershell -Command "Expand-Archive -Path ... -
  DestinationPath ..."`) → applies to: `scripts/rumdl-action.sh` (bloque de extracción del `.zip`)
- Paths filters del fixture-fix: agregar dos entradas al array `paths:` ya existente en el YAML,
  mismo patrón que las entradas actuales (globs) → applies to:
  `.github/workflows/test-action.yml` (rama `fix/restore-malformed-fixture`, PR separado)

### Files to create
(ninguno)

### Files to modify
| File | Change | Why | Validation | Priority |
|------|--------|-----|------------|----------|
| `scripts/rumdl-action.sh` | Gate `command -v curl` antes de intentar instalación por binario, fallback informativo a `install_via_pip` | Bloqueante de la review — evita fallo duro en `python:3.12-slim` y contenedores solo-Python | Simular `PATH` sin `curl` localmente; confirmar que cae a `install_via_pip` sin abortar | P0 |
| `scripts/rumdl-action.sh` | `\|\| exit 1` en `workdir=$(mktemp -d)` y `cd "$workdir"` | Should-fix — evita que un fallo silencioso deje archivos sueltos en el repo del usuario (confirmado por repro empírico) | Re-adaptar el repro empírico de discovery al script real; confirmar que un `cd` fallido aborta con `exit 1` | P1 |
| `scripts/rumdl-action.sh` | `tr -d '\r'` en el pipeline de `expected_hash` | Nit pedido explícitamente — evita falso negativo de checksum en BSD awk/macOS | Descargar el `.zip.sha256` real de un release publicado y confirmar que el hash calculado coincide tras normalizar | P2 |
| `scripts/rumdl-action.sh` | Fallback a `Expand-Archive` si `tar.exe` no existe en la ruta hardcodeada | Nit pedido explícitamente — cubre self-hosted runners sin `C:` como drive de sistema | No se puede ejecutar en este entorno (Linux). En `windows-latest` hosted `tar.exe` sí existe, así que el matrix del fork valida que el camino normal no se rompe pero **no ejercita la rama `Expand-Archive`** — revisión manual de sintaxis PowerShell, dejarlo explícito en el PR como no probado en Windows real | P2 |
| `.github/workflows/test-action.yml` (rama `fix/restore-malformed-fixture`) | Agregar `__tests__/fixtures/**` y `.github/workflows/test-action.yml` a `paths:` | Pedido explícito del maintainer — evita que un fixture roto pase desapercibido en CI a futuro | Confirmar sintaxis YAML válida; **no se pushea ni se abre el PR sin confirmación explícita del usuario** | P1 (bloqueado por confirmación) |

### Data flow
N/A — script bash lineal invocado por GitHub Actions, sin servicios intermedios. El único flujo
relevante es de control: `resolve_target` → gate de `curl` → `try_install_binary` (o fallback
directo a `install_via_pip`) → uso de `$rumdl_cmd` para lintear.

### Build sequence
0. **Checkout de `fix/action-install-without-pip`** — el worktree local está en
   `fix/restore-malformed-fixture`; editar ahí aplicaría los cambios sobre la versión vieja
   pip-only del script, no sobre el código real que revisó rvben.
1. Gate de `curl` (P0, bloqueante) — validar localmente simulando ausencia de `curl`.
2. `\|\| exit 1` en `mktemp`/`cd` (P1) — validar con el repro empírico adaptado al script real.
3. `tr -d '\r'` en checksum (P2) — validar contra un asset real de un release publicado (smoke
   test del pipeline, no reproduce el bug de CRLF en sí — eso solo ocurre en BSD awk/macOS).
4. Fallback `Expand-Archive` (P2) — validar sintaxis; dejar nota explícita de no-testeado-en-
   Windows-real (ni siquiera el matrix `windows-latest` del fork la ejercita, ver Risks).
5. Push a la rama del PR + validar Actions del fork (milestone 2 del PRD, sigue el flujo de
   `feedback_fork_ci_validation`).
6. Paths filters en la otra rama (P1, bloqueado por confirmación del usuario) — trabajo separado,
   no se ejecuta como parte de este mismo build sin luz verde explícita.

### Risks & open questions
- Risk: el fallback a PowerShell no se puede probar en este entorno Linux. Mitigación: sintaxis
  revisada manualmente, se deja explícito en el PR que no se testeó en Windows real, y el matrix
  de Actions del fork (que sí corre `windows-latest`) lo valida antes del push definitivo.
- Assumption: el mensaje de log del fallback de `curl` será informativo (no completamente
  silencioso) — sigue pendiente como Open Question del PRD, a confirmar con el usuario.
- Defer to planner: ninguno — todo cabe en una sola sesión de trabajo (`light` track), ya
  establecido en discovery y PRD.
