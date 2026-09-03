# Email & File Delivery (Nodemailer, Nitro, Strapi)

Read before sending a mail, wiring a contact form, building a newsletter, or serving a PDF. Applies to Nitro-only projects and to Strapi-backed ones.

---

## 1. Ownership gate — one app owns SMTP

Strapi ships an email plugin backed by Nodemailer. If it is already configured, **do not add a second set of SMTP credentials to Nuxt.** Two senders means two credential rotations, two suppression lists, two places to look when a mail does not arrive, and two `From` identities landing in spam for different reasons.

| Situation | Sender |
| :--- | :--- |
| Strapi is present and already sends (admin invites, password resets) | **Strapi.** Nuxt calls it server-to-server. |
| The email body is content editors own (templates, campaigns, localised copy) | **Strapi** — the template lives with the content. |
| No Strapi, or the mail is Nuxt-owned (Nitro auth flows, a form that stores nothing) | **Nitro** (§3). |
| Bulk / marketing / anything with a list | **Neither directly** — an ESP (§5). |

Whichever you pick, **credentials live in one app's private env only.** SMTP creds in `runtimeConfig.public`, in a component, or in a client-visible payload are a critical finding — an open relay for anyone who reads the bundle.

---

## 2. Strapi sends, Nuxt calls it

Configure the provider once in `config/plugins.ts` (`strapi-backend.md` §5):

```ts
email: {
  config: {
    provider: 'nodemailer',
    providerOptions: {
      host: env('SMTP_HOST'),
      port: env.int('SMTP_PORT', 587),
      auth: { user: env('SMTP_USER'), pass: env('SMTP_PASS') },
    },
    settings: {
      defaultFrom: env('SMTP_FROM'),
      defaultReplyTo: env('SMTP_REPLY_TO'),
    },
  },
},
```

Then expose a narrow custom route in Strapi (a controller that validates and sends), and call it from **Nitro**, never from the browser:

```ts
// server/api/contact.post.ts
const config = useRuntimeConfig(event)
await $fetch(`${config.cmsUrl}/api/contact-submissions`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${config.strapiToken}` },   // private config only
  body: { data: payload },
})
```

Strapi stores the submission and sends the mail in one lifecycle hook — the record and the notification cannot drift apart. Do not have Nuxt store *and* Strapi send; pick one owner per side effect.

---

## 3. Nitro sends — the transporter singleton

Same lifecycle problem as Prisma (`patterns.md` §3): a transporter created per request opens a new SMTP connection each time and exhausts the provider's connection limit under load, and HMR multiplies it in dev.

```ts
// server/utils/mailer.ts
import nodemailer, { type Transporter } from 'nodemailer'

let transporter: Transporter | undefined

export function useMailer(): Transporter {
  if (transporter) return transporter

  const config = useRuntimeConfig()
  const { smtpHost, smtpPort, smtpUser, smtpPass } = config

  if (!smtpHost || !smtpUser || !smtpPass) {
    throw createError({ statusCode: 500, statusMessage: 'Mail transport is not configured' })
  }

  transporter = nodemailer.createTransport({
    host: smtpHost,
    port: Number(smtpPort) || 587,
    secure: Number(smtpPort) === 465,   // 465 = implicit TLS; 587 = STARTTLS
    auth: { user: smtpUser, pass: smtpPass },
    pool: true,
    maxConnections: 3,
  })

  return transporter
}
```

- **Named `useMailer` but living in `server/utils/`** — Nitro auto-imports it; it is not a Vue composable and never runs on the client. (`smells.md` prefers no `use` prefix for server utils; if the project already uses `useX` in `server/utils/`, match the project.)
- **Lazy, not module-scope.** A transporter created at import time runs during build/prerender, where the env may not exist yet.
- **Fail loudly on missing config** — no `|| 'smtp.gmail.com'` default (`smells.md` §3).
- **`pool: true`** for anything sending more than occasionally; without it every send is a fresh TCP + TLS handshake.
- `nodemailer` is a **server dependency**. Suggest `npm i nodemailer && npm i -D @types/nodemailer`; do not run it unprompted. If it ever appears in a client bundle, an import escaped `server/`.

### The route

```ts
// server/api/contact.post.ts
import { z } from 'zod'

const ContactSchema = z.object({
  name: z.string().min(1).max(120),
  email: z.string().email(),
  message: z.string().min(10).max(5000),
  company: z.string().max(0).optional(),   // honeypot: real users leave it empty
})

export default defineEventHandler(async (event) => {
  const body = await readValidatedBody(event, ContactSchema.parse)

  if (body.company) return { ok: true }    // silently drop bots — never tell them why

  try {
    await useMailer().sendMail({
      from: useRuntimeConfig(event).smtpFrom,   // your domain, always
      replyTo: body.email,                      // the visitor's address goes here
      to: useRuntimeConfig(event).contactInbox,
      subject: `Contact form — ${body.name}`,
      text: body.message,
    })
  }
  catch (error) {
    console.error('[mail] send failed', error)
    throw createError({ statusCode: 502, statusMessage: 'Message could not be sent' })
  }

  return { ok: true }
})
```

- **`from` is your authenticated domain, `replyTo` is the visitor.** Sending `from: visitor@gmail.com` fails DMARC and lands in spam — the single most common contact-form bug.
- **Never interpolate user input into HTML.** Send `text`, or escape every value into a template. A raw `html: \`<p>${body.message}</p>\`` is HTML injection into whatever renders that inbox.
- **Never leak the SMTP error to the client** (`backend-errors.md` §2) — provider hostnames and auth failures are internal. Log it, return a generic 502.
- **Rate-limit.** A contact endpoint with no limit is a free relay for spam through your domain's reputation. `nuxt-security`'s rate limiter, or a per-IP counter in Nitro storage — plus the honeypot, which costs nothing.
- **Never expose the endpoint's payload shape to the client without the same schema.** Share `ContactSchema` from `shared/` so the form and the route validate identically.

