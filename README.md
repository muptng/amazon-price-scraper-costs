# Amazon Price Scraper: Build It Yourself or Buy One? A Field Checklist, Cost per 1,000 Results, and All Plans Side by Side

Pulling a price off one Amazon page is a weekend job: a headless Chrome instance, a three-second sleep, an XPath for `span.a-price`, and you have a CSV. The trouble starts the moment that script has to run daily across a few hundred ASINs and still report who holds the Buy Box, whether the item is in stock, and whether a coupon is stacking on top of the listed price. At that point you are not analysing prices any more. You are maintaining a scraper.

This article covers the three specific places a hand-rolled amazon price scraper breaks (with the selectors and headers that cause it), the fields that make price data usable rather than decorative, a method for converting any vendor's pricing into a cost per 1,000 results, and how ScraperAPI's plans and credit costs line up against that math.

> **Quick take:** If you need a one-off snapshot of a category page, a free Selenium script is enough and costs nothing. If you are running price monitoring or feeding a comparison tool, ScraperAPI bills Amazon requests at 5 credits each, so the Free tier's 1,000 monthly credits cover about 200 Amazon page pulls, enough to build and validate your parser before you pay. The 7-day trial adds 5,000 credits (roughly 1,000 Amazon page requests) and needs no card.
>
> 👉 [See ScraperAPI's current plans and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan on one screen

ScraperAPI sells eight tiers. The Free and Advanced tiers publish hard numbers; the five tiers between them are quoted on the pricing page with monthly and annual pricing, and those figures move with promotions and annual discounting, so the page is the only place they stay current. Treat the empty cells below as "read it on the day you buy", not as missing features.

| Plan | Credits per month | Concurrent threads | Price |
| --- | --- | --- | --- |
| [Free](https://www.scraperapi.com/pricing?fp_ref=coupons) | 1,000 | 5 | $0 |
| [Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) | Tiered by credit volume | — | Quoted on the pricing page |
| [Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) | Tiered by credit volume | — | Quoted on the pricing page |
| [Business](https://www.scraperapi.com/pricing?fp_ref=coupons) | Tiered by credit volume | — | Quoted on the pricing page |
| [Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) | Tiered by credit volume | — | Quoted on the pricing page |
| [Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) | Tiered by credit volume | — | Quoted on the pricing page |
| [Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) | 21,500,000 (plus a 500,000 bonus while the offer runs) | 500 | **$1,975/month**, or **$1,777.50/month** billed annually ($21,330/year) |
| [Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons) | Custom | Custom | Contact sales |

Two structural rules matter more than the list price. Auto-upgrade, which moves you to the next tier the moment credits run out, is available on Hobby, Startup, Business, Scaling and Professional. Pay-as-you-go, which lets you keep scraping past your limit at a fixed per-credit rate up to a cap you set, is available on Scaling, Professional, Advanced and Enterprise. Below those, running out of credits means scraping pauses until renewal.

### What a request actually costs in credits

Credits are not one-per-page. The multiplier is set by the target, and it is the number that decides your real bill:

| Request type | Credits |
| --- | --- |
| Standard page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google or Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Bot-protection bypass (Cloudflare, DataDome, PerimeterX and similar) | +10 |
| JS rendering (`render=true`) | +10 |
| Premium proxy (`premium=true`) | +10 |
| Screenshot | +10 |
| Geotargeting, sessions, device type, output format, auto parsing | 0 |

So 1,000 credits is 200 Amazon page requests, not 1,000. The Advanced tier's 21.5 million credits work out to 4.3 million Amazon page requests a month, which is why the price sits where it does. Every response also reports its own cost in the `sa-credit-cost` header, and you can set `max_cost` per scrape to stop a single URL from eating the allowance.

## Where a hand-rolled amazon price scraper breaks

The free Oxylabs script is a good stand-in for what most people write: Selenium with seleniumwire, headless Chrome, `driver.get()` followed by a hard `time.sleep(3)`, then XPath constants for the product cards. It outputs four columns: `title`, `url`, `price`, `currency`. It fails in three specific, fixable-in-principle places that most tutorials skip.

**The title XPath depends on a class string Amazon rotates.** The selector is `.//h2[@class='a-size-base-plus a-spacing-none a-color-base a-text-normal']/span`. Amazon uses several different class stacks on the same results page depending on placement, ad status and layout variant. When one placement changes, that selector returns nothing for those cards, and the per-product `except Exception: continue` wrapper swallows it. No error, no warning, just fewer rows than you expected in the CSV.

**The request headers are hardcoded to one marketplace.** The interceptor sets a Chrome 90 user agent on macOS 10.15.7, plus `Referer: https://www.amazon.com/` and `Host: www.amazon.com`. Point that at amazon.de and you get a redirect, a German layout your selectors do not match, or both. Every additional marketplace is a code change plus a full re-test of the XPath set.

**Out-of-stock items disappear from your dataset.** When `span.a-price` is missing, the parser raises `NoSuchElementException`, logs "Price not found for product {title}. Likely out of stock" and returns `None`. For a one-off export that is tidy housekeeping. For price history it is a hole: the product did not vanish, it went out of stock, and your time series now looks like a delisting. You need a separate availability field to tell those two events apart.

Add the missing proxy layer, the missing retry logic and the fact that each run handles exactly one URL, and you have a script that works until the day it quietly does not.

## Fields that make the data worth paying for

A price on its own answers almost nothing. Below is the field set from a production Amazon scraper, grouped by what each group decides.

**Price.** `price` is what is being charged now. `listPrice` is the strikethrough reference price Amazon displays, which nobody pays. `typicalPrice` is the 90-day norm, and it is the field that tells you whether today is genuinely cheap or just cheap against an inflated list. `savingsPercent` derives from `listPrice`, so it inherits the same problem. `unitPrice` matters when you are comparing pack sizes, and `usedPrice` matters when the new buy box is empty.

**Buy Box.** `buyBox` carries sold-by, ships-from, fulfilment method and an FBA flag. A low price from a third-party seller with no FBA and a three-week delivery window is not the same offer as the same price shipped Prime.

**Stock.** `inStock`, `inStockText`, `stockLeft` and `maxOrderQuantity`. The last one caps how much you can actually buy, which breaks naive arbitrage calculations.

**Ranks and badges.** `bestsellerRanks`, `isAmazonChoice`, `bestsellerBadge`, `dealBadge`. Category rank movement is often a better demand signal than review count.

**Promotions.** `coupon`, `promotions` and `subscribeAndSavePrice`. Skip these and your comparison will report a product as more expensive than the price a customer actually pays at checkout.

ScraperAPI's own structured Amazon endpoint returns a different shape but covers the same ground: a `product_information` block containing `asin`, `best_sellers_rank`, dimensions and attributes, alongside brand and description data, with the documentation describing prices, reviews, variants and metadata by ASIN.

## Cost per 1,000 results, done properly

Two units get confused constantly. Some tools bill per saved row, others per request. A request that fetches a search page can return a whole page of products, so the two prices are not comparable until you know the row yield per request.

Apify's Amazon scraper bills per result: **$3.50 per 1,000** on the Free plan, $3.15 on Starter, $2.80 on Scale, **$2.45 per 1,000** on Business and above, plus a $0.00005 per-GB actor start fee and platform usage on top. Its arithmetic is easy to run: 500 ASINs scraped daily for 30 days is 15,000 rows, about $52.50 on Free or $36.75 on Business. Only rows with status `OK` or `CARD` are billed; `NOT_FOUND` and `FAILED` rows are saved free, and switching on "save only changed products" removes unchanged rows from the bill entirely.

ScraperAPI bills per request, at 5 credits per Amazon page. On the Advanced tier that is 4.3 million Amazon page requests for $1,975, or roughly **$0.46 per 1,000 page requests** ($0.41 on annual billing). Whether that beats a per-row price depends entirely on how many products you extract from each page. One product page per ASIN and the comparison is direct; one search page yielding 48 products and the per-request model wins by a wide margin.

Three switches move the ScraperAPI bill: adding `render=true`, `premium=true` or `screenshot=true` each adds 10 credits per request, geotargeting and auto parsing add nothing, and go over your concurrency limit and you get a 429 rejection rather than an overage charge.

👉 [Check what your monthly Amazon request volume costs on each tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Three settings to lock before the first paid run

**Delivery location.** Amazon prices change with the ZIP or postcode you ship to, and by country. Apify exposes this directly as a setting (ZIP, postcode or country, plus page language and currency) and returns `locationText` in the output. ScraperAPI handles it through the `tld` and `country_code` parameters on the Amazon endpoint, which is how you request amazon.com priced for a Canadian address. Pick one location and keep it fixed. A series that switches ZIP halfway through the month is not a price series.

**Marketplace and currency.** Multi-marketplace support is not a default. One Apify actor covers amazon.com and amazon.co.jp only. ScraperAPI's documented `tld` list runs to 21 marketplaces, including co.uk, de, fr, it, es, jp, in, mx, br, au, ae, sa, se and pl. If your catalogue sits outside the US, check the list before you choose anything.

**Concurrency and delay.** The Free tier allows 5 concurrent threads, and exceeding that gets your request refused rather than queued. One Amazon scraper's own guidance recommends a minimum 1,000ms delay, and 1 to 3 concurrent requests with 2,000 to 5,000ms delays when you need stability over speed. If you plan to hit thousands of ASINs a day, concurrency is the thing you are actually buying, so size the tier on that rather than on credits alone.

## Is scraping Amazon prices legal?

Amazon's Terms of Service apply when you register an account and accept them. A scraper that never logs in and reads only pages any visitor can see arguably sits outside that agreement, which is the position vendors take when they say the restrictions "may not apply" to this kind of collection. Regional law is a separate question: data protection rules such as GDPR can apply depending on where you are and what you do with the data. Public pages and no login reduce the risk; they do not settle it. If your use case carries commercial weight, get a lawyer's read rather than a vendor's.

👉 [Review ScraperAPI's plans and terms before you commit](https://www.scraperapi.com/?fp_ref=coupons)

## FAQ

**Why does the scraped price differ from what I see in my browser?**
Almost always the delivery location or the currency in the session. The second most common cause is the offer itself: coupons and subscribe-and-save discounts are applied at checkout, so the listed `price` is higher than what a logged-in buyer pays. A third cause is comparing against `listPrice` rather than `price`.

**Why is there no price for out-of-stock products?**
Amazon removes the price block along with the buy box, so price selectors return nothing. That is why availability needs its own field. Without it, out-of-stock periods are indistinguishable from products being delisted.

**How often should I scrape?**
As often as prices in your category actually move, and no more. Work out the monthly request count first (ASINs × runs per day × 30, or search pages × pages per query × 30), then pick a tier that covers it with headroom. Daily is plenty for most categories; hourly is for flash deals and grocery.

**How far does the free tier go?**
1,000 credits a month, 5 concurrent threads, which is 200 Amazon page requests. The 7-day trial gives 5,000 credits, about 1,000 Amazon page requests, with no credit card required.

## Which one to pick

If you need one page of prices once, for a spreadsheet or a one-off analysis, write the Selenium script. Forty lines, no billing relationship, no account.

If you need daily tracking across a few hundred ASINs with Buy Box seller, stock status and coupon data attached, the script's maintenance cost passes the API bill within the first month you spend debugging a broken XPath. Run the request math before choosing a tier: a few hundred ASINs once a day is a few hundred Amazon requests, or 1,500 credits, which is a small plan. Search-page scraping is cheaper per product because each request carries a page of results.

If your catalogue spans more than the US marketplace and you want structured JSON per ASIN instead of raw HTML to parse yourself, check the `tld` and `country_code` support first, then confirm the concurrency you need against your daily volume.

👉 [Compare ScraperAPI's Amazon plans and start with the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)
