# Domain onboarding and Worker cutover

Use this reference when a website already works on Cloudflare and the user later obtains a domain.

## Phase 1: baseline and DNS inventory

Record the working `workers.dev` URL and test it before touching DNS. Resolve and save the current nameservers and all important records, especially MX, SPF, DKIM, DMARC, verification TXT records, and third-party subdomains. For a live domain, establish a rollback record before cutover.

Clarify the desired result:

- canonical apex: `https://example.com`
- canonical `www`: `https://www.example.com`
- application subdomain: `https://app.example.com`

A Worker Custom Domain is exact-hostname based. Choosing one does not automatically configure the others.

## Phase 2: move authoritative DNS to Cloudflare

1. Add the apex domain to the correct Cloudflare account. A normal Free or Pro onboarding is a full zone setup.
2. Review the DNS scan and manually restore anything missing before nameserver cutover.
3. If the old delegation uses DNSSEC, remove or disable the old DS record as directed by the registrar before replacing nameservers. Changing nameservers with stale DNSSEC can make the domain unreachable.
4. At the registrar, replace the existing authoritative nameservers with the two names assigned by Cloudflare, copied exactly.
5. Wait for the zone to become active. Independently check NS answers; cached public resolvers can lag.
6. After the Cloudflare zone is active, enable DNSSEC in Cloudflare if desired and publish the new DS information through the registrar when the setup requires it.

Changing nameservers may take time. Poll at sensible intervals and stop for user action rather than repeatedly resubmitting the same change.

## Phase 3: attach the Worker

Before attaching the hostname, remove only a conflicting web CNAME that blocks the Custom Domain. Do not remove unrelated records.

Dashboard path:

`Workers & Pages` → select Worker → `Settings` → `Domains & Routes` → `Add` → `Custom Domain`

Wrangler source configuration shape:

```jsonc
{
  "routes": [
    {
      "pattern": "example.com",
      "custom_domain": true
    }
  ]
}
```

Run the repository's normal build/deploy path afterward. When a framework build generates a deployment config, edit the source Wrangler file and confirm the generated output contains the route rather than editing generated files directly.

The application may also need its production URL changed. Search rather than guessing:

```sh
rg -n "workers\\.dev|BASE_URL|PUBLIC_SITE_URL|SITE_URL|ORIGIN|TRUSTED_ORIGINS|CORS|CALLBACK|COOKIE" .
```

Keep secrets out of the search output and logs. OAuth callback or webhook changes in third-party consoles are separate external mutations and require the appropriate user authorization.

## Phase 4: canonical host and validation

If the alternate hostname should redirect, use a Cloudflare redirect rule and a proxied originless DNS record as described in the current redirect documentation. If both hostnames should serve the Worker, declare both exact Custom Domains and ensure the application emits one canonical URL.

Acceptance checks:

- Cloudflare shows the zone active.
- Authoritative NS results match Cloudflare's assigned nameservers.
- The canonical hostname resolves and completes TLS without a certificate warning.
- The Worker serves expected pages, assets, APIs, and direct deep links on the custom domain.
- The alternate hostname serves or redirects exactly as intended, with no loop.
- Login/session cookies and OAuth callbacks use the new origin.
- Sitemap, feeds, Open Graph URLs, canonical tags, and robots references use the production domain where applicable.
- Existing mail and verification DNS records still resolve.
- The prior `workers.dev` endpoint remains available for diagnosis unless intentionally disabled.

## Rollback

If the Worker custom domain fails after the zone is active, remove or revert the application hostname change and custom-domain route while retaining the known-good `workers.dev` endpoint. If the nameserver migration itself fails, restore the registrar's previous nameservers and DNSSEC state from the recorded baseline. Explain that resolver caches may delay recovery.

## Current authoritative sources

- Full DNS zone setup: <https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/>
- Worker Custom Domains: <https://developers.cloudflare.com/workers/configuration/routing/custom-domains/>
- Redirect `www` to apex: <https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-www-to-root/>
- Redirect apex to `www`: <https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-root-to-www/>
- DNSSEC: <https://developers.cloudflare.com/dns/dnssec/>
- Universal SSL: <https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/>
