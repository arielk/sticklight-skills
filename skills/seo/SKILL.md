---
title: SEO
description: SEO best practices for React + Vite + Tailwind CSS apps built with sticklight.com. Use when building or reviewing web pages, writing content, generating meta tags, improving search rankings, setting up sitemap/robots, adding structured data, or optimizing images and performance in a Vite-based SPA.
---

# SEO Skill for Sticklight Apps

This skill helps you build React apps with proper SEO implementation for search engines and social sharing.

## Overview

SPAs (Single Page Applications) need special SEO handling because search engines may not execute JavaScript. This skill covers:
- Static fallback meta tags in `index.html`
- Dynamic SEO with `@unhead/react`
- Sitemaps (dynamic and static)
- robots.txt configuration
- Semantic HTML structure
- Image optimization
- Performance best practices

---

## 1. Static SEO in index.html

Add static meta tags as fallbacks for crawlers that don't execute JavaScript. Use `data-hid` attributes so @unhead/react can replace them when React hydrates.

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  
  <title>Site Name | Your tagline here</title>
  <meta name="description" content="Your site description" data-hid="description">
  
  <!-- Open Graph -->
  <meta property="og:title" content="Site Name | Your tagline here" data-hid="og:title">
  <meta property="og:description" content="Your site description" data-hid="og:description">
  <meta property="og:type" content="website" data-hid="og:type">
  <meta property="og:image" content="/og-image.jpeg" data-hid="og:image">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" data-hid="twitter:card">
  <meta name="twitter:title" content="Site Name | Your tagline here" data-hid="twitter:title">
  <meta name="twitter:description" content="Your site description" data-hid="twitter:description">
  <meta name="twitter:image" content="/og-image.jpeg" data-hid="twitter:image">
  
  <!-- Canonical -->
  <link rel="canonical" href="https://yourdomain.com" data-hid="canonical">
</head>
```

**Important:** The `data-hid` attribute is required for @unhead/react to identify and replace these tags with dynamic values.

---

## 2. Dynamic SEO with @unhead/react

### 2.1 Create the Provider

Create `src/providers/UnheadProvider.tsx`:

```typescript
import { type ReactNode } from 'react';
import { createHead, UnheadProvider } from '@unhead/react/client';

const head = createHead();

export function AppUnheadProvider({ children }: { children: ReactNode }) {
  return <UnheadProvider head={head}>{children}</UnheadProvider>;
}
```

### 2.2 Wrap Your App

In `src/main.tsx`, wrap your app with the provider:

```typescript
import { AppUnheadProvider } from '@/providers/UnheadProvider';

<AppUnheadProvider>
  <App />
</AppUnheadProvider>
```

### 2.3 Create the useSEO Hook

Create `src/hooks/useSEO.ts`:

```typescript
import { useHead } from '@unhead/react';

const SITE_NAME = 'Your Site Name';
const SITE_TITLE = 'Your Site Name | Your tagline';
const BASE_URL = 'https://yourdomain.com';
const DEFAULT_DESCRIPTION = 'Your default description';
const DEFAULT_IMAGE = '/og-image.jpeg';

export interface SEOProps {
  title?: string;
  description?: string;
  image?: string;
  url?: string;
  type?: 'website' | 'article';
  publishedTime?: string;
  modifiedTime?: string;
  author?: string;
  noIndex?: boolean;
}

