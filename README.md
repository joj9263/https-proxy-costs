# https proxy: what it actually does, how to set one up in curl, Python and your browser, and what residential IPs cost

Three different people type "https proxy" into a search box.

One is staring at `407 Proxy Authentication Required` in a terminal. One wants their scraper to stop getting blocked on a Cloudflare-protected retail site. And one is comparing price pages, wondering why residential IPs are sold by the hundreds when a VPN subscription looks cheaper.

The answers are different enough that most articles on this topic end up answering none of them well. So here's the mechanics first, then the setup, then the failure modes, then what a residential pool actually costs from a provider you can buy from today.

## What an HTTPS proxy is, minus the marketing

Strip it down and an HTTPS proxy is a proxy server your client talks to over TLS, using the standard `CONNECT` method to open a raw TCP tunnel to the destination — usually port 443.

That tunnel is the whole trick. Your client asks the proxy to connect to `github.com:443`, the proxy replies `200 Connection Established`, and from that point the TLS handshake and everything after it happens between your machine and the site. The proxy is relaying packets it can't read.

That distinction matters because two things get sold as "HTTPS proxy" and only one of them is private:

- **Pass-through (the normal case).** The proxy sees which host and port you asked for, and nothing inside the encrypted payload. This is what 9Proxy, Decodo and most residential providers sell when they say HTTP/HTTPS support.
- **SSL inspection / bridging.** The proxy terminates your TLS session, decrypts the traffic, then opens a new TLS connection to the site. Enterprise security tools do this. It only works if you install the proxy's root certificate on your machine — which is exactly why you should think twice before doing so.

There's a third source of confusion: "HTTPS proxy" is also used to describe the *proxy's own* protocol. A proxy whose endpoint is `https://proxy.example.com:8443` is a proxy you reach over TLS. A proxy at `http://proxy.example.com:8080` that tunnels your HTTPS traffic is still, in everyday usage, "an HTTPS proxy." The first is rare outside corporate setups; the second is what you'll actually be configuring.

### Which proxy type is which

| Type | What it understands | Runs on | Typical use |
| --- | --- | --- | --- |
| HTTP proxy | HTTP only, plain text, can read and cache requests | Any IP | Legacy corporate filtering |
| HTTPS proxy | `CONNECT` tunnels, cannot read TLS payload | Any IP | Scrapers, browsers, `curl`, automation |
| SOCKS5 | Any TCP traffic, no HTTP semantics | Any IP | Bots, anti-detect browsers, non-web protocols |
| VPN | Whole-device tunnel at the network layer | Your OS | One IP for everything, no per-request control |

If your tool takes `host:port:user:pass` and you just need browser or HTTP traffic, an HTTPS proxy is the straightforward pick. SOCKS5 shows up when you're routing odd protocols or an anti-detect browser that wants it. A VPN is the wrong tool for anything that needs a different IP per request.

## Where an HTTPS proxy actually earns its keep

Data collection is the obvious one: price monitoring, SERP tracking, public data aggregation. The proxy isn't there to be invisible — it's there so that the request arrives from an address the target site considers normal, in the country you're querying.

Then there's geo-checking, where a country-level IP isn't enough. What a user in Frankfurt sees on a retailer's page is not what a user in Ohio sees, down to pricing and shipping copy. City and ISP-level targeting is the difference between "German traffic" and traffic that looks like it came from a home connection in Munich.

Ad verification and brand-safety work lean on the same thing. So does WAF testing on a professional engagement, where your test traffic needs a clean residential baseline or it gets flagged before it ever reaches the application layer.

Where you don't need one: a single account you manage from one machine, or anything where you want one stable IP across your entire device. That's a VPN's job.

## Setting up an HTTPS proxy without guessing

### Environment variables and curl

The oldest interface is still the one that trips people up. `http_proxy` and `https_proxy` hold the URL of the proxy used for HTTP and HTTPS requests respectively — and here's the part that confuses nearly everyone, `https_proxy` normally contains an `http://` URL, because that's the scheme you use to speak to the proxy, not the scheme of the traffic it carries.


