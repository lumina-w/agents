# agents
<!-- dev-agent:start -->
## Dev agent (`dev-agent.yml`)

El dev agent de Lúmina W vive **una sola vez** en este repo. Cada repo de código tiene solo un `claude.yml` corto que lo llama con un tag fijo (`@v4`) y le pasa lo que es propio del repo. No hay copias de la lógica en los repos.

### Qué hace y qué no

| Hace | No hace |
|---|---|
| Implementa un issue y abre un PR contra `dev` | Mergear PRs |
| Itera sobre su PR cuando alguien comenta `@claude` | Pushear a `dev`, `stg` o `main`, ni force push |
| Corrige su PR cuando el CI falla (máximo 2 rondas) | Promover `dev -> stg -> main` |
| Actualiza el checkpoint del repo en el mismo PR | Leer `.env`, instalar dependencias por su cuenta, levantar servidores |

Su trabajo termina en el PR. El resto del flujo es de los workflows del repo:

```
issue + label ──> dev agent ──> PR a dev ──> CI ──(verde)──> automerge-dev ──> merge squash a dev
                                   ^           │                                    │
                                   │        (rojo)                                  v
                                   └── modo fix (máx. 2) <─┘         delete-merged-branches (cada 12 h)
                                                                                    │
                                              promote-dev-to-stg (10:00 UTC) ──> stg ──(persona)──> main
```

### Cómo se dispara

| Evento | Modo | Modelo por defecto | Turnos |
|---|---|---|---|
| Label `dev-ft-ready` en un issue | implement | `opus` | 80 |
| Label `dev-ft-small` en un issue (tareas pequeñas) | implement | `sonnet` | 40 |
| Comentario con `@claude` en un PR | iterate | `sonnet` | 40 |
| El workflow `CI` falla en un PR con label `dev-agent` | fix | `opus` | 80 |

Reglas del disparo:

- Solo cuenta si quien pone el label o comenta tiene permiso `write`, `maintain` o `admin` en el repo.
- Los eventos de bots se ignoran (evita bucles).
- Una sola ejecución a la vez por issue o PR.
- Modo fix: solo en PRs que abrió el agente (label `dev-agent`), marca cada ronda con `agent-fix-1`, `agent-fix-2`; al llegar al límite comenta en el PR y se detiene. Para seguir, una persona comenta `@claude` con instrucciones.
- Modo fix no existe en repos sin workflow `CI` (hoy `luminaw-page`).

### Capas

| # | Capa | Qué hace |
|---|---|---|
| 1 | Disparo | Verifica permisos, ignora bots, resuelve el modo y limita las rondas de fix |
| 2 | Stack | Configura Python, npm o pnpm con las versiones del repo e instala dependencias antes de que el agente empiece |
| 3 | Entorno | Exporta variables de configuración locales del repo (nunca secrets) |
| 4 | Contexto | Lee `CLAUDE.md`, `AGENTS.md`, los docs extra del repo, sus skills en `.claude/skills/` y, en solo lectura, el repo de docs del proyecto (`<proyecto>-docs`) |
| 5 | Comandos | Solo puede correr los comandos de "tarea terminada" del `CLAUDE.md` del repo más unos pocos extra (formatear, un test puntual). Primero el más acotado; la lista completa una vez antes de cada push |
| 6 | Bloqueos | Deniega merge, force push, push a `dev`/`stg`/`main`, `reset --hard` y lectura de `.env` |
| 7 | Retroalimentación | Espera los checks del PR; si el CI falla después, el modo fix lo retoma |
| 8 | Checkpoint | El agente escribe el checkpoint completo en un archivo temporal y el workflow lo commitea como `.claude/CHECKPOINT.md` en la rama del PR (`docs(docs): update checkpoint after #<n>`, push con `PROMOTE_TOKEN` para que los checks vuelvan a correr). Claude no puede escribir en `.claude/`: es ruta protegida para sus herramientas. Si el repo aún lo tiene en la raíz, lo mueve con `git mv`. Es el mismo archivo que leen el hook `SessionStart`, el comando `/checkpoint` y el sync de docs |
| 9 | Texto del PR y ready | Reemplaza los guiones largos del título y el cuerpo del PR por ", ". El agente abre el PR como draft y el workflow lo marca listo al final (`gh pr ready` con `PROMOTE_TOKEN`), después del checkpoint: `automerge-dev` ignora los drafts y arma el auto-merge con el evento `ready_for_review`, así que no puede mergear antes de que llegue el checkpoint. Si el agente falla, el PR queda en draft |

