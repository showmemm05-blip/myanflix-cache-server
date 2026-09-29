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

`.env` needs two more values than it used to (see `.env.example`):
`STREAM_SIGNING_SECRET`, which must be byte-for-byte the backend's
`STREAM_SIGNING_SECRET`, and `LEGACY_UNSIGNED_MEDIA` (`allow` / `deny`).
Without the secret nginx refuses to start — deliberately.

## How access control works here

This cache **is** the access-control point for media. Every playback link the
backend hands out looks like

```
http://<cache>:8080/s/<expires>/<signature>/movies/videos/<movie-id>/hls/master.m3u8
```

and `signature = base64url(md5("<expires> <scope> <secret>"))`, where the
scope is the folder the token unlocks. There are exactly four:

| scope | unlocks |
|---|---|
| `videos/<id>/hls` | one title's whole playback package — master, every rendition, and the generated subtitle tracks under `subs/` |
| `audio/<id>/hls` | the same for audio (reserved — nothing is stored there yet) |
| `subtitles/<movieId>` | one title's uploaded subtitle **source** files |
| `books/<b>/<e>/<c>/pages` | one chapter's generated reader pages |

Those strings are not written by hand: they are generated from
`backend/src/common/storage/media-taxonomy.ts` and pasted into both signed
locations of `nginx/templates/default.conf.template`, and the backend's
`nginx-scopes.spec.ts` fails when that template drifts from the registry. See
`../docs/media-storage-layout.md` for the full key layout, and
`cd ../backend && npx ts-node scripts/print-stream-scopes.ts` to print the
alternation.

nginx's `secure_link` module checks the signature on every request, cache HIT
or MISS, before anything is served: bad signature → **403**, good signature
but past `<expires>` → **410**. The token sits in the *path* because a
playlist's segment URIs are relative — the player inherits the prefix for
every segment without knowing the scheme exists. The query string is still
ignored.

What is checked is "a live token for this title", not "this viewer": anyone
holding a link can play that one title until the link expires (12 h by
default, minted on hour boundaries — so 11–12 h). That is the price of one
shared cache entry per segment; per-viewer links would need a backend round
trip per segment. Revoking a subscription bites at the next stream lookup.

Paths without a token: `/movies/images/**` (posters, covers, avatars) stay
plain and public — that is what makes the catalogue browsable to a signed-out
visitor, and it is why nothing private is ever filed under `images/`.

Refused (**403**) token-less in BOTH legacy modes — the `allow` switch below
does not reach them:

- `/movies/videos/<id>/original.*`, `/movies/audio/<id>/original.*` — archived
  masters. The backend refuses to sign these too, so there is no URL that
  serves them at all; they are read only server-side with an access key.
- `/movies/documents/**` — book chapter source PDFs, same story: denied and
  unsignable.
- `/movies/temp/**` — object-store staging, denied and unsignable.
- `/movies/subtitles/**` — uploaded subtitle sources. These are signable (the
  stream response links them under `subtitles/<movieId>`), so only the
  token-less request is refused.

Token-less HLS segments, audio segments and book pages are refused too unless
`LEGACY_UNSIGNED_MEDIA=allow`, which exists only for the rollout window below.

## Smoke test (signed links)

The running `nginx:alpine` image ships `secure_link`; confirm once:

```bash
docker compose exec cache nginx -V 2>&1 | tr ' ' '\n' | grep secure_link
# --with-http_secure_link_module
```

Sign a link exactly the way the backend does (needs a READY movie id and the
secret from `.env`):

```bash
SECRET=$(grep '^STREAM_SIGNING_SECRET=' .env | cut -d= -f2-)
MOVIE=<movie-uuid>
SUB=<subtitle-uuid>            # a Subtitle row of that movie
SCOPE="videos/$MOVIE/hls"
SUBSCOPE="subtitles/$MOVIE"    # the uploaded SOURCE files, a scope of their own
EXP=$(( $(date +%s) / 3600 * 3600 + 43200 ))
sign() { printf '%s %s %s' "$EXP" "$1" "$SECRET" | openssl md5 -binary | openssl base64 | tr '+/' '-_' | tr -d '='; }
SIG=$(sign "$SCOPE")
SUBSIG=$(sign "$SUBSCOPE")
BASE="http://localhost:8080"

curl -sI "$BASE/movies/videos/$MOVIE/hls/master.m3u8" | head -1          # 403 (deny) / 200 (allow)
curl -sI "$BASE/movies/videos/$MOVIE/original.mp4" | head -1             # 403 — in BOTH modes
curl -sI "$BASE/movies/subtitles/$MOVIE/$SUB.srt" | head -1              # 403 — in BOTH modes
curl -sI "$BASE/movies/documents/books/<b>/<e>/<c>/original.pdf" | head -1  # 403 — in BOTH modes
curl -sI "$BASE/s/$EXP/$SIG/movies/$SCOPE/master.m3u8" | grep -i 'HTTP/\|x-cache'   # 200 MISS
curl -sI "$BASE/s/$EXP/$SIG/movies/$SCOPE/master.m3u8" | grep -i 'HTTP/\|x-cache'   # 200 HIT
curl -sI "$BASE/s/$EXP/$SIG/movies/$SCOPE/720p/segment_000.ts" | head -1 # 200 — same token, whole title
curl -sI "$BASE/s/$EXP/$SIG/movies/$SCOPE/subs/$SUB.vtt" | head -1       # 200 — same token again: the
                                                                        #       generated tracks live INSIDE
                                                                        #       the title's scope on purpose
curl -sI "$BASE/s/$EXP/$SUBSIG/movies/$SUBSCOPE/$SUB.srt" | head -1      # 200 — the uploaded source, own token
curl -sI "$BASE/s/$EXP/${SIG}x/movies/$SCOPE/master.m3u8" | head -1      # 403 — tampered signature
OLD=$(( EXP - 86400 )); OLDSIG=$(printf '%s %s %s' "$OLD" "$SCOPE" "$SECRET" | openssl md5 -binary | openssl base64 | tr '+/' '-_' | tr -d '=')
curl -sI "$BASE/s/$OLD/$OLDSIG/movies/$SCOPE/master.m3u8" | head -1      # 410 — expired
curl -sI "$BASE/movies/images/movie/<image-uuid>.jpg" | head -1          # 200 — images stay public
curl -si -X OPTIONS "$BASE/s/$EXP/$SIG/movies/$SCOPE/master.m3u8" | grep -i 'HTTP/\|access-control-allow'  # 204 + CORS
```

