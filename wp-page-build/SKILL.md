---
name: wp-page-build
description: Design, build and publish a page into a WordPress/Elementor site over the REST API, then verify it landed correctly. Covers scoped HTML blocks that cannot leak into the theme, byte-level publish verification, expert and author bio pages with correct Person schema, and the claim-safety rules that stop unverified marketing claims going live. Use when asked to redesign, rebuild, optimise or publish a WordPress page, duplicate a product page for testing, fix Person or author schema on a team page, or push HTML into Elementor. Triggers on "redesign this page", "push it live", "update the page", "build this in WordPress", "optimise this bio page".
version: 1.0.0
category: Platform
subcategory: WordPress
user-invocable: true
argument-hint: "<page URL> [--draft] [--audit-only]"
license: Apache 2.0
---

# WordPress page build and publish

Build a page as a self-contained HTML block, push it into Elementor over the REST API, and prove it landed byte-for-byte. Every gotcha below came from a real build.

Use only on sites you administer, with the owner's credentials and permission.

## 1. Confirm access before designing

Authenticate with a **WordPress application password** over the REST API. Never put a password through `wp-login.php`.

```
GET /wp-json/wp/v2/users/me?context=edit
Authorization: Basic base64(user:app password)
```

Check `roles` includes `administrator`, and that these capabilities are true: `manage_options`, `edit_pages`, `publish_pages`, `upload_files`, `unfiltered_html` (required for `<style>` / `<script>` in content), `activate_plugins`.

A role label is not proof of write access. Prove it with a contained cycle — create a draft post, update it, force-delete it — and report the three status codes.

## 2. Writing to the site

**Some managed hosts block non-browser requests.** An authenticated request from a shell client can come back with a challenge or interstitial body rather than your JSON. That looks identical to a bad password and is not one. Detect it before concluding anything about credentials:

```js
const r = await fetch(url, {credentials: 'same-origin'});
const t = await r.text();
const challenged = !t.trim().startsWith('{') && !t.trim().startsWith('[');
```

Where that happens, drive the REST API from an authenticated browser context on the site's own origin, with `credentials:'same-origin'`. Using `credentials:'omit'` drops the session and re-triggers the challenge — an easy way to get a clean-looking result that means nothing.

### Endpoint differences that will cost you time

Products accept arbitrary meta through WooCommerce:

```
PUT /wp-json/wc/v3/products/<id>    body: { meta_data: [ {key, value}, … ] }
```

Pages do **not**. `wp/v2/pages` accepts only *registered* meta, and writing a value identical to the existing one returns `500 rest_meta_database_error`. Read current values first and send only keys that actually differ.

`_elementor_data` is a JSON **string** containing an array. One full-width container holding one HTML widget:

```js
[{ id:"x1", elType:"container", isInner:false,
   settings:{ content_width:"full",
              padding:{unit:"px",top:"0",right:"0",bottom:"0",left:"0",isLinked:true} },
   elements:[{ id:"x2", elType:"widget", widgetType:"html",
               settings:{ html: BLOCK }, elements:[] }] }]
```

Send alongside it: `_elementor_edit_mode:"builder"`, `_wp_page_template:"elementor_header_footer"`, `_elementor_page_settings:{hide_title:"yes"}`, and **empty** `_elementor_css` / `_elementor_page_assets`.

Never copy `_elementor_css`, `_elementor_page_assets` or `_elementor_controls_usage` from another post. They are per-post-ID caches; copying them points the new page at another page's stale CSS. Elementor regenerates them on render.

### SEO plugin meta

Rank Math's `rank_math_title` / `rank_math_description` are not REST-registered for pages. Use the plugin's own endpoint:

```
POST /wp-json/rankmath/v1/updateMeta
{ objectID:<id>, objectType:"post", meta:{ rank_math_title:"…", rank_math_description:"…" } }
```

On products they can also travel in the WooCommerce `meta_data` array.

## 3. Building the block

A single HTML widget renders exactly as designed but is not editable widget-by-widget in Elementor. State that trade-off to the client and offer a native rebuild once the design is signed off.

**Scope every selector** under one wrapper class (e.g. `.x-page`) so nothing leaks into the theme. Map `body` → the wrapper and `*` → `wrapper *`. Leave `:root` alone for custom properties.

Three failures worth knowing before you hit them:

- **Strip CSS comments before scoping.** A scoper that only recognises a comment at the exact cursor position will swallow `/* SECTION */` into the following selector, producing `.x-page /* SECTION */ :root{…}` and silently killing your entire token block. Every colour then falls back to the theme's.
- **Set `font-family` explicitly on headings.** Themes ship `h1,h2,h3{font-family:…}`, which beats inheritance from your wrapper. Relying on inheritance gives you the theme's face, not yours.
- **Drop dark-mode blocks for the live build.** A dark page body inside a light theme header and footer reads as broken.

Verify isolation by rendering the block inside a mock theme with deliberately clashing styles, and check both directions: theme chrome unchanged, and your block unaffected by it.

**Images:** use the sizes WordPress already generated (`-300x300`, `-600x600`, `-768x768`) with `srcset` and `sizes`. Serving a 1080px file into a 150px slot is a genuine page-experience failure. Watch format too — a 300×300 **PNG photograph** can run ~140KB where the WebP equivalent is ~5KB.

## 4. Verify, do not eyeball

After every push, compare the SHA-256 of what you sent with what reads back. WordPress can mangle slashes inside `_elementor_data`.

```js
const sha = async s => {
  const b = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s));
  return [...new Uint8Array(b)].map(x => x.toString(16).padStart(2,'0')).join('');
};
```

Then measure on the rendered page: element counts, horizontal scroll, overflowing elements, images loaded, and key strings both **present** and — for anything you removed — **absent**.

