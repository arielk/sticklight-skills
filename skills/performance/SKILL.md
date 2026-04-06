---
title: Performance
version: "1.0"
description: |
  Performance and loading optimization for React + TypeScript + Vite +
  Tailwind CSS apps built with sticklight.com. Use when optimizing page
  load speed, reducing Core Web Vitals issues, improving First Contentful
  Paint (FCP), Largest Contentful Paint (LCP), Cumulative Layout Shift
  (CLS), or Total Blocking Time (TBT). Also use when diagnosing slow
  renders, large bundles, memory leaks, or poor Lighthouse scores.
  Covers: critical rendering path (loading shell, resource hints,
  above-the-fold optimization), font loading strategy (preconnect, preload,
  self-hosting, display=swap, subsetting, variable fonts), third-party script deferral (analytics, GTM, chat
  widgets), image optimization (dimensions, loading priority, fetchPriority,
  WebP/AVIF, responsive images with srcset), CSS performance (will-change,
  GPU-accelerated animations, transform/opacity, Tailwind CSS purge),
  event listener optimization (passive scroll, debounce, cleanup),
  animation best practices (CSS-only, prefers-reduced-motion, requestAnimationFrame),
  code splitting with React.lazy and dynamic imports, skeleton screens
  for perceived performance, Vite bundle
  optimization (manual chunks, tree shaking, compression), React render
  performance (React.memo, useMemo, useCallback, virtualization, context
  splitting), network optimization (resource hints, prefetch, dns-prefetch,
  HTTP caching headers, service workers), data fetching performance
  (Supabase query optimization, pagination, stale-while-revalidate,
  optimistic updates), memory management (subscription cleanup, AbortController,
  large object disposal), loading sequence priority in index.html,
  performance budgets, and measuring with Lighthouse and web-vitals.
---

# Performance & Loading Optimization Skill for Sticklight Apps

This skill helps you optimize web application loading performance, reduce Core Web Vitals issues, and ensure fast page loads in React + Vite + Tailwind CSS apps.

## Your Task

When this skill is active, follow these steps:

1. **Audit current performance** — Run Lighthouse or PageSpeed Insights to get a baseline score; check the project for missing image dimensions, render-blocking scripts, unoptimized fonts, and layout shift issues
2. **Optimize the critical rendering path** — Add a lightweight loading shell to `index.html`, configure resource hints (`preconnect`, `dns-prefetch`, `preload`), and structure `<head>` for optimal loading order
3. **Optimize font loading** — Add `preconnect`, `preload`, and `display=swap` for all web fonts; consider self-hosting for maximum control
4. **Defer third-party scripts** — Move analytics, GTM, chat widgets, and tracking scripts to load after the page is ready
5. **Fix image loading** — Add `width`/`height` to all images, set correct `loading` and `fetchPriority` attributes based on viewport position, use responsive images with `srcset` where appropriate
6. **Optimize CSS and animations** — Use `transform`/`opacity` instead of layout-triggering properties, add `will-change` hints sparingly, respect `prefers-reduced-motion`
7. **Fix event listeners and memory** — Make scroll/resize listeners passive, ensure cleanup on unmount, use `AbortController` for fetch requests, dispose large objects
8. **Optimize React renders** — Use `React.memo`, `useMemo`, `useCallback` where appropriate; split contexts; virtualize long lists
9. **Split code and optimize the bundle** — Use `React.lazy()` and `<Suspense>` for route-level code splitting; configure Vite manual chunks, tree shaking, and compression
10. **Optimize data fetching** — Select only needed columns in Supabase queries, paginate large datasets, add database indexes, use stale-while-revalidate patterns
11. **Set performance budgets** — Define bundle size limits, enforce them in CI, and monitor with real user metrics
12. **Measure improvement** — Run Lighthouse again, compare before/after, verify Core Web Vitals targets are met
13. **Verify** — Run through the Performance Checklist (section 15) before shipping

## Core Web Vitals Targets

