# Rotating proxies: how per-request rotation works, which type to pick, and what it should cost

Most people land on rotating proxies after something broke. A scraper that ran fine for 200 pages starts returning 403s, or a page that came back clean yesterday now hands you a Cloudflare challenge. The proxy isn't usually the whole story, but it's the first place the fix lives.

Here's the part that gets glossed over in most roundups: "rotating proxies" isn't one product. It's a rotation policy applied to a pool, and the pool matters more than the rotation. A cheap datacenter pool rotating on every request gets blocked faster than a good residential pool rotating on a timer, because the rotation only changes *which* IP looks suspicious, not whether it looks suspicious at all.

So the useful questions are narrower than the marketing suggests. What triggers the rotation? Which network should the IPs come from? And what does the per-gigabyte price actually buy you once failed requests are counted?

## What rotation actually changes

A proxy sits between your client and the target. The site sees the proxy's IP instead of yours. Rotation decides how often that IP changes:

- **Per-request rotation** — a new exit IP for every request. Built for workloads made of many independent fetches: search result pages, listing pages, product lookups. No session state, no reason for request N to remember request N-1.
- **Timer or session rotation** — the same IP holds for a window (30 seconds, 10 minutes, 2 hours, depending on the provider), then switches. Built for anything that carries cookies, a CSRF token, a cart, or a paginated flow tied to a server-side session.
- **Manual rotation** — you trigger a new IP on demand, usually before a sensitive step like a login.

The failure mode of getting this backward is loud in both directions. Rotate mid-login and the session cookie issued to IP A gets presented from IP B in a different city — that contradiction is easier for a site to detect than a slightly odd user agent. Hold one IP while hammering a rate-limited endpoint and you collect 429s until the address is burned.

Rotation is not secrecy. It's traffic distribution. If a site is counting requests per IP, spreading them across many addresses keeps any single one below the threshold.

## Per-request or sticky: pick by what your requests depend on

A useful test: does request 2 depend on anything request 1 established?

| Your workload | Rotation to use | Why |
| --- | --- | --- |
| SERP scraping, keyword tracking | Per-request | Every request is a fresh, stateless lookup |
| Large product catalog crawl | Per-request | Nothing carries between URLs |
| Login flows, account dashboards | Sticky, short window | Cookies and auth tokens bind to one IP |
| Paginated results behind a session | Sticky (1–30 min) | Server tracks your position by session |
| Price monitoring on schedule | Sticky or per-request | Depends on whether the page sets session state |
| Ad verification by region | Sticky + geo targeting | The IP has to match the market being checked |

There's a second axis people miss: the browser. A single page load is dozens of requests — document, subresources, XHR, sometimes a websocket — all tied together by state the site handed you on arrival. Sending that through per-request rotation produces one visitor whose requests contradict their own cookies. If you're driving a headless browser rather than a raw HTTP client, sticky is usually the honest default, and rotation belongs at the task boundary: one identity per visit, then switch.

## The proxy type ladder, and why rotation changes the price

Rotation policy is free; the pool it runs on isn't. The four network types sit on a detectability-versus-cost curve, and the gap between the bottom and top rungs is roughly 10x.

| Type | What the IP is | Rotation behavior | Typical market price |
| --- | --- | --- | --- |
| Datacenter | Hosting-provider ranges | Per-request or timed | ~$0.50–$2 per GB |
| Residential | Real home broadband addresses | Per-request or sticky | ~$1–$8 per GB |
| ISP / static residential | Datacenter-hosted, ISP-registered | Effectively always sticky | ~$2–$6 per IP per month |
| Mobile (4G/5G) | Carrier NAT addresses | Per-request or sticky | ~$2–$15+ per GB |

The practical rule is to start one rung below where you think you need to be. Most scraping work is a long tail of easy domains plus a handful of defended ones, and paying residential rates for the easy majority is how budgets quietly disappear. Move up only for the hosts that actually block you.

Mobile is the last resort, not the default. Blocking one mobile IP means blocking thousands of real subscribers behind carrier NAT, which is why these pools survive where others fail — and why they cost several times residential.

## The metric that matters isn't $/GB

Price per gigabyte is the sticker. What you pay for is successful responses, and blocked responses still bill.

If 20% of your requests fail and you retry each once, your effective cost is roughly 1.2x the headline rate. A $1/GB pool with a 40% failure rate costs more per usable page than a $3/GB pool failing 5% of the time. That's arithmetically obvious and almost universally ignored in comparison tables.

Three numbers decide it, and only one is on the pricing page:

