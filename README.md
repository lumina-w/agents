# agents
<!-- dev-agent:start -->
## Dev agent (`dev-agent.yml`)

El dev agent de Lúmina W vive **una sola vez** en este repo. Cada repo de código tiene solo un `claude.yml` corto que lo llama con un tag fijo (`@v3`) y le pasa lo que es propio del repo. No hay copias de la lógica en los repos.

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
| Label `dev-ft-ready` en un issue | implement | `claude-opus-5` | 40 |
| Label `dev-ft-small` en un issue (tareas pequeñas) | implement | `claude-sonnet-5` | 20 |
| Comentario con `@claude` en un PR | iterate | `claude-sonnet-5` | 20 |
| El workflow `CI` falla en un PR con label `dev-agent` | fix | `claude-opus-5` | 40 |

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
| 8 | Checkpoint | Actualiza el checkpoint del repo (`.claude/CHECKPOINT.md` por defecto) en el mismo PR |

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
    uses: lumina-w/agents/.github/workflows/dev-agent.yml@v3
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
| `checkpoint_file` | | `.claude/CHECKPOINT.md` | Checkpoint que actualiza en cada PR |
| `base_branch` | | `dev` | Rama de la que parte y a la que abre el PR |
| `model` / `small_model` | | `claude-opus-5` / `claude-sonnet-5` | Modelo normal y el de `dev-ft-small` e iterate |
| `max_turns` / `small_max_turns` | | `40` / `20` | Turnos por modo |
| `max_fix_rounds` | | `2` | Rondas máximas del modo fix |
| `timeout_minutes` | | `30` | Límite del job |

### Secrets (nivel organización, `secrets: inherit`)

| Secret | Obligatorio | Visible para | Uso |
|---|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | sí | Todos los repos de código | Autenticación de Claude |
| `PROJECT_DOCS_READ_TOKEN` | no | Todos los repos de código | Leer `<proyecto>-docs`. Fine-grained PAT, repos `terracore-docs`, `okroot-docs`, `luminaw-docs`, Contents: Read-only. Sin él, el agente trabaja sin contexto de producto |
| `PROMOTE_TOKEN` | para auto-merge | Repos con `automerge-dev` | Arma el auto-merge; el push a `dev` resultante dispara los workflows del repo |

### Repos que lo usan

| Repo | Stack | Verificación | Docs |
|---|---|---|---|
| terracore-back | Python 3.14 | ruff, black, pytest, check, makemigrations --check | terracore-docs |
| terracore-front | npm, Node 20 | format:check, lint, type-check, test, build | terracore-docs |
| terracore-page | pnpm, `.nvmrc` | format:check, lint, typecheck, test, build (e2e si toca markup) | terracore-docs |
| okroot-back | Python 3.12 | lint, ruff format --check, test, makemigrations --check | okroot-docs |
| okroot-front | pnpm, Node 20 | typecheck, lint, test, build | okroot-docs |
| okroot-page | npm, Node 22 | lint (astro check), build | okroot-docs |
| blogw-front | npm, Node 20 | lint, build | luminaw-docs |
| luminaw-page | npm, `.nvmrc` | format:check, build | luminaw-docs |

### Otros workflows compartidos de este repo

| Workflow | Qué hace |
|---|---|
| `automerge-dev.yml` | Arma auto-merge squash en PRs no-draft a `dev` (excluye forks y Dependabot). Requiere `dev` protegida con checks requeridos |
| `delete-merged-branches.yml` | Borra ramas de PRs ya mergeados a `dev` (el caller lo agenda cada 12 h) |
| `docs-sync.yml` | Sincroniza cada día `<proyecto>-docs` desde la rama `dev` de los repos del proyecto, con espejo a Drive |

### Versionado

Los callers apuntan a un tag (`@v3`), nunca a `main`. Cambio en un workflow compartido: PR a este repo, merge, tag nuevo, y PR en cada caller para subir el tag. Así un cambio nunca llega a todos los repos sin revisión.

### Probarlo

1. Crea un issue pequeño en un repo migrado y ponle `dev-ft-small`.
2. Revisa la ejecución en Actions del repo (job `claude / agent`).
3. El PR debe llegar con label `dev-agent`, checkpoint actualizado y solo los checks marcados que corrió.
<!-- dev-agent:end -->
