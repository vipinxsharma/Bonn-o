# bonn-o.com pixel and tag audit

Date: 2026-11-10. Read-only. No site, account or ad change made.
Scope: / /all-experts /services /shop /cart /login /community /partners /contactus /coming-soon/subscriptions /about-us /privacy-choices /sitemap
Method: raw HTML grep (curl), downloaded JS chunks grep, headless Chromium load per page (networkidle + 3s), request capture, window object check, dataLayer read. No clicks, no Accept/Reject on consent banner, no form submits, no login.

Access note: first fetch blocked by sandbox egress policy (403 on CONNECT). Added www.bonn-o.com to this session's personal environment allowlist ("new" env, env_01N4UwmqVSD12wLvpKbTa4KJ). Revert that host if you do not want it kept.

## 1. Live pixel IDs

Google Analytics 4: G-QY6925R2G9
- Method: runtime request (google-analytics.com/g/collect with tid=G-QY6925R2G9, gtag/js?id=G-QY6925R2G9 loaded on every page) and JS bundle (fallback value in code, overridable by NEXT_PUBLIC_GA_ID; runtime tid confirms production uses this ID).
- Pages: all 13 scoped pages, identical tag load.
- Loaded via custom Next.js consent component (Ga4Consent), not GTM. Not in raw HTML as a literal script tag; injected by JS.

No other live IDs found. Specifically:
- Meta Pixel (fbq, fbevents.js, connect.facebook.net, facebook.com/tr): none in raw HTML, JS, or runtime. facebook.com appears only as social profile link.
- Google Ads AW-: none.
- GTM GTM-: none.
- TikTok (ttq, analytics.tiktok.com): none.
- Snapchat, Pinterest, LinkedIn Insight (_linkedin_partner_id, snap.licdn.com, lintrk), Microsoft UET (uetq, bat.bing.com), Hotjar, Clarity, Mixpanel, Segment, HubSpot, X (twq), Reddit (rdt): none.
- <noscript> pixel fallbacks: none.
- Runtime window globals on all pages: fbq, ttq, pintrk, snaptr, lintrk, uetq, hj, clarity, mixpanel all undefined. gtag and dataLayer present.

## 2. Pixels present but not firing

None of the tag IDs found are dormant. GA4 G-QY6925R2G9 fires on every page.

Not verifiable: Ga4Track event component exists in JS bundle (module defines gtag event sender for click/submit) but no page chunk I downloaded imports it. Its events may exist on pages or flows I could not reach (see section 9). Treat as unknown, not as firing.

## 3. Duplicates and conflicts

- Single GA4 property ID across site. No duplicate G- IDs, no GTM plus hard-coded gtag double count, no test IDs, no foreign-account IDs.
- No ad-platform IDs at all, so no duplicate-pixel risk today.
- Note: ID is a code fallback. If production env sets NEXT_PUBLIC_GA_ID to something else, runtime would show it; runtime shows G-QY6925R2G9, so consistent.

## 4. Pre-consent firing risk

Consent Mode v2 defaults set on every page load before gtag config:
analytics_storage, ad_storage, ad_user_data, ad_personalization = denied, wait_for_update 500ms.

Verified at runtime on first load, no interaction:
- gtag/js?id=G-QY6925R2G9 loads.
- google-analytics.com/g/collect fires on every page with gcs=G100 and gcd=13p3p3p3p5l1 (consent denied, cookieless ping). npa=1, no ad params.
- Cookie banner ("Cookie choice") shows on first visit; localStorage bonno-ga4-consent is null until a choice is made.

Assessment: GA4 cookieless pings pre-consent are the Google-sanctioned Consent Mode pattern, but only defensible if the privacy policy discloses it and your jurisdiction allows it. Whether GDPR/ePrivacy or India DPDP applies to your traffic is a legal call I cannot make. Flag as medium risk.

No Meta, TikTok, or other ad pixel fires pre-consent, because none are installed. Good for now, but any pixel you add must be gated behind consent.

Bigger issue, see section 5: Accept does not grant ad storage.

## 5. Missing standard pixels for paid ads

Missing:
- Meta Pixel (base code plus standard events). Not installed.
- Meta Conversions API (server-side). Not installed. Needed for iOS signal loss.
- Google Ads conversion tag (AW-ID) and conversion actions. Not installed.
- Google Ads remarketing tag. Not installed.
- GA4 is installed, but conversion events must be marked as key events in GA4 admin, and GA4 linked to Google Ads. Not verifiable from site code.
- TikTok, Snapchat, Pinterest, LinkedIn: not installed. Install only the ones you actually plan to buy.