La protección de ramas de GitHub sigue siendo la barrera real contra pushes a ramas protegidas; los bloqueos del agente son una segunda capa.

### Caller mínimo (`.github/workflows/claude.yml` en cada repo)

```yaml
name: Claude Code
on:
  issues:
    types: [labeled]
  issue_comment:
    types: [created]
  workflow_run:
    workflows: [CI]
    types: [completed]
jobs:
  claude:
    if: >-
      (github.event_name == 'issues' && (github.event.label.name == 'dev-ft-ready' || github.event.label.name == 'dev-ft-small')) ||
      (github.event_name == 'issue_comment' && github.event.issue.pull_request && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'workflow_run' && github.event.workflow_run.event == 'pull_request' && github.event.workflow_run.conclusion == 'failure')
    uses: lumina-w/agents/.github/workflows/dev-agent.yml@v4
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    secrets: inherit
    with:
      docs_repo: terracore-docs
      toolchain: npm
      node_version: "20.x"
      install_command: npm ci
      verify_commands: |
        npm run lint
        npm run test
        npm run build
```

### Inputs

| Input | Obligatorio | Default | Uso |
|---|---|---|---|
| `toolchain` | sí | | `python`, `npm`, `pnpm` o `none` |
| `python_version` | | | Versión de Python del repo |
| `python_cache_path` | | | Archivo de requirements para la caché de pip |
| `node_version` / `node_version_file` | | | Versión de Node, o archivo como `.nvmrc` |
| `install_command` | | | Instala dependencias antes del agente (`npm ci`, `pnpm install --frozen-lockfile`, `pip install -r ...`) |
| `env_vars` | | | `CLAVE=valor` por línea para checks locales. Nunca secrets |
| `verify_commands` | sí | | Comandos de "tarea terminada" del `CLAUDE.md`, uno por línea. Son los únicos checks permitidos |
| `extra_allowed_tools` | | | Reglas extra de Claude Code, una por línea (`Bash(npm run format)`) |
| `context_files` | | | Docs del repo que debe leer además de `CLAUDE.md` y `AGENTS.md` |
| `docs_repo` | | | Repo de docs del proyecto (`terracore-docs`, `okroot-docs`, `luminaw-docs`) |
| `checkpoint_file` | | `.claude/CHECKPOINT.md` | Checkpoint que actualiza en cada PR (estándar de la org: dentro de `.claude/`) |
| `base_branch` | | `dev` | Rama de la que parte y a la que abre el PR |
| `model` / `small_model` | | `opus` / `sonnet` | Modelo normal y el de `dev-ft-small` e iterate |
| `max_turns` / `small_max_turns` | | `80` / `40` | Turnos por modo |
| `max_fix_rounds` | | `2` | Rondas máximas del modo fix |
| `timeout_minutes` | | `30` | Límite del job |

### Secrets (nivel organización, `secrets: inherit`)

| Secret | Obligatorio | Visible para | Uso |
|---|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | sí | Todos los repos de código | Autenticación de Claude |
| `DOCS_SOURCES_READ_TOKEN` | no | Repos de código y repos `*-docs` | Token de solo lectura de la organización (fine-grained PAT, Contents: Read-only) sobre los repos de código, `agents` y los 3 `*-docs`. El sync lo usa para leer el código; el agente, para leer `<proyecto>-docs`. Sin él, el agente trabaja sin contexto de producto |
| `PROMOTE_TOKEN` | para auto-merge | Repos con `automerge-dev` | Arma el auto-merge; el push a `dev` resultante dispara los workflows del repo. El dev agent lo usa para commitear el checkpoint en la rama del PR; sin él, el checkpoint no se commitea |

### Repos que lo usan

