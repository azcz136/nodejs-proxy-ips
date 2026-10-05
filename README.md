# nodejs proxy: Set Up Rotating Residential IPs in axios, fetch, and Playwright Without Leaking Your Real One

Most tutorials about using a proxy in Node.js stop at the config object. That's the easy part. The part that eats a weekend is everything around it: authentication that survives a password with special characters, figuring out which layer of Node actually owns proxy routing in the version you're running, and working out why a script burns gigabytes against a target that only ever answers with 403.

Here's the setup end to end, using DataImpulse as the concrete provider, since a working proxy integration always has to match the provider's own details: a shared gateway, targeting written into the username, different ports for rotating and sticky sessions.

## Four places a Node.js proxy quietly falls back to your real IP

1. **You set `proxy` in axios and also pass an agent.** Axios then has two routing instructions. The explicit agent usually wins, but if you didn't set `proxy: false`, you've left two systems claiming the same request.
2. **You rely on `HTTP_PROXY` with native `fetch`.** Node's global fetch is powered by undici and ignores those environment variables unless you opt in, which is why a script that works fine with `curl` suddenly exposes your home IP.
3. **`NO_PROXY` matches more than you intended.** A trailing wildcard or a hostname substring can disable the proxy for the exact target you cared about, silently.
4. **The retry path builds a fresh client.** Rotation logic that reconstructs an axios instance on failure is a common culprit: the new instance never got your proxy config.

The cheap way to catch all four: send one request to an IP echo endpoint (`https://api.ipify.org?format=json`, `https://httpbin.io/ip`) at the start of a run and log the answer. If it isn't the proxy's exit IP, nothing downstream matters.

## What a DataImpulse endpoint looks like before you write any code

One host serves the traffic: `gw.dataimpulse.com`. Two rotating ports, 823 for HTTP/HTTPS and 824 for SOCKS5, plus a sticky range that runs from 10000 to 20000. Credentials live in the dashboard under your plan, in a "Proxy Access" section, and you can either authenticate with login and password or whitelist your own IP and skip credentials entirely.

Targeting doesn't get its own dashboard field per request. It rides in the username. Two underscores start the parameters, semicolons separate them, and the port has to match the protocol you chose:

bash
curl -x "http://login__cr.au;sessid.123:password@gw.dataimpulse.com:823" https://api.ipify.org/


`cr.au` pins the exit to Australia; `sessid.123` labels an IP you want to keep coming back to. The equivalent in Node is just string building, which is convenient once you get past the slightly odd format: country targeting is included in the base rate, city, ZIP, and ASN are listed as paid extras.

👉 Grab the gateway credentials from the DataImpulse dashboard before you touch the client code.

## axios: the built-in proxy option is usually enough

For a single upstream, axios handles HTTP and HTTPS proxies without extra packages. HTTPS targets get tunneled through CONNECT, so the proxy never sees decrypted traffic.

js
import axios from "axios";

const client = axios.create({
  proxy: {
    protocol: "http",
    host: "gw.dataimpulse.com",
    port: 823,
    auth: {
      username: "LOGIN__cr.us",
      password: process.env.DI_PASSWORD,
    },
  },
  timeout: 20000,
});

const { data } = await client.get("https://api.ipify.org?format=json");
console.log(data); // the proxy's exit IP, not yours


Two behaviours worth knowing. Axios resolves `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` through its `proxy-from-env` dependency when you *haven't* set an explicit proxy object, so setting both is how people end up unable to explain which route a request took. And the `proxy` field is a forward-proxy instruction, not a "send everything through here" switch, so anything that bypasses it (a different client, a raw socket, a worker with its own client) is on you to catch.

### When an explicit agent is the better call

Native proxy config is designed for one fixed proxy. The moment you want per-request control, a SOCKS5 route, or a rotation pool, use `https-proxy-agent` and turn off axios's own handling:

js
import axios from "axios";
import { HttpsProxyAgent } from "https-proxy-agent";

const user = encodeURIComponent("LOGIN__cr.us;sessid.script01");
const pass = encodeURIComponent(process.env.DI_PASSWORD);
const agent = new HttpsProxyAgent(
  `http://${user}:${pass}@gw.dataimpulse.com:823`
);

await axios.get(target, { httpsAgent: agent, proxy: false });


`encodeURIComponent` isn't decoration. Proxy passwords commonly contain characters that break URL parsing, and a mangled password surfaces as a 407 that looks like an account problem when it's a string problem. Keeping credentials in environment variables also keeps them out of your git history and out of crash logs.

## Node's built-in fetch needs an explicit dispatcher

