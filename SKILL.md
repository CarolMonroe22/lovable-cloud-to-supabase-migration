---
name: lovable-cloud-migration
description: Guide for migrating from Lovable Cloud to your own Supabase project. Use when users ask about exporting data from Lovable Cloud, removing or disconnecting Lovable Cloud, pausing Lovable Cloud, restoring a Lovable Cloud .backup export, migrating to their own Supabase, getting data out of managed Lovable database, switching from Lovable Cloud to external Supabase, connecting their own Supabase to a Lovable project, or Lovable Cloud limitations (no SQL editor access, no custom auth emails, no direct Supabase dashboard access, no service role key).
license: MIT
metadata:
  author: Carol Monroe - Lovable Champion and Supabase SupaSquad Member
  author_url: https://carolmonroe.com
  author_github: CarolMonroe22
  version: "4.0.0"
  tested: "2026-07-06"
  tags:
    - supabase
    - lovable
    - migration
    - database
    - lovable-cloud
    - mcp
    - export
---

# Lovable Cloud to Own Supabase Migration

> Created by **Carol Monroe** - Lovable Champion and Supabase SupaSquad Member
> Every step verified on a real end-to-end migration (12 tables, 237 rows, 23 users,
> 40 storage files, 5 edge functions, 2 cron jobs). Last verified 2026-07-06.

## What Changed in v4 (July 2026)

Lovable shipped official **Export**, **Pause**, and **Remove** buttons for Cloud
(Cloud tab > Overview > Advanced settings). **Lovable Cloud is no longer permanent.**
This rewrites the migration playbook:

- The old "Cloud can never be disconnected" fact is obsolete. You can now export
  your database, remove Cloud from the SAME project, and connect your own Supabase.
- The native export file (a `pg_dump` custom-format backup) is now the PRIMARY data
  source. It carries things the MCP flow had to reconstruct by hand, including
  auth users WITH password hashes and identities.
- The old 68-step MCP migration into a fresh Lovable project is now the FALLBACK
  path, kept for the cases the native export cannot serve.

## What This Skill Does

Migrates an entire Lovable Cloud project to your own Supabase: database schema,
data, RLS policies, functions, triggers, sequences, auth users with original
passwords and identities, storage buckets and files, edge functions with correct
per-function verify_jwt, cron jobs, and secrets inventory. Then removes Cloud and
connects the user's own Supabase to the same Lovable project.

## Decision Point (ALWAYS start here)

Ask what the user actually needs, then pick the path:

| User's situation | Path |
|---|---|
| "My app burns credits while I'm not working on it" | No migration. **Pause Cloud** (Cloud tab > Overview > Advanced settings). Done. |
| "I want backups of my Cloud data" | No migration. **Export project data** works standalone, once per 24h, keep building on Cloud. |
| Ready to run on own Supabase, database ≤ 5 GB | **SAME-PROJECT PATH** (primary, below): Export + Remove + Connect on the existing Lovable project. |
| Database > 5 GB, export unavailable/failing, or user wants the original Cloud project kept untouched | **FRESH-PROJECT PATH** (legacy fallback): full MCP migration into a new Lovable project. See [references/fresh-project-path.md](references/fresh-project-path.md). |

Remind the user: Export/Pause/Remove buttons cost no credits. Only agent prompts do.

## Why Migrate at All (the right reasons)

Lovable Cloud is a solid managed backend. Migration makes sense when the project
needs things only a full Supabase setup provides:

| You want to... | Why it needs your own Supabase |
|---|---|
| Customize auth emails (sender, design, content) | Cloud sends from `no-reply@auth.lovable.cloud` |
| Access the full Supabase dashboard | SQL editor, extensions, logs, monitoring |
| Use the service role key | Admin operations and some edge functions |
| Have staging + production environments | Cloud is one database per project |
| Own your infrastructure long-term | Your project, your org, your billing |

If none of these apply and the user is happy on Cloud, say so: there is no need
to migrate, and Export-as-backup + Pause cover most worries now.

