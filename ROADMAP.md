# Roadmap

Pending work for this repo.

## Next

- **`docs-sync.yml` must support a docs folder.** It looks for and validates the dated markdown
  (`<type>-YYYY-MM-DD.md`) in the root of the docs repo (`ls *.md`, `^[^/]+\.md$`), and it writes the
  renamed files there. `terracore-docs` moved its markdown into `docs/` (2026-10-03), so its daily
  sync will not find them. Add an input or a config key (for example `docs.dir`) for the folder, use it
  in the glob, the validation of changed paths, the rename step and the prompt, keep the root as the
  default, and update `README.md` and `AGENTS.md` in the same PR.