| Metric | Target | What It Measures |
|--------|--------|------------------|
| LCP | < 2.5s | Largest Contentful Paint — when the biggest visible element finishes rendering |
| INP | < 200ms | Interaction to Next Paint — responsiveness to user input (replaces FID) |
| CLS | < 0.1 | Cumulative Layout Shift — how much the page layout jumps during load |
| FCP | < 1.8s | First Contentful Paint — when the first text or image appears |
| TBT | < 200ms | Total Blocking Time — total time the main thread is blocked |
| TTFB | < 800ms | Time to First Byte — server response time for the initial document |

---

## 1. Critical Rendering Path

The critical rendering path is the sequence of steps the browser takes to convert HTML, CSS, and JavaScript into pixels on screen. Minimize the work the browser must do before the first paint.

### Loading Shell (SPA-Specific)

In a React SPA, the browser shows a blank page until JavaScript downloads, parses, and renders the first component. Unlike server-rendered pages, there is no HTML content to style with "inline critical CSS" — the initial document is just an empty `<div id="root"></div>`.

Instead, add a lightweight loading shell directly in `index.html` so users see something instantly while the JS bundle loads:

```html
<body>
  <div id="root">
    <!-- Loading shell — visible until React mounts and replaces this -->
    <div style="display:flex;align-items:center;justify-content:center;min-height:100vh;font-family:system-ui,sans-serif;background:#fff">
      <div style="text-align:center">
        <div style="width:40px;height:40px;border:3px solid #e5e7eb;border-top-color:#3b82f6;border-radius:50%;animation:spin 0.6s linear infinite;margin:0 auto"></div>
      </div>
    </div>
  </div>
  <style>@keyframes spin{to{transform:rotate(360deg)}}</style>
  <script type="module" src="/src/main.tsx"></script>
</body>
```

React's `createRoot(document.getElementById('root')!).render(...)` automatically replaces the loading shell content when the app mounts — no cleanup needed.

**Why not inline critical CSS in an SPA?** Traditional critical CSS extraction (inlining above-the-fold styles and async-loading the rest) is designed for server-rendered pages that have real HTML content before JS runs. In a Vite + React SPA, Vite bundles and injects all CSS automatically alongside the JS. The CSS and JS arrive together — there's no separate stylesheet to "defer." Focus on keeping the total CSS bundle small through Tailwind purge (section 5) instead.

### Resource Hints

Place resource hints at the top of `<head>`, before any other tags:

| Hint | Purpose | When to Use |
|------|---------|-------------|
| `preconnect` | Establish connection (DNS + TCP + TLS) early | Domains you will fetch from in the first few seconds |
| `dns-prefetch` | Resolve DNS only | Domains used later or as a fallback for `preconnect` |
| `preload` | Download a specific resource immediately | Critical fonts, hero images, above-fold CSS |
| `prefetch` | Download a resource for the next navigation | Assets for likely next pages |
| `modulepreload` | Preload ES module scripts | Critical JS modules for the current page |

```html
<head>
  <!-- Preconnect: domains used immediately -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  
  <!-- DNS prefetch: domains used later -->
  <link rel="dns-prefetch" href="https://your-project.supabase.co" />
  <link rel="dns-prefetch" href="https://www.googletagmanager.com" />
  
  <!-- Preload: critical assets -->
  <link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
  <link rel="preload" as="image" href="/hero-image.webp" />
</head>
```

### Above-the-Fold Priority

The browser should spend its first moments downloading and rendering what the user sees immediately. Structure your HTML so critical content appears first in the DOM:

```tsx
function LandingPage() {
  return (
    <>
      {/* Above the fold — rendered immediately */}
      <nav>...</nav>
      <Hero />
      
      {/* Below the fold — can be deferred */}
      <Suspense fallback={<SectionSkeleton />}>
        <Features />
        <Testimonials />
        <Footer />
      </Suspense>
    </>
  );
}
```

---

## 2. Font Loading Strategy

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

### Font Subsetting

