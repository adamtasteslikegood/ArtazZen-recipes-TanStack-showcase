# Recipe & Art Blog (TanStack Demo)

> A recipe blog with photos and artwork, built as a demo web application to explore TanStack.

**Status:** Early stage. The repository is currently just a shell, and nothing has been built yet.

## About

This project has two purposes:

1. **A TanStack demo app.** It explores the TanStack framework and compares it with the author's other stacks: Flask/Python and TypeScript/Angular.
2. **A combined content site.** It is a blog that brings together the output of two existing projects:
   - [tasteslikegood.org](https://tasteslikegood.org): recipes and food photography
   - [artazzen.com](https://artazzen.com): artwork

The blog is the real-world content the framework comparison is built around.

## Goals

- Publish recipes, each with photos, in one place.
- Show artwork alongside the recipes, so food and art live in the same blog.
- Learn how TanStack handles routing, data loading and rendering, and compare that with Flask and Angular.

## Roadmap

These are planning notes, not commitments.

- [x] Choose the TanStack variant: Start with TypeScript and React (see [ADR-0001](docs/adr/0001-use-tanstack-start-with-typescript-and-react.md))
- [x] Define the content model: one blog with two content types, recipe and artwork (see [ADR-0002](docs/adr/0002-unified-blog-content-model-and-file-based-storage.md))
- [x] Decide how content is stored: files first, database as phase 2 (see ADR-0002)
- [ ] Agree the field-level schemas using sample recipe and artwork JSON
- [ ] Build the recipe list and detail pages
- [ ] Build the artwork gallery
- [ ] Bring in existing content from both sites
- [ ] Write up what was learned compared with Flask and Angular

## Getting Started

Setup instructions will be added once the application is scaffolded.

## Related Projects

- [tasteslikegood.org](https://tasteslikegood.org)
- [artazzen.com](https://artazzen.com)
