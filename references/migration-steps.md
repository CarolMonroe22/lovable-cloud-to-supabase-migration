# Migration Steps (Updated 2026-07-06)

> Since July 2026 the PRIMARY method is the native Export + Remove + Connect flow
> on the same Lovable project - see SKILL.md. The methods below are the fallback
> (fresh-project path, DB > 5 GB, or export unavailable).

> Previous version of this guide said passwords can't be exported. That was WRONG.
> Passwords CAN be migrated: they come in the native export, or by copying the
> bcrypt hashes from auth.users via MCP.

## Method: Full MCP Migration (fallback)

This method uses Lovable MCP + Supabase MCP + GitHub CLI to automate the entire migration. Only 2 steps require manual dashboard clicks.

### Requirements
- Claude Code (Pro or Max)
- Lovable MCP connected
- Supabase MCP plugin
- GitHub CLI installed and authenticated

### Phases (see SKILL.md for detailed steps)

1. **Scan Source** - Read schema, data, auth users, storage URLs from Lovable Cloud via MCP
2. **Create Destination** - New Supabase project via MCP
3. **Apply Schema** - Enums, tables, constraints, RLS policies
4. **Auth Users** - Insert with original bcrypt password hashes (BEFORE data)
5. **Insert Data** - Catalogs first, then user-owned, then junction tables
6. **Storage** - Create buckets, deploy migration function, transfer files, rewrite URLs
7. **Edge Functions** - Deploy all functions to new project
8. **Frontend Code** - Push via GitHub (two-way sync works)
9. **Verify** - Full audit of all components

### What gets migrated
- Database schema (tables, enums, constraints, FKs)
- RLS policies (all of them, identical)
- All data (preserving UUIDs and timestamps)
- Auth users with original passwords
- Storage buckets and files
- Edge functions
- Frontend code (exact same files)
- Environment variables (auto-configured by Lovable)

---

## Legacy Method: Edge Function Export (No Claude Code needed)

For users without Claude Code, the edge function approach still works but is more limited:

1. Deploy an export edge function via Lovable chat
2. Call it to get JSON data
3. Manually create tables in new Supabase
4. Import data via Supabase dashboard or CLI
5. Users must reset passwords (this legacy method can't access auth.users - the native export and the MCP method both carry passwords, use those instead)

See [export-methods.md](export-methods.md) for the edge function template.

---

## Legacy Method: REST API Export

1. Find Supabase URL + anon key from project config
2. Pull data via REST API with curl
3. Import into new Supabase

Limitation: Only gets data from tables with public SELECT RLS policies.

---

## Fresh-Project Workaround (legacy - Cloud is removable since July 2026)

No longer needed for the standard case: Cloud can now be REMOVED from the same
project (SKILL.md primary path). Keep this only when the original project must
stay untouched:

1. Export everything using MCP method above
2. Create new blank Lovable project
3. Connect YOUR Supabase (not Lovable Cloud)
4. Connect GitHub
5. Push code from original repo to new repo
6. Lovable picks up the code via two-way GitHub sync
7. Tell Lovable agent to rebuild

---

## Auth Migration Notes (CORRECTED)

- Passwords **CAN be exported** via Lovable MCP: `SELECT encrypted_password FROM auth.users`
- Copy the bcrypt hashes directly to the new project's auth.users table
- Users log in with their exact same credentials - no password reset needed
- Google OAuth users need correct redirect URIs configured in new project
- User metadata (display names, avatars) is in raw_user_meta_data jsonb column

---

## Self-Hosting Resources

- Lovable self-hosting docs: https://docs.lovable.dev/tips-tricks/self-hosting
- Carol's export guide: https://carolmonroe.com/blog/export-lovable-cloud-claude-code
- Carol's disconnect guide: https://carolmonroe.com/blog/disconnect-lovable-cloud
- Video tutorial: https://youtu.be/jEBVpl1GBvQ
