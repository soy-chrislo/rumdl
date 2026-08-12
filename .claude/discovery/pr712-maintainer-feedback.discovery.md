---
artifact: discovery
generated: 2026-07-11T00:00:00Z
verdict: go
---

# Discovery: Implementar feedback CHANGES_REQUESTED de rvben en PR #712

## The bet
Implementar los 4 cambios que rvben (maintainer, owner de rvben/rumdl) pidió en su review
CHANGES_REQUESTED del PR #712 (`fix/action-install-without-pip`), como condición para que
mergee el PR, y abrir por separado el PR de paths-filters de CI que también pidió.

## Load-bearing assumptions
- La review es la fuente de requisitos real, no una idea a validar — Evidence: concreta, vía
  `gh pr view 712 --repo rvben/rumdl --comments` y `gh pr view 712 --repo rvben/rumdl --json reviews`
  (único review, `CHANGES_REQUESTED`, autor `rvben` (owner), `submittedAt: 2026-07-11T20:39:48Z`,
  cero comentarios posteriores).
- Las 3 afirmaciones técnicas del maintainer son correctas — Evidence: concreta, verificada de
  forma independiente por 3 agentes de investigación con fuentes primarias:
  - `set -e` suprimido dentro de una función invocada en `if !` (incl. comandos intermedios, no
    solo el último): GNU Bash Reference Manual ("The Set Builtin") + BashFAQ/105 (Greg's Wiki).
  - `python:3.12-slim` trae `pip` (vía `ensurepip`) pero no `curl`: Dockerfile fuente en
    `docker-library/python` (3.12/slim-bookworm).
  - CRLF sobrevive en awk sobre macOS/BSD pero no en Git Bash/MSYS2 (por la capa de traducción
    de modo texto de MSYS2, no por ser "otro awk"): GNU gawk manual (`PC-Using`, `BINMODE`) +
    MSYS2-packages#2315.
- El fix del punto 2 (should-fix) es exactamente lo que pidió el maintainer, sin alcance oculto —
  Evidence: concreta, repro empírico en bash (`set -e` se suprime para TODO comando intermedio
  dentro del subshell, confirmado con un `false` que no aborta) + lectura completa de
  `scripts/rumdl-action.sh` en la rama del PR: cada comando riesgoso dentro del subshell (`curl`,
  verificación de checksum) ya tiene chequeo explícito de `http_code`/hash con `exit 1`/`exit 2`;
  la extracción con `tar` es la última instrucción del subshell así que su código de salida se
  propaga solo. Los únicos dos puntos sin chequeo explícito son exactamente `mktemp -d` y `cd`,
  tal como pidió el maintainer — no hace falta auditar el resto del subshell.
- El trabajo de CI (paths filters) es una tarea separada, ya secuenciada por el maintainer —
  Evidence: concreta, texto literal de la review: "Please open your `fix/restore-malformed-fixture`
  branch as its own PR... add `__tests__/fixtures/**` and `.github/workflows/test-action.yml`
  itself to the `paths` filters... I'll merge that first so this PR can rebase and go fully green."

## Kill criteria tested
| Criterion | Finding | Evidence? |
|---|---|---|
| Demand | Máxima señal posible en OSS: el propio maintainer/owner del repo lo exige como condición de merge de un PR ya abierto. No hay debate de demanda que tenga sentido forzar aquí. | concreta |
| Feasibility | Confirmada en las 3 dimensiones técnicas (bash, Docker, awk) + a nivel de código real del repo. Cambio acotado a un solo script bash (~4 fixes puntuales) + un workflow YAML en otra rama. | concreta |
| Legal/regulatory | N/A — mismo repo, mismo PR, sin cambio de licencia ni de dependencias externas. | concreta |
| Fit | Encaja directamente con el objetivo ya en curso (mergear #712); el trabajo de CI encaja con la memoria ya guardada sobre no bundlear fixes no relacionados (`feedback_pr_scope_and_evidence`). | concreta |

## Verdict: go
Las 4 correcciones son requisito explícito y ya confirmado de merge por el maintainer, con las
3 afirmaciones técnicas subyacentes verificadas de forma independiente (no se acepta la review
como verdad per se — se contrastó con fuentes primarias + repro empírico) y con el alcance del
fix ya acotado a nivel de código real (no hay ambigüedad de "hasta dónde llega el subshell fix").
No aplica un debate artificial de demanda: no existe evidencia de demanda más fuerte en OSS que
una condición de merge explícita del propio maintainer. Próximo paso: `/plan-prd` con esta
discovery como input, pero escalado al tamaño real de la tarea (un PRD de párrafos, no páginas) —
el contenido con valor real está en el *secuenciamiento* (fixture-fix PR primero, luego rebase de
#712) más los 4 ítems de aceptación, no en re-derivar requisitos que el maintainer ya fijó.

---
*Status: discovery only. Requirements pending via /plan-prd (verdict == go).*
