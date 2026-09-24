# Analytics & Third-Party Scripts (`@nuxt/scripts`, GA4)

Read before adding an analytics vendor, touching `scripts.registry` in `nuxt.config.ts`, or diagnosing numbers in an analytics dashboard that look wrong.

---

## 1. Know the path a hit takes — before anything else

A tracking hit either goes **browser → vendor** or **browser → our Nitro server → vendor**. Every analytics diagnosis starts by establishing which, because the second path changes what the vendor sees.

`@nuxt/scripts` 1.x **proxies registry scripts through Nitro by default.** For `googleAnalytics` the registry ceiling is `{ bundle: true, proxy: true }` with IP anonymisation on, and the resolved default is the ceiling — so unless `proxy: false` is set, the proxy is on. Verify against the installed version, not memory:

```bash
# capabilities per script, and which keys the registry accepts as script options
grep -n 'm("googleAnalytics"' node_modules/@nuxt/scripts/dist/registry*.mjs
grep -n "SCRIPT_OPTION_KEYS =" node_modules/@nuxt/scripts/dist/module.mjs
```

Cheapest runtime check, for the developer: live site → DevTools → Network → filter `collect`. Request host is our own domain → proxied. `region1.google-analytics.com` (or similar) → direct.

---

## 2. Proxied analytics geolocates to the server

The vendor geolocates the IP that **connects** to it. Through the proxy, that is the production server. `@nuxt/scripts` does forward an anonymised `x-forwarded-for`, but GA4's browser collection endpoint does not use it — observed in production: every visitor was attributed to one country where the host's IP is registered. (An anonymised /24 would still resolve to the right country, so a wrong country means the vendor is reading the connecting IP, not the forwarded one.)

**Signature:**

| Signal | Proxy geolocation | Ghost spam | Real bot on our pages |
| :--- | :--- | :--- | :--- |
| Country spread | Collapses to 1–2 countries, one implausible | Adds a country on top of a normal spread | Adds a country on top of a normal spread |
| Hostname (GA4 Explore) | Our domain | Not ours, or `(not set)` | Our domain |
| Sources | Real: google, bing, linkedin, chatgpt.com | Junk or `(direct)` | Mostly `(direct)` |
| Engagement | Normal | ~0s | ~0s |
| Pages | Real pages, real locales | Often none / fake | Crawl-shaped |
| vs Search Console geography | Disagrees wholesale | Agrees apart from the spam country | Agrees apart from the bot country |

The trap is the first two columns look alike on a country report. **Do not diagnose spam or bots from a country chart alone.** Sessions-by-source and hostname separate them in one look.

Dev runs make it worse: in dev the proxy runs on the developer's machine, so dev hits geolocate to the developer's country and land in the same property. Keep dev traffic out of the production property (a separate measurement ID, or no ID outside production).

---

## 3. Fix — send hits direct

Per script, in `scripts.registry`. `proxy` is a script option in the object form:

```ts
// nuxt.config.ts
scripts: {
  registry: {
    googleAnalytics: {
      id: process.env.NUXT_PUBLIC_SCRIPTS_GOOGLE_ANALYTICS_ID,
      trigger: 'server',
      // proxied hits geolocate to the server's IP, not the visitor's
      proxy: false,
    },
  },
},
```

- **CSP:** direct hits need `*.google-analytics.com` (and `*.googletagmanager.com` for the loader) in `connect-src` / `script-src`. Check before deploying — a CSP block fails silently in analytics.
- **Verify after deploy:** Network tab shows the vendor host (§1), then GA4 Realtime shows a normal country spread within minutes.
- **Every analytics entry states `proxy` explicitly**, with a one-line why. It is a data-correctness decision, not a default to inherit.

**When first-party collection is genuinely required** (ad-blocker resilience, consent posture): keep the proxy only for a vendor that documents honouring a forwarded client IP. Plausible's proxy guide does. For GA4 the supported route is server-side GTM, not the `@nuxt/scripts` proxy. Say this plainly rather than shipping a proxy that corrupts geography.

---

## 4. What is damaged, and what is not

Tell the developer this in plain terms — they will be asked by someone non-technical.

- **GA4 never reprocesses.** Data collected through the proxy keeps the wrong country forever. The fix is forward-only. Suggest an annotation on the fix date so nobody compares across it.
- **Wrong:** country / region / city reports, key events or conversions by geography, audiences built on location, Ads geo-targeting if the property is linked.
- **Still usable:** user and session counts, sources, pages, engagement.
- **Leads are fine** when forms post to our own API (`email-pdf.md` §1) — GA is a reporter, not the system of record. If conversions are *imported* from GA into Ads, flag that the geo on those is wrong.

---

## 5. Diagnosing any analytics discrepancy

Same Diagnose-mode discipline as any bug (`SKILL.md`). Two extra rules:

1. **Dashboards are readers.** A CMS dashboard plugin (e.g. `strapi-google-analytics-dashboard`) runs a plain GA4 Data API query and shows what the property holds. Do not debug the plugin; check the query has no filters and move on to the property.
2. **Sources that count different things never match.** Search Console = Google Search clicks, 28-day default. GA4 = all users, whatever window the report uses, only since the tag went live. Name the mismatch in units and window before calling anything missing.

The opening pair is almost always: *"Either the traffic is fake (spam / bot), or our pipeline is rewriting real traffic (proxy, dev hits, consent mode, a filter)."* The cheapest separator is the developer reading Sessions by source and Hostname for the suspect segment — ask for that before theorising further.
