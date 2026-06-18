# 📲 nfc-text

> ## 🤖⚠️ heads up: LLM-written
>
> This project was substantially written by Claude (Anthropic's AI assistant) in a Claude Code session — including the research, design, and the code in this repo. A human directed the work and reviewed it, but the heavy lifting was done by an LLM. Read the code before trusting it with anything that matters. The session UUID is recorded at the top of [`RESEARCH.md`](./RESEARCH.md) for resumability.

## 🤔 what is it

📡 An NFC tag taps a phone. The phone opens a URL. The URL is this site. The site shows a message that lives entirely in the URL fragment.

That's the whole thing. 🎉

## 🤷 why does it exist

🍎 iPhones, when you tap an NFC tag with the phone unlocked but no app open, only act on a small set of NDEF URI schemes: `http(s):`, `tel:`, `mailto:`, `sms:`, `facetime:`, and a few others. They **ignore plain-text NDEF records.**

So if you want a tag to display:

- 🚪 a door code
- 🎟️ a coupon code
- 📶 a Wi-Fi password
- 🗒️ a short note
- 🧭 directions ("knock 3 times, then wait")

...you can't just write the text to the tag. You have to point the tag at a webpage that shows the text. This is that webpage. ✨

For 📞 phone numbers, ✉️ emails, and 💬 SMS, you don't need this site — write `tel:`/`mailto:`/`sms:` directly to the tag and iOS handles it natively. The [generator](./make/) will tell you when that applies.

## 🚀 how to use it

1. 🛠️ Open the [generator](./make/) (`/make/` on the deployed site).
2. ⌨️ Type your message.
3. 📋 Copy the generated URL.
4. 📱 Use an NFC writer app ([NFC Tools iOS](https://apps.apple.com/app/id1252962749) / [Android](https://play.google.com/store/apps/details?id=com.wakdev.wdnfc)) to write that URL to a blank NTAG21x tag.
5. 👆 Tap the tag with any phone. The message appears.

## 🪞 examples

Each of these is a working link — tap to see the page render:

| 🏷️ kind | 🔗 link |
|---|---|
| 🚪 door code | `/#t=Door%20code%3A%204827` |
| 🎟️ coupon | `/#t=SAVE20PCT` |
| 📝 note | `/#t=The%20Wi-Fi%20password%20is%20on%20the%20fridge.` |
| 📞 phone (Call CTA) | `/#t=%2B15551234567` |
| ✉️ email (Mail CTA) | `/#t=hello%40example.com` |
| ↩️ multiline | `/#t=Front%20door%0AKnock%203%20times%0AThen%20wait` |

## 🧱 architecture

```
nfc-text-display/
├── 📄 README.md         (this file)
├── 📚 RESEARCH.md       (problem statement, research, design decisions)
├── 🖼️ index.html        (display page + pitch page in one)
├── 🛠️ make/index.html   (URL generator)
├── ☁️ netlify.toml      (caching, security headers, redirects)
└── 🎴 og.png            (1200×630 unfurl image)
```

- 🪶 **No build step.** Open `index.html` directly in a browser. No `package.json`, no bundler, no framework toolchain.
- 🔐 **No backend.** The message lives in the URL fragment (after `#`). Browsers never send fragments to servers, so the content stays out of HTTP logs and analytics.
- 🌗 **Auto light/dark.** Tiny JS shim mirrors `prefers-color-scheme` into Bootstrap 5.3's `data-bs-theme`.
- 🅱️ **Bootstrap 5.3** for theming and reset, heavily overridden so it doesn't look like a generic SaaS landing page. Aesthetic borrowed from [txt.fyi](https://txt.fyi): monospace, centered, plain.

## 🧪 local dev

```sh
cd nfc-text-display
python3 -m http.server 8765
```

Then open:

- 🏠 http://localhost:8765/ → pitch page
- 💬 http://localhost:8765/#t=Door%20code%3A%204827 → display mode
- 🛠️ http://localhost:8765/make/ → generator
- 📝 http://localhost:8765/make/?t=Door%20code → generator pre-filled

## ☁️ deploy

Drop the repo onto Netlify and you're done. The included `netlify.toml` handles:

- 🚫 `must-revalidate` on HTML (so JS updates reach taps immediately)
- 📅 1-day cache on `og.png`
- 🔒 `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`
- ➡️ `/make` → `/make/` 301 redirect

## 🎯 v1 scope

| 🟢 in | 🔴 out (deferred) |
|---|---|
| plain text display | 📋 markdown rendering |
| auto Call/Email/Open CTAs | 📶 Wi-Fi credential mode |
| copy-to-clipboard | 🔲 QR code preview in generator |
| byte-count vs tag capacity | 🤖 multi-record Android Text fallback |
| bypass suggestions (tel/mailto/sms) | 🌐 i18n |

See [`RESEARCH.md`](./RESEARCH.md) → "Future / v2 ideas" for the full deferred list and rationale.

## 📚 see also

- 📑 [`RESEARCH.md`](./RESEARCH.md) — problem statement, prior-art survey (NoPaste / txt.fyi / 0Bin etc.), iOS NDEF compatibility table, design decisions
- 🔗 [equipter's NDEF-on-iOS gist](https://gist.github.com/equipter/de2d9e421be9af1615e9b9cad4834ddc) — which URI schemes iOS background reading actually handles
- 🏗️ [NoPaste](https://github.com/bokub/nopaste) — closest adjacent prior art (fragment-encoded pastebin)
- 🪶 [txt.fyi](https://txt.fyi/) — the aesthetic inspiration

## 📜 license

No license declared yet. Treat as all rights reserved until one is added. 🤷
