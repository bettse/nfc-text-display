# NFC Text Display via URL — Problem, Solution, and Existing Tools

<!-- Claude Code session: 10e9b072-0cb0-4a94-bd3a-1f7514504d33
     To resume:  claude --resume 10e9b072-0cb0-4a94-bd3a-1f7514504d33
     (run from /Users/bettse/Downloads — the project the session was started in) -->


## The Problem

NFC tags (NTAG213/215/216, MIFARE Ultralight, etc.) support the NDEF standard, which defines multiple record types — most notably:

- **URI / URL records** — open a web page when scanned
- **Text records** — display plain text
- **vCard, WiFi, App-launch, etc.**

On Android, scanning a tag with an NDEF Text record will surface that text to the user (via the system or a tag-reader app), so a maker can write "Hello, my phone is 555-1234" directly to the tag and be done.

**iOS is the problem.** The built-in iOS NFC reader (the "background tag reading" feature on iPhone XS and later) **only acts on URI/URL records**. Plain Text records are silently ignored unless the user has installed and opened a dedicated NFC-reading app. For practical purposes, if you want a tag to "just work" when an iPhone user taps it, the payload must be a URL.

This creates an unnecessary friction for a very common use case: **the tag writer doesn't actually want to send the user to a website — they just want to show a short string.** Examples:

- A phone number to call
- A coupon / promo code to copy
- A short instruction or message ("Door code: 4827", "Wi-Fi is on the fridge")
- An address or location string
- A serial number / asset tag identifier
- A short note attached to a physical object

Today, the workaround is to spin up your own web page (a static HTML file, a Notion page, a link-in-bio service, etc.) just to host a single line of text. That's a lot of overhead and a long-term maintenance burden (domain renewals, hosting, link rot) for what is essentially a string.

## Proposed Solution

A **stateless static website** that renders a message from URL query parameters or the URL fragment. The NFC tag holds a URL like:

```
https://nfc-text.example/?t=Door%20code%3A%204827
```

or, to keep the message fully client-side and out of server logs:

```
https://nfc-text.example/#t=Door%20code%3A%204827
```

When tapped, the iPhone opens the URL, the page reads the param/fragment, decodes it, and renders the message in large, readable text. No backend required — a single `index.html` on any static host (GitHub Pages, Netlify, Cloudflare Pages, etc.) is enough.

### Desirable Features

- **Plain text mode** — display the string, large and centered.
- **Auto-detect content type** — recognize phone numbers (`tel:` link), emails (`mailto:`), URLs (link), and addresses (open in Maps), and offer the appropriate action button.
- **Copy-to-clipboard** button — critical for coupon codes, Wi-Fi passwords, etc.
- **Fragment-based payload (`#`)** — keeps the content out of HTTP logs and analytics; the fragment is never sent to the server.
- **Compression / encoding** — optional base64+gzip or similar to squeeze longer messages into the ~500-byte capacity of common tags.
- **No tracking, no ads, no JS frameworks** — the whole page should be a few KB.
- **Optional title / styling params** — `?title=...&color=...` so different makers can tune presentation.
- **Open source** — so anyone who doesn't trust the hosted version can self-host.

## Implementation Decisions (v1)

- **Stack:** vanilla HTML/CSS/JS, no build step. Single `index.html` for display, `make/index.html` for generator.
- **Encoding:** plaintext URL fragment, `#t=<percent-encoded-text>` (and `&title=<...>` for header). Human-readable, no decoder JS needed.
- **Hosting:** Netlify. Custom short domain planned eventually (e.g. `nfc.ericbetts.dev` or similar) — code builds URLs from `window.location.origin` so the domain is config, not code.
- **Styling:** Bootstrap 5.3+, using its built-in `data-bs-theme` for auto light/dark via `prefers-color-scheme`. Aim: doesn't look generic / AI-generated. Some visual personality (custom typography, layout that isn't a centered gradient card).
- **`/` (bare domain) behavior:** simple pitch page explaining what the site does, with a CTA to `/make`. Same `index.html` decides: payload present → display mode; no payload → pitch mode.
- **Display page actions:** auto-detect content type (phone / email / URL) and surface a primary CTA. Always show a "Copy" button.
- **Generator page:** input field, live URL preview, byte-count vs tag capacity (NTAG213/215/216), and a *bypass suggestion* — if the input matches a phone/email/SMS pattern, recommend writing `tel:`/`mailto:`/`sms:` directly to the tag instead of routing through this site. This is honest about iOS's native NDEF support and respects the user's time.
- **Deferred to v2:** QR code preview in the generator, Wi-Fi credentials renderer, markdown/header rich text, multi-record fallback for Android.

