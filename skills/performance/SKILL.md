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
  above-the-fold optimization), font loading (preconnect, preload,
  display=swap), third-party script deferral (analytics, GTM, chat
  widgets), image optimization (dimensions, loading priority,
  fetchPriority), CSS performance (GPU-accelerated animations,
  transform/opacity, prefers-reduced-motion), passive event listeners,
  code splitting with React.lazy, skeleton screens, and avoiding
  heavy dependencies.
---

# Performance & Loading Optimization Skill for Sticklight Apps

You are a performance auditor that identifies and fixes Core Web Vitals issues in React + Vite + Tailwind CSS apps. When given code to optimize, scan for the anti-patterns listed below, fix each one, preserve the component's behavior, and verify the result against Lighthouse targets. Focus on what search engines and users experience when the page loads.

## Your Task

When this skill is active, follow these steps:

1. **Scan for anti-patterns** — Read the code and identify every performance issue from the Anti-Patterns Quick Reference below
2. **Fix each issue** — Apply the correct pattern from the detailed sections (1–7)
3. **Preserve behavior** — Keep the component's functionality, props API, and routing intact
4. **Self-audit** — After fixing, ask: "What Core Web Vitals issues remain in this code?" Answer briefly, then fix any remaining issues
5. **Verify** — Run through the Performance Checklist (section 8) before shipping

## Anti-Patterns Quick Reference

Scan every file for these. Each links to a detailed section with the correct fix.

### Images (section 3) — affects CLS, LCP
- `<img>` without `width` and `height` → causes CLS
- `<img>` without `loading` attribute → all images load eagerly by default
- Hero/logo images without `fetchPriority="high"` → delays LCP
- `<img>` without `className="w-full h-auto"` → renders at fixed pixel size instead of responsive
- Missing `alt` text → accessibility and SEO issue

### Fonts (section 2) — affects FCP, LCP, CLS
- Google Fonts `<link>` without `preconnect` above it → slow DNS + connection
- Missing `display=swap` or `font-display: swap` → invisible text during load (FOIT)

### Scripts (section 4) — affects FCP, TBT, LCP
- Analytics/GTM loaded in `<head>` without deferral → blocks rendering
- Chat widgets or social embeds loaded on page init → hundreds of KB before interaction

### CSS & Animations (section 5) — affects INP, TBT, CLS
- Animating `top`, `left`, `width`, `height`, `margin` → triggers layout on every frame
- Missing `@media (prefers-reduced-motion: reduce)` → Lighthouse accessibility flag
- Tailwind dynamic classes via string interpolation → purge can't detect them, inflates CSS

### Event Listeners (section 6) — affects INP, TBT
- `addEventListener('scroll', handler)` without `{ passive: true }` → Lighthouse flags this directly

### Code Splitting (section 7) — affects FCP, TBT, LCP
- All routes in one bundle (no `React.lazy`) → full app downloads on first page
- Modals/dialogs not lazy-loaded → code downloaded even if never opened
- `<Suspense fallback={null}>` for page-level loading → blank screen
- Full lodash imported instead of `lodash-es` or individual imports → 70KB+ added
- `moment` instead of `date-fns` or `dayjs` → 300KB+ added
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

**Affects:** FCP, LCP

### Loading Shell (SPA-Specific)

In a React SPA, the browser shows a blank page until JavaScript downloads, parses, and renders the first component. Add a lightweight loading shell directly in `index.html` so users see something instantly while the JS bundle loads:

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

React's `createRoot(document.getElementById('root')!).render(...)` automatically replaces the loading shell content when the app mounts.

### Resource Hints

Place resource hints at the top of `<head>`, before any other tags:

| Hint | Purpose | When to Use |
|------|---------|-------------|
| `preconnect` | Establish connection (DNS + TCP + TLS) early | Domains you will fetch from in the first few seconds |
| `dns-prefetch` | Resolve DNS only | Domains used later or as a fallback for `preconnect` |
| `preload` | Download a specific resource immediately | Critical fonts, hero images |

```html
<head>
  <!-- Preconnect: domains used immediately -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  
  <!-- DNS prefetch: domains used later -->
  <link rel="dns-prefetch" href="https://your-project.supabase.co" />
  
  <!-- Preload: critical assets -->
  <link rel="preload" as="font" type="font/woff2" href="/fonts/Inter-Regular.woff2" crossorigin />
</head>
```

### Above-the-Fold Priority

Structure your components so critical content renders first:

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

## 2. Font Loading

**Affects:** FCP, LCP, CLS

### Preconnect to Font Servers

Place these at the top of `<head>` in `index.html`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
```

### Preload Font Stylesheets

```html
<link rel="preload" as="style" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" />
```

### Font Display Strategy

Always use `display=swap` to prevent invisible text during font loading (FOIT). The browser shows fallback text immediately, then swaps in the web font when ready.

---

## 3. Image Optimization

**Affects:** CLS, LCP

### Always Include Dimensions

Prevent Cumulative Layout Shift (CLS) by specifying `width` and `height` on every image. Adding `className="w-full h-auto"` (Tailwind) makes the image scale responsively:

```tsx
<img
  src={image}
  alt="Product photo showing red sneakers"
  width={800}
  height={600}
  className="w-full h-auto"
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

