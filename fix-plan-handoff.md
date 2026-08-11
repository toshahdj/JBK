A full technical/SEO/GEO audit was run against production (jbkhokhani.com) on 11 Aug 2026: all 274 sitemap URLs crawled, plus DNS, TLS, headers, redirects, structured data and asset payloads. Below are the findings and the fixes. Work through them **in order** — the URL migration in Phase 1 changes the addresses everything else references, so doing content work first means doing it twice.

**How I want you to work:**
- One fix at a time. Show me the diff, tell me how to verify it, wait for me before moving to the next.
- Don't batch unrelated changes into one commit.
- Where a fix depends on server config I control (`.htaccess`, DNS), tell me exactly what to paste and where — don't guess at my hosting setup.
- If something in this list contradicts what you see in the codebase, **say so and stop** rather than forcing it. The audit was black-box against production; you can see the source and I trust your read over mine on implementation detail.
- Several items are mine to do, not yours (marked HUMAN). Skip those, but remind me they're outstanding.

---

## Decisions already made — don't re-litigate these

1. **Canonical host is the bare domain: `https://jbkhokhani.com`** (no `www`). Reason: it's what Google has actually indexed. The `www` host must keep resolving but 301 to bare.
2. **URLs carry a trailing slash: `/products/metformin/`.** Reason: pages deploy as directories (`/about/index.html`) and Apache's `DirectorySlash` already 301s to the slash form. We match the server rather than fight it.

---

## PHASE 1 — Critical. These four compound; nothing else matters until they're done.

### 1.1 — DNS migration [HUMAN — not a code task]
Authoritative DNS is failing. The domain delegates to four `orderbox-dns.com` nameservers (162.251.82.x) which return SERVFAIL/REFUSED.
- Cloudflare `1.1.1.1`: **SERVFAIL on both apex and www, 6/6 attempts.** "No Reachable Authority at delegation."
- AdGuard DNS: SERVFAIL on www.
- Google `8.8.8.8`: resolves, but NS took 2039 ms and MX took 3031 ms ("some clients may time out"); TXT queries return REFUSED, so SPF/DMARC are unreadable.
- No DS record — DNSSEC not enabled. MX points at Microsoft 365.

Anyone on 1.1.1.1 (a very common router/phone default, and every WARP device) cannot reach the site at all. I'm moving DNS to Cloudflare/Route 53 and enabling DNSSEC. **You can't fix or verify this — just flag if you see anything DNS-related in the repo.**

### 1.2 — Migrate every published URL to the canonical host + trailing slash
Currently **all 274 pages** emit `https://www.jbkhokhani.com/...` with **no** trailing slash. Find the base-URL constant / SEO helper and change it once. Every one of these needs updating:
- `<link rel="canonical">` (273 of 274 currently lack the trailing slash)
- `og:url`, `og:image`, `twitter:image`
- JSON-LD: `Organization.url`, `Organization.logo`, `Organization.image`, `WebSite.url`, `WebSite.potentialAction.target.urlTemplate`, `LocalBusiness.@id`, `LocalBusiness.url`, `LocalBusiness.logo/image`, `Product.url`, `BreadcrumbList.itemListElement[].item`, `WebPage.url`, `WebPage.isPartOf.url`
- `sitemap.xml` — all 274 `<loc>` entries
- `robots.txt` — the `Sitemap:` line
- `llms.txt` — all 9 "Key pages" links, the products/probiotics URL patterns, and the sitemap link
- **All internal `<a href>` values.** I sampled 17 page types and found **757 of 757** hrefs without a trailing slash. Every internal link on the site is currently a 301 hop.

Also confirm the React Router config still matches routes with the trailing slash (and ideally both forms) so client-side navigation doesn't break.

**Verify:** `curl -sI https://jbkhokhani.com/products/metformin/` returns 200 directly (not 301), and the canonical in that page's HTML is byte-identical to the URL requested.

### 1.3 — Redirects and HSTS [.htaccess — give me the block to paste]
`http://jbkhokhani.com/` currently returns **200 OK and serves the full site** — no redirect, no HSTS. Every page exists at four addresses (http/https × bare/www) with no canonicalisation. The result Google is actually showing is the `http://` one, with a stale title and an old "310+ APIs" claim that no longer exists on the live site.

