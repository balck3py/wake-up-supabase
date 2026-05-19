# Wake Up Supabase

A tiny GitHub Actions helper that prevents free Supabase projects from being automatically paused due to inactivity.

Supabase pauses inactive free-tier projects after about a week.  
If a project stays paused for 90 days, it is permanently deleted.

Supabase's 7-day inactivity timer is tracked against **database activity** —
not API requests, dashboard visits, or auth health checks. So this workflow
performs a lightweight daily query against a real table in each project via
the PostgREST data API (`/rest/v1/<table>`). That query actually hits Postgres,
which resets the timer and keeps your projects alive.

---

## 🚀 Features

- Keeps any number of Supabase projects awake  
- Works even if projects span multiple Supabase accounts  
- Stores keys only in GitHub Secrets (never in the repo)  
- Fully automated — runs once per day  
- Completely free

---

## 🛡 Security

- Uses anon keys only (never the service_role key)  
- Secrets stored securely in GitHub Actions Secrets  
- Workflow never prints keys  
- Repo contains zero sensitive information

---

## 🧩 Setup

### 1. Fork this repository  
Click the Fork button at the top right of GitHub.

---

### 2. Add your Supabase projects to a GitHub Secret

Go to:

Settings → Secrets and variables → Actions → New repository secret

Create a secret named:

    SUPABASE_PROJECTS_JSON

Paste your projects in this format:

    [
      {
        "url": "https://your-project-1.supabase.co",
        "anon_key": "YOUR_PROJECT_1_ANON_PUBLIC_KEY",
        "table": "your_table_1"
      },
      {
        "url": "https://your-project-2.supabase.co",
        "anon_key": "YOUR_PROJECT_2_ANON_PUBLIC_KEY",
        "table": "your_table_2"
      }
    ]

Add as many projects as you want.

💡 Only use anon keys — never the service_role key.  
You can find anon keys under: Supabase Dashboard → Project Settings → API

### `table` — important

The `table` value must be a **real table that the `anon` role can `SELECT`**.
The workflow runs `SELECT * ... LIMIT 1` against it, and that query is what
actually keeps the project awake.

- The table must exist and be exposed via the API.
- Its Row Level Security (RLS) policy must allow the `anon` role to read it,
  otherwise the request returns `401`/`403`.
- If you don't already have a suitable table, create a tiny one (e.g.
  `keep_alive`) with one row and an RLS policy granting `SELECT` to `anon`.

---

### 3. Enable GitHub Actions

Go to the Actions tab → enable workflows if required.

---

### 4. (Optional) Run manually once

Actions tab → select the workflow → Run workflow.

---

## ⏱ How It Works

This action runs every day at 06:00 UTC.

For each project in your SUPABASE_PROJECTS_JSON secret, it:

1. Builds the data-API URL  
   `https://your-project.supabase.co/rest/v1/<table>?select=*&limit=1`

2. Sends the anon key in both the `apikey` and `Authorization: Bearer` headers  
3. Runs that `SELECT ... LIMIT 1` query and checks the HTTP status  

That query runs against Postgres, which is the activity Supabase tracks — so
the inactivity timer is reset and the project is not paused.

A failing project never aborts the run: a missing field, a curl error, or a
non-2xx status is logged and the loop moves on to the next project.

> ⚠️ This workflow **prevents** pausing — it cannot wake an already-paused
> project. If a project is already paused, restore it once from the Supabase
> dashboard; the daily query then keeps it awake from there on.

---

## 🧪 Example Log Output

    📦 Found 2 Supabase project(s).
    🌐 Querying DATABASE via: https://abc123.supabase.co/rest/v1/keep_alive
    ✅ https://abc123.supabase.co responded with HTTP 200 — database query succeeded, timer reset.
    🌐 Querying DATABASE via: https://def456.supabase.co/rest/v1/keep_alive
    ✅ https://def456.supabase.co responded with HTTP 200 — database query succeeded, timer reset.

---

## ❓ FAQ

**Why query a table instead of pinging `/auth/v1/health`?**  
Supabase's inactivity timer only counts **database activity**. The auth health
endpoint never touches Postgres, so pinging it does not reset the timer and the
project still gets paused. A `SELECT` on a real table is an actual database
query, so it does.

**Does this work for multiple Supabase accounts?**  
Yes — simply add all your projects (url, anon_key, table) to the secret.

**Can this repo be public?**  
Yes — all sensitive data stays inside GitHub Secrets.

**Does this violate Supabase rules?**  
No. You're using public anon endpoints exactly as intended.

---

## ❤️ Contribute

This project is intentionally minimal.  
Feel free to open issues or PRs to add features like:

- Slack / Discord alerts  
- Error notifications  
- JSON schema validation  
- Multi-region health checks  
- Optional logging dashboard  

---

Enjoy!  
@wilhelmsendk
