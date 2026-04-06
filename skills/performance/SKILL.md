---
title: Performance
version: "1.0"
description: |
  Performance and loading optimization for React + Vite + Tailwind CSS
  apps built with sticklight.com. Use when optimizing page load speed,
  reducing Core Web Vitals issues, improving First Contentful Paint (FCP),
  Largest Contentful Paint (LCP), or Cumulative Layout Shift (CLS).
  Covers: font loading strategy (preconnect, preload, display=swap),
  third-party script deferral (analytics, GTM), image optimization
  (dimensions, loading priority, fetchPriority, WebP), CSS performance
  (will-change, GPU-accelerated animations, transform/opacity), event
  listener optimization (passive scroll, cleanup), animation best
  practices (CSS-only, prefers-reduced-motion), loading sequence
  priority, code splitting with React.lazy, Vite bundle optimization,
  and Tailwind CSS purge.
---

# Performance & Loading Optimization Skill for Sticklight Apps

This skill helps you optimize web application loading performance, reduce Core Web Vitals issues, and ensure fast page loads in React + Vite + Tailwind CSS apps.

## Your Task

When this skill is active, follow these steps:

1. **Audit current performance** — Check the project for missing image dimensions, render-blocking scripts, unoptimized fonts, and layout shift issues
2. **Optimize font loading** — Add `preconnect`, `preload`, and `display=swap` for all web fonts
3. **Defer third-party scripts** — Move analytics, GTM, and tracking scripts to load after the page is ready
4. **Fix image loading** — Add `width`/`height` to all images, set correct `loading` and `fetchPriority` attributes based on position
5. **Optimize animations** — Use `transform`/`opacity` instead of layout-triggering properties, add `will-change` hints, respect `prefers-reduced-motion`
6. **Fix event listeners** — Make scroll listeners passive, ensure cleanup on unmount
7. **Split code** — Use `React.lazy()` and `<Suspense>` for route-level code splitting
8. **Optimize the bundle** — Configure Vite chunk splitting, audit bundle size, ensure Tailwind purges unused CSS
9. **Set the loading sequence** — Structure `index.html` to load resources in optimal order
10. **Measure** — Test with Lighthouse, PageSpeed Insights, or Chrome DevTools Performance tab
11. **Verify** — Run through the Performance Checklist (section 13) before shipping

## Core Web Vitals Targets

| Metric | Target | What It Measures |
|--------|--------|------------------|
| LCP | < 2.5s | Largest Contentful Paint — when the biggest visible element finishes rendering |
| FID | < 100ms | First Input Delay — time from first interaction to browser response |
| CLS | < 0.1 | Cumulative Layout Shift — how much the page layout jumps during load |
| FCP | < 1.8s | First Contentful Paint — when the first text or image appears |
| TBT | < 200ms | Total Blocking Time — total time the main thread is blocked |

---

## 1. Font Loading Strategy

### Preconnect to Font Servers

Establish early connections to font servers before the browser discovers them naturally. Place these at the top of `<head>` in `index.html`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
```

### Preload Font Stylesheets

Load font CSS before the browser reaches the stylesheet link:

```html
<link rel="preload" as="style" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" />
```

### Font Display Strategy

Always use `display=swap` to prevent invisible text during font loading (FOIT). The browser shows fallback text immediately, then swaps in the web font when ready.

### Self-Hosting Fonts (Best Performance)

For maximum control, download fonts and serve them from your own domain. This eliminates the DNS lookup and connection to Google Fonts entirely:

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/Inter-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}
```

Place font files in `public/fonts/` and preload the most critical weight:

```html
<link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
```

---

## 2. Third-Party Scripts (Analytics, GTM)

### Defer Non-Critical Scripts

Load analytics and tracking scripts AFTER the page is fully loaded. Never let them block rendering:

```html
<script>
  window.addEventListener('load', function() {
    setTimeout(function() {
      // Google Tag Manager
      (function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
      new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
      j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
      'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
      })(window,document,'script','dataLayer','GTM-XXXXXXX');
    }, 100);
  });
</script>
```

### Why This Matters

- Faster First Contentful Paint (FCP) — the browser renders content before loading tracking code
- Lower Total Blocking Time (TBT) — the main thread is free for user interactions
- Better experience on slow connections — users see the page first, analytics load silently

### Script Loading Attributes

| Attribute | Behavior | Use For |
|-----------|----------|---------|
| (none) | Blocks parsing until executed | Critical inline scripts only |
| `async` | Downloads in parallel, executes immediately when ready | Independent scripts (analytics) |
| `defer` | Downloads in parallel, executes after HTML is parsed | Scripts that need the DOM |
| Post-load injection | Loads after everything else | Non-critical tracking, chat widgets |