**Verify the SMTP path early.** A dev inbox (Mailpit, Ethereal, MailHog) in `.env.example` beats discovering on launch day that port 587 is blocked in the deploy environment.

---

## 4. Slow sends and the request path

SMTP is slow and fails in ways the visitor cannot act on. For a single transactional mail, awaiting it is correct — the user needs to know whether it went. Beyond that:

- **Do not `await` a batch of sends inside a request handler.** It blocks the response and hits the platform's function timeout at exactly the point where half the batch has been sent — with no record of which half.
- Use Nitro's background/task primitives (`runTask`, a scheduled task, or the platform's queue). Verify what your Nitro version and deploy target actually support before designing around it.
- **Make sends idempotent.** Store a send record keyed by (recipient, template, entity) and check it first, or a retry doubles every mail.

---

## 5. Email marketing is not Nodemailer

If the requirement is a campaign to a list, say so plainly and do not build it on raw SMTP. You would be rebuilding list management, bounce and complaint handling, suppression, unsubscribe, and IP reputation — and doing it badly enough to get the domain blocklisted.

Use an ESP (Resend, Brevo, Postmark, Mailchimp) behind a Nitro route. What stays your responsibility:

- **Consent recorded at signup** — timestamp, source, IP. Store subscribers in a Strapi collection so editors can see the list.
- **A working one-click unsubscribe** in every campaign, and a `List-Unsubscribe` header. Legally required in most jurisdictions (GDPR, CAN-SPAM, PECR); functionally, it is what stops complaints becoming blocklistings.
- **Transactional and marketing separated** — different streams, ideally different subdomains. A blocklisted campaign must not take password resets down with it.
- **SPF, DKIM and DMARC on the sending domain** before the first campaign, not after the first bounce report.
- **Never mail a list you did not collect consent for**, and never scrape addresses out of the CMS to build one. If asked to, say no and explain the deliverability and legal exposure.

---

## 6. PDFs

Three distinct cases. Decide which one you have before writing code.

**(a) The PDF already exists in Strapi media.** Do not regenerate it. Link to the media URL through `absoluteMediaUrl` (`strapi-nuxt.md` §9). Only proxy it through Nitro if it must be access-controlled — in which case the media file must not also be publicly reachable, or the proxy is decoration.

**(b) Generated from data — the normal case.** Generate in Nitro and stream it:

```ts
// server/api/invoices/[id].pdf.get.ts
export default defineEventHandler(async (event) => {
  const session = await requireUserSession(event)          // auth-middleware.md
  const id = getRouterParam(event, 'id')!
  const invoice = await getInvoiceForUser(id, session.user.id)   // scoped by session, never by a client id

  if (!invoice) throw createError({ statusCode: 404, statusMessage: 'Invoice not found' })

  const pdf = await renderInvoicePdf(invoice)   // returns a Buffer/Uint8Array

  setResponseHeaders(event, {
    'Content-Type': 'application/pdf',
    'Content-Disposition': `attachment; filename="invoice-${invoice.number}.pdf"`,
    'Cache-Control': 'private, no-store',
  })
  return pdf
})
```

- **Authorise before rendering**, and scope the lookup by the session user — a numeric id in the URL is a client-supplied claim, and an unscoped `findUnique(id)` is the classic document-enumeration hole.
- **Sanitise the filename.** Content from the CMS goes into a header; strip quotes, newlines and path separators, or it becomes header injection.
- **`inline` vs `attachment`** — `inline` previews in the browser tab, `attachment` downloads. Pick deliberately; mobile browsers handle them differently.
- **The link is a plain `<a :href>` to the endpoint**, or an `$fetch` in an event handler. Never fetch a PDF at the top level of `setup()` — it runs on the server too (`data-fetching.md`).

**(c) Rendered from HTML with a headless browser.** Puppeteer/Playwright gives pixel-perfect output and costs ~300MB of Chromium, a cold start measured in seconds, and does not run on most edge/serverless targets. Reach for a document library (`pdf-lib`, `pdfmake`, `@react-pdf`-style templating) first. If HTML-to-PDF is genuinely required, isolate it in its own long-running service or a Node-runtime function, never in the same handler that serves pages — and say the tradeoff out loud before adding it.

**Client-side generation** (`jsPDF`, `html2canvas`) is acceptable only for something the user is looking at anyway — a chart export, a small receipt. It cannot be trusted for anything of record (the client controls the content), it ships a large dependency to every visitor, and it will not match the server's fonts.

**Emailing a PDF:** attach the Buffer directly to `sendMail` (`attachments: [{ filename, content }]`) rather than writing a temp file — most deploy targets have a read-only or ephemeral filesystem. Keep attachments small; link to a signed URL instead once they exceed a few MB, because attachment size is a deliverability factor.

---

## 7. Smells

1. **SMTP credentials in two apps**, or in `runtimeConfig.public`.
2. **A transporter created per request** instead of the §3 singleton.
3. **`from` set to the visitor's address** on a contact form.
4. **User input interpolated into an HTML mail body.**
5. **An unrate-limited, honeypot-free public send endpoint.**
6. **Raw SMTP errors returned to the client.**
7. **A batch of sends awaited inside a request handler.**
8. **A campaign path with no unsubscribe, no consent record, or transactional mail on the same stream.**
9. **A PDF endpoint that trusts a client-supplied id** without scoping to the session.
10. **Puppeteer added to a serverless deployment** for a document a template library could render.
11. **Regenerating a PDF that already sits in Strapi media.**
