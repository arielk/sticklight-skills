---
title: Performance
version: "1.0"
description: |
  Core Web Vitals optimization for React + TypeScript + Vite + Tailwind CSS
  apps built with sticklight.com. Use when optimizing page load speed,
  improving Lighthouse scores, fixing First Contentful Paint (FCP),
  Largest Contentful Paint (LCP), Cumulative Layout Shift (CLS), Total
  Blocking Time (TBT), or Interaction to Next Paint (INP). Focused on
  what search engines and users experience on page load.
  Covers: critical rendering path (loading shell, resource hints,
  above-the-fold optimization), font loading strategy (preconnect,
  preload, self-hosting, display=swap, subsetting, variable fonts),
  third-party script deferral (analytics, GTM, chat widgets), image
  optimization (dimensions, loading priority, fetchPriority, WebP/AVIF,
  responsive images with srcset, src/assets vs public), CSS performance
  (GPU-accelerated animations, transform/opacity, will-change, Tailwind
  CSS purge, content-visibility), passive event listeners, animation
  best practices (CSS-only, prefers-reduced-motion), code splitting
  with React.lazy and dynamic imports, skeleton screens, avoiding
  heavy dependencies, loading sequence priority in index.html, and
  measuring with Lighthouse and web-vitals.
---

# Performance & Loading Optimization Skill for Sticklight Apps

You are a performance auditor that identifies and fixes Core Web Vitals issues in React + Vite + Tailwind CSS apps. When given code to optimize, scan for the anti-patterns listed below, fix each one, preserve the component's behavior, and verify the result against Lighthouse targets. Focus on what search engines and users experience when the page loads.

## Your Task

When this skill is active, follow these steps:

1. **Scan for anti-patterns** — Read the code and identify every performance issue from the Anti-Patterns Quick Reference below
2. **Fix each issue** — Apply the correct pattern from the detailed sections (1–11)
3. **Preserve behavior** — Keep the component's functionality, props API, and routing intact
4. **Self-audit** — After fixing, ask: "What Core Web Vitals issues remain in this code?" Answer briefly, then fix any remaining issues
5. **Verify** — Run through the Performance Checklist (section 12) before shipping

## Anti-Patterns Quick Reference

Scan every file for these. Each links to a detailed section with the correct fix.

### Images (section 4) — affects CLS, LCP
- `<img>` without `width` and `height` → causes CLS
- `<img>` without `loading` attribute → all images load eagerly by default
- Hero/logo images without `fetchPriority="high"` → delays LCP
- `<img>` without `className="w-full h-auto"` → renders at fixed pixel size instead of responsive
- Missing `alt` text → accessibility and SEO issue
- Static image path like `src="/hero.jpg"` instead of importing from `src/assets/` → misses Vite hashing and optimization

### Fonts (section 2) — affects FCP, LCP, CLS
- Google Fonts `<link>` without `preconnect` above it → slow DNS + connection
- Missing `display=swap` or `font-display: swap` → invisible text during load (FOIT)
- Multiple static font weight files when a variable font exists → unnecessary requests
- Font files in formats other than woff2 → larger downloads

### Scripts (section 3) — affects FCP, TBT, LCP
- Analytics/GTM loaded in `<head>` without deferral → blocks rendering
- Chat widgets or social embeds loaded on page init → hundreds of KB before interaction

### CSS & Animations (section 5) — affects INP, TBT, CLS
- Animating `top`, `left`, `width`, `height`, `margin` → triggers layout on every frame
- `will-change` on many elements or left permanently → wastes GPU memory
- Missing `@media (prefers-reduced-motion: reduce)` → Lighthouse accessibility flag
- Tailwind dynamic classes via string interpolation → purge can't detect them, inflates CSS

### Event Listeners (section 6) — affects INP, TBT
- `addEventListener('scroll', handler)` without `{ passive: true }` → Lighthouse flags this directly

### Code Splitting & Bundle (section 7) — affects FCP, TBT, LCP
- All routes in one bundle (no `React.lazy`) → full app downloads on first page
- Components that always render wrapped in `lazy()` → adds waterfall for no benefit
- Modals/dialogs not lazy-loaded → code downloaded even if never opened
- `<Suspense fallback={null}>` for page-level loading → blank screen, poor perceived perf
- Full lodash imported instead of `lodash-es` or individual imports → 70KB+ added to bundle
- `moment` instead of `date-fns` or `dayjs` → 300KB+ added to bundle
- Full icon library instead of individual icon imports → large bundle

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

**Affects:** FCP, LCP

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

**Affects:** FCP, LCP, CLS

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

For maximum control, download fonts and serve them from your own domain. This eliminates the DNS lookup and connection to Google Fonts entirely.

