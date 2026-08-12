---
artifact: prd
generated: 2026-07-11T00:00:00Z
source_discovery: .claude/discovery/pr712-maintainer-feedback.discovery.md
source_generated: 2026-07-11T00:00:00Z
status: draft
---

# Desbloquear merge de PR #712 (rumdl-action.sh) implementando feedback de rvben

## Problem
rvben (maintainer/owner de rvben/rumdl) dejó una review `CHANGES_REQUESTED` en el PR #712
(`fix/action-install-without-pip`) el 2026-07-11T20:39:48Z. El PR no puede mergear hasta resolver
4 puntos puntuales sobre `scripts/rumdl-action.sh`. Además de bloquear el merge, el código actual
tiene comportamiento riesgoso latente: falla duro (en vez de degradar a pip) en contenedores
solo-Python, puede escribir archivos descargados dentro del propio repo del usuario si `cd` falla
silenciosamente, y puede dar un falso negativo de checksum en runners Windows sin Git Bash.

## Evidence
- Review real verificada vía `gh pr view 712 --repo rvben/rumdl --comments` — único review,
  `CHANGES_REQUESTED`, sin comentarios posteriores al timestamp dado (`gh pr view --json reviews`).
- Las 3 afirmaciones técnicas subyacentes (supresión de `set -e` en función invocada en `if !`,
  ausencia de `curl` en `python:3.12-slim`, CRLF sobreviviendo en awk de macOS/BSD) verificadas
  independientemente con fuentes primarias (GNU Bash manual, BashFAQ/105, Dockerfile de
  docker-library/python, GNU gawk manual + MSYS2-packages#2315).
- Repro empírico en bash + lectura completa de `scripts/rumdl-action.sh` en la rama del PR:
  confirma que el fix del punto 2 es exactamente las 2 líneas que pidió el maintainer
  (`mktemp -d` y `cd`), sin necesidad de auditar el resto del subshell.

## Users
- **Primary**: rvben, como maintainer/gatekeeper de merge de rvben/rumdl.
- **Secondary**: consumidores de la Action en runners self-hosted (contenedores solo-Python,
  Windows sin C: como drive de sistema, macOS ejecutando el paso de verificación de checksum).
- **Not for**: usuarios de la Action en runners hosted estándar de GitHub — ya funcionan hoy y no
  se ven afectados por ninguno de los 4 fixes.

## Hypothesis
Creemos que implementar los 4 cambios pedidos por rvben (gate de `curl` con fallback a pip,
`|| exit 1` explícito en `mktemp -d`/`cd`, normalización CRLF del checksum esperado, fallback a
`Expand-Archive` si `tar.exe` no está en `C:`) desbloqueará el merge de PR #712.
Lo sabremos cuando rvben apruebe el PR o el CI del fork quede verde sin nuevos comentarios de
review tras el push.

## Success Metrics
| Metric | Target | How measured |
|---|---|---|
| Review de rvben | Aprobado (o sin nuevos `CHANGES_REQUESTED`) | `gh pr view 712 --repo rvben/rumdl --json reviews` |
| CI del fork tras el push | Verde en los jobs que dependen de este script (excluyendo los 4 fixture jobs pre-existentes, ajenos a este PR, hasta que se complete el rebase de milestone 4) | Actions del fork `soy-chrislo/rumdl` |
| CI de #712 tras el rebase (milestone 4) | Completamente verde, incluyendo los 4 jobs de fixture | Checks del PR en GitHub |
| Validación local | `try_install_binary` degrada a pip sin abortar cuando no hay `curl` | Repro local simulando ausencia de `curl` (`PATH` sin `curl`) |

## Scope
**MVP** — Los 4 ítems de la review, todos in-scope (los "nits" fueron pedidos explícitamente por
el maintainer, no se tratan como opcionales):
1. Gate `command -v curl` antes de `try_install_binary`; si no existe, fallback silencioso (con
   una línea informativa, ver Open Questions) a `install_via_pip`.
2. `|| exit 1` explícito en `workdir=$(mktemp -d)` y `cd "$workdir"` dentro de `try_install_binary`.
3. `tr -d '\r'` en el pipeline del hash esperado (`awk '{print $1}' "$checksum_asset"`) antes de
   comparar con `actual_hash`.
4. Fallback a `Expand-Archive` (PowerShell) si `/c/Windows/System32/tar.exe` no existe, antes de
   intentar extraer el `.zip`.

**Out of scope**
- Fix de `fix/restore-malformed-fixture` (paths filters `__tests__/fixtures/**` y
  `.github/workflows/test-action.yml` en `test-action.yml`) — PR separado, por pedido explícito
  del maintainer (lo mergea primero para que #712 pueda rebasar y quedar verde) y por la regla ya
  guardada de no bundlear fixes no relacionados en un mismo PR.
- Cualquier refactor más amplio del subshell de `try_install_binary` — el repro empírico confirmó
  que no hace falta; el resto de los comandos riesgosos ya tienen chequeo explícito o son la
  última instrucción del subshell.

## Delivery Milestones
| # | Milestone | Outcome | Status | Blueprint |
|---|---|---|---|---|
| 1 | Implementar los 4 fixes en `scripts/rumdl-action.sh` + validación local (sin pushear) | Código listo, probado localmente (curl ausente → fallback a pip, mktemp/cd con `exit 1`, CRLF normalizado, fallback `Expand-Archive`) | pending | — |
| 2 | Pushear a la rama del PR y validar Actions del fork en verde | rvben puede volver a revisar sin el bloqueante ni el should-fix pendientes, con confianza previa de que no rompe CI (sigue el flujo de `feedback_fork_ci_validation`) | pending | — |
| 3 | Preparar (no pushear sin confirmación) el fix de paths-filters en `fix/restore-malformed-fixture` | Listo para abrir como PR separado en cuanto el usuario confirme — rvben pidió explícitamente abrirlo, no es opcional, solo está gateado por la confirmación del usuario | pending | — |
| 4 | Rebasar #712 sobre `main` una vez que rvben mergee el fix del fixture | #712 queda completamente verde en CI (incluyendo los 4 jobs de fixture que hoy fallan por una causa ajena a este PR) | pending | — |

## Open Questions
- [ ] Wording exacto del fallback a pip cuando no hay `curl`: ¿sin log, o con una línea
  informativa (`echo "curl not found, falling back to pip"`)? Recomendación del advisor: una
  línea informativa es estándar y difícilmente objetable.

## Assumptions (no open questions, pero explícitas)
- El push de los 4 fixes a #712 (milestone 2) es independiente de si se abre o no el PR del
  fixture (milestone 3): son ramas y archivos distintos, y rvben mismo distingue ambos como
  "must fix before merging" vs. "fixture breakage" con su propio plan.

## Risks
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Nueva vuelta de review si el wording de "silencioso" no coincide con lo que rvben esperaba | Baja | Baja (una vuelta más de review, no bloquea el proyecto) | Usar una línea informativa neutra, no un `::warning::` ruidoso |
| Bundlear sin querer el fix del fixture en el push a #712 | Baja | Media (mezclaría dos PRs, contradice el pedido explícito del maintainer) | Verificar rama activa antes de cada `git add`/commit; el fixture-fix vive en otra rama |
| CI del fork deshabilitado por defecto (gotcha ya conocido) retrasa la validación pre-push | Media | Baja (solo demora, no bloquea) | Seguir el flujo ya documentado en memoria (`feedback_fork_ci_validation`) |

---
*Status: DRAFT — requirements only. Design pending via /architect.*