| Repo | Stack | Verificación | Docs |
|---|---|---|---|
| terracore-back | Python 3.14 | ruff, black, pytest, check, makemigrations --check | terracore-docs |
| terracore-front | npm, Node 20 | format:check, lint, type-check, test, build | terracore-docs |
| terracore-page | pnpm, `.nvmrc` | format:check, lint, typecheck, test, build (e2e si toca markup) | terracore-docs |
| okroot-back | Python 3.12 | lint, ruff format --check, test, makemigrations --check | okroot-docs |
| okroot-front | pnpm, Node 20 | typecheck, lint, test, build | okroot-docs |
| okroot-page | npm, Node 22 | lint (astro check), build | okroot-docs |
| blog-w | npm, Node 20 | lint, build | luminaw-docs |
| luminaw-page | npm, `.nvmrc` | format:check, build | luminaw-docs |

### Otros workflows compartidos de este repo

| Workflow | Qué hace |
|---|---|
| `automerge-dev.yml` | Arma auto-merge squash en PRs no-draft a `dev` (excluye forks y Dependabot). Requiere `dev` protegida con checks requeridos |
| `delete-merged-branches.yml` | Borra ramas de PRs ya mergeados a `dev` (el caller lo agenda cada 12 h) |
| `docs-sync.yml` | Sincroniza cada día `<proyecto>-docs` desde la rama `dev` de los repos del proyecto |

### Versionado

Cada cambio mergeado en `agents` lleva un tag inmutable `v4.Y.Z` y mueve el tag mayor `v4` a ese commit (mismo esquema que las actions oficiales de GitHub).

| Tipo de cambio | Ejemplos | Qué se hace | Callers |
|---|---|---|---|
| Compatible | Arreglo, quitar un paso, input o secret opcional nuevo | Tag `v4.Y.Z` y mover `v4` | No cambian |
| Rompe callers | Input o secret requerido nuevo, renombrar o quitar un input | Tag mayor nuevo (`v5`, `v5.0.0`) | PR en cada caller para subir a `@v5` |

Rollback: mover `v4` al `v4.Y.Z` anterior. Los callers apuntan siempre al tag mayor, nunca a `main`.

### Probarlo

1. Crea un issue pequeño en un repo migrado y ponle `dev-ft-small`.
2. Revisa la ejecución en Actions del repo (job `claude / agent`).
3. El PR debe llegar con label `dev-agent`, checkpoint actualizado y solo los checks marcados que corrió.
<!-- dev-agent:end -->

<!-- docs-sync:start -->
## Docs sync (`docs-sync.yml`)

Mantiene al día los repos `lumina-w/<proyecto>-docs` desde la rama `dev` de los repos de cada proyecto. Cada repo de docs tiene un caller corto (`docs-daily-sync.yml`, escalonado: terracore 06:00, okroot 06:20, luminaw 06:40 COT) y su `docs-sync.config.yml` (repos fuente, docs gestionados y protegidos, día de auditoría completa).

| Caso | Qué pasa | Modelo |
|---|---|---|
| Sin commits nuevos | No corre nada | |
| Solo dependencias (Dependabot, lockfiles) | Changelog y estado por script | |
| Commits de código | Script: `changelog` (desde `git log`) y `checkpoint` (desde `.claude/CHECKPOINT.md`). Claude recibe `_changes.md` con el diff y actualiza solo los docs afectados | `sonnet`, 40 turnos |
| Auditoría completa: `full_audit_weekday` (domingo por defecto), primer sync de un repo o `force=true` | Claude revisa todos los docs gestionados | `opus`, 150 turnos |

- Los alias `opus` y `sonnet` resuelven al modelo más reciente que permita `CLAUDE_CODE_OAUTH_TOKEN` (hoy `opus` = `claude-opus-5`).
- Inputs opcionales: `model` y `max_turns` para forzar valores en cualquier modo.
- Guardas: solo `.md` de la raíz del repo de docs; Claude no puede tocar `changelog`, `checkpoint` ni docs protegidos; un archivo por tipo.
- Publicación: PR con auto-merge, o push directo a `main` del repo de docs si la empresa no deja a Actions crear PRs.
- Sin espejo a Drive (retirado el 2026-09-23): los docs viven solo en GitHub y en local.
<!-- docs-sync:end -->

<!-- shared-workflows:start -->
## Workflows compartidos de CI/CD (`shared-*.yml`)

