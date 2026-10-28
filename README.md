# My LED Lights

A single-page Web Bluetooth controller for LotusLamp X (`MELK-…` / `ELK-…`) and MR Star (`GATT--DEMO`) LED strips. It turns them on and off together and gives every strip the same colour and brightness.

## Run locally

Web Bluetooth needs `https://` or `http://localhost`, so serve the folder instead of opening the file directly:

```sh
cd ~/Downloads/LEDController
python3 -m http.server 8000
```

Then open <http://localhost:8000> in desktop Chrome or Edge.

## One-time setup

1. Enable both `chrome://flags/#enable-experimental-web-platform-features` and `chrome://flags/#enable-web-bluetooth-new-permissions-backend`, then relaunch Chrome. Together they let the page remember lights between visits. Open the page from its `https://` link; a downloaded or shared copy of the file can't remember anything.
2. Turn Bluetooth off on the phone running the vendor apps. Each controller accepts only one connection.
3. Press **+ Add a light** once per strip. The picker only shows supported strips.

After that, opening or refreshing the page reconnects every light automatically, and keeps retrying if a light is unplugged or out of range.

Chrome remembers lights per site, so `localhost` and a GitHub Pages address each need their own one-time "Add a light".

## Android app

Chrome pauses a page a few seconds after you switch to another app, which freezes replays. The Android app in `android/` loads the website from GitHub Pages and keeps it running in the background and with the screen off:

- Bluetooth goes through Android directly. `android/app/src/main/assets/bridge.js` gives the page the `navigator.bluetooth` calls it uses, so `index.html` works unchanged. The bridge only answers the website itself.
- A foreground service, with the notification "Keeping your lights connected", stops Android pausing the app. While a replay or the phone mic runs, it also keeps the CPU awake.
- The page is always told it's visible, so its timers run at full speed in the background.
- It needs internet to start; without it, a "No internet" screen offers Try again.
- For sync events (app v1.2 and later): it tells the page the phone's maker, model and name, (v1.5 and later) when the app goes to the background and comes back, since the page itself always counts as on screen, and (v1.6 and later) its own readings: the phone's position and each movement from its motion sensors (v1.7 and later; a quick pickup counts, a still phone sends nothing), each light's signal strength, and a scan of nearby Bluetooth devices every 5 minutes and after a light drops. It doesn't look at other apps.
- On every launch until it's set (app v1.4 and later), it asks to set its battery use to Unrestricted, because Android otherwise holds its internet in the background: following a screen's colour stops and sync events wait until the app is opened again. Until then (and, from app v1.8, while Android's "Allow background usage" is off) a full-screen block covers the whole app, with no way past it, and buttons that open the right Android setting; it closes by itself once Android allows it. Its appearing and going away are sync events, so a computer can show it too.

`index.html` is the single source of truth: website changes reach the app on its next launch. Only changes to the native code need a new APK. The page shows "Website v…" under the title (and "App v…" inside the app); bump `PAGE_VERSION` in `index.html` with each change, and `versionName` in `android/app/build.gradle.kts` with each app change.

Build it on a Mac with the Android SDK:

```sh
cd android
./gradlew exportApk
# Writes MyLEDLights_v<version>.apk next to index.html, replacing the previous one.
```

Install it with `adb install -r MyLEDLights_v1.9.apk` (USB debugging on), or copy the file to the phone and open it. The app is signed with this Mac's debug key, so updates must be built on the same Mac; otherwise uninstall first, which deletes the app's saved replays. The app keeps its own remembered lights and replays, separate from Chrome: add the lights once, and move replays with Export and Import.

Limit: "This phone's mic" music patterns only listen while the app is open, because Android doesn't let a background app use the mic without a separate permission type.

## Screen colour sync

The lights follow the main colour on a computer screen, like an ambient TV light. On a computer (Chrome or Edge, Mac or Windows), open the website, then press **Send this screen** in the **Screen** tab (PRO, between Music and Replays; before website v3.0 it sat under the colour picker): the page looks at the screen twice a second and works out the strongest vivid colour and the overall brightness. A big jump, like a cut from ocean to forest, is sent as soon as the "at most every" slider allows (0.2–3 s, default 1 s). That slider also limits everything else the computer sends, and it shows how many hours a busy screen lasts on the free plan, with a warning below 1 s. Smaller drifts are sent on the interval picked in the section (1–30 s, default 3 s). An unchanging screen still checks in every 30 s. On the phone with the lights, press **Follow my screen**: it fades the lights to each new colour over Smoothness. Its **Brightness range** (two handles, 1.0–100.0%, default 10.0–100.0%, which changes nothing) is always applied while following, so dark scenes keep the room lit and bright scenes stay under the top: the screen's brightness is squeezed into it, and a line shows the screen's brightness and what the lights get. The range and Smoothness are remembered on the device. When the free daily writes run out, both devices say so. How the colour is picked: [screen-colour-algorithm.md](screen-colour-algorithm.md).

**Advanced colour detection** (Settings, on by default) looks at the screen 2–20 times a second (default 10; it doesn't cost more writes), softens what it sees over time, and picks the colour with the person's preset under **Colour tuning** (PRO; always open on computers, folded on phones until opened): a 12-band colour boost (for example Sky down so a bluish sky doesn't beat green trees), favourite colours for screens without colour (instead of white), the strength of that colour, the minimum share of colour, and the softening time. Presets are saved per person in Firestore at `users/<name>/config/screen` and can be edited on the phone while the computer sends; the computer picks up a change within about a second. Turning the feature off brings back the simple detector.

**Show how the colour is picked** (Settings, on by default) lays Colour tuning out as the steps the computer takes, top to bottom with arrows: your screen → shrunk to 64 × 36 → each pixel sorted into colour, faint tint or ignored → this look's colours → softened (with a 10-second timeline of each look against the softened result) → boosted and picked → favourites when there's no colour → the colour → sent to Firebase (with a 1-minute timeline of each send and why). Each setting sits beside its step, and **Show the maths** toggles the formulas. The pictures are live on the computer while it sends, at most 5 times a second and only while visible; phones, which can't share a screen, always get the compact layout and don't show this setting. With the simple detector it shows that detector's shorter flow. Turned off, Colour tuning is the compact layout.

It runs through Firestore in the `neon-controller` Firebase project, with no server of our own:

- **Access key:** the Firebase `apiKey` is not in this repo. Paste it once on each device in the Screen colour sync section, as `AIza…---YourName`. The name picks whose lights these are (Prabhashwara, Chamuth or Test), and each person's computer reaches only their own phone. The other config values in `index.html` are identifiers that grant nothing on their own.
- **Sign-in:** anonymous Firebase sign-in. `firebase/firestore.rules` lets signed-in visitors read and write only `users/<name>/live/screen`, only as `{ h, s, b, t }`, `users/<name>/config/screen`, only as `{ active, presets (up to 20), t }`, and new documents under `users/<name>/mobile_and_desktop_sync_events`, only as `{ device, from, to, events (up to 50) }`, which can't be changed afterwards. The page checks every preset it reads back.
- **Never run `firebase deploy`:** Firebase Hosting would publish the whole config, key included.

Firebase console setup, done once:

1. Firestore Database created in production mode, and `firebase/firestore.rules` published under Rules (publish it again after it changes).
2. Authentication → Sign-in method → Anonymous enabled; Authorized domains include `r3ki3g.github.io`.
3. Firestore Database → Indexes → Single field → Add exemption, for collection `mobile_and_desktop_sync_events`, field `events`, with every index type turned off for both the collection and the collection group. Repeat for field `device`. This keeps Firestore from building a searchable copy of every event, which would roughly double the stored data; [firebase/firestore.indexes.json](firebase/firestore.indexes.json) records the same setting.
4. Recommended: Google Cloud Console → APIs & Services → Credentials → Browser key → website restriction `r3ki3g.github.io/*`.

Testing without the real project: in `firebase/`, run `firebase emulators:start --only firestore,auth --project neon-controller-cf97b`, then open the page with `?emulator=127.0.0.1` and any key ending in `---Prabhashwara`.

Limits: copy-protected video (Netflix, Disney+) is captured as black. On a Mac, Chrome needs the Screen & System Audio Recording permission.

## Features

- **Colour:** one colour picker and presets for every light, plus a shared brightness slider from 1% to 100%. On phones it's a built-in colour wheel that updates the lights live while you drag, because Android Chrome's own picker only reports the colour when its dialog closes. Its outer ring is a dimmer from Min to Max; it never switches the lights off, and overshooting an end while dragging stays at that end. Desktop can switch to the wheel with "Use colour wheel".
- **Reliable on/off:** on/off, scene and mic-mode commands ask the light to confirm delivery, so one can't be silently dropped among other commands.
- **Replays (PRO):** press ● Record colour changes, play with colours and brightness (up to 10 minutes), then save. The Replays tab lists every replay with a colour ribbon and brightness line, and plays it back after a 3-2-1 countdown. Replays can be renamed, deleted (with a confirmation), and exported to or imported from a backup file. They're saved in this browser only.
- **Playback settings per replay:** smoothing (as recorded, smooth, extra smooth), speed (0.25×–4× presets or any custom speed up to 20×), or a total length to stretch it over (up to 2 hours; a 3-second recording can play over a minute), and Repeat. They're saved with each replay and shown on its row. Replay all plays each with its own settings and has its own Repeat the list.
- **Colour sequences:** ＋ Create colour sequence lets you pick 2–12 colours (each with its own brightness and an optional hold) and link them in order. Each link has a time (0.1 s to 60 min), a curve (Linear, Ease in, Ease out, Smooth, Step) and a colour path (Blend, Short way round, Long way round, Through dark). Sequences can loop back to the first colour, be previewed on the lights while editing, and be edited again later.
- **Settings:** every feature can be turned off for a simpler app. Features that need another one are greyed out while it's off. Sections fold away and stay folded.
- **Always reconnecting:** lights that drop out are retried every few seconds (never more than 10 s apart), and straight away when the page comes back on screen or Bluetooth turns back on.
- **Catch-up:** a change made while a light is disconnected waits up to 20 s for it. A bar shows "3 of 4 updated · … waiting", then whether every light got the change or one missed it.
- **Scenes:** "Together" scenes that pick the closest match on each brand, each brand's full scene list, and a shared speed slider.
- **Music:**
  - *Lights' own mics:* each controller listens with its own mic. LotusLamp X uses the chosen pattern (Energy, Rhythm, Spectrum, Scroll 1/2); MR Star uses its built-in music mode. One sensitivity slider covers both.
  - *This phone's mic:* the page listens and sends the same colour and brightness to every light about 20 times a second, so all lights stay in sync. Patterns: Pulse, Beat colours, Spectrum. The page keeps the screen on while listening.
- **Sync events** (always on, with no switch in Settings): what people tap and change by hand, lights dropping and coming back, and the device's browser and model, sent to Firestore at `users/<name>/mobile_and_desktop_sync_events` in batches: 10 seconds after activity stops, and at most a minute after the first event. Every device also sends a heartbeat every 5 minutes (app in front or not, lights connected, battery, internet, events waiting and the last send error), so a quiet device can be told apart from one that's offline or failing. Real taps are told apart from replays, screen sync and the page's own code by the browser's `isTrusted` flag. The access key and typed text are never logged. What's stored and how: [firebase-gorilla-guide.md](firebase-gorilla-guide.md).
- **Diagnostics:** environment checks, "Pick any device", passive scan, service listing, a log, and a command tester for sending one scene number or raw bytes to a single light. It also shows how each light keeps up with Bluetooth sending over the last 10 s: changes a second, time until each was done, and changes skipped.
- **Layout (website v3.0):** All lights (on/off, brightness) is its own card, and the tabs (Colour, Scenes, Music, Screen PRO, Replays PRO) sit on a second card as real tabs: the active one is lit, has an accent edge and joins its content. In the Screen tab, the access key, Follow my screen / Send this screen and Faster Bluetooth turns are always shown; the follow and send settings fold away under "Screen colour sync settings", and setting names read as titles.
- **Your colours (website v3.0):** "＋ Save current colour" under the default colours saves the current hue and paleness (not brightness) with a name; tap one to use it, "Remove some" to delete. With the access key they're saved per person in Firestore (`users/<name>/config/colours`) and shared by all their devices; otherwise in that browser only.
- **Lights:** each light can be renamed (Rename); the new name shows everywhere on the page, while logs and sync events keep the light's own name with the alias beside it. In the Android app (v1.8+) each light shows its signal strength, and on any device how fast it answers, each as a curve of the last 10 minutes; a light answering 3× slower than the others' median (and over 100 ms) is marked **Slow**.
- **Faster Bluetooth turns** (Settings → Bluetooth, and a tick box under Follow my screen; Android app v1.8+): asks Android for its shortest connection interval with every light (11–15 ms instead of 30–50 ms). Below 30.0% battery a red line warns that it drains the battery faster.
- **Research data** (in the code marked FOR RESEARCH ONLY): manual taps and changes, light drops and returns, and heartbeats carry each light's signal, answer times, waiting changes, the Bluetooth settings and the phone's last position, to find out why lights lag or drop.
- **Stuck lights (website v3.1, app v1.9):** a light that looks connected but doesn't take changes (a change with no answer for 5 s, or 2 failed changes in a row) is reconnected, and the change it missed still lands. When Bluetooth turns off, the app drops every link and says so, and a light that keeps refusing writes counts as dropped. Each forced reconnect is a `light_stuck` sync event.
- **Reconnect a light that falls behind** (Settings → Bluetooth, on by default): with Confirmed or Mixed sending, a light answering over 0.5 s and 3× slower than the others for 5 s is reconnected for a fresh turn on the radio, at most every 2 minutes.
- **Pause nearby scans while lights stream** (Settings → Bluetooth, on by default, app v1.9): the 5-minute nearby scan waits during a wheel drag, Follow my screen, a replay or the phone mic.
- **Why the app was ended (app v1.9):** on a fresh start the app reports Android's reason for the last exit (low memory, crash, user stopped, battery restrictions…) as an `app_exit` sync event at the time it happened; heartbeats add Battery Saver, deep sleep and the app's standby bucket.
- **Bluetooth sending** (Settings, for an A/B test): *As today* sends colour and brightness without waiting for the light; *Confirmed* waits for each light's "got it" and sends only the newest change, so a light with a weak signal skips to the newest colour instead of replaying old ones (a light that can't confirm is throttled instead); *Throttled* sends at most 10 changes a second to each light; *Mixed* (website v3.1) sends most changes without waiting but every 5th, and the last one after a drag, confirmed. The heartbeat carries the setting and each light's numbers. Why: [bluetooth-lag-gorilla.md](bluetooth-lag-gorilla.md).

## Protocols

Both brands use service `fff0` and write to characteristic `fff3`.

| Brand | Frame | Source |
|---|---|---|
| LotusLamp X | `7e … ef`: RGB colour, brightness 0–100, scenes `7e 05 03 NN 06 …`, mic patterns `7e 05 03 8N 04 …`, mic on/off, sensitivity and speed 0–100 | [elkbledom](https://github.com/dave-code-ruiz/elkbledom) (MELK-OA10 model) |
| MR Star | `bc CMD LEN DATA 55`: hue/saturation colour, brightness 0–1,000, scenes `bc 06`, speed `bc 08`, mic mode `bc 0f`, sensitivity `bc 12` | [homey-mrstar](https://github.com/smarthomesven/homey-mrstar), [ha-findn-led-ble](https://github.com/hirotasoshu/ha-findn-led-ble), [Tira-led](https://github.com/Luiscrespo-bot/Tira-led) |

Not yet confirmed on real hardware: the names of LotusLamp X scenes 2–5, the order of mic patterns `84`–`87`, MR Star's mic patterns, and MR Star's "leave mic mode" command (`bc 0f 01 00 55` is a guess).
