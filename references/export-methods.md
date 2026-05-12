# Data Export Methods

## Method 1: Lovable MCP query_database (RECOMMENDED)

The fastest and most complete method. Requires Claude Code with Lovable MCP connected.

```
Lovable MCP query_database gives full SQL access to Lovable Cloud database,
including auth.users (with encrypted passwords), all public tables, 
storage.objects, and system catalogs.
```

### What you can export

| Data | Query |
|---|---|
| Tables + schema | `information_schema.tables`, `information_schema.columns` |
| Constraints + FKs | `information_schema.table_constraints`, `key_column_usage` |
| Custom enums | `pg_type` + `pg_enum` |
| RLS policies | `pg_policies` |
| All table data | `SELECT * FROM each_table` |
| Auth users + passwords | `SELECT id, email, encrypted_password, raw_user_meta_data FROM auth.users` |
| Storage file list | `SELECT * FROM storage.objects` |

### Advantages
- Full access to everything including auth.users
- Can export passwords (bcrypt hashes)
- Can read schema metadata for automatic migration
- No need to deploy edge functions first

---

## Method 2: Edge Function Export (No Claude Code needed)

For users without Claude Code. Deploy an edge function via Lovable chat that exports data.

```
User --- X --- Database (no direct access)
  |
  +-- Edge Function -- OK -- Database (internal supabase client)
```

### Deploy via Lovable chat:

> Create an Edge Function called 'export-data' that requires a secret key in header 'x-export-key', queries all my tables, and returns everything as JSON. Add EXPORT_SECRET to environment variables.

### Edge Function template:

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async (req) => {
  const authHeader = req.headers.get('x-export-key')
  if (authHeader !== Deno.env.get('EXPORT_SECRET')) {
    return new Response('Unauthorized', { status: 401 })
  }

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  const tableNames = ['your', 'table', 'names', 'here']
  const exportData: Record<string, any[]> = {}

  for (const tableName of tableNames) {
    const { data } = await supabase.from(tableName).select('*')
    exportData[tableName] = data || []
  }

  return new Response(JSON.stringify(exportData, null, 2), {
    headers: { 'Content-Type': 'application/json' }
  })
})
```

### Usage:

```bash
curl -H "x-export-key: your-secret" \
  https://your-project.supabase.co/functions/v1/export-data > backup.json
```

### Limitations
- Cannot export auth.users passwords (edge functions can list users but not encrypted_password)
- Cannot export schema metadata (only data)
- Large datasets may timeout
- Requires knowing your table names

---

## Method 3: REST API Export

For tables with public SELECT RLS policies.

```bash
curl "https://PROJECT_REF.supabase.co/rest/v1/TABLE_NAME?select=*" \
  -H "apikey: ANON_KEY" \
  -H "Authorization: Bearer ANON_KEY" > table_backup.json
```

### Limitations
- Only works for tables with public read access
- Cannot export auth, storage, or protected tables
- Need to know the Supabase URL and anon key

---

## Method 4: Storage Export

Storage files must be exported separately from database data.

### Via MCP (recommended):
1. Query storage URLs from database columns
2. Deploy migrate-storage edge function to NEW project
3. Function downloads from old public URLs and uploads to new storage
4. See references/migrate-storage-function.md

### Via Edge Function (no Claude Code):
```typescript
const { data: files } = await supabase.storage.from('bucket-name').list()
for (const file of files) {
  const { data } = await supabase.storage.from('bucket-name').download(file.name)
}
```

---

## Comparison

| Method | Auth users | Passwords | Schema | Data | Storage | Requires |
|---|---|---|---|---|---|---|
| MCP query_database | Yes | Yes | Yes | Yes | Via URLs | Claude Code + Lovable MCP |
| Edge Function | Partial | No | No | Yes | Partial | Lovable chat |
| REST API | No | No | No | Partial | No | URL + anon key |

**Recommendation:** Use Method 1 (MCP) whenever possible. It's the only method that gives you everything including passwords.
