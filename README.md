# Scrape Tripadvisor reviews: how many can you get per run, why does the official API stop at 5, and what does 1,000 reviews cost? (fields, caps, credit math)

Tripadvisor's own Content API returns **a maximum of 5 reviews and 5 photos per location**, and new developers get roughly **5,000 free calls a month** only after putting billing details on file. That ceiling is deliberate. The API exists so approved partners can display Tripadvisor content inside their own product; it was never built to export a city's review history into a spreadsheet.

So if you need thousands of reviews, you're picking between three routes: a general-purpose scraping API you point at Tripadvisor URLs, a prebuilt scraper sold per record, or a script you maintain yourself. What actually separates them is below: which fields survive the trip, how many reviews one run can hold, the cost per 1,000 reviews, and where the personal-data line sits.

> **Start with credits, not a scraper subscription.** A general scraping API bills per request, so a 2,000-review sample fits inside ScraperAPI's free trial and you only start paying when the volume is real. 👉 [Check ScraperAPI's current plans and credit pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)

## All ScraperAPI plans, compared

ScraperAPI sells credits, not records. One credit is a standard page request, and the multiplier depends on what the target site throws at you. Tripadvisor is a heavy case: it blocks datacenter IPs aggressively and loads reviews with JavaScript, so a review page usually needs premium proxies plus rendering, which is **25 credits per request** rather than 1.

