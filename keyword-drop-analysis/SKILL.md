---
name: keyword-drop-analysis
description: Investigate why a client's keyword dropped in rankings and recommend what to do about it. Confirms the drop in Ahrefs Rank Tracker (day-on-day), cross-checks it against Google Search Console data in SEO Gets, checks search volume trends, detects ranking-URL swaps and cannibalisation, diffs the SERP to see which competitors moved, checks the AI Overview (who Google cites) and the paid ads and Shopping results pushing organic down, compares their title, meta description, H1 and on-page content against ours, and ends with a short, prioritised fix list. Use when asked why a keyword dropped, lost rankings, fell out of the top 10, or when a client asks what happened to a term. Triggers on "keyword drop", "why did this keyword drop", "lost ranking", "ranking dropped", "keyword drop analysis", "dropped position", "what happened to our ranking".
license: Apache-2.0
metadata:
  version: "1.2.0"
  category: "SEO"
  subcategory: "Rank tracking"
---

# Keyword drop analysis

**Usage:** `/keyword-drop-analysis "<keyword>" <client domain> [--country AU] [--device mobile|desktop] [--date YYYY-MM-DD]`, or just ask "why did <keyword> drop for <client>?"

Two questions, in this order: **why did it drop**, and **what do we do about it**. Every conclusion in the report must point to a number from Ahrefs or GSC. If the evidence doesn't support a cause, don't claim it.

## 0. Inputs

Required: **keyword** and **client** (a domain, or a client name you can match to a domain).

Defaults if not given: country `AU`, both devices (report the one that dropped, and mobile first if both did), latest date vs the previous check.

If the client name is ambiguous, or matches more than one Ahrefs project or GSC property, ask. Don't guess.

## 1. Find the client in both tools

**Ahrefs.** `management-projects` → find the project whose target matches the client domain and has `has_keywords: true`. Then `management-project-keywords` for that project → confirm the keyword is tracked and get its `country`, `language_code` and `location_id`. One keyword can be tracked in several locations, such as Sydney and national. Analyse the location that dropped and name it in the report. Keep the `location_id` and `language_code`: `rank-tracker-serp-overview` refuses a city-tracked keyword without them ("keyword not tracked in the specified location").

**No Rank Tracker project?** Many clients only have GSC connected. If so, make GSC (SEO Gets) the primary source for steps 2–3. Use `get_site_performance` filtered to the query with `dimensions: ["date"]` for the position trend, and compare the last 28 days with the previous 28. Then use Ahrefs `site-explorer-organic-keywords` (`mode: subdomains`, `date_compared` about 30 days back, filtered to the keyword) for Ahrefs' view, and `serp-overview` (with `date` for a past snapshot) for the competitor SERP. Say in the report that there's no daily or weekly rank tracking behind the numbers. If no keyword was given, start by finding the non-branded queries with the biggest position or click loss over 28 days, and offer the user the top few.

**SEO Gets (GSC).** `list_sites` → match the property. Use the exact property string it returns. Domain properties and URL-prefix properties are not interchangeable.

Call `mcp__Ahrefs__doc` once per Ahrefs tool before first use in a session. The schemas are strict.

## 2. Confirm the drop is real

`rank-tracker-overview` with `date` = today and `date_compared` = yesterday, filtered to the keyword:

```
select: keyword,location,position,position_prev,position_diff,url,url_prev,
        best_position_kind,best_position_kind_previous,serp_features,serp_features_prev,
        target_positions_count,volume,keyword_difficulty,serp_updated,serp_updated_prev,
        keyword_is_frozen
where:  {"field":"keyword","is":["eq","<keyword>"]}
```

Run it for both `mobile` and `desktop`. Then run it again with `date_compared` set 7 and 28 days back. One day only tells you something moved. The longer windows tell you whether it's a blip, a slide or a step change.

**Check how often the project is tracked.** Many projects update weekly, not daily. If `serp_updated` and `serp_updated_prev` are the same timestamp, today and yesterday are the same reading. Compare the latest check with the one before it, and pull each weekly check over the last month so you can see the shape: one-off dip, bouncing, or steady slide.

**Don't trust `where` alone.** The keyword filter sometimes comes back ignored, returning every keyword in the project. Always pick out the target keyword's row yourself, and ignore the other rows in a filtered call (their `serp_updated` can read null).

