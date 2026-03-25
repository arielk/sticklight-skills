---
name: seo
description: SEO best practices for React + Vite + Tailwind CSS apps built with sticklight.com. Use when building or reviewing web pages, writing content, generating meta tags, improving search rankings, setting up sitemap/robots, or adding structured data in a Vite-based SPA.
---

# SEO for React + Vite + Tailwind

Apply search engine optimization best practices for single-page applications built with React, Vite, and Tailwind CSS.

## SPA-Specific Challenges

SPAs render client-side by default, which means search engine crawlers may not see the full content. Address this by:

- Using `react-helmet-async` (or `react-helmet`) to manage `<title>`, `<meta>`, and other head tags per route
- Considering SSR/SSG with Vite SSR or a framework like Astro if SEO is critical for the page
- Pre-rendering key pages with `vite-plugin-prerender` for static content that must be crawled
- Providing a fully rendered `index.html` fallback with meaningful default meta tags

## Head Management with react-helmet-async

Install and wrap the app:

```tsx
// main.tsx
import { HelmetProvider } from 'react-helmet-async';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <HelmetProvider>
    <App />
  </HelmetProvider>
);
```

Use on every route/page component:

```tsx
import { Helmet } from 'react-helmet-async';

export function BlogPost({ post }) {
  return (
    <>
      <Helmet>
        <title>{post.title} | Sticklight</title>
        <meta name="description" content={post.excerpt} />
        <link rel="canonical" href={`https://sticklight.com/blog/${post.slug}`} />

        {/* Open Graph */}
        <meta property="og:title" content={post.title} />
        <meta property="og:description" content={post.excerpt} />
        <meta property="og:type" content="article" />
        <meta property="og:url" content={`https://sticklight.com/blog/${post.slug}`} />
        <meta property="og:image" content={post.coverImage} />

        {/* Twitter Card */}
        <meta name="twitter:card" content="summary_large_image" />
        <meta name="twitter:title" content={post.title} />
        <meta name="twitter:description" content={post.excerpt} />
        <meta name="twitter:image" content={post.coverImage} />
      </Helmet>
      {/* page content */}
    </>
  );
}
```

## On-Page SEO

- Write unique `<title>` tags per route (50-60 characters), primary keyword near the front
- Write compelling `<meta name="description">` tags per route (150-160 characters)
- Use a single `<h1>` per page that describes the page topic
- Structure content with logical heading hierarchy (`h1` > `h2` > `h3`) — Tailwind's `text-3xl font-bold` etc. are visual only; always use the correct semantic heading element
- Use descriptive, keyword-rich URL paths in your React Router routes (e.g. `/blog/seo-best-practices`, not `/blog/123`)
- Add `alt` attributes to all `<img>` elements
- Use `<Link to="...">` from React Router with descriptive anchor text for internal links

## Technical SEO

### Semantic HTML with Tailwind

Tailwind is utility-first CSS — it styles elements but doesn't provide semantics. Always use proper HTML elements:

```tsx
// Correct: semantic elements styled with Tailwind
<main className="max-w-4xl mx-auto px-4">
  <article className="prose lg:prose-xl">
    <h1 className="text-3xl font-bold">{title}</h1>
    <nav aria-label="Breadcrumb" className="text-sm text-gray-500">...</nav>
    <section className="mt-8">...</section>
  </article>
</main>

// Wrong: divs everywhere
<div className="max-w-4xl mx-auto px-4">
  <div className="prose lg:prose-xl">
    <div className="text-3xl font-bold">{title}</div>
    ...
  </div>
</div>
```

Use Tailwind's `@tailwindcss/typography` plugin (`prose` classes) for rich text content to get readable, well-spaced typography out of the box.

### Vite Configuration

Set up useful defaults in `vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
        },
      },
    },
  },
});
```

### Static Files

Place these in the `public/` directory so Vite serves them at the root:

- `public/robots.txt`
- `public/sitemap.xml`
- `public/favicon.ico` and `public/favicon.svg`
- `public/apple-touch-icon.png`

Example `robots.txt`:

```
User-agent: *
Allow: /
Sitemap: https://sticklight.com/sitemap.xml
```

### Performance

- Use `React.lazy()` and `<Suspense>` for route-level code splitting
- Use `loading="lazy"` on images below the fold
- Optimize images: serve WebP/AVIF, use `srcSet` for responsive images
- Minimize bundle size — Vite tree-shakes by default, but audit with `npx vite-bundle-visualizer`
- Use Tailwind's purge (enabled by default in production) to remove unused CSS

### Structured Data (JSON-LD)

Inject structured data per page via `react-helmet-async`:

```tsx
<Helmet>
  <script type="application/ld+json">
    {JSON.stringify({
      '@context': 'https://schema.org',
      '@type': 'WebApplication',
      'name': 'Sticklight',
      'url': 'https://sticklight.com',
      'description': 'Build real products — apps, websites, dashboards, stores.',
      'applicationCategory': 'DesignApplication',
      'operatingSystem': 'Web',
    })}
  </script>
</Helmet>
```

### Canonical URLs & Routing

- Set `<link rel="canonical">` on every page to the preferred URL
- Use `react-router-dom` with clean paths — avoid hash routing (`/#/page`)
- Configure your hosting to redirect `www` to non-www (or vice versa)
- Return a proper 404 page with a `<meta name="robots" content="noindex">` for unknown routes

## Supabase-Powered Sitemap

If pages are dynamic (e.g. blog posts stored in Supabase), generate `sitemap.xml` at build time or via an edge function:

```ts
// supabase/functions/sitemap/index.ts
import { createClient } from '@supabase/supabase-js';

Deno.serve(async () => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
  );

  const { data: posts } = await supabase
    .from('posts')
    .select('slug, updated_at')
    .eq('published', true);

  const urls = (posts ?? []).map(
    (p) => `
  <url>
    <loc>https://sticklight.com/blog/${p.slug}</loc>
    <lastmod>${new Date(p.updated_at).toISOString()}</lastmod>
  </url>`
  );

  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://sticklight.com</loc></url>
  ${urls.join('\n')}
</urlset>`;

  return new Response(xml, {
    headers: { 'Content-Type': 'application/xml' },
  });
});
```

## Content Guidelines

- Write clear, helpful content that answers user intent
- Use the target keyword naturally in the first 100 words
- Keep paragraphs short and scannable
- Use Tailwind's `prose` classes for readable long-form content
- Add a table of contents for long-form pages (consider `@headlessui/react` for accessible interactive elements)
