# ADR-0002: Unified blog content model with file-based storage first

## Status

Proposed (the author settled the direction on 2026-10-28; the recipe schema now follows the served payload of tasteslikegood.org, checked against three live samples, and the art schema comes from the real sidecar schema; field mappings stay provisional until a real art sidecar is checked)

## Context

ADR-0001 chose TanStack Start with TypeScript and React for a blog that combines the output of tasteslikegood.org (recipes and photos) and artazzen.com (artwork). Two things were left open: the content model and the content storage.

What the author told us about the existing sites (not yet verified against real files):

- **tasteslikegood.org** stores recipes as JSON. They began as individual `.json` files and now live in a PostgreSQL table with a `data` column holding the JSON string. The live site can still export recipes as JSON.
- **artazzen.com** stores artwork metadata in JSON sidecar files, one beside each image.
- Both sites keep images outside the JSON.

The author's decisions:

1. The target is **one blog with two content types** (recipe and artwork), not two separate sites.
2. Start with **files** as the content source. Revisit a **database as phase 2**, once the file-based version works.

## Decision

### 1. One blog, two content types

A single site, with `recipe` and `artwork` as content types in one shared system:

- Shared routes and layout, with type-specific detail pages, for example `/recipes/<slug>` and `/art/<slug>`.
- A common envelope of fields that every entry has (`type`, `slug`, `title`, `date`, `tags`, `images`, `summary`). The envelope is derived at import time and mapped from the existing fields; see "Field mapping from the real data" below.
- Type-specific fields for each content type, kept close to the source data.
- Cross-type features, such as a combined feed, shared tags and side-by-side views, come from the common envelope. This is what lets the demo show "both sides" without running two sites.

### 2. File-based storage, structured data in JSON and prose in Markdown

The author suggested Markdown as the page source if that is the convention. Markdown with frontmatter is the usual convention for blog prose. However, the existing data is structured JSON (recipes in particular have lists such as ingredients and steps), and that fits poorly in frontmatter. So the proposed split is:

- **JSON** holds the structured data, in a shape as close as possible to the existing recipe export and art sidecar files, so migration is a copy and a light transform.
- **Markdown (or MDX)** holds optional long-form prose, such as a recipe story or notes about an artwork, linked to the entry by slug.
- Images stay as separate files, referenced from the JSON by path.

Proposed layout (illustrative, to be adjusted):

```
content/
  recipes/
    <slug>/
      recipe.json      # structured data (from the existing export)
      body.md          # optional narrative
      <images>
  art/
    <slug>/
      artwork.json     # metadata (from the existing sidecar)
      body.md          # optional notes
      <image>
```

Entries are validated against a schema when loaded, so bad data fails the build and not the page.

### 3. Field mapping from the real data

Sources: `docs/ImageSidecar.schema.json` (the artwork sidecar schema from artazzen.com, a JSON Schema document) and `docs/Recipe.schema.json` (the shape served by tasteslikegood.org, from three samples in `docs/samples/tasteslikegood/rest/`; the author states served data is always migrated to the current schema, so the served shape is the source of truth). Both schemas validate against the JSON Schema 2020-12 meta-schema, and the three served recipes and the older export all validate against the recipe schema.