| Plan | Credits / month | Threads & geotargeting | Price | Sign up |
| --- | --- | --- | --- | --- |
| Free | 1,000 | 5 threads | $0 | [Open a free account](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 threads, US & EU | Shown on the pricing page | [See the Hobby price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 threads, US & EU | Shown on the pricing page | [See the Startup price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 threads, country-level | Shown on the pricing page | [See the Business price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 threads, country-level | **$475/mo, or $427.50/mo billed annually** ($5,130/yr) | [Start on Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 threads, country-level | Shown on the pricing page | [See the Professional price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 threads, country-level | Shown on the pricing page | [See the Advanced price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ threads, country-level | Custom | [Talk to sales via the pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons) |

One transparency note on that column: the only tier price I could confirm against the page today is Scaling at $475/month, discounted 10% on annual billing, which is the discount rate the billing table lists for every paid tier. ScraperAPI prints each plan's live number on the pricing page, and I'd rather send you there than repeat figures I couldn't verify.

A few mechanics that matter more than the headline credit counts:

- **Auto-upgrade** is opt-in on Hobby through Scaling, so running out mid-job can pause scraping instead of silently billing you more.
- **Pay-as-you-go** exists only on Scaling, Professional, Advanced and Enterprise, capped by a monthly limit you set.
- **Credits don't roll over** on any plan.
- **The free trial is 7 days with 5,000 credits**, no card. The permanent free plan is 1,000 credits a month with 5 threads.
- **There's a 7-day refund policy**, no questions asked.

## Where the official Tripadvisor API actually stops

Five numbers explain why people end up scraping at all. According to the 2026 write-ups of Tripadvisor's developer documentation:

- Review and photo endpoints return **up to 5 reviews and 5 photos per location**.
- New developers get around **5,000 free API calls per month**, with billing information required upfront.
- Usage runs against a **daily budget you configure in advance**; when it's hit, requests are blocked rather than throttled.
- Search endpoints are capped near **10,000 requests per day**.
- **One API key per account**, which is awkward the moment two projects or two teammates need separate quotas.

Run the arithmetic on a real project. Say you want 10,000 reviews across 200 Boston restaurants. At 5 reviews per location, the API caps you at 1,000 reviews no matter how many calls you have left. You'd hit the ceiling at 20% of the target and still be inside your monthly quota. Depth is the constraint, not volume.

## How many reviews can one run return?

This is where product-specific limits get mistaken for Tripadvisor limits. They aren't the same thing.

One prebuilt scraper documents a per-URL review limit of 0 (everything on the page) or 1–50, with **no extra pagination beyond the page itself**. Its runs keep page order, so the result set is a sample, not the newest reviews, and raising the single-page cap doesn't widen the sample because it never fetches the next page. The same tool takes up to 600 places from a city list (default 60) or 200 detail URLs per run, with 30 places per list page, deduplicated in page order.

A different scraper advertises **up to 500 reviews per listing, newest first**, default 50, with start URLs accepted in bulk and a language filter. Both statements are accurate about the products that make them. Neither is a Tripadvisor rule.

With a general scraping API the ceiling moves to your budget. You supply the URL, including the offset variants that public examples use to walk a listing (`-Reviews-or20-`), and you pay per request. That means full-listing coverage is possible, but it's also where blocked requests start costing money without returning rows.

## Fields: what comes back, and what quietly doesn't

If the review data is going into sentiment analysis or a reputation dashboard, field completeness decides whether the project works. A solid review record looks like this:

| Field | Reality check |
| --- | --- |
| `review_id`, `review_url` | Stable identifiers; keep them to dedupe repeat runs |
| `rating`, `title`, `text` | Rating out of five, headline, full body |
| `author`, `author_location`, `author_profile_url` | Reviewer identity, which is personal data under GDPR |
| `author_contributions` | Total contribution count for that reviewer |
| `helpful_votes` | **Hotel pages only.** Restaurant and attraction records won't have it |
| `stay_date`, `trip_type` | Visit month and travel party, in the site's own wording |
| `published_date` | At whatever granularity the page shows |
| `subratings` | Hotels: value, rooms, location, cleanliness, service, sleep quality. Restaurants: value, service, food, atmosphere. Empty when the reviewer skipped them |
| `owner_response`, `owner_response_from`, `owner_response_date` | The management reply, its signatory and its date |
| `photos` | Images attached to the review |
| `language` | Review language, in some outputs but not all |

Two gotchas that rarely make it into tutorials:

**Visit dates get faked into false precision.** When the page only shows a month, one scraper normalizes it to the first of that month, so "August 2025" arrives as `2025-08-01`. Build a monthly time series with that and every review looks like it happened on the 1st. Aggregate to month before charting.

**Subratings aren't comparable across business types.** A five on `food` and a five on `cleanliness` come from different questions on different forms. Don't average them into one "quality" score.

## What 1,000 reviews actually costs

Two pricing models are in play. Prebuilt scrapers charge per record; APIs charge per request, and the request cost swings with how hard the page is to fetch.

| Route | Stated rate | Approx. cost per 1,000 reviews | Assumption |
| --- | --- | --- | --- |
| Prebuilt scraper (published pricing) | $0.55 per 1,000 reviews + $0.001 start fee | $0.55 | $5 of monthly free usage ≈ 9,000 reviews |
| ScraperAPI, premium proxy + JS rendering | $475 per 5M credits (Scaling) | ≈ $0.24 | 25 credits per page, ~10 reviews per page |
| ScraperAPI, ultra-premium + rendering | $475 per 5M credits | ≈ $0.71 | 75 credits per page, ~10 reviews per page |

The 10-reviews-per-page figure comes from how paging examples step their offsets by 10. Your mileage will vary by listing and layout.

The same arithmetic tells you what the free tiers are worth in reviews rather than credits. The 7-day trial's 5,000 credits cover about **200 rendered pages, roughly 2,000 reviews**, at the premium-proxy rate. The permanent free plan's 1,000 credits cover about **40 pages, roughly 400 reviews** a month, or as few as 13 pages if a site forces ultra-premium.

Where that flips: if Tripadvisor rejects your requests repeatedly, each failed attempt still consumes the request budget, and the effective cost per 1,000 usable reviews climbs past the flat $0.55. Budget a retry allowance, and check the multiplier for your exact URL with the Domain Cost Estimator in the dashboard before you commit to a plan. You can also set `max_cost` per scrape so one expensive page can't burn a chunk of your allowance.

👉 [Compare ScraperAPI plans against your review volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The compliance part, briefly

Scraping publicly visible pages is generally treated as legal, but the data still falls under GDPR and CCPA. Reviewer names and profile links are personal data, so collect them only if you have a documented reason, and skip emails and phone numbers entirely. Don't republish review text or photos at length; copyright stays with the author.

## FAQ

**Do I need a Tripadvisor account or API key?**
Not for public pages. Scrapers work off the same URLs you'd open in a browser. The official API is the opposite: key required, use case approved.

**Can I get every review a place has ever received?**
Rarely in one pass. Prebuilt tools cap per listing (50 by default, 500 max on one, 50 on another) or return only what's on the page. API-based scraping can walk further, but you pay per page and hit bot protection at scale.

**Can I filter by language?**
Some scrapers accept a language code such as `en`, `fr` or `de`. Note that leaving it empty usually means English *including machine-translated reviews*, which is not the same as English-language originals.

**Do owner responses come back with dates?**
Yes in most review outputs, along with the signatory where the property names one.

**Why do helpful votes disappear for restaurants?**
The field is only populated on hotel pages. Plan your engagement metrics around hotels or drop the column.

## Which route fits your project

Match the tool to the shape of the job:

- **A few hundred reviews for a one-off analysis.** Use free credits and stop. ScraperAPI's trial covers roughly 2,000 reviews' worth of rendered pages, and a prebuilt scraper's $5 monthly allowance covers about 9,000 records. Whichever you pick, you shouldn't be paying a subscription.
- **A whole city's listings plus reviews, multiple languages.** Count pages, not plans. A 600-place list needs about 20 list pages plus one review page per place, so 5,000 credits gets you through discovery with room for review pages; Hobby's 100,000 credits covers that many times over. Concurrency only starts to matter past a few thousand pages.
- **Ongoing monitoring.** ScraperAPI's pricing page doesn't advertise a built-in scheduler, so plan on running it from cron or your own job runner and deduping on `review_id`. A scraper with native scheduling is less work here.

Two things to confirm before you enter a card: run the Domain Cost Estimator on an actual Tripadvisor review URL so you know whether it's a 25-credit or 75-credit page, and set `max_cost` so a retry loop can't drain the plan. Then the 7-day refund window gives you a real test rather than a guess.

👉 [Start with the 5,000-credit ScraperAPI trial and price your Tripadvisor volume](https://www.scraperapi.com/pricing?fp_ref=coupons)