export https_proxy=http://user:pass@proxy.host:8080
curl -v https://example.com          # watch for "CONNECT ... 200 Connection Established"


If you'd rather not export globally, pass it per request:


curl -x http://user:pass@proxy.host:8080 -U user:pass https://example.com


Any tool that respects these variables — `wget`, most Python and Go HTTP clients, `git` — inherits the same behavior.

### Python

python
import requests

proxies = {
    "http":  "http://user:pass@proxy.host:8080",
    "https": "http://user:pass@proxy.host:8080",
}
r = requests.get("https://example.com", proxies=proxies, timeout=30)


Note that both keys point at `http://` URLs. The `https` key means "use this proxy for HTTPS destinations," not "this proxy speaks HTTPS." Getting that backwards produces a certificate error or a hang, depending on the library.

In Node, the equivalent is an agent: `new HttpsProxyAgent('http://proxy.host:8080')` passed into the request options, which also works for WebSocket connections since both go through the same `CONNECT` path.

### Browsers and anti-detect browsers

Manual browser config is manual: host, port, protocol, username, password. In anti-detect browsers like Dolphin{anty}, AdsPower or Hidemyacc you pick the protocol from a dropdown — HTTP, HTTPS or SOCKS5 — and paste in host, port and credentials, then hit the built-in proxy check before saving the profile.

Then verify it worked, from the outside. Load an IP checker and confirm the address that answers is the proxy's, not yours. A proxy that "connects" but still shows your real IP is usually a misconfigured bypass rule, not a broken proxy.

## The five failures you'll actually hit

**407 Proxy Authentication Required.** Credentials are missing, wrong, or not URL-encoded. A `@` or `:` inside your password will silently break the URL form — encode it, or use the separate `-U user:pass` flag.

**403 on the CONNECT request.** The proxy is refusing to tunnel to that host or port. Some proxies block non-standard ports entirely; some block by destination.

**Certificate errors.** If you get one through a proxy that you don't get on a direct connection, something is intercepting the TLS session. That's a terminating proxy with a root cert you haven't trusted — or one you shouldn't trust.

**429 and 403 from the target site itself.** The proxy worked. The IP is burned. Datacenter ranges get caught fast; residential addresses last longer, but a pool that rotates through the same burned addresses repeatedly will keep serving you the same wall.

**Silent timeouts and dead sockets.** Residential IPs are real devices. They go offline. A pool that doesn't actively cycle dead exits will keep handing you IPs that answer nothing, and your success rate tanks without any clear error. This is the failure mode that costs the most engineering time, which is why replacement policy and auto-refresh matter more than the headline price.

## What to check before buying an HTTPS proxy

Protocols first — HTTP/HTTPS plus SOCKS5 covers browsers, scrapers and anti-detect tools without workarounds.

Then authentication. Username/password is portable; IP whitelisting is convenient but breaks the moment your own IP changes or you move to a server.

Then targeting depth. Country-only is enough for streaming checks and useless for local pricing. State, city and ISP-level selection is what makes geo-specific work reliable.

Then IP longevity and session control. Some workflows need one IP to persist across hundreds of requests to hold a logged-in session; others want a new IP every request. Both should be configurable, not fixed at purchase.

Then the economics. Per-IP with unlimited bandwidth makes sense when traffic is unpredictable or heavy. Per-GB makes sense when you rotate constantly and each request is small. Most buyers don't know which one they are until they run one job and watch the numbers.

Last: dead-IP policy. Read it before you pay, because this is where budget providers quietly differ.

## 9Proxy's current plans and prices

9Proxy runs a residential pool advertised at 20M+ IPs across 90+ countries, with targeting down to country, state, city and ISP. It's invite-code sign-up rather than a classic checkout flow, so use the link below rather than navigating in cold.

One thing to know before reading the tables: on **June 1, 2026, 9Proxy raised prices on its IP-based packages and bundle packages**. GB-based pricing was left unchanged. Everything below reflects the post-adjustment rates.