## Prerequisites

| Requirement | Needed for | How to set up |
|---|---|---|
| Lovable account with Cloud enabled | Everything | - |
| Supabase account | Everything | https://supabase.com |
| GitHub connected to the project | Everything (function code travels via the repo) | Editor > + menu > GitHub > Connect |
| **Lovable MCP** | Strongly recommended - it makes life EASY: the baseline inventory, config.toml reading, and every source-side check become one tool call instead of manual dashboard reading | `/mcp` in Claude Code, or claude.ai Settings > Connectors - https://docs.lovable.dev/integrations/mcp-servers |
| Supabase MCP | Destination-side automation (create project, run every verification) | Built into Claude Code - https://supabase.com/docs/guides/getting-started/mcp |
| pg_restore 16+ with zstd | Same-project path Step 14 | macOS: `brew install postgresql@18` |
| GitHub CLI (`gh`) | Fresh-project path only | `brew install gh` + `gh auth login` |
| Supabase CLI | Optional - faster function deploys and storage uploads | https://supabase.com/docs/guides/getting-started |

Works in Claude Code and claude.ai, and also in other MCP-compatible agents like
Cursor. Without any MCP, the same-project path still works - the user reads the
counts off the Cloud panel by hand and Claude guides the terminal steps.

## Choose Migration Level

Ask the user which level they want before starting:

| Level | Audience | Behavior |
|---|---|---|
| **Guided** | First migration, not technical | Explain each phase, confirm before proceeding, show what is happening |
| **Standard** | Knows Supabase/Lovable | Confirm key decisions, run phases with minimal pauses |
| **Express** | Advanced, done it before | Ask for source + destination, run everything, report at the gate |

Every level stops at the same hard checkpoints: cost confirmation, the Phase 7
gate, and the Remove click.

## Key Facts (verified 2026-07-06 on a live migration)

### The native export
- Format: `pg_dump` v18 **custom format** with **zstd compression** (source Postgres 17).
- Restoring requires `pg_restore` v16+ **built with zstd**. The libpq build from
  Homebrew does NOT include zstd and fails with "does not support compression with
  zstd" even when the version number looks fine. Use `postgresql@18` (Trap 18).
- INCLUDED: full schema (public, auth, storage, cron, vault), all table data, RLS
  policies, triggers, custom functions, sequences WITH current values,
  `auth.users` + `auth.identities` WITH bcrypt password hashes, `cron.job` rows,
  `storage.buckets` + `storage.objects` rows (METADATA ONLY).
