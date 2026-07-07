# Fresh-Project Path (Legacy) - Full 68-Step MCP Migration

> This is the v3.1 migration flow, kept as the FALLBACK path in v4.0.0.
> Use it when the native export is not an option (database over 5 GB, export
> unavailable, or the user wants to keep the original Cloud project untouched
> and migrate into a NEW Lovable project instead).
> For the primary same-project path (official Export + Remove + Connect), see SKILL.md.

Prerequisites for THIS path: Lovable MCP (required - the whole scan phase runs on
query_database), Supabase MCP, GitHub CLI (`gh` + auth), git, and optionally the
Supabase CLI for bulk function deploys. See the Prerequisites table in SKILL.md.

## Migration Phases (68 Steps, Correct Order)

### Phase 1: Scan Source (Lovable Cloud) - Steps 1-18

Use Lovable MCP on the source project.
Docs: [Lovable MCP](https://docs.lovable.dev/integrations/mcp-servers)

```
Step 1: Identify source project
  Lovable MCP: list_workspaces - confirm workspace
  Lovable MCP: get_project - get latest commit SHA and project metadata
  Save: project_id, workspace_id, latest_sha

Step 2: Read package.json to detect tech stack (Trap 1)
  Lovable MCP: read_file for package.json at latest_sha
  If name === "vite_react_shadcn_ts" -> tech_stack = "classic"
  If name contains "tanstack_start" -> tech_stack = "modern"
  Default fallback: "classic" (most migrations are pre-May-2026 projects)
  Save: tech_stack

Step 3: Read config.toml for verify_jwt mapping (Trap 5)
  Lovable MCP: read_file for supabase/config.toml at latest_sha
  Parse [functions.<name>] sections to build verify_jwt map
  Example output: { "process-payment": true, "stripe-webhook": false }
  Default if not specified: true
  Save: verify_jwt_map

Step 4: List all tables
  Lovable MCP: query_database
  SELECT table_name FROM information_schema.tables
  WHERE table_schema = 'public' AND table_type = 'BASE TABLE'
  Save: table_list

Step 5: Get columns, types, defaults, constraints
  Lovable MCP: query_database
  SELECT table_name, column_name, data_type, is_nullable, column_default
  FROM information_schema.columns WHERE table_schema = 'public'
  Save: column_definitions

Step 6: Get constraints and foreign keys
  Lovable MCP: query_database
  SELECT tc.table_name, tc.constraint_name, tc.constraint_type,
         kcu.column_name, ccu.table_name AS foreign_table
  FROM information_schema.table_constraints tc
  JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
  LEFT JOIN information_schema.constraint_column_usage ccu ON tc.constraint_name = ccu.constraint_name
  WHERE tc.table_schema = 'public'
  Save: constraints

Step 7: Get custom enums
  Lovable MCP: query_database
  SELECT t.typname, e.enumlabel, e.enumsortorder
  FROM pg_type t JOIN pg_enum e ON t.oid = e.enumtypid
  JOIN pg_namespace n ON t.typnamespace = n.oid
  WHERE n.nspname = 'public'
  Save: enums

Step 8: Get RLS policies
  Lovable MCP: query_database
  SELECT tablename, policyname, permissive, roles, cmd, qual, with_check
  FROM pg_policies WHERE schemaname = 'public'
  Save: rls_policies

Step 9: Get custom functions with definitions (Trap 8)
  Lovable MCP: query_database
  SELECT p.proname AS function_name,
         pg_get_functiondef(p.oid) AS definition
  FROM pg_proc p
  JOIN pg_namespace n ON p.pronamespace = n.oid
  WHERE n.nspname = 'public'
    AND p.proname NOT LIKE 'pg_%'
  Save: custom_functions

Step 10: Get triggers (Trap 8)
  Lovable MCP: query_database
  SELECT trigger_schema, trigger_name, event_object_table,
         event_manipulation, action_timing, action_statement
  FROM information_schema.triggers
  WHERE trigger_schema IN ('public', 'auth')
    AND trigger_name NOT LIKE 'pg_%'
    AND trigger_name NOT LIKE 'RI_%'
  Save: triggers

Step 11: Get sequences with last_value (Trap 9)
  Lovable MCP: query_database
  SELECT sequencename, last_value, start_value, increment_by,
         min_value, max_value, cycle
  FROM pg_sequences WHERE schemaname = 'public'
  Save: sequences

Step 12: Get custom indexes (Trap 10)
  Lovable MCP: query_database
  SELECT schemaname, tablename, indexname, indexdef
  FROM pg_indexes
  WHERE schemaname = 'public'
    AND indexname NOT LIKE '%_pkey'
    AND indexname NOT LIKE '%_key'
  Save: custom_indexes

Step 13: Export auth users and identities (Trap 11)
  Lovable MCP: query_database
  SELECT id, email, encrypted_password, email_confirmed_at,
         raw_user_meta_data, raw_app_meta_data, created_at
  FROM auth.users ORDER BY created_at
  Save: auth_users

  Lovable MCP: query_database
  SELECT id, user_id, provider, provider_id, identity_data,
         created_at, updated_at, last_sign_in_at
  FROM auth.identities
  Save: auth_identities

Step 14: Scan storage buckets with visibility flag (Trap 4)
  Lovable MCP: query_database
  SELECT id, name, public FROM storage.buckets
  Save: storage_buckets (noting public vs private)

  Lovable MCP: query_database
  SELECT bucket_id, name, metadata->>'mimetype' AS mimetype
  FROM storage.objects
  Save: storage_objects

Step 15: Scan text/jsonb columns for URL references (Trap 7) + run audit counts
  Lovable MCP: query_database
  SELECT table_name, column_name, data_type
  FROM information_schema.columns
  WHERE table_schema = 'public'
    AND data_type IN ('text', 'ARRAY', 'jsonb', 'json')
  Save: url_candidate_columns

  For each candidate column, check for old ref:
  SELECT count(*) FROM public.<table> WHERE <column>::text LIKE '%<old_ref>%'
  Save: columns_with_old_refs

  Run comprehensive audit counts:
  WITH counts AS (
    SELECT 'tables' AS category, COUNT(*) AS n FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE'
    UNION ALL SELECT 'enums', COUNT(*) FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid WHERE n.nspname='public' AND t.typtype='e'
    UNION ALL SELECT 'rls_policies', COUNT(*) FROM pg_policies WHERE schemaname='public'
    UNION ALL SELECT 'functions_custom', COUNT(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public'
    UNION ALL SELECT 'triggers', COUNT(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth') AND trigger_name NOT LIKE 'RI_%'
    UNION ALL SELECT 'sequences', COUNT(*) FROM pg_sequences WHERE schemaname='public'
    UNION ALL SELECT 'indexes_custom', COUNT(*) FROM pg_indexes WHERE schemaname='public' AND indexname NOT LIKE '%_pkey' AND indexname NOT LIKE '%_key'
    UNION ALL SELECT 'auth_users', COUNT(*) FROM auth.users
    UNION ALL SELECT 'auth_identities', COUNT(*) FROM auth.identities
    UNION ALL SELECT 'storage_buckets', COUNT(*) FROM storage.buckets
    UNION ALL SELECT 'storage_objects', COUNT(*) FROM storage.objects
  )
  SELECT * FROM counts ORDER BY category
  Save: source_audit_counts (used as baseline for Phase 9 comparison)

Step 16: Detect database extensions (Trap 13)
  Lovable MCP: query_database
  SELECT extname, extversion FROM pg_extension
  WHERE extname NOT IN ('plpgsql')
  ORDER BY extname
  Save: source_extensions

  Common extensions in complex projects: pg_cron, pg_net, pgcrypto,
  pg_graphql, pg_vector, pgjwt, pg_stat_statements, uuid-ossp, moddatetime.
  ALL must be enabled in the destination before schema migration.

Step 17: Detect cron jobs (Trap 14)
  Lovable MCP: query_database
  SELECT jobid, schedule, command, nodename, active
  FROM cron.job
  ORDER BY jobid
  If this query fails with "relation cron.job does not exist", pg_cron is not
  enabled and there are no cron jobs to migrate. Skip this step.
  Save: cron_jobs (may be empty)

Step 18: Detect vault secrets (Trap 15)
  Lovable MCP: query_database
  SELECT name, description, created_at, updated_at
  FROM vault.secrets
  If this query fails with "relation vault.secrets does not exist", vault is not
  enabled and there are no secrets to migrate. Skip this step.
  NEVER query the secret column - only detect names.
  Save: vault_secret_names (may be empty)
```

Output a structured summary to the user before proceeding:

```
Source project: <name>
Tech stack: classic | modern
Tables: N | Enums: N | RLS policies: N
Custom functions: N (handle_new_user: yes/no)
Custom triggers: N
Custom sequences: N (with last_value for each)
Custom indexes: N
auth.users: N | auth.identities: N
Storage buckets: N (public: N, private: N)
Storage files: N
Edge functions: N (config.toml verify_jwt mapping: ...)
Columns with URL references: N columns across M tables
Extensions: N (list names)
Cron jobs: N (list schedule + command for each)
Vault secrets: N (list names only, never values)
```

If cron jobs, vault secrets, or non-standard extensions are detected, flag them:

```
⚠ Advanced components detected:
  - Cron jobs: N jobs (these must be recreated manually in the destination)
  - Vault secrets: N secrets (values must be re-entered in the destination dashboard)
  - Extensions: [list any beyond pg_net] (must be enabled before schema migration)

These components are detected and reported but not automatically migrated.
The skill will remind you at the appropriate phase.
```

### Phase 2: Create Destination (Supabase) - Steps 19-23

Use Supabase MCP.
Docs: [Supabase MCP](https://supabase.com/docs/guides/getting-started/mcp) | [Pricing](https://supabase.com/pricing)

```
Step 19: List organizations
  Supabase MCP: list_organizations - find where to create project
  Save: organization_id

Step 20: Check pricing
  Supabase MCP: get_cost - check pricing ($10/mo on paid org, free if slot available)
  Tell the user: "This will create a new Supabase project at $X/mo. Confirm to proceed."

Step 21: Confirm cost
  Supabase MCP: confirm_cost - get confirmation ID
  Save: confirm_cost_id

Step 22: Create project (Trap 2 - default us-west-1)
  Supabase MCP: create_project
    name: derive from user preference
    organization_id: from Step 19
    region: "us-west-1" (default - us-east-1 has had multiple capacity outages)
    confirm_cost_id: from Step 21

  If us-east-1 was specifically requested and create_project fails with capacity errors,
  retry with us-west-1 and inform the user.
  Save: new_project_ref, new_project_id

Step 23: Wait for project to be ready
  Supabase MCP: get_project - wait for status = ACTIVE_HEALTHY
  May take 30-90 seconds. Poll every 15 seconds.
  Save: new_anon_key, new_service_role_key
```

### Phase 3: Apply Schema - Steps 24-32

Use Supabase MCP `apply_migration`.
Order is critical: extensions -> enums -> tables -> sequences with setval -> functions -> triggers -> indexes -> RLS.

```
Step 24: Enable all detected extensions (Trap 3, Trap 13)
  Supabase MCP: apply_migration
  For each extension from source_extensions (Step 16):
  CREATE EXTENSION IF NOT EXISTS <extname> WITH SCHEMA extensions;

  pg_net is required for storage migration and is not enabled by default.
  pg_cron is required if cron jobs were detected in Step 17.
  Always enable pg_net even if not in source list (needed for Phase 6).
  Expected output: all source extensions enabled in destination

Step 25: Create all custom enums
  Supabase MCP: apply_migration
  CREATE TYPE public.<enum_name> AS ENUM ('value1', 'value2', ...);
  One CREATE TYPE per enum from Step 7.
  Expected output: types created matching source enum count

Step 26: Create all tables with columns, defaults, constraints, foreign keys
  Supabase MCP: apply_migration
  ORDER MATTERS: tables referenced by FKs must be created first.
  Typical order: profiles -> categories -> products -> orders -> junction tables
  Include column types, NOT NULL constraints, DEFAULT values, PRIMARY KEYs, UNIQUE constraints, and FOREIGN KEYs.
  Expected output: all tables created matching source table count

Step 27: Create custom sequences with correct last_value (Trap 9)
  Supabase MCP: apply_migration
  For each sequence from Step 11:
  CREATE SEQUENCE public.<sequence_name>
    START WITH <start_value>
    INCREMENT BY <increment_by>
    MINVALUE <min_value>
    MAXVALUE <max_value>
    <CYCLE | NO CYCLE>;
  SELECT setval('public.<sequence_name>', <last_value>, true);

  The "true" argument means the next nextval() returns last_value + 1.
  Expected output: sequences created with correct last_value

Step 28: Create custom functions (Trap 8)
  Supabase MCP: apply_migration
  Use the exact pg_get_functiondef output from Step 9.
  This includes critical functions like handle_new_user.
  Expected output: functions created matching source function count

  WARNING: If handle_new_user or similar auth trigger functions are missing,
  new user signups will succeed but the user gets no profile row.

Step 29: Create triggers (Trap 8)
  Supabase MCP: apply_migration
  For each trigger from Step 10:
  CREATE TRIGGER <trigger_name>
    <action_timing> <event_manipulation>
    ON <event_object_table>
    FOR EACH ROW
    EXECUTE FUNCTION <function_name>();
  Expected output: triggers created matching source trigger count

Step 30: Create custom indexes (Trap 10)
  Supabase MCP: apply_migration
  Run each indexdef directly from Step 12 (it is a complete CREATE INDEX statement).
  Example: CREATE INDEX idx_products_slug ON public.products USING btree (slug);
  Expected output: indexes created matching source custom index count

Step 31: Enable RLS and create all policies
  Supabase MCP: apply_migration
  ALTER TABLE public.<table> ENABLE ROW LEVEL SECURITY;
  CREATE POLICY "<name>" ON public.<table> FOR <cmd>
    TO <roles>
    USING (<qual>)
    WITH CHECK (<with_check>);
  Expected output: RLS policies created matching source policy count

Step 32: Verify schema migration
  Supabase MCP: execute_sql
  SELECT
    (SELECT count(*) FROM information_schema.tables WHERE table_schema='public' AND table_type='BASE TABLE') AS tables,
    (SELECT count(*) FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid WHERE n.nspname='public' AND t.typtype='e') AS enums,
    (SELECT count(*) FROM pg_proc p JOIN pg_namespace n ON p.pronamespace=n.oid WHERE n.nspname='public') AS functions,
    (SELECT count(*) FROM information_schema.triggers WHERE trigger_schema IN ('public','auth') AND trigger_name NOT LIKE 'RI_%') AS triggers,
    (SELECT count(*) FROM pg_sequences WHERE schemaname='public') AS sequences;

  Compare each count against source_audit_counts from Step 15.
  If any mismatch, stop and fix before proceeding.
```

### Phase 4: Auth Users and Identities (BEFORE data) - Steps 33-36

Use Supabase MCP `execute_sql`.
Docs: [Supabase Auth](https://supabase.com/docs/guides/auth) | [Admin API](https://supabase.com/docs/reference/javascript/admin-api)

```
Step 33: Insert auth.users with ORIGINAL encrypted_password hashes
  Supabase MCP: execute_sql
  INSERT INTO auth.users (
    id, instance_id, email, encrypted_password,
    email_confirmed_at, raw_user_meta_data, raw_app_meta_data,
    created_at, updated_at, role, aud
  ) VALUES (
    '{id}', '00000000-0000-0000-0000-000000000000',
    '{email}', '{original_encrypted_password}',  -- EXACT bcrypt hash ($2a$10$...)
    '{email_confirmed_at}', '{metadata}'::jsonb, '{app_metadata}'::jsonb,
    '{created_at}', now(), 'authenticated', 'authenticated'
  );

  NEVER generate temporary passwords. ALWAYS copy the original bcrypt hash.
  Expected output: auth.users count matches source

Step 34: Insert auth.identities (Trap 11)
  Supabase MCP: execute_sql
  INSERT INTO auth.identities (
    id, user_id, provider, provider_id, identity_data,
    created_at, updated_at, last_sign_in_at
  ) VALUES (
    '{id}', '{user_id}', '{provider}', '{provider_id}',
    '{identity_data}'::jsonb,
    '{created_at}', '{updated_at}', '{last_sign_in_at}'
  );

  Without identities, login partially works but session recovery breaks.
  Password recovery flows fail. OAuth users cannot re-link accounts.
  Expected output: auth.identities count matches source

Step 35: Verify auth migration
  Supabase MCP: execute_sql
  SELECT
    (SELECT count(*) FROM auth.users) AS users,
    (SELECT count(*) FROM auth.identities) AS identities;
  Compare against source counts from Step 13 (auth scan).

Step 36: Optional - preserve JWT secret
  If the user wants existing sessions to remain valid (no forced re-login):
  1. In source dashboard: Settings -> API -> JWT Secret -> copy
  2. In destination dashboard: Settings -> API -> JWT Secret -> paste
  This MUST be done via the dashboard - there is no API for it.
  Mention this only if relevant to the user's use case.
```

### Phase 5: Insert Data + URL Rewriting - Steps 37-41

Use Supabase MCP `execute_sql`.

```
Step 37: Insert catalog/reference tables first (no user FKs)
  Supabase MCP: execute_sql
  For each source table with no FK to auth.users or profiles:
  SELECT row_to_json(t) FROM (SELECT * FROM public.<table>) t
  Then generate INSERT statements and execute on destination.
  Batch in groups of 100 rows for large tables.
  Expected output: row counts match source for each table

Step 38: Insert user-owned tables
  Supabase MCP: execute_sql
  profiles, categories, orders, etc.
  profiles.id references auth.users.id - since we inserted auth.users
  in Phase 4, the FK is satisfied naturally. No need to drop constraints.
  Expected output: row counts match source for each table

Step 39: Insert junction/relation tables
  Supabase MCP: execute_sql
  order_items, project_members, bookings, etc.
  These depend on both catalog and user-owned tables already existing.
  Expected output: row counts match source for each table

Step 40: Rewrite URLs in ALL text/jsonb columns (Trap 7)
  Using the columns_with_old_refs from Step 15, for each column:

  For text columns:
  Supabase MCP: execute_sql
  UPDATE public.<table>
  SET <column> = REPLACE(<column>, '<old_ref>', '<new_ref>')
  WHERE <column> LIKE '%<old_ref>%';

  For jsonb columns (cast technique):
  Supabase MCP: execute_sql
  UPDATE public.<table>
  SET <column> = REPLACE(<column>::text, '<old_ref>', '<new_ref>')::jsonb
  WHERE <column>::text LIKE '%<old_ref>%';

  Do NOT only check columns named *_url or *_image.
  URLs hide in metadata, config, specs, and arbitrary JSONB fields.
  Expected output: 0 rows remaining with old ref in any text/jsonb column

Step 41: Verify data migration
  Supabase MCP: execute_sql
  For each table:
  SELECT '<table>' AS t, count(*) AS n FROM public.<table>
  Compare against row counts captured during Phase 1.
  Also verify no old refs remain:
  SELECT count(*) FROM public.<table> WHERE <column>::text LIKE '%<old_ref>%'
  Expected output: all counts match, zero old ref hits
```

### Phase 6: Storage - Steps 42-49

Use Supabase MCP `execute_sql` + `deploy_edge_function`.
Docs: [Supabase Storage](https://supabase.com/docs/guides/storage) | [Creating buckets](https://supabase.com/docs/guides/storage/buckets/creating-buckets)

```
Step 42: Create storage buckets with correct visibility (Trap 4)
  Supabase MCP: execute_sql
  For each bucket from Step 14:
  INSERT INTO storage.buckets (id, name, public) VALUES ('<id>', '<name>', <public_bool>);
  Expected output: bucket count matches source

Step 43: Create storage RLS policies
  Supabase MCP: apply_migration
  Recreate matching storage.objects RLS policies from source.
  Expected output: storage policies applied

Step 44: Deploy migrate-storage edge function
  Supabase MCP: deploy_edge_function
  name: migrate-storage
  verify_jwt: false
  files: [{ name: "index.ts", content: <see references/migrate-storage-function.md> }]
  Expected output: function deployed successfully

Step 45: Generate migration payload for public buckets
  For each public bucket, build the file list:
  Lovable MCP: query_database
  SELECT jsonb_build_object(
    'files', jsonb_agg(
      jsonb_build_object(
        'source_url', 'https://<source_ref>.supabase.co/storage/v1/object/public/' || bucket_id || '/' || name,
        'bucket', bucket_id,
        'path', name,
        'content_type', metadata->>'mimetype'
      )
    )
  ) AS payload
  FROM storage.objects
  WHERE bucket_id = '<public_bucket_id>'
  Save: public_payload

Step 46: Generate migration payload for private buckets (Trap 4)
  For each private bucket, generate signed URLs from source:
  Lovable MCP: query_database
  SELECT storage.create_signed_url('<bucket_id>', name, 3600) AS signed_url,
         bucket_id, name, metadata->>'mimetype' AS mimetype
  FROM storage.objects
  WHERE bucket_id = '<private_bucket_id>'

  Build payload using signed_url as source_url instead of public URL.
  The signed URL has a 1-hour TTL - complete the migration within that window.
  Save: private_payload

Step 47: Invoke migrate-storage via pg_net (Trap 3 - pg_net enabled in Step 24)
  Supabase MCP: execute_sql (on destination)
  SELECT net.http_post(
    url := 'https://<dest_ref>.supabase.co/functions/v1/migrate-storage',
    headers := '{"Content-Type": "application/json"}'::jsonb,
    body := '<payload>'::jsonb
  ) AS request_id;
  Save: request_id

  For large buckets (>50 files): split into chunks of 25 files each.
  Loop the pg_net calls with different request IDs.

Step 48: Verify storage migration
  Supabase MCP: execute_sql
  Wait 5-30 seconds depending on file count, then check:
  SELECT id, status_code, content::jsonb, error_msg, created
  FROM net._http_response
  WHERE id = <request_id>;
  Expected output: status_code = 200, all files ok: true

  Also verify object count:
  SELECT count(*) FROM storage.objects;
  Compare against source storage_objects count from Step 14.

Step 49: Delete temporary migrate-storage edge function
  The user must delete it via the Supabase dashboard:
  Settings -> Edge Functions -> migrate-storage -> Delete
  (Supabase MCP does not have a delete_edge_function tool)
  Tell the user to do this after confirming storage migration succeeded.
```

### Phase 7: Edge Functions - Steps 50-54

Use Supabase MCP `deploy_edge_function` or Supabase CLI.
Docs: [Edge Functions](https://supabase.com/docs/guides/functions) | [Deploy](https://supabase.com/docs/guides/functions/deploy)

```
Step 50: List all edge functions and detect shared code (Trap 16)
  Lovable MCP: read_file for supabase/functions/ directory listing at latest_sha
  Or scan the GitHub repo: supabase/functions/{name}/index.ts
  Save: edge_function_names

  Check for shared code directory:
  Lovable MCP: read_file at path supabase/functions/_shared/
  If it exists, read ALL files in _shared/ recursively.
  Common patterns: _shared/cors.ts, _shared/supabase-client.ts, _shared/utils.ts
  Save: shared_function_files (may be empty)

  If shared code exists and there are more than 10 edge functions,
  recommend Supabase CLI deployment (Step 52 Option B) instead of
  MCP one-by-one deployment. CLI handles shared imports automatically.

Step 51: Read each function source and detect secrets (Trap 6)
  For each edge function:
  Lovable MCP: read_file at path supabase/functions/<name>/index.ts

  Also check for multi-file functions:
  Some functions have additional files beyond index.ts (e.g., types.ts, helpers.ts).
  Lovable MCP: read_file for supabase/functions/<name>/ directory listing
  Read ALL files in each function directory, not just index.ts.

  Grep for secrets the function uses:
  Look for Deno.env.get("<SECRET_NAME>") calls across ALL function files.
  Track which secrets are needed, excluding auto-provided ones:
    - SUPABASE_URL (auto)
    - SUPABASE_ANON_KEY (auto)
    - SUPABASE_SERVICE_ROLE_KEY (auto)
    - SUPABASE_DB_URL (auto)

  Grep for shared imports:
  Look for imports from "../_shared/" or "../../_shared/" patterns.
  If found and shared_function_files is empty, STOP - shared code was missed in Step 50.

  Save: function_sources (ALL files per function), secrets_per_function, shared_imports

Step 52: Deploy each function with correct verify_jwt (Trap 5, Trap 16)
  If shared_function_files is NOT empty or function count > 15:
    STRONGLY RECOMMEND Option B (Supabase CLI) - it handles shared code
    and deploys all functions in one command. MCP deploy_edge_function
    does not support shared imports between functions.

  Option A (MCP - only if NO shared code):
  Supabase MCP: deploy_edge_function
  For each function:
    name: same as source
    files: include ALL files from function directory (not just index.ts)
    verify_jwt: from verify_jwt_map (Step 3), default true if not specified

  Option B (CLI - recommended for shared code or many functions):
  The source repo should already be cloned from Phase 8 Step 56.
  cd /tmp/source && supabase functions deploy --all --project-ref <dest_ref>
  This deploys all functions including _shared/ imports in one command.

  Webhooks (Stripe, OAuth callbacks) typically need verify_jwt: false.
  User-facing functions typically need verify_jwt: true.
  Expected output: each function deployed successfully

Step 53: Generate secrets inventory for the user (Trap 6)
  After all functions are deployed, compile the list of secrets:
  Present per-function breakdown:
    process-payment: requires STRIPE_SECRET_KEY
    send-notification: requires RESEND_API_KEY
    generate-report: requires OPENAI_API_KEY

  Tell the user:
  "Set these in: Supabase Dashboard -> Settings -> Edge Functions -> Secrets.
   The functions will fail until these are set."

Step 54: Verify edge function deployment
  Supabase MCP: list_edge_functions
  Compare count against edge_function_names from Step 50.
  Expected output: function count matches source
```

### Phase 8: Frontend Code - Steps 55-61

**REQUIRES HUMAN ASSISTANCE** at Steps 58 and 59.
Docs: [Lovable GitHub integration](https://docs.lovable.dev/integrations/github)

```
Step 55: Find the original project's GitHub repo
  gh repo list {username} --json name,url
  Look for the repo that matches the original Lovable project.
  Save: original_repo_url

Step 56: Clone the original repo locally
  git clone https://github.com/{user}/{original-repo}.git /tmp/source
  Expected output: repo cloned successfully

Step 57: Create new Lovable project (Trap 1 - pass correct tech_stack)
  Lovable MCP: create_project
    description: "Migration target for <source_name>"
    workspace_id: same workspace
    tech_stack: from Step 2 detection (usually "classic" for pre-May-2026 projects)
    visibility: private
    NO initial_message - we want it as empty as possible

  WARNING: If tech_stack does not match the source, Lovable cannot build the pushed code.
  A "classic" source pushed to a "modern" scaffold will fail.
  A "modern" source pushed to a "classic" scaffold will fail.
  Save: new_lovable_project_id

>>> HUMAN STEP 58: Connect Supabase in Lovable dashboard
    Open the project > Supabase > Connect > Select the new Supabase project
    (Lovable MCP add_connector only returns a URL, user must click through)

>>> HUMAN STEP 59: Connect GitHub in Lovable dashboard
    Open the project > GitHub > Connect > Creates a new repo
    (No GitHub connector in MCP catalog)
    Save: new_github_repo

Step 60: Copy code from original to new repo
  git clone https://github.com/{user}/{new-repo}.git /tmp/new-repo
  rsync -av --exclude='.git' --exclude='.env' --exclude='bun.lockb' --exclude='package-lock.json' /tmp/source/ /tmp/new-repo/
  git add -A && git commit -m "Migrate from Lovable Cloud to Supabase" && git push

  Note: This works with private repos as long as the user has gh auth login configured.
  If push fails due to auth issues, the user can temporarily make the repo public:
    gh repo edit {user}/{new-repo} --visibility public --accept-visibility-change-consequences
  And set it back to private after the push:
    gh repo edit {user}/{new-repo} --visibility private --accept-visibility-change-consequences
  Expected output: code pushed, Lovable picks it up automatically

Step 61: Tell Lovable to rebuild
  Lovable MCP: send_message
  project_id: new_lovable_project_id
  message: "The codebase has been updated via GitHub. Please rebuild the app against the connected Supabase database. Do not modify any files."
  Expected output: Lovable rebuilds with the new Supabase connection
```

### Phase 9: Verify - Steps 62-68

```
Step 62: Check env variables
  Lovable auto-configures SUPABASE_URL and SUPABASE_ANON_KEY when Supabase is connected.
  Verify these point to the NEW project, not the old Lovable Cloud instance.

Step 63: Run comprehensive audit query on destination (Trap 12)
  Supabase MCP: execute_sql
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
    UNION ALL SELECT 'pg_net_enabled', CASE WHEN EXISTS(SELECT 1 FROM pg_extension WHERE extname='pg_net') THEN 1 ELSE 0 END
  )
  SELECT * FROM counts ORDER BY category
  Save: destination_audit_counts

Step 64: Scan for old Supabase ref leaks (Trap 7)
  For each text/jsonb column identified in Step 15:
  Supabase MCP: execute_sql
  SELECT count(*) FROM public.<table> WHERE <column>::text LIKE '%<old_ref>%'
  Expected output: 0 for every column

Step 65: Build comparison table (Trap 12 - compare all 12 categories)
  Present to the user:
  Category           Source  Destination  Status
  -----------------  ------  -----------  --------
  tables             N       N            OK
  enums              N       N            OK
  rls_policies       N       N            OK
  functions_custom   N       N            OK or MISSING
  triggers           N       N            OK or MISSING
  sequences          N       N            OK or MISSING
  indexes_custom     N       N            OK or MISSING
  auth_users         N       N            OK
  auth_identities    N       N            OK or MISSING
  storage_buckets    N       N            OK
  storage_objects    N       N            OK or PARTIAL
  pg_net_enabled     1       1            OK
  old_ref_leaks      0       0            OK

  If any row shows MISSING or PARTIAL, point to the specific phase:
    functions_custom MISSING -> re-run Phase 3 Step 28
    triggers MISSING -> re-run Phase 3 Step 29
    sequences MISSING -> re-run Phase 3 Step 27
    indexes_custom MISSING -> re-run Phase 3 Step 30
    auth_identities MISSING -> re-run Phase 4 Step 34
    storage_objects PARTIAL -> re-run Phase 6 Steps 45-48
    old_ref_leaks > 0 -> re-run Phase 5 Step 40

Step 66: Visual check
  Lovable MCP: get_project screenshot of new project
  Compare against original - should match in layout and content.
  Check that images load (storage URLs rewritten correctly).

Step 67: Test login with original credentials
  Use one of the migrated user emails + original password.
  If login fails, check:
    - encrypted_password was copied correctly (not regenerated)
    - auth.identities were migrated (Step 31)
    - email_confirmed_at is set (not null)

Step 68: Final sign-off
  Tell the user:
  1. The new Supabase project is ready: <dest_ref>
  2. Remind them to set edge function secrets (from Step 53)
  3. Tell them to delete the migrate-storage edge function (from Step 49) if not already done
  4. The ORIGINAL Lovable project keeps its Cloud instance (this path never touches it).
     If the user wants to stop paying for it, they can Pause it or remove it
     (Cloud tab > Overview > Advanced settings) once the new project is verified.
     The NEW Lovable project runs on the user's own Supabase from day one.
  5. Optional: preserve JWT secret (from Step 36) if they want existing sessions to survive
  6. If cron jobs were detected (Step 17): remind the user to recreate them in the destination
     Supabase Dashboard -> Database -> Cron Jobs, or via SQL:
     SELECT cron.schedule('job_name', 'schedule', $$command$$);
     The commands likely reference the OLD project URL - update them to the new one.
  7. If vault secrets were detected (Step 18): remind the user to re-enter values in the destination
     Supabase Dashboard -> Settings -> Vault
     Secret names were logged but values were never read - the user must provide them.
  8. If external workers were mentioned: remind the user to update any external services
     (Fly.io, Railway, Cloudflare Workers, etc.) that point to the old Supabase project URL.
```

## Human Assistance Points

| Step | What user must do | Why it cannot be automated |
|---|---|---|
| 58 | Connect Supabase in Lovable dashboard | Lovable MCP add_connector only returns a URL |
| 59 | Connect GitHub in Lovable dashboard | No GitHub connector in MCP catalog |
| 49 | Delete migrate-storage edge function | No delete_edge_function in MCP |
| 53 | Set edge function secrets in dashboard | No MCP tool for setting secrets |
| 68 | Recreate cron jobs in destination | Commands reference project-specific URLs |
| 68 | Re-enter vault secret values in destination | Values are never read during scan |
| 68 | Re-point external workers to new project URL | External services are outside Supabase |
| Cost | Confirm Supabase project cost | MCP requires explicit cost confirmation |

## Integrated Traps Reference

All 16 traps are integrated as explicit steps in their respective phases. This table maps each trap to its prevention step:

| Trap | Severity | Description | Prevention step |
|---|---|---|---|
| 1 | Critical | TanStack default changed May 6, 2026 | Step 2 (detect), Step 57 (pass tech_stack) |
| 2 | Degradation | us-east-1 capacity outages | Step 22 (default us-west-1) |
| 3 | Degradation | pg_net not enabled by default | Step 24 (enable first in Phase 3) |
| 4 | Silent bug | Private buckets return 400 with public URLs | Step 14 (detect visibility), Step 46 (signed URLs) |
| 5 | Silent bug | verify_jwt per function ignored | Step 3 (read config.toml), Step 52 (pass per-function) |
| 6 | Silent bug | Edge function secrets not listed | Step 51 (grep Deno.env.get), Step 53 (inventory) |
| 7 | Silent bug | URLs in JSONB and arrays not rewritten | Step 15 (scan all text/jsonb), Step 40 (rewrite all) |
| 8 | Critical | Custom functions/triggers not migrated | Step 9-10 (scan), Steps 28-29 (create) |
| 9 | Critical | Sequences last_value reset to 1 | Step 11 (scan with last_value), Step 27 (setval) |
| 10 | Degradation | Custom indexes not recreated | Step 12 (scan), Step 30 (recreate) |
| 11 | Critical | auth.identities not migrated | Step 13 (scan), Step 34 (insert) |
| 12 | Silent bug | Phase 9 verifies almost nothing | Steps 63-65 (compare 12 categories) |
| 13 | Silent bug | Database extensions not recreated | Step 16 (scan), Step 24 (enable all) |
| 14 | Silent bug | Cron jobs lost silently | Step 17 (detect), Step 68 (remind to recreate) |
| 15 | Silent bug | Vault secrets not migrated | Step 18 (detect names), Step 68 (remind to re-enter) |
| 16 | Silent bug | Shared edge function code (_shared/) not deployed | Step 50 (detect), Step 52 (CLI deploy) |

## Common Mistakes to Avoid

| Mistake | What happens | Correct approach |
|---|---|---|
| Using remix to copy project | Inherits Lovable Cloud, cannot connect own Supabase | Create new empty project + push code via GitHub |
| Generating temp passwords | Users cannot log in with original credentials | Copy encrypted_password bcrypt hashes from auth.users |
| Inserting profiles before auth.users | FK violation, must drop constraint | Insert auth.users FIRST, then profiles |
| Trying to pull code from GitHub into Lovable | GitHub integration is one-way OUT from Lovable | Push TO the connected GitHub repo from outside |
| Using send_message to rebuild code | Agent may modify files | Only use send_message for rebuild, not code creation |
| Skipping auth.identities | Login works but session recovery breaks | Always migrate auth.identities alongside auth.users |
| Using public URL for private bucket | Download returns 400, files silently skipped | Use signed URLs for private buckets |
| Wrong tech_stack on create_project | Lovable build fails, framework mismatch | Detect from package.json, default to "classic" |
| Skipping handle_new_user function | New signups succeed but no profile row created | Scan and migrate all custom functions and triggers |
| Not setting sequences last_value | Duplicate keys or FK violations on new records | Use setval with the source last_value |
| Not scanning for cron jobs | Scheduled tasks stop running silently | Scan cron.job in Phase 1, remind in Phase 9 |
| Not scanning for vault secrets | Functions fail at runtime with missing secrets | Scan vault.secrets names in Phase 1, remind in Phase 9 |
| Only enabling pg_net extension | Other extensions (pg_cron, pgcrypto, etc.) missing | Scan all extensions in Phase 1, enable all in Phase 3 |
| Deploying only index.ts per function | Functions with shared imports or multi-file structure fail | Read all files per function, use CLI if _shared/ exists |

## Things That Can Go Wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `create_project` returns capacity error | us-east-1 outage (Trap 2) | Retry with `us-west-1` |
| `net.http_post does not exist` | pg_net not enabled (Trap 3) | Run Step 24: `CREATE EXTENSION IF NOT EXISTS pg_net WITH SCHEMA extensions` |
| Storage migration shows 0 files migrated | Private bucket, public URL returns 400 (Trap 4) | Use signed URLs from source (Step 43) |
| Edge function returns 401 on webhook | verify_jwt is true but caller has no JWT (Trap 5) | Redeploy with `verify_jwt: false` per config.toml |
| Edge function fails with "API key not configured" | Secrets not set in destination (Trap 6) | Set secrets in dashboard per Step 50 inventory |
| Broken images after migration | URLs in JSONB not rewritten (Trap 7) | Re-run Step 40 on all text/jsonb columns |
| New signups succeed but app crashes on profile load | handle_new_user trigger missing (Trap 8) | Re-run Steps 28-29 to create functions and triggers |
| Duplicate order numbers or PK collisions | Sequence last_value reset (Trap 9) | Re-run Step 27 with correct setval |
| Slow queries after migration | Custom indexes not recreated (Trap 10) | Re-run Step 30 |
| Password recovery or session refresh fails | auth.identities not migrated (Trap 11) | Re-run Step 34 |
| Lovable build fails after code push | Wrong tech_stack (Trap 1) | Delete project, recreate with correct tech_stack from Step 2 |
| `apply_migration` rejected with dependency error | Wrong creation order in Phase 3 | Follow exact order: extensions -> enums -> tables -> sequences -> functions -> triggers -> indexes -> RLS |
| pg_net returns null content | Edge function crashed or timed out | Check `error_msg` in `net._http_response`, split into smaller batches |
| All migration looks good but verification shows mismatches | Phase 9 was incomplete (Trap 12) | Run the full 12-category audit in Step 63 |
| Cron jobs not running after migration | Cron jobs exist in source but not recreated (Trap 14) | Check cron_jobs from Step 17, recreate in destination |
| Edge functions fail with missing extension | pg_cron or other extensions not enabled (Trap 13) | Enable all source extensions in Step 24 |
| Vault-dependent functions return errors | Vault secrets not re-entered in destination (Trap 15) | Re-enter secret values in Supabase Dashboard -> Vault |
| Edge functions deploy but fail with import errors | _shared/ directory not included in deployment (Trap 16) | Use `supabase functions deploy --all` via CLI instead of MCP |
| Edge function deployment is extremely slow | Too many functions deployed one-by-one via MCP | Use CLI: `supabase functions deploy --all --project-ref <ref>` |
