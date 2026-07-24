# Cache server (local test rig)

Local stand-in for the **Flokinet (VPS) Cache Server** role from the
architecture diagram: an nginx container with `proxy_cache`, sitting in front
of the storage origin. It's deliberately standalone — it does **not** bundle
the storage server. That's a separate service (`../storage-server`), matching
the real deployment where each is its own VPS.

Run `../storage-server` first — this has nothing to cache without it.

## Run it

```bash
cd cacheserver
cp .env.example .env
docker compose up -d
```

The cache server listens on `http://localhost:8080` and forwards to whatever
`STORAGE_ORIGIN` in `.env` points at (defaults to `../storage-server`'s MinIO,
reached via `host.docker.internal:9000`).

## Test that caching actually works

1. With `../storage-server` running, open its MinIO console
   (`http://localhost:9001`), create a bucket (e.g. `movies`), and upload any
   test file.
2. Under the bucket's **Access Policy**, set it to allow anonymous/public
   read — just for this local smoke test, so you can fetch the object
   without wiring up presigned URLs yet.
3. Request the file **through the cache server, not MinIO directly**:

   ```bash
   curl -i http://localhost:8080/movies/<your-file-name>
   ```

   Check the `X-Cache-Status` response header:
   - First request → `MISS` (fetched from the storage origin, then cached)
   - Every request after → `HIT` (served straight from the cache server, the origin never touched)

That `MISS` → `HIT` transition is the whole point: repeat requests for the
same file stop reaching the storage origin (and its VPS's uplink) at all.

## Moving to a real VPS

Set `STORAGE_ORIGIN` in `.env` to the Movie Storage Server VPS's real
address, and deploy this same image/config to the cache VPS. Nothing else
changes.

## Known limitation — read before wiring in real permission checks

The cache key intentionally ignores the URL's query string, so every
viewer's differently-signed presigned URL for the *same* file still shares
one cache entry (otherwise nothing would ever cache — each presigned URL has
a unique signature). The tradeoff: a cache **HIT** is served without
re-validating that specific request's signature, since the storage origin —
the thing that would check it — is never contacted on a hit.

That's fine as long as permission is enforced **before** a viewer ever
receives a URL (Flokinet's app server checks ownership, *then* hands out a
link) — it stops being fine if this cache is ever treated as the access
control itself. If per-request revocation matters later, add a token-check
(`auth_request`) in front of the `location /` block in
`nginx/templates/default.conf.template`.

## Not done yet

- `backend/`'s `StorageService` still only writes to local disk — it doesn't
  talk to the storage server yet. That integration (upload via S3 API,
  presigned URL generation) is the next piece.
