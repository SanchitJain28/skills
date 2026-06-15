---
name: Next.js SEO Expert
description: Expert assistant for Next.js App Router SEO, metadata, sitemap, robots.txt, structured data, Open Graph, canonical URLs, indexing issues, Core Web Vitals, and technical SEO audits.
---

# Next.js SEO Optimization

Use this skill when implementing or auditing SEO in a Next.js App Router project. Covers metadata, Open Graph, Twitter cards, JSON-LD structured data, sitemap, robots.txt, canonical URLs, OG image generation, and Core Web Vitals.

> Validated against Next.js 15+ App Router. Do NOT use `next-seo` package — it's for Pages Router. Use the built-in Metadata API exclusively.

---

## Quick Audit Checklist

```bash
curl https://your-site.com/robots.txt
curl https://your-site.com/sitemap.xml
# Check page source for <title>, <meta name="description">, application/ld+json
# Run Lighthouse in Chrome DevTools for Core Web Vitals
```

---

## 1. Root Metadata — `app/layout.tsx`

```typescript
import type { Metadata, Viewport } from 'next'

// Viewport MUST be a separate export (Next.js 14+)
export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  maximumScale: 5,
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
}

export const metadata: Metadata = {
  // Required for relative URLs in metadata to resolve correctly
  metadataBase: new URL('https://your-site.com'),

  title: {
    default: 'Site Title — Primary Keyword',
    template: '%s | Site Name', // Used by child pages
  },
  description: 'Concise, keyword-rich description (150–160 chars)',
  keywords: ['keyword1', 'keyword2'],

  openGraph: {
    type: 'website',
    locale: 'en_US',
    url: 'https://your-site.com',
    siteName: 'Site Name',
    title: 'Site Title',
    description: 'Description for social sharing',
    images: [
      {
        url: '/og-image.png', // 1200×630px
        width: 1200,
        height: 630,
        alt: 'Descriptive alt text',
      },
    ],
  },

  twitter: {
    card: 'summary_large_image',
    title: 'Site Title',
    description: 'Description for Twitter',
    images: ['/og-image.png'],
    // creator: '@handle', // Optional
  },

  alternates: {
    canonical: '/',
  },

  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      'max-image-preview': 'large',
      'max-snippet': -1,
    },
  },
}
```

---

## 2. Dynamic Metadata — `app/blog/[slug]/page.tsx`

```typescript
import type { Metadata } from 'next'

interface Props {
  params: { slug: string }
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await fetchPost(params.slug)

  if (!post) {
    return {
      title: 'Post Not Found',
      robots: { index: false },
    }
  }

  return {
    title: post.title,
    description: post.excerpt,
    alternates: {
      canonical: `/blog/${params.slug}`,
    },
    openGraph: {
      type: 'article',
      title: post.title,
      description: post.excerpt,
      publishedTime: post.publishedAt,
      modifiedTime: post.updatedAt,
      authors: [post.author.name],
      images: [
        {
          url: post.ogImage ?? '/og-default.png',
          width: 1200,
          height: 630,
          alt: post.title,
        },
      ],
    },
  }
}
```

---

## 3. Sitemap — `app/sitemap.ts`

```typescript
import type { MetadataRoute } from 'next'

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = 'https://your-site.com'

  const posts = await fetchAllPosts()

  const staticRoutes: MetadataRoute.Sitemap = [
    {
      url: baseUrl,
      lastModified: new Date(),
      changeFrequency: 'weekly',
      priority: 1,
    },
    {
      url: `${baseUrl}/about`,
      lastModified: new Date(),
      changeFrequency: 'monthly',
      priority: 0.8,
    },
  ]

  const dynamicRoutes: MetadataRoute.Sitemap = posts.map((post) => ({
    url: `${baseUrl}/blog/${post.slug}`,
    lastModified: new Date(post.updatedAt),
    changeFrequency: 'weekly',
    priority: 0.7,
  }))

  return [...staticRoutes, ...dynamicRoutes]
}
```

