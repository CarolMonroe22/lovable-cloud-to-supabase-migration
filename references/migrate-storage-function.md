# Storage Migration Edge Function

Deploy this temporary edge function to the NEW Supabase project. It downloads files from the old public storage URLs and uploads them to the new project's storage.

## Function Code

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

Deno.serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response(null, { headers: corsHeaders })
  }

  try {
    const { urls } = await req.json()
    if (!urls || !Array.isArray(urls)) {
      return new Response(JSON.stringify({ error: 'urls array required' }), {
        status: 400,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      })
    }

    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
    )

    const results = []

    for (const oldUrl of urls) {
      try {
        const urlObj = new URL(oldUrl)
        const pathParts = urlObj.pathname.split('/storage/v1/object/public/')
        if (pathParts.length < 2) {
          results.push({ url: oldUrl, status: 'skipped', error: 'not a storage URL' })
          continue
        }
        const fullPath = pathParts[1]
        const slashIdx = fullPath.indexOf('/')
        const bucket = fullPath.substring(0, slashIdx)
        const filePath = fullPath.substring(slashIdx + 1)

        const response = await fetch(oldUrl)
        if (!response.ok) {
          results.push({ url: oldUrl, status: 'failed', error: `download failed: ${response.status}` })
          continue
        }
        const blob = await response.blob()
        const arrayBuffer = await blob.arrayBuffer()
        const uint8 = new Uint8Array(arrayBuffer)

        const { error } = await supabase.storage
          .from(bucket)
          .upload(filePath, uint8, {
            contentType: response.headers.get('content-type') || 'application/octet-stream',
            upsert: true
          })

        if (error) {
          results.push({ url: oldUrl, status: 'failed', error: error.message })
        } else {
          results.push({ url: oldUrl, status: 'success', newPath: `${bucket}/${filePath}` })
        }
      } catch (e) {
        results.push({ url: oldUrl, status: 'failed', error: e instanceof Error ? e.message : String(e) })
      }
    }

    const succeeded = results.filter(r => r.status === 'success').length
    const failed = results.filter(r => r.status === 'failed').length

    return new Response(JSON.stringify({ succeeded, failed, total: urls.length, results }, null, 2), {
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
3. Call with all old storage URLs found in the database
4. Process in batches of ~15-20 URLs per call to avoid timeouts
5. Delete the function after migration is complete

## Notes

- Only works if old storage URLs are public (Lovable Cloud storage is public by default)
- Preserves file paths and content types
- Uses upsert: true so it's safe to retry
