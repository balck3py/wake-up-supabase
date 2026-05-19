# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single GitHub Actions workflow that keeps free-tier Supabase projects from being auto-paused. There is no application code, build step, or test suite — the entire repo is `.github/workflows/wake-up-supabase.yml`.

## Architecture

`wake-up-supabase.yml` runs daily (`cron: "0 6 * * *"`, 06:00 UTC) or on manual `workflow_dispatch`. It:

1. Validates the `SUPABASE_PROJECTS_JSON` repository secret exists.
2. Parses that secret with `jq` — it is a JSON array of `{ "url", "anon_key", "table" }` objects.
3. For each project, curls `<url>/rest/v1/<table>?select=*&limit=1` with `apikey` and `Authorization: Bearer` headers.

Key invariants when editing the workflow:

- The ping **must hit the PostgREST data API** (`/rest/v1/<table>`), not `/auth/v1/health`. Supabase's 7-day inactivity timer tracks *database* activity only — auth health checks never reset it. The `/rest/v1` query executes against Postgres, which is what keeps the project alive.
- Use **anon keys only**, never the `service_role` key.
- The job must never print key values to logs.
- A failure on one project (bad URL, curl error, non-2xx) must not abort the loop — `set +e`/`set -e` brackets the curl call and `continue` skips bad entries. Preserve this fault isolation.

## Testing changes

There is no local test harness. To verify a workflow change, push it and trigger a run from the **Actions** tab via "Run workflow" (`workflow_dispatch`). Confirm `SUPABASE_PROJECTS_JSON` is set on the repo's secrets first, or the run fails at the validation step by design.
