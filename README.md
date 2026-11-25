# Supabase Keepalive Template

Keeps a free-tier Supabase project from being paused for inactivity, using one SQL file and one GitHub Actions workflow. Reusable in any project.

## What is in this template

```
supabase/keepalive.sql                    run once in the Supabase SQL Editor
.github/workflows/supabase-keepalive.yml the only workflow file you need
.github/keepalive/last-run.txt            created automatically by the workflow (one line), only when needed
```

## How it works

Every day the workflow does two independent things:

1. **Ping job** calls a database function through the Supabase REST API. The function inserts one row and trims the table to the last 5 rows, so it counts as real database activity and the table never grows.
2. **Repo-activity job** checks the date of the last commit. If the repository had a commit in the last 30 days, it does nothing. Otherwise it writes a single line to `.github/keepalive/last-run.txt` and commits it as `github-actions[bot]`. This stops GitHub from disabling scheduled workflows on inactive repositories.

## Security model

- The table has RLS enabled with **no policies**, and all privileges are revoked from `anon` and `authenticated`. The API cannot read or write it directly.
- The `anon` role (publishable key) can execute exactly one function, `zz_keepalive_ping`, which runs as `security definer` with an empty `search_path`.
- Even if someone copies the publishable key and calls the function in a loop, the table stays at 5 rows and no other table is reachable.
- Use the **publishable key** (`sb_publishable_...`) only. Never put the secret key or `service_role` key in GitHub for this.
- The workflow pings with `permissions: {}`. Only the commit job gets `contents: write`.

The names `zz_keepalive_heartbeat` and `zz_keepalive_ping` are deliberately unusual so they will not clash with your own tables. Every Supabase project has its own database, so you can reuse the same names in every project. If you ever rename the function, change it in the SQL and in `KEEPALIVE_FN` at the top of the workflow. That is the only place the name appears.

## Setup (per project, about 3 minutes)

1. **SQL:** open the project in the Supabase Dashboard, go to SQL Editor, paste `supabase/keepalive.sql`, and run it once.
2. **Secrets:** in the GitHub repo go to Settings, Secrets and variables, Actions, and add two repository secrets:
   - `SUPABASE_URL` (Project URL from Supabase, Project Settings, API)
   - `SUPABASE_PUBLISHABLE_KEY` (the publishable key from Project Settings, API Keys)
3. **Files:** copy `.github/workflows/supabase-keepalive.yml` into the repo (and `supabase/keepalive.sql` if you want it versioned).
4. **Permissions:** Settings, Actions, General. Under Workflow permissions choose "Read repository contents and packages permissions" (the commit job asks for write access on its own). Keep fork pull request approval enabled.
5. **First run:** Actions tab, Supabase Keepalive, Run workflow. Both jobs should be green.

Private repositories work too. Each job is billed as at least one minute, so the daily run uses roughly 60 minutes a month against the free private-repo allowance. Public repositories are free.

## Testing

**Test the ping**

Run the workflow manually. In the ping job log you should see `Supabase keepalive OK`. In Supabase, Table Editor, `zz_keepalive_heartbeat` shows a new row. Run it a few more times and confirm the table never exceeds 5 rows.

**Test "skip because the repo is active"**

Run the workflow manually with `commit_every_days` left at `30`. If you committed recently, the log says `Repository already active. No commit needed.` and no commit is created.

**Test "commit when inactive"**

Run the workflow manually with `commit_every_days` set to `0`. The log says `Keepalive commit pushed.` and a commit by `github-actions[bot]` appears with `.github/keepalive/last-run.txt`. Delete that file afterwards if you do not want it.

**Test the real 30-day behaviour**

The threshold is compared against the last commit date, so you cannot wait it out in a short test. Instead, run manually with `commit_every_days` set to a small number such as `1` on a repository whose last commit is older than that (commits older than 1 day commit, newer ones skip). The 30-day default uses the same code path, only with a different number.

**Check it over the long term**

After about 60 days, open the Actions tab and confirm "Supabase Keepalive" is still enabled and has a recent scheduled run. Also open your Supabase dashboard occasionally to confirm the project is not paused.

## Limitations

- **Default branch only.** GitHub runs scheduled workflows from the default branch. The workflow file must be there.
- **Protected branches.** If the default branch is protected (for example, requiring pull requests), the bot's `git push` will be rejected. Either allow `github-actions[bot]` to bypass the rule, or accept that only the ping job works.
- **Schedule timing.** GitHub may delay scheduled runs by several minutes during busy times. Daily is more than enough.
- **60-day rule.** GitHub disables scheduled workflows in repositories with no activity for 60 days. This template commits every 30 days to avoid that, but whether a bot commit always counts as activity is something to verify on your own repo (see "Check it over the long term").
- **No guarantee from Supabase.** Free-tier pausing rules are Supabase's and may change. A paid plan is the only way to be fully safe.
- **Failure alerts.** If a run fails, GitHub emails you according to your notification settings. Make sure Actions notifications are on.

## Removing it

Delete the workflow file, then in Supabase run:

```sql
drop function if exists public.zz_keepalive_ping();
drop table if exists public.zz_keepalive_heartbeat;
```