Check these before you trust the numbers:

| What you see | What it really means |
|---|---|
| `serp_updated` equals `serp_updated_prev` | No new SERP check between the two dates, so the "drop" isn't a new reading. Widen the comparison |
| `keyword_is_frozen: true` | The keyword is over the plan limit and isn't being updated. Stop and tell the user |
| `position` is null | We fell out of the tracked range, or Ahrefs couldn't find us. Treat it as a big drop and check that the page is still live and indexable (step 5) |
| `best_position_kind_previous` was `snippet`, `local_pack` or `ai_overview_sitelink`, now `organic` | We lost a SERP feature, not necessarily organic ground. This is a different problem with a different fix |
| `serp_features` gained `ai_overview`, `local_pack`, `video` or `discussion` | The page may hold the same organic slot but sit lower on screen. Clicks fall even though position barely moves |
| `position` is null **and** `serp_features` is empty | When we aren't ranking, the overview also returns no SERP features, so you can't tell a real drop from a failed capture. Open the SERP itself (`rank-tracker-serp-overview`). If it has results, the drop is real |
| A move of 1–2 positions inside the top 10 | Normal daily fluctuation. Report it, recommend watching for 3–5 days, and don't build a big fix list on it |

A drop is worth a full investigation when it's **3+ positions inside the top 10**, **falling out of the top 3 or top 10**, **5+ positions outside the top 10**, or a **lost SERP feature**. Below that, do steps 2–4 only and say why you stopped.

## 3. Cross-check with Google Search Console (SEO Gets)

Rank Tracker is one SERP check from one location. GSC averages every real impression. They answer different questions, and you need both.

GSC data lags 2–3 days. Yesterday's drop usually isn't visible in GSC yet. Never query the last 36 hours and present it as complete.

1. **Query trend.** `get_site_performance` with `dimensions: ["date"]`, filtered to the query, metrics `clicks, impressions, ctr, position`, last 28 days vs the 28 days before.
2. **Which pages rank for it.** Same query filter with `dimensions: ["page"]`. Two or more pages sharing the impressions is **cannibalisation**, especially if the split changed recently. A URL tagged `?utm_source=google&utm_medium=organic&utm_campaign=gmb` (or similar) is the **Google Business Profile listing in the local pack**, not an organic page. Report it separately. A strong map-pack position can hide an organic drop in a date-only view, and the reverse is also true.
4. **Site-wide trend.** `dimensions: ["date"]`, `branded_queries: false`, no query filter. This tells you whether the whole site moved (algorithm or technical) or just this keyword.
3. **Device split.** `dimensions: ["device"]` if Ahrefs showed the drop on only one device.

Read them together:

| Ahrefs | GSC | Likely story |
|---|---|---|
| Dropped | Position stable, impressions stable | Local or one-off SERP variance. Watch it, don't act yet |
| Dropped | Position also worse | A real ranking loss. Carry on to steps 5–7 |
| Stable | Clicks down, impressions stable | CTR problem: SERP features, a competitor's snippet, or our title or meta being rewritten |
| Stable | Clicks and impressions down | Demand drop, not a ranking problem (step 4) |
| URL changed | Impressions split across pages | Cannibalisation or Google reassessing intent |

## 4. Search volume: has demand changed?

`keywords-explorer-volume-history` for the keyword and country over the last 24 months. Compare:

- this month vs last month
- this month vs the **same month last year**, which separates seasonality from a genuine decline
- GSC impressions over the same window, which reflect real demand for our footprint

Volume doesn't cause a ranking drop, but it explains a traffic drop. Keep the two separate in the report. "Position held; searches fell 40% seasonally, the same as last October" is a valid and useful finding.

## 5. Our side: did something change on our page?

