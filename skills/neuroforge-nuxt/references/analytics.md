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

## 3. Registering a vendor — the canonical entry

This is both the setup for a new vendor and the fix for §2:

```ts
// nuxt.config.ts
$production: {
  scripts: {
    registry: {
      googleAnalytics: {
        trigger: 'onNuxtReady',
        // proxied hits geolocate to the server's IP, not the visitor's
        proxy: false,
      },
    },
  },
},
```

Every line is a decision:

- **Under `$production`.** Dev never loads the script, so the developer's own visits never reach the live property (§2). Test a real hit with `nuxt build && nuxt preview` and the env var set.
- **`trigger` is explicit.** `onNuxtReady` loads after the page is interactive, which keeps analytics off the critical path. Verified in 1.3.9: a registry entry with no trigger **never loads at all**. The module won't warn you, and it looks exactly like a working install with zero traffic.
- **`proxy: false`, with a one-line why.** This is a data-correctness decision, so don't inherit it from the default.
- **No `id` in the config, and no `runtimeConfig` block.** For registry scripts with env defaults, the module reads `NUXT_PUBLIC_SCRIPTS_<KEY>_<FIELD>` (e.g. `NUXT_PUBLIC_SCRIPTS_GOOGLE_ANALYTICS_ID`) and seeds `runtimeConfig.public.scripts.<key>` itself. Because that key exists, Nuxt overrides it from the same env var when the server starts. The ID is set at deploy time, not baked into the build.

Do not add this by hand. It is a smell:

```ts
// ❌ duplicates what the module already writes
runtimeConfig: { public: { scripts: { googleAnalytics: { id: process.env.NUXT_PUBLIC_SCRIPTS_GOOGLE_ANALYTICS_ID || '' } } } }
```

It puts the ID in two places to keep in sync. The `|| ''` hides a missing production ID: GA silently doesn't load (hard stop 7's logic applies to config too, see `smells.md` §3). And it sits outside `$production`, so dev carries an ID nothing uses. The same goes for `id: process.env.X` inside the registry entry, which is redundant and bakes the build-time value in as the default.

Verify against the installed version before relying on this:

```bash
grep -n "envDefaults\|NUXT_PUBLIC_SCRIPTS_" node_modules/@nuxt/scripts/dist/module.mjs | head
```

**Expected dev warning:** *"NUXT_PUBLIC_SCRIPTS_GOOGLE_ANALYTICS_ID is set but googleAnalytics is not registered"* appears when the local `.env` has the ID and the entry lives under `$production`. It's harmless. Clear the ID from the local `.env` to silence it. Don't move the registry entry out of `$production`.

**Before deploying:**

- **Env:** the real `G-…` ID is set in the production environment as `NUXT_PUBLIC_SCRIPTS_GOOGLE_ANALYTICS_ID`.
- **CSP:** direct hits need `*.google-analytics.com` (and `*.googletagmanager.com` for the loader) in `connect-src` / `script-src`. A CSP block fails silently in analytics.

**After deploying:** Network → filter `collect` shows requests to the vendor host carrying the `G-…` ID (§1), and GA4 Realtime shows the visit within a minute with a plausible country.

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

**No `collect` request at all** on the live site is a different problem. The script never ran. Check the Console first: an app crash before `onNuxtReady` means the trigger never fires, and the only clue is a "gtag … preloaded but not used" warning (`debugging.md` §7).

---

## 6. Strapi GA dashboard plugin — setup

For `strapi-google-analytics-dashboard` or similar. The plugin passes the credentials straight to Google's `BetaAnalyticsDataClient`. Its settings need three values that are easy to mix up:

| Field | Value | Where from |
| :--- | :--- | :--- |
| Property ID | A **number**, not `G-…` | GA4 → Admin → Property details, or the `p123456789` in the GA URL |
| Measurement ID | `G-…` | The web data stream |
| Credentials | The **whole** service-account JSON key, pasted unedited (`\n` inside `private_key` stays as is) | Google Cloud → IAM → Service accounts → Keys → JSON |

Prerequisites, in order:
1. Enable the **Google Analytics Data API** in the Cloud project.
2. Create the service account. It needs no Cloud roles.
3. Add its email in **GA4** → Property access management as **Viewer**. Skipping this step is the usual cause of an "invalid credentials" error.
4. Rebuild the Strapi admin after installing (`npm run build`), because the plugin adds admin pages.

Reading the result:

- **"Invalid credentials or property ID"**: Viewer access is missing or hasn't propagated yet (it can take minutes), or `G-…` was pasted as the Property ID.
- **"No data"**: the credentials were **accepted**. Either the site isn't sending hits yet (§3, after deploying), or GA's standard reports haven't caught up, which takes 24–48 h. Realtime confirms hits long before the plugin does.

**The key file is a password.** Never committed, never pasted into chat, deleted locally once it is saved in Strapi. Restrict Strapi Settings to Super Admins, because anyone with Settings access can read the key.
