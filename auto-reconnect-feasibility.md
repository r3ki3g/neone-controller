# Auto-reconnect on refresh: feasibility

Checked 10/23/26 against the current Chromium source code.

## Short answer

**Yes, this is possible in Chrome, with two conditions:**
1. The page is opened from a real `https://` website.
2. Two Chrome flags are turned on.

You then add each light once, and every refresh reconnects all of them with no taps.

Saving the light IDs ourselves and connecting with them is **not** possible in a browser.

## Why it forgets today

1. **The page was opened from WhatsApp, so Chrome treats it as a local file.** Chrome saves Bluetooth permissions per website. A local file is not a website, so there is nothing to save them under.
2. **Only one of the two flags is on.** `enable-experimental-web-platform-features` only adds the "list remembered lights" function. Saving the permission across refreshes is a separate flag, `enable-web-bluetooth-new-permissions-backend`, which is off by default on Android and desktop.

## Why the page can't just store the IDs

- Web Bluetooth only lets a page talk to a light you picked in Chrome's picker. This is a security rule built into Chrome.
- The ID the page sees is a random token for that website, not the light's real address. It cannot be used to connect without Chrome's saved permission.
- The picker selects one light at a time, so the first setup is 4 picks. That happens once per website.

## Plan A: GitHub Pages + 2 flags (recommended)

1. Publish the page to GitHub Pages so it has an `https://` address.
2. In phone Chrome, enable both flags, then press Relaunch:
   - `chrome://flags/#enable-experimental-web-platform-features`
   - `chrome://flags/#enable-web-bluetooth-new-permissions-backend`
3. Open the GitHub Pages link and add each light once.
4. Refresh. The lights reconnect on their own.

**Risks:** both flags are experimental. A Chrome update could change or remove them, and clearing Chrome's data turns them off.

## Plan B: small Android app (only if Plan A fails)

Wrap the same page in an Android app (Capacitor plus a Bluetooth plugin). Like the vendor apps, it remembers each light's real Bluetooth address and connects on launch, with no flags needed.

**Cost:** you need Android build tools, and the app has to be installed by hand. A plain "website inside an app" won't work, because Android's built-in web view has no Web Bluetooth. The Bluetooth plugin is required.

## Recommendation

Do Plan A first. It costs nothing, and one test on the phone tells us whether it works.