Native `fetch` takes `dispatcher`, not `agent`. Install undici and hand it a `ProxyAgent`:

js
import { ProxyAgent } from "undici";

const dispatcher = new ProxyAgent(
  "http://LOGIN__cr.us:PASSWORD@gw.dataimpulse.com:823"
);

const res = await fetch("https://api.ipify.org?format=json", { dispatcher });
console.log(await res.json());


If you're still on `node-fetch` v3 instead, the option is named `agent` and you use `HttpProxyAgent` from the proxy-agents project. `node-fetch` v3 is ESM-only, so `"type": "module"` has to be in your `package.json` or the import fails before the proxy ever gets a chance.

There's also a newer environment-variable path. Node added built-in env-var proxy support behind `NODE_USE_ENV_PROXY=1` (or the `--use-env-proxy` flag) starting in v24.0.0, backported to v22.21.0. If you're on a mid-range Node 22 patch release, the flag simply doesn't exist, which is worth checking before you debug anything else.


NODE_USE_ENV_PROXY=1 node scraper.js


Pick one routing system and stick to it. Enabling the flag while also configuring axios's `proxy` object means two mechanisms can decide the same request's route, and the failure mode is confusing rather than loud.

## SOCKS5, and why port 824 exists

js
import { SocksProxyAgent } from "socks-proxy-agent";

const agent = new SocksProxyAgent(
  "socks5://LOGIN__cr.us:PASSWORD@gw.dataimpulse.com:824"
);


SOCKS tunnels at the TCP level rather than speaking HTTP proxy semantics, which is handy when a client library only exposes SOCKS, or when you're routing non-HTTP traffic alongside your scraping code. Same credentials, same targeting syntax, different port. Mixing them up returns a connection failure that looks like the proxy is down.

## Rotating versus sticky, in code

Rotating sessions hand you a new exit IP on each request: good for high-volume collection where every request stands alone. Sticky sessions hold one IP for a window, which is what you want for logins, carts, and multi-step forms where a mid-flow IP change reads as suspicious.

DataImpulse lets you specify a rotation interval of up to 120 minutes, with 30 minutes as the default when nothing is set. That 30 minutes is an expectation rather than a guarantee, and the reason is structural: residential IPs belong to real devices, so if the underlying device goes offline the session rotates no matter what you asked for. DataImpulse support put the average sticky duration at around 30 minutes and confirmed the interval can't be guaranteed.

The pragmatic version, then, is to treat sticky as "probably stable for a few minutes" and build your retry logic accordingly:

js
function proxyUser({ country, session }) {
  const tags = [
    country ? `cr.${country}` : null,
    session ? `sessid.${session}` : null,
  ].filter(Boolean);

  return tags.length
    ? `${process.env.DI_LOGIN}__${tags.join(";")}`
    : process.env.DI_LOGIN;
}


Two rules keep rotation from turning a bad request into a cascade. Retry only idempotent methods (a replayable GET is safe; a POST that already hit the origin is not), and cap the retry budget instead of looping until something works. A proxy that returns nothing useful is usually telling you the target's policy, not the proxy's health.

## The bill is bandwidth, so watch what you actually download

Residential traffic on DataImpulse runs at $1 per GB with no subscription, which makes the useful mental model a per-response one. A 200 KB JSON response costs roughly $0.0002. The $5 entry plan covers about 25,000 responses of that size, and unused balance doesn't expire, so a scraper that runs twice a month isn't paying for the weeks it sits idle.

The bandwidth mistakes are boring and expensive:

- Fetch JSON endpoints instead of full HTML pages when the site offers them.
- Block images, fonts, and media in Playwright or Puppeteer, and skip rendering entirely when a plain request returns the data.
- Send `Accept-Encoding: gzip` and actually decompress what comes back.
- Don't log full response bodies at debug level in production, or your log pipeline becomes the largest consumer.

The dashboard's usage table breaks traffic down by site and by minute, which is more granularity than most providers offer and the fastest way to find the endpoint quietly eating your balance.

One pricing trap specific to proxies with targeting: DataImpulse's own comparison pages list city, ZIP, and ASN as paid extras while country targeting is free, and third-party reviews report that residential traffic routed through advanced targeting filters is billed at double the base rate. A stray `city.newyork` tag in a username can therefore double your effective cost per gigabyte. Confirm the current treatment with support before you budget around it.

## Full plan comparison

DataImpulse doesn't sell fixed monthly packages. Every product is pay-as-you-go, and the prices below are the published tiers and volume rates across all four proxy types.

