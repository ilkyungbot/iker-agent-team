# SEO — 검색엔진 최적화

> SEO는 기술이 아니라 사용자에게 유용한 콘텐츠를 기계가 이해할 수 있게 돕는 것이다.

## 핵심 원칙
- **메타데이터**: 각 페이지 고유의 title, description
- **시맨틱 HTML**: 구조화된 마크업으로 크롤러 이해도 향상
- **Core Web Vitals**: 구글 랭킹 요소 — LCP, INP, CLS
- **구조화 데이터**: JSON-LD로 리치 결과 지원

## 코드 예시

✅ Next.js Metadata API
```typescript
// app/products/[id]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata({
  params,
}: {
  params: { id: string }
}): Promise<Metadata> {
  const product = await getProduct(params.id)

  return {
    title: `${product.name} | 우리 서비스`,
    description: product.description.slice(0, 160),
    openGraph: {
      title: product.name,
      description: product.description,
      images: [{ url: product.imageUrl, width: 1200, height: 630 }],
      type: 'website',
    },
    twitter: {
      card: 'summary_large_image',
      title: product.name,
    },
    alternates: {
      canonical: `https://example.com/products/${params.id}`,
    },
  }
}
```

✅ 구조화 데이터 (JSON-LD)
```typescript
function ProductPage({ product }: { product: Product }) {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.imageUrl,
    offers: {
      '@type': 'Offer',
      price: product.price,
      priceCurrency: 'KRW',
      availability: product.inStock
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
    },
  }

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      {/* 페이지 콘텐츠 */}
    </>
  )
}
```

✅ sitemap.ts
```typescript
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const products = await getAllProducts()

  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1 },
    ...products.map((p) => ({
      url: `https://example.com/products/${p.id}`,
      lastModified: p.updatedAt,
      priority: 0.8,
    })),
  ]
}
```

✅ robots.ts
```typescript
// app/robots.ts
import type { MetadataRoute } from 'next'

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      { userAgent: '*', allow: '/', disallow: ['/api/', '/admin/'] },
    ],
    sitemap: 'https://example.com/sitemap.xml',
  }
}
```

## 실전 팁
- 각 페이지 title 60자 이내, description 160자 이내
- `next/image`로 이미지 alt 속성 필수
- h1은 페이지당 1개, 계층 구조 지키기 (h1 > h2 > h3)
- Google Search Console로 색인 상태 모니터링

## 주의사항
- CSR(클라이언트 사이드 렌더링) 콘텐츠는 SEO 불리 — SSR/SSG 검토
- 중복 title/description 금지 — 각 페이지 고유하게
- noindex 남발 금지 — 중요 페이지가 누락될 수 있음