Place font files in `src/assets/fonts/` so Vite can hash them for long-term caching. Reference them in your CSS:

```css
@font-face {
  font-family: 'Inter';
  src: url('@/assets/fonts/Inter-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}
```

For the `preload` link in `index.html`, fonts need a stable URL — place a copy in `public/fonts/` or use the Vite-resolved path after build:

```html
<link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
```

### Font Subsetting

Load only the character ranges you need. A full font file with all Unicode blocks can be 200KB+, but subsetting to just Latin and specific scripts can save up to 90%:

```css
@font-face {
  font-family: 'Inter';
  src: url('@/assets/fonts/Inter-Regular-Latin.woff2') format('woff2');
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
  src: url('@/assets/fonts/Inter-Variable.woff2') format('woff2-variations');
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

**Affects:** FCP, TBT, LCP

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

**Affects:** CLS, LCP

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

### Asset Placement: `src/assets/` vs `public/`

In a Vite project, where you put assets determines how they're served:

| Location | How to Reference | Vite Processing | Use For |
|----------|-----------------|-----------------|---------|
| `src/assets/` | `import logo from '@/assets/logo.svg'` then `src={logo}` | Hashed, bundled, optimized | Images, fonts, and files used in components |
| `public/` | `src="/hero.jpg"` (absolute path) | Served as-is, no hashing | `favicon.svg`, `robots.txt`, `og-image.jpeg`, files that need a stable URL |

Assets in `src/assets/` get content-hashed filenames (e.g., `logo-a1b2c3d4.svg`) enabling long-term caching. Assets in `public/` keep their original filename and are copied as-is to the build output.

```tsx
// Imported from src/assets/ — Vite hashes and optimizes
import logo from '@/assets/images/logo.svg';
import heroImage from '@/assets/images/hero.webp';

// Referenced from public/ — stable URL, no processing
<link rel="icon" href="/favicon.svg" />
```

**Rule of thumb:** if you `import` it in a component, put it in `src/assets/`. If it needs a fixed URL (meta tags, `index.html`, third-party references), put it in `public/`.

### Implementation Examples

```tsx
import logo from '@/assets/images/logo.svg';
import heroImage from '@/assets/images/hero.webp';
import galleryImage from '@/assets/images/gallery-preview.webp';

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

Serve different image sizes based on the viewport to avoid downloading oversized images on mobile. Import each size from `src/assets/`:

```tsx
import hero400 from '@/assets/images/hero-400.webp';
import hero800 from '@/assets/images/hero-800.webp';
import hero1200 from '@/assets/images/hero-1200.webp';

<img
  srcSet={`${hero400} 400w, ${hero800} 800w, ${hero1200} 1200w`}
  sizes="(max-width: 640px) 400px, (max-width: 1024px) 800px, 1200px"
  src={hero800}
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

Always compress images before adding to the project. Place them in `src/assets/` for component use (Vite will hash and optimize them) or `public/` for stable-URL files like `og-image.jpeg`. Target file sizes:

| Image Type | Target Size |
|------------|-------------|
| Hero / full-width | < 150KB |
| Thumbnail / card | < 50KB |
| Icon / logo | < 10KB |

---

## 5. CSS Performance

**Affects:** INP, TBT, CLS, FCP

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

### Will-Change Property

Hint the browser about upcoming animations to optimize rendering. Use sparingly — overuse wastes GPU memory:

```tsx
// For animated elements (marquee, sliders, carousels)
style={{ willChange: 'transform' }}
```

Remove `will-change` when the animation is done if it's a one-time transition:

```tsx
onTransitionEnd={() => ref.current.style.willChange = 'auto'}
```

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

### Reduce Motion Preference

Respect the user's motion preferences. Lighthouse flags missing reduced-motion support:

```css
@media (prefers-reduced-motion: reduce) {
  * {
    transition-duration: 0.01ms !important;
    animation-duration: 0.01ms !important;
  }
}
```

---

## 6. Passive Event Listeners

**Affects:** INP, TBT — Lighthouse flags this directly

Never block scroll events. Passive listeners tell the browser the handler will not call `preventDefault()`, so scrolling can proceed without waiting:

```tsx
// Correct — passive, non-blocking
window.addEventListener('scroll', handleScroll, { passive: true });

