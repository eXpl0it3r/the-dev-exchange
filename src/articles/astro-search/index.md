# Adding Search to an Astro Site

Once your blog grows to a certain size, you may want to provide different ways to discover existing articles beyond just scrolling through the given chronological list.
This could take different forms such as having categories, tag clouds, featured list, recommendations, or a site search.
One option doesn't exclude the other, so there isn't a specifically better or worse starting point.

For The Dev Exchange blog, I decided to add a basic site-wide search.
The main criteria were:

- Needs to work for a static sites, i.e. no dynamic backend
- Integrates well with Astro, the website framework used for this blog
- Doesn't add a lot of complexity, neither in dependencies nor code-wise

## pagefind

A brief querying of the preferred LLM and even an old school web search, recommended pagefind as a very lightweight search library built for static websites.
The idea is to generate a search index during the build of the static site, which can then be statically served to the frontend search code.
Pagefind also implements various optimizations to help with large sites, so that only a small amount of data needs to be transferred to the client.

Next to creating and serving that search index, pagefind offers a pre-built UI, filtering, sorting, and essentially anything else one would expect from a search.
It's the prefect match for an existing static website that "just wants to add search".

## astro-pagefind

Unsurprisingly, there's already a dedicated Astro integration available, called astro-pagefind.
Instead of having to fiddle with manually integrating pagefind and adjusting build scripts, astro-pagefind takes care of that, allowing you to continue developing the Astro site the same way you've gotten used to.
You don't need to call npx pagefind, but can build Astro with npm run build / npx astro build.

However, there's one caveat when it comes to the Astro dev server.
In order to build a search index, we need to build the website, but in order to build the website with a search, we need to have a search index.
The way astro-pagefind solves this, is to use the search index of last site build from the `dist/` (i.e. `outDir`) directory. This means, that when running `npm run dev` / `npx astro dev` without previously running `npm run build` / `npx astro build`, the search index won't be available.

In short: If you want to search pages during the development process, you need to build the website at least once before starting the dev server.

## Usage

The idea for The Dev Exchange blog is to use the pre-built astro-pagefind search component and restyle it to match the look and feel of the blog.

But before using pagefind, the integration needs be prepared by adding a modal that opens over the page with a search input field and in the navigation header a button with a search icon.

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

Next astro-pagefind needs to be installed with `npm i astro-pagefind` or your favorite package manager command.

Then the placeholder search input field gets replaced with the pre-built component, like so:

```mdx
---
import Search from "astro-pagefind/components/Search";
---

<Search />
```

Finally, the style needs to be changed by overwriting pagefind's CSS variables globally and fine tuning specific section, like hiding the clear button in favor of a general close button.

All in all the final search integration looks like so:

GIF

## Next Steps

As previously mentioned, pagefind comes with more options than "just" a text search. Making use of the filtering capability for tags or authors could turn out to be quite useful.
However, for tags The Dev Exchange blog still needs some extra work first, as the tags are current not displayed anywhere.
On the upside, having a search that can filter for tags, would allow for turning tags into links, that open the search with the given tag pre-selected.
So one might be able to circumvent the need for a separate tag list page.
Additionally, the tags would need to be reviewed a bit more, so that they are consistent across articles and maybe also reduced to the most fitting tags.

Pagefind does support showing thumbnails in the search results and while it generally worked out of the box, it for some reason missed to fetch the cover image at times.
Given the odd blank space in most searches, I disable the feature for now, but if the root cause were to be found, it could be re-enabled.

Currently, there's the issue that pages are indexed based on the markdown files and you may get image URLs as a result in the search.
So far I've been unable to find someone talking about this and as such also couldn't find any solution.
I may try to reach out to the author of astro-pagefind or authors of pagefind itself to determine if this could be somehow worked around.