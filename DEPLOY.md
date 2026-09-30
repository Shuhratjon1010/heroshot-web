# Deploying heroshot.app on Cloudflare Pages

The site is plain static HTML and CSS. There is no build step: Cloudflare serves the repository root as is.
`_headers` sets security and cache headers, `_redirects` keeps old links working, and `404.html` is the not-found page.

Do the steps in order. Steps 1 to 4 take about 15 minutes plus DNS propagation.

## 1. Create the Pages project

1. Sign in at https://dash.cloudflare.com.
2. Go to **Workers & Pages**, click **Create**, pick the **Pages** tab, then **Connect to Git**.
3. Choose GitHub, authorize Cloudflare for `Shuhratjon1010/heroshot-web`, and select the repo.
4. Set up builds:
   - Project name: `heroshot-web`
   - Production branch: `main`
   - Framework preset: **None**
   - Build command: *(leave empty)*
   - Build output directory: `/`
5. Click **Save and Deploy**. You get `https://heroshot-web.pages.dev`. Every pull request also gets its own preview URL.

Prefer the command line? From a checkout of this repo on your Mac:

```sh
npx wrangler login
npx wrangler pages deploy . --project-name heroshot-web --branch main
```

## 2. Put heroshot.app on Cloudflare DNS

Skip this step if heroshot.app already shows as **Active** under **Websites** in the Cloudflare dashboard.

1. In the dashboard click **Add a domain**, enter `heroshot.app`, choose the **Free** plan.
2. Cloudflare scans your current DNS records. Check that anything you still use is listed (for example `api` for the HeroShot worker, or existing MX records). Then continue.
3. Cloudflare shows two nameservers, like `xxx.ns.cloudflare.com`. At the registrar where you bought heroshot.app, replace the domain's nameservers with those two. Remove DNSSEC at the registrar first if it's on.
4. Wait until Cloudflare emails you that the domain is active (usually under an hour, can take up to 24).

A root domain like heroshot.app can only point at Pages when its DNS is on Cloudflare, which is why this step is needed.

## 3. Attach the domain to the site

1. **Workers & Pages** > `heroshot-web` > **Custom domains** > **Set up a custom domain**.
2. Enter `heroshot.app` and click **Activate domain**. Cloudflare creates this record for you:

   | Type | Name | Target | Proxy |
   |---|---|---|---|
   | CNAME | `heroshot.app` (shown as `@`) | `heroshot-web.pages.dev` | Proxied (orange cloud) |

3. Repeat for `www.heroshot.app`:

   | Type | Name | Target | Proxy |
   |---|---|---|---|
   | CNAME | `www` | `heroshot-web.pages.dev` | Proxied |

4. Send `www` to the root: **heroshot.app** > **Rules** > **Redirect Rules** > **Create rule** > template **Redirect from WWW to root** > Deploy.
5. **SSL/TLS** > **Overview**: set mode to **Full (strict)**. **SSL/TLS** > **Edge Certificates**: turn on **Always Use HTTPS**.

`.app` domains only work over HTTPS; Cloudflare issues the certificate automatically within a few minutes of step 2.

## 4. Turn on the support and privacy email addresses

The site, the app and the App Store listing point people to `support@heroshot.app` and `privacy@heroshot.app`. Make them real:

1. **heroshot.app** > **Email** > **Email Routing** > **Get started**. Let Cloudflare add its MX and TXT (SPF) records.
2. **Destination addresses**: add the inbox you read, and click the link in the verification email.
3. **Routing rules** > **Create address**: `support` forwards to that inbox. Do the same for `privacy`.
4. Optional: turn on **Catch-all** to forward anything else sent to @heroshot.app.

## 5. Check it works

```sh
curl -sI https://heroshot.app/ | head -1                # HTTP/2 200
curl -sI https://heroshot.app/privacy | grep -i location  # -> /privacy/
curl -sI https://heroshot.app/privacy/ | head -1        # HTTP/2 200
curl -sI https://www.heroshot.app/ | grep -i location   # -> https://heroshot.app/
curl -sI https://heroshot.app/content-policy | grep -i location  # -> /terms/#acceptable-use
```

Then send a test email to support@heroshot.app and confirm it arrives.

## 6. Point App Store Connect at the new pages

Do this as soon as step 5 passes. Until then the old github.io links keep working: `privacy.html`, `terms.html` and `support.html` stay in the repo as small redirect pages for GitHub Pages, and Cloudflare redirects them instead.

| Where in App Store Connect | Field | Value |
|---|---|---|
| App Information | Privacy Policy URL | `https://heroshot.app/privacy/` |
| iOS App > version 1.0 | Support URL | `https://heroshot.app/support/` |
| iOS App > version 1.0 | Marketing URL | `https://heroshot.app/` |
| iOS App > version 1.0 > Description (last line) | Terms of Use link | `Terms of Use: https://heroshot.app/terms/` |

Apple requires a Terms of Use (EULA) link for auto-renewing subscriptions. The description line above covers it; alternatively paste the terms into **App Information > License Agreement**.

## 7. Search engines

1. Add `heroshot.app` as a Domain property in Google Search Console (Cloudflare can add the TXT record for you).
2. Submit `https://heroshot.app/sitemap.xml`.

## Things to update later

- **App Store link.** Every download button points to `https://apps.apple.com/app/id6772790208`. It will 404 until the app is live, then work without changes. If the app ID is different, run `grep -rl id6772790208 .` and replace it.
- **App Store badge.** `assets/app-store-badge.svg` follows Apple's badge layout. For the official artwork, download "Download on the App Store" (black, SVG) from https://tools.applemarketingtools.com and save it over that file.
- **Legal entity.** The Terms say they're governed by the laws of where HeroShot's developer is established. Once you have a company or want to name a country and courts, edit section 13 of `terms/index.html`.
- **Legacy pages.** `welcome` and `founding-claim` are the pre-launch founding-seller pages. They're kept, unlinked and hidden from search, so any links already emailed still open.

## Local preview

```sh
npx wrangler pages dev .
```

This serves the site at http://localhost:8788 with the same headers and redirects as production.
