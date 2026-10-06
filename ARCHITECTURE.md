# ARCHITECTURE.md

The architecture document for this repository, following the
[architecture.md](https://architecture.md) schema — built so an agent (or a
new colleague) can comprehend the repository from this file alone. This is a
**shared-workflows** repository for PHP/Laravel projects: it ships the GitHub
Actions templates and Dependabot config a Laravel app or module copies into
its `.github/` folder, plus the PHP/Laravel conventions those projects
follow. Fill every section; update it in the same change that alters the
architecture it describes.

## 1. Project Structure

Flat: the workflows are the product. GitHub does not traverse into
submodules, so these files are **copied** into a consumer's `.github/` folder,
not consumed as a submodule (only the devshell is a submodule).

```
.github-php/
├── workflows/                     # the templates a consumer copies
│   ├── test.yaml                  # thin wrapper → this repo's real test workflow
│   ├── analyse.yaml               # thin wrapper → this repo's real analyse workflow
│   ├── changelog.yaml             # reusable changelog workflow
│   └── automatic-updates.yaml     # reusable Dependabot/agent workflow
├── .github/workflows/             # the REAL PHP workflows, called with `uses:`
│   ├── test.yaml                  # reusable test workflow (Pest)
│   └── analyse.yaml               # reusable deptrac boundary gate
├── dependabot.yaml                # the Dependabot config a consumer copies
├── devshell/                      # git submodule: devshell-php
├── .editorconfig, .prettierrc     # shared formatting
└── README.md
```

