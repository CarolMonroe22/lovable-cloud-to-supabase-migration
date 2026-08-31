# Lovable Cloud to Supabase Migration

[![Version](https://img.shields.io/badge/version-4.1.1-FF5CD7)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration)
[![Tested](https://img.shields.io/badge/tested-2026--08--31-green)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow)](./LICENSE)

![Moving day: from Lovable Cloud to your own Supabase](assets/hero.png)

Migrate your entire Lovable Cloud project to your own Supabase. Schema, data,
auth users and identities, storage, edge functions with correct verify_jwt, cron
jobs, and a verified switch of the SAME Lovable project to your own Supabase.
Password resets are the official default; an advanced MCP route can preserve
existing hashes when available and explicitly approved.

| | |
|---|---|
| **From** | Lovable Cloud (managed backend) |
| **To** | Your own Supabase project (full dashboard, custom emails, service role key) |
| **What moves** | Schema, data, auth users and identities, storage, edge functions, cron jobs; passwords require reset by default or the optional MCP route |
| **How** | Official Export + Remove + Connect on your existing project (primary path), with the full MCP rebuild kept as fallback |

## Install

```bash
npx skills add CarolMonroe22/lovable-cloud-to-supabase-migration
```

Once installed, your AI agent picks it up automatically whenever you ask about migrating from, exporting, or removing Lovable Cloud.

## v4.1.1 - Attribution correction

Lovable never documented or endorsed password-hash migration. Carol found the
method in an external community guide, tested it herself, and incorporated the
working approach into this skill. The official Lovable position is the reset
path; hash preservation remains a community-derived, independently verified MCP
workaround.

## v4.1.0 - Password export clarification

Lovable's current documentation says the official project export does not contain
user passwords in a usable form, so the supported default is a password reset.
That is different from the advanced MCP path: `query_database` can still access
`auth.users.encrypted_password` on supported Cloud projects. Count-only access
was rechecked on 2026-08-31 without exposing any hash value.

Lovable never presented that MCP/SQL technique as an official capability. Carol
learned it from a community guide and verified it independently.

The skill now makes that distinction everywhere. It preserves the MCP method,
but gates it behind a capability check, explicit consent, and strict handling.
Lovable MCP is a research preview, so the reset route remains the reliable
fallback.

## v4.0.0 - Official Export + Remove + Connect

**Lovable Cloud is no longer permanent.** In July 2026 Lovable shipped official **Export**, **Pause**, and **Remove** buttons (Cloud tab > Overview > Advanced settings). v4 is a rewrite around that reality, verified end to end on a real migration: 12 tables, 237 rows, 23 users with passwords, 40 storage files, 5 edge functions, 2 cron jobs, all accounted for on the other side.

| New in v4 | What it does |
|---|---|
| Decision point | Same-project path (Export + Remove + Connect, NEW primary) vs fresh-project path (legacy fallback) vs just Pause |
| Native export as data source | The official `pg_dump` export carries the database; current docs do not guarantee usable passwords |
| The sacred order | Export -> DOWNLOAD -> build -> 12-count verification gate -> only then Remove |
| 10 new verified traps (17-26) | zstd restore tooling, identities FK ordering, cron permission + zombie URLs, storage ghost rows + protect_delete, signed URL death, client.ts overwrite breaking SSR, temp functions replicating via GitHub sync, LOVABLE_API_KEY survival, export-time snapshot |
| Lovable agent capabilities mapped | It CAN deploy edge functions to your connected Supabase (verified). It CANNOT set secrets - only you can |

The v3.1 flow (68 deterministic steps into a fresh Lovable project) is preserved intact as the fallback path in `references/fresh-project-path.md`, with its 16 original traps. Use it for databases over the 5 GB export cap or when the original Cloud project must stay untouched.

## Which path is yours?

| Your situation | Path |
|---|---|
| App burns credits while idle | No migration - **Pause Cloud**, wake it anytime |
| Want off-site backups | No migration - **Export** works standalone, once per 24h |
| Ready for your own Supabase, DB ≤ 5 GB | **Same-project path** (primary): export, restore, verify, remove, connect |
| DB > 5 GB or original project must stay untouched | **Fresh-project path** (legacy fallback) |

## How the primary path works

33 steps across 8 phases. Nothing is removed until its replacement is alive and verified - Cloud stays as the safety net until the final gate is green.

| Phase | What happens |
|---|---|
| 1. Baseline + GitHub | 12-count inventory of everything the project owns; repo connected (function code travels via the repo) |
| 2. Export + Download | Official export triggered, downloaded (it lives INSIDE Cloud storage), storage files downloaded |
| 3. Create destination | New Supabase project, session pooler connection string |
| 4. Restore | `pg_restore` of the official backup (zstd-capable tooling), identities fix pass |
| 5. Post-restore fixups | Cron recreation + zombie URL hunt, storage ghost-row cleanup, file upload, URL rewriting |
| 6. Functions + secrets | Deploy via CLI / MCP / Lovable agent; secrets re-entered by you; Lovable AI re-wired |
| 7. The gate | Full 12-category comparison vs baseline + tested reset or original-password login after MCP preservation. All green or stop |
| 8. Remove + Connect | Remove Cloud, connect your own Supabase to the SAME project, fix the integration overwrite |

## What Gets Migrated

| Component | Status |
|---|---|
| Database schema (tables, enums, constraints, indexes, sequences with values) | In the official export |
| RLS policies, custom functions, triggers | In the official export |
| All table data | In the official export |
| Auth users + identities | Verify after restore; official password path requires resets |
| Cron jobs | Rows in the export; recreated via cron.schedule with fresh URLs |
| Storage buckets | In the official export |
| Storage files | Downloaded and re-uploaded (never in any DB export) |
| Edge functions | Redeployed from your repo (CLI, MCP, or the Lovable agent itself) |
| Edge function secrets | Names inventoried; values re-entered by you (never exportable, by design) |
| Lovable AI features | Keep working - LOVABLE_API_KEY survives, calls re-wired |
| Vault secrets | Names inventoried; values re-entered by you (encrypted with the old project's key) |
| Frontend code | Stays put - same Lovable project, same repo |
| OAuth providers | Manual recreation (client ID/secret + redirect URLs) |
| External workers (Fly.io, Railway, etc.) | Not detectable; re-point them in the final checklist |

## Good to Know

- **Passwords have two paths.** Official export means reset. The optional, community-derived MCP path can preserve hashes when the capability check passes and the sensitive flow is approved. Lovable has never documented or endorsed that workaround.
- **Your Lovable project stays your Lovable project.** Same editor, same repo, same URL - only the backend home changes.
- **The export file deserves password-level care.** It contains personal and authentication data and may contain credential material. Keep it local, delete it after verifying.
- **Buttons are free.** Export, Pause, Remove and Connect cost no credits; only agent prompts do.
- **Tested August 2026.** Lovable ships fast; re-run capability checks before relying on MCP behavior.

## Not using Claude Code?

The full flow also runs from claude.ai chat with the Lovable + Supabase connectors (no CLI, no terminal). See the [Migrate with AI guide](https://carolmonroe.com/blog/migrate-lovable-cloud-with-ai) for both routes side by side. There is also a [chat-native version of this skill](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat) you can load into claude.ai directly.

Also works in **Cursor** and other MCP-compatible agents.

## What you need

| Tool | Required? | Note |
|---|---|---|
| Lovable MCP | Strongly recommended | It makes life easy: the baseline inventory and every source-side check become one tool call - [setup](https://docs.lovable.dev/integrations/mcp-servers) |
| Supabase MCP | Strongly recommended | Destination-side automation - [setup](https://supabase.com/docs/guides/getting-started/mcp) |
| pg_restore 16+ with zstd | For the restore step | macOS: `brew install postgresql@18` |
| Supabase CLI | Optional | Faster function deploys and storage uploads |
| GitHub CLI | Fallback path only | `brew install gh` |

## Resources

| Resource | Link |
|---|---|
| The manual walkthrough this skill automates | [carolmonroe.com/blog/export-remove-lovable-cloud](https://carolmonroe.com/blog/export-remove-lovable-cloud) |
| Migrate with AI (claude.ai + Claude Code routes) | [carolmonroe.com/blog/migrate-lovable-cloud-with-ai](https://carolmonroe.com/blog/migrate-lovable-cloud-with-ai) |
| Legacy MCP migration guide | [carolmonroe.com/blog/lovable-cloud-mcp-migration](https://carolmonroe.com/blog/lovable-cloud-mcp-migration) |
| MCP setup for claude.ai chat | [carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude](https://carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude) |
| Lovable Cloud docs | [docs.lovable.dev/integrations/cloud](https://docs.lovable.dev/integrations/cloud) |
| Supabase restore docs | [supabase.com/docs/guides/platform/migrating-within-supabase](https://supabase.com/docs/guides/platform/migrating-within-supabase) |

## License

MIT

## Author

[Carol Monroe](https://carolmonroe.com) - Lovable Champion and Supabase SupaSquad Member

- GitHub: [@CarolMonroe22](https://github.com/CarolMonroe22)
- X: [@carolmonroe](https://x.com/carolmonroe)
- LinkedIn: [carolmonroe](https://linkedin.com/in/carolmonroe)