---

## 3. Image Optimization

### Always Include Dimensions

Prevent Cumulative Layout Shift (CLS) by specifying `width` and `height` on every image. The browser reserves space before the image loads:

```tsx
<img
  src={image}
  alt="Product photo showing red sneakers"
  width={800}
  height={600}
/>
```

### Loading Priority Attributes

| Image Location | Attributes | Why |
|----------------|------------|-----|
| Logo, hero images | `loading="eager" fetchPriority="high"` | Visible immediately, critical for LCP |
| Above the fold | `loading="eager"` | Visible on initial viewport |
| Below the fold | `loading="lazy"` | Not visible until user scrolls |
| Decorative/background | `loading="lazy"` | Not essential content |

### Implementation Examples

```tsx
// Critical — Logo (always visible, affects LCP)
<img
  src={logo}
  alt="Sticklight logo"
  width={152}
  height={28}
  loading="eager"
  fetchPriority="high"
/>

// Hero image (above fold, largest paint candidate)
<img
  src={heroImage}
  alt="App builder dashboard showing a website preview"
  width={1200}
  height={630}
  loading="eager"
  fetchPriority="high"
/>

// Below fold — Gallery item
<img
  src={galleryImage}
  alt="Template preview for e-commerce store"
  width={400}
  height={300}
  loading="lazy"
/>
```

### Image Format

- Prefer **WebP** for photos (30-50% smaller than JPEG at same quality)
- Use **SVG** for icons and logos (scalable, tiny file size)
- Use **AVIF** where supported (even smaller than WebP, but slower to encode)
- Always compress images before adding to `public/`

---

## 4. CSS Performance

### Will-Change Property

Hint the browser about upcoming animations to optimize rendering. Use sparingly — overuse wastes GPU memory:

```tsx
// For animated elements (marquee, sliders, carousels)
style={{ willChange: 'transform' }}

// For dynamic content (canvas, live updates)
style={{ willChange: 'contents' }}
```

Remove `will-change` when the animation is done if it's a one-time transition:

```tsx
onTransitionEnd={() => ref.current.style.willChange = 'auto'}
```

### GPU-Accelerated Animations

Use `transform` and `opacity` for animations. These run on the GPU compositor thread and don't trigger layout recalculations:

```css
/* Good — GPU accelerated, no layout triggered */
.slide-in {
  transform: translateX(-100px);
  opacity: 0.5;
  transition: transform 0.3s ease, opacity 0.3s ease;
}

/* Bad — triggers layout recalculation on every frame */
.slide-in-bad {
  left: -100px;
  margin-left: -100px;
  width: 200px;
}
```

**Properties that trigger layout (avoid animating these):**

| Triggers Layout | Use Instead |
|-----------------|-------------|
| `top`, `left`, `right`, `bottom` | `transform: translate()` |
| `width`, `height` | `transform: scale()` |
| `margin`, `padding` | `transform: translate()` |
| `font-size` | `transform: scale()` |

### Tailwind CSS Production Purge

Tailwind purges unused CSS by default in production builds. Ensure your `tailwind.config.js` content paths cover all files:

```js
export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx}',
  ],
  // ...
};
```

If your production CSS is larger than ~30KB, audit for unused classes.

---

## 5. Event Listener Optimization

### Passive Scroll Listeners

Never block scroll events. Passive listeners tell the browser the handler will not call `preventDefault()`, so scrolling can proceed without waiting:

```tsx
// Correct — passive, non-blocking
window.addEventListener('scroll', handleScroll, { passive: true });

// Wrong — blocks scrolling until handler completes
window.addEventListener('scroll', handleScroll);
```

### Cleanup on Unmount

Always remove event listeners to prevent memory leaks:

```tsx
useEffect(() => {
  const handleScroll = () => {
    // scroll logic
  };
  window.addEventListener('scroll', handleScroll, { passive: true });

  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### Debounce Expensive Handlers

For resize and scroll handlers that trigger expensive calculations, debounce them:

```tsx
import { useCallback, useEffect, useRef } from 'react';

function useDebounce(fn: () => void, delay: number) {
  const timeout = useRef<ReturnType<typeof setTimeout>>();

  return useCallback(() => {
    clearTimeout(timeout.current);
    timeout.current = setTimeout(fn, delay);
  }, [fn, delay]);
}

// Usage
const handleResize = useDebounce(() => {
  // expensive layout calculation
}, 150);

