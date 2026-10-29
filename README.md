# YoYo Archive developer API

Read public Archive data as JSON. No API key required. This repository contains API documentation; server implementation lives in [The YoYo Archive](https://github.com/rkalajian/The-Yoyo-Archive).

**Status:** v1 implementation prepared; production availability requires deployment. Examples below use the intended production URL.

Base URL: `https://yoyoarchive.org/wp-json/yoyo-archive/v1/public`

## Quick start

```sh
curl 'https://yoyoarchive.org/wp-json/yoyo-archive/v1/public/items?type=product&per_page=10'
```

```js
const base = 'https://yoyoarchive.org/wp-json/yoyo-archive/v1/public';
async function get(path) {
  const response = await fetch(`${base}${path}`);
  if (!response.ok) throw new Error(`Archive API: ${response.status}`);
  return response.json();
}
const { items } = await get('/items?type=product&search=butterfly');
console.log(items.map(({ title, url, meta }) => ({ title, url, meta })));
```

## Endpoints

All endpoints accept `GET`. No public create, update, or delete endpoints exist. WordPress handles `OPTIONS` discovery. Omit credentials; the separate authenticated agent API has a different contract.

| Path | Response |
| --- | --- |
| `/` | Discovery, supported types, license, endpoint URLs |
| `/items` | Paginated records, default type `product` |
| `/items/{id}` | One record by positive integer WordPress ID |
| `/taxonomies` | Paginated public taxonomy terms |

### List records

| Query parameter | Default | Meaning |
| --- | --- | --- |
| `type` | `product` | `product`, `company`, `catalog`, `competition`, `article`, `story`, `review`, `profile`, or `yya_media` |
| `page` | `1` | Page number, 1–1,000,000 |
| `per_page` | `25` | Page size, 1–100 |
| `search` | — | WordPress text search, at most 200 characters |
| `slug` | — | Exact record slug, at most 200 characters |
| `taxonomy` | — | Taxonomy name, supplied together with `term` |
| `term` | — | Term slug, at most 200 characters; requires `taxonomy` |

Supported taxonomies: `yya_category`, `yya_material`, `yya_play_style`, `yya_media_type`. Filters combine with AND. Records sort by ID ascending. Unknown types, invalid parameters, and incomplete taxonomy/term pairs return 400.

```sh
curl 'https://yoyoarchive.org/wp-json/yoyo-archive/v1/public/items?type=product&taxonomy=yya_material&term=aluminum'
curl 'https://yoyoarchive.org/wp-json/yoyo-archive/v1/public/items?type=company&slug=duncan'
curl 'https://yoyoarchive.org/wp-json/yoyo-archive/v1/public/taxonomies?taxonomy=yya_material&per_page=100'
```

Term slugs vary: discover valid slugs with `/taxonomies` before filtering. Its `taxonomy` parameter is optional; `page` and `per_page` use the same defaults and limits as `/items`.

Collection responses contain `items`, `page`, `per_page`, `total`, and `total_pages`. HTTP headers `X-WP-Total` and `X-WP-TotalPages` also provide counts. Empty and out-of-range pages return empty `items` with accurate totals.

```json
{
  "items": [],
  "page": 1,
  "per_page": 25,
  "total": 0,
  "total_pages": 0
}
```

Fetch every page:

```js
async function* records(type) {
  for (let page = 1; ; page++) {
    const result = await get(`/items?${new URLSearchParams({ type, page, per_page: 100 })}`);
    yield* result.items;
    if (page >= result.total_pages) break;
  }
}
for await (const item of records('company')) console.log(item.title);
```

Pagination reads current data, without snapshot guarantees. Records may change during a crawl. Deduplicate by ID, cache locally, and periodically refresh; deleted or unpublished records disappear. This API does not provide deletion events or incremental synchronization.

### Record fields

| Field | Meaning |
| --- | --- |
| `id` | WordPress integer ID |
| `slug` | URL slug |
| `url` | Canonical Archive page |
| `type` | Singular content type |
| `title` | Decoded plain title |
| `date_modified` | UTC modification date, `YYYY-MM-DD` |
| `content` | Plain text extracted from static core blocks; dynamic blocks omitted |
| `meta` | Object of exportable registered `yya_*` fields; empty fields omitted |
| `terms` | Object mapping taxonomy names to term slug arrays |
| `relations` | Object mapping relationship keys to arrays of `{slug,url,title}` |
| `folio` | Optional positive Archive folio number |

Metadata follows the existing open dataset. Values retain their stored types; measurements are often strings. Missing fields mean unknown or unavailable, not zero. File/gallery metadata resolves to `{url,title,type}` entries. There is no featured-image field in v1. `date_modified` has day precision and is unsuitable as an incremental cursor. Always render text through safe text APIs such as `textContent`.

Only published records without passwords are returned, even when the caller is an administrator. Draft/private records and protected relationship targets are excluded. Internal metadata and email addresses are excluded/redacted. This is a public export, not a replacement for editorial access.

Taxonomy rows contain `taxonomy`, `slug`, `name`, `description`, and `parent` (parent slug or empty string). Only terms used by visible Archive records are returned.

### Errors

WordPress error bodies contain `code`, `message`, and `data.status`.

| Status | Meaning |
| --- | --- |
| `400` | Invalid parameter or incomplete filter |
| `404` | Record unavailable, unsupported, protected, or nonexistent; unsupported method/route |
| `401` / `403` | Host security/authentication policy blocked request |
| `429` / `5xx` | Host throttling or transient failure; retry with bounded exponential backoff |

No API-specific request quota is currently implemented. Keep requests sequential, cache results, and respect any hosting restrictions. WordPress handles CORS; hosting/security configuration may further restrict browser access.

## Attribution and media

Archive's existing dataset declares **[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)**. Attribute The YoYo Archive, link the source and license, indicate changes, and follow share-alike requirements where applicable. Example: “Data from [The YoYo Archive](https://yoyoarchive.org), CC BY-SA 4.0; modified.”

Image and PDF links do not grant reuse rights. Third-party photographs, scans, logos, and linked assets may have separate rights; verify permissions with their owners. This documentation does not change existing content licenses.

## Implementation and verification

Implementation: `wp-content/plugins/yoyoarchive-cpts/app/api/public.php`. Shared dataset serializer: `app/cli/DatasetExporter.php`.

Run on a disposable WordPress installation with the Archive plugin active:

```sh
wp eval-file wp-content/plugins/yoyoarchive-cpts/tests/public-api-selftest.php
```

The integration test creates temporary fixture records and terms, then removes its own fixtures. Do not use a production database for routine testing.
