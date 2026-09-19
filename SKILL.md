---
name: cloudflare-connect-domain
description: Onboard an owned domain to Cloudflare authoritative DNS and connect it to an already deployed Cloudflare Worker website. Use after a site works on workers.dev and the user has a registered domain; do not use for buying a domain or a first website deployment.
---

# Cloudflare Connect Domain

Move DNS authority deliberately, then attach the verified hostname to the existing Worker. Keep domain registration, DNS hosting, and website hosting conceptually separate: this workflow normally changes authoritative DNS, not the registrar of record.

## Establish scope and prerequisites

1. Confirm the user owns the registered domain and identify its registrar, current nameservers, current DNS provider, target Cloudflare account, deployed Worker, and working `workers.dev` URL.
2. Decide the canonical hostname: apex (`example.com`) or a subdomain such as `www.example.com`. Decide what the alternate hostname should do.
3. Inspect the application's source configuration for public base URLs, canonical URLs, cookie domains, trusted origins, CORS, OAuth callbacks, webhooks, sitemap/feed URLs, and environment-specific settings.
4. Read [references/domain-workflow.md](references/domain-workflow.md) before changing the zone, nameservers, DNS records, or Worker routes.

Do not proceed with cutover if the Worker is not independently healthy, the user cannot control the registrar, or the current DNS record inventory is incomplete.

## Preserve DNS before cutover

- Inventory the existing A, AAAA, CNAME, MX, TXT, CAA, SRV, verification, and mail-security records. Cloudflare's quick scan is a starting point, not proof of completeness.
- Preserve mail and third-party verification records exactly unless their provider requires a change. Web records may be replaced only when the new Worker custom domain makes them obsolete.
- If DNSSEC is active at the old provider or registrar, follow Cloudflare's current migration guidance. For a normal full setup, disable the old DS/DNSSEC delegation before changing nameservers, then re-enable DNSSEC in Cloudflare after the zone is active.
- If the domain was purchased through Cloudflare Registrar, do not invent a nameserver-switch step; it already uses Cloudflare authoritative DNS.

## Onboard the zone

Add the apex domain to the intended Cloudflare account, review imported records, select the intended plan, and obtain the two assigned Cloudflare nameservers. Nameserver changes occur at the registrar and may require the user to authenticate or confirm them manually.

After the registrar change, wait for Cloudflare to report the zone as active and verify authoritative nameservers independently. Do not attach the production hostname while the zone is still pending if that would create a confusing partial cutover.

## Attach the Worker custom domain

- Reconfirm that the hostname has no conflicting CNAME and belongs to the active Cloudflare zone.
- Prefer a Worker Custom Domain for an originless Worker site. In Wrangler source configuration, use an exact hostname with `custom_domain: true`; Custom Domains match exact hostnames, not wildcards.
- Update the application's production public/base/canonical URLs and any trusted-origin, cookie, OAuth, webhook, or CORS configuration that depends on the hostname. Update external OAuth/webhook consoles only with explicit user authorization and credentials.
- Deploy with the project's pinned Wrangler and existing deployment script. Cloudflare creates the Worker DNS record and certificate for a Custom Domain; do not add a competing manual CNAME.
- Configure the alternate `www` or apex hostname explicitly. Either attach both exact hostnames when both should serve the app, or create a proxied redirect to the canonical hostname using the current Cloudflare guidance.

Keep `workers.dev` enabled as a diagnostic fallback unless the user explicitly asks to disable it and the custom domain has passed verification.

## Verify and hand off

Verify Cloudflare zone status, authoritative NS, DNS answers, HTTPS certificate, canonical redirects, representative routes/assets, authentication and cookies, APIs, feeds/sitemaps, and any callback-dependent integration. Test both the canonical and alternate hostnames and check for redirect loops.

Finish with the zone/account, registrar, canonical hostname, Worker, nameservers, Custom Domains and redirects created, application settings updated, DNSSEC state, checks performed, propagation caveats, and any remaining manual steps. Never claim completion solely because the nameserver form was submitted.