**The PHP/Laravel conventions consumers inherit** (baked in here, enforced in
their repos): a project is a Laravel app (PSR-4 `App\` → `app/`) built on
`laravel-package-skeleton`, pulling in modules as Composer packages. Each
module is a bounded context with hexagonal-flavored layering (`App/` /
`Domain/` / `Infrastructure/`) and PSR-4 root `Lines\<Module>\` → `src/`.
Actions carry the logic; DTOs cross the boundary; models stay lean. The
workflows here test and analyse those projects; release is the changelog
workflow's job.

## 2. High-Level System Diagram

```
  .github-php (this repo)                 consumer Laravel project
  ┌──────────────────────────┐            ┌──────────────────────────────┐
  │ .github/workflows/       │  uses: →   │ .github/workflows/test.yaml  │
  │   test.yaml (real)       │            │ .github/workflows/analyse…   │
  │   analyse.yaml (real)    │            │   (thin wrappers)            │
  │ workflows/*.yaml         │  copy →    │ .github/workflows/changelog… │
  │   (thin wrapper          │            │ .github/workflows/automatic… │
  │   templates)             │            │ .github/dependabot.yaml      │
  │ dependabot.yaml          │            └──────────────────────────────┘
  └────────────┬─────────────┘
               │  changelog + automatic-updates still delegate to:
               ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  99linesofcode/.github  (org reusable workflows)                      │
  │  changelog.yaml · automatic-updates.yaml · update-agent.yaml          │
  └──────────────────────────────────────────────────────────────────────┘
```

## 3. Core Components

| Component | Responsibility |
|---|---|
| `.github/workflows/test.yaml` | The real reusable test workflow: runs the consumer's Pest suite, parameterised by `calling_repository_name` |
| `.github/workflows/analyse.yaml` | The real reusable boundary gate: deptrac enforces the layer contract on every PR |
| `workflows/test.yaml` | Thin wrapper template a consumer copies; delegates to this repo's real test workflow |
| `workflows/analyse.yaml` | Thin wrapper template a consumer copies; delegates to this repo's real analyse workflow |
| `workflows/changelog.yaml` | Reusable changelog generation on push to `main` |
| `workflows/automatic-updates.yaml` | Reusable Dependabot/agent update workflow on PRs |
| `dependabot.yaml` | Dependabot config covering gitsubmodule, npm and Composer ecosystems |
| `devshell/` | Pinned PHP dev environment (`devshell-php`) |

### Ports & adapters

Not applicable to this repo. For the Laravel projects it serves, the
pattern is fixed: the `Domain` owns a port when a real external seam exists,
`Infrastructure/` holds the adapter, and the `*ServiceProvider` is the
composition root. This repo adds no ports of its own.

## 4. Data Stores

None. This repo holds workflow definitions and configuration; the consumer's
Laravel app owns its database. The test workflow runs the consumer's tests
against the consumer's configured database (the skeleton defaults to SQLite
`:memory:`).

## 5. External Integrations / APIs

- **GitHub Actions** — the real PHP workflows live in THIS repo at
  `.github/workflows/` and are invoked with `uses:`; the changelog and
  automatic-updates templates still delegate to
  `99linesofcode/.github/.github/workflows/`. Method: reusable workflows.
- **GitHub Dependabot** — `dependabot.yaml` drives dependency updates.
- No runtime service is integrated.

## 6. Deployment & Infrastructure

- **Consumption**: copy the wrapper template(s) you need into your project's
  `.github/` folder. GitHub cannot traverse into submodules, so these are
  copies, not a submodule. The wrappers delegate to this repo's real
  workflows with `uses:`.
- **Test**: `workflows/test.yaml` (wrapper) runs on non-`main` pushes and
  PRs, delegating to this repo's real test workflow.
- **Analyse**: `workflows/analyse.yaml` (wrapper) runs the deptrac boundary
  gate on non-`main` pushes and PRs.
- **Release**: `workflows/changelog.yaml` generates the changelog on push to
  `main`.
- **Local environment**: the `devshell` submodule (`devshell-php`) via
  `.envrc` → `use flake ./devshell`.
- **Monitoring/logging**: app-owned; nothing here.

## 7. Security Considerations

- **Secrets are GitHub Actions secrets**, passed explicitly and never
  inlined. The Kamal deploy workflow (with its `LARAVEL_ENV_ENCRYPTION_KEY`
  and `SSH_PRIVATE_KEY` forwarding) was dropped — deployment is superseded
  by the Kubernetes fleet.
- **Least privilege in the workflow permissions**: test/analyse declare
  `contents: read`; changelog declares `contents: write`; automatic-updates
  declares `pull-requests`, `contents` and `issues` write (it needs to push
  updates and comment).
- **`.env` files are git-ignored**; only `.env.example` is committed.
- **Dependabot** keeps the submodule, npm and Composer dependencies current.
- No application secrets or runtime attack surface live in this repo.

## 8. Development & Testing Environment

- **Local setup**: `git submodule update --init --recursive` then
  `direnv allow` for the `devshell-php` environment.
- **Testing**: this repo has no test suite of its own; the workflows it ships
  run the consumer's Pest suite via the org test workflow.
- **Code quality**: `.prettierrc` / `.editorconfig` for the YAML/Markdown;
  the workflows are validated by GitHub on use.
- **Mechanical gates and what each makes impossible**:
  - The **reusable test workflow** makes a red test suite un-mergeable in the
    consumer's PR pipeline.
  - The **reusable analyse workflow** makes a layer-contract violation
    un-mergeable in the consumer's PR pipeline.
  - The **branch filter** (`branches-ignore: main` for test/analyse) makes a
    post-merge run unnecessary while keeping PRs gated.
  - The **boundary gate** for consumers is **deptrac** (see §12).

## 9. Future Considerations / Roadmap

**Deliberate non-goals:**

- **Not a submodule.** GitHub does not traverse into submodules for
  workflows, so these files are copied; the README states this explicitly.
- **PHP/Laravel only.** This is the PHP variant of the shared workflow set;
  other ecosystems get their own starter.
- **No application code or deployment logic here.** The test/analyse heavy
  lifting lives in this repo's real workflows; changelog and automatic-updates
  still delegate to the org's reusable workflows. Kamal deploy logic was
  dropped — deployment is superseded by the Kubernetes fleet.
- **No secrets in the repo.** Secrets are supplied by the consumer's
  repository settings.

**Known debt / open items**: none recorded.

## 10. Project Identification

Project Name: .github-php (github-php)

Repository URL: https://github.com/99linesofcode/.github-php

Primary Contact/Team: Jordy Schreuders (99linesofcode)

Date of Last Update: 2026-10-06

## 11. Glossary / Acronyms

- **Reusable workflow** — a GitHub Actions workflow invoked with `uses:` from
  another repository.
- **Community health files** — repository files (workflows, Dependabot,
  issue templates) shared across an org; here they are copied, not inherited.
- **PSR-4** — the PHP autoloading standard; `Lines\<Module>\` → `src/`,
  `App\` → `app/`.
- **Module** — a bounded context as a Composer package.
- **Action / DTO** — the Laravel use-case seam and its readonly data shape.
- **ServiceProvider** — the Laravel composition root.
- **deptrac** — the PHP dependency-boundary analyser; the layer gate.
- **Testbench** — the package that hosts a Laravel app for testing a package.
- **Devshell** — a `nix develop` environment (`devshell-php`).

## 12. Conventions & Boundaries

The house standards for the PHP/Laravel projects this repo serves — stated
here in full; this section records what is enforced and by which gate.

- **Folder structure**: PSR-4. A host app autoloads `App\` → `app/`; a module
  uses `Lines\<Module>\` → `src/` with the hexagonal-flavored layering
  `App/` (UI) / `Domain/` / `Infrastructure/`. Modules are Composer packages
  resolved into `vendor/`, never nested in the project tree. The path
  locates the layer and module; the name locates the role.
- **File naming**: role suffixes — `*Action`, `*Data`, `*Status`, `*Factory`,
  `*ServiceProvider`, `*QueryBuilder`, `*Collection`, `*Event`, `*Rule`.
  Models stay bare. One class per file.
- **Entry point**: Laravel's `public/index.php` + `artisan`; the
  `*ServiceProvider` is the composition root — the one place wiring and
  interface bindings happen (never `app()`/`resolve()` inside a class body).
- **Dependency direction**: `App` (UI) → `Domain` → `Infrastructure`;
  `Domain` imports no Filament/Livewire; `Infrastructure` implements the
  domain's ports; the UI never calls Eloquent directly. Enforced by
  **deptrac**, the boundary gate (the PHP equivalent of
  `eslint-plugin-boundaries`) — a naming standard without a gate erodes one
  change at a time.
- **Actions carry the logic**: every user story is an invokable action taking
  a DTO; actions compose actions; constructor injection;
  `DB::transaction()` around multi-write operations.
- **Canonical DTOs**: one readonly DTO per domain concept, extending the
  shared `DataTransferObject` base with `casts()`; raw provider shapes are
  mapped at the boundary.
- **Lean models**: UUID keys, data + identity only; no calculations in
  accessors; scopes become `*QueryBuilder`s; collection logic becomes a
  `*Collection`.
- **Tooling**: Pint formatting, Larastan/PHPStan analysis, Rector refactors,
  Pest + Testbench tests — wired through Composer scripts and the shared
  `test.yaml` workflow.
- **Workflow consumption**: copy the `.yaml` into `.github/` (not a
  submodule); the devshell is the only submodule.
- **Documentation surfaces**: WHY comments at the change site; a change to a
  shared workflow or a convention updates this file in the same change.