Lógica de CI/CD que hoy está copiada en cada repo de producto. Cada repo la llama con un caller corto y el tag mayor (`@v4`), igual que el dev agent. Los repos de producto todavía no los usan: siguen con sus copias.

Reglas comunes:

- Todos aceptan `runner-label`. Vacío corre en `ubuntu-latest`; con una etiqueta (`terracore-vps`), en el runner que la tenga; con un array JSON (`'["self-hosted", "build", "terracore-front"]'`), en el runner que tenga todas.
- Versiones fijas de acciones: `actions/checkout@v7`, `actions/setup-node@v7`, `actions/setup-python@v7`, `pnpm/action-setup@v6.1.0`, `docker/build-push-action@v7`, `docker/login-action@v4`, `docker/metadata-action@v6`, `github/codeql-action@v4`.
- Los triggers (`on:`) los pone el caller; el workflow compartido solo declara `workflow_call`.
- Cada job declara sus permisos mínimos y el job del caller debe conceder al menos esos (un reusable no puede pedir más que su caller). Excepción: `shared-cd-docker-publish.yml` usa los del caller.
- En GitHub el check queda como `<job del caller> / <nombre del job>`. Al migrar un repo hay que actualizar los checks requeridos de la protección de ramas.

### Caller mínimo

```yaml
name: Commit lint
on:
  push:
    branches: ['**']
permissions:
  contents: read
jobs:
  commitlint:
    uses: lumina-w/agents/.github/workflows/shared-commitlint.yml@v4
    with:
      runner-label: ubuntu-latest
```

### `shared-commitlint.yml`

Origen: `commit-lint.yml`. Lint de Conventional Commits sobre los commits que trae cada push.

- Trigger del caller: `push: branches: ['**']`
- Permisos: `contents: read`
- Check: `Conventional Commits`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `config-file` | string | `.commitlintrc.json` | Config de commitlint del repo |
| `install-dependencies` | boolean | `false` | `true` corre `npm ci` antes del lint, para un `extends` que apunta a un paquete del `package.json` del repo (`@lumina-w/dev-standards`) |
| `node-version` | string | `'22'` | Node para `npm ci` cuando `install-dependencies` es `true` |

| Secret | Obligatorio | Uso |
|---|---|---|
| `DEV_STANDARDS_DEPLOY_KEY` | no | Deploy key de solo lectura para instalar `@lumina-w/dev-standards` por `git+ssh`. Solo se usa con `install-dependencies: true` |

### `shared-pr-title.yml`

Origen: `pr-title.yml`. Valida el título del PR (el commit que queda tras el squash) con el mismo `.commitlintrc.json`.

- Trigger del caller: `pull_request: types: [opened, edited, synchronize, reopened]`
- Permisos: `contents: read`
- Check: `PR title (Conventional Commits)`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `config-file` | string | `.commitlintrc.json` | Config de commitlint del repo |
| `node-version` | string | `'22'` | Node para instalar commitlint |
| `commitlint-version` | string | `'21'` | Versión mayor de `@commitlint/cli` y `@commitlint/config-conventional` |
| `install-dependencies` | boolean | `false` | `true` corre `npm ci` y valida desde la raíz del repo, para un `extends` que apunta a un paquete del `package.json` del repo. Solo agrega `@commitlint/cli`, sin tocar `package.json` |

| Secret | Obligatorio | Uso |
|---|---|---|
| `DEV_STANDARDS_DEPLOY_KEY` | no | Igual que en `shared-commitlint.yml` |

### `shared-validate-pr-base.yml`

Origen: `validate-pr-base.yml`. Exige el flujo `dev -> stg -> main` (o una rama de resolución de conflictos con el mismo contenido que la rama esperada).

- Trigger del caller: `pull_request: types: [opened, edited, synchronize, reopened]`
- Permisos: `contents: read`
- Check: `PR base must follow dev -> stg -> main`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |

### `shared-promote-dev-to-stg.yml`

Origen: `promote-dev-to-stg.yml`. Abre o reutiliza el PR de promoción, espera los checks requeridos y lo mergea. Una sola promoción a la vez por par de ramas.