useEffect(() => {
  window.addEventListener('resize', handleResize, { passive: true });
  return () => window.removeEventListener('resize', handleResize);
}, [handleResize]);
```

---

## 6. Animation Best Practices

### CSS-Only Animations

Prefer CSS animations over JavaScript for simple movements. CSS animations run on the compositor thread and are more efficient:

```css
.marquee-content {
  animation: marquee-scroll 20s linear infinite;
}

@keyframes marquee-scroll {
  from { transform: translateX(0); }
  to { transform: translateX(-33.33%); }
}
```

### Reduce Motion Preference

Respect the user's motion preferences. Some users experience motion sickness or have vestibular disorders:

```css
@media (prefers-reduced-motion: reduce) {
  .marquee-content {
    animation: none;
  }

  * {
    transition-duration: 0.01ms !important;
    animation-duration: 0.01ms !important;
  }
}
```

In React, check programmatically:

```tsx
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
```

---

## 7. Code Splitting

### Route-Level Splitting with React.lazy

Split each route into its own chunk so the browser only downloads the code needed for the current page:

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const BlogPost = lazy(() => import('./pages/BlogPost'));

function App() {
  return (
    <Suspense fallback={<LoadingSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/post/:slug" element={<BlogPost />} />
      </Routes>
    </Suspense>
  );
}
```

### Vite Chunk Configuration

Configure manual chunks in `vite.config.ts` to separate vendor code from your app code:

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
          supabase: ['@supabase/supabase-js'],
        },
      },
    },
  },
});
```

### Audit Bundle Size

Use `npx vite-bundle-visualizer` to see what's in your bundles and identify large dependencies.

---

## 8. Loading Sequence in index.html

Structure your `<head>` to load resources in optimal order. Critical resources first, non-critical last:

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- 1. Preconnect to external domains (establish connections early) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link rel="dns-prefetch" href="https://your-project.supabase.co" />

  <!-- 2. Preload critical assets (fonts, hero image) -->
  <link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
  <link rel="preload" as="style" href="https://fonts.googleapis.com/css2?family=Inter&display=swap" />

  <!-- 3. Critical CSS / font stylesheet -->
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter&display=swap" />

  <!-- 4. Favicon -->
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />

  <!-- 5. Meta tags (SEO, OG, theme) -->
  <title>Your App</title>
  <meta name="description" content="Your description" />
  <meta name="theme-color" content="#YOUR_BRAND_COLOR" />
  <link rel="canonical" href="https://yourdomain.com/" />
</head>
<body>
  <div id="root"></div>

  <!-- 6. Main app bundle (Vite injects this) -->
  <script type="module" src="/src/main.tsx"></script>

  <!-- 7. Deferred analytics (load after everything else) -->
  <script>
    window.addEventListener('load', function() {
      setTimeout(function() {
        // GTM, analytics, or tracking code
      }, 100);
    });
  </script>
</body>
```

**Loading order summary:**

| Priority | What | Why |
|----------|------|-----|
| 1st | Preconnect / DNS prefetch | Establish connections before they're needed |
| 2nd | Preload critical assets | Start downloading fonts and hero images early |
| 3rd | CSS / font stylesheets | Render text correctly from the start |
| 4th | Meta tags | SEO and browser hints |
| 5th | App bundle | The actual application |
| 6th | Eager images | Above-the-fold content |
| 7th | Lazy images | Below-the-fold, loaded on scroll |
| Last | Analytics / tracking | Non-critical, loaded after page is ready |

---

## 9. React-Specific Optimizations

### Avoid Unnecessary Re-Renders

Use `React.memo` for components that receive stable props:

```tsx
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
});
```

Use `useMemo` for expensive calculations:

```tsx
const sortedItems = useMemo(
  () => items.sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

Use `useCallback` for functions passed as props:

```tsx
const handleClick = useCallback((id: string) => {
  setSelected(id);
}, []);
```

### Virtualize Long Lists

For lists with 100+ items, use a virtualization library to only render visible items:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 10. Supabase Query Optimization

### Select Only Needed Columns

Don't use `select('*')` when you only need a few fields:

```tsx
// Bad — fetches all columns
const { data } = await supabase.from('posts').select('*');

// Good — fetches only what's needed
const { data } = await supabase.from('posts').select('id, title, slug, published_at');
```

### Paginate Large Datasets

Never load all rows at once. Use `range()` for pagination:

```tsx
const PAGE_SIZE = 20;

const { data, count } = await supabase
  .from('posts')
  .select('id, title, slug', { count: 'exact' })
  .order('published_at', { ascending: false })
  .range(page * PAGE_SIZE, (page + 1) * PAGE_SIZE - 1);