---

## 4. Third-Party Scripts (Analytics, GTM)

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

### Third-Party Script Audit

Common offenders and how to handle them:

| Script | Impact | Solution |
|--------|--------|----------|
| Google Tag Manager | 50-100KB, blocks main thread | Post-load injection (see above) |
| Chat widgets (Intercom, Drift) | 200-500KB, heavy JS | Load after user interaction or after 5s delay |
| Social embeds (Twitter, Instagram) | 100-300KB each | Use static screenshots with links instead |
| Google Maps | 200KB+, many sub-requests | Load on user interaction (click to load map) |
| reCAPTCHA | 150KB+ | Load only on pages that need it |

---

## 5. CSS Performance

**Affects:** INP, TBT, CLS

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

### Tailwind Dynamic Class Names

Tailwind purges unused CSS at build time. Dynamic string interpolation breaks this:

```tsx
// Bad — Tailwind cannot detect this
const color = isActive ? 'blue' : 'gray';
<div className={`bg-${color}-500`} />

// Good — Tailwind can detect complete class names
<div className={isActive ? 'bg-blue-500' : 'bg-gray-500'} />
```

### Reduce Motion Preference

Lighthouse flags missing reduced-motion support:

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

Never block scroll events. Passive listeners tell the browser the handler will not call `preventDefault()`:

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

Only lazy-load components that are **not rendered on initial page load** — modals, dialogs, and conditionally rendered widgets. If a component always renders, lazy-loading it just adds a waterfall.

```tsx
const ShareDialog = lazy(() => import('./components/ShareDialog'));

function Post() {
  const [showShare, setShowShare] = useState(false);

  return (
    <>
      <PostContent />
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

### Avoid Heavy Dependencies

Large dependencies inflate the initial JS bundle, directly hurting FCP and TBT:

| Problem | Solution |
|---------|----------|
| Full lodash imported | Use `lodash-es` or individual imports: `import debounce from 'lodash-es/debounce'` |
| Moment.js (300KB+) | Replace with `date-fns` or `dayjs` (2-7KB) |
| Full icon library imported | Import individual icons: `import { IconMenu } from '@tabler/icons-react'` |

### Skeleton Screens

Use skeleton screens instead of spinners for `<Suspense>` fallbacks. Match dimensions to avoid CLS:

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

---

## Process

When optimizing a project's performance, follow this workflow:

1. Read the code carefully
2. Scan for every anti-pattern in the Anti-Patterns Quick Reference
3. Fix each issue using the patterns from sections 1–7
4. Ensure the revised code preserves existing behavior, props interfaces, and routing
5. **Self-audit** — ask: "What Core Web Vitals issues remain in this code?" Answer with a brief list
6. Fix any remaining issues found in the self-audit
7. Run through the Performance Checklist (section 8)
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
- No `prefers-reduced-motion` for the slide-in animation
- Inline `left: '100px'` was removed but CSS class should use `transform` not `left`

**Final revision** — CSS for the animation:

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
- **LCP fix:** Hero image — added `loading="eager"`, `fetchPriority="high"`
- **CLS fix:** Gallery images — added `width`, `height`, `className="w-full h-auto"`, `loading="lazy"`, descriptive `alt`
- **INP/TBT fix:** Scroll listener — added `{ passive: true }`, added cleanup on unmount
- **TBT fix:** Animation — replaced `left` (layout-triggering) with `transform: translateX()` (GPU-accelerated)
- **Accessibility:** Added `@media (prefers-reduced-motion: reduce)`

---

## 8. Performance Checklist

### Critical Rendering Path
- [ ] Loading shell in `index.html` inside `<div id="root">`
- [ ] Resource hints (`preconnect`, `dns-prefetch`, `preload`) are in `<head>`
- [ ] No render-blocking third-party scripts in `<head>`

### Fonts
- [ ] Fonts use `preconnect` and `preload`
- [ ] Fonts use `display=swap` (or `font-display: swap` for self-hosted)

### Images
- [ ] All images have `width` and `height` attributes
- [ ] Hero/logo images use `loading="eager"` and `fetchPriority="high"`
- [ ] Below-fold images use `loading="lazy"`
- [ ] All images have descriptive `alt` text

### Third-Party Scripts
- [ ] Analytics/tracking scripts are deferred (load after page ready)
- [ ] Chat widgets load on interaction or after delay

### CSS & Animations
- [ ] Animations use `transform`/`opacity` only (no `top`/`left`/`width`/`height`)
- [ ] `prefers-reduced-motion` is respected
- [ ] No dynamic Tailwind class interpolation (`bg-${color}-500`)

### Event Listeners
- [ ] Scroll/touch listeners are `{ passive: true }`

### Code Splitting & Bundle
- [ ] Routes use `React.lazy()` and `<Suspense>`
- [ ] Suspense fallbacks use skeleton screens for page-level loading
- [ ] Heavy components behind interactions (modals, dialogs) are lazy loaded
- [ ] No oversized dependencies (moment.js, full lodash, full icon libraries)