// Wrong — Lighthouse will flag this
window.addEventListener('scroll', handleScroll);
```

Always add cleanup to prevent listener accumulation:

```tsx
useEffect(() => {
  const handleScroll = () => {
    // scroll logic
  };
  window.addEventListener('scroll', handleScroll, { passive: true });

  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

---

## 7. Code Splitting & Bundle Size

**Affects:** FCP, TBT, LCP

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

### Avoid Heavy Dependencies

Large dependencies inflate the initial JS bundle, directly hurting FCP and TBT. Use lightweight alternatives:

| Problem | Solution |
|---------|----------|
| Full lodash imported | Use `lodash-es` or individual imports: `import debounce from 'lodash-es/debounce'` |
| Moment.js (300KB+) | Replace with `date-fns` or `dayjs` (2-7KB) |
| Full icon library imported | Import individual icons: `import { IconMenu } from '@tabler/icons-react'` |

### Skeleton Screens

Use skeleton screens instead of spinners for `<Suspense>` fallbacks. Skeletons mimic the shape of the incoming content, reducing perceived load time:

```tsx
function PageSkeleton() {
  return (
    <div className="animate-pulse space-y-4 p-6">
      <div className="h-8 w-48 rounded bg-gray-200" />
      <div className="h-64 w-full rounded-lg bg-gray-200" />
      <div className="space-y-2">
        <div className="h-4 w-full rounded bg-gray-200" />
        <div className="h-4 w-5/6 rounded bg-gray-200" />
        <div className="h-4 w-4/6 rounded bg-gray-200" />
      </div>
    </div>
  );
}

<Suspense fallback={<PageSkeleton />}>
  <Dashboard />
</Suspense>
```

Match skeleton dimensions to the real content to avoid CLS when the actual content replaces the skeleton.

---

## 8. Loading Sequence in index.html

**Affects:** FCP, LCP

Structure your `<head>` to load resources in optimal order. Critical resources first, non-critical last:

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- 1. Preconnect to external domains (establish connections early) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link rel="dns-prefetch" href="https://your-project.supabase.co" />

  <!-- 2. Preload critical assets (fonts) -->
  <link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
  <link rel="preload" as="style" href="https://fonts.googleapis.com/css2?family=Inter&display=swap" />

  <!-- 3. Font stylesheet -->
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
  <div id="root">
    <!-- 6. Loading shell — visible until React mounts (see section 1) -->
    <div style="display:flex;align-items:center;justify-content:center;min-height:100vh;font-family:system-ui,sans-serif">
      <div style="width:40px;height:40px;border:3px solid #e5e7eb;border-top-color:#3b82f6;border-radius:50%;animation:spin 0.6s linear infinite"></div>
    </div>
  </div>
  <style>@keyframes spin{to{transform:rotate(360deg)}}</style>

  <!-- 7. Main app bundle (Vite injects this) -->
  <script type="module" src="/src/main.tsx"></script>

  <!-- 8. Deferred analytics (load after everything else) -->
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
| 2nd | Preload critical assets | Start downloading fonts early |
| 3rd | Font stylesheets | Render text correctly from the start |
| 4th | Meta tags | SEO and browser hints |
| 5th | Loading shell | Immediate visual feedback inside `<div id="root">` |
| 6th | App bundle | The actual application (Vite injects CSS alongside JS) |
| 7th | Eager images | Above-the-fold content (rendered by React) |
| 8th | Lazy images | Below-the-fold, loaded on scroll |
| Last | Analytics / tracking | Non-critical, loaded after page is ready |

---

## 9. Measuring Performance

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

### Performance Budgets

Target sizes to aim for when writing code and choosing dependencies:

| Resource | Budget |
|----------|--------|
| Total font files | < 100KB |
| Hero image | < 150KB |
| Lighthouse Performance score | >= 90 |

---

## Process

When optimizing a project's performance, follow this workflow:

1. Read the code carefully
2. Scan for every anti-pattern in the Anti-Patterns Quick Reference
3. Fix each issue using the patterns from sections 1–9
4. Ensure the revised code preserves existing behavior, props interfaces, and routing
5. **Self-audit** — ask: "What Core Web Vitals issues remain in this code?" Answer with a brief list
6. Fix any remaining issues found in the self-audit
7. Run through the Performance Checklist (section 12)
8. Present the final optimized code

---

## Output Format

Provide:

1. **Optimized code** — the full rewritten component or file
2. **"What Core Web Vitals issues remain?"** — brief bullet list of any remaining concerns (or "None" if clean)
3. **Final revision** — fix any issues found in step 2
4. **Summary of changes** — brief list of what was fixed and which CWV metric it improves

When modifying existing code, preserve:

1. Existing component API / props interfaces
2. Existing routing structure and paths
3. Existing `useEffect` cleanup patterns (don't remove existing cleanups)
4. Existing Tailwind class names (don't remove classes that may be used conditionally)

---

## Full Example

**Before (unoptimized component):**

```tsx
import { useEffect, useState } from 'react';
import { supabase } from '@/integrations/supabase/client';

export default function Gallery() {
  const [images, setImages] = useState([]);

  useEffect(() => {
    supabase.from('images').select('*').then(({ data }) => {
      setImages(data || []);
    });
  }, []);

  useEffect(() => {
    window.addEventListener('scroll', () => {
      console.log(window.scrollY);
    });
  }, []);

  return (
    <div>
      <img src="/hero.jpg" alt="" />
      <div style={{ animation: 'slideIn 0.3s', left: '100px' }}>
        <h2>Gallery</h2>
      </div>
      {images.map((img, i) => (
        <img key={i} src={img.url} />
      ))}
    </div>
  );
}
```

**Optimized code:**

```tsx
import { useEffect, useState } from 'react';
import { supabase } from '@/integrations/supabase/client';
import heroBanner from '@/assets/images/hero.webp';

export default function Gallery() {
  const [images, setImages] = useState<{ id: string; url: string; alt: string }[]>([]);

  useEffect(() => {
    supabase
      .from('images')
      .select('id, url, alt')
      .then(({ data }) => setImages(data || []));
  }, []);

  useEffect(() => {
    const handleScroll = () => {
      console.log(window.scrollY);
    };
    window.addEventListener('scroll', handleScroll, { passive: true });

    return () => window.removeEventListener('scroll', handleScroll);
  }, []);

  return (
    <div>
      <img
        src={heroBanner}
        alt="Gallery hero banner"
        width={1200}
        height={630}
        className="w-full h-auto"
        loading="eager"
        fetchPriority="high"
      />
      <div className="animate-slide-in">
        <h2>Gallery</h2>
      </div>
      {images.map((img) => (
        <img
          key={img.id}
          src={img.url}
          alt={img.alt}
          width={400}
          height={300}
          className="w-full h-auto"
          loading="lazy"
        />
      ))}
    </div>
  );
}
```

**What Core Web Vitals issues remain?**
- No `prefers-reduced-motion` for the slide-in animation (Lighthouse accessibility)
- Inline `left: '100px'` animation was removed but CSS class should use `transform` not `left`

**Final revision** — add reduced-motion and verify CSS uses transform:

```css
.animate-slide-in {
  transform: translateX(100px);
  animation: slideIn 0.3s ease forwards;
}

@keyframes slideIn {
  to { transform: translateX(0); }
}

@media (prefers-reduced-motion: reduce) {
  .animate-slide-in {
    animation: none;
    transform: none;
  }
}
```

**Changes made:**
- **CLS fix:** Hero image — added `width`, `height`, `className="w-full h-auto"`, descriptive `alt`
- **LCP fix:** Hero image — added `loading="eager"`, `fetchPriority="high"`, imported from `src/assets/`
- **CLS fix:** Gallery images — added `width`, `height`, `className="w-full h-auto"`, `loading="lazy"`, descriptive `alt`
- **INP/TBT fix:** Scroll listener — added `{ passive: true }`, added cleanup on unmount
- **TBT fix:** Animation — replaced `left` (layout-triggering) with `transform: translateX()` (GPU-accelerated)
- **Accessibility:** Added `@media (prefers-reduced-motion: reduce)`

---

## 12. Performance Checklist

### Critical Rendering Path
- [ ] Loading shell in `index.html` inside `<div id="root">`
- [ ] Resource hints (`preconnect`, `dns-prefetch`, `preload`) are in `<head>`
- [ ] `index.html` loads resources in optimal order (section 8)
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
- [ ] Component images imported from `src/assets/` (not hardcoded paths)

### Third-Party Scripts
- [ ] Analytics/tracking scripts are deferred (load after page ready)
- [ ] Chat widgets load on interaction or after delay
- [ ] Social embeds are lazy loaded or replaced with static previews

### CSS & Animations
- [ ] Animations use `transform`/`opacity` only (no `top`/`left`/`width`/`height`)
- [ ] `will-change` used sparingly and removed after one-time transitions
- [ ] `prefers-reduced-motion` is respected
- [ ] Tailwind CSS purge paths cover all files
- [ ] Production CSS < 30KB

### Event Listeners
- [ ] Scroll/touch listeners are `{ passive: true }`

### Code Splitting & Bundle
- [ ] Routes use `React.lazy()` and `<Suspense>`
- [ ] Suspense fallbacks use skeleton screens for page-level loading
- [ ] Heavy components behind interactions (modals, dialogs) are lazy loaded
- [ ] No oversized dependencies (moment.js, full lodash, full icon libraries)

### Measurement
- [ ] Lighthouse performance score is 90+
- [ ] Core Web Vitals meet targets (LCP < 2.5s, INP < 200ms, CLS < 0.1)