Load only the character ranges you need. A full font file with all Unicode blocks can be 200KB+, but subsetting to just Latin and specific scripts can save up to 90%:

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/Inter-Regular-Latin.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-007F, U+0080-00FF; /* Basic Latin + Latin Extended */
}
```

Google Fonts does this automatically — when you request a font, it serves subset files per script. If self-hosting, use tools like `pyftsubset` (from `fonttools`) or [glyphanger](https://github.com/zachleat/glyphanger) to strip unused characters.

### Variable Fonts

Use a variable font instead of loading separate files for each weight. One file covers the full weight range (and optionally width, italic, etc.):

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/Inter-Variable.woff2') format('woff2-variations');
  font-weight: 100 900;
  font-style: normal;
  font-display: swap;
}
```

| Approach | Files | Typical Size |
|----------|-------|-------------|
| 4 static weights (400, 500, 600, 700) | 4 files | ~100KB total |
| 1 variable font (100-900) | 1 file | ~50-70KB total |

Variable fonts reduce HTTP requests and often have a smaller total size than multiple static files. Most modern fonts (Inter, Roboto, Open Sans) offer variable versions.

### Font Loading Budget

Limit web fonts to reduce download size and prevent layout shift:

| Guideline | Recommendation |
|-----------|----------------|
| Number of families | 1-2 maximum |
| Number of weights | Variable font preferred, or 3-4 static weights max |
| Formats | woff2 only (best compression, widest support) |
| Subsetting | Only needed character ranges (Latin, Hebrew, etc.) |
| Total font size | < 100KB for all font files |
| Fallback stack | Always include system fonts: `Inter, system-ui, -apple-system, sans-serif` |

---

## 3. Third-Party Scripts (Analytics, GTM)

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

### Third-Party Script Audit

Common offenders and how to handle them:

| Script | Impact | Solution |
|--------|--------|----------|
| Google Tag Manager | 50-100KB, blocks main thread | Post-load injection (see above) |
| Chat widgets (Intercom, Drift) | 200-500KB, heavy JS | Load after user interaction or after 5s delay |
| Social embeds (Twitter, Instagram) | 100-300KB each | Use static screenshots with links instead, or lazy load |
| Google Maps | 200KB+, many sub-requests | Load on user interaction (click to load map) |
| reCAPTCHA | 150KB+ | Load only on pages that need it, use v3 for invisible verification |

```tsx
function LazyMap() {
  const [showMap, setShowMap] = useState(false);

  if (!showMap) {
    return (
      <button onClick={() => setShowMap(true)}>
        Show Map
      </button>
    );
  }

  return <GoogleMapEmbed />;
}
```

---

## 4. Image Optimization

### Always Include Dimensions

Prevent Cumulative Layout Shift (CLS) by specifying `width` and `height` on every image. The browser uses these to calculate the aspect ratio and reserve space before the image loads:

```tsx
<img
  src={image}
  alt="Product photo showing red sneakers"
  width={800}
  height={600}
  className="w-full h-auto"
/>
```

The `width` and `height` attributes tell the browser the aspect ratio for space reservation. Adding `className="w-full h-auto"` (Tailwind) makes the image scale responsively within its container instead of rendering at the fixed pixel dimensions. Without this, the image renders at exactly 800x600px regardless of container size.

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
  className="w-full h-auto"
  loading="eager"
  fetchPriority="high"
/>

// Below fold — Gallery item
<img
  src={galleryImage}
  alt="Template preview for e-commerce store"
  width={400}
  height={300}
  className="w-full h-auto"
  loading="lazy"
/>
```

### Responsive Images with srcset

Serve different image sizes based on the viewport to avoid downloading oversized images on mobile:

```tsx
<img
  srcSet="/images/hero-400.webp 400w, /images/hero-800.webp 800w, /images/hero-1200.webp 1200w"
  sizes="(max-width: 640px) 400px, (max-width: 1024px) 800px, 1200px"
  src="/images/hero-800.webp"
  alt="App builder dashboard"
  width={1200}
  height={630}
  loading="eager"
  fetchPriority="high"