| Proxy type | Entry point | Standard rate | Volume tiers | Billing | Plan page |
| --- | --- | --- | --- | --- | --- |
| Residential (90M+ IPs, 195 countries) | $5 for 5 GB | $1.00/GB | $800 / 1 TB ($0.80/GB); $0.70/GB at 5 TB | Pay as you go, traffic never expires | [Get the residential plan](https://bit.ly/dataimPulse) |
| Datacenter | $5 for 10 GB | $0.50/GB | $50 / 100 GB; $450 / 1 TB ($0.45/GB); custom from $2,250 for 5 TB+ | Pay as you go, traffic never expires | [Get the datacenter plan](https://bit.ly/dataimPulse) |
| Mobile (3G/4G/5G/LTE) | $5 for 2.5 GB | $2.00/GB | $50 / 25 GB; $1,600 / 1 TB ($1.60/GB); custom from $8,000 for 5 TB+ | Pay as you go, traffic never expires | [Get the mobile plan](https://bit.ly/dataimPulse) |
| Premium residential | $5 minimum top-up | $5.00/GB | Volume discount starts at 1 TB+ | Pay as you go, traffic never expires | [See premium residential plans](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

Three conditions sit on top of that table and change the real entry cost:

- **The $5 minimum applies per first purchase of each proxy type.** The intro offer can be used again when you switch to a different proxy type, but only on a first purchase.
- **Subsequent top-ups have a $50 minimum.** Buying another plan of the same type, or adding funds to an existing one, requires at least $50. Testing for $5 is easy; running at $5 a month isn't a long-term shape.
- **Payments go through card (Stripe) or cryptocurrency (Cryptomus, supporting USDT, Bitcoin, Ethereum, Litecoin).** There's no free trial, so every route into the service starts with that $5.

If you want the refund terms before committing: a review of the platform by HostAdvice notes a 7-day money-back guarantee on intro plans for card payments when less than 80% of the traffic has been consumed, with crypto purchases on intro plans non-refundable.

## Where DataImpulse fits a Node.js project, and where it doesn't

It fits scripts with uneven usage. If your crawler runs hard for three days and then sits idle for a month, per-GB pricing with non-expiring balance beats a monthly bundle you only half use. Same for anything under roughly 50 GB a month, where subscription providers are charging you for a quota you won't finish. Country-level targeting inside the username string also suits request-level routing, since you can vary the exit country per call without touching dashboard settings.

It doesn't fit everything. There's no static ISP proxy product, and no fully managed scraping API, so if you want someone else to own the browser fingerprinting problem, this isn't it. Rotating residential IPs alone won't get you past a Cloudflare challenge; that needs a rendering and fingerprint strategy on top. DataImpulse's own material also frames the service as unsuitable for banking and government targets. And if your workflow needs a guaranteed multi-hour sticky session, residential IPs sourced from real devices can't promise that.

For scale expectations: DataImpulse publishes a 90M+ first-party IP pool across 195 countries, a 99.51% success rate, and a 4.8/5 rating on G2. The first-party detail is the one that matters for scraping, since pools that resell other networks carry more shared abuse history, which shows up as blocks on high-security sites.

## Checklist before you deploy

1. Confirm your exit IP with an echo endpoint on the first request of every run.
2. Pass credentials as `username`/`password` fields rather than inline URLs, or URL-encode both parts if you must inline them.
3. Set `proxy: false` whenever you attach an explicit agent.
4. Pick one routing layer: axios config, agent, or Node's env-proxy flag, never two.
5. Keep requests to advanced targeting only where the task needs it, given the reported 2× billing.
6. Retry GETs with a hard cap, never replay POSTs blindly.
7. Track your own byte counts and reconcile them against the dashboard's per-site usage.
8. Store the login and password in environment variables, and rotate them if a log ever printed them.

## Quick answers

**Do I need a proxy library for Node.js?** For HTTP/HTTPS with axios, no. For SOCKS5, or for native `fetch`, yes: `socks-proxy-agent`, or undici's `ProxyAgent` as a dispatcher.

**Why does my request still show my real IP?** Almost always routing, not credentials. Check `NO_PROXY`, check whether `proxy: false` is set where it shouldn't be, and check whether the retry path built a client without your config.

**Can I test this for $5?** Yes. The minimum payment is $5, which buys 5 GB of residential, 10 GB of datacenter, or 2.5 GB of mobile traffic, and it doesn't expire. Just remember the $50 minimum sitting behind any follow-up top-up.

The code above is genuinely the short part. What takes longer is deciding whether your workload wants per-request rotation or a sticky window, and then measuring bytes so the $1 per GB stays a line item you can predict instead of a surprise.
