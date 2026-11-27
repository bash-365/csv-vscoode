<!-- kit: sdlc v0.2.0 · AGENTS.md is the CANONICAL agent instructions file (decision D4, UPS-AGENT-07/08).
     Keep these six `##` headings (UPS-AGENT-07 checks they exist, in any order; add your own headings as needed). CLAUDE.md is a pointer to this file and carries no rules of its own. -->

# csv-vscoode

## What this repo is

The source of **CSV Grid Editor**, a VS Code extension published as `bash-365.csv-grid-viewer` (version 1.0.0) on the Visual Studio Marketplace and Open VSX. It opens `.csv`, `.tsv`, `.tab` and `.psv` files in a spreadsheet-style grid over the text document itself, so save, undo, the dirty indicator and Git behave as they do for any file, and adds Excel-style AutoFilter, find and replace, statistics, charts, SQL over the file (sql.js) and repeatable `*.csvpipe.json` pipelines. It serves VS Code users who work with delimited data. User documentation is published at <https://apps.bash-365.com/csv-vscode/>, built from `docs/` by `site/`. A change is done when `npm run lint`, `npm test` and `npm run build` pass, `docs/` and `CHANGELOG.md` describe any user-visible change, and a webview change has been exercised by hand in the Extension Development Host (cell edit and undo, range selection and copy, a filter, a save), because the webview has no automated UI test.

## Stack & commands

TypeScript on the VS Code extension API (`engines.vscode` `^1.90.0`), bundled with esbuild into `out/extension.js` (host, Node) and `out/webview/main.js` (browser). Runtime libraries: PapaParse (parsing), Tabulator (grid), sql.js (SQLite in WebAssembly), Chart.js (charts). Vitest for tests, ESLint with typescript-eslint for lint. CI and the release run on Node 22; the extension itself targets Node 20 (see Conventions). The commands, as in [CONTRIBUTING.md](CONTRIBUTING.md):

```bash
npm install                  # install
npm run compile              # development build: both bundles plus the sql.js copy into out/sqljs
npm run watch                # rebuild on change
# run: press F5 in VS Code ("Run CSV Grid Editor") for an Extension Development Host with samples/ open
npm test                     # Vitest unit tests in test/
npm run lint                 # ESLint over the whole repository
npm run typecheck            # tsc --noEmit for the host and the webview
npm run build                # typecheck plus production bundles; what CI runs
npm run package:vsix         # build and produce csv-grid-viewer-<version>.vsix
python3 tools/unwrap-prose.py --write   # markdown is one paragraph per line; CI enforces it
npm run docs:install && npm run docs:dev  # the documentation site in site/ (Astro + Starlight)
```

## Layout

