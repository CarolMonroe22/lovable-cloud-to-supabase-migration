# Same-Project Path - The Primary Migration (33 Steps)

> The v4 primary flow: official Export + Remove + Connect on the SAME Lovable
> project. Read SKILL.md first for the decision point, the sacred order, and
> the hard stops. This file is the operational playbook.

> Auth note, updated 2026-08-31: Lovable's official export documentation now
> says passwords are not exported in a usable form. Use the reset path by
> default. The separate MCP hash-preservation option is documented in
> `export-methods.md` and requires a capability check plus explicit approval.


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

  Do NOT treat matching user counts as proof that original passwords work.
  Choose the auth path now, while Cloud is still available:
  A. OFFICIAL DEFAULT: configure and test password recovery in the destination.
     Tell migrated users they must set a new password.
  B. ADVANCED MCP OPTION: follow references/export-methods.md Method 1. Start
     with the count-only capability check, explain that hashes pass through
     MCP/agent execution, obtain explicit approval, and migrate the minimum
     required auth fields. If any check fails, return to path A.

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
  - Test the chosen auth path with a REAL migrated user:
    - reset path: complete recovery, set a new password, then log in;
    - MCP hash path: log in with the original password.
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
  - Delete the local .backup and any protected auth artifact once verified.
  - Optional: JWT secret copy (dashboard-only) if existing sessions must survive.
  - If external workers (Fly.io, Railway, ...) point at the old URL, re-point them.
```


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
| Original passwords fail after restore | Current official export does not guarantee usable passwords (Trap 27) | Complete reset flow, or run the optional MCP hash route before removing Cloud |

Legacy symptoms (fresh-project path) remain in
[fresh-project-path.md](fresh-project-path.md) and
[troubleshooting.md](troubleshooting.md).