- NOT included: storage FILES (actual bytes), edge function CODE (lives in the
  repo), edge function SECRET values, vault secret VALUES (rows restore but are
  encrypted with the old project's key, unrecoverable cross-project).
- The export saves INTO the project's own Cloud storage as a bucket named like
  `database_export_06_07_26`. **Download it before removing Cloud** - it dies with
  Cloud (Trap 17). Limits: 5 GB, one export per 24 hours.
- The "we'll email you" toast is unreliable. Don't wait for the email: check
  Storage for the export bucket (~1 minute for a small database).

### The Lovable side
- After connecting an own Supabase, the Lovable agent CAN deploy edge functions to
  it (confirmed directly with the agent, config.toml respected). It CANNOT set
  edge function secrets - only the user can, in the Supabase dashboard.
- `LOVABLE_API_KEY` is a managed WORKSPACE secret, independent of Cloud. It
  SURVIVES Cloud removal. AI features keep working after re-wiring (Trap 25).
- Connecting the Supabase integration auto-rewrites `.env` (correct URL + key, no
  manual editing). But it may also overwrite `src/integrations/supabase/client.ts`
  with a template that breaks SSR on modern (TanStack) stack projects (Trap 23).
- GitHub sync is two-way. Temporary helper functions you delete in Supabase come
  BACK if they still live in the repo - delete them everywhere (Trap 24).
- Remix still does NOT work for migration - it inherits Lovable Cloud.

### Security
- The `.backup` file contains password hashes and personal data. Treat it like a
  password: keep it local, never commit it, delete it after the migration.
- If the database password touched a chat, a script, or an AI session during the
  migration, rotate it at the end (Dashboard > Settings > Database).
- Before pushing any migration artifacts to a public repo, scan them for project
  refs, keys, emails, and personal data.

## THE SACRED ORDER

```
1. EXPORT      the database (button)
2. DOWNLOAD    the export + the storage files    ← they live inside Cloud
3. BUILD       the new Supabase (restore, fix, upload, deploy)
4. VERIFY      the 12-count gate — ALL GREEN or stop
5. REMOVE      Lovable Cloud                     ← only now, nothing before this is destructive
6. CONNECT     your own Supabase to the same project
```

Nothing is removed until its replacement is alive and verified. At every step
before 5, the app still runs on Cloud, untouched. If any step fails, stop and fix -
Cloud is the safety net until the gate is green.

## SAME-PROJECT PATH (primary, 33 steps)

### Phase 1: Baseline + GitHub - Steps 1-4

```
Step 1: Connect GitHub (HUMAN if not already connected)
  Editor > + menu next to chat input > GitHub > Connect.
  Lovable creates a repo and keeps it in two-way sync.
  The export file does NOT carry edge function code - the repo does.
  Verify: gh repo view {user}/{repo} or Lovable MCP get_project (latest SHA).
  Save: repo_url, latest_sha

Step 2: Run the 12-count baseline inventory
  Lovable MCP: query_database
  WITH counts AS (
    SELECT 'tables' AS category, COUNT(*) AS n FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE'
    UNION ALL SELECT 'enums', COUNT(*) FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid WHERE n.nspname='public' AND t.typtype='e'
    UNION ALL SELECT 'rls_policies', COUNT(*) FROM pg_policies WHERE schemaname='public'
    UNION ALL SELECT 'functions_custom', COUNT(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public' AND p.proname NOT LIKE 'pg_%'
    UNION ALL SELECT 'triggers', COUNT(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth') AND trigger_name NOT LIKE 'RI_%'
    UNION ALL SELECT 'sequences', COUNT(*) FROM pg_sequences WHERE schemaname='public'
    UNION ALL SELECT 'indexes_custom', COUNT(*) FROM pg_indexes WHERE schemaname='public' AND indexname NOT LIKE '%_pkey' AND indexname NOT LIKE '%_key'
    UNION ALL SELECT 'auth_users', COUNT(*) FROM auth.users
    UNION ALL SELECT 'auth_identities', COUNT(*) FROM auth.identities
    UNION ALL SELECT 'storage_buckets', COUNT(*) FROM storage.buckets
    UNION ALL SELECT 'storage_objects', COUNT(*) FROM storage.objects
  )
  SELECT * FROM counts ORDER BY category
  Also capture: per-table row counts, extensions (pg_extension), cron jobs
  (SELECT jobid, jobname, schedule, command FROM cron.job), vault secret NAMES
  only, Cloud secret NAMES (Cloud tab > Secrets).
  Save: baseline (this is the Phase 7 gate checklist)

  No MCP? The Cloud panel shows the same numbers: Database, Users, Storage,
  Functions, Secrets, Jobs. Write them down by hand.

Step 3: Map edge functions and verify_jwt
  Read supabase/config.toml from the repo at latest_sha.
  Parse [functions.<name>] sections -> verify_jwt map (default true).
  List supabase/functions/ directories, note _shared/ if present.
  Grep all function files for Deno.env.get("...") -> secrets inventory,
  excluding auto-provided SUPABASE_URL / SUPABASE_ANON_KEY /
  SUPABASE_SERVICE_ROLE_KEY / SUPABASE_DB_URL.
  Save: verify_jwt_map, function_names, secrets_per_function

Step 4: Present the baseline to the user
  Show the 12 counts + functions + secrets + cron jobs as a table.
  This exact table gets re-checked at the gate (Step 26).
```

### Phase 2: Export + Download - Steps 5-8

```
Step 5: Trigger the export (HUMAN)
  Cloud tab > Overview > Advanced settings > Export project data > Export data.

Step 6: Find the export (don't wait for the email)
  The email/toast is unreliable. After ~1 minute, check Cloud tab > Storage for
  a new bucket named database_export_DD_MM_YY with a .zip inside.
  Or via Lovable MCP: query_database
  SELECT bucket_id, name FROM storage.objects WHERE bucket_id LIKE 'database_export%';

Step 7: Download and verify the export (HUMAN downloads, agent verifies)
  Download the .zip, unzip it -> a .backup file.
  Verify readability and take stock:
  pg_restore --list your-export.backup | head -50
  pg_restore --list your-export.backup | grep -c "TABLE DATA"
  If pg_restore errors on zstd here, fix the tooling NOW (Step 13) before
  going further.
  Save: backup_path, toc_summary

Step 8: Download the storage files
  Files always travel: Cloud storage -> local machine -> new Supabase storage.
  One local folder per bucket, same names, keep subfolder structure.
  SKIP the database_export_* bucket (already downloaded, must not move over).
  Few files: Cloud tab > Storage > per-file menu > Download.
  Many files: script it - public buckets via public URLs from storage.objects,
  private buckets via signed URLs (see references/migrate-storage-function.md).
  Verify: local file count == baseline storage_objects (minus export bucket).
```

### Phase 3: Create the Destination - Steps 9-12

```
Step 9: Organization + cost
  Supabase MCP: list_organizations -> organization_id
  Supabase MCP: get_cost -> tell the user the price -> confirm_cost -> id

Step 10: Create the project
  Supabase MCP: create_project (region: us-west-1 default; us-east-1 has had
  capacity outages - retry on us-west-1 if creation fails).
  Save: new_ref. Generate a strong DB password, save it somewhere safe.

Step 11: Wait for ACTIVE_HEALTHY
  Supabase MCP: get_project, poll every 15s (30-90s typical).
  Save: anon_key (get_publishable_keys).

Step 12: Build the connection string
  Use the SESSION POOLER string (Dashboard > Connect), port 5432:
  postgresql://postgres.[new_ref]:[password]@[pooler-host]:5432/postgres
  - Special characters in the password must be percent-encoded (! -> %21).
  - "Direct connection" is IPv6-only, which many home networks can't reach.
    The session pooler exists exactly for this.
  Save: conn_string
```

### Phase 4: Restore - Steps 13-16

```
Step 13: Verify pg_restore can read zstd (Trap 18)
  pg_restore --version   (need 16+, BUILT WITH zstd)
  Test against the real file: pg_restore --list <backup_path> > /dev/null
  If "unsupported compression method" or zstd errors on macOS:
    brew install postgresql@18
    use /opt/homebrew/opt/postgresql@18/bin/pg_restore explicitly
  (The brew libpq keg does NOT ship zstd support. Version number alone lies.)

Step 14: Restore
  pg_restore --no-owner --no-privileges -d "<conn_string>" <backup_path>
  EXPECTED: a wall of errors (hundreds is normal - the verified test run printed 290).
  The new project already has managed auth/storage/cron/vault foundations; the
  backup tries to recreate them and gets "already exists" / "permission denied".
  Data lands fine around them. Capture stderr to a log anyway:
  pg_restore ... 2> restore-errors.log

Step 15: Fix auth.identities (Trap 19 - FK ordering)
  Supabase MCP: execute_sql
  SELECT (SELECT count(*) FROM auth.users) AS users,
         (SELECT count(*) FROM auth.identities) AS identities;
  If identities < users (the restore loads identities before users and the FK
  rejects them):
  pg_restore --data-only -n auth -t identities -d "<conn_string>" <backup_path>
  Re-run the count. users == identities == baseline, or stop.

Step 16: Snapshot awareness (Trap 26)
  The restore reflects the database AT EXPORT TIME. Any data changed on Cloud
  after the export (including auth metadata cleanups) is NOT in the file.
  If the app kept running after the export, either re-export fresh or re-apply
  the deltas. Ask the user when the export was taken vs when writes stopped.
```

### Phase 5: Post-Restore Fixups - Steps 17-21

```
Step 17: Recreate cron jobs + hunt zombies (Trap 20)
  cron.job rows are IN the backup but the restore cannot write them
  ("permission denied for table job" - the cron schema is protected).
  Enable pg_cron (Dashboard > Database > Extensions), then per job from baseline:
  SELECT cron.schedule('<jobname>', '<schedule>', $$<command>$$);

  THEN hunt zombies - any job whose command calls a URL still points at the
  OLD project:
  SELECT jobid, jobname, command FROM cron.job WHERE command LIKE '%supabase.co%';
  For each hit with the old ref:
  SELECT cron.alter_job(<jobid>, command := $$<command with new_ref>$$);
  Verify: SELECT count(*) FROM cron.job WHERE command LIKE '%<old_ref>%';  -- 0

Step 18: Clear storage ghost rows (Trap 21)
  The restore brought storage.buckets AND storage.objects METADATA - rows that
  point to files that don't exist yet, plus the database_export_* bucket rows.
  Ghost rows make uploads fail with "resource already exists" and make the
  export bucket reappear in the new project.
  DO NOT delete them with SQL - a protect_delete trigger (and storage
  consistency) blocks/undermines it. Use the Storage API or CLI:
  - Per object: DELETE via storage API (supabase storage rm, or the JS client
    storage.from(bucket).remove([paths]), or dashboard multi-select > Delete)
  - Drop the database_export_* bucket entirely (dashboard: Delete bucket).
  Buckets themselves stay (they're real and correctly configured by the restore).
  Verify: SELECT count(*) FROM storage.objects;  -- 0 before uploads

Step 19: Upload the storage files
  Dashboard > Storage > per bucket > Upload (keep folder structure), or CLI:
  supabase storage cp --recursive ./local-bucket-folder ss:///bucket-name -p <ref>
  Verify: SELECT bucket_id, count(*) FROM storage.objects GROUP BY bucket_id;
  matches baseline per bucket. Open one image in the browser.

Step 20: Rewrite old URLs in data (Trap 22 for signed URLs)
  Find every text/jsonb column that might hold URLs:
  SELECT table_name, column_name FROM information_schema.columns
  WHERE table_schema='public' AND data_type IN ('text','jsonb','json','ARRAY');
  For each, count LIKE '%<old_ref>%', then rewrite:
  text:  UPDATE t SET c = REPLACE(c, '<old_ref>', '<new_ref>') WHERE c LIKE '%<old_ref>%';
  jsonb: UPDATE t SET c = REPLACE(c::text, '<old_ref>', '<new_ref>')::jsonb WHERE c::text LIKE '%<old_ref>%';
  EXCEPTION - signed URLs (contain '?token='): they are cryptographically tied
  to the OLD project and die with it. Text-replacing them produces broken links
  that LOOK right. NULL them or regenerate fresh signed URLs from the new
  project; most apps regenerate on render.
  Verify: 0 rows LIKE old_ref anywhere.

Step 21: Verify sequences
  SELECT sequencename, last_value FROM pg_sequences WHERE schemaname='public';
  The custom-format dump carries SEQUENCE SET entries, so values should already
  match the source baseline. If any sequence reads 1 with existing rows, fix:
  SELECT setval('public.<seq>', <source_last_value>, true);
```

### Phase 6: Edge Functions + Secrets - Steps 22-25

```
Step 22: Deploy edge functions (pick one)
  A. Supabase CLI from the repo clone (handles _shared/ and config.toml):
     supabase functions deploy --project-ref <new_ref> --workdir <repo_path>
  B. Supabase MCP deploy_edge_function per function (only if no _shared/),
     passing verify_jwt from the Step 3 map.
  C. Defer to the Lovable agent AFTER Connect (Step 32) - it CAN deploy to the
     connected Supabase (verified). Trade-off: functions come online a few
     minutes after the switch instead of before it.
  Verify (A/B): Supabase MCP list_edge_functions == function_names count.

Step 23: Secrets (HUMAN - the agent cannot do this)
  Neither the Lovable agent nor any MCP can set edge function secrets.
  User enters them: Supabase Dashboard > Edge Functions > Secrets.
  - Names from the Step 3 inventory, EXACT capitalization.
  - Recommend FRESH keys from each provider (Stripe, Resend...) over hunting
    old values - same effort, free security hygiene.
  - Never copy SUPABASE_* keys - the new project auto-provides them.
  - VAULT secrets too: their rows rode in with the restore but the values are
    encrypted with the OLD project's key and unrecoverable. Re-enter each value
    from the baseline names list: Dashboard > Settings > Vault.

Step 24: Re-wire Lovable AI features (Trap 25)
  LOVABLE_API_KEY survives Cloud removal (workspace secret), but own-Supabase
  edge functions can't read Lovable's secret store. Two supported paths:
  A. RECOMMENDED: move AI calls into Lovable server functions (createServerFn)
     where the key is auto-injected - ask the Lovable agent to do it.
  B. Copy LOVABLE_API_KEY into the new project's edge function secrets.

Step 25: Delete temp helpers EVERYWHERE (Trap 24)
  Any temporary function deployed during migration (exporters, helpers) must be
  deleted BOTH from Supabase (dashboard) AND from supabase/functions/ in the
  repo. GitHub sync is two-way: if it stays in the repo, it comes back.
```

### Phase 7: THE GATE - Steps 26-28

```
Step 26: Re-run the 12-count audit on the DESTINATION
  Same query as Step 2, via Supabase MCP execute_sql.
  Build the comparison table:
  Category           Baseline  Destination  Status
  tables             N         N            OK / MISMATCH
  ... (all 12 rows) ...
  Plus: per-table row counts, cron job count, edge function count,
  extensions list vs baseline (SELECT extname FROM pg_extension),
  vault secret names re-entered, old_ref_leaks = 0.

Step 27: Functional checks
  - Log in with a REAL migrated user and their ORIGINAL password.
  - Open a real storage image in the browser.
  - Invoke one edge function (or accept deferred deploy, path C).

Step 28: GATE DECISION
  ALL GREEN -> proceed to Phase 8.
  ANY RED   -> STOP. Fix and re-verify. Cloud is still alive and untouched;
  nothing has been lost. Do NOT remove Cloud with a red gate.
  Point each mismatch at its fix:
    identities -> Step 15 | cron -> Step 17 | storage counts -> Steps 18-19
    old refs -> Step 20 | sequences -> Step 21 | functions -> Step 22
```

### Phase 8: Remove + Connect - Steps 29-33

```
Step 29: Remove Lovable Cloud (HUMAN)
  Cloud tab > Overview > Advanced settings > Remove Lovable Cloud.
  Two checkboxes + type the project name. This deletes the Cloud instance
  INCLUDING its storage (and any un-downloaded exports).
  There is no rush - some users sit at the green gate for days. That's fine.

Step 30: Connect the own Supabase (HUMAN)
  The Cloud tab transforms. At the bottom: "Already have a Supabase project?
  Connect it here." > authorize the Supabase org > pick the new project > Connect.
  Lovable auto-rewrites .env with the new URL + key (visible in GitHub history).

Step 31: Fix the integration overwrite (Trap 23)
  The integration may also refresh src/integrations/supabase/client.ts (or
  equivalent) with a template. On modern (TanStack/SSR) stacks this can break
  the build/preview - an error page right after connecting is THIS, not data loss.
  Send the Lovable agent:

    Review my Supabase connection: confirm the app points to my new Supabase
    project, check that auth, database queries, storage and edge functions all
    work against it, and list anything still referencing the old backend.

    Important context: the database, users, storage files and edge functions
    ALREADY EXIST in the new project, they were migrated. Do NOT create tables,
    run migrations, re-seed data, or rewrite files beyond the connection wiring.
    Fix wiring only.

  The Do NOTs matter: the agent is eager and a bare "fix my connection" can
  trigger re-migrations or file rewrites.

Step 32: Deferred function deploy (only if path C in Step 22)
  Send the Lovable agent:
    Deploy all edge functions from supabase/functions to the connected Supabase
    project, keeping the verify_jwt settings from supabase/config.toml.
    Don't modify their code.

Step 33: Final tidy
  - Rotate the DB password if it touched any chat/script/AI session.
  - Recreate OAuth providers (Google etc.): client ID/secret + redirect URI
    https://<new_ref>.supabase.co/auth/v1/callback + Site URL.
  - Delete the local .backup (it holds password hashes) once verified.
  - Optional: JWT secret copy (dashboard-only) if existing sessions must survive.
  - If external workers (Fly.io, Railway, ...) point at the old URL, re-point them.
```

## FRESH-PROJECT PATH (legacy fallback)

The complete v3.1 flow - 68 deterministic steps, 9 phases, MCP-driven scan and
rebuild into a NEW Lovable project - lives in
[references/fresh-project-path.md](references/fresh-project-path.md).

Use it when:
- The database exceeds the 5 GB export cap (also: email support@lovable.dev with
  the project ID for a manual export).
- The export feature is unavailable or failing for the project.
- The user wants the original Cloud project kept running/untouched (staging-style
  migration with zero risk to the original).

That reference keeps its own trap table (Traps 1-16) and troubleshooting rows.

## Traps Added in v4 (17-26, all hit and verified live)

| Trap | Severity | Description | Prevention |
|---|---|---|---|
| 17 | Critical | Export saves INTO Cloud storage - Remove deletes it | Sacred order: Export -> DOWNLOAD -> Remove last (Steps 5-7, 29) |
| 18 | Blocker | brew libpq pg_restore lacks zstd, fails on the .backup | Test with pg_restore --list first; postgresql@18 (Step 13) |
| 19 | Critical | auth.identities dropped by restore FK ordering | Count check + second data-only pass (Step 15) |
| 20 | Silent bug | cron.job restore = permission denied; restored/recreated commands point at OLD project URL and run against a dead endpoint | cron.schedule + zombie hunt + cron.alter_job (Step 17) |
| 21 | Silent bug | storage.objects ghost rows block uploads ("resource already exists"); protect_delete blocks SQL DELETE | Clear via Storage API/CLI before upload, never SQL (Step 18) |
| 22 | Silent bug | Signed URLs (?token=) die with the old project; text-replace makes them LOOK migrated | Regenerate or NULL, never rewrite (Step 20) |
| 23 | Breaker | Supabase integration overwrites client.ts, breaks SSR on modern stack | Post-connect fix prompt with explicit Do NOTs (Step 31) |
| 24 | Zombie | Temp helper functions replicate back via two-way GitHub sync | Delete in Supabase AND in the repo (Step 25) |
| 25 | Surprise (good) | LOVABLE_API_KEY survives Cloud removal (workspace secret) | Re-wire: server functions (recommended) or copy the key (Step 24) |
| 26 | Data loss | Restore is a snapshot at EXPORT time; later writes are missing | Ask when writes stopped; re-export or re-apply deltas (Step 16) |

## Human Assistance Points (same-project path)

| Step | What the user must do | Why not automated |
|---|---|---|
| 1 | Connect GitHub in the editor | Dashboard-only flow |
| 5 | Click Export project data | Dashboard-only button |
| 7 | Download the export zip | Browser download |
| 8 | Download storage files (small projects) | Browser download (scriptable for large) |
| 23 | Enter edge function secrets | No MCP/agent can set secrets |
| 29 | Click Remove Lovable Cloud | Destructive, deliberately manual |
| 30 | Connect own Supabase | Dashboard OAuth flow |
| 33 | OAuth providers, JWT secret | Dashboard-only settings |
| Cost | Confirm Supabase project cost | MCP requires explicit confirmation |

## Things That Can Go Wrong (v4 additions)

| Symptom | Likely cause | Fix |
|---|---|---|
| "does not support compression with zstd" on restore | brew libpq build (Trap 18) | postgresql@18 binaries |
| Hundreds of errors scroll during pg_restore | Managed schemas already exist - EXPECTED | Nothing; verify counts after |
| identities count < users count after restore | FK ordering (Trap 19) | pg_restore --data-only -n auth -t identities |
| "permission denied for table job" during restore | cron schema is protected (Trap 20) | Recreate via cron.schedule |
| Cron jobs run but nothing happens | Zombie commands hitting the OLD project URL (Trap 20) | cron.alter_job with new URL |
| Upload fails "The resource already exists" | Ghost storage.objects rows (Trap 21) | Delete via Storage API/CLI, then upload |
| DELETE FROM storage.objects fails/blocked | protect_delete trigger (Trap 21) | Storage API/CLI, not SQL |
| database_export_* bucket appears in the NEW project | Its metadata rows rode in with the restore | Delete the bucket in the new project |
| Images broken only where URLs have ?token= | Signed URLs from the old project (Trap 22) | Regenerate signed URLs, don't text-replace |
| Error page right after Connect | client.ts overwritten by integration template (Trap 23) | Step 31 fix prompt - wiring only |
| Deleted helper function reappears | Still in the repo, GitHub sync restored it (Trap 24) | Delete from supabase/functions/ too |
| AI features dead after migration | Edge functions can't read Lovable's secret store (Trap 25) | Server functions or copy LOVABLE_API_KEY |
| Recent user/data changes missing in new project | Export predates them (Trap 26) | Re-export or re-apply deltas |
| Export email never arrives | Toast/email unreliable | Check Storage for database_export_* bucket |

Legacy symptoms (fresh-project path) remain in
[references/fresh-project-path.md](references/fresh-project-path.md) and
[references/troubleshooting.md](references/troubleshooting.md).

## Plan Requirements

| Tool | Plan needed | Cost |
|---|---|---|
| Lovable | Any plan with Cloud | Varies |
| Supabase | Free works if slot available | $0-$10/mo |
| Claude Code (optional, automates everything) | Pro or Max | $20/mo+ |
| claude.ai (optional, no-CLI route) | Free/Pro | $0+ |
| GitHub | Free | $0 |

## Documentation Links

| Resource | URL |
|---|---|
| Lovable Cloud (export, pause, remove) | https://docs.lovable.dev/integrations/cloud |
| Lovable GitHub integration | https://docs.lovable.dev/integrations/github |
| Lovable MCP | https://docs.lovable.dev/integrations/mcp-servers |
| Supabase: restore a backup | https://supabase.com/docs/guides/platform/migrating-within-supabase/dashboard-restore |
| Supabase: database backups | https://supabase.com/docs/guides/platform/backups |
| Supabase MCP | https://supabase.com/docs/guides/getting-started/mcp |
| Supabase Edge Functions | https://supabase.com/docs/guides/functions |
| Supabase Storage | https://supabase.com/docs/guides/storage |
| Supabase Auth | https://supabase.com/docs/guides/auth |

## Resources by Carol Monroe

- The manual walkthrough this skill automates:
  https://carolmonroe.com/blog/export-remove-lovable-cloud
- Migrate with AI guide (claude.ai route + Claude Code route):
  https://carolmonroe.com/blog/migrate-lovable-cloud-with-ai
- Legacy MCP migration guide (fresh-project path):
  https://carolmonroe.com/blog/lovable-cloud-mcp-migration
- MCP setup for claude.ai chat:
  https://carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude
