# Lovable Cloud to Supabase Migration

[![Version](https://img.shields.io/badge/version-3.1.0-blue)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration)
[![Tested](https://img.shields.io/badge/tested-2026--05--12-green)](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow)](./LICENSE)

Migrate your entire Lovable Cloud project to your own Supabase. Schema, data, auth users (with original passwords and identities), storage (public and private buckets), edge functions (with correct verify_jwt per function), and frontend code.

| | |
|---|---|
| **From** | Lovable Cloud (managed database) |
| **To** | Your own Supabase project (full dashboard, custom emails, service role key) |
| **What moves** | Everything: schema, data, auth users with passwords and identities, storage, edge functions, code |
| **How** | 68 deterministic steps across 9 phases. Your agent does the work, you just connect Supabase and GitHub in the Lovable dashboard |

## Install

```bash
npx skills add CarolMonroe22/lovable-cloud-to-supabase-migration
```

Once installed, your AI agent picks it up automatically whenever you ask about migrating from Lovable Cloud.

## v3.1.0 - What's New

v3.1 adds detection of advanced components that complex projects depend on: cron jobs, vault secrets, and database extensions. These are scanned during Phase 1 and flagged for manual recreation in the destination.

| New in v3.1 | What it does |
|---|---|
| Database extensions scan | Detects all extensions (pg_cron, pgcrypto, pg_vector, etc.) and enables them in the destination |
| Cron job detection | Scans cron.job table and reminds you to recreate scheduled tasks |
| Vault secrets detection | Scans vault.secrets names (never values) and reminds you to re-enter them |
| Shared edge function code | Detects _shared/ directory and multi-file functions, recommends CLI deploy for reliability |

### v3.0.0

Integrated 16 real-world traps as explicit steps in the migration flow.

| Trap | What went wrong | What the skill does |
|---|---|---|
| TanStack detection | Created destination with wrong framework | Auto-detects classic vs modern from package.json |
| auth.identities | Password recovery and session refresh broke | Migrates identities alongside users |
| Custom functions | handle_new_user missing, new signups broke | Exports all functions via pg_get_functiondef |
| Sequences | Duplicate IDs and order number collisions | Preserves last_value with setval |
| Private buckets | Files silently missing (download returned 400) | Detects visibility, uses signed URLs for private |
| verify_jwt | Webhooks returned 401 or functions were public | Reads config.toml per function |
| Edge function secrets | Functions deployed but failed at runtime | Scans for Deno.env.get and reports required secrets |
| URL rewriting | Broken images despite files being migrated | Scans all text and jsonb columns, not just *_url |
| Custom indexes | Slow queries on previously indexed columns | Recreates non-PK indexes |
| pg_net | Storage migration failed | Enables extension early in Phase 3 |
| us-east-1 outages | Project creation failed | Defaults to us-west-1 |
| Phase 9 verification | Only counted tables | Compares 12 categories between source and destination |
| Extensions not recreated | pg_cron, pgcrypto missing in destination | Scans and enables all source extensions |
| Cron jobs lost | Scheduled tasks stopped silently | Detects and reports for manual recreation |
| Vault secrets missing | Functions fail with missing secret errors | Detects secret names for manual re-entry |
| Shared edge function code | Functions fail with import errors | Detects _shared/ directory, recommends CLI deploy |

68 deterministic steps with exact tool calls, exact SQL, exact expected output. Agents follow steps exactly, no room for interpretation.

## Not using Claude Code?

If this skill doesn't fit your setup, there is a [claude.ai chat version](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat) of this skill. It runs directly from claude.ai chat with no CLI, no terminal, and no GitHub setup required. Recommended for less technical users or if you only need to migrate your database. It does not include the frontend code migration.

## When Does It Make Sense to Migrate?

Lovable Cloud is a great managed database that works well for many projects. You get a Supabase backend without having to set anything up yourself, and for a lot of use cases that's exactly what you need.

This migration makes sense when your project grows and you need things that require a full Supabase setup:

| You want to... | Why it needs your own Supabase |
|---|---|
| Customize auth emails (sender, design, content) | Cloud sends from `no-reply@auth.lovable.cloud` |
| Access the full Supabase dashboard | Extensions, logs, monitoring, advanced management |
| Use the service role key | Required for admin operations and some edge functions |
| Have staging + production environments | Cloud is one database per project |
| Own your infrastructure long-term | Cloud can't be disconnected once enabled |

If you're happy with Cloud and none of these apply, there's no reason to migrate.

## What You Need

| Tool | Required? | How to set up |
|---|---|---|
| Claude Code (Pro or Max plan) | Yes | [Get started](https://docs.anthropic.com/en/docs/claude-code/overview) |
| Lovable MCP | Yes | Type `/mcp` in Claude Code and select Lovable - [docs](https://docs.lovable.dev/integrations/mcp-servers) |
| Supabase MCP | Yes | Already available in Claude Code - [docs](https://supabase.com/docs/guides/getting-started/mcp) |
| GitHub CLI | Yes | `brew install gh` then `gh auth login` - [docs](https://docs.github.com/en/github-cli/github-cli/quickstart) |
| Supabase CLI | Optional | Makes edge function deploys faster - [docs](https://supabase.com/docs/guides/getting-started) |

Also works with **Cursor** and other MCP-compatible agents.

## How It Works

The skill runs 9 phases in order. Your agent handles everything except 2 clicks in the Lovable dashboard (connecting Supabase and GitHub to the new project).

| Phase | What happens |
|---|---|
| 1. Scan Source | Reads schema, data, auth users, identities, functions, triggers, sequences, indexes, storage, config.toml, tech stack, extensions, cron jobs, and vault secrets |
| 2. Create Destination | Creates a new Supabase project (defaults to us-west-1) |
| 3. Apply Schema | Recreates tables, enums, constraints, sequences with last_value, custom functions, triggers, indexes, and RLS policies |
| 4. Auth Users | Imports users with original passwords AND identities (password recovery works) |
| 5. Insert Data | Moves all data and rewrites storage URLs in all text and jsonb columns |
| 6. Storage | Migrates files from public and private buckets via temporary edge function |
| 7. Edge Functions | Deploys functions with correct verify_jwt per function and reports required secrets |
| 8. Frontend Code | Creates a new Lovable project with correct tech_stack and pushes code via GitHub |
| 9. Verify | Compares 12 categories between source and destination, scans for old URL leaks |

## What Gets Migrated

| Component | Status |
|---|---|
| Database schema (tables, enums, relationships, constraints) | Fully automated |
| Custom functions (handle_new_user, etc.) | Fully automated |
| Custom triggers | Fully automated |
| Sequences with current last_value | Fully automated |
| Custom indexes | Fully automated |
| Row-level security policies | Fully automated |
| All table data | Fully automated |
| URL rewriting (text, jsonb, arrays) | Fully automated |
| Auth users with original passwords | Fully automated, no password resets |
| Auth identities | Fully automated |
| Storage buckets (public and private) | Fully automated |
| Storage URLs in database | Fully automated, rewritten to new project |
| Edge functions with correct verify_jwt | Fully automated |
| Edge function secrets inventory | Listed for manual setup |
| Frontend code | Fully automated via GitHub |
| TanStack detection (classic vs modern) | Fully automated |
| Database extensions (pg_cron, pgcrypto, etc.) | Fully automated (detected and enabled) |
| Cron jobs | Detected, listed for manual recreation |
| Vault secrets | Names detected, values must be re-entered manually |
| External workers (Fly.io, Railway, etc.) | Not detectable, mentioned in final checklist |
| OAuth/redirect config | Manual (update provider settings in Supabase dashboard) |

## Good to Know

- **Your users keep their passwords.** The skill copies the original password hashes and identities, so everyone logs in with the same credentials and password recovery works.
- **Lovable keeps working.** After migration, you still edit your app in Lovable - it just connects to your own Supabase instead of Cloud.
- **Remix won't work for this.** Remixed projects inherit Cloud. The skill creates a fresh Lovable project and pushes your code via GitHub instead.
- **Classic to classic, modern to modern.** The skill detects whether your project uses Vite or TanStack Start and creates the destination with the correct stack. It does not convert between stacks.
- **This skill is based on the Lovable MCP**, which is still in early stages. Tool availability may change. Everything here was tested and working as of May 2026.

## Resources

| Resource | Link |
|---|---|
| MCP setup guide for claude.ai chat | [carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude](https://carolmonroe.com/blog/connect-lovable-supabase-mcp-to-claude) |
| Migration guide (blog post) | [carolmonroe.com/blog/lovable-cloud-mcp-migration](https://carolmonroe.com/blog/lovable-cloud-mcp-migration) |
| Chat version of this skill | [github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat](https://github.com/CarolMonroe22/lovable-cloud-to-supabase-migration-chat) |
| Disconnect guide | [carolmonroe.com/blog/disconnect-lovable-cloud](https://carolmonroe.com/blog/disconnect-lovable-cloud) |
| Lovable MCP docs | [docs.lovable.dev/integrations/mcp-servers](https://docs.lovable.dev/integrations/mcp-servers) |
| Supabase MCP docs | [supabase.com/docs/guides/getting-started/mcp](https://supabase.com/docs/guides/getting-started/mcp) |

## License

MIT

## Author

[Carol Monroe](https://carolmonroe.com) - Lovable Champion and Supabase SupaSquad Member

- GitHub: [@CarolMonroe22](https://github.com/CarolMonroe22)
- X: [@carolmonroe](https://x.com/carolmonroe)
- LinkedIn: [carolmonroe](https://linkedin.com/in/carolmonroe)