/>
```

### Image Format

| Format | Use For | Compression |
|--------|---------|-------------|
| **WebP** | Photos, complex images | 30-50% smaller than JPEG at same quality |
| **AVIF** | Photos (where supported) | 20-30% smaller than WebP, slower to encode |
| **SVG** | Icons, logos, illustrations | Scalable, tiny file size, no quality loss |
| **PNG** | Screenshots, images with text | Lossless, larger than WebP |

Always compress images before adding to `public/`. Use tools like `sharp`, `squoosh`, or online services. Target file sizes:

| Image Type | Target Size |
|------------|-------------|
| Hero / full-width | < 150KB |
| Thumbnail / card | < 50KB |
| Icon / logo | < 10KB |

---

## 5. CSS Performance

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
| `border-width` | `box-shadow` or `outline` |

### Tailwind CSS Production Purge

Tailwind purges unused CSS by default in production builds. Ensure your `tailwind.config.js` content paths cover all files:

```js
export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx}',
  ],
};
```

If your production CSS is larger than ~30KB, audit for unused classes. Common issues:

- Dynamic class names that Tailwind can't detect (use complete class names, not string concatenation)
- Missing content paths (e.g., component libraries in `node_modules`)
- Safelist entries that are no longer needed

```js
// Bad — Tailwind cannot detect this
const color = isActive ? 'blue' : 'gray';
<div className={`bg-${color}-500`} />

// Good — Tailwind can detect complete class names
<div className={isActive ? 'bg-blue-500' : 'bg-gray-500'} />
```

### Contain Property

Use CSS containment to limit the scope of browser layout, style, and paint calculations:

```css
/* Cards in a grid — each card's layout is independent */
.card {
  contain: layout style;
}

/* Offscreen content — browser can skip rendering entirely */
.offscreen-section {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px;
}
```

`content-visibility: auto` is especially powerful for long pages — the browser skips rendering off-screen sections entirely and uses the intrinsic size for scroll calculations.

---

## 6. Event Listener Optimization

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

const handleResize = useDebounce(() => {
  // expensive layout calculation
}, 150);

useEffect(() => {
  window.addEventListener('resize', handleResize, { passive: true });
  return () => window.removeEventListener('resize', handleResize);
}, [handleResize]);
```

### Throttle vs Debounce

| Pattern | Behavior | Use For |
|---------|----------|---------|
| **Debounce** | Waits until activity stops, then fires once | Search input, resize calculations, form validation |
| **Throttle** | Fires at most once per interval | Scroll position tracking, infinite scroll, progress bars |

```tsx
function useThrottle(fn: () => void, interval: number) {
  const lastRun = useRef(0);
  const timeout = useRef<ReturnType<typeof setTimeout>>();

  return useCallback(() => {
    const now = Date.now();
    const remaining = interval - (now - lastRun.current);

    if (remaining <= 0) {
      lastRun.current = now;
      fn();
    } else {
      clearTimeout(timeout.current);
      timeout.current = setTimeout(() => {
        lastRun.current = Date.now();
        fn();
      }, remaining);
    }
  }, [fn, interval]);
}
```

---

## 7. Animation Best Practices

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

### Use a Custom Hook for Motion Preference

```tsx
import { useState, useEffect } from 'react';

function usePrefersReducedMotion() {
  const [prefersReduced, setPrefersReduced] = useState(
    () => window.matchMedia('(prefers-reduced-motion: reduce)').matches
  );

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}
```

### requestAnimationFrame for Complex Animations

When JavaScript-driven animation is necessary, always use `requestAnimationFrame` instead of `setInterval` or `setTimeout`:

```tsx
useEffect(() => {
  let frameId: number;

  function animate() {
    // animation logic — runs at display refresh rate
    frameId = requestAnimationFrame(animate);
  }

  frameId = requestAnimationFrame(animate);
  return () => cancelAnimationFrame(frameId);
}, []);
```

---

## 8. Code Splitting

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

### Component-Level Splitting

Split components that are **hidden behind user interaction** — modals, dialogs, expandable panels, and conditionally rendered heavy widgets. This reduces the initial JS bundle size, which directly improves TBT and LCP.

Only lazy-load components that are **not rendered on initial page load**. If a component always renders when the page opens, lazy-loading it just adds a network waterfall and a flash of loading state without reducing the work needed for first paint.