export function useSEO({
  title,
  description = DEFAULT_DESCRIPTION,
  image = DEFAULT_IMAGE,
  url = '',
  type = 'website',
  publishedTime,
  modifiedTime,
  author,
  noIndex = false
}: SEOProps = {}) {
  const fullTitle = title ? `${title} | ${SITE_NAME}` : SITE_TITLE;
  const fullUrl = `${BASE_URL}${url}`;
  const fullImage = image?.startsWith('http') ? image : `${BASE_URL}${image}`;

  useHead({
    title: fullTitle,
    meta: [
      { key: 'description', name: 'description', content: description },
      { key: 'og:type', property: 'og:type', content: type },
      { key: 'og:title', property: 'og:title', content: fullTitle },
      { key: 'og:description', property: 'og:description', content: description },
      { key: 'og:image', property: 'og:image', content: fullImage },
      { key: 'og:url', property: 'og:url', content: fullUrl },
      { key: 'og:site_name', property: 'og:site_name', content: SITE_NAME },
      { key: 'twitter:card', name: 'twitter:card', content: 'summary_large_image' },
      { key: 'twitter:title', name: 'twitter:title', content: fullTitle },
      { key: 'twitter:description', name: 'twitter:description', content: description },
      { key: 'twitter:image', name: 'twitter:image', content: fullImage },
      ...(publishedTime ? [{ key: 'article:published_time', property: 'article:published_time', content: publishedTime }] : []),
      ...(modifiedTime ? [{ key: 'article:modified_time', property: 'article:modified_time', content: modifiedTime }] : []),
      ...(author ? [{ key: 'article:author', property: 'article:author', content: author }] : []),
      ...(noIndex ? [{ key: 'robots', name: 'robots', content: 'noindex, nofollow' }] : [])
    ],
    link: [
      { key: 'canonical', rel: 'canonical', href: fullUrl }
    ]
  });
}
```

### 2.4 JSON-LD Structured Data for Articles

Add this to the same `useSEO.ts` file for blog posts:

```typescript
export function useArticleSchema({
  title,
  description,
  image,
  url,
  publishedTime,
  modifiedTime,
  authorName
}: {
  title: string;
  description: string;
  image?: string;
  url: string;
  publishedTime: string;
  modifiedTime?: string;
  authorName: string;
}) {
  const fullUrl = `${BASE_URL}${url}`;
  const fullImage = image?.startsWith('http') ? image : image ? `${BASE_URL}${image}` : `${BASE_URL}${DEFAULT_IMAGE}`;

  useHead({
    script: [
      {
        key: 'article-schema',
        type: 'application/ld+json',
        innerHTML: JSON.stringify({
          '@context': 'https://schema.org',
          '@type': 'Article',
          headline: title,
          description,
          image: fullImage,
          url: fullUrl,
          datePublished: publishedTime,
          dateModified: modifiedTime || publishedTime,
          author: { '@type': 'Person', name: authorName },
          publisher: { '@type': 'Organization', name: SITE_NAME, url: BASE_URL }
        })
      }
    ]
  });
}
```

### 2.5 Usage in Pages

**Homepage:**
```typescript
useSEO({ url: '/', type: 'website' });
```

**Blog Post:**
```typescript
useSEO({
  title: post.title,
  description: post.excerpt,
  image: post.cover_image_url,
  url: `/post/${post.slug}`,
  type: 'article',
  publishedTime: post.published_at,
  modifiedTime: post.updated_at,
  author: post.author_name
});

useArticleSchema({
  title: post.title,
  description: post.excerpt,
  image: post.cover_image_url,
  url: `/post/${post.slug}`,
  publishedTime: post.published_at,
  modifiedTime: post.updated_at,
  authorName: post.author_name
});
```

**404 Page:**
```typescript
useSEO({
  title: 'Page Not Found',
  description: 'The page you are looking for does not exist.',
  noIndex: true
});
```

---

## 3. Sitemap

### Option A: Dynamic Sitemap (with Cloud Backend)

If the app uses Cloud Backend, create an Edge Function that queries the database and returns an XML sitemap. This automatically includes new content.

**Important:** Use `SUPABASE_ANON_KEY` (not service role) to ensure the sitemap only includes content that is publicly accessible via RLS policies.

**Edge Function** - `supabase/functions/sitemap/index.ts`:

```typescript
import { createClient } from 'npm:@supabase/supabase-js@2';

const SITE_URL = 'https://yourdomain.com';

interface Post {
  slug: string;
  updated_at: string;
  published_at: string | null;
}