### Future / v2 ideas

- **Header + subheader params** — e.g. `#h=Wi-Fi&sh=Network%20info&t=SSID%3A%20HomeNet`. Cheap to add: two extra fragment keys, minor CSS. Gives a maker a way to label what the message *is* without spending bytes on inline prose.
- **Light markdown support** — opt-in via a flag like `#t=...&md=1`. Use a tiny renderer (e.g. `snarkdown` ≈ 1 KB) restricted to the safe subset: bold, italic, bullets, links — no raw HTML (sanitized, since the input is attacker-controlled). Long-form markdown isn't realistic given tag-byte limits, but light formatting on a coupon code or door code (e.g. "**Door code:** 4827, valid 9am–5pm") would be readable. Skip in v1 to keep the page tiny and the threat model simple.
- **Wi-Fi credentials renderer** — `#wifi=SSID:WPA:password` mode that renders network info with a one-tap copy for the password. Fills a known iOS gap (iOS doesn't act on NDEF Wi-Fi records).
- **Multi-record fallback** — write both a URL record (for iOS) and a Text record (for Android system-level handling) so Android users see the message instantly without the website round-trip.

### Alternative / Adjacent Approaches

1. **Data URLs** — `data:text/html,<h1>Hello</h1>` written directly to the tag. *Doesn't work on iOS Safari* — blocked for security reasons, and not handled by background tag reading.
2. **`tel:` / `mailto:` / `sms:` / `facetime:` URI schemes** — already work for phone/email/SMS use cases without a website. iOS background tag reading handles these natively (per the [equipter NDEF-on-iOS gist](https://gist.github.com/equipter/de2d9e421be9af1615e9b9cad4834ddc)). For these slices of the problem, a maker can write the URI scheme directly to the tag and skip this site entirely.
3. **`https://maps.apple.com/?q=...` / `https://wa.me/...`** — universal links that open Maps / WhatsApp natively.
4. **`shortcuts://` deep links** — invoke an iOS Shortcut that displays the text. Requires the user to install the Shortcut first; not viable for arbitrary tags handed to strangers.
5. **App Clip / Universal Link** — overkill for showing a string.
6. **QR code instead** — sidesteps NFC entirely; the iOS camera app handles many URI schemes, but the user still has to point and aim.

This site fills the *remaining* gap: arbitrary plain text (door codes, coupon codes, Wi-Fi passwords, short notes) that doesn't match a supported native URI scheme. For inputs that *do* match (phone, email, etc.), the generator should suggest the native-scheme bypass so the maker can skip the site entirely.

## Existing Solutions — Research

### Direct hits (purpose-built "NFC text via URL" service)

**None found.** Across multiple targeted searches I did not surface a hosted service explicitly built for "encode a short message in a URL, tap an NFC tag, see the message." Closest patterns are general-purpose pastebins that happen to encode data in the URL — they could be used for NFC but aren't optimized for it (no copy-to-clipboard CTA, no large-text presentation, no tag-capacity warnings, no auto-action for phone/email/URL strings).

### Adjacent — URL-encoded pastebins (best repurposing candidates)

- **NoPaste** — https://nopaste.boris.sh ([source](https://github.com/bokub/nopaste))
  Closest existing pattern to the proposed solution. Compresses the text with LZMA, base64-encodes it, and stores it entirely in the URL fragment (`#...`). 100% client-side; the server never sees the content. Open source, self-hostable. Optimized for code with syntax highlighting, but works for plain text. **Drawback for NFC use:** the compressed-fragment URLs are long, which is a problem for small tags (NTAG213 ≈ 132 bytes user memory). And the UI is a code editor, not a "display this message" page.

- **0Bin** — encrypted pastebin where the AES key lives in the URL fragment, so the server stores ciphertext only. Privacy-preserving but heavier than needed for a public NFC tag.

- **txt.fyi** — https://txt.fyi — a minimalist publishing site by Rob Beschizza. Plain markdown text, no tracking. **Stores content server-side** with a generated URL — so it works as a "post text, get URL" workflow, but isn't a pure URL-encoding scheme. Was shut down in 2022 and re-launched in 2024. Could work for NFC use today but has the long-term link-rot risk every hosted-content service carries.

- **GitHub Gist** — short URLs, free, but again server-side and not a one-step "show this string" page.

### Adjacent — iOS URI schemes that bypass the need for a website entirely

Per the [equipter NDEF-on-iOS gist](https://gist.github.com/equipter/de2d9e421be9af1615e9b9cad4834ddc) and GoToTags' background-reading reference, iOS background tag reading natively handles these schemes — no website needed:

- ✓ `http(s):` → Safari
- ✓ `tel:` → prompts to call
- ✓ `mailto:` → opens Mail
- ✓ `sms:` → opens Messages
- ✓ `facetime:` / `facetime-audio:` → starts FaceTime
- ✓ `bitcoin:` → opens a compatible crypto wallet
- ✓ `shortcuts:` → triggers an iOS Shortcut (if installed)

Not handled by iOS background reading (real gaps this site addresses):

- ✗ `Text` NDEF records (plain text) — silently ignored
- ✗ Wi-Fi credential records — silently ignored
- ✗ vCard / contact records — workaround is to host a `.vcf` and link to it
- ✗ Bluetooth pairing records — silently ignored
- ✗ `data:` URIs — silently ignored
- ✗ Local file URIs — silently ignored

Caveat from the gist: *iOS will select & run only the first compatible record it finds within a tag*, so record ordering matters for mixed-device (Android + iOS) compatibility.

So the **real gap this site fills** is: arbitrary plain text on iOS (door codes, coupon codes, notes, Wi-Fi passwords). For phone/email/SMS use cases, the maker should just write the appropriate URI scheme directly to the tag — no website needed — and the generator page should *tell them so* rather than route them through this site unnecessarily.

### Adjacent — iOS Shortcuts

The `shortcuts://run-shortcut?name=...&input=text&text=...` scheme can invoke a Shortcut on the user's device that displays the passed string. **Not viable for tags handed to arbitrary users** — they'd need the named Shortcut already installed. Useful only for personal/household tags.

### Tooling that creates the tag but doesn't provide the destination

These are tag-writing apps; they're listed because they keep appearing in searches and reinforce that the *destination side* is the gap:

- **NFC Tools** (popular iOS/Android writer)
- **NFC Helper** ([App Store](https://apps.apple.com/py/app/nfc-helper/id6472720100)) — supports URL-scheme launching for write flows
- **Simply NFC** ([App Store](https://apps.apple.com/us/app/simply-nfc-tag-writer-reader/id1262550712))
- **NFC URL Encoder** ([App Store](https://apps.apple.com/qa/app/nfc-url-encoder/id6751737019)) — supports custom URL parameter mapping for writing, but the URL still has to point *somewhere*
- **GoToTags**, **Shop NFC**, **Seritag** — vendor tooling
- **mofosyne/NFCMessageBoard** ([GitHub](https://github.com/mofosyne/NFCMessageBoard)) — initially looked promising (name implied a hosted message-board), but it's an Android-only app that writes plain Text records to the tag itself. Doesn't help with iOS.

### Open-source NDEF libraries (for building your own)

- **andijakl/ndef-nfc** ([GitHub](https://github.com/andijakl/ndef-nfc)) — C# / JS NDEF parser+writer
- **impekable/NFCDecoder** ([GitHub](https://github.com/impekable/NFCDecoder)) — extracts strings/URLs from `NFCNDEFPayload` on iOS
- **nfcpy** — Python NDEF library
- **Plugin.NFC** — Xamarin/MAUI cross-platform NFC plugin

## Summary

The "iPhone-only-reads-URL-NDEF-records" gap is real and widely acknowledged in vendor docs (GoToTags, Seritag, ShopNFC all call it out), but the natural fix — a small static page that turns `?t=...` (or `#t=...`) into a nicely rendered message — does not appear to exist as a dedicated service. The closest adjacent tool is **NoPaste** (bokub/nopaste), which already implements the hard part (text → fragment-encoded URL, fully client-side, hosted version available) but presents itself as a code-syntax pastebin rather than an NFC-message display.

There is room for a purpose-built tool. The differentiation versus NoPaste / 0Bin / txt.fyi would be:

1. **Display-first UI** — large readable text by default, not a code editor.
2. **NFC-aware tag generator** — a sibling page that helps the user write the URL to a tag, with a live byte-count vs. tag-capacity readout (NTAG213/215/216 thresholds).
3. **Auto-action detection** — if the payload parses as a phone number / email / address / URL, surface a primary CTA button (Call, Email, Open Maps, Copy).
4. **Copy-to-clipboard** as a one-tap default — critical for coupon codes, Wi-Fi passwords, door codes.
5. **Smart suggestions** — when the user types a phone number into the generator, suggest "you might just want to write `tel:...` directly to the tag — no website needed" and explain when each approach is appropriate.
6. **Fragment-by-default** (`#t=...` rather than `?t=...`) so the message stays out of HTTP logs and analytics.
7. **Optional short-URL form** (e.g. a `/c/<code>` route mapping to a known canned message) for tags too small to hold the full payload — adds back a stateful component, so it's an opt-in upgrade, not the default.
8. **Wi-Fi credential renderer** — iOS can't act on NDEF Wi-Fi records natively, so a `?wifi=...` mode that renders SSID + password + a copy button would directly fill a known iOS gap.

In short: the building blocks all exist (URLSearchParams, fragment parsing, LZMA compression à la NoPaste, NDEF writer apps), but nobody seems to have packaged them as a single NFC-targeted service. Worth building.

---

## Sources

- [NoPaste source](https://github.com/bokub/nopaste) — closest adjacent pattern (fragment-stored, client-side)
- [txt.fyi](https://txt.fyi/) and [Wikipedia: Txt.fyi](https://en.wikipedia.org/wiki/Txt.fyi) — server-side minimalist pastebin
- [GoToTags: iOS Background NFC Tag Reading](https://learn.gototags.com/nfc/software/iphone/reading/background) — confirms iOS only acts on a fixed set of URI schemes
- [Seritag: How To Read NFC Tags With An iPhone](https://seritag.com/learn/using-nfc/how-to-read-nfc-tags-with-an-iphone) — confirms "text cannot be read natively using an iPhone"
- [ShopNFC: NFC and iPhone](https://shopnfc.com/en/content/20-nfc-iphone) — vendor confirmation of the same limitation
- [Apple Developer Forums: iOS 11 CoreNFC NDEF limitation](https://developer.apple.com/forums/thread/79690) — Apple acknowledges iOS picks only the first compatible NDEF record
- [Apple: PhoneLinks (`tel:` scheme)](https://developer.apple.com/library/safari/featuredarticles/iPhoneURLScheme_Reference/PhoneLinks/PhoneLinks.html)
- [RFC 6068: `mailto:` URI scheme](https://datatracker.ietf.org/doc/html/rfc6068)
- [Hackster: URL for NFC Tag Automation using iOS Shortcuts](https://www.hackster.io/nelson-m/url-for-nfc-tag-automation-using-ios-shortcuts-app-6c3b3d) — `shortcuts://` URL scheme reference
- [mofosyne/NFCMessageBoard](https://github.com/mofosyne/NFCMessageBoard) — Android-only NFC text writer (false lead, but worth noting)
- [andijakl/ndef-nfc](https://github.com/andijakl/ndef-nfc) — NDEF parsing library
- [impekable/NFCDecoder](https://github.com/impekable/NFCDecoder) — iOS NDEF decoder
- [MDN: URI fragment](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Fragment) — confirms fragment is never sent to server
- [Macworld: What Apple's background tag reading NFC update means](https://www.macworld.com/article/231911/what-apples-background-tag-reading-nfc-update-means-for-you-and-businesses.html)
- [NFC Helper (App Store)](https://apps.apple.com/py/app/nfc-helper/id6472720100), [Simply NFC](https://apps.apple.com/us/app/simply-nfc-tag-writer-reader/id1262550712), [NFC URL Encoder](https://apps.apple.com/qa/app/nfc-url-encoder/id6751737019) — tag-writing apps
