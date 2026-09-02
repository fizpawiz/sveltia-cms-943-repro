# Repro: `effect_update_depth_exceeded` from an unresolvable `preview_path`

Minimal reproduction for [sveltia/sveltia-cms#943](https://github.com/sveltia/sveltia-cms/issues/943).

No repository, no OAuth and no backend are needed. It uses the `test-repo`
backend, so the whole thing is two files plus a static file server.

## Run it

```
python3 -m http.server 8943
```

Open <http://localhost:8943/admin/>.

1. Click **Work with Test Repository**.
2. Click **New**, put anything in Title, click **Save**.
3. Click the saved entry in the list to open it.

`?cms=<version>` switches releases for one page load, for example
`http://localhost:8943/admin/?cms=0.203.2`. The default is 0.205.0.

## What happens

Within a few seconds the console fills with hundreds of failed requests to
`/blog/<random hex>`, a new random path roughly every 4 ms:

```
GET http://localhost:8943/blog/e31065e92ace  404
GET http://localhost:8943/blog/953092dd86d1  404
GET http://localhost:8943/blog/633ad875b397  404
...
```

After about 1,100 of them:

```
Uncaught Error: https://svelte.dev/e/effect_update_depth_exceeded
    at Di (sveltia-cms.js:84:3713)
    at Ga (sveltia-cms.js:84:14874)
    at #g (sveltia-cms.js:84:12233)   [repeated ~180x]
```

The reactive graph is dead from that point on. Clicking the back arrow changes
the URL to `#/collections/blog`, but the entry editor stays on screen and the
entry list never renders. Only a full page reload recovers.

## The trigger

One line in `admin/config.yml`:

```yaml
preview_path: blog/{{fields.slug}}
```

The collection has no `slug` field, so the template cannot resolve. The
unresolved value appears to be regenerated on every render, which changes the
preview URL, which schedules another render.

Changing only that line is enough:

| `preview_path`                          | result                     |
|-----------------------------------------|----------------------------|
| `blog/{{fields.slug}}` (no such field)  | infinite loop, UI frozen   |
| `blog/{{fields.title}}` (a real field)  | one request, works         |
| `blog/{{slug}}`                         | one request, works         |
| omitted                                 | works                      |

## Versions tested

Reproduces identically on **0.203.2, 0.204.0 and 0.205.0**, in Chromium via
Playwright on Ubuntu.

Because 0.203.2 is affected too, this loop is probably **not** the 0.204.0
regression reported in #943. It is filed here because it produces the exact
symptom described there, and because it was present in the configuration in use
when that issue was written.
