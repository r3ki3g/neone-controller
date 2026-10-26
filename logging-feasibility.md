# Logging for the pilot: what can we know?

> **Built** in website v1.8 and Android app v1.2 (10/26/26), as **sync events**, always on since website v2.2 (no Settings switch). What's stored, field by field: [firebase-gorilla-guide.md](firebase-gorilla-guide.md), section 6.

| Question | Possible? |
|---|---|
| 1. Did the user change something by hand, or was it automatic? | **Yes, reliably** |
| 2. Who, which browser, which phone? | **Yes** (phone model on Android; computers give only the OS) |
| 3. Lights dropping and coming back | **Yes** |
| 4. Which other apps are running on the phone | **Only partly**, and only in our Android app. Built in app v1.2, then **removed** in v1.4 |

---

## 1. Manual or automatic? Yes

**The trick:** log where the user touches the page, not where the lights get their commands.

- **The manual paths:** the colour picker, the on/off switch, brightness and every other button go through their own click and input handlers. Those handlers fire only when a person uses them.
- **The automatic paths:** autoplay, replays and screen colour following drive the lights from timers, so they never go through those handlers.
- **Extra safety:** the browser marks real taps and clicks as `event.isTrusted = true`. Code can't fake that flag.

**One rule:** dragging the colour picker or a slider fires dozens of events. Log **once, when the finger lifts**, with the starting and final values.

Example log: `colour_picked · #ff0000 → #00ff88 · manual · phone`

We can log the same way when someone starts or stops a show, a replay, screen sending or following, and when they change a setting or preset.

---

## 2. Who, which browser, which phone? Yes

| What | Where it comes from | Example |
|---|---|---|
| Person | The access key, after `---` | `Prabhashwara` |
| Browser and version | The browser reports it | `Chrome 142` |
| Phone model, Chrome on Android | Client Hints, the same as Gym notes | `Pixel 9 Pro`, or `SM-S918B` for a Galaxy S23 Ultra |
| Phone model, our Android app | Android tells the app directly | `Google Pixel 9 Pro` |
| Phone's own name (app only) | Android settings | `Prabhashwara's Pixel` |
| Computer | OS and version only; no model name | `macOS 15` |
| Versions | Our own constants | website `1.7`, app `1.1` |

Samsung reports model codes (for example `SM-S918B`), not names. Store the code, and turn it into a name with a small lookup table when showing it.

---

## 3. Lights dropping and coming back: Yes

The page already hears every disconnect from each light, and every reconnect.

**How it works:** keep a count of connected lights and log whenever it changes. With 4 lights:

| What happens | Log |
|---|---|
| One light drops | `light_down · MELK-OA10 46 · 3 of 4 connected` |
| The last one drops | `all_down · 0 of 4 connected` |
| One comes back | `light_up · MELK-OA10 46 · 1 of 4 connected · was down 42 s` |
| All are back | `all_up · 4 of 4 connected · all were down 1 min 10 s` |

- **Dropped or switched off?** We know the difference: the user pressing Disconnect is logged as manual, not as a drop.
- **Reconnect attempts:** the page retries every few seconds, so we **don't** log each try, only real changes.
- **Only the phone logs these,** since it's the one holding the Bluetooth connections.
- **No internet at the time?** Firebase queues the logs and sends them when the connection returns, as long as the app stays open.

---

## 4. Which other apps are running? Only partly

- **In a browser (the website):** **no**. A web page can never see other apps.
- **In our Android app:** Android has hidden other apps' running processes from all ordinary apps since 2015 (Android 5.1), for privacy. The closest we can get is **Usage access**:
  - The user switches it on once, by hand: Settings → Usage access → My LED Lights. Android doesn't allow a normal pop-up for this.
  - We then get **which apps were on screen, and when**. Example: `WhatsApp · 10/26/26 9:41 PM · 3 min`.
  - We do **not** get a full list of what runs in the background, or which app is using the microphone.
  - It's fine for our sideloaded app. The Play Store would ask questions.

**Verdict:** it's worth it for the pilot if "which apps were used around the time of a light drop" is useful. It won't give a true "running in the background" list.

**Decision (10/26/26): dropped.** On a Pixel, Android warned that Usage access is risky and wouldn't let it be switched on easily, and it isn't needed. App v1.4 no longer asks for it or has the permission.

---

## 5. Where the logs go (a suggestion)

Unlike the screen colour, logs **do accumulate**. That's the point.

```
users/<name>/mobile_and_desktop_sync_events/<id>   → { device, from, to, events: [ … up to 50 … ] }
```

- **Send logs in batches:** 10 seconds after activity stops (at most a minute after the first event), or once 50 events build up. That's **1 write instead of 50**. Logs share the free plan's 20,000 writes a day with the screen colour.
- **Each event:** `{ type, at, … }`, with its own fields per type. Each batch carries the device info once.
- **The rules need a new entry** for `mobile_and_desktop_sync_events`, then the rules must be republished.
- **Tell the pilot users** what's logged.
- **The future sync** can read the same collection from the computer and the phone.