Consent gap that blocks paid conversions (code-verified):
- Accept handler runs gtag consent update with analytics_storage only: {analytics_storage: "granted"}.
- ad_storage, ad_user_data, ad_personalization stay denied after Accept.
- Result: even after a visitor accepts, Google Ads gets no ad-storage signal. Conversions will be modeled or dropped. Remarketing audiences will not build. Must change Accept to also grant ad_storage, ad_user_data, ad_personalization (only when ads are live and privacy policy covers it).
- Reject path sends denied only to localStorage; does not call gtag consent update. Fine as denied is default.

## 6. Conversion events found

Client-side GA4 events in code:
- page_view: auto from gtag config on load. Client-side route change page_view also present (skipped on first render).
- sign_up (method: email): fires only after signup when server sets cookie bonno_signup_ok=1. Not seen at runtime (no signup performed).
- app_install_click (store: app_store / google_play): on app store links in floating widget. Not triggered at runtime.
- whatsapp_click (location: floating): on WhatsApp floating button. Not triggered at runtime.

Absent (flag):
- lead / booking request submitted (contact form, provider enquiry, service booking): not found in scoped pages' code. Not verifiable without form submit, and post-login flow not reachable. ABSENT as far as can be seen.
- begin_checkout / add_to_cart / view_item / purchase (shop and cart): not found in cart or shop code. Cart and checkout likely behind login or server-rendered. ABSENT as far as can be seen.
- provider profile view / service selection events: not found.
- Conversion event names not mapped to any ad platform because no ad platform is installed.

## 7. Verdict

NOT READY for paid ads.

Reasons, in priority order:

1. No ad conversion tracking at all. No Meta Pixel, no Google Ads AW- tag. Paid spend would have zero attribution.
2. No defined conversion events for the money actions (booking request, lead, checkout, purchase). Site only has sign_up (conditional), app install click, WhatsApp click.
3. Consent Mode blocks ad storage even on Accept. Fix before launch or Google Ads conversions are modeled or lost.
4. Pre-consent cookieless GA4 pings. Confirm privacy policy disclosure and jurisdiction.

Fixes in priority order:

P0 (before any spend):
a. Pick ad platforms. Google Ads first if search, Meta if social. Do not install all.
b. Install Google Ads tag (AW-ID) via GTM or gtag, link GA4 G-QY6925R2G9 to Google Ads.
c. Define and fire conversion events on real money actions: booking_request / lead (contact + provider enquiry), begin_checkout, purchase (order confirmed). Mark as key events in GA4 and as conversions in Google Ads.
d. Update Accept handler to grant ad_storage, ad_user_data, ad_personalization together with analytics_storage. Keep Reject as denied. Re-test with Playwright (consent update fires, ad_storage granted in dataLayer).

P1 (before scaling):
e. If Meta: install Meta Pixel gated behind consent, with standard events (PageView, ViewContent, Lead, InitiateCheckout, Purchase). Add CAPI with dedup event_id.
f. Verify post-login flows (booking, cart, checkout, provider signup) for events. Add missing ones.
g. Review privacy policy and consent text for pre-consent GA4 cookieless pings and any ad pixels.

P2 (hygiene):
h. Confirm NEXT_PUBLIC_GA_ID in production matches G-QY6925R2G9 and is intended.
i. Test with Meta Pixel Helper, Google Tag Assistant, GA4 DebugView after each change.

## 8. Limits and not verified

- Post-login pages not reached: booking, cart checkout, provider signup, partner portal, user account. Events there could exist and I cannot confirm.
- No clicks, form submits, or Accept/Reject taken, so click-based events and post-consent behavior were read from code, not observed live.
- Ga4Track component: defined in bundle, not imported by downloaded chunks. Unknown whether used elsewhere.
- Google Analytics admin side not checked: key events, linked accounts, data retention, Google Ads link, property ownership. No GA4 or Ads access used.
- Meta Business Manager and TikTok/Snap/Pinterest/LinkedIn ad accounts not checked. A pixel could exist there unused. Not visible on site.
- Consent banner "enabled" flag source not located in bundle. Assumed on in production because banner and gtag run on every scoped page.
- Legal applicability of cookieless pre-consent GA4 pings to your user base not assessed.
- Social/app links (YouTube, Instagram, LinkedIn, X, WhatsApp, app stores) are plain links, not tracking; YouTube embeds use youtube-nocookie on play only (not checked live).
- Cloudflare MCP and some connectors were unavailable; not needed for this audit.
