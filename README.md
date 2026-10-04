# Jitesh Engineering Works - B2B Industrial Website

A modern, high-converting B2B website for an engineering workshop and industrial fabrication unit based in Boisar, Palghar, India.

Inquiries submitted on the site are emailed straight to the owner's inbox. There is no database, no admin panel, and no login â€” the site is stateless.

## Tech Stack

- **Framework:** Next.js 14 (App Router, JavaScript, no TypeScript)
- **Styling:** Tailwind CSS
- **Email:** Nodemailer over Gmail SMTP, pinned at `nodemailer@6.9.16`
- **Database:** none

## Features

- âœ… SEO-optimized pages for all services, with `sitemap.xml` and `robots.txt`
- âœ… Inquiry/RFQ form with file upload (PDF, CAD drawings, images)
- âœ… Submissions emailed to the owner with attachments included
- âœ… WhatsApp floating button on all pages
- âœ… Click-to-call on mobile
- âœ… Google Maps integration
- âœ… Mobile-first responsive design
- âœ… Fast loading, production-ready

## How an Inquiry Travels

```
Visitor fills the form on /contact
  â”‚   (src/components/InquiryFormPreview.js)
  â–¼
POST /api/inquiries                     runtime = 'nodejs'
  â”‚   parse multipart body â†’ honeypot check â†’ rate limit
  â”‚   â†’ validate text fields â†’ validate attachment metadata
  â”‚   â†’ read attachment bytes into memory
  â–¼
src/lib/mailer.js                       Nodemailer SMTP transport
  â–¼
Gmail SMTP (smtp.gmail.com:465)
  â–¼
Owner's inbox
```

Nothing is stored. Attachments are read into in-memory buffers and handed directly to the email message â€” no file is ever written to disk, and no field value is written to a database. When the request ends, the data is gone.

The route exposes a `POST` handler only. There is no API for listing, reading, updating, or deleting leads, because there are no stored leads.

### What "no database" means in practice

**The owner's inbox is the only record of a lead.** Keep those emails, or forward them into whatever CRM or spreadsheet the business actually uses. If a message is deleted from the inbox, the lead is gone.

Two safety nets exist for the case where the email fails to send:

- **Lead-recovery log.** If the send fails, the route writes the complete submission â€” every field value, plus each attachment's name and size â€” to the server log at error level. On Vercel that appears in the project's runtime logs, so the lead can be recovered by hand even though the email never arrived. Attachment contents are deliberately excluded from the log.
- **Optional BCC.** Setting `INQUIRY_BCC_EMAIL` blind-copies every inquiry to a second address, giving a redundant copy outside the primary inbox.

## Quick Start

### Prerequisites

- Node.js 18 or newer
- A Gmail account for sending, with a Mail App Password (see below)

### Installation

```bash
# Install dependencies
npm install

# Create your local environment file
cp .env.example .env.local
# Then edit .env.local and fill in the real values

# Run the development server
npm run dev
```

The site is served at `http://localhost:3000`.

The build does not require SMTP credentials â€” configuration is validated at send time, not at build time, so `npm run build` succeeds with the mail variables unset. A form submission without them returns a generic failure message that points the visitor at the phone and WhatsApp numbers.

## Gmail App Password (required before email works)

Gmail refuses basic-auth logins from applications, so **a regular Google account password will not work here.** The sending account needs a Mail App Password, which is a separate 16-character credential scoped to one app.

1. Sign in to the Google account that will send the inquiry emails.
2. Enable **2-Step Verification** on that account (Google Account â†’ Security). App passwords are not available until 2-Step Verification is on.
3. Go to **Google Account â†’ Security â†’ App passwords**.
4. Create a password for the **Mail** app. Google shows a 16-character value once.
5. Put that value in `SMTP_PASSWORD` in `.env.local`, and set `SMTP_USER` to the same Gmail address.

### Where the App Password is allowed to live

The App Password belongs in exactly two places:

1. **`.env.local`** on the local machine. This file is git-ignored through the `.env*.local` entry in `.gitignore`.
2. **The Vercel project's environment variables**, under Settings â†’ Environment Variables.

That is the whole list. Never commit it, never paste it into a chat message, an issue, a screenshot, or a support thread, and never put it in `.env.example` â€” that file is tracked and holds placeholders only. If the value is ever exposed, revoke it in the Google App passwords screen and generate a new one.

Also note that `SMTP_PASSWORD` has no `NEXT_PUBLIC_` prefix, and must never be given one. Every `NEXT_PUBLIC_*` variable is inlined into the JavaScript bundle that ships to the browser and is readable by anyone who views source.