Need: 301 http → https, 301 www → bare, then `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Order matters — HSTS goes on *after* 1.1 is confirmed working.

### 1.4 — Re-submit sitemap [HUMAN]
After the above deploys: re-submit in Search Console, watch the Pages report for the http/www URLs dropping out.

---

## PHASE 2 — High. On-page SEO.

Structurally this is already sound: 274 unique titles, 274 unique descriptions, 274 unique H1s, exactly one H1 per page, zero duplicates, correct 404 status codes, alt text on every image. The problems are calibration and depth.

### 2.1 — Meta descriptions: 240 pages are exactly 320 characters
Median across the site is 320 chars; 269 of 274 exceed 160. Google renders ~155 on desktop, less on mobile — so on a typical product page the snippet cuts off mid-boilerplate before any reason to click appears.

Cap the generator at ~155 and front-load the commercial hook. Suggested product template (lands ~140):
`{Name} API, CAS {cas}. IP/BP/EP/USP grades from a Mumbai supplier since 1965. CoA, MSDS and DMF support. Request a sample.`

### 2.2 — Titles: 143 of 274 exceed 60 characters
Cause is the 23-character suffix ` | J. B. Khokhani & Co.` on every page. Switch to ` | JBK` on **product, category and probiotic pages** (251 pages), keeping the full legal name on Home, About, Contact and Quality. Then fix the outliers by hand — the worst is 99 chars: "Complete Pharmaceutical Product Index 250+ APIs, Veterinary & Nutraceuticals" → something like "Full Product Index: 250+ APIs & Nutraceuticals". Nutraceutical product titles are 92–97 chars and also need shortening.

### 2.3 — `/products-index` is `noindex` but submitted in the sitemap
It's the only page of 274 carrying `noindex, follow`, and it sits in sitemap.xml at priority 0.7. Search Console reports this as an **error** ("Submitted URL marked 'noindex'"). It's also the best internal-linking page on the site, with 262 outbound links.

Ask me which I want: (a) drop from sitemap, keep `noindex, follow` — links still pass; or (b) remove the `noindex` and let a 656-word index of 250+ substances be indexed. Don't pick for me.

### 2.4 — Product schema is missing `image` — no product page is rich-result eligible
`Product` nodes carry `name`, `category`, `description`, `url`, `brand` and a good `additionalProperty` array with CAS numbers. `image` is **required** by Google for Product rich results and is absent on all 251. Also missing: `identifier`, `offers`, `sameAs`.

Underneath that: **product pages have no product imagery at all** — the only two `<img>` elements are the header and footer logos. That forfeits image search entirely.

Add one real image per product (molecular structure diagram, drum/packing photo, or a consistent branded card with name + CAS + grades), reference it from `image`, and add `identifier` as a `PropertyValue` with `propertyID: "CAS"` (keep `additionalProperty` — `identifier` is the field consumers actually read).

### 2.5 — The 404 page is Apache's raw default (355 bytes, unstyled)
Status code is correct — keep that. But there's no chrome, nav, search or branding. With 274 URLs, long chemical slugs prone to mistyping, and a trailing-slash migration that will strand some links, this leaks. Prerender a `/404/` page with site chrome, the product search box, and links to Products/Probiotics/Contact, then `ErrorDocument 404 /404/index.html`. Must still return a genuine 404 status.

---

## PHASE 3 — Medium. Delivery and performance.

The homepage pulls **693 KB compressed** across the assets its HTML references. Roughly two-thirds is logos and JS a brochure site doesn't need.

### 3.1 — 451 KB of logo across three files for one mark
| Asset | Role | Transferred |
|---|---|---|
| `2_without_background…png` | Footer lockup | **302.2 KB** |
| `1_without_background.optimized…png` | Header logo, renders 40px tall | **88.7 KB** |
| `1_without_background…webp` | *Same* header logo, preloaded `fetchpriority=high` | **60.2 KB** |

PNGs don't compress further over the wire — those are true sizes. Export both marks to AVIF with WebP fallback (expect ~15–25 KB footer, <8 KB header — roughly 420 KB saved), or ship SVG if the mark is vector in origin.

**Also:** the prerendered markup renders the `.png` while a `.webp` of the same logo is preloaded at high priority. Unless hydration swaps in exactly the preloaded file, **both download**. Verify in DevTools which file actually paints.

### 3.2 — JS bundle: 213 KB gzip / 670 KB parsed, on fully prerendered pages
Entry bundle `index-*.js` is 213.1 KB gzip, plus 20.7 KB CSS. The build carries **MUI *and* Radix UI** *and* Motion *and* Embla *and* canvas-confetti. Since every route is already prerendered to static HTML, all of it exists only to hydrate and animate content the visitor can already see — pure INP cost on the mid-range phone a procurement manager is holding.

Run `rollup-plugin-visualizer`. Start with the two overlapping component libraries — MUI and Radix do the same job. Lazy-load carousels, confetti and below-fold animation behind `IntersectionObserver`. Target <120 KB gzip entry chunk.

### 3.3 — Enable Brotli
Everything currently comes back `Content-Encoding: gzip` only. Brotli saves another 15–20% on JS/CSS — ~35 KB per first visit at current bundle size. `mod_brotli` + `AddOutputFilterByType BROTLI_COMPRESS` for HTML/CSS/JS/JSON/SVG/XML. **Caching is already correct — don't touch it** (hashed assets `max-age=31536000`, HTML `must-revalidate`).

### 3.4 — Remove the Lighthouse/PageSpeed user-agent exception
There's an inline script in `<head>` testing `navigator.userAgent` against `/lighthouse|headlesschrome|chrome-lighthouse|googlebot|bingbot|yandexbot|duckduckbot|baiduspider|twitterbot|facebookexternalhit|linkedinbot|whatsapp|slackbot|pingdom|gtmetrix|pagespeed/` and skipping the full-viewport intro curtain for matches. Real visitors get the curtain with a 6-second failsafe.

Two problems. The measurement one is bigger: lab scores structurally cannot see the worst rendering path, so Search Console field data will keep disagreeing with PageSpeed and we won't know why. The policy one is smaller but real: serving materially different rendering to Googlebot by user-agent is the pattern cloaking rules describe.

Drop the bot list and let the tools measure the real page. If the resulting Speed Index is bad, that's the honest signal to make the intro shorter/lighter — and worth asking whether a 6-second curtain earns its cost on a B2B sourcing site at all.

### 3.5 — Security headers (currently none at all)
No `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` or `CSP`. Not a ranking factor, but this is a site whose whole proposition is being a trustworthy supplier holding regulatory documentation, and client security teams do scan vendor headers. Add nosniff, `Referrer-Policy: strict-origin-when-cross-origin`, and a restrictive `Permissions-Policy`. **CSP last, report-only first** — Google Tag Manager will need care.

### 3.6 — Sitemap `lastmod` is the build clock
All 274 entries say the same date. When every page claims to change on every deploy, crawlers learn to ignore the field and we lose our only lever for signalling what actually changed. Emit `lastmod` from real per-page content modification.

---

## PHASE 4 — Answer engines (GEO/AEO). Strongest area, most headroom.

Already good and **don't regress**: llms.txt exists and is honest (it states plainly that the company is a distributor not a manufacturer, gives CAS coverage, and has a "Notes for answer engines" section flagging which figures are company-stated — that's rare and exactly right). robots.txt names GPTBot, ClaudeBot, PerplexityBot, CCBot, Google-Extended and Applebot-Extended explicitly. Structured data covers Product, BreadcrumbList, FAQPage, HowTo, Dataset, CollectionPage.

### 4.1 — llms.txt host fix + a factual inconsistency
Every URL in it points to `www`. Fix as part of 1.2. **While you're in the file:** it says the catalogue is 241 products, but site marketing says "250+". Reconcile — that exact kind of small discrepancy makes a model hedge instead of asserting.

### 4.2 — Consolidate the entity graph
Every page emits `Organization`, `WebSite`, `LocalBusiness` and `WebPage` as **separate script blocks**, and only `LocalBusiness` has an `@id`. Nothing references anything else, so a consumer sees four unrelated things sharing a name.

Give each a stable `@id` (`/#organization`, `/#website`, `/#localbusiness`, `{url}#webpage`) and wire them: `WebSite.publisher` → Organization; `LocalBusiness.parentOrganization` → Organization; `WebPage.isPartOf` → WebSite; `WebPage.about` → Product. Emit as a single `@graph` instead of five blocks. Add to `Organization`: `numberOfEmployees`, `slogan`, `knowsAbout` (therapeutic categories), and `award`/`hasCredential` for the Quality page certifications.

