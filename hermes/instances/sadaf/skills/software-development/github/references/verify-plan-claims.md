# Empirically verifying claims made by an issue plan

An audit plan asserts a performance or correctness claim and proposes a fix.
Reasoning is not evidence: run a throwaway local reproduction, quote the
output in the review comment. Everything here is disposable and must never
touch the app database or leave files in the tracked tree.

## SQL equivalence (query rewrites, correlated subqueries)

Start a scratch cluster instead of using the app's database:

```bash
PGBIN=$(ls -d /usr/lib/postgresql/*/bin | head -1)
PGDATA=$SCRATCH/pgtest
$PGBIN/initdb -D $PGDATA -U postgres -A trust
$PGBIN/pg_ctl -D $PGDATA -o "-p 55432 -k /tmp" -l /tmp/pg.log start
$PGBIN/psql -h /tmp -p 55432 -U postgres -f fixture.sql
```

- Seed the boundaries, not just typical rows: one row older than the window,
  rows inside it, one on the first/last bucket, and a NULL in every column the
  statements compare.
- Run the old and the proposed statement side by side for several window sizes
  and diff the series row by row.
- Recurring failure of "generate_series + LEFT JOIN + cumulative window"
  rewrites: a row whose month falls outside the generated series never joins,
  so a cumulative sum starts at zero and silently drops all pre-window stock
  (an undercount of the whole series, not just the first bucket). The fix is a
  baseline aggregate over rows strictly before the first bucket, added to the
  window sum.
- Check NULL semantics too: a `<` comparison excludes NULL and `date_trunc`
  on NULL produces a group that never matches — the old and new statements must
  agree on that, and the review should say they do.
- Stop and remove: `$PGBIN/pg_ctl -D $PGDATA stop -m fast`, then delete the dir.

## nginx config semantics

Validate a snippet in isolation before endorsing it:

```bash
nginx -t -p $PREFIX -c conf/nginx.conf    # PREFIX holds conf/, logs/, www/
nginx -p $PREFIX -c conf/nginx.conf       # serve on a high loopback port
curl -sI http://127.0.0.1:PORT/build/assets/x.css
```

- `add_header` is not inherited additively: any `add_header` inside a
  `location` discards every `add_header` from the enclosing `server` block for
  that location. Static-asset locations therefore silently lose security
  headers unless they repeat them or the headers arrive via `include`. Prove
  it by curling a static path and looking for the parent's headers.
- `expires` plus a manual `Cache-Control` yields two `Cache-Control` headers;
  keep one (`add_header … always`, which also covers non-2xx responses).
- gzip applies only above `gzip_min_length` and to the listed types; `woff2`
  and images are already compressed and should not be listed.
- `immutable` is only safe for content-hashed paths; unhashed `/js`, `/css`,
  `/fonts` need a short TTL or they serve stale code after a deploy.
- A config that lives only in a markdown install guide has exactly one copy:
  "mirror it in the other file" steps in the plan are void.

## Frontend claims (fonts, which CSS rule wins)

Serve the repo's `public/` locally, write a throwaway probe page into it, and
read computed values — do not reason about the cascade:

```bash
python3 -m http.server 8971     # background, cwd = repo/public
```

```python
js("getComputedStyle(document.body).fontFamily")
js("(() => [...document.fonts].map(f => [f.family, f.weight, f.status]))()")
js("document.fonts.check('16px \"Family\"')")
```

- Unlayered CSS beats `@layer` utilities regardless of specificity. Tailwind v4
  emits utilities into a layer, so a plain `body { font-family: … }` in the app
  stylesheet can win over `.font-sans`. Check the theme variable
  (`@theme { --font-sans: … }`) before claiming a utility controls the font, and
  prefer fixing the token there over adding `!important` rules.
- `format("truetype")` on a `.woff2` file still decodes in Chromium
  (`FontFace.status === 'loaded'`). Do not claim a font fails to load without
  measuring, and mark engines you did not test as unverified.
- Delete probe files afterwards — they sit in a tracked tree.

## Build metrics and manifests

- Build outputs are commonly gitignored: `npm ci && npm run build`, then
  `stat -c '%n %s' public/build/assets/*` for real byte counts.
- `manifest.json` maps **source** paths to `{file: assets/<hashed>}` with one
  entry per source. Two keys naming the same font are not "two emitted copies";
  count emitted files and check whether the stylesheet's URL survived the
  rewrite (`format(...)` hints can be preserved while the path is rewritten).

## Third-party CDN claims

- An unversioned CDN URL is a moving target: read the version banner and
  compare it with the vendored file's banner — a major-version gap turns a
  "move it local" change into a downgrade that needs a visual parity check.

```bash
curl -s -H 'Referer: https://example.com/' -r 0-200 https://cdn.example/lib.js | head -3
```

- Some CDNs reject requests without a `Referer` (`Missing Referer`). On an
  intranet app that makes the asset fragile: a referrer-stripping policy or
  extension blanks the page. Response headers (`last-modified`) also date the
  build being served.