## Project Structure

```
â”œâ”€â”€ src/
â”‚   â”œâ”€â”€ app/
â”‚   â”‚   â”œâ”€â”€ page.js              # Home page
â”‚   â”‚   â”œâ”€â”€ layout.js            # Root layout + site metadata
â”‚   â”‚   â”œâ”€â”€ globals.css          # Global styles
â”‚   â”‚   â”œâ”€â”€ sitemap.js           # sitemap.xml (public pages only)
â”‚   â”‚   â”œâ”€â”€ robots.js            # robots.txt
â”‚   â”‚   â”œâ”€â”€ about/page.js        # About page
â”‚   â”‚   â”œâ”€â”€ contact/page.js      # Contact / RFQ page
â”‚   â”‚   â”œâ”€â”€ projects/page.js     # Projects gallery
â”‚   â”‚   â”œâ”€â”€ services/
â”‚   â”‚   â”‚   â”œâ”€â”€ page.js          # Services listing
â”‚   â”‚   â”‚   â””â”€â”€ [slug]/page.js   # Individual service pages + SERVICE_SLUGS
â”‚   â”‚   â””â”€â”€ api/
â”‚   â”‚       â””â”€â”€ inquiries/
â”‚   â”‚           â””â”€â”€ route.js     # POST-only: validate, then email
â”‚   â”œâ”€â”€ components/
â”‚   â”‚   â”œâ”€â”€ Header.js
â”‚   â”‚   â”œâ”€â”€ Footer.js
â”‚   â”‚   â”œâ”€â”€ WhatsAppButton.js
â”‚   â”‚   â”œâ”€â”€ IndustriesSection.js
â”‚   â”‚   â””â”€â”€ InquiryFormPreview.js
â”‚   â””â”€â”€ lib/
â”‚       â”œâ”€â”€ mailer.js            # Nodemailer transport + message composition
â”‚       â”œâ”€â”€ escapeHtml.js        # HTML escaping for the email body
â”‚       â”œâ”€â”€ rateLimit.js         # In-memory per-IP submission limiter
â”‚       â””â”€â”€ siteUrl.js           # Public origin resolution + fallback
â”œâ”€â”€ public/
â”‚   â””â”€â”€ images/                  # Site and project gallery images
â”œâ”€â”€ .env.example                 # Environment template (placeholders only)
â””â”€â”€ package.json
```

## Pages & SEO

| Page | URL | Target Keywords |
|------|-----|-----------------|
| Home | `/` | CNC machining Boisar, metal fabrication Palghar |
| CNC Machining | `/services/cnc-machining` | CNC machining services Boisar Palghar |
| Lathe Machining | `/services/lathe-machining` | Lathe workshop near me Palghar |
| Sheet Metal | `/services/sheet-metal-cutting` | Sheet metal cutting Maharashtra |
| Fabrication | `/services/metal-fabrication` | Metal fabrication Boisar Palghar |
| Custom Parts | `/services/custom-parts` | Custom parts manufacturing Palghar |
| About | `/about` | Engineering workshop Boisar |
| Projects | `/projects` | CNC machined parts gallery |
| Contact | `/contact` | Get quote CNC machining Palghar |

Canonical URLs, Open Graph URLs, the sitemap, and robots all derive their origin from `NEXT_PUBLIC_SITE_URL`. When that variable is unset or malformed, the app falls back to `FALLBACK_SITE_URL` in `src/lib/siteUrl.js`, currently `https://jitesh-engineering-works.vercel.app`. **Update that constant as well as the environment variable when a custom domain is attached**, otherwise any environment without an explicit `NEXT_PUBLIC_SITE_URL` will keep advertising the old origin.

## Deployment to Vercel

The site is a standard Next.js app with no database and no persistent storage, so a deploy is just a build plus environment variables.