| Path | What lives there |
| --- | --- |
| `src/core/` | Pure TypeScript shared by the host, the webview bundle and the tests: parse, serialize, edits (`EditOp`), transforms, filters, pipelines, inference, statistics. No `vscode` import, no DOM. |
| `src/` | The extension host (Node): the custom editor (`csvEditorProvider.ts`), the document model, pipeline evaluation, SQL, status bar, text mode, and the sidebar views in `src/views/`. |
| `webview/` | The webview bundle (browser): the Tabulator grid, find bar, filter popup, dialogs, and the statistics, SQL and pipeline panels. DOM only, no Node APIs. |
| `test/` | Vitest tests for `src/core/` and the Node pipeline evaluator, run against real scripts and subprocesses. |
| `schemas/` | `csvpipe.schema.json`, the JSON Schema VS Code validates `*.csvpipe.json` against. |
| `samples/` | Sample CSV/TSV files, pipelines and pipeline scripts; the F5 launch opens this folder. |
| `media/` | Icons, the demo GIF and the screenshots used by the README and the site. |
| `docs/` | The user guides (the source of the site's guide pages), `RELEASING.md`, `FEATURE-PARITY.md`, and the decision log in `docs/adr/`. Indexed in [docs/README.md](docs/README.md). |
| `site/` | The documentation site (Astro + Starlight, its own `package.json`). `site/scripts/sync-docs.mjs` generates its guide and reference pages from the repository's markdown, `package.json` and the schema; see [site/README.md](site/README.md). |
| `scripts/` | Publishing helpers behind the `publish:*` and `package:vsix` npm scripts. |
| `tools/` | `unwrap-prose.py`, vendored from the bamr87 hub. |
| `.github/` | Workflows: `ci.yml` (the hub's `standard-ci.yml@v2` on Node 22), `release.yml` (a `v*` tag builds, packages and publishes the VSIX), `docs.yml` (builds, link-checks and deploys the site to GitHub Pages), `markdown-oneline.yml`, `claude.yml`; and `sdlc.yml`, the SDLC declaration. |
| `out/` | Build output. Git-ignored. |

## Conventions

- **Backlog of record:** GitHub Issues at <https://github.com/bash-365/csv-vscoode/issues> with the fleet labels. Pick up issues labelled `agent:ready`; never touch `agent:hold` or `human-review`.
- **Definition of Done:** the bar at the end of "What this repo is" above. The repository has no `.github/pull_request_template.md` of its own.
- **Decisions:** ADRs in `docs/adr/NNNN-slug.md`, indexed in [docs/adr/README.md](docs/adr/README.md). Add one for anything hard to reverse.
- **Commits:** Conventional Commits, `type(scope): description`, using `feat`, `fix`, `docs`, `refactor`, `test`, `chore` or `ci`. The PR title follows the same form.
- **Branches:** `<type>/<slug>` off `main` (for example `ci/claude-oauth-only`), landed by pull request.
- **Changelog and versions:** `CHANGELOG.md` is maintained by hand. Add an entry under `## Unreleased` for every user-visible change. The `package.json` version changes only as part of a release, following [docs/RELEASING.md](docs/RELEASING.md).
- **Architecture:** keep `src/core/` free of `vscode` and the DOM; anything expressible as a pure function on a `CsvTable` belongs there, with a test. A change to data is an `EditOp` (a case in `applyEdit` plus a test), so undo works for free. A new pipeline step is a `StepDef` entry, a case in `applyPureStep` or `applyStep`, and an entry in `schemas/csvpipe.schema.json`. The full walkthrough is "Adding a feature" in [CONTRIBUTING.md](CONTRIBUTING.md).
- **Node ceiling:** code under `src/` must not use APIs newer than Node 20, the oldest extension host for `^1.90.0`. That is why `esbuild.mjs` targets `node20`, `@types/vscode` stays at `1.90.0`, and TypeScript stays on the 6.x line.
- **Workspace trust:** pipeline steps that evaluate code (expressions, JavaScript, SQL, external commands) run only in a trusted workspace with `csv.pipelines.allowScripts` on (`src/pipelines.ts`). Keep that gate on every new code-evaluating step.
- **Documentation:** edit the markdown in `docs/`, `README.md`, `CHANGELOG.md` or `CONTRIBUTING.md`; the site regenerates from it. Read the nearest `README.md` before changing a directory and update it afterwards. Markdown is one paragraph per line.
- **Secrets:** publishing tokens (`VSCE_PAT`, `OVSX_PAT`) live in a git-ignored `.env` made from `.env.example`, and in repository secrets for CI. A new secrets file is excluded in both `.gitignore` and `.vscodeignore`, or it ships inside the VSIX.
- **Never:** push to `main` directly; commit secrets or a real `.env`; suppress type errors (`as any`, `@ts-ignore`); leave an empty `catch` (one that deliberately ignores an error says why in a comment); edit generated files by hand: the site's generated pages (`site/src/content/docs/guides/`, `reference/` and the others listed in `site/.gitignore`; edit their source instead), `out/`, or `package-lock.json` (npm writes it).

## Fleet context

Member of the bamr87 fleet, owned by the `bash-365` organization. The standards live in the hub: <https://github.com/bamr87/bamr87/tree/main/specs> (the Universal Project Standard). This repo's SDLC declaration is `.github/sdlc.yml`. The hub registers it in `_data/projects.yml` but does not check it out as a submodule, so changes are committed and pushed here.

Org-wide context lives in the org brain, <https://github.com/bash-365/bash-365.com/tree/main/brain>.

Hub kits seeded `ci.yml` (a thin caller of the hub's `standard-ci.yml`), `claude.yml` (the `@claude` mention handler, agent-context kit v0.4.0), `markdown-oneline.yml` with `tools/unwrap-prose.py` (prose kit), and this file, `CLAUDE.md`, `.github/sdlc.yml` and `docs/adr/` (sdlc kit v0.2.0). Change shared logic in the hub kit, not in these copies. Two automations act on this repo besides people: a comment mentioning `@claude` starts a Claude Code run (`claude.yml`), and `markdown-oneline.yml` pushes an unwrapping fix onto pull request branches.

## Standard deviations

Departures from the standard, listed so an agent does not "fix" them; each is also in `.github/sdlc.yml` `deviations:`.

- UPS-REPO-21, UPS-QA-32: no release-please. A release is a hand-written `CHANGELOG.md` entry and `package.json` bump, then a `v*` tag; `release.yml` checks the tag against `package.json`, packages the VSIX, attaches it to a GitHub release and publishes it to the Marketplace and Open VSX ([docs/RELEASING.md](docs/RELEASING.md)).
- UPS-QA-40, UPS-REPO-07: `package-lock.json` and `site/package-lock.json` are committed, because `release.yml` and `docs.yml` install with `npm ci`, which requires them. Never edit them by hand.
