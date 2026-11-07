# CrateDrop

Muziekplatform voor platenzoekers, gebouwd op de Discogs-catalogus. Live op [cratedrop.app](https://cratedrop.app). De broncode is privé; dit is de architectuur.

## Stack

- Next.js 16, React 19, TypeScript
- Cloudflare Workers via OpenNext, D1, PostgreSQL
- Discogs API via een eigen proxy
- GitHub Actions: typecheck en build

## Requests

```
request -> worker.ts
             |- /og, /card    -> aparte worker (share-afbeeldingen)
             |- publieke GET  -> edge cache -> Next.js
             |- overig        -> Next.js

cron (elk kwartier) -> worker.ts -> /api/cron/*
```

## Edge cache

Het gratis plan geeft 10 ms CPU per request en een gerenderde pagina kost ongeveer 70 ms. Herhaalde views komen daarom uit de Cache API (ongeveer 1 ms). De cachesleutel bevat de deployversie, dus elke deploy start met een lege cache.

```ts
function edgeTtl(res: Response): number {
  const cc = res.headers.get('cache-control') ?? ''
  if (res.status !== 200 || /private|no-store|no-cache/.test(cc) || res.headers.has('set-cookie')) return 0
  const m = cc.match(/s-maxage=(\d+)/)
  return m ? Math.min(parseInt(m[1], 10), EDGE_MAX) : 0
}

async function cachedFetch(req: Request, url: URL, env: Env, ctx: Ctx): Promise<Response> {
  const keyUrl = new URL(url)
  for (const p of DROP_PARAMS) keyUrl.searchParams.delete(p)
  keyUrl.searchParams.set('__v', env.CF_VERSION?.id ?? 'dev')
  const key = new Request(keyUrl.toString())
  const cache = caches.default
  const hit = await cache.match(key)
  if (hit) {
    const out = new Response(hit.body, hit)
    out.headers.set('x-edge-cache', 'HIT')
    return out
  }
  const res: Response = await handler.fetch(req, env, ctx)
  const ttl = edgeTtl(res)
  if (!ttl) return res
  const stored = new Response(res.clone().body, res)
  stored.headers.set('cache-control', `public, max-age=${ttl}`)
  ctx.waitUntil(cache.put(key, stored))
  const out = new Response(res.body, res)
  out.headers.set('x-edge-cache', 'MISS')
  return out
}
```

## Cron

Eén trigger, met een tabel van jobs in de worker. Elke job roept zijn eigen route aan.

```ts
async scheduled(event: { scheduledTime: number }, env: Env, ctx: Ctx) {
  const t = new Date(event.scheduledTime)
  for (const [h, m, wd, path] of CRON_JOBS) {
    if (t.getUTCHours() !== h || t.getUTCMinutes() !== m || (wd !== null && t.getUTCDay() !== wd)) continue
    const req = new Request(`https://cratedrop.app${path}`, {
      headers: { authorization: `Bearer ${env.CRON_SECRET ?? ''}`, 'user-agent': 'cratedrop-cron' },
    })
    ctx.waitUntil(handler.fetch(req, env, ctx).then((r: Response) => console.log(path, r.status)))
  }
}
```