**Five false positives that will fool you:**

| Symptom | Real cause |
|---|---|
| Enormous document height, horizontal scroll, hundreds of overflowing elements | The viewport is collapsed. Check `document.documentElement.clientWidth` — if it is `0`, every measurement you just took is void |
| Page looks empty in screenshots | The renderer is not compositing, or `scroll-behavior:smooth` is not animating. Hide preceding sections to bring your target to scroll-zero instead of scrolling to it |
| A test page appears absent from shop, search or sitemap | The fetch was challenged and you parsed the interstitial. Assert the response is real HTML before trusting any absence |
| SEO tool reports multiple `<h1>` | The extension is reading a **logged-in** session. Duplicate-post and admin plugins inject admin-only overlays. Verify against raw served HTML and confirm no `wpadminbar` or `logged-in` body class |
| Elements sitting past the viewport edge | Off-screen carousel items inside their own `overflow-x:auto` track. Filter them out before calling it an overflow bug |

Report what you could **not** verify. If screenshots failed, say the checks were DOM-measured and ask for a visual confirmation.

## 5. Expert and author bio pages

The highest-impact fix on a team page is usually not the content.

**Check who the Person schema actually describes.** A page authored under an agency or ops account makes SEO plugins emit `Person.name = "ops@agency.com"` with a Gravatar — telling search engines the expert behind the page is your admin login. On an E-E-A-T page that is the single worst signal present. Fix it at the root: reassign the post author to the subject's own WP user (`{"author": <id>}`), then add an explicit `Person` JSON-LD for the page subject.

Include `jobTitle`, `worksFor`, `alumniOf`, `hasCredential`, `award`, `knowsAbout` and `sameAs`. `sameAs` is for URLs that **identify the person** — LinkedIn, Instagram, a personal site. A book's retail listing is not the person; link it in visible content instead.

Watch for content pasted straight from an intake document: ALL-CAPS field labels rendered as body copy (`EDUCATION::`), sections in template order rather than reading order (speaking engagements above the person's name), and meta descriptions auto-generating from the first line of pasted text.

**Layout:** never pair a fixed-width grid column with `white-space:nowrap`. A label longer than the column overflows and prints over the next column, and it will only show on your longest entries. Use `minmax(0, Npx)`, allow wrapping, and set `min-width:0` on both grid children. Verify by measuring `getBoundingClientRect()` gaps, not by looking at it.

## 6. Claim safety

You are publishing marketing claims for a business. Some categories are regulated — education and training, health, finance, legal.

- **Never republish a claim you cannot source.** If a page asserts a credential, an award, a refund term or a scope of practice, find it in the client's own material or ask. Do not carry it across from a sibling page and assume.
- **Read images before using them as proof.** A photo used as evidence of an outcome may not show what the caption implies — a certificate for a different qualification, or issued by a different provider. Open it and read it.
- **Refund, pricing and scope wording is compliance-sensitive.** Change it in **every** location at once. A page saying "full refund" in one place and "less a deposit" in another is worse than either alone.
- **Surface contradictions, do not silently pick.** When a client's own sources disagree — job titles, award years, who authored a book — say so and let them decide.
- **Flag any claim carried over from another page** as needing verification before launch.

For education clients specifically, treat scope-of-practice statements, "nationally recognised" wording, and anything implying a graduate outcome as claims requiring an explicit source.

## 7. Test pages

Duplicate first; never edit the live revenue page.

Copy the **raw** `post_content` from `wp/v2/<type>/<id>?context=edit`. The WooCommerce `description` field returns *filtered* HTML with shortcodes already expanded and plugin-injected links baked in — copying that produces a snapshot of rendered output, not a true duplicate.

Safeguard a published test page with `catalog_visibility:"hidden"` and noindex/nofollow robots meta, then **prove** it: fetch the shop, the category archives, site search and the sitemap, and confirm the test slug is absent while the original is present.

Give it a distinct title. A test page that inherits the original's title tag is a duplicate-meta problem from the moment it is published.

## 8. Where this applies

Categorised as **Platform › WordPress**. It works on any WordPress/Elementor site, but it was built out on **lead-gen** work in the **education and training** vertical, and those two contexts carry specifics worth knowing.

### Lead-gen sites

On a lead-gen site the high-value pages usually do **not** transact. Check the product type before you design:

- A WooCommerce product of `type: external` has **no cart** — it redirects to a form. If the page you are rebuilding is one of these, every CTA must point at that form, not at an on-page anchor.
- **Audit every CTA destination before launch.** Placeholder anchors like `href="#enrol"` render and style perfectly while doing nothing. A page can look finished and convert zero.
- **Confirm the form is the right one.** Check the destination form matches the product. A qualification page pointing at a generic enquiry form is a silent leak, and it is easy to inherit when duplicating a page.
- **Add a per-CTA tracking parameter** (`?cta=hero-guide`, `?cta=final-enrol`) so lead source is attributable per button. Use a neutral parameter name — `utm_*` will overwrite the visitor's real attribution.
- **Gated pricing is a strategy decision, not an oversight.** If pricing sits behind a form, do not un-gate it without asking. Offer it as a test.
- Where price is shown, show the payment plan honestly alongside it, including the total.

### Education and training

Anything implying a qualification, outcome or entitlement is a regulated claim:

- Scope-of-practice statements — what a graduate can and cannot legally do
- "Nationally recognised", accreditation numbers, awarding-body names
- Refund, deposit and cooling-off terms
- Who teaches or assesses the course, and what they are qualified in

Get each from the provider in writing. Do not infer them from a sibling page, and do not soften or sharpen the wording to improve the copy.

Turning a scope disclaimer into a plain "what you can do / what sits outside your scope" comparison usually improves both trust and conversion — the same facts, presented as confidence rather than as a legal warning.
