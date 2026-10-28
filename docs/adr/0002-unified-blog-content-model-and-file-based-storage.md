# ADR-0002: Unified blog content model with file-based storage first

## Status

Proposed (the author settled the direction on 2026-10-28; field-level schemas are pending samples of the existing data)

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
- A common envelope of fields that every entry has (proposed: `type`, `slug`, `title`, `date`, `tags`, `images`, `summary`).
- Type-specific fields for each content type, defined once schemas are confirmed.
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

### 3. Keep storage behind one interface

Routes and components do not read files directly. They call a small content-access module (for example `getRecipe(slug)`, `listArtwork()`). Only that module knows content comes from files. Phase 2 can then add a database by writing a second implementation of the same interface, without touching routes or components.

### 4. Database is phase 2

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

1. **Samples needed.** One exported recipe JSON and one art sidecar JSON would let us define the real schemas. I have not seen either, so the proposed envelope fields above are a starting point and not a mapping of the existing data.
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
