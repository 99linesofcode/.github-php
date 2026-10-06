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
│   ├── test.yaml                  # reusable test workflow (Pest)
│   ├── deploy.yaml                # reusable Laravel deploy workflow (SSH)
│   ├── changelog.yaml             # reusable changelog workflow
│   └── automatic-updates.yaml     # reusable Dependabot/agent workflow
├── dependabot.yaml                # the Dependabot config a consumer copies
├── .github/                       # this repo's own workflows + dependabot
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
workflows here test, deploy and release those projects.

## 2. High-Level System Diagram

```
  .github-php (this repo)                 consumer Laravel project
  ┌──────────────────────────┐            ┌──────────────────────────────┐
  │ workflows/test.yaml      │  copy →    │ .github/workflows/test.yaml  │
  │ workflows/deploy.yaml    │            │ .github/workflows/deploy.yaml│
  │ workflows/changelog.yaml │            │ .github/workflows/changelog… │
  │ workflows/automatic-…    │            │ .github/workflows/automatic… │
  │ dependabot.yaml          │            │ .github/dependabot.yaml      │
  └────────────┬─────────────┘            └───────────────┬──────────────┘
               │                                          │
               ▼                                          ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │  99linesofcode/.github  (org reusable workflows)                      │
  │  test.yaml · deploy.yaml · changelog.yaml · automatic-updates.yaml ·  │
  │  update-agent.yaml                                                    │
  └──────────────────────────────────────────────────────────────────────┘
```

## 3. Core Components

| Component | Responsibility |
|---|---|
| `workflows/test.yaml` | Reusable test workflow: runs the consumer's Pest suite, parameterised by `calling_repository_name` |
| `workflows/deploy.yaml` | Reusable Laravel deploy workflow: deploys over SSH with the encrypted `.env` key and SSH private key |
| `workflows/changelog.yaml` | Reusable changelog generation on push to `main` |
| `workflows/automatic-updates.yaml` | Reusable Dependabot/agent update workflow on PRs |
| `dependabot.yaml` | Dependabot config covering gitsubmodule, npm and Composer ecosystems |
| `.github/workflows/*` | This repo's own use of the templates |
| `devshell/` | Pinned PHP dev environment (`devshell-php`) |

### Ports & adapters

Not applicable to this repo. For the Laravel projects it serves, the
`laravel` skill defines the pattern: the `Domain` owns a port when a real
external seam exists, `Infrastructure/` holds the adapter, and the
`*ServiceProvider` is the composition root. This repo adds no ports of its
own.

## 4. Data Stores

None. This repo holds workflow definitions and configuration; the consumer's
Laravel app owns its database. The test workflow runs the consumer's tests
against the consumer's configured database (the skeleton defaults to SQLite
`:memory:`).

## 5. External Integrations / APIs

- **GitHub Actions** — the reusable workflows in
  `99linesofcode/.github/.github/workflows/` are invoked with `uses:`. Method:
  reusable workflows.
- **GitHub secrets** — `deploy.yaml` passes `LARAVEL_ENV_ENCRYPTION_KEY` and
  `SSH_PRIVATE_KEY` to the org deploy workflow.
- **GitHub Dependabot** — `dependabot.yaml` drives dependency updates.
- No runtime service is integrated.

## 6. Deployment & Infrastructure

- **Consumption**: copy the `.yaml` file(s) you need into your project's
  `.github/` folder. GitHub cannot traverse into submodules, so these are
  copies, not a submodule.
- **Test**: `workflows/test.yaml` runs on non-`main` pushes and PRs (branches
  off `main`), delegating to the org test workflow.
- **Deploy**: `workflows/deploy.yaml` runs on push to `main`, delegating to
  the org deploy workflow with the Laravel env encryption key and SSH key.
- **Release**: `workflows/changelog.yaml` generates the changelog on push to
  `main`.
- **Local environment**: the `devshell` submodule (`devshell-php`) via
  `.envrc` → `use flake ./devshell`.
- **Monitoring/logging**: app-owned; nothing here.

## 7. Security Considerations

- **Secrets are GitHub Actions secrets**, passed explicitly and never
  inlined: the deploy workflow forwards `LARAVEL_ENV_ENCRYPTION_KEY` (the
  key that decrypts the app's encrypted `.env`) and `SSH_PRIVATE_KEY`.
- **Least privilege in the workflow permissions**: test/deploy declare
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
  - The **branch filter** (`branches-ignore: main` for test) makes a
    post-merge test run unnecessary while keeping PRs gated.
  - The **deploy workflow's secret requirements** make a deploy without the
    env key or SSH key impossible.
  - The **boundary gate** for consumers is **deptrac** (see §12).

## 9. Future Considerations / Roadmap

**Deliberate non-goals:**

- **Not a submodule.** GitHub does not traverse into submodules for
  workflows, so these files are copied; the README states this explicitly.
- **PHP/Laravel only.** This is the PHP variant of the shared workflow set;
  other ecosystems get their own starter.
- **No application code or deployment logic here.** The heavy lifting lives
  in the org's reusable workflows; this repo is the thin, copyable layer.
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

The house standards for the PHP/Laravel projects this repo serves — the full
contract lives in the `software-architecture` and `laravel` skills; this
section records what is enforced and by which gate.

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