```

### Add Database Indexes

For columns you filter or sort by frequently, add indexes:

```sql
CREATE INDEX idx_posts_published_at ON posts (published_at DESC);
CREATE INDEX idx_posts_user_id ON posts (user_id);
CREATE INDEX idx_posts_slug ON posts (slug);
```

---

## 11. Measuring Performance

### Lighthouse

Run Lighthouse in Chrome DevTools (Lighthouse tab) or via CLI:

```bash
npx lighthouse https://yourdomain.com --view
```

### PageSpeed Insights

Test live URLs at [pagespeed.web.dev](https://pagespeed.web.dev/). Shows both lab data (simulated) and field data (real user metrics from Chrome UX Report).

### Chrome DevTools Performance Tab

1. Open DevTools -> Performance tab
2. Click Record, reload the page, stop recording
3. Look for long tasks (red bars), layout shifts, and rendering bottlenecks

### Web Vitals in Code

Add real user monitoring with the `web-vitals` library:

```tsx
import { onLCP, onFID, onCLS } from 'web-vitals';

onLCP(console.log);
onFID(console.log);
onCLS(console.log);
```

---

## 12. robots.txt and Sitemap

### robots.txt

Create `public/robots.txt`:

```
User-agent: *
Allow: /

Sitemap: https://yourdomain.com/sitemap.xml

# Block AI crawlers (optional)
User-agent: GPTBot
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: anthropic-ai
Disallow: /

User-agent: Google-Extended
Disallow: /
```

### sitemap.xml

Create `public/sitemap.xml` for static sites, or use a dynamic sitemap via Edge Function (see the SEO skill for full implementation):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://yourdomain.com/</loc>
    <lastmod>2025-01-01</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

---

## Process

When optimizing a project's performance, follow this workflow:

1. Run Lighthouse or PageSpeed Insights to get a baseline score
2. Check `index.html` for missing preconnect, preload, and font loading setup
3. Check all `<img>` tags for missing `width`, `height`, `loading`, and `alt` attributes
4. Check for render-blocking third-party scripts — defer analytics and tracking
5. Check CSS animations — replace layout-triggering properties with `transform`/`opacity`
6. Check event listeners — ensure scroll/resize handlers are passive and cleaned up on unmount
7. Check for code splitting — verify routes use `React.lazy()` and `<Suspense>`
8. Check Vite config for manual chunk splitting of vendor dependencies
9. Check Supabase queries — use column selection, pagination, and indexes
10. Run Lighthouse again to measure improvement
11. Run through the Performance Checklist (section 13)

---

## Output Format

When optimizing a page or component, include:

1. **Fixed `<img>` tags** — with `width`, `height`, `loading`, `fetchPriority`, and descriptive `alt`
2. **Optimized event listeners** — passive where applicable, cleanup in `useEffect` return
3. **CSS animation fixes** — `transform`/`opacity` instead of layout properties, `will-change` where needed
4. **`prefers-reduced-motion`** — media query or JS check for animations

When optimizing project-level performance, include:

1. **`index.html` `<head>`** — preconnect, preload, deferred scripts in correct order
2. **`vite.config.ts`** — manual chunks for vendor and supabase
3. **Route-level code splitting** — `React.lazy()` and `<Suspense>` wrapping routes
4. **`public/robots.txt`** — with sitemap reference
5. **Lighthouse before/after scores** — to verify improvements

When modifying existing code, preserve:

1. Existing `useEffect` cleanup patterns
2. Existing `useCallback` / `useMemo` usage
3. Existing Tailwind class names (don't remove classes that may be used conditionally)

---

## 13. Performance Checklist

- [ ] All images have `width` and `height` attributes
- [ ] Hero/logo images use `loading="eager"` and `fetchPriority="high"`
- [ ] Below-fold images use `loading="lazy"`
- [ ] All images have descriptive `alt` text
- [ ] Fonts use `preconnect` and `preload`
- [ ] Fonts use `display=swap`
- [ ] Analytics/tracking scripts are deferred (load after page ready)
- [ ] No render-blocking third-party scripts in `<head>`
- [ ] Scroll/resize listeners are `{ passive: true }`
- [ ] All event listeners are cleaned up on unmount
- [ ] Animations use `transform`/`opacity` only (no `top`/`left`/`width`/`height`)
- [ ] `will-change` hints on animated elements
- [ ] `prefers-reduced-motion` is respected
- [ ] Routes use `React.lazy()` and `<Suspense>`
- [ ] Vite config splits vendor chunks
- [ ] Supabase queries select only needed columns
- [ ] Large lists are paginated or virtualized
- [ ] `robots.txt` exists and is valid
- [ ] `sitemap.xml` exists and is valid
- [ ] `<link rel="canonical">` is set
- [ ] `<meta name="theme-color">` is set
- [ ] Lighthouse performance score is 90+
