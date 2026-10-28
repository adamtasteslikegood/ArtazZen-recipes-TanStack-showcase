# Ideas for Separate Comparison Demos

These are suggestions for future, separate demos. They are not part of this project's scope (see ADR-0001) and are written from general knowledge, not checked against current documentation. Verify the details before building any of them.

The idea throughout: keep the same content and the same goals (linkable routes, fast first load, interactive components) and change only the framework.

| Demo idea | What it would show | Main question to answer |
|---|---|---|
| **Astro** (islands architecture) | Mostly static HTML, with interactive React components loaded only where needed | How much JavaScript is saved on content pages compared with Start? |
| **Next.js** (App Router, server components) | The most widely used React meta-framework, with server components | How do its rendering model and conventions compare with TanStack Start? |
| **React Router v7 / Remix** | Another React framework with loader-based data loading | How do its loaders and routing compare with TanStack's typed routes? |
| **SvelteKit** | A compiler-based UI framework with less runtime JS | What is the size and speed difference for the same pages? |
| **Nuxt** (Vue) | A full-stack framework in another ecosystem | How does the developer experience compare? |
| **Angular SSR** | Angular's own SSR and hydration | A direct comparison with the author's existing Angular skills |
| **Flask + Jinja + htmx** | Server-rendered templates with small interactive pieces | How far does a minimal-JS approach go before React is needed? |
| **Eleventy or another static site generator** | Pure static output | What is the performance floor for a content-only blog? |

## Cross-cutting comparisons

These could be applied to any of the demos:

- **Image pipeline:** build-time optimisation vs a hosted image service.
- **Content storage:** Markdown/MDX files vs a headless CMS vs a database.
- **Performance:** the same budget (Core Web Vitals, JS per route) measured on each.
- **Deployment:** static host vs server runtime vs edge.
- **Developer experience:** type safety, build times, and effort to add a new route.

## Suggested order

Astro first: it directly tests the "accepted overhead" question from ADR-0001. Next.js second, as the most common alternative in the React world. Then Flask + htmx, which fits the author's existing Flask experience.
