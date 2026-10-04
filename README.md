# playwright proxy authentication: fixing dropped credentials, 407 errors and SOCKS5 auth in Chromium and Firefox

You pass a proxy to Playwright, the browser opens, the page loads, and everything looks fine — until you check the IP and it's your own. Or you get a bare `407 Proxy Authentication Required` and the docs seem to say credentials are supported. Both outcomes come from the same thing, and it isn't your proxy provider.

Here's what's actually happening, how to fix it per engine, and how to pick a proxy endpoint whose auth model matches how Playwright works — including the rotation pattern most tutorials skip.

## The reason most proxy auth fails in Playwright

Playwright does not parse credentials out of a proxy URL. `requests` and Scrapy do. Playwright doesn't.

So this looks correct and is not:

python
# credentials in the URL get dropped
browser.new_context(proxy={"server": "http://user:pass@gw.example.com:913"})


The proxy receives a blank `Proxy-Authorization` header, and your request either dies with a 407 or falls through to a direct connection. The fix is to split the parts into the keys Playwright expects:

python
browser.new_context(proxy={
    "server": "http://gw.example.com:913",
    "username": "user",
    "password": "pass",
})


That's the whole bug in one line, and it accounts for the majority of "Playwright proxy authentication not working" threads. Two details around it matter just as much:

**Proxy is per context, not per page.** Every page inside a context shares the context's exit IP. Rotating means creating a new context, not a new page.

**Context-level proxies need a hint.** If you set the proxy on `new_context()` rather than at launch, Playwright needs a non-empty placeholder at the browser level or it will quietly ignore the context proxy:

python
browser = chromium.launch(proxy={"server": "per-context"})
context = browser.new_context(proxy={
    "server": "http://gw.example.com:913",
    "username": "user",
    "password": "pass",
})


Don't confuse this with `http_credentials`, which authenticates you to the *target website* (HTTP basic auth), not to the proxy. They're separate mechanisms and mixing them up produces confusing results.

## Same config, different scopes

The proxy dict is identical everywhere; only the placement changes. Pick based on whether one proxy serves the run or each worker gets its own.

| Where you set it | Scope | When it makes sense |
| --- | --- | --- |
| `chromium.launch(proxy=...)` | Entire browser | One static proxy for the whole run |
| `browser.new_context(proxy=...)` | Single context | Per-worker or per-identity exit IPs |
| `use: { proxy: ... }` in `playwright.config.ts` | Whole test project | CI runs behind a fixed corporate proxy |

In Playwright Test, it goes in config rather than code:

javascript
// playwright.config.ts
export default defineConfig({
  use: {
    proxy: {
      server: 'http://gw.example.com:913',
      username: process.env.PROXY_USER,
      password: process.env.PROXY_PASS,
    },
  },
});


Keeping credentials in environment variables isn't paranoia here — CI logs and test reports are a common place for secrets to leak, and proxy passwords are exactly the kind of thing that ends up pasted into a screenshot.

`bypass` is worth knowing about too, especially on corporate networks where the proxy is for external traffic only:

python
bypass="localhost,127.0.0.1,*.internal.company.com,192.168.*"


Without it, internal hostnames get routed through the proxy and you get timeouts that look like proxy failures but aren't.

## SOCKS5 with a username and password

This is the part where the documentation itself is the trap.

Playwright's API reference says the `server` field supports HTTP and SOCKS proxies — `socks5://myproxy.com:3128` is a valid example. Separately, it describes `username` and `password` as being for proxies that require authentication *if the HTTP proxy requires it*. Those are two different lists. SOCKS appears in the transport list, not in the credentials list.

