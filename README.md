# ScraperAPI PHP Complete Guide: From Your First cURL Request to SDK Setup, JavaScript Rendering, Proxy Rotation, Credit Costs, and Full Plan Pricing Breakdown

If you ever tried to build a web scraper in PHP, you probably hit the same wall everyone hits. The first request works. The second one works. By the tenth, the site returns a CAPTCHA page, your IP gets rate-limited, and you're suddenly Googling "scraperapi php" at 2am trying to figure out why your Goutte script stopped returning product titles.

That's the moment when a managed scraping API stops looking like a luxury and starts looking like the obvious answer. This guide walks through everything around **scraperapi php** integration — how to send your first request, how to use the official SDK, how to handle JavaScript rendering and geo-targeting from inside a PHP codebase, how credits actually get charged, and which plan makes sense for the kind of scraping you're doing. I'll also unpack the pricing tiers in full so you can pick a plan without surprises on your first invoice.

If you'd rather just kick the tires first, ScraperAPI hands you **1,000 free API credits every month** plus a 7-day trial with 5,000 credits — no card required. 👉 [Start your free ScraperAPI trial here](https://www.scraperapi.com/?fp_ref=coupons).

---

## Why PHP Developers End Up at ScraperAPI

PHP isn't the first language people reach for when they think "web scraping." Python has Scrapy, Node has Cheerio and Puppeteer, and PHP has… a complicated reputation. But PHP still runs a huge slice of the web — WordPress alone is somewhere around 40% of all sites — and a lot of teams that need to scrape price data, SERPs, real estate listings, or competitor catalogs already have a PHP backend in production.

When you're already in PHP, spinning up a separate Python service just to scrape feels like overkill. The cleaner path is to handle the HTTP request from PHP, parse the HTML in PHP, and store the data in the same database your application already uses. The only piece missing is the proxy and anti-bot layer — and that's exactly what ScraperAPI replaces.

Instead of managing your own proxy pool, headless browser farm, CAPTCHA solver, and retry logic, you point your PHP HTTP client at a single endpoint:


https://api.scraperapi.com?api_key=API_KEY&url=TARGET_URL


ScraperAPI handles proxy rotation across 40 million+ IPs in 50+ countries, automatically retries failed requests, switches IPs and User-Agents when a CAPTCHA is detected, and optionally renders JavaScript with a headless browser. You get back the page HTML (or parsed JSON if you're hitting a structured data endpoint) and your PHP code stays clean.

---

## Three Ways to Use ScraperAPI From PHP

There's no single "right" way to call ScraperAPI from PHP. The right choice depends on what you already have installed, how much abstraction you want, and whether you're doing a one-off scrape or building a recurring pipeline. Here are the three common patterns.

### Method 1: Plain cURL (Zero Dependencies)

If you're on a shared host, a legacy CodeIgniter app, or just want to drop one file into a project without touching Composer, raw cURL is the path of least resistance. This is the example straight out of ScraperAPI's documentation:

php
<?php
// Replace the value for api_key with your actual API Key
$url = "https://api.scraperapi.com?api_key=API_KEY&url=https://example.com/";

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HEADER, false);

$response = curl_exec($ch);

if (curl_errno($ch)) {
    echo 'Curl error: ' . curl_error($ch);
} else {
    print_r($response);
}

curl_close($ch);


That's it. Eleven lines, no dependencies, works on any PHP install with the cURL extension (which is nearly all of them). The catch is that you're on your own for parsing the HTML — you'll need to feed `$response` into `DOMDocument`, Symfony's DomCrawler, or a library like Goutte to actually pull fields out of it.

### Method 2: Guzzle + Goutte (Recommended for Most Projects)