1. **Ranking URL swap.** Compare `url` with `url_prev` from step 2, and check `target_positions_count`. A different URL now ranking, or two URLs ranking, points to cannibalisation or Google preferring a different page type for this intent.
2. **Content changes.** Automatic change detection only exists for SEO Gets **super sites**. On other sites this call returns only annotations and Google updates, so say the content history couldn't be checked. Don't report "no changes". `list_content_changes` for the property, `page_contains` set to our ranking URL's path, from 30 days before the drop up to today, with `include_text_diff: true`. Look for title, meta, H1 or heading edits, words removed, internal links removed, and HTTP status changes. Any edit in the week before the drop is the first suspect.
3. **Google updates.** Run the same call with `event_types: ["google_updates"]`. A drop that lines up with a core or spam update, and hits many keywords at once, is a site-level quality signal rather than a page-level one. Check whether other tracked keywords dropped on the same day (`rank-tracker-overview` with `order_by: position_diff:desc`).
4. **The page itself.** Fetch the live URL and confirm: a 200 status, no `noindex`, a canonical pointing to itself, and the main content present in the raw HTML (not only after JavaScript runs). A page that 404s, redirects or self-canonicalises elsewhere explains everything, so check it before analysing competitors.

   **Bot protection.** Cloudflare and other firewalls often block requests from cloud servers with a 403 "Attention Required" page, even with a normal browser user-agent. Don't spoof Googlebot. Cloudflare blocks fake Googlebots by IP, so a Googlebot user-agent proves nothing. If you're blocked:
   - **Is Google affected?** If GSC impressions and positions are still flowing, Google can crawl. Recommend a URL Inspection live test in GSC to confirm.
   - **Ahrefs `nr_words: 0`** on our URL while competitors show real counts usually means AhrefsBot is blocked too. That makes Ahrefs content data for the client unreliable.
   - **AI crawlers.** The same rule may block GPTBot, ClaudeBot and PerplexityBot. That's an AI-visibility problem worth raising even when Google is fine.
   - **On-page comparison.** Ask for the page HTML, a screenshot or the CMS content, or have the client allow verified bots. Put it under "Couldn't verify". Never guess at the content.

## 6. Competitors: who moved and why

**Diff the SERP.** `rank-tracker-serp-overview` for the keyword, with `project_id`, `country`, `device`, `top_positions: 20`, and `location_id`/`language_code` from step 1. Run it twice: once with no `date` (latest), and once with `date` set to the `serp_updated_prev` timestamp from step 2.

For every URL, line up the two snapshots:

- position then and now, and **new entrants** that weren't in the top 20 before. Also note who **fell out**: the space they freed explains part of any shuffle
- `domain_rating`, `url_rating`, `refdomains`, `nr_words`, `traffic`. **These are current values, not historical.** They come back identical for both dates, so they can't show that a competitor expanded content or gained links. They're only useful for comparing pages against each other today. To prove a competitor changed their page, use evidence from step 7 or `site-explorer-refdomains-history` for links
- `page_type`: is Google now favouring a different format, such as a guide, category page, service page or tool?
- `type`: who sits in the AI Overview (`ai_overview_sitelink`), snippet or local pack
- the `question` rows (People Also Ask) on both dates. Their wording is a direct read of intent. If they're all about price, the SERP wants price information
- a row with `url: null` in the organic list is a result Ahrefs couldn't identify. Note it and don't guess what it is

**Project competitors.** `rank-tracker-competitors-overview` with `date_compared`, filtered to the keyword, `select: keyword,competitors_list,serp_features,volume`. This shows how the competitors we track for this client moved on this term, even outside the top 20.

**Is it page-wide or just this keyword?** For each competitor that overtook us, run `site-explorer-organic-keywords` on their URL (`mode: exact`) with `date_compared` set about 30 days back. If they gained across many related keywords, they improved the page. If they gained on this one term only, it's more likely SERP churn.

## 6b. AI Overview and paid ads: what sits above organic

A #3 organic ranking can sit below an AI Overview, four ads and a Shopping carousel, and get almost no clicks. Check both every time, in the same SERP snapshots from step 6. Present them as the **latest captured SERP** with its `update_date`. No connected tool gives a real-time Google result, so never call it "live" without a timestamp.