Upstream confirms it: [microsoft/playwright#10567](https://github.com/microsoft/playwright/issues/10567), "Support socks5 proxy with authentication," has been open since November 2021 and is still labeled as collecting feedback. A feature request open that long is not a feature that quietly works.

What happens when you pass credentials anyway depends on the engine:

- **Chromium** raises `net::ERR_PROXY_AUTH_UNSUPPORTED`, or accepts the arguments and drops the credentials on the SOCKS path. The second case is worse, because nothing in the visible behaviour tells you the auth never happened.
- **A silent fall-through to a direct connection looks exactly like success** until you look at the exit IP.

If your provider hands you an authenticated SOCKS5 endpoint, you have three realistic routes:

| Route | Works on | What it costs you |
| --- | --- | --- |
| Local relay that authenticates upstream | Every engine | A supervised process and a port |
| Use the provider's HTTP/HTTPS endpoint instead | Every engine | Provider has to offer one |
| Firefox proxy prefs via `firefox_user_prefs` | Firefox only | Needs a build that carries the credentials |

The local relay is the common answer in practice. A small local proxy listens without auth on `127.0.0.1`, injects the `Proxy-Authorization` header on your behalf, and Playwright points at the unauthenticated local address. It works on Chromium, WebKit and Firefox. The version of this pattern that shows up repeatedly in the wild targets a specific Chromium weakness: HTTPS `CONNECT` tunnels through an auth-requiring proxy can fail even when credentials are supplied through the proxy config, so the relay does the authentication the browser won't.

> Rule of thumb: Playwright's proxy dict documents username and password for HTTP(S) proxies. If your provider's authenticated endpoint is SOCKS5-only, budget for a relay or ask them for an HTTP endpoint before you write any automation code.

In practice most residential providers offer both, which makes the HTTP endpoint the path of least resistance.

## Verify the exit IP, not the status code

The single most useful debugging habit here: don't test proxy auth with a page load. Test it by asking the proxy what address it's presenting.

python
context = browser.new_context(proxy=PROXY)
page = context.new_page()
page.goto("https://ipinfo.io/ip")
print(page.inner_text("body"))  # must differ from your real IP


A 200 response proves a page rendered. It does not prove the request came from the proxy. Silent credential drops and silent direct connections both return 200. Comparing the reported address against your own IP is the only check that distinguishes the three outcomes.

Run that check before scaling to a hundred contexts, or you'll spend an afternoon diagnosing "blocked" pages that were never actually proxied.

## Rotating proxies across a Playwright run

Once auth works, rotation is a loop construct. Each `new_context()` gets its own exit IP; pages inside it inherit that IP.

python
for i in range(3):
    context = browser.new_context(proxy=proxy_config)
    page = context.new_page()
    page.goto("https://ipinfo.io/ip")
    print(page.inner_text("body"))
    context.close()   # release the context, not the browser


Two practical notes. Reusing one browser and cycling contexts is far cheaper than launching a browser per identity — browser startup dominates the runtime in most scraping loops. And rotating by creating new pages instead of new contexts does nothing at all, because the exit IP belongs to the context.

Some providers rotate server-side, so a rotating gateway hands you a fresh IP on every connection and the loop above doesn't need to close contexts at all. Others rotate by username parameter, which is where the next section matters.

## Choosing a proxy endpoint that fits how Playwright authenticates

Given that Playwright documents proxy credentials for HTTP(S) proxies and expects `host:port` plus separate `username`/`password`, the endpoint you buy decides how much glue code you write.

9Proxy runs two residential products with genuinely different auth models, and the difference is easy to miss until you're mid-integration:

**Residential by GB** authenticates with username/password or IP whitelisting and runs entirely from the dashboard — no desktop app. That's the shape Playwright's proxy dict expects, which means your context config looks like the examples above with no relay, no app, no local port.

**Residential by IPs** uses a desktop app that forwards proxies to local ports. By default those local ports need no credentials at all, so Playwright points at something like `http://127.0.0.1:9100` with no auth dict. Proxy authentication is optional and can be enabled in the Forwarding List if you want to restrict local access. It works, but it assumes the app is running on the machine — fine for a workstation, awkward for a headless CI container.

9Proxy reports 20M+ residential IPs across 90+ countries [1](https://docs.9proxy.com/overview.md).

The GB product embeds targeting inside the username string rather than in separate API calls, which suits Playwright because you can build a different context per worker from a single template:


<sub-user>-country-us-st-ohio-city-newyork-isp-as22773_Cox_Communications_Inc.-sst-15-ssid-bot01


The parts that matter for automation:

- `country-us` — two-letter country code. Targeting only by country returns the fastest responses; stacking state, city and ISP narrows the pool and can slow things down or thin availability.
- `sst-15` — sticky session duration in minutes. This is the parameter that keeps one IP for the lifetime of an authenticated session, which is what you want for logged-in flows and shopping carts.
- `ssid-bot01` — a session ID that lets you run parallel sticky sessions from one config. Each distinct `ssid` yields a different IP even when every other parameter is identical. This is the clean way to map N Playwright workers to N stable identities without maintaining a list of endpoints.

Omitting `sst` and `ssid` gives you rotation per request; specifying them gives you stickiness. You can mix modes across contexts in the same run.

If you want to check current endpoint options and package pricing directly, 👉 [see 9Proxy's residential proxy plans](https://bit.ly/9-Proxy).

## 9Proxy pricing: every plan on the page

9Proxy bills by balance rather than subscription, so packages are one-off purchases. IP-based packages don't expire as you use them; GB-based packages carry 180-day validity, unlimited on Enterprise. Prices below reflect the adjustment that took effect for IP-based and bundle packages on 1 June 2026 — GB-based pricing was unchanged [2](https://blog.9proxy.com/news/9proxy-pricing-update/)[3](https://geekflare.com/guides/9proxy-review/).

### Residential by IPs

Unlimited bandwidth per IP. Best when you need predictable, sustained sessions.

| Package | IPs | Price | Effective per IP | Purchase |
| --- | --- | --- | --- | --- |
| Starter | 100 | $24 | $0.24 | [Get the 100 IP package](https://bit.ly/9-Proxy) |
| Standard | 500 | $72 | $0.144 | [Get the 500 IP package](https://bit.ly/9-Proxy) |
| Value | 1,000 + 500 bonus | $126 | $0.084 | [Get the 1,500 IP package](https://bit.ly/9-Proxy) |
| Growth | 2,500 | $210 | $0.084 | [Get the 2,500 IP package](https://bit.ly/9-Proxy) |
| Scale | 5,000 | $360 | $0.072 | [Get the 5,000 IP package](https://bit.ly/9-Proxy) |
| Bulk | 15,000 | $720 | $0.048 | [Get the 15,000 IP package](https://bit.ly/9-Proxy) |
| Bulk | 25,000 | $863 | ~$0.035 | [Get the 25,000 IP package](https://bit.ly/9-Proxy) |
| Bulk | 50,000 | $1,438 | ~$0.029 | [Get the 50,000 IP package](https://bit.ly/9-Proxy) |

### Business IP packages

| Package | IPs | Price | Effective per IP | Purchase |
| --- | --- | --- | --- | --- |
| Business | 100,000 | $2,300 | ~$0.023 | [Get the 100,000 IP package](https://bit.ly/9-Proxy) |
| Business | 200,000 | $4,140 | ~$0.021 | [Get the 200,000 IP package](https://bit.ly/9-Proxy) |
| Business | 500,000 | $8,625 | ~$0.018 | [Get the 500,000 IP package](https://bit.ly/9-Proxy) |

### Residential by GB

Pay for traffic, generate unlimited endpoints. 180-day validity.

| Package | Traffic | Price | Effective per GB | Purchase |
| --- | --- | --- | --- | --- |
| Trial size | 5 GB | $15 | $3.00 | [Get the 5 GB package](https://bit.ly/9-Proxy) |
| Regular | 50 GB + 5 GB bonus | $105 | $2.10 | [Get the 55 GB package](https://bit.ly/9-Proxy) |
| Regular | 100 GB | $150 | $1.50 | [Get the 100 GB package](https://bit.ly/9-Proxy) |
| Regular | 200 GB | $200 | $1.00 | [Get the 200 GB package](https://bit.ly/9-Proxy) |
| Regular | 1,000 GB | $800 | $0.80 | [Get the 1,000 GB package](https://bit.ly/9-Proxy) |
| Regular | 2,000 GB | $1,500 | $0.75 | [Get the 2,000 GB package](https://bit.ly/9-Proxy) |

### Enterprise GB packages

Unlimited validity, team mode (1 owner + up to 5 members), per-member traffic controls and activity logs.

| Package | Traffic | Price | Effective per GB | Purchase |
| --- | --- | --- | --- | --- |
| Enterprise | 3,000 GB | $2,160 | $0.72 | [Get the 3,000 GB package](https://bit.ly/9-Proxy) |
| Enterprise | 6,000 GB | $4,200 | $0.70 | [Get the 6,000 GB package](https://bit.ly/9-Proxy) |
| Enterprise | 10,000 GB | $6,800 | $0.68 | [Get the 10,000 GB package](https://bit.ly/9-Proxy) |

### Bundle packages

IPs and traffic together, for runs where some contexts need stable identities and others just need volume.

| Bundle | Contents | Price | Purchase |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | [Get the Pro bundle](https://bit.ly/9-Proxy) |

Which model a Playwright project wants comes down to how your contexts behave. If each context holds one identity for a whole session — login flows, checkout tests, anything where changing IP mid-run breaks the test — per-IP billing with unlimited bandwidth is the cheaper structure, because you're paying for identities rather than data. If contexts are short-lived and you burn thousands of requests through thousands of different exit IPs, per-GB billing is almost always cheaper, because one IP per request would mean paying for IPs you use once.

The bundle tier exists for the mixed case: a persistent worker pool plus a rotating fleet in the same run.

## Error → cause → fix

| Error | Usual cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials embedded in the proxy URL | Split into `server`, `username`, `password` |
| Requests succeed, wrong IP | Context proxy ignored, or credentials silently dropped | Pass a placeholder `server` at launch for context-level proxies; verify with an IP echo endpoint |
| `net::ERR_PROXY_AUTH_UNSUPPORTED` | SOCKS5 with username/password | Use the provider's HTTP endpoint or a local relay |
| `net::ERR_PROXY_CONNECTION_FAILED` | Wrong host/port, or the port-forwarding app isn't running | Check the port and that the local forwarding process is up |
| Every page uses the same IP after rotation | New pages created instead of new contexts | Create `browser.new_context()` per identity |
| Internal hostnames time out | Traffic routed through the proxy unnecessarily | Add `bypass` |
| Auth works locally, fails in CI | Credentials not exported to the CI environment | Read from environment variables in `use.proxy` |

The first row accounts for the most wasted debugging time, and the second is the most dangerous, because it produces green tests that never touched the proxy.

## What to check before you scale

Three things, in order:

1. **Confirm credentials are structured, not embedded.** Print the proxy dict you pass to `new_context()` and check it has three keys, not one URL.
2. **Confirm the exit IP is the proxy's.** Hit an IP echo endpoint in every context at least once during development. A 200 means nothing here.
3. **Confirm your auth method matches your endpoint.** HTTP with username/password works as documented. Authenticated SOCKS5 does not, and no amount of config will change that.

Get those three right and the rest of the integration is routine. If you're still choosing a provider, 👉 [start with 9Proxy's IP-based and GB-based residential plans](https://bit.ly/9-Proxy) — the GB product's username/password model is the one that drops straight into Playwright's proxy config without a relay in front of it.
