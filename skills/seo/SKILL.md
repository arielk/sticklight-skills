---
name: seo
description: SEO best practices and optimization guidelines. Use when building or reviewing web pages, writing content for the web, generating meta tags, improving search engine rankings, or when the user asks about SEO.
---

# SEO Optimization

Apply search engine optimization best practices when building, reviewing, or improving web content.

## On-Page SEO

- Write unique, descriptive `<title>` tags (50-60 characters) with the primary keyword near the front
- Write compelling `<meta name="description">` tags (150-160 characters) that summarize the page and include a call to action
- Use a single `<h1>` per page that clearly describes the page topic
- Structure content with a logical heading hierarchy (`h1` > `h2` > `h3`)
- Use descriptive, keyword-rich URLs with hyphens (e.g. `/blog/seo-best-practices`)
- Add `alt` attributes to all images describing their content
- Use internal links with descriptive anchor text

## Technical SEO

- Ensure pages are mobile-responsive
- Use semantic HTML elements (`<article>`, `<nav>`, `<main>`, `<section>`, `<header>`, `<footer>`)
- Add canonical URLs (`<link rel="canonical">`) to avoid duplicate content issues
- Implement Open Graph (`og:`) and Twitter Card meta tags for social sharing
- Add structured data (JSON-LD) where applicable (Article, Product, FAQ, Breadcrumb, etc.)
- Ensure fast page load times: optimize images, minimize CSS/JS, use lazy loading
- Create and maintain a `sitemap.xml`
- Configure `robots.txt` properly
- Use HTTPS everywhere
- Implement `hreflang` tags for multilingual content

## Content Guidelines

- Write clear, helpful content that answers user intent
- Use the target keyword naturally in the first 100 words
- Use related keywords and synonyms throughout the content
- Keep paragraphs short and scannable
- Use bullet points and numbered lists for readability
- Add a table of contents for long-form content
- Update content regularly to keep it fresh and accurate

## Examples

### Meta Tags

```html
<title>Best Practices for SEO in 2025 | Your Site</title>
<meta name="description" content="Learn the latest SEO best practices to improve your search rankings, drive organic traffic, and create content that both users and search engines love.">
<link rel="canonical" href="https://example.com/blog/seo-best-practices">
```

### Structured Data (JSON-LD)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Best Practices for SEO",
  "author": { "@type": "Person", "name": "Author Name" },
  "datePublished": "2025-01-15",
  "description": "A comprehensive guide to SEO best practices."
}
</script>
```

### Open Graph Tags

```html
<meta property="og:title" content="Best Practices for SEO">
<meta property="og:description" content="Learn the latest SEO best practices.">
<meta property="og:type" content="article">
<meta property="og:url" content="https://example.com/blog/seo-best-practices">
<meta property="og:image" content="https://example.com/images/seo-guide.jpg">
```
