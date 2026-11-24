# WordPress Headless CMS Setup

The target CMS hostname is `cms.sicksaint.club`; its hosting and DNS are not configured/confirmed in this repository. Public articles should remain on `https://sicksaint.club/article/[slug]`.

## Host and REST API

1. Provision WordPress hosting, point `cms.sicksaint.club` DNS to it, enable HTTPS, and set both WordPress Address and Site Address to that HTTPS origin.
2. Set Permalinks to **Post name**. Keep public articles published and REST API access enabled.
3. Confirm these public URLs return JSON: `/wp-json`, `/wp-json/wp/v2/posts?per_page=1&_embed=1`, `/wp-json/wp/v2/categories`, `/wp-json/wp/v2/tags`, `/wp-json/wp/v2/pages`.
4. Set `WORDPRESS_SITE_URL=https://cms.sicksaint.club` and `WORDPRESS_API_URL=https://cms.sicksaint.club/wp-json/wp/v2`. These are server-side variables; the public read API needs no credential.
5. Publish a real test article and verify it at the SICKSAINT article URL and in `/blog`.

## Pagination and cache behavior

The REST client requests `status=publish`, `_embed=true`, and WordPress pagination. It reads `X-WP-Total` and `X-WP-TotalPages`. `/api/articles` accepts `page` plus `per_page` (1–100); it does not fetch the entire archive on each request. Use the paginated `/blog` archive for the complete set. When WordPress is offline, article API returns `503` or empty UI data instead of presenting the sample archive as live content.

## Secure revalidation webhook

1. Generate a random secret of at least 32 characters and set it as `WORDPRESS_WEBHOOK_SECRET` in Vercel/local server configuration.
2. Configure a WordPress webhook for publish/update/trash events. Send `POST https://sicksaint.club/api/revalidate` with `x-webhook-secret: <secret>` and a JSON body containing `slug` and, when known, `category`.
3. Do not put the secret in the URL, frontend code, or a public plugin setting. Restrict webhook configuration access to trusted CMS administrators.
4. Confirm unauthorized/missing-secret requests return `401` and valid requests invalidate article, homepage, category, blog and sitemap paths.

## Contributor publishing

Contributor submissions remain in application storage and are not automatically published. The implementation still needs a server-only WordPress Application Password (or equivalent scoped publishing credential) to create **pending** posts after approval. An Editor/Admin must review and publish in WordPress. Do not give CMS credentials to contributors or expose them in browser code.
