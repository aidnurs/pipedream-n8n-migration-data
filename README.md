# Pipedream workflows in n8n: 92 public workflows, imported and run

Pipedream Workflows stops on March 31, 2027. We wanted numbers on what actually happens when real Pipedream workflows
are moved to n8n, so we collected every public GitHub Sync workflow we could find, converted each one, imported it
into n8n 2.44.5 and ran it.

- Data: [`data/workflows.csv`](data/workflows.csv) (same in [`data/workflows.json`](data/workflows.json)), one row per workflow
- Sources: [`data/sources.csv`](data/sources.csv), 14 repositories pinned to the commit we read
- Code node probe: [`probe/code-node-globals.json`](probe/code-node-globals.json), import it into your own n8n to check
- Write-up: https://moveworkflows.com/pipedream-workflows-in-n8n-data

Run date: October 22, 2026.

## The corpus

GitHub code search for `"schema: workflow/2022.04" filename:workflow.yaml` (the GitHub Sync format) returned 95 files.
3 are test fixtures of our own converter and are excluded. That leaves **92 workflows from 14 repositories**.
We store links to the files at a fixed commit, not the code itself.

| | Count |
|---|---|
| Workflows | 92 (8 of them have 0 steps) |
| Steps | 391, median 3.5 per workflow, max 17 |
| Code steps | 266 (68% of all steps) |
| Python steps | 42, in 21 workflows |
| Workflows importing npm packages | 29 (`snowflake-sdk` in 15) |
| Control-flow blocks (if/else, switch) | 0 |

What the code in these workflows uses (workflows, out of 92):

| Pattern | Workflows |
|---|---|
| `fetch(` | 31 |
| `axios` | 29 |
| `$.respond(` | 27 |
| `$.flow.*` | 20 |
| `AbortController` | 18 |
| `process.env` | 15 |
| data stores | 12 |
| `new URL(` / `URLSearchParams` | 9 |
| Node built-ins (`crypto`, `fs`, …) | 9 |
| `/tmp` files | 6 |

Trigger kinds, read from the trigger id prefix in `workflow.yaml` (some workflows have several):
HTTP (`hi_`) 50, schedule (`ti_`) 24, deployed source (`dc_`) 21, subscription (`sub_`) 8.
GitHub Sync does not store trigger settings, so URLs, intervals and app trigger options have to be set again.

## Import

The 84 non-empty workflows were converted with moveworkflows (converter version of October 22, 2026) and imported
with `n8n import:workflow` into `n8nio/n8n:2.44.5` (Docker). **84 of 84 imported without errors.**

## Run

Each workflow was executed once with `n8n execute`, in a container with no network. The converted Webhook and Schedule
triggers were swapped for a Manual Trigger, because the CLI cannot start a workflow from a webhook. So every run starts
with no input data, no credentials and no network. Most failures below are caused by that setup, not by the conversion.
The `run` and `first_error` columns hold the first error of each workflow.

| Result | Workflows | What it means |
|---|---|---|
| Ran to the end | 8 | |
| Needs real trigger data | 20 | code expects a webhook body or event (`missing 'query' in request body`) |
| Python runner missing | 16 | the plain `n8nio/n8n` image has no Python; self-hosted n8n needs the `n8nio/runners` sidecar |
| Module disallowed | 13 | npm packages and Node built-ins (`snowflake-sdk` ×4, `crypto` ×4, `date-fns`, `fast-xml-parser`, `@atproto/api`, …) |
| No network | 10 | expected, network was off |
| No credentials | 8 | expected, credentials have to be created in n8n |
| Step left for manual conversion | 6 | app actions the converter does not map; flagged red before import |
| HTTP Request without URL | 1 | the URL was empty in the Pipedream source |
| Dynamic `import()` | 1 | not supported in the Code node sandbox |
| Timeout (60 s) | 1 | |

## Code node globals in n8n 2.44.5

Probe with the default JavaScript task runner (`probe/code-node-globals.json`):

| Undefined | Available |
|---|---|
| `fetch`, `URL`, `URLSearchParams`, `AbortController`, `structuredClone`, `Blob`, `process`, global `crypto` | `require`, `Buffer`, `setTimeout`, `TextEncoder`, `atob`, `btoa`, `FormData`, `this.helpers.httpRequest` |

On the plain `n8nio/n8n` image `require('crypto')`, `require('fs')`, `require('moment')` and `require('lodash')` all
return "Module … is disallowed" until you allow them (`NODE_FUNCTION_ALLOW_BUILTIN`, `NODE_FUNCTION_ALLOW_EXTERNAL`).
n8n Cloud allows `crypto` and `moment`.

## Columns

`id, repo, path, url` · `name` (workflow name) · `steps, code_steps, python_steps, empty` · `trigger_kinds` ·
`npm` (packages, `|`-separated) · `fetch … tmp_files` (pattern found in the code, true/false) ·
`light, manual_items, hours_left, hours_by_hand, red_reasons` (moveworkflows report: green/yellow/red, open manual items,
estimated hours after conversion and by hand) · `run, first_error` (result of the n8n run).

## Limits

- Public GitHub Sync repos are not a random sample of Pipedream users: 3 repos hold 67 of the 92 workflows,
  and none of them uses if/else or switch blocks.
- GitHub Sync files differ from the project export (Project Settings → Export Workflows): no trigger settings,
  no connected accounts.
- One offline run per workflow shows the first error only. A workflow that fails on missing input may fail again later.

## Disclosure and license

Made by the author of [moveworkflows.com](https://moveworkflows.com), a paid Pipedream → n8n converter.
The converter itself is not in this repository.

Data and text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The workflows belong to their authors;
this repository links to them and does not copy their code.
