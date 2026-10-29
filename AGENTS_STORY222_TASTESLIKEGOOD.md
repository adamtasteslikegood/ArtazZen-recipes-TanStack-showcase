# AGENTS_STORY222: An agent visits tasteslikegood.org

First-person field notes from an LLM agent (Claude, acting as the Systems Architect on the
ArtazZen-recipes-TanStack-showcase project) using tasteslikegood.org the way a "model web
citizen" would. Tested 2026-10-29. Raw responses are in `docs/samples/tasteslikegood/`.

## The story

I arrived with nothing but `curl`. No browser, no account, no API key.

1. **Knock on the front door.** `curl -L -H 'Accept: text/markdown' https://tasteslikegood.org/`
   The apex 301s to `www`, and then I got `200 text/markdown` with `Vary: Accept`. No HTML to
   strip, no scripts, no cookie banner. The same URL gives a browser HTML, so there is one
   URL per thing and the representation is negotiated. This is the best part of the experience.
2. **Read the map.** `/llms.txt` told me in about 2.6 KB what exists: sitemap, API catalog,
   OpenAPI, and an MCP server ("read-only, no account or key needed"). I did not have to guess.
3. **Follow the catalog.** `/.well-known/api-catalog` (RFC 9727 linkset) points each endpoint
   at `service-desc` (the OpenAPI) and `service-doc` (llms.txt).
4. **Use the tools.** MCP over Streamable HTTP at `/mcp`: `initialize`, `tools/list`,
   `search_recipes`, `get_recipe`. All worked, with JSON responses and no session.
5. **Fall back to REST.** The OpenAPI 3.1 spec lists five operations. All worked.
6. **Credit the source.** Every payload carried an attribution string asking me to credit
   Tasteslikegood.org and link the recipe. As an agent I appreciated being told this
   explicitly instead of guessing the etiquette.

## What worked well

| Area | Observation |
|---|---|
| Content negotiation | `Accept: text/markdown` works on `/`, `/browse`, `/r/<slug>`. `Vary: Accept` is set. |
| Discovery | llms.txt, sitemap.xml, api-catalog, OpenAPI, and `Link` headers on the home page. |
| MCP | Correct JSON-RPC: `-32601` for an unknown method, `202` for a notification, `isError: true` for tool-level failure ("Recipe not found"). |
| MCP results | Both `content[].text` (stringified JSON) and `structuredContent` are returned, so clients of either kind are served. |
| Tool schemas | `additionalProperties: false`, `limit` has default 10 and max 20, `slug` is described as "from search_recipes". |
| REST errors | 404 returns JSON `{"error":"Recipe not found"}`, not an HTML page. |
| Privacy | The public recipe payload has no `user_id` and no `ai_metadata` (unlike the raw export in `docs/`). |
| Rate limits | `ratelimit-policy: 300;w=900` headers are present, so an agent can pace itself. |
| Image | `/api/recipes/<id>/image` returns `image/jpeg`, `cache-control: public, max-age=86400`. |
| md4a CLI | `md4a fetch` output was byte-identical to curl (earlier test, 5778 bytes). |

## Issues and quirks found (prioritized)

1. **HTTP/2 POST to `/mcp` is reset.** `curl` (HTTP/2) got `stream reset ... PROTOCOL_ERROR` on
   every POST to `/mcp`. The same request over `--http1.1` returns 200. GET `/mcp` over h2 works
   (405, `Allow: OPTIONS, POST`). Agents using h2-by-default HTTP clients may fail with no
   useful message. This is the most important one to investigate (Cloud Run / Google Frontend
   in front of the app; possibly a request-body or header issue on the h2 path).
2. **`Set-Cookie: session=...` on a public, keyless API.** Even the MCP `initialize` response sets a
   `session` cookie (`Domain=tasteslikegood.org; HttpOnly`). With `cache-control: private` this
   defeats shared caching and gives anonymous agents a pointless session.
3. **CORS looks like a dev leftover.** `access-control-allow-origin: http://127.0.0.1:3000` plus
   `access-control-allow-credentials: true` on production responses.
4. **Notification returns `text/html`.** `notifications/initialized` returns `202` with
   `content-type: text/html`; an empty body should not claim HTML.
5. **No `Link` header on recipe pages.** The home page advertises llms.txt/sitemap/api-catalog,
   but `/r/<slug>` does not, so an agent landing deep on a recipe has no breadcrumb to the tools.
6. **Markdown ingredient lines are split.** Each ingredient renders as a bullet with amount,
   unit and name on separate indented lines:
   `* 1.5` / `  Tbsp` / `  Vegan butter *(notes)*`. It reads fine but is awkward to parse and
   wastes tokens; `* 1.5 Tbsp Vegan butter (notes)` would be better.
7. **Relative image URL in markdown.** `![...](/api/recipes/<id>/image?w=400&v=...)` is relative.
   Agents that only keep the markdown lose the host. Use an absolute URL (the JSON already does).
8. **Page chrome in the markdown.** "Save to your cookbook" and "Save to Pinterest" links are
   human-action chrome. An agent cannot use them, and they cost tokens on every page.
9. **No token-count or content-signal headers** (for example `x-markdown-tokens` or a
   `Content-Signal` policy). Optional, but useful for agents budgeting context.
10. **`/recipes` with a markdown Accept returns a 404 HTML page** (earlier test). A negotiated
    404 should be markdown or JSON too.
11. **Search is name-only.** The tool description says so ("by name"), and `chili` returned an
    empty list. An empty result is not an error. A hint such as "try fewer words, or use
    `/browse/tag/<tag>`" would help agents recover.
12. **Search is the only way to list.** MCP has no `list_recent` tool; `newest-recipes` exists
    only in REST. A parity tool would let MCP-only agents browse.
13. **Dates are inconsistent.** `newest-recipes` has `first_published_at`; the full public
    recipe has no date at all. This matters for our blog model (ADR-0002 open item: recipe date).
14. **Duplicated payloads.** MCP returns the data twice (`content[].text` and `structuredContent`).
    That is spec-compliant but doubles the tokens for clients that read both.

## Notes for our TanStack Start build (md4a-ts)

- Copy the good parts: one URL, `Accept` negotiation, `Vary: Accept`, llms.txt, api-catalog, a
  tiny read-only MCP route, JSON errors, attribution field.
- Generate markdown from the **typed content model**, not by converting HTML. That avoids items
  6 to 8 (split ingredient lines, relative image URLs, human-only chrome) by construction.
- Emit a `Link` header on every content route, not only the home page.
- Do not set cookies on agent/API routes. Set an explicit CORS allow-list.
- Test the MCP route over HTTP/2 in CI, since that is where tasteslikegood.org breaks.
- Give artwork the same treatment: absolute image URLs plus alt text and dimensions from the
  ImageSidecar fields, in both markdown and JSON.
- Our unified blog model needs a real `date` for recipes; the source site only exposes
  `first_published_at` in the newest list.

## Samples saved

`docs/samples/tasteslikegood/`: `openapi.json`, `mcp/*.json` (initialize, tools_list, search,
search_empty, get_recipe, get_bad, bad_method, notif; each ends with an `[HTTP ...]` trailer line),
`rest/*.json` and `rest/*.md` (three recipes in both JSON and markdown, tool list, newest list).

## Not tested

Authenticated/user features, write operations, behaviour at the 300 req / 15 min limit,
and whether the h2 reset reproduces with non-curl clients.