### IP-based packages — unlimited bandwidth per IP

You buy a pool of residential IPs, each with no traffic cap. Unused IPs don't expire until you activate them. IPs themselves stay live somewhere between a few hours and roughly 24 hours.

| Package | Price per IP | Total | Buying link |
| --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | [See 9Proxy's IP-based packages](https://bit.ly/9-Proxy) |
| 500 IPs | $0.144 | $72 | [Check 9Proxy's 500-IP package](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | $0.084 | $126 | [Grab the 1,500-IP package with bonus IPs](https://bit.ly/9-Proxy) |
| 2,500 IPs | $0.084 | $210 | [Compare 9Proxy's 2,500-IP tier](https://bit.ly/9-Proxy) |
| 5,000 IPs | $0.072 | $360 | [Check the 5,000-IP package](https://bit.ly/9-Proxy) |
| 15,000 IPs | $0.048 | $720 | [See 9Proxy's 15,000-IP pricing](https://bit.ly/9-Proxy) |
| 25,000 IPs | $0.035 | $863 | [View the 25,000-IP plan](https://bit.ly/9-Proxy) |
| 50,000 IPs | $0.029 | $1,438 | [Check 9Proxy's 50,000-IP package](https://bit.ly/9-Proxy) |
| 100,000 IPs (business) | $0.023 | $2,300 | [See business-scale IP pricing](https://bit.ly/9-Proxy) |
| 200,000 IPs (business) | $0.021 | $4,140 | [Review the 200,000-IP package](https://bit.ly/9-Proxy) |
| 500,000 IPs (business) | $0.018 | $8,625 | [Check enterprise IP volume pricing](https://bit.ly/9-Proxy) |

The Sweet spot for most solo workloads is the 1,000 + 500 tier: it drags the effective price from $0.24 down to $0.084 per IP, and since IPs don't expire until activated, buying the bigger bucket isn't a bet on using it all this month.

### GB-based packages — pay for traffic, rotate freely

Here you buy bandwidth and generate unlimited proxy endpoints against it. Better fit when each request is small but you churn through IPs constantly: SERP checks, geo-probes, lightweight API polling.

| Package | Price per GB | Total | Validity | Buying link |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [Start with a 5 GB 9Proxy package](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [See the 55 GB package](https://bit.ly/9-Proxy) |
| 100 GB | $1.50 | $150 | 180 days | [Check 9Proxy's 100 GB pricing](https://bit.ly/9-Proxy) |
| 200 GB | $1.00 | $200 | 180 days | [View the 200 GB package](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80 | $800 | 180 days | [Compare the 1,000 GB tier](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [See 9Proxy's 2,000 GB package](https://bit.ly/9-Proxy) |
| 3,000 GB (enterprise) | $0.72 | $2,160 | No expiry | [Check enterprise GB pricing](https://bit.ly/9-Proxy) |
| 6,000 GB (enterprise) | $0.70 | $4,200 | No expiry | [Review the 6,000 GB plan](https://bit.ly/9-Proxy) |
| 10,000 GB (enterprise) | $0.68 | $6,800 | No expiry | [See the 10,000 GB package](https://bit.ly/9-Proxy) |

The 180-day clock is the detail people miss. Standard GB packages expire; enterprise ones don't. If your project runs in bursts with long gaps, the expiry matters more than the per-GB rate.

### Bundle packages — IPs and bandwidth together

| Bundle | Contents | Price | Buying link |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Check the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [See the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | [Check the Pro bundle](https://bit.ly/9-Proxy) |

For a mixed workload — stable sessions on some tasks, rotation on others — the Popular bundle undercuts buying the IP tier and the GB tier separately. Traffic in bundles is valid for 180 days.

Payment-wise, 9Proxy takes cards, Apple Pay, Google Pay, Alipay and crypto via CoinPayments (BTC, ETH, LTC, TRX, USDT-TRC20/ERC20, DOGE, DAI, BCH). Crypto payments carry an automatic 5% IP bonus.

## How 9Proxy handles the HTTPS side specifically

Protocol support is HTTP/HTTPS and SOCKS5, so `CONNECT` tunnelling on 443 works the way you'd expect with `curl`, Python, Scrapy and any anti-detect browser.

The two product models authenticate differently, which matters when you're wiring things up. GB-based proxies take username/password or IP whitelisting, and targeting lives inside a structured username string:


<subaccount>-country-<country_code>-st-<state_code>-city-<city_code>-isp-<isp_code>-ssid-<session_id>-sst-<session_time>


A real example from their docs: `useruser123-country-US-ssid-rhdN1907ma`. Session ID controls stickiness, so rotating versus sticky is a matter of what you put in the username rather than a dashboard toggle.

IP-based proxies work differently: they forward through 9Proxy's desktop app on local ports (`127.0.0.1:17521`, for instance), with a browser-based alternative called Proxy2Web for machines where installing the app isn't practical. Mobile device setup isn't supported, which is worth knowing if your workflow assumes a phone.

Two operational features matter more than they sound. **Today List** lets you reuse an IP you already forwarded in the last 24 hours without spending another one from your balance. **Auto Refresh** detects offline IPs and replaces them, and 9Proxy's published policy covers replacement for IPs that die within about 60 seconds — narrow, but it exists, and most providers with cheap residential pools don't offer one at all.

On performance, independent testing is where you should look rather than the pool-size number. Geekflare ran 300 sequential requests through rotating residential IPs against a major e-commerce site behind Cloudflare: 293 passed (97.7%), 5 hit CAPTCHAs, 2 were hard-blocked, average response time 0.63 seconds. ProxyLook's directory listing puts average response closer to 1,300 ms with a 97% success rate — different test conditions, same order of magnitude. Residential routing is slower than datacenter by nature. Success rate is the number that decides whether the job finishes.

### Where it falls short

- **No self-serve free trial.** Trials have shown up periodically through promotions and support requests, and availability varies. You're largely buying before you validate.
- **90+ countries, not "everywhere."** Deep coverage in the US, Canada and parts of Asia; thinner in less common regions.
- **IP lifetime is variable.** Hours to roughly 24 hours on IP-based plans. Plan for it with auto-refresh rather than assuming stability.
- **The refund window is tight.** The published credit terms cover IPs that die within about 60 seconds, not "this IP underperformed for my use case."
- **No browser extension.** Proxy2Web covers the browser case; if your workflow is built around an extension, you'll be adapting it.

## FAQ

**Is an HTTPS proxy the same as a VPN?**
No. A VPN tunnels your entire device through one IP at the network layer. An HTTPS proxy is per-application or per-request, and you can change the exit IP between requests. Different tools for different problems, and running both usually causes routing conflicts.

**Do I need residential IPs, or will datacenter do?**
If your target is unprotected or lightly protected, datacenter IPs are cheaper and faster. Against sites with bot detection, residential addresses from the target's own city are what keep requests from being challenged in the first place. The block rate difference is not subtle.

**How much does this actually cost?**
Entry-level residential IPs from 9Proxy start at $24 for 100 IPs ($0.24 each) and fall to roughly $0.018–0.029 per IP at high volume. Bandwidth pricing runs from $3.00 per GB on small packages down to $0.68 per GB on the enterprise tiers. 👉 [Check 9Proxy's live pricing and sign-up](https://bit.ly/9-Proxy)

**Can I use 9Proxy with `curl`, Python and anti-detect browsers?**
Yes. GB-based proxies accept `host:port:username:password` with conditional targeting in the username; IP-based proxies forward through the desktop app or Proxy2Web. Documented integrations cover Dolphin{anty}, AdsPower, Hidemaya/Hidemyacc, ixBrowser, Multilogin and Proxifier.

**Is there a free trial?**
Not as a self-serve button on the site. Trial IPs have been distributed through promotions and support requests, so it's worth asking before you buy a large package.