### 4.3 — Link CAS numbers to external substance identity ← **highest-leverage GEO change**
CAS is published for 233 of 241 products, and llms.txt correctly calls it "the reliable key for matching a product here to the same substance in another source" — but nothing in the markup makes that join for a machine. No `sameAs` to PubChem/DrugBank/ChEMBL/Wikidata, no UNII, no InChIKey.

PubChem's PUG-REST API resolves CAS → CID in one call. Run it once over the 233, store the mapping, then per product add `sameAs` (PubChem, DrugBank, Wikidata) and `identifier` entries for CAS, UNII, InChIKey. This converts 241 orphan pages into 241 pages attached to substances the models already know — so a question about metformin sourcing can reach us through the substance, not just the brand name.

### 4.4 — Add commercial facts (there are currently none)
No `offers` node, no MOQ, pack size, lead time or availability anywhere. B2B sourcing questions put to an AI assistant are almost always commercial ("who supplies X in EP grade, what's the minimum order, how fast to Y") and there's nothing here to answer with — so it answers from a directory instead.

Add a visible specifications block: grades, standard packing (e.g. 25 kg HDPE drums), indicative MOQ, typical lead time, "price on application, sample on request". Mirror in schema as an `Offer` with `availability`, `areaServed`, `seller`, `businessFunction`. **Omit `price` rather than inventing one** — an Offer without a price is valid; a wrong price is not.