```tsx
const ChartDashboard = lazy(() => import('./components/ChartDashboard'));
const ShareDialog = lazy(() => import('./components/ShareDialog'));

function PostEditor({ showChart }: { showChart: boolean }) {
  const [showShare, setShowShare] = useState(false);

  return (
    <>
      {/* Always rendered — do NOT lazy-load, it just adds a waterfall */}
      <RichTextEditor />

      {/* Conditional — lazy-load, code is only fetched if user toggles this on */}
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <ChartDashboard />
        </Suspense>
      )}

      {/* Interaction-triggered — lazy-load, code is fetched on button click */}
      <button onClick={() => setShowShare(true)}>Share</button>
      {showShare && (
        <Suspense fallback={null}>
          <ShareDialog onClose={() => setShowShare(false)} />
        </Suspense>
      )}
    </>
  );
}
```

| Scenario | Lazy-load? | Why |
|----------|-----------|-----|
| Always rendered on page load | No | Adds waterfall and loading flash without reducing initial work |
| Behind a toggle, tab, or accordion | Yes | Code only loads if the user opens it |
| Modal or dialog (opens on click) | Yes | Code only loads on interaction |
| Below-fold section rarely scrolled to | Maybe | Only if the component is heavy (100KB+) and truly off-screen |

### Preload on Hover/Focus

Preload route chunks when the user hovers or focuses on a navigation link, so the chunk is ready by the time they click:

```tsx
const settingsImport = () => import('./pages/Settings');
const Settings = lazy(settingsImport);

<Link
  to="/settings"
  onMouseEnter={() => settingsImport()}
  onFocus={() => settingsImport()}
>
  Settings
</Link>
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

### Build Compression

Enable gzip and Brotli compression for production builds:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import compression from 'vite-plugin-compression';

export default defineConfig({
  plugins: [
    react(),
    compression({ algorithm: 'gzip' }),
    compression({ algorithm: 'brotliCompress' }),
  ],
});
```

### Audit Bundle Size

Use `npx vite-bundle-visualizer` to see what's in your bundles and identify large dependencies. Common issues:

| Problem | Solution |
|---------|----------|
| Full lodash imported | Use `lodash-es` or individual imports: `import debounce from 'lodash-es/debounce'` |
| Moment.js (300KB+) | Replace with `date-fns` or `dayjs` (2-7KB) |
| Full icon library imported | Import individual icons: `import { IconMenu } from '@tabler/icons-react'` |
| Unused dependencies | Remove from `package.json`, verify with `npx depcheck` |

### Skeleton Screens (Perceived Performance)

Use skeleton screens instead of spinners for `<Suspense>` fallbacks and data loading states. Skeletons mimic the shape of the incoming content, making the app feel faster because users perceive progress rather than waiting:

```tsx
function PageSkeleton() {
  return (
    <div className="animate-pulse space-y-4 p-6">
      {/* Nav placeholder */}
      <div className="h-8 w-48 rounded bg-gray-200" />
      {/* Hero placeholder */}
      <div className="h-64 w-full rounded-lg bg-gray-200" />
      {/* Content lines */}
      <div className="space-y-2">
        <div className="h-4 w-full rounded bg-gray-200" />
        <div className="h-4 w-5/6 rounded bg-gray-200" />
        <div className="h-4 w-4/6 rounded bg-gray-200" />
      </div>
    </div>
  );
}

// Use as Suspense fallback
<Suspense fallback={<PageSkeleton />}>
  <Dashboard />
</Suspense>
```

| Approach | Perceived Speed | CLS Risk | Use When |
|----------|----------------|----------|----------|
| Spinner | Slow — user sees no content | Low (fixed size) | Short, unpredictable waits (< 300ms) |
| Skeleton | Fast — user sees content shape | Low if dimensions match | Page loads, data fetching, route transitions |
| Blank / `null` | Worst — nothing visible | None | Never (avoid for any user-facing state) |

Match skeleton dimensions to the real content to avoid layout shift when the actual content replaces the skeleton. Tailwind's `animate-pulse` handles the shimmer animation automatically.

---

## 9. React Render Performance

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
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

Use `useCallback` for functions passed as props:

```tsx
const handleClick = useCallback((id: string) => {
  setSelected(id);
}, []);
```

### Context Splitting

A single large context causes all consumers to re-render when any value changes. Split contexts by update frequency:

```tsx
// Bad — everything re-renders when count changes
const AppContext = createContext({ user: null, theme: 'light', count: 0 });

// Good — split by update frequency
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<'light' | 'dark'>('light');
const CounterContext = createContext({ count: 0, increment: () => {} });
```

For state that changes frequently but is consumed by many components, consider a selector-based store like Zustand:

```tsx
import { create } from 'zustand';

interface AppStore {
  count: number;
  increment: () => void;
}

const useStore = create<AppStore>((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}));

// Only re-renders when count changes
const count = useStore((s) => s.count);
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

### Keys and Reconciliation

Always use stable, unique keys for list items. Never use array index as key for lists that can reorder, filter, or have items added/removed:

```tsx
// Bad — index keys cause incorrect reconciliation on reorder
{items.map((item, i) => <Card key={i} item={item} />)}

// Good — stable ID keys
{items.map(item => <Card key={item.id} item={item} />)}
```

---

## 10. Data Fetching Performance

### Supabase: Select Only Needed Columns

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

### Stale-While-Revalidate Pattern

Show cached data immediately while fetching fresh data in the background. This dramatically improves perceived performance:

```tsx
import { useState, useEffect, useRef } from 'react';

function useStaleWhileRevalidate<T>(key: string, fetcher: () => Promise<T>) {
  const [data, setData] = useState<T | null>(() => {
    const cached = sessionStorage.getItem(key);
    return cached ? JSON.parse(cached) : null;
  });
  const [loading, setLoading] = useState(!data);

  useEffect(() => {
    let cancelled = false;

    async function load() {
      try {
        const fresh = await fetcher();
        if (!cancelled) {
          setData(fresh);
          sessionStorage.setItem(key, JSON.stringify(fresh));
        }
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    load();
    return () => { cancelled = true; };
  }, [key]);

  return { data, loading };
}
```

### Optimistic Updates

For mutations, update the UI immediately and roll back on failure:

```tsx
async function toggleFavorite(postId: string) {
  // Optimistic: update UI immediately
  setFavorites(prev => {
    const isFav = prev.includes(postId);
    return isFav ? prev.filter(id => id !== postId) : [...prev, postId];
  });

  const { error } = await supabase
    .from('favorites')
    .upsert({ user_id: user.id, post_id: postId });

  if (error) {
    // Rollback on failure
    setFavorites(prev => {
      const isFav = prev.includes(postId);
      return isFav ? prev.filter(id => id !== postId) : [...prev, postId];
    });
  }
}
```

---

## 11. Memory Management

### AbortController for Fetch Requests

Cancel in-flight requests when a component unmounts or when a new request supersedes the old one:

```tsx
useEffect(() => {
  const controller = new AbortController();

  async function fetchData() {
    try {
      const response = await fetch('/api/data', { signal: controller.signal });
      const data = await response.json();
      setData(data);
    } catch (err) {
      if (err instanceof DOMException && err.name === 'AbortError') return;
      console.error('Fetch failed:', err);
    }
  }

  fetchData();
  return () => controller.abort();
}, []);
```

### Subscription Cleanup

Always clean up Supabase realtime subscriptions and other event sources:

```tsx
useEffect(() => {
  if (!supabase) return;

  const channel = supabase
    .channel('posts-changes')
    .on('postgres_changes', { event: '*', schema: 'public', table: 'posts' }, handleChange)
    .subscribe();

  return () => {
    supabase.removeChannel(channel);
  };
}, []);
```

### Timer and Interval Cleanup

Always clear timers on unmount:

```tsx
useEffect(() => {
  const interval = setInterval(() => {
    // polling logic
  }, 5000);

  return () => clearInterval(interval);
}, []);
```

### Detecting Memory Leaks

Common signs and their causes:

| Symptom | Likely Cause |
|---------|-------------|
| "Can't perform a React state update on an unmounted component" | Missing cleanup in useEffect |
| Page gets slower over time | Event listeners accumulating without removal |
| Browser tab memory grows continuously | Realtime subscriptions or intervals not cleaned up |
| Large DOM node count | Lists rendered without virtualization |

Use Chrome DevTools Memory tab to take heap snapshots and compare them over time.

---

## 12. Network & Caching

### HTTP Caching Headers

Configure your hosting platform to serve proper cache headers:

| Resource Type | Cache-Control | Why |
|---------------|---------------|-----|
| HTML (`index.html`) | `no-cache` | Always check for updates |
| JS/CSS (hashed by Vite) | `max-age=31536000, immutable` | Content-addressed, never changes |
| Fonts | `max-age=31536000, immutable` | Rarely change |
| Images | `max-age=86400` (1 day) to `max-age=31536000` | Depends on change frequency |
| API responses | `no-store` or `max-age=60` | Depends on data freshness needs |

Vite automatically adds content hashes to JS and CSS filenames (e.g., `main-a1b2c3d4.js`), making them safe to cache forever.

### Service Worker for Offline/Cache-First

For apps that benefit from offline support or aggressive caching, use Workbox with the Vite PWA plugin:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com/,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'google-fonts-stylesheets' },
          },
          {
            urlPattern: /^https:\/\/fonts\.gstatic\.com/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'google-fonts-webfonts',
              expiration: { maxEntries: 30, maxAgeSeconds: 60 * 60 * 24 * 365 },
            },
          },
        ],
      },
    }),
  ],
});
```

### Prefetch Next Page Resources

When you can predict the user's next navigation, prefetch those resources:

```tsx
function NavLink({ to, children }: { to: string; children: React.ReactNode }) {
  const prefetch = useCallback(() => {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = to;
    document.head.appendChild(link);
  }, [to]);

  return (
    <Link to={to} onMouseEnter={prefetch} onFocus={prefetch}>
      {children}
    </Link>
  );
}
```

---

## 13. Loading Sequence in index.html

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

## 14. Measuring Performance

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
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

Send metrics to your analytics for production monitoring:

```tsx
import { onLCP, onINP, onCLS, onFCP, onTTFB, type Metric } from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
  });
  
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/vitals', body);
  }
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

### Performance Budgets

Define maximum sizes for your bundles and enforce them in CI:

```ts
// vite.config.ts
export default defineConfig({
  build: {
    chunkSizeWarningLimit: 250, // warn if a chunk exceeds 250KB
  },
});
```

Create a budget file for CI enforcement:

| Resource | Budget |
|----------|--------|
| Total JS (compressed) | < 200KB |
| Total CSS (compressed) | < 50KB |
| Largest single chunk | < 100KB |
| Total font files | < 100KB |
| Hero image | < 150KB |
| Lighthouse Performance score | >= 90 |

---

## 15. robots.txt and Sitemap

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
2. Check `index.html` for missing preconnect, preload, resource hints, and font loading setup
3. Check all `<img>` tags for missing `width`, `height`, `loading`, `fetchPriority`, and `alt` attributes
4. Check for render-blocking third-party scripts — defer analytics, tracking, and chat widgets
5. Check CSS animations — replace layout-triggering properties with `transform`/`opacity`
6. Check event listeners — ensure scroll/resize handlers are passive and cleaned up on unmount
7. Check for memory leaks — verify `AbortController`, subscription cleanup, and timer disposal
8. Check React render patterns — look for missing `React.memo`, expensive inline calculations, and unsplit contexts
9. Check for code splitting — verify routes use `React.lazy()` and `<Suspense>`
10. Check Vite config for manual chunk splitting, compression, and tree shaking
11. Check Supabase queries — use column selection, pagination, and indexes
12. Check for large dependencies — audit bundle with `npx vite-bundle-visualizer`
13. Run Lighthouse again to measure improvement
14. Run through the Performance Checklist (section 16)

---

## Output Format

When optimizing a page or component, include:

1. **Fixed `<img>` tags** — with `width`, `height`, `loading`, `fetchPriority`, `srcset` where needed, and descriptive `alt`
2. **Optimized event listeners** — passive where applicable, cleanup in `useEffect` return
3. **CSS animation fixes** — `transform`/`opacity` instead of layout properties, `will-change` where needed
4. **`prefers-reduced-motion`** — media query or JS check for animations
5. **React render fixes** — `React.memo`, `useMemo`, `useCallback`, context splitting where needed

When optimizing project-level performance, include:

1. **`index.html` `<head>`** — preconnect, preload, deferred scripts in correct order
2. **`vite.config.ts`** — manual chunks, compression, chunk size warning limit
3. **Route-level code splitting** — `React.lazy()` and `<Suspense>` wrapping routes
4. **`public/robots.txt`** — with sitemap reference
5. **Lighthouse before/after scores** — to verify improvements

When modifying existing code, preserve:

1. Existing `useEffect` cleanup patterns
2. Existing `useCallback` / `useMemo` usage
3. Existing Tailwind class names (don't remove classes that may be used conditionally)
4. Existing component API/props interfaces
5. Existing routing structure and paths

---

## 16. Performance Checklist

### Critical Rendering Path
- [ ] Loading shell in `index.html` inside `<div id="root">` so users see feedback before JS loads
- [ ] Resource hints (`preconnect`, `dns-prefetch`, `preload`) are in `<head>`
- [ ] `index.html` loads resources in optimal order (section 13)
- [ ] No render-blocking third-party scripts in `<head>`

### Fonts
- [ ] Fonts use `preconnect` and `preload`
- [ ] Fonts use `display=swap` (or `font-display: swap` for self-hosted)
- [ ] Font files are woff2 format
- [ ] Fonts are subsetted to needed character ranges
- [ ] Variable font used where possible (single file for all weights)
- [ ] Total font download < 100KB

### Images
- [ ] All images have `width` and `height` attributes
- [ ] Hero/logo images use `loading="eager"` and `fetchPriority="high"`
- [ ] Below-fold images use `loading="lazy"`
- [ ] All images have descriptive `alt` text
- [ ] Images are compressed (WebP or AVIF where possible)
- [ ] Responsive images use `srcset` and `sizes` for large hero/banner images

### Third-Party Scripts
- [ ] Analytics/tracking scripts are deferred (load after page ready)
- [ ] Chat widgets load on interaction or after delay
- [ ] Social embeds are lazy loaded or replaced with static previews

### CSS & Animations
- [ ] Animations use `transform`/`opacity` only (no `top`/`left`/`width`/`height`)
- [ ] `will-change` hints on continuously animated elements (used sparingly)
- [ ] `prefers-reduced-motion` is respected
- [ ] Tailwind CSS purge paths cover all files
- [ ] Production CSS < 30KB

### JavaScript & React
- [ ] Routes use `React.lazy()` and `<Suspense>`
- [ ] Suspense fallbacks use skeleton screens (not spinners or blank) for page-level loading
- [ ] Heavy components behind interactions (modals, dialogs, conditional panels) are lazy loaded
- [ ] `React.memo` on expensive components with stable props
- [ ] Contexts are split by update frequency
- [ ] Large lists (100+ items) are virtualized
- [ ] Stable, unique keys on all list items

### Event Listeners & Memory
- [ ] Scroll/resize listeners are `{ passive: true }`
- [ ] All event listeners are cleaned up on unmount
- [ ] Fetch requests use `AbortController`
- [ ] Supabase realtime subscriptions are cleaned up
- [ ] Timers and intervals are cleared on unmount

### Data Fetching
- [ ] Supabase queries select only needed columns
- [ ] Large datasets are paginated
- [ ] Frequently filtered/sorted columns have database indexes
- [ ] Stale-while-revalidate or caching pattern for repeated queries

### Bundle & Build
- [ ] Vite config splits vendor chunks
- [ ] Bundle size audited with `npx vite-bundle-visualizer`
- [ ] No oversized dependencies (moment.js, full lodash, full icon libraries)
- [ ] Chunk size warning limit set in Vite config

### SEO & Crawlability
- [ ] `robots.txt` exists and is valid
- [ ] `sitemap.xml` exists and is valid
- [ ] `<link rel="canonical">` is set
- [ ] `<meta name="theme-color">` is set

### Measurement
- [ ] Lighthouse performance score is 90+
- [ ] Core Web Vitals meet targets (LCP < 2.5s, INP < 200ms, CLS < 0.1)
- [ ] `web-vitals` library installed for real user monitoring (optional)