1. **Success rate on your specific target.** Published benchmarks cover a handful of sites; yours may behave differently. Test with a few hundred requests before committing to volume.
2. **Latency.** A slow pool that succeeds looks efficient in a report and starves a pipeline in production.
3. **Expiry.** If unused bandwidth resets monthly, a lumpy workload pays for traffic it never used. Watch for a paid top-up threshold that lands above your real monthly volume.

Point 3 deserves more attention than it gets. A $49.99 monthly plan for 100 GB is $0.50/GB if you use all 100 GB and effectively $10/GB if you use 5. Subscription minimums that sit above your usage turn the best headline rate into the worst real one.

## Where DataImpulse fits

DataImpulse runs rotating residential, datacenter, mobile, and premium residential pools on a straight pay-as-you-go model, with the residential shelf price at $1/GB and no subscription. Traffic doesn't expire, so unused gigabytes roll over instead of resetting — the relevant difference if your monthly volume swings or you run ad-hoc jobs.

The rotation mechanics, per the provider's own configuration docs:

- **Rotating HTTP/HTTPS:** `gw.dataimpulse.com:823`
- **Rotating SOCKS5:** port `824`
- **Sticky sessions:** ports 10000–20000, held 1 to 120 minutes; if no interval is set, the default is 30 minutes

Targeting is passed through the proxy username rather than a separate API — `user:pass_country-us` for country, with city, state, ZIP, and ASN appended the same way. Country-level targeting and ASN exclusion are inside the base price. State, city, ZIP, and specific-ASN routing on residential plans bill at **2x the standard per-GB rate**, which is the single most common way a DataImpulse bill comes in higher than planned. If your workflow needs city-level precision, budget $2/GB on residential, not $1.

Two other things worth knowing before you buy. There's no managed scraping API — you get raw proxy credentials and write your own requests, retries, and parsing. That's a reasonable fit for teams already running their own collectors and a hard no for anyone who wanted a service that returns parsed JSON. And the pool is first-party, sourced from paid, opted-in users rather than resold, which is the claim behind the clean IP reputation. Independent reviewers report a published success rate around 99.51%, a G2 score of 4.8/5, and sessions to 120 minutes.

One limitation to flag plainly: if you need static ISP proxies, a fully managed scraping API, or access to banking and government sites, this isn't the right provider. It's built for public-data collection and access through rotating residential, mobile, and datacenter IPs.