For large sites (>50k URLs), split into multiple sitemaps:

```typescript
// app/sitemap/[id]/route.ts
export async function GET(req: Request, { params }: { params: { id: string } }) {
  const page = Number(params.id)
  const posts = await fetchPostsPaginated(page, 1000)
  // Return XML manually or use a sitemap library
}
```

---

## 4. Robots — `app/robots.ts`

```typescript
import type { MetadataRoute } from 'next'

export default function robots(): MetadataRoute.Robots {
  const baseUrl = 'https://your-site.com'

  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: ['/api/', '/admin/', '/dashboard/', '/_not-found'],
        // NEVER disallow /_next/ — crawlers need CSS/JS to render pages
      },
    ],
    sitemap: `${baseUrl}/sitemap.xml`,
    host: baseUrl,
  }
}
```

---

## 5. JSON-LD Structured Data

Inject via a `<script>` tag in Server Components. Do not use `next/head` — that's Pages Router.

### Organization (site-wide, in `app/layout.tsx`)

```typescript
export default function RootLayout({ children }: { children: React.ReactNode }) {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Organization',
    name: 'Your Company',
    url: 'https://your-site.com',
    logo: 'https://your-site.com/logo.png',
    sameAs: [
      'https://twitter.com/yourhandle',
      'https://linkedin.com/company/yourcompany',
    ],
  }

  return (
    <html lang="en">
      <body>
        <script
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
        />
        {children}
      </body>
    </html>
  )
}
```

### Article (blog post page)

```typescript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: post.title,
  description: post.excerpt,
  image: post.ogImage,
  datePublished: post.publishedAt,
  dateModified: post.updatedAt,
  author: {
    '@type': 'Person',
    name: post.author.name,
    url: `https://your-site.com/authors/${post.author.slug}`,
  },
  publisher: {
    '@type': 'Organization',
    name: 'Your Site',
    logo: {
      '@type': 'ImageObject',
      url: 'https://your-site.com/logo.png',
    },
  },
}
```

### Product

```typescript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: product.name,
  description: product.description,
  image: product.images,
  sku: product.sku,
  offers: {
    '@type': 'Offer',
    price: product.price,
    priceCurrency: 'USD',
    availability: product.inStock
      ? 'https://schema.org/InStock'
      : 'https://schema.org/OutOfStock',
    url: `https://your-site.com/products/${product.slug}`,
  },
  aggregateRating: {
    '@type': 'AggregateRating',
    ratingValue: product.rating,
    reviewCount: product.reviewCount,
  },
}
```

### FAQ

```typescript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: faqs.map((faq) => ({
    '@type': 'Question',
    name: faq.question,
    acceptedAnswer: {
      '@type': 'Answer',
      text: faq.answer,
    },
  })),
}
```

### BreadcrumbList

```typescript
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'BreadcrumbList',
  itemListElement: [
    { '@type': 'ListItem', position: 1, name: 'Home', item: 'https://your-site.com' },
    { '@type': 'ListItem', position: 2, name: 'Blog', item: 'https://your-site.com/blog' },
    { '@type': 'ListItem', position: 3, name: post.title, item: `https://your-site.com/blog/${post.slug}` },
  ],
}
```

---

## 6. Dynamic OG Image — `app/blog/[slug]/opengraph-image.tsx`

```typescript
import { ImageResponse } from 'next/og'

export const runtime = 'edge'
export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

interface Props {
  params: { slug: string }
}