- Trigger del caller: `schedule` (hoy `0 10 * * *`) y `workflow_dispatch`
- Permisos: `contents: read`, `pull-requests: write`, `checks: read`, `statuses: read`, `actions: read`
- Secrets: `PROMOTE_TOKEN` (obligatorio). Con `GITHUB_TOKEN` el PR no dispararía los checks

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `source-branch` | string | `dev` | Rama que se promueve |
| `target-branch` | string | `stg` | Rama destino |
| `merge-method` | string | `squash` | `squash`, `merge` o `rebase` |
| `timeout-minutes` | number | `60` | Límite del job |

### `shared-claude-code-review.yml`

Origen: `claude-code-review.yml`. Review de Claude con comentarios inline en el PR. Un push nuevo al mismo ref cancela la review en curso.

- Trigger del caller: `pull_request: types: [opened, synchronize, ready_for_review, reopened]`
- Permisos: `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`
- Secrets: `CLAUDE_CODE_OAUTH_TOKEN` (obligatorio)

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `model` | string | `''` | Se pasa como `--model` (`claude-sonnet-5`, `sonnet`). Vacío usa el modelo por defecto de la action |
| `timeout-minutes` | number | `15` | Límite del job |

### `shared-codeql.yml`

Origen: `codeql.yml`. Análisis CodeQL de un lenguaje. Por defecto es informativo: si el análisis o la subida del SARIF fallan (por ejemplo, Code Security deshabilitado en un repo privado), el check sigue en verde.

- Trigger del caller: `push`/`pull_request` a `main`, `stg`, `dev` y `schedule` (hoy lunes 06:00 UTC), o solo `workflow_dispatch` en repos sin GHAS
- Permisos: `security-events: write`, `actions: read`, `contents: read`
- Check: `analyze`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `language` | string | obligatorio | `python`, `javascript-typescript`, ... |
| `queries` | string | `''` | Suite extra (`security-and-quality`). Vacío usa la suite por defecto |
| `blocking` | boolean | `false` | `true` hace fallar el check si falla el análisis o la subida |
| `timeout-minutes` | number | `30` | Límite del job |

### `shared-security-audit-node.yml`

Origen: `security.yml` de terracore-front (npm) y el job de auditoría del `ci.yml` de okroot-page (npm) y terracore-page (pnpm). Instala dependencias y corre `npm audit` o `pnpm audit`.

- Trigger del caller: el que use el repo (`push`/`pull_request`/`schedule`)
- Permisos: `contents: read`
- Check: `npm-audit` o `pnpm-audit`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `package-manager` | string | obligatorio | `npm` o `pnpm` |
| `node-version` | string | `''` | Versión de Node (`20.x`, `22`) |
| `node-version-file` | string | `''` | Archivo de versión (`.nvmrc`) |
| `pnpm-version` | string | `''` | Versión de pnpm. Vacío lee `packageManager` de `package.json` |
| `working-directory` | string | `'.'` | Carpeta con `package.json` y lockfile |
| `install-command` | string | `''` | Vacío usa `npm ci` o `pnpm install --frozen-lockfile` |
| `audit-level` | string | `high` | `low`, `moderate`, `high` o `critical` |
| `blocking` | boolean | `false` | `true` hace fallar el check con hallazgos en o sobre `audit-level` |
| `timeout-minutes` | number | `15` | Límite del job |

### `shared-security-audit-python.yml`

Origen: `security.yml` de terracore-back. Dos jobs: `pip-audit` (dependencias, bloqueante por defecto) y `bandit` (SAST, informativo por defecto).

- Trigger del caller: el que use el repo (`push`/`pull_request`/`schedule`)
- Permisos: `contents: read`
- Checks: `pip-audit` y `bandit`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `python-version` | string | `''` | Versión de Python (`3.14`, `3.12`) |
| `python-version-file` | string | `''` | Archivo de versión (`.python-version`) |
| `working-directory` | string | `'.'` | Carpeta donde corre pip-audit |
| `requirements-file` | string | `requirements.txt` | Requirements que audita pip-audit, relativo a `working-directory` |
| `pip-audit-version` | string | `2.9.0` | Versión de pip-audit |
| `pip-audit-blocking` | boolean | `true` | `false` deja pip-audit solo informativo |
| `bandit-version` | string | `1.9.4` | Versión de bandit |
| `bandit-path` | string | `'.'` | Ruta que escanea bandit, relativa a la raíz del repo |
| `bandit-exclude` | string | `*/tests/*,*/migrations/*,*/.venv/*,*/venv/*` | Rutas excluidas de bandit |
| `bandit-blocking` | boolean | `false` | `true` hace fallar el check con hallazgos de bandit |
| `timeout-minutes` | number | `15` | Límite de cada job |