A book chapter is the same recipe with `SCOPE="books/<b>/<e>/<c>/pages"` and
`/pages/page-001.webp` as the object.

Two different valid tokens for one segment (mint a second one with
`EXP2=$((EXP + 3600))`) must give `HIT` on the second request: the cache key
drops the token prefix, so every token shares one entry.

## Rollout runbook (production)

Each step is independently reversible. Do them in this order — the backend
must never sign links the cache cannot verify, and the cache must never
refuse links a deployed client still holds.

One exception to "independently reversible": when the **scope list itself**
changes (a media class added, a prefix renamed), the alternation is part of
every signature, so this cache and the backend must be recreated **together**.
Every token already in flight 403s the moment either side moves — do it while
the catalogue is quiet, or accept one round of client-side recovery.

1. **Cache VPS first.** Put `STREAM_SIGNING_SECRET=<openssl rand -hex 32>`
   and `LEGACY_UNSIGNED_MEDIA=allow` in `.env`, then
   `docker compose up -d --force-recreate cache`. Run the smoke test above
   from your own machine: unsigned HLS still 200 (allow), `original.*`,
   `subtitles/**`, `documents/**` and `temp/**` 403 even in `allow`, signed
   200, tampered 403, expired 410. Nothing is broken for anyone yet.
2. **Backend VPS.** Set the SAME `STREAM_SIGNING_SECRET` (and optionally
   `STREAM_URL_TTL_SECONDS`) in `backend/.env`, redeploy. The backend refuses
   to boot without the secret. `GET /api/videos/<id>/stream` as a subscribed
   user must now return a `/s/<exp>/<sig>/` playlist URL, and that URL must
   play through the cache.
3. **Clients.** Ship the website image (the owner's `userwebsite` container
   from a new `webtest:vX`) and the mobile build that carry
   `staleTime: Infinity` on the stream query and the 403/410 recovery. Until
   then an old website tab can still rebuild its player after a reconnect
   across an hour boundary — a hiccup, not an outage.
4. **After ~24 h** (two link lifetimes, so no client still holds a plain
   link): set `LEGACY_UNSIGNED_MEDIA=deny`, recreate the cache container,
   and re-run the smoke test — unsigned HLS is now 403. To back out, flip it
   to `allow` again; nothing else changes.
5. **Storage VPS** (independent of 1–4, can go first): follow
   `../storage-server/README.md` — bucket policy limited to the cache
   address(es), forwarded-IP headers stripped on the upload proxy, port 9000
   firewalled. That is what stops the direct `:9000` / `:8443` download the
   QA report reproduced; the token work above is what stops the same
   download *through this cache*. Recreate this cache with the current
   template **before** that bucket policy goes on: the template blanks the
   viewer's `X-Forwarded-For` / `X-Real-IP` / `Forwarded` on the way to
   storage, because MinIO checks the policy's allowed address against those
   headers first, and a viewer's (or a TLS proxy's) header would otherwise
   get every cache MISS refused.

The secret ends up in the rendered `/etc/nginx/conf.d/default.conf` inside
the container — root-readable on the cache VPS, the same exposure class as
the backend's `.env`. Rotate by changing it on both sides and recreating
both; links signed with the old secret die immediately (403), so do it at a
quiet hour or accept one round of client-side recovery.

## Moving to a real VPS

Set `STORAGE_ORIGIN` in `.env` to the Movie Storage Server VPS's real
address, set the two token variables as above, and deploy this same
image/config to the cache VPS. The second cache server named in the
deployment plan needs exactly the same `.env` (same secret) and its address
added to the storage bucket policy and firewall.

On the backend side, a cache with its own address also needs
`STREAM_PUBLIC_BASE_URL=https://<cache-host>` **and**
`STREAM_PUBLIC_BASE_URL_IS_PUBLIC=true` in `backend/.env`. Without the flag
the backend keeps only the port of that base and swaps in the API's host, so
every playback link and poster would point at the API VPS, where no cache
runs. With `NODE_ENV=production` the backend refuses to start while the flag
is empty and the base is a public address (see `backend/DOCKER.md`, "Moving
to the real VPS").