1. Push the repository to GitHub (or GitLab / Bitbucket).
2. At [vercel.com/new](https://vercel.com/new), import the repository. Vercel detects Next.js and needs no build configuration.
3. Before the first deploy, add every variable from the table below under **Settings -> Environment Variables**. Apply them to Production, Preview, and Development.
4. Deploy. The first build publishes to a `*.vercel.app` subdomain.
5. Set `NEXT_PUBLIC_SITE_URL` to the real deployed origin and redeploy, so canonical URLs, Open Graph tags, `sitemap.xml`, and `robots.txt` advertise the right host.
6. Submit `https://your-domain/sitemap.xml` in Google Search Console.

Attaching a custom domain later: add it under **Settings -> Domains**, then update both `NEXT_PUBLIC_SITE_URL` and the `FALLBACK_SITE_URL` constant in `src/lib/siteUrl.js`.

A deploy fails fast if the build fails, and the build does not need SMTP credentials - mail configuration is validated at send time, not build time.

## Environment Variables

Server-only. These are read in the Node route handler and never shipped to the browser.

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `SMTP_HOST` | yes | `smtp.gmail.com` | SMTP server |
| `SMTP_PORT` | yes | `465` | `465` uses implicit TLS; `587` switches to STARTTLS with no code change |
| `SMTP_USER` | yes | `you@gmail.com` | Sending account, also used as the `From` address |
| `SMTP_PASSWORD` | yes | *16-char App Password* | See the Gmail App Password section above |
| `INQUIRY_TO_EMAIL` | yes | `you@gmail.com` | Inbox that receives inquiries |
| `INQUIRY_BCC_EMAIL` | no | | Optional second copy of every inquiry |

Public. **Every `NEXT_PUBLIC_*` value is inlined into the JavaScript bundle and is readable by anyone who views source.** Never give the App Password this prefix.

| Variable | Required | Example |
|---|---|---|
| `NEXT_PUBLIC_SITE_URL` | no | `https://jitesh-engineering-works.vercel.app` |
| `NEXT_PUBLIC_WHATSAPP_NUMBER` | yes | `918308383673` |
| `NEXT_PUBLIC_PHONE_NUMBER` | yes | `+919323397961` |
| `NEXT_PUBLIC_OFFICE_NUMBER` | no | `+919923507724` |
| `NEXT_PUBLIC_COMPANY_EMAIL` | yes | `jiteshengineeringworks20@gmail.com` |
| `NEXT_PUBLIC_OWNER_NAME` | no | `Rajesh Pujari` |

## Known Limitations

### Vercel caps request bodies at about 4.5MB

The form advertises a 10MB per-file limit and the API enforces it, but **Vercel's serverless functions reject request bodies larger than roughly 4.5MB with a 413 before the handler ever runs.** A visitor attaching a 7MB drawing on the deployed site gets a platform error, not the friendly message from this code.

This is an open decision, not a solved problem. The options are to lower the advertised limit to something under 4.5MB, or to move attachments to a direct browser-to-storage upload (S3 / Vercel Blob) and email a link instead. The client-side 10MB check is still worth keeping because it gives immediate feedback instead of an opaque network failure, and the limits are fully enforced when self-hosting outside Vercel.

### Gmail's 25MB attachment ceiling

The API caps total attachments at 20MB of raw bytes. Base64 encoding inflates that by roughly 37%, so a submission near the cap arrives at Gmail as about 27MB and is rejected at send time. The Vercel body limit above is stricter and will usually bite first. If a send does fail this way, the complete submission is written to the runtime log (see "What no database means in practice"), so the lead is recoverable by hand.

### Rate limiting is instance-local

`src/lib/rateLimit.js` holds its counters in memory, which makes it a speed bump rather than a guarantee under serverless hosting:

- Each concurrent instance keeps its own counters, so a client spread across *N* instances gets up to *N* times the quota.
- A cold start resets the counters entirely.
- A distributed source rotating IP addresses is not slowed at all.

It does stop a single naive script hammering the endpoint from one address, which is the realistic abuse case here. A durable limit needs shared state (Upstash Redis or Vercel KV with an atomic increment) or an edge rate-limit rule such as Vercel Firewall. Both are out of scope until the inbox actually gets flooded.

### No automated test suite yet

The spec defines 20 correctness properties covering validation, HTML escaping, and the no-leak error taxonomy, but the test tasks were deferred to ship the feature. `vitest` and `fast-check` are not yet installed. The security-relevant groups to write first are validation, escaping, and the error taxonomy.

## Customization

- Update contact details in `.env.local` â€” `NEXT_PUBLIC_PHONE_NUMBER`, `NEXT_PUBLIC_OFFICE_NUMBER`, `NEXT_PUBLIC_WHATSAPP_NUMBER`, `NEXT_PUBLIC_COMPANY_EMAIL`, `NEXT_PUBLIC_OWNER_NAME`. These are read across the header, footer, and call-to-action buttons, so changing a number in one place updates the whole site.
- Replace or add images in `public/images/`.
- Update the Google Maps embed in `src/app/contact/page.js` â€” swap the `src` of the `<iframe>` for a new embed URL from Google Maps.
- Modify services data in `src/app/services/[slug]/page.js`. Adding a key to `servicesData` also adds the page to the sitemap, since both read `SERVICE_SLUGS`.
- Adjust the inquiry email layout in `src/lib/mailer.js`.

## License

Private - All rights reserved.

