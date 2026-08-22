# Universal Accessibility Layer — Full Build Roadmap

A complete, ordered reference for building this Chrome extension in 10–12 days.

---

## 0. Project structure (build this folder first, before any code)

```
accessibility-extension/
├── manifest.json
├── background.js              # service worker — API calls, cross-tab state
├── content/
│   ├── content.js              # main injected script — orchestrates everything
│   ├── contrast.js             # contrast-fixing module
│   ├── clutter.js              # clutter-reduction module
│   ├── simplify.js             # language simplification module
│   ├── voice.js                # voice control module
│   └── observer.js             # MutationObserver / SPA re-application logic
├── popup/
│   ├── popup.html
│   ├── popup.css
│   └── popup.js
├── icons/
│   ├── icon16.png
│   ├── icon48.png
│   └── icon128.png
└── lib/
    └── (any small helper utilities, e.g. contrast math)
```

Why this shape: Chrome extensions run in **three separate JS contexts** that cannot see each other's variables directly — content script (lives inside the webpage), background service worker (lives outside, handles API calls), and popup (the UI when you click the icon). They only talk via `chrome.runtime.sendMessage`. Keeping each concern in its own file makes this separation obvious instead of confusing.

---

## 1. Prerequisite knowledge checklist (Day 0, before writing project code)

Go through this in order. Don't skip to the project until each box makes sense to you.

