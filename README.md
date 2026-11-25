# ScraperAPI Ruby: how the gem works, what each request really costs in credits, and which plan fits your crawl (full plan table)

The Ruby entry point to ScraperAPI is a gem called `scraperapi`, and a working request is three lines of code:

ruby
require "scraper_api"
client = ScraperAPI::Client.new(ENV["SCRAPERAPI_API_KEY"])
html = client.get("https://example.com/").raw_body


That part is simple. What decides your bill is what you pass after the URL. A plain GET costs 1 credit, `render: true` makes it 10, and `ultra_premium: true` combined with rendering makes it 75. The other thing worth knowing before you build: the gem's last release was **0.1.4 on May 17, 2020**, so a few parameters in the current documentation are newer than the library you're about to install.

Below is the setup, the parameter and credit tables, error handling by status code, the two ways to route Ruby traffic through the proxy, and a credit-based rule for picking a plan.

> **Quick answer for most readers:** a single Ruby service scraping HTML pages fits the **Hobby** tier (100,000 credits, 20 concurrent threads). If most of your targets need JavaScript rendering, that same allowance buys roughly 10,000 pages, so run the arithmetic in the credit section before you commit. The free tier gives you 1,000 credits and 5 concurrent connections to test with.
> 👉 [See the current plan prices and start on the free tier](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## All ScraperAPI plans side by side

| Plan | Credits / month | Concurrent threads | Geotargeting | Price | Link |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | US & EU | $0 | [Start free](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | On the pricing page | [See Hobby pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | On the pricing page | [See Startup pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | On the pricing page | [See Business pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | **$475/mo**, or **$427.50/mo** billed annually ($5,130/yr, save $570) | [See Scaling pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | On the pricing page | [See Professional pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | **$1,975/mo**, or **$1,777.50/mo** billed annually ($21,330/yr, save $2,370) | [See Advanced pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | On the pricing page | [See Enterprise pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two things the table doesn't show. Yearly billing takes 10% off the monthly rate across the board, which is where the annual figures for Scaling and Advanced come from. And every tier includes the same core feature set: JS rendering, premium proxies, JSON autoparsing, rotating proxy pools, CAPTCHA handling, custom sessions, desktop and mobile user agents, automatic retries and unlimited bandwidth. The tiers differ in volume and concurrency, not in whether a feature exists.

## Install the gem, then keep the key out of your code

ruby
# Gemfile
gem 'scraperapi'


then `bundle install`, or `gem install scraperapi` if you're not using Bundler.

The require path uses an underscore while the gem name doesn't, which trips people up on the first line:

ruby
require "scraper_api"

client = ScraperAPI::Client.new(ENV["SCRAPERAPI_API_KEY"])
html = client.get("https://example.com/").raw_body
puts html


`.raw_body` pulls the HTML string out of the response object. Parameters go after the URL as keyword arguments, and you can stack as many as the request needs:

ruby
html = client.get(
  "https://example.com/",
  render: true,
  country_code: "us"
).raw_body


Read the key from the environment, never from a literal in the file. On the Ruby version: the official skill documentation lists **Ruby >= 2.0**, while the RubyGems metadata for 0.1.4 says `>= 0`, which is an auto-generated field nobody maintains. Plan for 2.0 or newer and test against whatever your application already runs.

POST and PUT work through the same client:

ruby
client.post("http://httpbin.org/ip", "foo": "bar").raw_body
client.put("http://httpbin.org/ip", "foo": "bar").raw_body

# form-encoded POST
client.post(
  "http://httpbin.org/anything",
  body: { "foo" => "bar" },
  headers: { "Accept" => "application/json" }
).raw_body


## The parameters that matter, and the credit each one adds

| Request | Credits |
| --- | --- |
| Standard page | 1 |
| `render: true` | 10 |
| `premium: true` | 10 |
| `premium: true` + `render: true` | 25 |
| `ultra_premium: true` | 30 |
| `ultra_premium: true` + `render: true` | 75 |
| `country_code` | no extra cost |
| `session_number` | no extra cost |
| `device_type` | no extra cost |
| `autoparse` | no extra cost |

A few rules that only show up when you read the parameter reference closely:

- **`premium` and `ultra_premium` are mutually exclusive.** Passing both is not a stronger request, it's an invalid one. The documented escalation order is standard (1) → render (10) → premium (10) → ultra_premium (30).
- **`keep_headers` is ignored when `ultra_premium` is on.** Your custom headers get discarded at the top tier, which matters if the target site needs a specific `Accept-Language` or auth header to return real content.
- **`wait_for_selector` needs `render: true`.** It waits for a CSS selector in the rendered DOM, so it's useless on its own.
- **`render: true` is not a default.** The documentation is explicit that you should try without it first, because it adds roughly 9 credits and a chunk of latency per request.
- **`autoparse: true`** returns structured JSON instead of HTML for supported platforms, so `client.get("https://amazon.com/dp/B09V3KXJPB", autoparse: true).raw_body` hands you parsed product data rather than a page you have to scrape.

The cheapest way to write this is a ladder that stops as soon as the page comes back intact:

ruby
def scrape_with_escalation(client, url)
  tiers = [
    {},
    { render: true },
    { premium: true },
    { premium: true, render: true },
    { ultra_premium: true },
  ]

  tiers.each do |params|
    result = client.get(url, **params).raw_body
    return result if result&.include?("<html")
  end

  nil
end


Checking for `"<html"` is a crude success test, but it's the one the official guide uses: a blocked response usually comes back as an error page or a challenge, not as the document you asked for.

👉 [Compare what each plan's credit allowance buys you](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Domain multipliers can outweigh your parameters

The parameter table above is only half the cost model. Some sites carry their own price, regardless of what you pass: **Amazon, Walmart and eBay cost 5 credits** per request, **Google and Bing cost 25** (subdomains included), and **LinkedIn costs 30**. Sites sitting behind Cloudflare, Cloudflare Turnstile, DataDome, PerimeterX/HUMAN or Akamai add **10 credits** when ScraperAPI bypasses the protection.

So a rendered Google search isn't 10 credits. It's 35, and a few thousand of those will drain a Hobby allowance faster than any Ruby-side optimisation can compensate for. Two tools keep this predictable: every response carries its cost in the `sa-credit-cost` header, and you can set `max_cost` on a request so a single scrape can't exceed a limit you choose. The Domain Cost Estimator in the dashboard gives you the number before you write the loop.

## Sessions, batches, and structured endpoints

**Sticky sessions.** Pass `session_number: 42` and every request carrying that integer reuses the same proxy IP, which is what paginated crawls and multi-step flows need. Any integer works. Sessions expire **15 minutes after last use**, so a nightly job that pauses longer than that starts on a fresh IP.

ruby
page1 = client.get("https://example.com/page1", session_number: 42).raw_body
page2 = client.get("https://example.com/page2", session_number: 42).raw_body


**Async jobs.** The SDK is synchronous: each `client.get` blocks until the response lands, up to about **70 seconds**. Twenty URLs in a `map` is 20 sequential waits. For batches, submit jobs to `https://async.scraperapi.com/jobs` with `{ apiKey, url, apiParams }`, get back `{ "id", "statusUrl" }`, and poll the `statusUrl` until `status == "finished"`, then read `data.dig("response", "body")`. A `"failed"` status should raise. The official example polls every 5 seconds with a 120-second ceiling.

**Structured endpoints.** For supported verticals you can skip HTML entirely:

ruby
results  = structured_get("google/search", { query: "ruby web scraping" })
product  = structured_get("amazon/product", { asin: "B09V3KXJPB" })
listings = structured_get("ebay/search", { query: "mechanical keyboard" })


The base is `https://api.scraperapi.com/structured/{vertical}`, with `api_key` and the vertical's own parameters in the query string.

## Error handling by status code

The gem raises on HTTP errors, so the status code is what drives your `rescue`:

| Code | Meaning | What to do |
| --- | --- | --- |
| 200 | Success | Parse the body |
| 401 | Invalid API key | Check `SCRAPERAPI_API_KEY` is actually set in the process |
| 403 | Blocked, or out of credits | Escalate to `premium: true` or `ultra_premium: true`; otherwise check the balance |
| 404 | Target page doesn't exist | Fix the URL, not the parameters |
| 429 | Concurrency limit hit | Lower parallelism or move to async jobs |
| 500 / 503 | Transient | Retry with exponential backoff |

**500 and 503 responses are not charged**, which is why they're safe to retry aggressively. A 429 isn't a hard failure either: the request is turned away rather than queued, and no extra charge applies for going over the limit. The relevant ceiling is your plan's concurrent threads, so a `Thread.new` fan-out of 50 on a 20-thread Hobby plan will produce 429s no matter how well the Ruby is written.

ruby
def safe_scrape(client, url, params = {})
  client.get(url, **params).raw_body
rescue => e
  status = e.respond_to?(:response) ? e.response&.code&.to_i : nil
  case status
  when 401 then raise "Invalid API key"
  when 403 then raise "Blocked or out of credits"
  when 429 then raise "Rate limit, reduce concurrency or go async"
  when 500, 503 then raise "Transient, retry with backoff"
  else raise
  end
end


## Check the gem version before you build around it

RubyGems lists 0.1.4 as the newest release, published **May 17, 2020**, 9.5 KB, MIT licensed, with a single runtime dependency on `unirest ~> 1.1.2`. Total downloads sit at 18,494, of which 11,012 are for 0.1.4. There has been no release since.

Meanwhile `ultra_premium`, `autoparse`, `device_type`, `wait_for_selector` and `instruction_set` appear in documentation dated 2024. The docs don't state whether the gem forwards those keyword arguments unchanged, and there's no changelog entry to settle it. Practically:

bash
gem list scraperapi


If you're on 0.1.4 and a newer parameter silently does nothing, you have two reliable routes that don't depend on the gem at all: the REST endpoint via `Net::HTTP`/`Faraday`, or the proxy. Both accept the full current parameter set.

👉 [Create an API key and test the REST endpoint first](https://www.scraperapi.com/?fp_ref=coupons)

## Routing Ruby through the proxy (two documented ways)

If you'd rather not depend on the gem, ScraperAPI works as an HTTP proxy, and this is also how you drive headless Chrome from Ruby.

**Proxy authentication**, usable with HTTParty or any client that supports proxy credentials:

ruby
response = HTTParty.get(
  "https://httpbin.org/ip",
  http_proxyaddr: "proxy-server.scraperapi.com",
  http_proxyport: "8001",
  http_proxyuser: "scraperapi",
  http_proxypass: ENV["SCRAPERAPI_API_KEY"]
)


**Proxy URL with query parameters**, which is the form the Ferrum guide uses:

ruby
require 'ferrum'
require 'dotenv/load'

proxy_url = "http://api.scraperapi.com:8001?api_key=#{ENV['SCRAPERAPI_KEY']}&render=true&country_code=us"
browser = Ferrum::Browser.new(browser_options: { 'proxy-server': proxy_url })
browser.goto('https://news.ycombinator.com/')
browser.css('.athing .titleline a').first(5).each { |a| puts a.text.strip }
browser.quit


Both appear in official material, so neither is a workaround. Ferrum needs Ruby 2.6+, Bundler and a local Chrome or Chromium install. It doesn't rotate IPs or solve CAPTCHAs on its own, which is the reason to put it behind the proxy. For concurrency, run several `Ferrum::Browser` instances in threads and wrap `goto` in a `rescue Ferrum::StatusError` retry; one browser per thread, not one browser shared.

Note the split: parameters such as `render` can be sent as headers (`x-sapi-render`, `x-sapi-api_key`) against the API endpoint, but when you go through the proxy, the credentials must be passed as proxy authentication, not as headers.

## How many credits does your Ruby scraper actually need?

Multiply your monthly page count by the credit cost of the request you'll really send, not the cheapest one. The same allowance in pages, before domain multipliers:

| Plan | Standard (1 cr) | With `render` (10 cr) | `premium` + `render` (25 cr) |
| --- | --- | --- | --- |
| Hobby | 100,000 | 10,000 | 4,000 |
| Startup | 1,000,000 | 100,000 | 40,000 |
| Business | 3,000,000 | 300,000 | 120,000 |
| Scaling | 5,000,000 | 500,000 | 200,000 |
| Professional | 10,500,000 | 1,050,000 | 420,000 |
| Advanced | 21,500,000 | 2,150,000 | 860,000 |

Then check concurrency separately. With blocking calls of up to 70 seconds each, 20 threads on Hobby is a theoretical ceiling of roughly 1,000 requests per hour, and you'll rarely hit it against slow targets. If your throughput requirement is higher than that arithmetic allows, the tier you need is decided by threads, not credits.

**Pick by situation:**

- Testing an integration or scraping a few hundred pages a month: **Free**. 1,000 credits, 5 threads, and you'll know within an afternoon whether the gem handles your target.
- One production service, mostly static HTML: **Hobby**. 100,000 credits covers 100,000 pages.
- Same service, but every target is a React or Vue app: still Hobby, but your real budget is 10,000 pages, so be honest about volume.
- Several services or a data team sharing one key: **Business** or above, where country-level geotargeting also becomes available.
- Continuous pipelines past 5 million credits: **Scaling** and up.

One limit to know before you pick a mid tier: **pay-as-you-go is only available on Scaling, Professional, Advanced and Enterprise.** On Free, Hobby, Startup and Business, credits simply pause when they run out and resume at renewal. Auto-upgrade to the next plan is offered on Hobby, Startup, Business, Scaling and Professional, which is the only way to keep a Business account running mid-cycle. If your traffic is spiky and you'd rather top up than jump tiers, that's a reason to start at Scaling.

Credits also reset at renewal rather than rolling over, and annual plans can renew early to pull a fresh batch.

## Before you write the first loop

Three checks that cost nothing: run `gem list scraperapi` so you know which version is answering, confirm the per-domain cost for your main target in the dashboard's Domain Cost Estimator, and set `max_cost` on anything that runs unattended. After that, the only decision left is size: take your monthly page count, multiply by the credit cost of the request type you'll actually use, and pick the smallest tier that clears it with room for retries.

👉 [Pick your plan and get the API key](https://www.scraperapi.com/pricing/?fp_ref=coupons)