**AI Overview (AIO)**
- Is there one? `type: ai_overview` in the snapshot, or `ai_overview` in `serp_features` from step 2. Compare with the previous snapshot: did the AIO **appear, disappear or stay**?
- **Who's cited?** Every `ai_overview_sitelink` row is a cited page. List them in order, then and now.
- **Are we cited?** Our domain among the `ai_overview_sitelink` rows (or `best_position_kind: ai_overview_sitelink` in step 2) means yes. Being cited while organic drops is a very different story from not being cited at all.
- **Who gained or lost citations** between snapshots, and are the cited pages also the ones that overtook us organically?
- **What gets cited:** open the cited pages (step 7) and look for what they share. That's usually a short, direct answer near the top, specific facts (prices, locations, timeframes), lists or tables, and clear entity signals such as business name, address and reviews.
- The Ahrefs Brand Radar AI Overview data (`brand-radar-ai-responses` with `data_source: google_ai_overviews_keywords`) needs an add-on that this account doesn't have. Don't call it; the SERP snapshot is the source. If it ever becomes available, pass `prompts: "ahrefs"` (it errors without a `report_id` otherwise).

**Paid ads and Shopping**
- Rows with `type` `paid_top`, `paid_bottom` or `paid_sitelink` are text ads, and `shopping` / `organic_shopping` is the Shopping carousel. Note **how many** ads sit above organic, then and now. More ads appearing pushes organic down the screen and cuts clicks even when position holds.
- **Who's advertising:** take the domain from each ad's `url` and the headline from its `title`. The URL's tracking parameters often reveal the keyword they're bidding on (for example `hsa_kw=custom%20made%20business%20suits`).
- **Competitive intel:** an ad headline like "Business Suits From $599" tells you the angle competitors pay to push, usually price, speed or proof. Use it when you rewrite our title and meta in step 9.
- **Is the client bidding?** If they're in the ads, check with the PPC team before recommending anything, because paid and organic should be read together.
- **Semrush as backup:** `keyword_research` → `phrase_adwords` / `phrase_adwords_historical` (database `au`), or `paid_search_research` → `resource_adwords` for a competitor domain. Semrush's Australian ad data is thin for local, low-volume terms. On a test it returned nothing for an advertiser that Ahrefs had just captured. Treat "nothing found" there as no data, not no ads.
- **If the user wants it truly live:** suggest a manual check in an incognito window from the target city, or the advertiser's page in the Google Ads Transparency Center.

## 7. On-page comparison: ours vs theirs

Fetch our ranking page plus **every competitor that moved above us** and the current **#1**. Cap it at five pages. Extract from the served HTML:

| Element | What to compare |
|---|---|
| Meta title | Keyword placement, length (about 50–60 characters), the angle (price, location, year, "best", "near me") |
| Meta description | Is it answering the intent? Does it have a reason to click? Is Google showing it or rewriting it? |
| H1 | Exact or close match to the keyword; does it match the searcher's intent? |
| H2/H3 outline | Subtopics they cover that we don't. This is usually the real gap |
| Word count and depth | Not length for its own sake. Are they answering more of the questions people ask? |
| Freshness | Visible updated date, current year, current pricing or regulations |
| Supporting content | FAQs, comparison tables, pricing, process steps, original images, video, reviews |
| E-E-A-T signals | Named author or expert, credentials, case studies, Australian business details |
| Schema | FAQPage, Service, Product, LocalBusiness, Review: what they have that we don't |
| Internal links | How many internal links point at the page, and with what anchor text |
| Links | `refdomains` and UR gap from step 6 |
| AI visibility | Using the findings from 6b: does their page answer the question in a clean, quotable passage near the top, which ours doesn't? |

Say plainly where **we're ahead** as well as behind. If our content is already stronger and the competitor won on links or brand, a content rewrite won't fix it, and the recommendation should say so.

## 8. Diagnosis

Pick the primary cause and up to two contributing causes. Rate each one **High / Medium / Low confidence** based on how directly the data supports it.

| Cause | Evidence that confirms it |
|---|---|
| Normal fluctuation | Small move, GSC stable, no SERP or page changes |
| Demand drop | Volume and GSC impressions down, position held |
| SERP layout change | New AI Overview, local pack or video block; position steady, clicks down |
| Lost the AI Overview | We were an `ai_overview_sitelink` before and aren't now, or competitors are cited and we aren't |
| Paid pressure | More ads or a new Shopping carousel above organic; position steady, CTR down in GSC |
| Lost SERP feature | `best_position_kind_previous` was a snippet or pack |
| Our page changed | An edit, status change or link removal in `list_content_changes` before the drop |
| Cannibalisation or URL swap | `url ≠ url_prev`, or impressions split across pages in GSC |
| Competitor improved content | Their `nr_words` or title changed, and they gained across related keywords |
| Competitor link growth | Their refdomains or UR now well ahead of ours |
| Intent shift | `page_type` across the top 10 changed (for example, guides replacing service pages) |
| Google update | The date matches an update and many keywords dropped at once |
| Technical issue | Non-200, noindex, wrong canonical, or content missing from the HTML |

