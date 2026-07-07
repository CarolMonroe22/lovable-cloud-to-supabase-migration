# Troubleshooting (Migration-Specific)

## "Can't disconnect Lovable Cloud"

**Outdated since July 2026.** Lovable Cloud can now be removed: Cloud tab > Overview > Advanced settings > **Remove Lovable Cloud** (with Export and Pause next to it). Follow the SACRED ORDER in SKILL.md: export, download everything, build and verify the new Supabase, and only then Remove.

The old workaround (new blank project + own Supabase + GitHub push) is now the fresh-project fallback path: see references/fresh-project-path.md.

---

## pg_restore fails with "does not support compression with zstd"

The export is zstd-compressed and your pg_restore build lacks zstd, even if the version is 16+. Common with Homebrew's libpq.

**Solution:** `brew install postgresql@18` and call its binary explicitly: `/opt/homebrew/opt/postgresql@18/bin/pg_restore`.

---

## Hundreds of errors during pg_restore

Expected. The new Supabase project already has managed auth/storage/cron/vault foundations; the backup tries to recreate them and gets "already exists" / "permission denied". Data lands fine. Verify with the 12-count gate, not the error count.

---

## Storage upload fails "The resource already exists"

The restore brought storage.objects METADATA rows (ghosts pointing at files that don't exist). SQL DELETE on them is blocked (protect_delete trigger).

**Solution:** Delete ghost objects via the Storage API, CLI, or dashboard multi-select, then upload. Also delete the restored `database_export_*` bucket.

---

## Remix inherits Lovable Cloud

If you remix a project that has Lovable Cloud, the remix also has Lovable Cloud enabled. You cannot disconnect it.

**Solution:** Do NOT use remix for migration. Create a new empty project instead and push code via GitHub.

---

## GitHub pull doesn't work (empty project after connecting repo)

Lovable's GitHub integration syncs one-way OUT by default. Pushing code from Lovable to GitHub works, but the project won't auto-import an existing repo.

**Solution:** Push code TO the connected GitHub repo from outside (git clone + rsync + git push). Lovable will pick up the commit. Then send a message to rebuild.

Docs: [Lovable GitHub integration](https://docs.lovable.dev/integrations/github)

---

## Lovable agent can't access private GitHub repo

The Lovable sandbox can't reach private repos when you send a URL via chat.

**Solution:** Temporarily make the repo public, then set it back to private after the agent pulls the code. Or push code to the connected repo from your local machine.

---

## Auth passwords not migrating

**Old guidance (WRONG):** "Passwords can't be exported"
**Correct:** Passwords CAN be migrated via Lovable MCP.

```sql
-- Read from original (Lovable MCP query_database)
SELECT id, email, encrypted_password FROM auth.users;

-- Write to new (Supabase MCP execute_sql)  
INSERT INTO auth.users (id, email, encrypted_password, ...)
VALUES ('{id}', '{email}', '{exact_bcrypt_hash}', ...);
```

Never generate temporary passwords. Always copy the original bcrypt hash.

Docs: [Supabase Auth Admin API](https://supabase.com/docs/reference/javascript/admin-api)

---

## FK violation when inserting profiles

Error: `profiles_id_fkey` foreign key constraint violated.

**Cause:** Trying to insert profiles before auth.users exist.
**Solution:** Insert auth.users FIRST (Phase 4), then profiles (Phase 5). The FK is satisfied naturally.

---

## Storage images broken after migration

Images show broken links because URLs still point to old Supabase storage.

**Solution (3 steps):**
1. Create matching storage buckets in new project
2. Deploy migrate-storage edge function (see references/migrate-storage-function.md)
3. Rewrite URLs in database: `UPDATE table SET col = replace(col, 'old-ref', 'new-ref')`

Docs: [Supabase Storage](https://supabase.com/docs/guides/storage)

---

## Env variables pointing to wrong Supabase

After connecting Supabase to the new Lovable project, the .env is auto-configured with the correct URL and anon key. If it's wrong:

1. Check via Lovable MCP send_message: "Check what SUPABASE_URL is set to in .env"
2. Get correct values: Supabase MCP get_project_url + get_publishable_keys
3. Update via send_message if needed

---

## Edge function deployment fails

**Via MCP:** Check that you're passing the full file content, not a file path.
**Via CLI:** Run `supabase functions deploy {name} --project-ref {ref}`

Docs: [Edge Functions deploy](https://supabase.com/docs/guides/functions/deploy)

---

## Google Auth 403 after migration (if applicable)

Only relevant if the project uses Google OAuth.

1. Set exact Redirect URI in Google Console: `https://{new-project-ref}.supabase.co/auth/v1/callback`
2. Complete OAuth Consent Screen (all required fields)
3. Verify Client ID and Client Secret match the new project
4. Check Site URL in Supabase Auth settings matches your app URL

Docs: [Supabase Auth](https://supabase.com/docs/guides/auth)