### `shared-gitleaks.yml`

Origen: job `gitleaks` del `ci.yml` de terracore-back y terracore-front. Escaneo de secretos del diff del PR o de los commits del push.

- Trigger del caller: el del CI del repo (`pull_request`)
- Permisos: `contents: read`, `pull-requests: read`
- Secrets: `GITLEAKS_LICENSE`. La action la exige en repos de organización; tiene que existir también en el store de secrets de Dependabot
- Check: `Secret scan (gitleaks)`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `timeout-minutes` | number | `10` | Límite del job |

### `shared-cd-docker-publish.yml`

Origen: job `docker-publish` de `cd-staging.yml`/`cd-production.yml` (terracore-front, okroot-back) y `docker-build-push.yml` (okroot-front). Construye la imagen, la escanea con Trivy (opcional, falla con cualquier CRITICAL que tenga arreglo), la sube a GHCR con el tag `<tag-prefix>-<sha completo>` y, si se pide, envía un `repository_dispatch`. El dispatch solo sale en un `push` real, nunca en `workflow_dispatch`.

- Trigger del caller: `push` a `stg` o `main` (y `workflow_dispatch` si se quiere republicar sin notificar)
- Permisos (los concede el caller): `contents: read` y `packages: write`; `contents: write` si hace dispatch al mismo repo sin `DISPATCH_TOKEN`
- Secrets: `BUILD_ARGS` (opcional, `CLAVE=valor` por línea) y `DISPATCH_TOKEN` (opcional; obligatorio si `dispatch-target-repo` es otro repo). Sin `DISPATCH_TOKEN` usa el token del workflow
- Payload del dispatch: `{"environment": <tag-prefix>, "image_digest": ..., "image": ..., "sha": ...}`, que cubre los campos que leen hoy terracore-back, okroot-back y okroot-front
- Outputs: `image` (nombre:tag) y `digest`

| Input | Tipo | Default | Uso |
|---|---|---|---|
| `runner-label` | string | `''` | Etiqueta del runner |
| `dockerfile` | string | `''` | Ruta del Dockerfile desde la raíz. Vacío usa `<context>/Dockerfile` |
| `context` | string | `'.'` | Contexto del build |
| `image-name` | string | `''` | Imagen completa en GHCR. Vacío usa `ghcr.io/<owner>/<repo>` |
| `tag-prefix` | string | obligatorio | Prefijo del tag (`stg`, `main`, `prod`); también va como `environment` en el payload |
| `build-args` | string | `''` | Build args no secretos, `CLAVE=valor` por línea |
| `environment` | string | `''` | GitHub Environment del job. Vacío no asocia ninguno |
| `trivy-scan` | boolean | `true` | Escanea la imagen antes de subirla |
| `dispatch-event` | string | `''` | Tipo de evento del `repository_dispatch`. Vacío no envía nada |
| `dispatch-target-repo` | string | `''` | `owner/repo` que recibe el dispatch. Vacío usa el repo actual |
| `timeout-minutes` | number | `15` | Límite del job |

Secrets de un GitHub Environment: el job del caller no puede tener `environment`, así que no puede pasarlos. Si se usa el input `environment` y ese Environment tiene secrets llamados `BUILD_ARGS` o `DISPATCH_TOKEN`, esos reemplazan a los que pase el caller.

```yaml
jobs:
  publish:
    uses: lumina-w/agents/.github/workflows/shared-cd-docker-publish.yml@v4
    permissions:
      # write: dispatch al mismo repo con el token del workflow
      contents: write
      packages: write
    with:
      tag-prefix: stg
      dockerfile: Dockerfile.prod
      dispatch-event: backend-image-updated
    secrets:
      BUILD_ARGS: |
        API_URL=${{ secrets.API_URL }}
```
<!-- shared-workflows:end -->