export default async function OGImage({ params }: Props) {
  const post = await fetchPost(params.slug)

  return new ImageResponse(
    (
      <div
        style={{
          width: '100%',
          height: '100%',
          display: 'flex',
          flexDirection: 'column',
          justifyContent: 'flex-end',
          padding: 60,
          backgroundColor: '#0a0a0a',
          color: '#ffffff',
          fontFamily: 'sans-serif',
        }}
      >
        <div style={{ fontSize: 48, fontWeight: 700, lineHeight: 1.2, marginBottom: 24 }}>
          {post.title}
        </div>
        <div style={{ fontSize: 24, opacity: 0.6 }}>your-site.com</div>
      </div>
    ),
    { ...size }
  )
}
```

---

## 7. Canonical URLs

Always set canonical to prevent duplicate content penalties (especially for paginated, filtered, or UTM-tagged URLs).

```typescript
// Static page
export const metadata: Metadata = {
  alternates: { canonical: '/blog' },
}

// Dynamic page
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  return {
    alternates: { canonical: `/blog/${params.slug}` },
  }
}

// Paginated page (/blog?page=2 → canonical to /blog)
export const metadata: Metadata = {
  alternates: { canonical: '/blog' },
}
```

---

## 8. Rendering Strategy Impact on SEO

| Strategy | When to Use | SEO Impact |
|---|---|---|
| Static (SSG) | Content rarely changes | Best — pre-rendered HTML at build time |
| ISR | Content updates periodically | Great — regenerates in background |
| SSR | Per-request dynamic content | Good — always fresh HTML |
| CSR | Dashboards, auth-gated pages | Poor — avoid for indexable pages |

For Next.js 15+ with `"use cache"`:

```typescript
export async function BlogList() {
  'use cache'
  cacheLife('hours')
  cacheTag('blog-list')

  const posts = await fetchPosts()
  return <PostGrid posts={posts} />
}
```

---

## 9. Core Web Vitals Targets

| Metric | Target | What Hurts It |
|---|---|---|
| LCP | < 2.5s | Unoptimized images, render-blocking JS |
| INP | < 200ms | Heavy JS on main thread, no Suspense |
| CLS | < 0.1 | Images without dimensions, dynamic inserts |

**Key fixes:**

```typescript
// Always specify width/height or use fill with a sized container
import Image from 'next/image'

<Image src="/hero.jpg" width={1200} height={630} alt="Hero" priority />

// Preload LCP image
export const metadata: Metadata = {
  other: {
    'link[rel=preload]': '/hero.jpg',
  },
}
```

---

## 10. `noindex` Patterns

```typescript
// Entire page
export const metadata: Metadata = {
  robots: { index: false, follow: false },
}

// Programmatic (e.g., draft posts)
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await fetchPost(params.slug)
  return {
    robots: post.published ? { index: true } : { index: false },
  }
}
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `next-seo` in App Router | Use built-in `Metadata` API only |
| Missing `metadataBase` | Required — relative image URLs break without it |
| `viewport` inside `metadata` object | Must be a separate `export const viewport` |
| Disallowing `/_next/` in robots | Never — crawlers need it to render JS/CSS |
| Mixing `metadata` export + `generateMetadata` | Use one per file, not both |
| Missing canonical on paginated/filtered URLs | Always set `alternates.canonical` |
| CSR for SEO-critical pages | Use SSG/SSR/ISR instead |
| No `alt` text on OG images | Required for accessibility + social cards |

---

## Deprecated Schema Types (as of 2025–2026)

Do NOT implement these — Google no longer shows rich results for them:

- `HowTo` — removed September 2023
- `FAQPage` rich results — removed May 2026 (still useful as entity signal, not rich result)
- `SpecialAnnouncement` — retired July 2025
- `ClaimReview`, `VehicleListing`, `EstimatedSalary` — retired June 2025

---

## Validation Tools

- **Google Search Console** — index status, Core Web Vitals, rich results
- **Rich Results Test** — https://search.google.com/test/rich-results
- **Schema Markup Validator** — https://validator.schema.org
- **OpenGraph Debugger** — https://developers.facebook.com/tools/debug/
- **Twitter Card Validator** — https://cards-dev.twitter.com/validator
- **PageSpeed Insights** — https://pagespeed.web.dev