## 9. Recommendations: what to do

Keep this short and specific. Three to five actions, most important first. For each action, give **what**, **why** (the evidence it fixes), **effort** (S/M/L) and **expected impact**.

Choose from:

- **Update metadata.** Write the new title and meta description in full. They must sound like a person wrote them, use Australian English, and not be stuffed with keywords.
- **Expand or restructure content.** List the exact missing sections as H2s, with one line on what each should cover. Don't just say "add more content".
- **Fix the H1 or heading structure** to match the intent the SERP is rewarding.
- **Resolve cannibalisation.** Name the page to keep and the page to merge, redirect, re-target or de-optimise.
- **Create a new page.** Only when the SERP clearly wants a different page type than the one we have, and say what that page type is.
- **Internal links.** Name the pages that should link in, with suggested anchor text.
- **Win back the SERP feature.** For a snippet, give a 40–60 word direct answer under a question-style H2. For an AI Overview, add a clear, citable summary with sources.
- **Get cited in the AI Overview.** Name the cited pages and what they do that ours doesn't. Then give the exact passage to add: a 40–80 word direct answer to the query near the top, plus the specific facts the AIO uses, and how we'll make the brand entity clear.
- **Respond to paid pressure.** If ads now dominate the SERP, say so plainly. Organic work alone may not win back the clicks. Brief the PPC team and borrow the winning ad angles for our title and meta.
- **Technical fix.** Exact issue and exact fix.
- **Links.** Only when the gap is the real cause. Give the size of the gap, not a generic "build links".
- **Wait and watch.** A legitimate recommendation for fluctuation. Set a recheck date.

If you write copy for the fix (titles, metas, intro paragraphs), follow the house voice: natural, plain-spoken, specific, no filler, nothing that reads as AI-generated.

## 10. Report format

```
## Keyword drop: "<keyword>" — <client domain>
<location> · <device> · <date> vs <comparison date>

**Summary:** One or two sentences. What happened, the main reason, and the single most important action.

### What changed
| | Before | Now |
| Position | | |
| Ranking URL | | |
| SERP features | | |
| GSC clicks / impressions / avg position (28d vs prior 28d) | | |
| Search volume (this month / same month last year) | | |

### Why it dropped
Primary cause (confidence) — the evidence.
Contributing causes (confidence) — the evidence.

### Who moved
| Competitor URL | Before → Now | What changed (title / words / links / format) |

### AI Overview and ads (latest captured SERP, <date>)
| | Before | Now |
| AI Overview shown? | | |
| Pages cited in AIO | | |
| Are we cited? | | |
| Ads above organic (count and advertisers) | | |
| Shopping carousel? | | |
Notable ad headlines and angles: …

### Ours vs theirs
The comparison table from step 7, limited to the rows that differ.

### What to do
1. Action — why — effort — impact
2. …

### Couldn't verify
Anything that was missing, stale or blocked.
```

## Rules

- **Evidence first.** No cause without a data point behind it. "Possibly" is fine when it's honest. A confident guess is not.
- **Separate ranking, demand and CTR.** Most "drops" a client notices are one of the three, and each needs a different fix.
- **Don't overreact to one day.** Daily rank tracking is noisy. Always show the 7-day and 28-day picture alongside the day-on-day change.
- **Units.** Ahrefs returns money values (`value`, `cost_per_click`) in USD cents. Divide by 100 and label them as USD.
- **Ahrefs rendering.** If a tool response includes `render_with`, call that render tool with the data before summarising.
- **Say what you couldn't check.** If a competitor page blocked the fetch, GSC hadn't caught up, or the keyword wasn't tracked, put it in "Couldn't verify". Don't leave silent gaps.
- **Stay read-only.** This skill diagnoses and recommends. Don't edit the client's site as part of it. Hand fixes to `wp-page-build` or the relevant workflow once the client or account lead agrees.