👉 [Start with a 5 GB rotating residential plan and test it on your own target](https://bit.ly/dataimPulse)

## Every current plan, in one place

All four product lines, all tiers, all pay-as-you-go. Nothing here is a subscription.

| Plan | Product | Traffic | Price | Effective rate | Purchase |
| --- | --- | --- | --- | --- | --- |
| Intro | Residential | 5 GB | $5 | $1.00/GB | [Get the residential Intro plan](https://bit.ly/dataimPulse) |
| Basic | Residential | 50 GB | $50 | $1.00/GB | [Get the residential Basic plan](https://bit.ly/dataimPulse) |
| Advanced | Residential | 1 TB | $800 | $0.80/GB | [Get the residential Advanced plan](https://bit.ly/dataimPulse) |
| Custom+ | Residential | 5 TB+ | From $4,000 | Custom | [Request residential Custom+ pricing](https://bit.ly/dataimPulse) |
| Intro | Datacenter | 10 GB | $5 | $0.50/GB | [Get the datacenter Intro plan](https://bit.ly/dataimPulse) |
| Basic | Datacenter | 100 GB | $50 | $0.50/GB | [Get the datacenter Basic plan](https://bit.ly/dataimPulse) |
| Advanced | Datacenter | 1 TB | $450 | $0.45/GB | [Get the datacenter Advanced plan](https://bit.ly/dataimPulse) |
| Custom+ | Datacenter | 5 TB+ | From $2,250 | Custom | [Request datacenter Custom+ pricing](https://bit.ly/dataimPulse) |
| Intro | Mobile | 2.5 GB | $5 | $2.00/GB | [Get the mobile Intro plan](https://bit.ly/dataimPulse) |
| Basic | Mobile | 25 GB | $50 | $2.00/GB | [Get the mobile Basic plan](https://bit.ly/dataimPulse) |
| Advanced | Mobile | 1 TB | $1,600 | $1.60/GB | [Get the mobile Advanced plan](https://bit.ly/dataimPulse) |
| Custom+ | Mobile | 5 TB+ | From $8,000 | Custom | [Request mobile Custom+ pricing](https://bit.ly/dataimPulse) |
| Intro | Premium residential | 1 GB | $5 | $5.00/GB | [Get the premium residential Intro plan](https://bit.ly/dataimPulse) |
| Basic | Premium residential | 10 GB | $50 | $5.00/GB | [Get the premium residential Basic plan](https://bit.ly/dataimPulse) |
| Custom+ | Premium residential | 5 TB+ | From $20,000 | Custom | [Request premium residential Custom+ pricing](https://bit.ly/dataimPulse) |

Some readings of this table:

- **Datacenter is the cheapest route to per-request rotation** at $0.50/GB, and state/city/ZIP/ASN targeting is listed as included on that product line rather than billed at 2x. Worth confirming with support, but if it holds, datacenter is where precise geo-targeting is cheapest.
- **Residential at $1/GB** is the balanced default: rotating and sticky from the same pool, 90M+ IPs across 195 countries, country targeting baked in.
- **Mobile at $2/GB** is unusually low for carrier IPs, which typically start near $3/GB. The catch is that volume discounts only kick in at the 1 TB tier.
- **Premium residential** is $5/GB and its Basic tier tops out at 10 GB before jumping to a custom 5 TB+ quote. It bundles a dedicated account manager and all targeting without surcharge — but if you need 100 GB of it, you're on a sales call.
- The Advanced tiers on residential, datacenter, and mobile include a dedicated account manager and custom features. If your volume is under 1 TB, none of that applies.

The premium residential gap deserves a note, because it's the one tier with no mid-volume option: 1 GB for $5, 10 GB for $50, then 5 TB+. Anyone needing 50 GB of high-quality residential traffic is priced out of the published ladder. Compare that against 50 GB of standard residential at $50 — the same money — before assuming premium is the upgrade you want.

## Setting it up: three steps

Rotating residential works the same way in curl, requests, httpx, and Playwright — the rotation is handled by the gateway, so there's no proxy list to maintain.

Get credentials from the dashboard after purchasing, then point your client at the rotating endpoint:

bash
curl -x http://USERNAME:PASSWORD@gw.dataimpulse.com:823 https://httpbin.org/ip


Run it twice. Different IP each time is the whole feature working. Add targeting by appending to the username:

bash
# Country targeting
curl -x http://USERNAME:PASSWORD_country-us@gw.dataimpulse.com:823 https://httpbin.org/ip

# Sticky session — same IP for the session identifier
curl -x http://USERNAME:PASSWORD_session-abc123@gw.dataimpulse.com:823 https://httpbin.org/ip


For Python, drop the same credentials into the client:

python
import httpx

proxy = "http://USERNAME:PASSWORD@gw.dataimpulse.com:823"

with httpx.Client(proxy=proxy, timeout=30) as client:
    r = client.get("https://httpbin.org/ip")
    print(r.json())


Two practical notes. Reuse one client rather than opening a new connection per request — connection setup, not rotation, is usually the throughput bottleneck. And add realistic headers; a fresh IP carrying an obviously automated header set is still a contradiction.

If you're running a headless browser instead of raw HTTP, point it at a sticky endpoint with a session ID and let rotation happen per browser launch, not per request.

👉 [Compare the rotating residential and datacenter pools side by side](https://bit.ly/dataimPulse)

## Common questions about rotation

**Does per-request rotation cost more than sticky?**
No. Both endpoints hit the same pool and bill the same per-GB rate. What costs more is advanced geo-targeting on residential, which doubles the rate.

**How many IPs do I actually get?**
Pool size is a ceiling, not a guarantee. A large advertised pool means better geographic spread and lower reuse; it doesn't mean every IP is fresh or unblocked. Test on your target.

**Can rotating proxies replace CAPTCHA solving?**
Partly. Clean, well-matched IPs reduce how often you're challenged. They don't solve the challenge when it appears. A rotation layer that's honest about this saves you a wasted purchase.

**What happens to unused traffic?**
With this provider, nothing — purchased gigabytes stay until you spend them. There's no monthly reset and no fair-use throttling. Third-party writeups also reference a 7-day refund window for new users; confirm the current terms at checkout rather than assuming.

**Is this legal?**
Running a proxy isn't the question. How you use it is — respecting target sites' terms and applicable law sits entirely on your side of the connection, regardless of which provider routes your traffic.

## What to do next

If you're under roughly 50 GB a month, the arithmetic is boring: $1/GB pay-as-you-go with a $5 entry point and non-expiring traffic beats any subscription, because subscriptions bill you for a bundle you don't finish. Above 50 GB, per-GB rates on volume tiers start to matter more, and a 1 TB residential tier at $0.80/GB is where that turns.

Start smaller than you think you need. Buy the 5 GB intro plan, point your existing scraper at the rotating endpoint, and measure two things on your actual targets: how often you get blocked, and how many gigabytes those blocked responses burn. Five dollars is a cheap way to find out whether the pool fits the job — and it's a lot cheaper than discovering it after a 1 TB commitment.

👉 [Grab the $5 rotating residential plan and test it against your own targets](https://bit.ly/dataimPulse)
