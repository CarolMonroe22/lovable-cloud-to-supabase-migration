# Storage Migration Edge Function

Deploy this temporary edge function to the NEW Supabase project. It downloads files from the old project's storage (public URLs or signed URLs) and uploads them to the new project's storage.

## Contract

The payload matches what Steps 45-46 of the fresh-project path generate:

```json
{
  "files": [
    {
      "source_url": "https://<old_ref>.supabase.co/storage/v1/object/public/avatars/seed/a.jpg",
      "bucket": "avatars",
      "path": "seed/a.jpg",
      "content_type": "image/jpeg"
    }
  ]
}
```

`bucket` and `path` are used EXACTLY as given - the function never derives them from
the URL. That is what makes signed URLs from private buckets work: the query token
stays on `source_url` for the download, and the destination comes from the explicit
fields. `content_type` is optional (falls back to the response header, then
`application/octet-stream`).

A legacy `{ "urls": [...] }` payload is still accepted for public URLs only.

## Function Code

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

interface FileSpec {
  source_url: string
  bucket: string
  path: string
  content_type?: string
}

// Legacy support: derive bucket/path from a PUBLIC storage URL.
// Signed URLs never take this path - they must use the files contract.
function fromPublicUrl(oldUrl: string): FileSpec | null {
  const parts = new URL(oldUrl).pathname.split('/storage/v1/object/public/')
  if (parts.length < 2) return null
  const fullPath = decodeURIComponent(parts[1])
  const slashIdx = fullPath.indexOf('/')
  if (slashIdx < 1) return null
  return {
    source_url: oldUrl,
    bucket: fullPath.substring(0, slashIdx),
    path: fullPath.substring(slashIdx + 1),
  }
}

Deno.serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders })
  }

  try {
    const body = await req.json()
    let files: FileSpec[] = []

    if (Array.isArray(body.files)) {
      files = body.files
    } else if (Array.isArray(body.urls)) {
      files = body.urls
        .map((u: string) => fromPublicUrl(u))
        .filter((f: FileSpec | null): f is FileSpec => f !== null)
    }

    if (files.length === 0) {
      return new Response(
        JSON.stringify({ error: 'files array required: [{ source_url, bucket, path, content_type? }]' }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      )
    }

    const bad = files.find(f => !f.source_url || !f.bucket || !f.path)
    if (bad) {
      return new Response(
        JSON.stringify({ error: 'every file needs source_url, bucket and path', offending: bad }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      )
    }

    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
    )

    const results = []

    for (const file of files) {
      try {
        const response = await fetch(file.source_url)
        if (!response.ok) {
          results.push({ path: `${file.bucket}/${file.path}`, status: 'failed', error: `download failed: ${response.status}` })
          continue
        }
        const uint8 = new Uint8Array(await response.arrayBuffer())

        const { error } = await supabase.storage
          .from(file.bucket)
          .upload(file.path, uint8, {
            contentType: file.content_type || response.headers.get('content-type') || 'application/octet-stream',
            upsert: true
          })

        if (error) {
          results.push({ path: `${file.bucket}/${file.path}`, status: 'failed', error: error.message })
        } else {
          results.push({ path: `${file.bucket}/${file.path}`, status: 'success' })
        }
      } catch (e) {
        results.push({ path: `${file.bucket}/${file.path}`, status: 'failed', error: e instanceof Error ? e.message : String(e) })
      }
    }

    const succeeded = results.filter(r => r.status === 'success').length
    const failed = results.filter(r => r.status === 'failed').length

    return new Response(JSON.stringify({ succeeded, failed, total: files.length, results }, null, 2), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' }
    })
  } catch (e) {
    return new Response(JSON.stringify({ error: e instanceof Error ? e.message : String(e) }), {
      status: 500,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' }
    })
  }
})
```

## How to use

1. Deploy via Supabase MCP `deploy_edge_function` with name `migrate-storage`
2. Create storage buckets first (matching the original bucket names)
3. Send the `files` payload built in Steps 45-46 (public URLs for public buckets, signed URLs for private buckets)
4. Process in batches of ~15-20 files per call to avoid timeouts
5. Delete the function after migration is complete - from the Supabase dashboard AND from the repo if it was committed (GitHub sync brings it back otherwise)

## Notes

- Private buckets work through signed URLs on `source_url` (1-hour TTL - finish each batch within the window)
- `bucket`/`path` are never derived from the URL, so tokens and URL encoding can't corrupt destinations
- Preserves content types; uses upsert: true so it's safe to retry