### 4.5 — Expand the top 30–40 product pages
241 product pages average **183 words** on one shared template; 264 of 274 pages are under 300 words. Thinnest is `/products/sabam` at 142. Two consequences: at 241 near-identical pages this is the shape quality systems examine when deciding whether a catalogue is a genuine reference or a doorway set; and searching "paracetamol API supplier India" returns PharmaCompass, PharmaOffer, ensun and Chemcopia — directory pages with specs, DMF status and multiple suppliers, which a 245-word page cannot beat.

Don't do all 241. Take the 30–40 we actually want enquiries for and give each 400–600 words of genuinely product-specific content: pharmacopoeial grades and against which monograph; pack sizes and packing type; typical assay and key impurity limits; storage and shelf life; salts/hydrates available and how they differ; documents shipped with a consignment (CoA, MSDS, DMF, CEP); typical applications. Leave the long tail on the current template.

### 4.6 — Publish answer-shaped articles
Only two pages on the site are answer-shaped: the FAQ (1,392 words, proper `FAQPage` markup) and the creatine comparison (`Article` + `Dataset`, method disclosed). They're the two most likely to be cited by an AI answer, and there are no others.

The creatine comparison is the model — specific question, real measured data, disclosed method. Build 6–10 more from in-house expertise: IP vs BP vs EP vs USP for a formulator; what a CoA should contain and how to read one; DMF vs CEP vs ASMF; how to qualify an API supplier in India; vitamin K2-7 forms compared; spore-forming vs freeze-dried probiotics.

### 4.7 — Third-party profile cleanup [HUMAN]
Directories contradict our own site, and answer engines average them. IndiaMART/TradeIndia describe us as "exporter, **manufacturer** and supplier" with "3 branches at Bhiwandi, Baddi and Haridwar" and "310+ APIs" — none of which matches llms.txt. I need to claim and correct IndiaMART, TradeIndia, PharmaSource and RocketReach, and create a Google Business Profile (we have none).

**Your part:** `sameAs` currently lists six profiles (LinkedIn, Facebook, IndiaMART, Ingredients Network, iPHEX, CPHI) and omits TradeIndia, PharmaSource and RocketReach, which are all indexed and being read. Add them once I've corrected them, plus the Google Business Profile URL.

---

## Do not regress any of this

Verified working, and the fixes above touch this machinery:
- Full prerendering — all 274 routes ship as static HTML with content, headings and metadata in source
- 274 unique titles / descriptions / H1s, exactly one H1 per page, zero duplicates
- All 274 sitemap URLs return 200 — no orphans, no soft 404s
- Unknown paths return a **genuine 404 status** (the mistake almost every SPA makes)
- Zero images missing alt text, and the alt text is descriptive rather than stuffed
- Contact form: 7 labelled fields, correct input types, honeypot, visible skip-to-content link
- Consent Mode v2 with analytics and ad storage denied by default, 500 ms `wait_for_update`
- Asset caching: content-hashed filenames `max-age=31536000`, HTML `must-revalidate`

## Verification I'll want after each phase

```bash
# canonical host + no redirect hop
curl -sI https://jbkhokhani.com/products/metformin/ | head -1
curl -s  https://jbkhokhani.com/products/metformin/ | grep -o '<link rel="canonical"[^>]*>'

# http and www both 301 to canonical
curl -sI http://jbkhokhani.com/ | grep -i '^location'
curl -sI https://www.jbkhokhani.com/ | grep -i '^location'

# no internal link points at a redirect
curl -s https://jbkhokhani.com/products/metformin/ | grep -o 'href="/[^"]*"' | sort -u

# compression + headers
curl -sI -H 'Accept-Encoding: br,gzip' https://jbkhokhani.com/assets/index-*.js | grep -i 'content-encoding'
curl -sI https://jbkhokhani.com/ | grep -iE 'strict-transport|x-content-type|referrer-policy'
```

Start with 1.2 (I'm handling the DNS migration myself). Show me the diff before you commit.
