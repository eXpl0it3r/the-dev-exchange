# Adding Search to an Astro Site

As your blog grows, simply scrolling through a chronological list of posts quickly becomes unwieldy.
Readers need better ways to discover content - be it categories, tag clouds, featured lists, recommendations, or a good old-fashioned site search.
There's no single "best" approach, each method can complement the others, and you can always layer them as your site evolves.


For The Dev Exchange blog, I decided it was time to add a basic site-wide search.
My criteria were straightforward:


- Must work for static sites; no dynamic backend
- Integrates smoothly with Astro, the framework powering this blog
- Keeps things simple with minimal dependencies and few code changes


## pagefind

A quick chat with the preferred LLM (and, yes, a classic web search) pointed me to pagefind, a lightweight search library built for static sites.
The idea is to generate a search index at build time, then serve it statically to the frontend.
Pagefind also implements various optimizations to help with large sites and transfers only what's needed to the client.

Beyond just creating and serving that search index, pagefind comes with a pre-built UI with filtering, sorting, and all the essentials you'd expect from a modern search experience.
It's the perfect match for an existing static site that "just wants to add search".


## astro-pagefind

Unsurprisingly, there's already a dedicated Astro integration: [astro-pagefind](https://github.com/shishkin/astro-pagefind)
Instead of having to fiddle with manually integrating pagefind and adjusting page build scripts, astro-pagefind takes care of that, letting you keep your usual Astro dev workflow.
You don't need to call `npx pagefind`, but can build Astro as usual with `npm run build` or `npx astro build`.

However, there is one small caveat with the Astro dev server: To build a search index, you need a built site; but to build the site with search, you need a search index.
The way astro-pagefind solves this, is by reusing the last built search index from the `dist/` (i.e. `outDir`) directory.
This means, that when running `npm run dev` or `npx astro dev` without previously running `npm run build` or `npx astro build`, the search index won't be available.

In short: To use search during development, build the site at least once before starting the dev server.


## Usage

For The Dev Exchange, I wanted to use the pre-built astro-pagefind search component, but with a look and feel that matches the rest of the blog.
Before dropping in pagefind, I needed a modal overlay for the search input, and a search button in the navigation header.


### Modal

```astro
---
import Search from "astro-pagefind/components/Search";
---

<div id="search-modal">
  <!-- astro-pagefind Search component -->
  <Search id="search" className="pagefind-ui" uiOptions={{ showImages: false }} />
</div>

<style is:global>
  /* Overrides of the astro-pagefind component styles */
  :root {
    --pagefind-ui-scale: 0.8;
    --pagefind-ui-primary: var(--color-dark);
    /* ... */
    &:where(.dark, .dark *) {
      --pagefind-ui-primary: var(--color-light);
      /* ... */
    }
  }
</style>
<script>
  function initSearchModal() {
    // Handle modal events
  }

  document.addEventListener('astro:page-load', initSearchModal);
</script>
```


### Header

```astro
---
import SearchModal from './SearchModal.astro';
---

<!-- ... -->
<button id="search-button" aria-label="Open search">
  <svg fill="none" height="30" stroke="currentColor" stroke-linecap="round"
    stroke-linejoin="round" stroke-width="2" viewBox="0 0 24 24" width="30"
    xmlns="http://www.w3.org/2000/svg">
    <circle cx="11" cy="11" r="8"/>
    <path d="m21 21-4.35-4.35"/>
  </svg>
</button>
<!-- ... -->
```


Next, install astro-pagefind with `npm i astro-pagefind` (or your package manager of choice).

Replace your placeholder search input with the pre-built component:


```mdx
---
import Search from "astro-pagefind/components/Search";
---

<Search />
```


Finally, tweak the style by overwriting pagefind’s CSS variables globally and fine-tuning specific sections—like hiding the clear button in favor of a general close button.

All in all, the final search integration looks like this:

GIF


## Next Steps

As mentioned, pagefind offers more than just text search. Its filtering capabilities—by tag or author—could be a real asset. For now, The Dev Exchange doesn’t display tags anywhere, so there’s some groundwork to do. But imagine: tags as clickable links that open search with the tag pre-selected. That could even make a separate tag list page unnecessary. Of course, the tags themselves need a bit of review to ensure consistency and relevance.

Pagefind also supports thumbnails in search results. While it mostly worked out of the box, sometimes the cover image just wouldn’t show, leaving an awkward blank space. For now, I’ve disabled the feature, but if I can track down the root cause, I’d love to re-enable it.

One current quirk: pages are indexed based on the markdown files, so sometimes image URLs show up in search results. I haven’t found much discussion or a solution for this yet. If all else fails, I might reach out to the astro-pagefind or pagefind authors to see if there’s a workaround.