Deno.serve(async () => {
  try {
    const supabaseUrl = Deno.env.get('SUPABASE_URL')!;
    const supabaseKey = Deno.env.get('SUPABASE_ANON_KEY')!;
    
    const supabase = createClient(supabaseUrl, supabaseKey);

    const { data: posts, error } = await supabase
      .from('posts')
      .select('slug, updated_at, published_at')
      .not('published_at', 'is', null)
      .order('published_at', { ascending: false });

    if (error) {
      console.error('Database error:', error);
      return new Response('Error generating sitemap', { status: 500 });
    }

    const xml = generateSitemapXml(posts || []);

    return new Response(xml, {
      headers: {
        'Content-Type': 'application/xml',
        'Cache-Control': 'public, max-age=3600',
      },
    });
  } catch (err) {
    console.error('Sitemap generation error:', err);
    return new Response('Error generating sitemap', { status: 500 });
  }
});

function generateSitemapXml(posts: Post[]): string {
  const today = new Date().toISOString().split('T')[0];

  const postUrls = posts.map((post) => {
    const lastmod = post.updated_at 
      ? new Date(post.updated_at).toISOString().split('T')[0]
      : today;
    
    return `
  <url>
    <loc>${SITE_URL}/post/${escapeXml(post.slug)}</loc>
    <lastmod>${lastmod}</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>`;
  }).join('');

  return `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>${SITE_URL}</loc>
    <lastmod>${today}</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>${postUrls}
</urlset>`;
}

function escapeXml(str: string): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;');
}
```

**Why `SUPABASE_ANON_KEY`?**
- Respects Row Level Security (RLS) policies
- Only returns data that anonymous/public users can access
- If RLS policies change, sitemap automatically reflects those changes
- Prevents exposing URLs for draft, private, or restricted content

Deploy with `verifyJwt: false` so search engines can access it.

**Static Sitemap Index** - Create `public/sitemap.xml` that points to the Edge Function:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://YOUR_PROJECT_ID.supabase.co/functions/v1/sitemap</loc>
  </sitemap>
</sitemapindex>
```

---

### Option B: Static Sitemap (without Cloud Backend)

If the app does NOT use Cloud Backend, create a static `public/sitemap.xml` file. This must be manually updated each time a new page is added.

**Static Sitemap** - `public/sitemap.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://yourdomain.com</loc>
    <lastmod>2024-01-15</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://yourdomain.com/about</loc>
    <lastmod>2024-01-10</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
  <url>
    <loc>https://yourdomain.com/contact</loc>
    <lastmod>2024-01-10</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.7</priority>
  </url>
</urlset>
```

**When to update:** Add a new `<url>` block each time a page is published. Update the `<lastmod>` date when page content changes.

**Priority guide:**

| Page Type | Priority | Change Frequency |
|-----------|----------|------------------|
| Homepage | 1.0 | weekly |
| Main pages (About, Services) | 0.8 | monthly |
| Blog posts / Articles | 0.8 | monthly |
| Secondary pages (Contact, FAQ) | 0.7 | monthly |
| Legal pages (Privacy, Terms) | 0.5 | yearly |

---

## 4. robots.txt

Create `public/robots.txt`:

```
User-agent: *
Allow: /

Sitemap: https://yourdomain.com/sitemap.xml
```

---

## 5. Semantic HTML Structure

```html
<body>
  <header>
    <!-- Logo, navigation, site-wide header -->
    <nav>
      <a href="/">Home</a>
      <a href="/about">About</a>
    </nav>
  </header>

  <main>
    <!-- Primary content of the page (only ONE per page) -->
    
    <article>
      <!-- Self-contained content: blog post, news article, product -->
      <h1>Page Title</h1>
      <p>Content...</p>
    </article>

    <section>
      <!-- Thematic grouping of content -->
      <h2>Section Title</h2>
    </section>

    <aside>
      <!-- Sidebar, related content -->
    </aside>
  </main>

  <footer>
    <!-- Copyright, links, contact info -->
  </footer>
</body>
```

**Rules:**
- One `<main>` per page
- One `<h1>` per page (should match the page title/intent)
- Use `<article>` for standalone content that could be syndicated
- Use `<section>` for thematic groupings with headings
- Use `<nav>` for navigation blocks

