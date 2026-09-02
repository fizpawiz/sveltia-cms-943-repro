# Repro: unresolvable `preview_path` template hangs the editor

Minimal reproduction for [sveltia/sveltia-cms#943](https://github.com/sveltia/sveltia-cms/issues/943).

Two files plus a static file server. No repository, no OAuth, no backend: it
uses `test-repo`.

## Run it

```
python3 -m http.server 8943
```

Open <http://localhost:8943/admin/>.

1. Click **Work with Test Repository**.
2. Click **New**, put anything in Title, click **Save**.
3. Click the saved entry in the list to open it.
4. Wait a few seconds, then click the editor's back arrow.

`?cms=<version>` switches releases for one page load, for example
`http://localhost:8943/admin/?cms=0.203.2`. The default is 0.205.0.

## What happens

The console fills with requests to `/blog/<random hex>`, a new random path
each time. After about 1,100 of them:

```
Uncaught Error: https://svelte.dev/e/effect_update_depth_exceeded
```

From that point the reactive graph is dead. The back arrow changes the URL to
`#/collections/blog` but the entry editor stays on screen and the list never
renders. Only a full page reload recovers.

## The trigger

One line in `admin/config.yml`:

```yaml
preview_path: blog/{{fields.slug}}
```

The collection has no `slug` field, so the template cannot resolve.

## The mechanism, in the source

File and line references are for v0.205.0.

1. `src/lib/services/common/template/replacers.js:120` falls back to
   `generateUUID('short')` when a template tag resolves to `undefined` and no
   `default` transformation is set. Every evaluation of the template returns a
   **different** value.
2. `src/lib/components/contents/details/preview-link-button.svelte:113` runs an
   `$effect` that pings the composed preview URL. The comment inside it says
   the derived values "stay the same across such an update, so the effect
   doesn't re-run". The random fallback breaks that assumption.
3. `src/lib/services/deployments/ping.js:125` records the ping result in the
   `pageLiveness` store, keyed by URL. The URL is new every time, so the
   "skip an update that changes nothing" guard never applies and the store
   changes on every ping.
4. The store change recomputes the `$derived` link, which evaluates the
   template again, which mints a new random URL, which re-runs the effect.
   Around 1,000 cycles later Svelte's `effect_update_depth_exceeded` guard
   fires and kills the graph.

## Measurements

Identical on **0.203.2, 0.204.0 and 0.205.0**, in Chromium and in
Firefox 150, with the server answering 404 or 200, with this minimal config
and with the full production config that hit this in the wild, whether the
entry is opened from the list or by direct URL.

| Condition | Result, per version |
| --- | --- |
| Local server, ~4 ms responses | ~1,100 pings, then the error |
| 150 ms responses (live-like), Firefox | 166 pings, error 16.2 s after the entry opens |

The numbers match across all three versions to the ping. Issue #943 was
originally filed as a 0.204.0 regression; that attribution was wrong. The bad
`preview_path` and the 0.204.0 update shipped in the same deploy, and the
checks that seemed to clear 0.203.2 were interactions shorter than the ~16
seconds the loop needs, against an origin whose SPA fallback answers 200, which
keeps every runaway request out of the console.

## Verified workaround

A `default` transformation prevents the random fallback, so the URL is stable
and the editor makes exactly one liveness request:

```yaml
preview_path: blog/{{fields.slug | default('missing')}}
```

Removing `preview_path`, or naming a field that exists, also fixes it.

## Suggested fix

The random-ID fallback makes sense when a slug is generated once at save time.
`preview_path` is recomputed reactively on every render, so an unresolvable tag
there needs a stable result: a fixed placeholder, or `undefined` so no link
renders. Either breaks the cycle. A config-time warning for a `{{fields.*}}`
tag that names no field in the collection would have surfaced this in seconds.