Guzzle is the de facto HTTP client in modern PHP, and Goutte (built by Symfony's creator Fabien Potencier) sits on top of it and gives you a fluent DOM crawler with CSS selectors. This is the combination ScraperAPI's own PHP tutorial walks through, and it's what I'd reach for in most real projects.

Install both libraries with Composer:

bash
composer require guzzlehttp/guzzle
composer require fabpot/goutte


Then route your Goutte request through ScraperAPI by simply replacing the target URL with the ScraperAPI endpoint URL:

php
<?php
require 'vendor/autoload.php';
use Goutte\Client;

$client = new Client();

// Point Goutte at ScraperAPI, with the real target URL inside the query string
$scraperApiUrl = 'https://api.scraperapi.com?api_key=API_KEY'
    . '&url=https://www.example.com/products'
    . '&country_code=us';

$crawler = $client->request('GET', $scraperApiUrl);

// Now use Goutte's CSS selector API like normal
$crawler->filter('.product-card')->each(function ($node) {
    $name  = $node->filter('.product-title')->text();
    $price = $node->filter('.product-price')->text();
    $link  = $node->filter('a')->attr('href');

    echo "$name — $price — $link\n";
});


The trick here is that Goutte doesn't know — and doesn't care — that the request is going through ScraperAPI. As far as your code is concerned, you're just scraping `example.com`. ScraperAPI is invisible: it handles the proxy, rotates the IP, returns the HTML, and Goutte parses it as if the request came straight from a browser.

### Method 3: The Official ScraperAPI PHP SDK

ScraperAPI ships a first-party PHP SDK on Packagist. If you're building a scraper that needs concurrent requests, retries, or you simply prefer a typed client over hand-rolled URLs, this is the cleanest option.

Install it:

bash
composer require scraperapi/sdk


Then use it like this:

php
<?php
require __DIR__ . '/vendor/autoload.php';
use ScraperAPI\Client;

$client = new Client("API_KEY");

// Plain request
$result = $client->get("https://example.com/")->raw_body;
echo $result;

// With JavaScript rendering enabled
$rendered = $client->get("https://example.com/", ["render" => true])->raw_body;

// With render + premium proxy + US geo
$premium = $client->get(
    "https://example.com/",
    ["render" => true, "premium" => true, "country_code" => "us"]
)->raw_body;


The SDK accepts an optional `params` array as the second argument to `get()`, which maps directly to ScraperAPI's query parameters — `render`, `premium`, `ultra_premium`, `country_code`, `session_number`, `device_type`, and so on. It also runs on top of Guzzle internally, so it plays nicely with the rest of a modern PHP stack.

> The SDK is the cleanest abstraction, but it's not mandatory. cURL works everywhere, and Guzzle+Goutte gives you a DOM crawler for free. Pick whichever fits your project's existing dependencies.

---

## JavaScript Rendering, Geo-Targeting, and Sessions From PHP

The reason most scrapers fail isn't HTTP — it's that modern sites render content with JavaScript, geo-fence their content, or fingerprint your session. ScraperAPI's parameters solve each of those from a single query string, and they all work identically from PHP.

**JavaScript rendering.** Add `render=true` and ScraperAPI fires up a headless browser on its side, waits for the page to finish loading, and returns the fully rendered HTML. From PHP it's just one extra parameter:

php
$url = "https://api.scraperapi.com?api_key=API_KEY&render=true&url=https://example.com/";


**Geo-targeting.** Pass `country_code=us`, `country_code=de`, etc. to send the request through a residential proxy in that country. This is the fix for sites that show different content based on the visitor's location — Amazon, NewChic, Zillow, anything with regional pricing. Below the Business plan you're limited to US and EU proxies; country-level targeting unlocks at Business and above.

**Session persistence.** Pass `session_number=123` to pin all subsequent requests to the same IP for up to 15 minutes of inactivity. Useful when you need to log in (via cookies you set on the request) or scrape a paginated list without each page coming from a different IP and breaking the site's session.

**Premium and ultra-premium proxies.** `premium=true` switches to ScraperAPI's residential pool; `ultra_premium=true` switches to the highest-quality pool for the hardest targets (LinkedIn, sites behind Cloudflare Turnstile / DataDome / PerimeterX). Use these only when a standard request fails — they cost more credits.

One important rule from ScraperAPI's docs: always put ScraperAPI's own parameters **before** the `url=` parameter in the query string, so they don't get mixed up with parameters that already exist on the target URL.

---

## How ScraperAPI Credits Actually Get Charged

This is the part that catches almost everyone off guard. The headline on the pricing page says "100,000 API credits" for the Hobby plan, and most people read that as "100,000 requests." It's not. The number of credits a single request costs depends on what you're scraping and which features you've enabled.

**Base cost by domain type:**

| Domain category | Credits per request | Examples |
| --- | --- | --- |
| Normal websites | 1 | Blogs, news sites, simple HTML |
| E-commerce (Amazon, Walmart) | 5 | Product pages, search results |
| SERP (Google, Bing) | 25 | Search engine result pages |
| LinkedIn | 30 | Public LinkedIn pages |

**Extra credits for features:**

| Parameter | Extra credits |
| --- | --- |
| `render=true` (JS rendering) | +10 |
| `premium=true` (residential proxy) | +10 |
| `premium=true` + `render=true` combined | **+25** (not +20) |
| `ultra_premium=true` | +30 |
| `ultra_premium=true` + `render=true` combined | **+75** (not +40) |
| Anti-bot bypass (Cloudflare, DataDome, PerimeterX) | +10, auto-applied |

Notice the non-linear stacking: combining `premium` with `render` costs 25 credits, not 20. Combining `ultra_premium` with `render` costs 75, not 40. That's the single biggest source of "why did my credits disappear so fast?" complaints, and it's worth internalizing before you write a single line of PHP.

Parameters that cost **zero** extra credits: `country_code`, `session_number`, `device_type`, `wait_for_selector`, `output_format`, `keep_headers`, `autoparse`. Use those freely.

A practical worked example: a Hobby plan ($49/month, 100,000 credits) scraping Amazon product pages without rendering burns 5 credits per request — that's 20,000 product pulls per month. The same plan scraping Google SERPs (25 credits each) gets you 4,000 SERP requests. The same plan scraping a JS-rendered LinkedIn page (30 + 10 = 40 credits) gets you 2,500 requests. Same plan, very different effective capacity.

ScraperAPI exposes a `sa-credit-cost` response header on every request showing exactly how many credits that specific request cost, and a `urlcost` API endpoint you can call to price a URL before you scrape it at scale. If you're building a production pipeline in PHP, hitting `urlcost` once for each new target is a cheap way to avoid surprises.

---

## Full ScraperAPI Plan Comparison

ScraperAPI runs a seven-tier credit ladder (six self-serve plans plus a custom Enterprise tier). Every paid plan includes all core features — JS rendering, premium proxies, structured data endpoints, rotating proxy pools, CAPTCHA handling, unlimited bandwidth — the differences are credit volume, concurrent threads, geotargeting scope, and whether you get pay-as-you-go overage when you run out.

Annual billing takes 10% off every paid plan. New accounts get a 7-day trial with 5,000 free credits on top of the permanent 1,000-credit free tier.

| Plan | Monthly Price | Annual (billed yearly) | API Credits / Month | Concurrent Threads | Geotargeting | Pay-As-You-Go Overage | Get Started |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Free** | $0 | — | 1,000 | 5 | None | No |  [Start free](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49 / mo | $44.10 / mo | 100,000 | 20 | US & EU only | No (must upgrade) |  [Get Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | $149 / mo | $134.10 / mo | 1,000,000 | 50 | US & EU only | No (must upgrade) |  [Get Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | $299 / mo | $269.10 / mo | 3,000,000 | 100 | Country-level (50+ countries) | No (must upgrade) |  [Get Business](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** | $475 / mo | $427.50 / mo | 5,000,000 | 200 | Country-level | Yes |  [Get Scaling](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** | $975 / mo | $877.50 / mo | 10,500,000 | 300 | Country-level | Yes |  [Get Professional](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** | $1,975 / mo | $1,777.50 / mo | 21,500,000 | 500 | Country-level | Yes |  [Get Advanced](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22,000,000+ | 500+ | Country-level + dedicated | Yes, with dedicated support |  [Contact Sales](https://www.scraperapi.com/?fp_ref=coupons) |

A few notes that don't fit neatly into a table:

- **Concurrency is a separate governor from credits.** Two customers with the same credit pool can have very different throughput ceilings. A Hobby plan with 20 concurrent threads can't run more than 20 requests in parallel, even if it has credits to spare. If you're building a PHP queue worker with 50 parallel Guzzle promises, you'll bottleneck on the thread cap before you bottleneck on credits — and that's a deliberate reason to move up a tier.

- **Pay-As-You-Go only unlocks at Scaling and above.** On Hobby, Startup, and Business, exhausting your credits mid-cycle means you're cut off until renewal or you upgrade. On Scaling / Professional / Advanced / Enterprise, a pop-up lets you keep going at a fixed per-credit rate, with a configurable monthly spend cap so you don't get a surprise bill.

- **Credits don't roll over.** Whatever you don't use by the end of the billing cycle is gone. Build your scraping schedule around that, not around stockpiling.

- **Geotargeting outside the US and EU requires Business ($299/mo) or higher.** If you need to scrape German Amazon from a German IP, or Japanese Rakuten from a Japanese IP, you need to be on Business or above.

- **Structured Data Endpoints** (Amazon, Google SERP, Walmart, eBay, Redfin) are available on every plan including Free, and they return parsed JSON instead of raw HTML. For Amazon these endpoints cost 5 credits per request and return 18+ fields per product; for Google SERP they cost 25 credits and return organic results, ads, featured snippets, and People Also Ask.

---

## Picking the Right Plan for a PHP Project

The right plan depends less on your PHP code and more on what you're scraping. Here's how I'd map it out:

**Free plan — prototyping only.** Use the 1,000 monthly credits to validate that ScraperAPI can actually scrape your target site. Test success rates, measure latency, and figure out which parameters you need (`render`, `premium`, `country_code`) before you commit to paying anything.

**Hobby ($49) — single-site scraping at low volume.** A side project that pulls 5,000–20,000 pages a month from a single well-behaved target. Don't pick Hobby if you need JS rendering on every request — 10 credits per render means your 100K credits turn into 10K rendered pages fast.

**Startup ($149) — small production pipeline.** 1M credits, 50 concurrent threads. Enough headroom for a recurring nightly scrape of a few thousand product pages or a few hundred SERP queries per day. Still US/EU only.

**Business ($299) — first "real" production tier.** 3M credits, 100 threads, and crucially **country-level geotargeting** in 50+ countries. This is where most production PHP scrapers should land. If you're scraping Amazon US, Google SERPs across regions, or any geo-fenced site, this is the entry point.

**Scaling ($475) — the "Most Popular" tier, and the first with PAYG.** 5M credits, 200 threads, pay-as-you-go overage. If your scraping volume is bursty and unpredictable (e.g., a competitive price-monitoring tool that spikes during sales events), PAYG is the safety net that keeps your scraper from going dark mid-cycle.

**Professional ($975) and Advanced ($1,975) — high-volume, multi-source pipelines.** 10.5M and 21.5M credits respectively, 300 and 500 threads, priority support and priority routing. These are for teams running continuous data collection across many domains — think SEO agencies tracking thousands of keywords, or e-commerce ops scraping every competitor every hour.

**Enterprise — custom.** 22M+ credits, dedicated support via Slack, custom pricing. Sales-led. Reach out when you're consistently hitting the Advanced ceiling.

If you want to test all of this on your actual targets before committing, 👉 [start with the free trial here](https://www.scraperapi.com/?fp_ref=coupons) — you get 5,000 credits for 7 days and 1,000 credits per month forever after, with no card on file.

---

## A Realistic PHP Scraping Example, End to End

Let's tie it together with a complete example: scraping a paginated product catalog with JavaScript rendering, US geotargeting, and CSV export. This is a pattern I've actually used in production PHP projects.

php
<?php
require 'vendor/autoload.php';

use Goutte\Client;
use ScraperAPI\Client as ScraperApiClient;

$apiKey = getenv('SCRAPERAPI_KEY');
$target = 'https://www.example.com/products?page=';

$scraperClient = new ScraperApiClient($apiKey);
$goutte = new Client();

$csv = fopen('products.csv', 'w');
fputcsv($csv, ['Name', 'Price', 'URL']);

for ($page = 1; $page <= 20; $page++) {
    // Route through ScraperAPI with JS rendering + US geo
    $apiUrl = 'https://api.scraperapi.com?api_key=' . $apiKey
            . '&render=true&country_code=us'
            . '&url=' . urlencode($target . $page);

    $crawler = $goutte->request('GET', $apiUrl);

    $crawler->filter('.product-card')->each(function ($node) use ($csv) {
        $name  = $node->filter('.product-title')->text();
        $price = $node->filter('.product-price')->text();
        $link  = $node->filter('a')->attr('href');
        fputcsv($csv, [$name, $price, $link]);
    });

    // Be polite — 1 second between page requests
    sleep(1);
}

fclose($csv);
echo "Done.\n";


A few things worth pointing out in this example:

- The API key is read from an environment variable, not hardcoded. Never commit your ScraperAPI key to git.
- `render=true` and `country_code=us` are placed **before** `url=` in the query string, per ScraperAPI's documentation rule.
- The target URL is URL-encoded with `urlencode()` so its own query string (`?page=N`) doesn't collide with ScraperAPI's parameters.
- A `sleep(1)` between pages is more than just politeness — it also keeps you well under your concurrent-thread cap and avoids burning through credits faster than you need to.
- Each request here costs 11 credits (10 for `render=true` + 1 base for a normal website), so 20 pages costs 220 credits — comfortably inside even the Free tier.

---

## What People Actually Say About ScraperAPI

Aggregated reviews from third-party platforms paint a fairly consistent picture:

| Platform | Rating | Review count |
| --- | --- | --- |
| G2 | 4.4 / 5 | 16 |
| Capterra | 4.6 / 5 | 62 |
| Trustpilot | 4.5 / 5 | 43 |

Capterra's sub-ratings put **Ease of Use at 4.9/5** and Customer Service at 4.6/5, which lines up with the general sentiment that ScraperAPI's setup experience is genuinely one of the smoother ones in the scraping API category. Most positive reviews mention how fast you can go from signing up to a working request — typically under 10 minutes if you're comfortable with HTTP.

The complaints cluster around two themes. First, the credit multiplier system catches people off guard — multiple reviews mention signing up expecting 100,000 requests and finding that 100,000 credits gets them a fraction of that once rendering or premium proxies are enabled. Second, on harder targets (LinkedIn at 30 credits, Google SERPs at 25) costs add up faster than expected. Both of those are solved by understanding the multiplier table above before you commit to a plan.

Independent benchmark data from Scrapeway shows ScraperAPI performing strongly on e-commerce and real estate — 98% success on Amazon, 100% on Zillow, 99% on Etsy — and struggling on social media (0% success on Instagram, X, and Booking.com). It's explicitly not designed to scrape behind logins; for that, you need a browser-based tool, not an API.

---

## Gotchas Worth Knowing Before You Build

A handful of things that bit me or that I've seen bite other PHP developers:

- **404 responses consume credits.** ScraperAPI charges for any 2xx or 4xx response, not just successful ones. Validate your URLs before you scrape.
- **Cancelled requests are charged if they pass the 70-second processing window.** If you set a Guzzle timeout shorter than 70 seconds and the request is still in flight, you eat the credits anyway.
- **10-minute forced caching on difficult targets.** For protected sites, ScraperAPI caches the response for 10 minutes. If you're scraping time-sensitive pricing or stock levels, you may be reading stale data.
- **No proactive usage alerts.** The dashboard shows usage but doesn't email you when you're at 80%. Set a cron job that calls the `account` API endpoint and pings you, or just check the dashboard daily during your first month.
- **DataPipeline credit costs are higher.** ScraperAPI's no-code scheduled-scraping product uses a separate credit schedule — a basic normal request costs 6 credits in DataPipeline versus 1 via the standard API. If you're considering DataPipeline instead of writing your own PHP cron jobs, factor that in.
- **Credits don't roll over.** Build your scraping schedule around monthly renewal, not around stockpiling credits for a big quarterly pull.

---

## Alternatives Within the PHP Ecosystem

If you're shopping around, the other scraping APIs commonly compared to ScraperAPI are ScrapingBee, Scrapfly, ZenRows, and Bright Data. They all have HTTP APIs you can call from PHP the same way you'd call ScraperAPI — cURL, Guzzle, or a community SDK — so the integration story is roughly identical across all of them. The differences are in pricing model, success rates on hard targets, and whether JS rendering is enabled by default.

The reason **scraperapi php** is such a common search is that ScraperAPI's combination of a clean single-endpoint API, a first-party PHP SDK, transparent (if sometimes confusing) credit pricing, and a permanent free tier makes it the path of least resistance for most PHP developers getting started with managed scraping. The 7-day / 5,000-credit trial is enough to actually validate your use case, not just play around.

---

## Wrapping Up

Building a web scraper in PHP isn't hard. Building one that survives contact with a real website — anti-bot defenses, CAPTCHAs, geo-fencing, JavaScript-rendered SPAs, IP bans — is genuinely hard, and it's the part that eats weeks of engineering time if you try to do it yourself. ScraperAPI's pitch is that for a few dollars a month you can outsource all of that infrastructure to a managed service and keep your PHP code focused on what to do with the data, not how to fetch it.

The integration is genuinely simple: install the SDK with `composer require scraperapi/sdk`, or just paste the cURL example into any PHP file. The hard part — and the part worth spending an hour on before you commit to a paid plan — is understanding the credit multiplier system so you can predict what your actual monthly cost will be. Run a few real requests through the free tier, watch the `sa-credit-cost` header, multiply by your expected volume, and pick the plan that fits.

If you're ready to try it on your own targets, 👉 [grab the free ScraperAPI trial here](https://www.scraperapi.com/?fp_ref=coupons) — 1,000 credits every month forever, 5,000 credits for the first 7 days, no credit card required. Test your actual target sites, see how many credits each one costs, and then decide which tier fits your volume. The math is much easier to do with real numbers from your own dashboard than with hypotheticals from a pricing page.
