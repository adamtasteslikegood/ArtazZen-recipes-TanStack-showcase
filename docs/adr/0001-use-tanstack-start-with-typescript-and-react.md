# ADR-0001: Use TanStack Start with TypeScript and React

## Status

Proposed (author confirmed the direction on 2026-10-28; mark Accepted after the verification steps below)

## Context

This project is a demo that explores the TanStack framework and compares it with the author's other stacks (Flask/Python and TypeScript/Angular). Its content is a blog that combines recipes and photos (tasteslikegood.org) with artwork (artazzen.com).

Requirements that drive the decision:

- **Linkable, indexable pages.** Each recipe and each artwork needs its own route (for example `/recipes/<slug>`, `/art/<slug>`) that can be shared and crawled.
- **Fast first load.** Content pages should arrive as HTML rather than wait for a large client bundle.
- **Rich interactive components.** Galleries, lightboxes, animated or "designer" presentation of artwork and recipes should be possible, without turning the whole site into an SPA.
- **Explicit non-goal.** A client-rendered React SPA monolith.
- **Demo goal.** Exercise TanStack's type-safe routing and data loading, and produce a fair comparison with Flask and Angular.

## Options Considered

| Option | Rendering | Fit for linkable, indexable pages | Interactive React components | Notes |
|---|---|---|---|---|
| **TanStack Start + TypeScript + React** | SSR with streaming; static prerendering available | Good (per-route HTML) | Native | Built on TanStack Router and Vite; matches the demo goal |
| TanStack Router only (SPA) | Client-side | Weak by default; SSR needs manual wiring | Native | Rejected: the SPA shape is an explicit non-goal |
| TanStack Router + Solid | n/a | n/a | Solid, not React | Rejected: React components are the stated goal |
| Frameworks outside TanStack | Varies | Varies | Varies | Out of scope for this demo; see `docs/alternative-demo-comparisons.md` |

## Decision

Build the demo with **TanStack Start**, written in **TypeScript**, using **React** for the UI.

- **Routing:** file-based routes, one per recipe and per artwork, plus list and gallery routes.
- **Rendering:** server-render route content by default. Consider static prerendering for pages that rarely change.
- **Data loading:** route loaders on the server fetch or read content.
- **Interactivity:** React components for galleries and other "designer" presentation, applied to specific areas of a page and not to the whole page.
- **Content:** Markdown/MDX files in the repository as the starting point, subject to a later ADR on content model and storage.

## Consequences

### Positive

- Pages are server-rendered HTML with stable URLs, which suits search indexing and link sharing.
- TypeScript types extend across routes, params and search params.
- The same React components can be used for simple pages and for richer interactive displays.
- The server side (server functions and routes) maps to Flask views and endpoints, and SSR maps to Angular SSR, so the comparison is meaningful.

### Negative

- React still hydrates on the client, so static content pages carry JavaScript they might not need. This is the accepted overhead and it must be managed deliberately.
- Interactive components can erode load speed if used heavily.
- Start's maturity and API stability must be checked. A younger framework may bring breaking changes or gaps in documentation.

### Neutral

- A hosting target with a server or edge runtime is needed for SSR, unless pages are fully prerendered.

## Managing the accepted overhead

The author wants flashy, interactive components while keeping the site fast. Practices to apply and measure:

- Keep interactivity out of pages that do not need it. Prefer plain server-rendered markup for body text and recipe steps.
- Split code by route and lazy-load heavy components such as galleries or animation libraries.
- Serve responsive, modern-format images with explicit dimensions and lazy loading below the fold. Images are likely the largest cost on this site.
- Render the core content on the server so it is readable before hydration.
- Set a performance budget (for example Core Web Vitals targets and a maximum JS size per route) and check it as the demo grows.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Start API changes or immature areas | Medium | Medium | Pin versions, read current docs before scaffolding, keep the app small |
| Heavy components slow content pages | Medium | High | Performance budget, lazy loading, review per route |
| Image weight dominates load time | High | High | Image optimisation strategy decided before importing content |
| Deployment needs a server runtime | Low | Medium | Prerender where possible; choose a host that supports the chosen adapter |

## Verification before moving to Accepted

This ADR was written from general knowledge and has not been checked against current documentation.

1. Confirm TanStack Start's current release status and recommended project setup on tanstack.com.
2. Confirm how SSR, streaming and static prerendering are configured in the current version.
3. Confirm the current options for deployment targets.
4. Scaffold a minimal app with one recipe route and one artwork route, and check that each returns server-rendered HTML.

## Implementation

1. Complete the verification steps above.
2. Scaffold a Start project with TypeScript.
3. Write a follow-up ADR on the content model (recipe, artwork, photo, tags) and storage (files vs database).
4. Implement the recipe list and detail routes, then the artwork gallery.
5. Add interactive components one at a time, measuring each against the performance budget.

## Monitoring

The decision is working if:

- Recipe and artwork pages return meaningful HTML without running client JavaScript.
- Each page has a stable, shareable URL.
- Performance stays inside the agreed budget as interactive components are added.
- The Flask and Angular comparison can be built on the same content without awkward workarounds.

Revisit this ADR if Start proves unstable, or if the performance budget cannot be met with React hydration.
