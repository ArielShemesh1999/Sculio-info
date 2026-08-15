# Sculio

> **A browser-only visual editor for landing pages: describe a site in plain English, clone a live URL, or upload HTML, then edit any element by hand and download one self-contained file.** For people who want a real page out the other end, not an account they can't leave.

**Live:** [editor-beta-ruby.vercel.app](https://editor-beta-ruby.vercel.app) - source is private; this repo is the write-up.

## Turning three entry doors into one editable canvas

An AI builder writes semantic HTML from a prompt, a URL clone pulls a live page in with its CSS, upload takes anything else. All three land on one canvas where text, fonts, colour, position and icons are editable. 70 brand templates repaint that same markup; Mix fuses three brands into one look.

One of 142 design-system brand voices is appended to the builder's system prompt, but voice tunes **copy only** - zero classes, ids or inline styles, so the skeleton survives repainting by any template.

## Resolving `@import` chains so a clone keeps its CSS

The clone enhancer inlines a page's stylesheets and wraps them in `@layer uploaded-css{}` so cloned CSS can't outrank the editor's own. Per spec `@import` is invalid inside a `@layer` block, so every `@import` a fetched sheet pulled in was **silently dropped by the browser** and sites that split CSS that way cloned unstyled.

The enhancer now resolves the chain itself: fetch each bare `@import` through the proxy chain, recurse, splice in place, bounded by 12 fetches per clone and a URL cache that breaks import cycles. Qualified `media`/`layer()`/`supports()` imports stay intact, a failed fetch leaves the `@import` alone rather than corrupting the sheet, and the result merges into the *stored* clone so a reload doesn't lose it.

## Gating paid API spend at the edge

`/api/ai/build`, `/api/ai/research` and `/api/analyze/run` sit behind a same-origin check plus a single-use Cloudflare Turnstile token verified server-side, fail-closed, since Origin alone is forgeable. Behind that, `api/_entitlement-token.js` mints a non-forgeable HMAC entitlement (`{sub, tier, tokens, exp}`) wired into all three routes but deliberately **inert** behind an env flag until a Supabase balance and payment webhook exist. Signing a client-supplied tier would have been the half-fix; it wasn't shipped.

Scraping runs behind a self-hosted `url-proxy-v2` Worker with KV rate caps. The analyze scraper's direct fetch walks redirects manually, revalidating each hop against the SSRF allowlist, since `redirect: 'follow'` chases a 30x to a link-local address blind; the matching revalidation in `url-proxy-v2` is written but still awaits a `wrangler deploy`, since Workers don't deploy on git push.

## Isolating a competitor-analysis report inside the editor

Analyze is a framework-free 13-module engine on Vercel Node routes: no persistence, no paid keys by default, company facts from Wikidata, GLEIF, SEC and RDAP. Its charts are all inline SVG (138 in the sample report, a 5-axis radar, box plots) and the report mounts in a **shadow root** with its own stylesheets, so it and the editor's CSS can't reach each other.

## Verifying past an anti-clone wipe with curl and a seeded Playwright session

The landing page runs an anti-clone script that blanks the DOM under automation - which also blanks your own headless QA. Verification splits: `curl` for headers, endpoint gates and served bundles; an admin-seeded Playwright session for the editor interior, since the wipe only covers `/`. Compositor animations don't tick under Chrome's `--virtual-time-budget`, so the hero demo's beats were checked by seeking with negative `animation-delay` per frame. The unit suite stood at 635 green at the last hardening pass.

## Stack

`vanilla JS` `CSS` `HTML`, no framework. Vercel serverless routes under `api/`; Node for tests. Self-hosted Cloudflare Workers: `url-proxy-v2`, `sculio-turnstile-verify`, `ai-builder` (Claude Sonnet, HMAC-signed) and `email-send` (Resend dispatcher, HMAC-verified, R2 idempotency).

Supabase auth and subscriptions are wired but dormant until env keys are set. RTL is supported; the editor is desktop-only below 760px, gated at one choke point, not per entry door.