---

## 6. Heading Hierarchy

```html
<h1>Main Page Title</h1>           <!-- Only ONE per page -->

  <h2>Major Section</h2>           <!-- Main sections -->
    <h3>Subsection</h3>            <!-- Within h2 -->
      <h4>Detail</h4>              <!-- Within h3 -->

  <h2>Another Major Section</h2>
    <h3>Subsection</h3>
```

**Rules:**
- Never skip levels (don't go from h1 to h3)
- h1 should match or closely relate to the `<title>` tag
- Use headings for structure, not for styling (use CSS for size)
- Screen readers and search engines use headings to understand page structure

---

## 7. Image SEO

```html
<!-- Good -->
<img 
  src="/images/product-red-shoes.webp" 
  alt="Red running shoes with white soles on wooden floor"
  loading="lazy"
  width="800"
  height="600"
/>

<!-- Bad -->
<img src="/images/IMG_1234.jpg" alt="image" />
```

**Rules:**
- **Descriptive alt text** - Describe what's in the image (not "image of..." or "photo of...")
- **Lazy loading** - Add `loading="lazy"` for below-the-fold images
- **Dimensions** - Always include `width` and `height` to prevent layout shift
- **File names** - Use descriptive names (`red-running-shoes.webp` not `IMG_1234.jpg`)
- **Format** - Prefer WebP for better compression
- **Decorative images** - Use `alt=""` (empty) for purely decorative images

---

## 8. Link Best Practices

```html
<!-- Internal links - use descriptive text -->
<a href="/pricing">View our pricing plans</a>

<!-- External links - add rel attributes -->
<a href="https://external.com" target="_blank" rel="noopener noreferrer">
  External Resource
</a>

<!-- Avoid generic link text -->
❌ <a href="/pricing">Click here</a>
❌ <a href="/pricing">Learn more</a>
✅ <a href="/pricing">See pricing details</a>
```

**Rules:**
- Use descriptive anchor text (not "click here" or "learn more")
- Add `rel="noopener noreferrer"` for external links with `target="_blank"`
- Ensure all internal pages are reachable within 3 clicks from homepage

---

## 9. URL Structure

```
Good URLs:
✅ /blog/how-to-build-landing-page
✅ /products/wireless-headphones
✅ /about

Bad URLs:
❌ /blog/post?id=12345
❌ /p/12345
❌ /blog/How_To_Build_A_Landing_Page_2024_Updated
```

**Rules:**
- Use lowercase letters
- Use hyphens (-) not underscores (_)
- Keep URLs short and descriptive
- Include primary keyword when relevant
- Avoid query parameters for content pages

---

## 10. Performance & Core Web Vitals

```html
<head>
  <!-- Preconnect to external domains you'll use -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  
  <!-- DNS prefetch for domains used later -->
  <link rel="dns-prefetch" href="https://api.yourdomain.com">
</head>
```

**Performance checklist:**
- Use `preconnect` for critical third-party origins (fonts, APIs)
- Use `dns-prefetch` for non-critical external resources
- Lazy load images below the fold
- Avoid layout shifts (always set image dimensions)
- Minimize blocking JavaScript

---

## 11. Favicon & Social Image

Place these files in the `public` folder:

**Favicon** - `public/favicon.svg`
- SVG format for modern browsers (scalable, small file size)
- Referenced in index.html: `<link rel="icon" type="image/svg+xml" href="/favicon.svg" />`

**Social Sharing Image** - `public/og-image.jpeg`
- **Dimensions:** 1200 x 630 pixels (1.91:1 ratio)
- **Format:** JPEG (keep under 200KB)
- **Content:** Site name/logo, clear readable text, brand colors
- Referenced in meta tags: `<meta property="og:image" content="/og-image.jpeg">`

---

## 12. Mobile-First Design

Google uses mobile-first indexing, meaning it primarily uses the mobile version of your site for ranking.

**Checklist:**
- Responsive design that works on all screen sizes
- Touch-friendly tap targets (minimum 44x44 pixels)
- Readable text without zooming (16px minimum body text)
- No horizontal scrolling
- Fast loading on mobile networks

---

## 13. SEO Checklist

When building an app with SEO:

- [ ] Add static meta tags in `index.html` with `data-hid` attributes
- [ ] Set up `@unhead/react` provider
- [ ] Create `useSEO` hook with site constants
- [ ] Call `useSEO()` on every page with appropriate props
- [ ] Add JSON-LD structured data for articles/products
- [ ] Create sitemap (dynamic with Cloud Backend, static without)
- [ ] Add `robots.txt` with sitemap reference
- [ ] Add `favicon.svg` to public folder
- [ ] Add `og-image.jpeg` (1200x630) to public folder
- [ ] Use semantic HTML (`<header>`, `<main>`, `<article>`, `<section>`, `<footer>`)
- [ ] Single `<h1>` per page matching the page intent
- [ ] Proper heading hierarchy (h1 -> h2 -> h3, no skipping)
- [ ] Descriptive `alt` attributes on all images
- [ ] Descriptive link text (not "click here")
- [ ] Clean URL structure with hyphens
- [ ] Mobile-responsive design
- [ ] Submit sitemap to Google Search Console
- [ ] Set up IndexNow for instant search engine notifications (if using Cloud Backend)

---

## 14. IndexNow Integration

IndexNow instantly notifies search engines (Bing, Yandex, Seznam, Naver) when content changes, enabling faster indexing than waiting for crawlers.

### Setup

1. **Generate an IndexNow key** at [indexnow.org](https://www.indexnow.org/) or use any 8-128 character alphanumeric string (e.g., UUID)

2. **Add the secret** in Settings -> Secrets:
   - Name: `INDEXNOW_KEY`
   - Value: Your generated key

3. **Create verification file** at `public/{your-key}.txt` containing just the key:
   ```
   your-key-here
   ```

4. **Deploy the Edge Function** at `supabase/functions/indexnow/index.ts`:
   - Accepts `slugs[]` or `urls[]` + `action` (publish/update/delete)
   - Submits to IndexNow API with key verification
   - Returns success/failure status

### Automatic Triggers

Integrate IndexNow calls into content management flows:

```typescript
// Helper function (add to post editor or content service)
async function notifyIndexNow(slug: string, action: 'publish' | 'update' | 'delete') {
  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
  
  if (!supabaseUrl || !supabaseAnonKey) return;

  await fetch(`${supabaseUrl}/functions/v1/indexnow`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${supabaseAnonKey}`,
    },
    body: JSON.stringify({ slugs: [slug], action }),
  });
}
```

**Call after successful database operations:**

| Action | When to Call |
|--------|--------------|
| `'publish'` | New post published |
| `'update'` | Published post content updated |
| `'delete'` | Post archived or unpublished |

### Edge Function Endpoint

```
POST /functions/v1/indexnow

Body:
{
  "slugs": ["my-post-slug"],      // OR
  "urls": ["https://site.com/post/my-post"],
  "action": "publish"             // publish | update | delete
}

Response:
{
  "success": true,
  "message": "Successfully submitted 1 URL(s) to IndexNow",
  "urls": ["https://site.com/post/my-post"]
}
```

### Notes

- Notifications run silently in background (non-blocking)
- Failures are logged but don't interrupt user flow
- IndexNow shares submissions across participating search engines
- Rate limit: No strict limit, but batch URLs when possible (up to 10,000 per request)

---

## Key Points

1. **Always use `key` attributes** in useHead meta tags for proper deduplication
2. **Static fallbacks in index.html** ensure crawlers see content before JS executes
3. **`data-hid` attributes** link static tags to dynamic replacements
4. **Use `SUPABASE_ANON_KEY`** for sitemaps to respect RLS and only expose public content
5. **Canonical URLs** prevent duplicate content issues
6. **noIndex** for pages that shouldn't be indexed (404, admin pages)
7. **One h1 per page** - should match the page's main topic
8. **Semantic HTML** helps search engines understand your content structure
