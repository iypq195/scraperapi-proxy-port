# ScraperAPI Proxy Port: What Host, Port and Password Do You Use, and Why Do Puppeteer and Selenium Setups Break on the API Endpoint?

The ScraperAPI proxy port is `proxy-server.scraperapi.com:8001`, the username is `scraperapi`, and the password is your API key. The full connection string is `http://scraperapi:APIKEY@proxy-server.scraperapi.com:8001`, and that single line drops into curl, Axios, Python Requests, Scrapy or a browser's proxy settings without changing anything else in your code.

If you're driving Puppeteer, Playwright or Selenium, put that proxy on the browser and keep requests pointed at the real target site. Posting URLs to `api.scraperapi.com` from a headless browser breaks page assets and link clicks, which is why ScraperAPI's own browser guides list the endpoint method as not recommended for those tools.

What follows: the four fields and where each one goes, configs you can paste, the two ways to handle SSL, the username parameter syntax most tutorials skip, and the limits worth checking before you pay for a tier.

> **Quick pick:** start on the smallest paid tier that covers your monthly credit burn and move up when you hit the ceiling. New accounts get 5,000 free credits with no card required, and the free tier after that is 1,000 credits a month with 5 concurrent connections, which is enough to prove your proxy config works but not enough to run a pipeline.
> 👉 [Check the current plan rates and claim the 5,000 free credits](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The connection string, field by field

Four values, and only one of them is something you look up.

| Field | Value | Where it goes |
| --- | --- | --- |
| Host | `proxy-server.scraperapi.com` | proxy host setting |
| Port | `8001` | proxy port setting |
| Username | `scraperapi` | proxy auth username |
| Password | your API key | proxy auth password |

The password is not a separate proxy credential. It's the same API key from your dashboard that you'd otherwise pass as `api_key` in a query string, so there's no second secret to rotate.

Here it is in curl, which is the fastest way to confirm the credentials work before you touch a browser:

bash
curl --proxy 'http://scraperapi:API_KEY@proxy-server.scraperapi.com:8001' \
  -k \
  'https://www.example.com'


The `-k` flag skips certificate verification, and ScraperAPI's docs state that your code must be configured this way for requests to be forwarded properly. That's the quick path. There's a proper path too, covered below.

ScraperAPI describes the proxy as a front-end that takes your request, forwards it to the API, and handles proxy rotation, CAPTCHA solving and retries on their side, with the same features and performance as the API endpoint. Practically, that means you don't lose anything by routing through port 8001 instead of the REST endpoint.

## Axios: four fields, no query string

The Node config maps one-to-one onto the table above:

js
import axios from 'axios';

axios.get('https://www.example.com', {
  proxy: {
    host: 'proxy-server.scraperapi.com',
    port: 8001,
    auth: {
      username: 'scraperapi',
      password: 'API_KEY'   // your API key, not a proxy password
    },
    protocol: 'http'
  }
});


The `protocol` field has to be `http` even when the target URL is HTTPS. The tunnel to the proxy is plain HTTP; TLS to the destination happens inside it.

## Puppeteer: proxy on the browser, credentials on the page

This is where most integrations fail. The proxy address goes in the launch arguments. The username and password go on the page, in a separate call. Putting credentials inside the `--proxy-server` string doesn't authenticate anything.

js
const browser = await puppeteer.launch({
  ignoreHTTPSErrors: true,
  args: ['--proxy-server=http://proxy-server.scraperapi.com:8001']
});

const page = await browser.newPage();
await page.authenticate({
  username: 'scraperapi',
  password: 'API_KEY'
});

await page.goto('http://quotes.toscrape.com/page/1/', { timeout: 180000 });


Two details in that snippet are worth copying exactly. `page.authenticate()` has to run before your first navigation, otherwise the initial request goes out unauthenticated and you get a 407. And the official example uses a 180-second navigation timeout for a page that renders normally in a browser, because requests passing through rendering and anti-bot logic take far longer than a plain HTTP fetch. If your script inherits Puppeteer's 30-second default, slow targets will time out before ScraperAPI finishes.

ScraperAPI also notes that occasional conflicts happen between Puppeteer and the proxy port, typically showing up as certain resources or assets not loading. Their advice is to contact support with the specifics rather than assume the config is wrong. It's an acknowledged edge case, not a routine failure.

👉 [Run your config against the 5,000 free credits before scaling](https://www.scraperapi.com/?fp_ref=coupons)

## Selenium: selenium-wire, and don't forget `no_proxy`

Vanilla Selenium can be configured with a proxy, but the official guide recommends Selenium Wire instead, because it exposes request-level control that plain Selenium hides.

bash
pip install selenium-wire


python
API_KEY = 'YOUR_API_KEY'

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=option,
    seleniumwire_options=proxy_options
)


Both `http` and `https` keys need the same value, and `no_proxy` should exclude your local addresses. Skip it and local test servers, driver health checks and anything on localhost get routed out through the proxy, which fails in confusing ways that look like a credential problem.

## Why the API endpoint fails in a headless browser

Sending a GET to `api.scraperapi.com` returns the HTML you asked for, so the first page appears to work. The breakage shows up on everything the page requests next.

ScraperAPI's example: a browser pointed at `quotes.toscrape.com` needs `/static/main.css`. Normally that resolves to `http://quotes.toscrape.com/static/main.css`. Through the API endpoint it resolves to `http://api.scraperapi.com/static/main.css`, which returns an error. The browser keeps the first HTML response and loses every stylesheet, script and image after it, which defeats the point of running a real browser in the first place. Clicking a relative link has the same outcome, so any script that navigates or mimics user behaviour stalls.

With the proxy port, the browser still believes it's talking to the target domain. Relative paths resolve correctly, assets load, and the proxy layer handles rotation underneath.

## SSL: disable verification, or install the proxy CA

ScraperAPI intercepts traffic to strip and rotate proxies, so its certificate doesn't chain to a public root by default. You have two ways to deal with that, and they cost different things.

**Fast route.** Turn verification off in the client: `-k` in curl, `ignoreHTTPSErrors: true` in Puppeteer, `verify=False` in Requests. It takes one line and works immediately. The cost is that you lose certificate validation for every HTTPS request that client makes, including any unrelated API calls in the same process, and the setting tends to survive copy-paste into production.

**Durable route.** Download ScraperAPI's proxy CA certificate and add it to your trust store, then remove the bypass. Per platform:

- **Windows 10/11:** `Win + R`, type `mmc`, then File > Add/Remove Snap-ins > Certificates > Add > Computer account > Local account > Finish > OK. Under Console Root, right-click Trusted Root Certification Authorities > All Tasks > Import and select the certificate file.
- **macOS:** open Keychain Access, select the System keychain, drag the certificate in (or File > Import Items), enter your admin password, double-click the ScraperAPI CA entry, expand Trust and set "When using this certificate" to Always Trust.
- **Linux:** copy the file to `/usr/local/share/ca-certificates/proxyca.pem`, then run `sudo update-ca-certificates`.

My read: local scripts and throwaway crawlers should keep the bypass, because the setup cost buys nothing you'll notice. Anything running in CI or in production alongside calls to APIs you do want to validate should install the CA once and drop the bypass permanently. Mixing the two, with a global "ignore cert errors" flag left on in a shared HTTP client, is the version that eventually bites you.

## Extra API parameters ride on the username

This is the part most tutorials leave out, and it's the one that changes what you can do in proxy mode.

API parameters can be appended to the **username**, separated by dots. Want JavaScript rendering on and traffic exiting from the US?

bash
curl --proxy 'http://scraperapi.render=true.country_code=us:API_KEY@proxy-server.scraperapi.com:8001' \
  -k \
  'https://www.example.com'


In Axios, the same idea:

js
auth: {
  username: 'scraperapi.render=true.country_code=us',
  password: 'API_KEY'
}


Multiple parameters stack in one username, and the base segment `scraperapi` stays first. The consequence worth flagging in code review: the username is no longer the constant `scraperapi`. If you build it dynamically, keep the base segment and the dot separator right, because a malformed username fails as a proxy authentication error rather than as a bad parameter.

Rendering isn't free. A standard page costs 1 credit, Amazon costs 5, Google and Bing cost 25, LinkedIn costs 30, and bypassing bot protection from Cloudflare, DataDome or PerimeterX adds 10 per request. Rendering, premium IPs and screenshots each multiply that further. The dashboard's Domain Cost Estimator gives you the exact charge for a specific URL, and `max_cost` caps any single request so it returns an error instead of silently overcharging.

👉 [Compare tiers once you know your per-request credit cost](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Limits, timeouts and the errors you'll actually hit

**50MB per request.** That's the documented ceiling on request size, and it applies across all access methods, so large PDFs and media files can fail even when the proxy credentials are fine.

**407 Proxy Authentication Required.** The proxy is rejecting your credentials. Almost always the password field holds something other than the API key, or Puppeteer is missing the `page.authenticate()` call. A 401 means the destination site rejected you; a 407 is the proxy in front of you.

**TLS handshake failures.** Either the bypass isn't applied in the right place (a per-request flag won't help if the browser itself validates) or the CA certificate wasn't installed into the store the client actually reads.

**Missing assets in Puppeteer.** Covered above. ScraperAPI's support team asks for specifics when this happens.

**Slow responses.** Rendered and protected pages can take far longer than a plain fetch, which is what the 180-second timeout in the official example is for. Set your own timeouts above the expected worst case rather than at the default, and use the Async API if you're pushing volume and don't need responses inline.

## Which plan actually fits proxy port work

The proxy port is available on every tier, so the decision comes down to credit burn and which features unlock where.

| Plan | What it gives you | Overage behaviour | Fits |
| --- | --- | --- | --- |
| Free | 1,000 credits/month, 5 concurrent connections | None; requests stop at the cap | Proving the config works |
| Hobby | Entry paid allowance | Optional auto-upgrade to the next tier when credits run out | Personal projects, a few thousand standard pages |
| Startup | Larger bundled allowance | Auto-upgrade available | Small teams running a scheduled pipeline |
| Business | Opens country-level targeting beyond the US and EU | Auto-upgrade available | Geo-restricted SERP or retail data |
| Scaling | Higher volume tier | Pay-as-you-go up to a monthly cap you set | Workloads that spike without warning |
| Professional | Higher volume tier | Auto-upgrade plus pay-as-you-go | Agencies juggling several clients |
| Advanced | High volume tier | Pay-as-you-go | Continuous scraping |
| Enterprise | Custom volume, sales contact | Custom pricing; Ultra Premium requests carry their own minimum rate | Large operations with predictable heavy load |

Rates are quoted per plan and per billing cycle, and annual billing changes the effective per-credit price, so each row links to the live pricing page rather than a number that ages badly.

Three rules of thumb from the tier structure:

Static HTML only, no rendering, no premium IPs? The smallest paid tier covers a surprising amount, because standard pages cost 1 credit and a handful of thousands of pages a month fits comfortably.

Turning on `render=true` or premium proxies? Your credit burn multiplies by roughly an order of magnitude per request, so pick a tier with headroom and set `max_cost` before you scale, or one runaway loop eats the month's allowance.

Need country targeting outside the US and EU? That starts at Business. If your whole reason for using ScraperAPI is localized SERP data, factor that in before comparing smaller tiers on price.

Before you check out, confirm two things: whether your workload is mostly standard pages or mostly rendered ones, since that single ratio decides the tier more than anything else, and whether you want annual billing, which lowers the per-credit rate but locks the allowance in for the year.

👉 [Pick the tier that matches your rendering load](https://www.scraperapi.com/pricing/?fp_ref=coupons)
