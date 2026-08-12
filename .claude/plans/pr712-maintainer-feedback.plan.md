---
artifact: plan
generated: 2026-07-11T00:00:00Z
source_blueprint: .claude/blueprints/pr712-maintainer-feedback.blueprint.md
source_generated: 2026-07-11T00:00:00Z
delivery_track: light
---

# Plan: Implementar feedback CHANGES_REQUESTED de rvben en PR #712

**Source blueprint**: `.claude/blueprints/pr712-maintainer-feedback.blueprint.md`
**Complexity**: Small

## Summary
Cuatro cambios puntuales en `scripts/rumdl-action.sh` (rama `fix/action-install-without-pip`,
PR #712) siguiendo el estilo ya establecido en el propio archivo, más un quinto cambio en
`.github/workflows/test-action.yml` en otra rama, bloqueado hasta confirmación explícita del
usuario. Working tree ya está en `fix/action-install-without-pip` (checkout hecho en la fase
`/architect`).

## Patterns to Mirror
| Category | Source | Pattern |
|---|---|---|
| Naming | `scripts/rumdl-action.sh:79-80` | snake_case para vars/funciones locales (`try_install_binary`, `curl_status`, `http_code`) |
| Errors | `scripts/rumdl-action.sh:85-90` | capturar exit code explícito (`var=0; cmd \|\| var=$?`), chequear con `if [ "$var" -ne 0 ]`, `echo "::error::..."` + `exit 1` en fallo duro |
| Logging | `scripts/rumdl-action.sh:170` | mensaje informativo plano (`echo "..."`) para fallback esperado, sin `::warning::`; `::error::` reservado a fallo duro |
| Tool availability | `scripts/rumdl-action.sh:65` | `command -v <tool> >/dev/null 2>&1` como gate antes de usar una herramienta opcional |
| Tests | `.github/workflows/test-action.yml:217-236` | job con `continue-on-error: true` + step separado que verifica `steps.<id>.outcome` — patrón para casos "se espera que falle/degrade" |

## Files to Change
| File | Action | Why |
|---|---|---|
| `scripts/rumdl-action.sh` | UPDATE | 4 cambios de la review de rvben (bloqueante + should-fix + 2 nits) |
| `.github/workflows/test-action.yml` (rama `fix/restore-malformed-fixture`) | UPDATE | Paths filters — **bloqueado, no ejecutar sin confirmación separada del usuario** |

## Tasks

### Task 1: Gate de `curl` con fallback a pip (P0, bloqueante)
- **Action**: en el bloque de invocación (líneas 162-172), anidar un chequeo `command -v curl
  >/dev/null 2>&1` dentro de la rama `if [ -n "$target_info" ]`, para no colapsar el mensaje
  existente de "no prebuilt binary" con el nuevo de "no curl":
  ```bash
  echo
  target_info=$(resolve_target)
  if [ -n "$target_info" ]; then
      if command -v curl >/dev/null 2>&1; then
          read -r target ext <<<"$target_info"
          if ! try_install_binary "$target" "$ext"; then
              install_via_pip
          fi
      else
          echo "curl not found — falling back to pip"
          install_via_pip
      fi
  else
      echo "No prebuilt rumdl binary for RUNNER_OS='${RUNNER_OS:-}' RUNNER_ARCH='${RUNNER_ARCH:-}' — falling back to pip"
      install_via_pip
  fi
  ```
- **Mirror**: `command -v` gate de `scripts/rumdl-action.sh:65` (mismo idioma que `sha256_of`);
  wording de mensaje plano igual a `scripts/rumdl-action.sh:170`.
- **From blueprint**: Design decision "Gate de curl fuera de `try_install_binary`... check
  anidado" + "Mensaje de fallback informativo y neutro".
- **Validate**: simular localmente `PATH` sin `curl` (`PATH=$(dirname "$(command -v pip)") bash
  scripts/rumdl-action.sh` con `curl` removido del `PATH` vía un dir filtrado) y confirmar que cae
  a `install_via_pip` sin abortar. Ejemplo de repro: `PATH="$(echo "$PATH" | tr ':' '\n' | grep -v
  curl-containing-dir | paste -sd:)" GHA_RUMDL_PATH=__tests__/fixtures/clean/
  GHA_RUMDL_REPORT_TYPE=logs bash scripts/rumdl-action.sh` — más simple: copiar el script a un
  contenedor `python:3.12-slim` real (sin `curl`) y correrlo ahí, matching el escenario que motivó
  el bloqueante.

### Task 2: `|| exit 1` explícito en `mktemp -d` y `cd`
- **Action**: línea 104: `workdir=$(mktemp -d)` → `workdir=$(mktemp -d) || exit 1`. Línea 108:
  `cd "$workdir"` → `cd "$workdir" || exit 1`.
- **Mirror**: chequeo explícito de exit code, mismo idioma que `scripts/rumdl-action.sh:85-90`.
- **From blueprint**: Design decision "`\|\| exit 1` explícito solo en las 2 líneas identificadas
  por el repro empírico".
- **Validate**: repro empírico ya hecho en la fase de discovery (forzar `mktemp -d` a apuntar a un
  dir inexistente y confirmar que el script aborta con `exit 1` en vez de continuar en el
  directorio original). Re-ejecutar ese repro contra el script real ya modificado.

### Task 3: Normalizar CRLF en el hash esperado
- **Action**: línea 131: `expected_hash=$(awk '{print $1}' "$checksum_asset" | tr '[:upper:]'
  '[:lower:]')` → `expected_hash=$(awk '{print $1}' "$checksum_asset" | tr -d '\r' | tr
  '[:upper:]' '[:lower:]')`.
- **Mirror**: modificación mínima de un pipeline existente, sin cambiar su forma general
  (`scripts/rumdl-action.sh:131-132`).
- **From blueprint**: Design decision "`tr -d '\r'` insertado en el pipeline existente".
- **Validate**: descargar un `.zip.sha256` real de un release publicado (smoke test del pipeline —
  no reproduce el bug de CRLF en sí, que solo ocurre en BSD awk/macOS) y confirmar que
  `expected_hash` sigue calculándose correctamente tras el cambio.

### Task 4: Fallback a `Expand-Archive` si no existe `tar.exe`
- **Action**: líneas 138-143, dentro del bloque `if [ "$ext" = "zip" ]`:
  ```bash
  echo "Extracting $asset"
  if [ "$ext" = "zip" ]; then
      if [ -x "/c/Windows/System32/tar.exe" ]; then
          /c/Windows/System32/tar.exe -xf "$asset"
      else
          powershell -NoProfile -Command "Expand-Archive -Path '$asset' -DestinationPath '.' -Force"
      fi
  else
      tar -xzf "$asset"
  fi
  ```
- **Mirror**: chequeo de existencia antes de usar una ruta hardcodeada, mismo espíritu que el gate
  de `command -v` de `scripts/rumdl-action.sh:65`.
- **From blueprint**: Design decision "Fallback a `Expand-Archive`: chequear existencia... si no
  existe, invocar PowerShell".
- **Validate**: no ejecutable en este entorno (Linux). Revisión manual de sintaxis PowerShell.
  **No se ejercita en CI** — en `windows-latest` hosted `tar.exe` sí existe, así que el matrix del
  fork valida el camino normal pero nunca la rama `Expand-Archive`. Dejar esta limitación explícita
  en la descripción del push/PR.

### Task 5: Paths filters en `test-action.yml` — **BLOQUEADO, no ejecutar sin confirmación separada**
- **Action**: en la rama `fix/restore-malformed-fixture`, líneas 5 y 7 de
  `.github/workflows/test-action.yml` (`paths: ["action.yml", "scripts/rumdl-action.sh"]`) →
  agregar `"__tests__/fixtures/**"` y `".github/workflows/test-action.yml"` a ambos arrays.
- **Mirror**: mismo formato de array YAML ya usado en esas líneas.
- **From blueprint**: Design decision "Paths filters del fixture-fix: agregar dos entradas al
  array `paths:` ya existente".
- **Validate**: sintaxis YAML válida (`yamllint` o parseo simple). **No pushear ni abrir PR sin
  confirmación explícita del usuario** — esto es un PR separado, en otra rama, con su propio gate.

## Validation
```bash
# Sintaxis del script modificado
bash -n scripts/rumdl-action.sh

# Shellcheck (no instalado localmente, vía Docker) — rvben piensa en estos términos: el pedido de
# || exit 1 en cd/mktemp es literalmente un finding estilo SC2164. No lo detecta bash -n.
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable scripts/rumdl-action.sh
# Si aparecen warnings preexistentes (no introducidos por este diff), NO arreglarlos en este PR
# (mismo principio de scope que el resto de la cadena) — solo identificarlos para no confundir
# "preexistente" con "introducido" al revisar el diff final.

# Repro del gate de curl — ANTES del fix (debe reproducir el bug real: "::error::Could not reach
# GitHub... (curl exit 127)" en la línea 86-89) y DESPUÉS del fix (debe caer a install_via_pip sin
# abortar). Correr solo el "después" no demuestra que se reprodujo el escenario del bloqueante.
docker run --rm -v "$PWD":/repo -w /repo python:3.12-slim bash -c \
  'GHA_RUMDL_PATH=__tests__/fixtures/clean/ GHA_RUMDL_REPORT_TYPE=logs bash scripts/rumdl-action.sh'

# Repro del fallo de cd/mktemp (adaptado del repro empírico de discovery)
# — forzar mktemp -d a un path que se borra antes del cd, confirmar exit 1

# Stage explícito — .claude/ está untracked en el working tree; NO usar git add -A / git add .
# para no meter los artefactos de esta cadena en el commit del PR.
git add scripts/rumdl-action.sh

# Push + validar en Actions del fork antes de notificar (sigue feedback_fork_ci_validation)
git push origin fix/action-install-without-pip
gh run list --repo soy-chrislo/rumdl --branch fix/action-install-without-pip --limit 5
```

## Risks
| Risk | Likelihood | Mitigation |
|---|---|---|
| El repro de Task 1 en `python:3.12-slim` requiere Docker local — si no está disponible, degradar a simular `PATH` sin `curl` (menos fiel al escenario real) | Media | Documentar cuál de los dos métodos se usó al reportar la validación |
| La rama `Expand-Archive` de Task 4 no se ejercita en ningún CI existente (ni local ni fork) | Alta | Aceptado explícitamente — revisión manual de sintaxis, nota en el PR, matrix `windows-latest` solo cubre el camino `tar.exe` |
| Bundlear sin querer el cambio de Task 5 en el push de #712 | Baja | Verificar `git branch --show-current` antes de cada commit; Task 5 vive en otra rama y no se toca hasta confirmación separada |

## Acceptance
- [ ] Tasks 1-4 completas y validadas localmente
- [ ] `bash -n scripts/rumdl-action.sh` sin errores de sintaxis
- [ ] Push a `fix/action-install-without-pip` con Actions del fork en verde (excluyendo los 4 jobs de fixture, ajenos a este PR)
- [ ] Task 5 preparada pero NO pusheada/abierta sin confirmación explícita separada del usuario
- [ ] Cada Design decision del blueprint tiene una task que la cubre (verificado: 5/5)
