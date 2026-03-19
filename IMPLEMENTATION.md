# IMPLEMENTATION.md

This landing page implies a product that offers custom QR codes, dynamic redirects, and analytics (scan counts, unique vs repeat, device, location, time series). Below is a concrete backend proposal, optimized for a small team while leaving room to scale.

## 1. Analytics (scan tracking)

### Approach

Log each scan as an immutable event, then aggregate for dashboards. Use a lightweight edge function for the redirect, which also writes a scan event to Postgres. Optionally maintain daily aggregates to keep dashboards fast.

### Stack Option (recommended)

- Edge runtime: Vercel Edge Functions or Cloudflare Workers
- DB: Postgres (Neon or Supabase)
- ORM: Prisma (or Drizzle)
- Optional: TimescaleDB extension if you want hypertables and continuous aggregates

### Event flow

1. QR slug requested (`/r/:slug`).
2. Edge function looks up destination.
3. Record scan event (server timestamp, user agent, device, geo, visitor id).
4. Redirect with `302` by default.

### Unique vs repeat

- Create a stable `visitor_id` cookie (UUID v4) on first scan.
- Unique scan for a QR = first time `visitor_id` sees that `qr_code_id` (derive in query, or write a `scan_uniques` table with `first_seen_at`).

### Device and location

- Parse user agent server-side (e.g. `ua-parser-js`), store `device_type`, `os`, `browser`.
- Use edge-provider geolocation: Cloudflare `cf.*` or Vercel `request.geo`.

### Schema (Postgres)

```sql
-- organizations / users for multi-tenancy
create table organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz not null default now()
);

create table users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  name text,
  created_at timestamptz not null default now()
);

create table memberships (
  user_id uuid references users(id) on delete cascade,
  organization_id uuid references organizations(id) on delete cascade,
  role text not null default 'owner',
  primary key (user_id, organization_id)
);

-- QR metadata / link mapping
create table qr_codes (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid references organizations(id) on delete cascade,
  name text not null,
  slug text unique not null,
  destination_url text not null,
  status text not null default 'active',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- immutable scan events
create table scan_events (
  id bigint generated always as identity primary key,
  qr_code_id uuid references qr_codes(id) on delete cascade,
  scanned_at timestamptz not null default now(),
  visitor_id uuid, -- from cookie
  ip_hash text, -- sha256(ip + salt), optional
  device_type text, -- mobile / desktop / tablet / bot
  os text,
  browser text,
  country text,
  region text,
  city text,
  referrer text
);

-- optional daily aggregate for faster dashboards
create table scan_daily (
  qr_code_id uuid references qr_codes(id) on delete cascade,
  day date not null,
  scans integer not null default 0,
  unique_scans integer not null default 0,
  primary key (qr_code_id, day)
);
```

### Aggregation logic

- On each scan, insert into `scan_events`.
- Optionally update `scan_daily` with an upsert.
- Unique scans can be computed by `count(distinct visitor_id)` for a range or via a `scan_uniques` table:

```sql
create table scan_uniques (
  qr_code_id uuid references qr_codes(id) on delete cascade,
  visitor_id uuid not null,
  first_seen_at timestamptz not null default now(),
  primary key (qr_code_id, visitor_id)
);
```

## 2. Dynamic QR / URL Redirects

### Behavior

- QR encodes a short URL: `https://qrf.ge/r/:slug`.
- Edge function resolves `slug` to destination URL, logs analytics, then redirects.
- `302` by default for flexibility; allow `301` per-link if customer needs permanent SEO semantics.

### Hosting

- Vercel Edge Functions or Cloudflare Workers. Both provide low latency and geolocation metadata.

### Schema for links

The `qr_codes` table above doubles as the link table. If you want explicit separation:

```sql
create table links (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid references organizations(id) on delete cascade,
  slug text unique not null,
  destination_url text not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  status text not null default 'active',
  redirect_mode text not null default '302'
);
```

### Edge handler (pseudo-TS)

```ts
export default async function handler(req: Request) {
  const url = new URL(req.url);
  const slug = url.pathname.split('/').pop();

  const link = await db.links.findUnique({ where: { slug } });
  if (!link || link.status !== 'active') {
    return new Response('Not found', { status: 404 });
  }

  const visitorId = getOrSetVisitorIdCookie(req);
  const ua = parseUserAgent(req.headers.get('user-agent'));
  const geo = req.geo || {};

  await db.scan_events.create({
    data: {
      link_id: link.id,
      visitor_id: visitorId,
      device_type: ua.deviceType,
      os: ua.os,
      browser: ua.browser,
      country: geo.country,
      region: geo.region,
      city: geo.city,
      referrer: req.headers.get('referer') || null,
    }
  });

  const status = link.redirect_mode === '301' ? 301 : 302;
  return Response.redirect(link.destination_url, status);
}
```

### Updating destination without changing QR

- QR encodes only the slug. Changing the `destination_url` in the DB updates the redirect immediately without reprinting QR codes.

## 3. QR Code Generation

### Recommendation

For styled QR codes with dots, gradients, and embedded logos, use `qr-code-styling` (client-side or server-side with node-canvas). It is widely used and supports logo embedding and custom shapes.

### Example (TypeScript)

```ts
import QRCodeStyling from 'qr-code-styling';

const qrCode = new QRCodeStyling({
  width: 1000,
  height: 1000,
  data: 'https://qrf.ge/r/summer-2025',
  margin: 16,
  image: 'https://cdn.qrf.ge/brand/logo.png',
  dotsOptions: {
    color: '#121212',
    type: 'rounded'
  },
  backgroundOptions: {
    color: '#F7F4EF'
  },
  imageOptions: {
    crossOrigin: 'anonymous',
    margin: 8
  }
});

// Browser:
qrCode.append(document.getElementById('qr'));
// Or download:
qrCode.download({ name: 'qrforge', extension: 'png' });
```

If you want deterministic server-side generation for email exports or PDFs, pair `qr-code-styling` with `node-canvas`, or use `qrcode` + `sharp` for post-processing and logo compositing.

## 4. Auth & Multi-tenancy

### Key points

- Users belong to organizations (teams). Use `memberships` with roles.
- Each QR code is owned by an organization.
- All data access is scoped by `organization_id`.

### Auth

- NextAuth (or Clerk) with email magic links or OAuth.
- `organizations` + `memberships` are enforced in the API and DB queries.
- Add RLS (row-level security) if using Supabase.

## 5. Recommended Stack (for solo dev / small team)

### Top pick

- Frontend: Next.js (App Router)
- Hosting: Vercel
- DB: Postgres (Neon or Supabase)
- ORM: Prisma
- Auth: NextAuth
- Edge redirects: Vercel Edge Functions
- Analytics: Postgres `scan_events` + optional daily aggregates
- File storage (logos / QR exports): S3-compatible (R2 or S3)

### Why this stack

- Fast to ship, minimal infra overhead.
- Edge functions enable low-latency redirects and built-in geo data.
- Postgres gives relational correctness and simple analytics without extra vendors.