| Envelope field | Recipe source | Artwork source |
|---|---|---|
| `type` | constant `recipe` | constant `art` |
| `slug` | `slug` (a real field in the served payload; derive from `name` only for the older export) | **none in the data**; derive from the image filename (the sidecar sits beside the image) |
| `title` | `name` | `title` |
| `summary` | `description` | `caption`, falling back to `description` |
| `tags` | `tags` | `tags` |
| `date` | **none in the served payload**; the export's `ai_metadata.image_generation.timestamp` is internal and stripped, so recipes have no date unless one is added | `detected_at` (Unix epoch seconds) |
| `images` | `image` (absolute URL on the live site's API, not a file; same value as `ai_image_url`) | the image file beside the sidecar |

Recipe-specific fields: `ingredients` (an object grouped as `dry`, `wet` and `other`, each a list of `{amount, name, units, notes?}`), `instructions` (served as a list of plain strings; the older export uses `{step, description}` objects, which the importer should flatten to strings), `prepTime`, `cookTime`, `servings`, `notes`. Times have no unit in the data (minutes assumed).

Artwork-specific fields: `description`, `caption`, `artist`, `copyright`, `collections` and `series` (slug arrays), `status` (`pending`, `approved` or `hidden`).

Rules that follow from the data:

- **Publish only approved artwork.** The sidecar has a curation `status`; the importer should include only `approved` items.
- **Use the current fields, not the deprecated ones.** The sidecar marks `reviewed`, `author` and `collection` as deprecated in favour of `status`, `artist` and `collections`.
- **Strip internal fields at import, before content reaches the repository.** The older recipe export carries `ai_metadata` (prompts, model names, session ids), `ai_image_gcs` (an internal storage bucket path), `image_keywords` and `user_id`; the served payload omits all four (grep of the saved samples found no `user_id` or `ai_metadata`). Served-only fields that are not content, `breadcrumbs`, `is_public` and `stock_image_url`, are also dropped, and only recipes with `is_public: true` are imported. Art sidecars carry `ai_details` (including `raw_response`, `error_body` and the prompt) and `ai_fields`. None of this is needed for display, and committing it would publish internal detail. Keep only the fields the site renders.
- **Images need a download step.** The served `image` is an absolute URL on the live site (the export's `ai_image_url` is a relative path), so the importer must fetch the image and store it as a file.
- **Slugs and dates.** Served recipes have a real `slug`, so only artwork needs a derived slug (decide the rule and collision handling). Recipes have no date in the served payload; decide whether to show none, add one at import, or take a field from the live database if the author exposes one.

### 4. Keep storage behind one interface

Routes and components do not read files directly. They call a small content-access module (for example `getRecipe(slug)`, `listArtwork()`). Only that module knows content comes from files. Phase 2 can then add a database by writing a second implementation of the same interface, without touching routes or components.

### 5. Database is phase 2

PostgreSQL is not part of phase 1. Revisit it once the file-based version works. Triggers for revisiting include:

- Content volume making builds or loads slow.
- Needing editing, search or filtering that files handle poorly.
- Wanting to share one source of truth with the live tasteslikegood.org database.

When revisited, the existing `data` column (JSON string) maps naturally onto the same JSON documents, which is a point in favour of keeping JSON as the structured format now.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| **A. Markdown with frontmatter only** | The usual blog convention; simple authoring | Structured recipe data fits frontmatter poorly; existing JSON would need rewriting |
| **B. JSON only** | Matches existing data exactly; easy to validate | No natural place for long-form prose |
| **C. JSON for data plus optional Markdown (chosen)** | Reuses existing data, supports prose, maps to a later database | Two files per entry; needs a convention to link them |
| D. Database from the start | Single source of truth; good querying | More moving parts for a demo; deferred by the author |
| E. Two separate sites | Simple per-site logic | Contradicts the author's decision; more moving parts later |

## Consequences

### Positive

- Fewer moving parts in phase 1: no database to run, and content is versioned in git.
- Existing recipe exports and art sidecars can be reused with little change.
- One content system supports cross-type design features, which is the point of the demo.
- The access interface keeps the phase 2 database change local.

### Negative

- Two files per entry (data and prose) need a clear convention.
- Large image sets in git may become heavy; an image storage decision is still needed.
- Relationships between recipes and artwork, if wanted, must be modelled explicitly.

### Neutral

- File-based content suits static prerendering, which ADR-0001 lists as an option.

## Open items

1. **More samples.** The recipe schema is checked against three served recipes, so which fields are optional (for example `notes`, `units`, or ingredient groups other than `dry`, `wet` and `other`) is still unverified, and `amount` may be a string in some recipes. One real art sidecar plus its image would confirm the artwork mapping. The tasteslikegood.org database schema has not been supplied; the recipe export is the only evidence so far.
1a. **Slug and date rules** (see the mapping above).
2. **Image storage.** In the repository, a separate asset host, or a hybrid. This affects build size and the image optimisation pipeline.
3. **Markdown or MDX.** Plain Markdown is simpler; MDX allows embedding React components in prose, which supports the "designer" goals but adds build complexity.
4. **Linking recipes and artwork.** Whether entries can reference each other.
5. **Schema validation tool.** Choose one when schemas are defined.

## Implementation

1. Obtain sample recipe and art JSON files and agree the common envelope and per-type fields.
2. Define TypeScript types and a validation schema for each content type.
3. Create the content-access module over the file layout.
4. Import a small set of real entries of each type.
5. Build the list and detail routes on top of the access module.

## Monitoring

The decision is working if:

- Existing recipe and artwork data imports with only light transformation.
- Routes never read files directly.
- A new content type could be added without changing existing ones.
- The access interface can be implemented over a database without changes to routes, once phase 2 begins.
