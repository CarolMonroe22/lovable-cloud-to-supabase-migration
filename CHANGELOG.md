# Changelog

All notable changes to this skill, in human language.

## v4.0.0 - 2026-07-06 - "Cloud is no longer permanent"

The big one. In July 2026 Lovable shipped official **Export**, **Pause**, and
**Remove** buttons for Cloud, and that deserved a rewrite, not a patch. Everything
below was verified on a real end-to-end migration (12 tables, 237 rows, 23 users
with their original passwords, 40 storage files, 5 edge functions, 2 cron jobs).

### Changed
- **New primary path: same-project migration.** Export your database, remove
  Cloud, connect your own Supabase to the SAME Lovable project. No new project,
  no code push, your users never notice. 33 steps across 8 phases.
- **The official export file is now the data source.** It's a proper `pg_dump`
  backup carrying your schema, data, RLS, triggers, sequences, cron rows, and
  your auth users WITH password hashes. The skill restores and verifies it
  instead of rebuilding everything by hand.
- **The old 68-step MCP flow is now the fallback**, preserved intact in
  `references/fresh-project-path.md` for databases over the 5 GB export cap or
  when the original Cloud project must stay untouched.
- The decision point now includes the honest options: sometimes the answer is
  just Pause, or Export-as-backup, and no migration at all.

### Added - 10 new traps, all hit personally so you don't have to
| # | The trap |
|---|---|
| 17 | The export saves INTO the Cloud storage you're about to delete. Download before Remove, always |
| 18 | The export is zstd-compressed; Homebrew's minimal Postgres tools can't read it even at "version 16+". postgresql@18 can |
| 19 | The restore drops auth.identities on a foreign-key ordering quirk. One extra pass fixes it |
| 20 | Cron jobs restore as "permission denied", and recreated ones still point at your OLD project URL. Zombie hunt included |
| 21 | The restore brings storage metadata rows without files. They block uploads, and SQL can't delete them - the Storage API can |
| 22 | Signed URLs (?token=) die with the old project. Rewriting them makes broken links that LOOK right |
| 23 | Connecting your own Supabase can overwrite client.ts and break SSR builds. One prompt fixes the wiring |
| 24 | Temporary helper functions come BACK via two-way GitHub sync. Delete them in the repo too |
| 25 | Good news trap: LOVABLE_API_KEY survives Cloud removal. Your AI features live, they just need re-wiring |
| 26 | The export is a snapshot. Anything written after export time isn't in it |

### Also
- Verified live: the Lovable agent CAN deploy edge functions to your connected
  Supabase. It CANNOT set secrets (nobody's agent should).
- 12-count verification gate before the destructive step, with a real-login test.
- Companion guides: [the full walkthrough](https://carolmonroe.com/blog/export-remove-lovable-cloud)
  and [Migrate with AI](https://carolmonroe.com/blog/migrate-lovable-cloud-with-ai).

## v3.1.0 - 2026-05-10

Added detection of advanced components that complex projects depend on:

| Added | What it does |
|---|---|
| Database extensions scan | Detects all extensions (pg_cron, pgcrypto, pg_vector, etc.) and enables them in the destination |
| Cron job detection | Scans cron.job and reminds you to recreate scheduled tasks |
| Vault secrets detection | Scans vault.secrets names (never values) and reminds you to re-enter them |
| Shared edge function code | Detects _shared/ and multi-file functions, recommends CLI deploy |

## v3.0.0 - 2026-05-10

Integrated 16 real-world traps as explicit steps in the 68-step flow: TanStack
stack detection, auth.identities, custom functions (handle_new_user), sequences
with last_value, private bucket signed URLs, per-function verify_jwt, edge
function secrets inventory, URL rewriting in all text/jsonb columns, custom
indexes, pg_net enablement, us-east-1 capacity fallback, and a 12-category
verification phase.

## v2.x - 2026 (spring)

The pre-MCP era: 9 phases, 40 steps, edge-function exports. Passwords "couldn't
be migrated" back then. We got better.