- [ ] **DOM basics**: `document.querySelectorAll`, `element.style.x = y`, `element.classList.add/remove`, `element.textContent` vs `innerHTML`
- [ ] **Async JS**: `async/await`, `fetch()`, `.then()` — you'll use this for every AI API call
- [ ] **What a Manifest V3 extension is**: read https://developer.chrome.com/docs/extensions/get-started (the official "Hello World" tutorial) and actually build their sample once
- [ ] **The 3-context model**: content script vs background worker vs popup — read https://developer.chrome.com/docs/extensions/develop/concepts/architecture-overview
- [ ] **Message passing**: `chrome.runtime.sendMessage()` / `chrome.runtime.onMessage.addListener()` — this is how your popup tells your content script "turn on contrast mode," and how your content script asks the background worker "call the AI API for me"
- [ ] **chrome.storage API**: `chrome.storage.local.set({key: value})` / `.get(['key'], callback)` — read https://developer.chrome.com/docs/extensions/reference/api/storage
- [ ] **MutationObserver**: https://developer.mdn.org/en-US/docs/Web/API/MutationObserver — needed because modern sites (React/Vue) redraw themselves and silently erase your DOM changes
- [ ] **WCAG contrast ratio formula** (just the math, no need to memorize — you'll look it up when coding contrast.js)
- [ ] **Basic prompt writing for LLMs** — you already have this from using Claude/ChatGPT day to day; the skill here is just being specific and constraining the output format

Time estimate: 1–2 focused days if you're already comfortable with JS. Longer if DOM manipulation is new to you.

---

## 2. Get your dev environment ready (Day 1, morning)

1. Install/confirm Chrome is up to date.
2. Create your project folder (structure above).
3. Get an Anthropic API key (console.anthropic.com) — this is what will power the language-simplification feature. Keep it somewhere you won't accidentally commit to a public repo.
4. Load your (empty/skeleton) extension into Chrome now, before writing real features, so you know the load/reload workflow cold:
   - Go to `chrome://extensions`
   - Toggle **Developer mode** (top right)
   - Click **Load unpacked** → select your project folder
   - Any time you edit code, come back here and hit the refresh icon on your extension's card to reload it
5. Learn where your two debug consoles live — you will use both constantly:
   - **Content script console**: normal page DevTools (F12) → Console tab — logs from content.js show up here
   - **Background worker console**: `chrome://extensions` → your extension card → click **"service worker"** link — logs from background.js show up here
   - **Popup console**: right-click the extension icon → **Inspect popup**

---

## 3. manifest.json — write this first (Day 1)

This is the file that declares what your extension is allowed to do. Rough shape you'll need:

- `manifest_version: 3`
- `permissions`: `["activeTab", "storage", "scripting"]`
- `host_permissions`: `["<all_urls>"]` (needed since you want this to work on any site — note this triggers stricter Chrome Web Store review later, not a problem for hackathon demo via "Load unpacked")
- `background`: point to `background.js` as a service worker
- `content_scripts`: point to your `content/*.js` files, `matches: ["<all_urls>"]`, and set `all_frames: true` if you want to reach into iframes
- `action`: defines the popup (`default_popup: "popup/popup.html"`)
- `icons`: your three icon sizes

**Checkpoint**: load this skeleton (even with empty JS files) into Chrome. Confirm it appears in `chrome://extensions` with no errors. If there are manifest errors, Chrome will show them right there on the card — fix before moving on.

---

## 4. Prove the 3-context wiring works before building any real feature (Day 1, afternoon)

Build a trivial end-to-end test:
1. Popup has a button. Clicking it sends a message to the content script.
2. Content script receives it, changes `document.body.style.backgroundColor = 'yellow'` as a dumb visible proof-of-life.
3. Content script sends a message to the background worker.
4. Background worker logs it in its own console.

**Why this step matters**: this is where most people lose the most time on their first extension — not the "real" features, but silently broken message passing. Get this working and confirmed on a real webpage before writing a single line of contrast/clutter/AI logic. Once this works, every later feature is just "replace the yellow background with real logic."

---

## 5. Build feature 1: Contrast fixer (Day 2–3)

No API calls, pure math — good first real feature, builds confidence with DOM manipulation patterns.

Steps:
1. Select all visible text-containing elements on the page (`document.querySelectorAll('*')`, filtered to elements with direct text content).
2. For each, read computed style: `getComputedStyle(el).color` and `getComputedStyle(el).backgroundColor` (walk up the parent chain if background is transparent, since transparent elements inherit their visible background from an ancestor).
3. Convert both colors to relative luminance, compute the WCAG contrast ratio.
4. If ratio fails the WCAG AA threshold, override with a guaranteed-readable pair (e.g., dark text on light background, or vice versa depending on the original background's own luminance).
5. **Before overriding, store the original inline style** (or lack thereof) in a `WeakMap` keyed by the element, so you can restore it exactly when the user toggles the feature off. This revert mechanism should be built now — you'll reuse the exact same pattern for every other feature.

**Checkpoint**: toggle it on/off on a real low-contrast site (search for one, or intentionally style a test HTML page with pale gray text) and confirm both the fix and the clean revert work.

---

## 6. Build feature 2: Clutter reducer (Day 3–4)

Also rule-based, no API needed.

Steps:
1. Write a list of common "non-core-content" selector patterns: nav bars, sidebars, ad containers/iframes, cookie banners, footers, elements with very low text-to-element ratio (likely decorative/structural, not content).
2. Hide matches (`element.style.display = 'none'`), again storing original state for revert.
3. Increase whitespace/line-height and font-size on what remains, for breathing room.
4. Add a "simplicity level" concept now (mild / moderate / aggressive) — this is just a multiplier/threshold on steps 1–3, and you'll expose it as a slider in the popup later.

**Checkpoint**: test on a genuinely dense, real site (a news homepage is a good torture test — lots of sidebars/widgets). Confirm the page is still usable (links still work, nothing critical got hidden) — this is the step where you'll iterate the most on your selector rules.

---

## 7. Build the MutationObserver / SPA-resilience layer (Day 4, before moving to AI features)

Do this now, not later — you'll want it in place before testing on more complex (React/Vue-based) sites in the next steps.

1. Set up a `MutationObserver` watching `document.body` for added/changed nodes.
2. Debounce it (don't re-run your fixes on every single tiny mutation — batch with a short timeout, e.g. 200–300ms after the last mutation).
3. On trigger, re-apply whichever features are currently toggled on, to any *new* elements that weren't covered before (avoid redoing work on already-fixed elements — check a marker attribute or your WeakMap first).

**Checkpoint**: test on a React-heavy site (anything with infinite scroll or client-side routing is a good test) — confirm your fixes survive re-renders instead of disappearing after a few seconds.

---

## 8. Set up the background worker's API-calling logic (Day 5)

This is the plumbing the AI feature will run through.

1. In `background.js`, add a message listener for something like `{action: "simplifyText", payload: [...]}`.
2. Inside, make the `fetch()` call to the Anthropic API (`https://api.anthropic.com/v1/messages`), with your API key in headers, model, and a prompt built from the payload.
3. Parse the response, send it back to the content script via `sendResponse` or a follow-up `chrome.runtime.sendMessage`.
4. Wrap in try/catch — API calls will sometimes fail (rate limits, network) and your UI should degrade gracefully, not break the page.

**Why background and not content script directly**: keeps API-key handling and network logic in one place, and avoids some CSP (Content Security Policy) restrictions certain sites impose on content scripts making external requests.

---

## 9. Build feature 3: Language simplification (Day 5–7 — your centerpiece AI feature)

This will take the longest and need the most iteration. Break it into sub-steps:

1. **Extraction**: walk the DOM, collect visible text nodes (skip `<script>`, `<style>`, hidden elements, nav/footer boilerplate if you're already hiding those via clutter reduction). Keep a reference from each text node back to its DOM position so you can write the simplified version back into the *same* spot.
2. **Chunking**: don't send the whole page in one API call — group text into reasonably sized batches (by section/container), both for latency and to avoid hitting token limits.
3. **Prompt design**: this is where you'll iterate most. Something like: *"Rewrite the following text at a 6th-grade reading level. Preserve all factual meaning. Do not add commentary. Return only the rewritten text in the same order, separated by [your chosen delimiter]."* Test this against real page content and adjust — LLMs will sometimes add preambles or change meaning slightly; tightening the prompt fixes most of this.
4. **Caching**: hash each text chunk (or use URL + chunk index) and store the simplified result in `chrome.storage.local`, so revisiting a page doesn't re-call the API every time. This also makes your demo faster and cheaper.
5. **Write-back**: replace `textContent` (not `innerHTML`, to avoid breaking any event listeners attached to child elements) at the exact original DOM location.
6. **Revert**: store original text alongside the simplified version so toggling off restores instantly from cache, no re-fetch needed.

**Checkpoint**: test on a page with genuinely dense text — a legal/terms page or a dense Wikipedia article works well. Confirm meaning is preserved (read it yourself, don't just trust it looks shorter) and that the revert works instantly.

---

## 10. Build feature 4 (stretch): Voice control (Day 8)

Uses the browser's built-in `SpeechRecognition` API — no external service needed.

1. Define a small, fixed vocabulary: "simplify page," "increase text," "reduce clutter," "read this," "scroll down," "go back." Don't attempt open-ended command parsing — too unreliable for a live demo.
2. Map recognized phrases to the same functions your popup buttons already call (reuse, don't duplicate logic).
3. Add a clear visual indicator when listening (so users, and your demo audience, know it's active).

**Checkpoint**: test in a quiet room and in a slightly noisy one — know its limits before you demo it live.

---

## 11. Build feature 5 (only if time remains): Image description (optional)

1. Select `<img>` elements without meaningful `alt` text.
2. Send image URL (or base64 if needed) to a vision-capable model via your background worker, same pattern as text simplification.
3. Inject the description as `aria-label` and/or a visible tooltip on hover.

This is the riskiest feature for a live demo (higher latency, more failure surface) — only build it if features 1–4 are already solid and tested. Better to cut this than to have it flake mid-demo.

---

## 12. Build the popup UI (Day 9)

1. Toggle switches for each feature (contrast / clutter / simplify / voice).
2. A simplicity-level slider (mild/moderate/aggressive) feeding into your clutter and simplify thresholds.
3. Persist settings per-domain using `chrome.storage.local`, keyed by `location.hostname`, so a user's preferences stick when they return to a site.
4. Wire every toggle to send the right message to the content script (reusing the message-passing pattern you proved out on Day 1).

---

## 13. Cross-site testing pass (Day 10)

Pick 4–5 real, well-known sites *now* if you haven't already, and test all features against each, methodically:

- A news site (dense layout, good clutter-reduction test)
- A government or institutional site (often genuinely complex, good "real world need" demo)
- An e-commerce product page (tests iframes/ads)
- A React-heavy modern site (tests your MutationObserver resilience)
- Wikipedia (reliable fallback, dense text, good simplify-feature showcase)

For each: toggle every feature on and off, confirm clean revert, confirm nothing critical (checkout buttons, search bars, nav links) breaks or disappears. Note failures and fix your **generic** rules — don't patch per-site, strengthen the underlying logic so the fix helps everywhere.

Known failure points to specifically check for:
- **Shadow DOM** — some modern components hide content in a shadow root your `querySelectorAll` won't reach by default; you'll need to explicitly traverse `element.shadowRoot` where present (only works for *open* shadow roots — closed ones are genuinely inaccessible, note this as a known limitation in your pitch rather than losing time fighting it)
- **CSP restrictions** blocking style injection on some sites — prefer `chrome.scripting.insertCSS` over direct inline style manipulation where you hit this
- **iframes** — set `all_frames: true` in manifest if you need to reach embedded content

---

## 14. Polish + demo prep (Day 11)

1. Add a smooth CSS transition when features toggle (not an instant jarring swap) — meaningfully improves how "finished" it looks.
2. Fix your demo flow to the specific 4–5 sites you tested — **never demo live on an untested site**, the risk of an unexpected break in front of judges is not worth it.
3. Write your pitch structure: problem statement (use the "most websites aren't accessible, this fixes it without asking every site to redesign" framing from your source slide) → live demo → brief tech explanation → future scalability.
4. Prepare answers for the two questions judges will likely ask: "how do you handle the API key/cost at scale" (answer: backend proxy, mention it as the known next step) and "does this work on every site" (answer honestly: works robustly on standard sites, has known limitations with closed shadow DOM — showing awareness of edge cases reads as more credible than claiming universal coverage).

---

## 15. Deploy (Day 12)

For the hackathon demo itself: **you don't need the Chrome Web Store**. "Load unpacked" via `chrome://extensions` (Developer Mode) is instant and is what you should use live.

If you also want it public afterward:
1. Zip the extension folder's *contents* (not the folder itself).
2. Go to the Chrome Web Store Developer Dashboard (one-time $5 registration fee).
3. Upload the zip, fill in store listing (screenshots, description), and **privacy practices disclosure** — you must be upfront that page text is sent to an external AI API for processing.
4. Because of `host_permissions: ["<all_urls>"]`, expect manual review, which can take several days — plan this well outside your hackathon window if you want it live for judges to install themselves later.

Keep Day 12 as buffer even if deployment goes smoothly — something in the last 48 hours always needs a fix.

---

## Quick answers to your other questions

**Hackathon-worthy?** Yes — clear problem, strong live-demo moment, social-good framing judges respond to.

**Scalable?** Yes, architecturally — the per-page cost is the main scaling constraint (LLM API calls), solved with the caching approach in step 9 and eventually a backend proxy for API-key security and rate limiting.

**Is AI/ML actually used, and where?** Yes — LLMs (not custom-trained ML) for language simplification and (optional) image description. Contrast and clutter reduction are deterministic math/rules, not AI — be precise about this distinction in your pitch, it reads as more credible than vague "powered by AI" framing.

**Extra ideas if time allows:**
- Persistent user accessibility profile that auto-applies across all sites, not just per-page
- Dyslexia-friendly font swap (OpenDyslexic) — trivial to add, high perceived value
- Reading-level badge showing the Flesch-Kincaid score before/after simplification
- Keyboard-navigation overlay with numbered jump points
- "Explain this element" hover/click mode powered by the same LLM pipeline
