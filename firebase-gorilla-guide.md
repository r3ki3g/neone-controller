# Firebase in My LED Lights: the gorilla guide

This guide explains how the app stores data in Firebase and how the screen colour gets from a computer to the lights. You don't need to know Firebase already.

**In one line:** Firebase is a shared mailbox. The computer keeps putting its newest screen colour in the mailbox. The phone watches the mailbox and paints the lights whenever something new lands. We run no server of our own; Google's Firestore is the mailbox.

---

## 1. The big picture

```
   COMPUTER (sender)                 FIREBASE (Firestore)                PHONE (hub)
 ┌───────────────────┐            ┌──────────────────────┐          ┌───────────────────┐
 │ "Send this screen"│  setDoc    │ users/prabhashwara/  │ onSnap-  │ "Follow my screen"│
 │ looks at screen   │ ─────────► │   live/screen        │ ───────► │ fades to colour   │
 │ picks ONE colour  │  (write)   │   { h, s, b, t }     │  shot    │ over Bluetooth    │
 └───────────────────┘            │                      │ (listen) └─────────┬─────────┘
          ▲                       │   config/screen      │                    │
          │       onSnapshot      │   { active,          │      setDoc        ▼
          └────────────────────── │     presets, t }     │ ◄──────────── LED strips
             (listen for presets) └──────────────────────┘  (when you edit presets on the phone)
```

There are three jobs:

1. **Live colour** (`live/screen`): computer → phone, many times a minute.
2. **Colour tuning presets** (`config/screen`): goes both ways, only when someone edits a preset.
3. **Sync events** (`mobile_and_desktop_sync_events`, the usage log): every device → Firebase, in batches 10 seconds after someone stops tapping. Unlike the other two, **these pile up** (see section 6).

**Rule of thumb:** all the thinking happens on the computer. The phone never works out colours; it only shows what it's told.

---

## 2. Words you need

| Word | What it means here |
|---|---|
| **Firestore** | Google's database. It stores **documents** inside **collections**, like files inside folders. |
| **Collection** | A folder of documents. Example: `users`. |
| **Document** | One small record of fields (a bit like a JSON object). Example: `screen`. |
| **Subcollection** | A folder inside a document. Example: `live` inside `users/prabhashwara`. |
| **Path** | The address of a document. It alternates collection/document: `users/prabhashwara/live/screen`. |
| **`setDoc`** | "Write this document." Our code always **replaces the whole document**. |
| **`onSnapshot`** | "Watch this document and call me every time it changes." This is how the phone hears new colours instantly. Nobody has to refresh. |
| **Anonymous sign-in** | Firebase gives each browser an invisible guest account. No email or password, but the rules can tell "signed in" from "a stranger". |
| **Security rules** | A small file, [firebase/firestore.rules](firebase/firestore.rules), that Firebase checks on every read and write. |

---

## 3. The whole database

That's all of it. There are three kinds of document:

```
(root)
└── users                          ← collection
    ├── prabhashwara               ← document (never written itself, it only holds folders)
    │   ├── live                   ← subcollection
    │   │   └── screen             ← document: the latest screen colour (always 1, overwritten)
    │   ├── config                 ← subcollection
    │   │   ├── screen             ← document: colour tuning presets (always 1, overwritten)
    │   │   └── colours            ← document: saved colours (always 1, overwritten; website v3.0)
    │   └── mobile_and_desktop_sync_events   ← subcollection: sync events (grows, 1 new document per batch)
    │       ├── 1793045700000-d557dzsn3-k2f9
    │       ├── 1793045763000-d557dzsn3-x8qa
    │       └── …
    ├── chamuth
    │   ├── live/screen
    │   ├── config/screen
    │   └── mobile_and_desktop_sync_events/…
    └── test
        ├── live/screen
        ├── config/screen
        └── mobile_and_desktop_sync_events/…
```

- **People are fixed in code:** `PEOPLE = ['Prabhashwara', 'Chamuth', 'Test']` in [index.html](index.html).
- **The path uses the lower-case name:** `prabhashwara`, not `Prabhashwara`.
- **A document appears the first time someone writes it.** Before that, reading it returns "doesn't exist", and the code handles that.
- **`users/prabhashwara` itself is empty.** In the Firebase console it shows in grey italics with "This document does not exist". That's normal: it's only a parent for `live` and `config`.

---

## 4. Document 1: `users/<name>/live/screen` (the live colour)

**What it holds:** the one colour the computer wants the lights to be **right now**.

### Fields

| Field | Type | Range | Meaning | Example |
|---|---|---|---|---|
| `h` | number | 0–359 | **Hue**, the colour's angle on the colour wheel (0 red, 120 green, 240 blue). | `110` (green) |
| `s` | number | 0–1,000 | **Saturation** in tenths of a percent, so `870` = 87.0%. `0` means **white**. | `870` |
| `b` | number | 1–100 | **Brightness** in percent. The detectors give 10–100, so a dark scene dims the lights without switching them off. | `83` |
| `t` | number | ms | **When** the computer sent it, in milliseconds since 01/01/70 (`Date.now()`). | `1793045700000` |

### Example

```json
{ "h": 110, "s": 870, "b": 83, "t": 1793045700000 }
```

That reads as: "Green, 87.0% saturated, 83% bright, sent 10/26/26 at 1:15 PM US Pacific time."

### Why saturation is 0–1,000, not 0–100

It keeps the numbers whole (no decimals) and still gives fine steps, 0.1% each. The phone divides by 1,000 when it uses it.

### Who writes it and who reads it

| | Who | How | When |
|---|---|---|---|
| **Writes** | The computer, after you press **Send this screen** | `setDoc`, replacing the whole document | Only when the colour changed enough (see section 7) |
| **Reads** | The phone, after you press **Follow my screen** | `onSnapshot`, a live listener | Every time the document changes |

### Why one document that keeps getting overwritten?

We could have saved every colour as a new document (a history log). We don't, because:

- **The phone only needs the latest colour.** Old ones are useless.
- **It never grows.** No clean-up, no deletes, and Firestore storage stays tiny.
- **The listener stays cheap.** One document changes, and the phone gets exactly that one change.

The last minute of history does exist, but only in the computer's memory, for the pictures in **How the colour is picked**. It never goes to Firebase.

---

## 5. Document 2: `users/<name>/config/screen` (colour tuning presets)

**What it holds:** that person's **own** presets for Advanced colour detection, and which one is in use.

### Top-level fields

| Field | Type | Meaning |
|---|---|---|
| `active` | text (up to 40 characters) | The id of the preset in use. It can be a built-in (`neutral`, `nature`, `warm`) or one of the ids in `presets`. |
| `presets` | map (up to 20 entries) | Your own presets, keyed by an id like `pmvpp44o0`. |
| `t` | number | When it was last saved (ms since 01/01/70). |

### One preset

| Field | Type | Range | Meaning |
|---|---|---|---|
| `name` | text | up to 40 characters | What the dropdown shows, like `My Nature`. |
| `boost` | list of 12 numbers | 0–3 each | The colour boost equalizer, one slider per band: Red, Orange, Yellow, Lime, Green, Mint, Cyan, Sky, Blue, Violet, Purple, Pink. `0` ignores that colour; `2` doubles it. |
| `favourite` | list of 12 numbers | 0–5 each | The favourites equalizer, used when the screen has no real colour. All `0` means white. |
| `strength` | number | 0.1–1 | How saturated the favourite colour is. `0.6` = 60.0%. |
| `minColour` | number | 0.01–0.2 | How much of the screen must be clearly coloured before favourites step aside. `0.03` = 3.0%. |
| `soften` | number | 0, 250, 500, 1,000, 2,000, 3,000 or 5,000 | Softening time in ms. `1000` = 1 second. |

### Example

```json
{
  "active": "pmvpp44o0",
  "presets": {
    "pmvpp44o0": {
      "name": "My Nature",
      "boost":     [1, 1, 1.25, 1.5, 2, 1.25, 0, 0.5, 1, 1, 1, 1],
      "favourite": [0, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
      "strength": 0.6,
      "minColour": 0.03,
      "soften": 1000
    }
  },
  "t": 1793045700000
}
```

In this example, Cyan is set to 0 (`boost[6]`), so cyan is ignored. If there's no colour, the lights go orange (`favourite[1] = 3`).

### Things that surprise people

- **Built-in presets are not stored.** Neutral, Nature and Warm live in the code (`BUILTIN_PRESETS`). The document only holds the ones *you* made.
- **Changing a built-in makes your own copy first.** Move a slider while on Nature and the app creates "My Nature" with a new id, then edits that copy.
- **A new id is `p` + the current time in base 36**, for example `p` + `mvpp44o0`. Unique enough for one person's 20 presets.
- **The whole document is saved every time**, with all presets in it, not just the one you changed.
- **The app checks everything it reads back** (`cleanPreset` in [index.html](index.html)). A bad value, like a boost of 99, is clamped to its range. Anyone with the access key could write this document, so it doesn't trust what it reads.

### Saved colours: `users/<name>/config/colours` (website v3.0)

"Your colours" under the Colour tab's default colours. One document per person, overwritten on every save or removal, shared by all of that person's devices.

| Field | Type | Meaning |
|---|---|---|
| `colours` | list, up to 24 | `{ id, name, h (0–359), s (0–1,000) }`: hue and paleness only, never brightness |
| `t` | number | When it was last saved (ms) |

- **Without the access key** the colours stay in that browser (`led-lights-colours`); the first time a person is picked, they become that person's.
- **Checked on read** (`cleanColours`), like presets. The rules only allow the two fields and at most 24 colours, so **the rules must be published again** for website v3.0.

---

## 6. Documents 3 and on: `users/<name>/mobile_and_desktop_sync_events/<id>` (sync events)

**What it holds:** a **batch** of up to 50 things that happened on one device: taps, changes, and lights dropping and coming back. It's for the pilot, so we can see which features people use and when lights drop.

**This is the one that grows.** Every batch is a **new** document. Nothing is overwritten.

### When a batch is sent

- **10 seconds after things go quiet.** Turn on a light and do nothing: it's sent 10 s later. Turn it on, then change the colour 3 s later: both go together, 10 s after the colour change.
- **At most a minute after the first event,** even if someone keeps tapping non-stop.
- **Straight away at 50 events.**
- **When the page goes to the background** (another tab, or the phone's home screen).
- **Idle devices send only a heartbeat**, every 5 minutes, so the viewer can tell a quiet device from one that's off, closed, offline or failing to send: it marks a device live within 10 minutes of its last heartbeat, late within 20, then silent.
- **Android app: battery must be Unrestricted.** Otherwise Android holds the app's internet while it's in the background: events are still noted at the right time, but they only reach Firebase when the app is opened again. App v1.4 asks on every launch until it's set, and the page shows a red warning at the top until then (logged as `battery_warning`).

The batch goes to the person picked on that device, under Screen colour sync. Until a key and a person are set, events wait on the device.

### Fields

| Field | Type | Meaning |
|---|---|---|
| `device` | map | Which device sent it (table below) |
| `from` | number | Time of the earliest event in the batch (ms since 01/01/70) |
| `to` | number | Time of the latest event |
| `events` | list (up to 50) | What happened, in the order the device noticed it |

The document's id is `<from>-<device id>-<4 random letters>`, for example `1793045700000-d557dzsn3-k2f9`. Ids starting with the time list in time order in the Firebase console.

### `device`: who and what

| Field | Example | Notes |
|---|---|---|
| `id` | `d557dzsn3` | A random id kept in that browser, so two identical phones can be told apart |
| `kind` | `app` or `browser` | `app` = our Android app |
| `browser` | `Chrome 156` | |
| `os` | `Android 15`, `macOS 26` | |
| `model` | `Pixel 9 Pro` | Phones only. Samsung gives a code, like `SM-S918B` for a Galaxy S23 Ultra |
| `maker` | `Google` | App only |
| `name` | `Prabhashwara's Pixel` | App only: the name the phone was given |
| `page` | `1.8` | Website version |
| `app` | `1.2` | App version, app only |

### Event types

Every event has `type` and `at` (when, in ms).

| `type` | When | Extra fields |
|---|---|---|
| `open` | A fresh start: the app launched from closed, or the website page loaded | none |
| `background` | The app went to the background, or the browser tab was hidden | `reason`: `left_app` (home or another app), `screen_off`, `closing`, or `tab_hidden` in a browser |
| `foreground` | Came back to the app (or tab) while it was still running | `awayFor` (ms): how long it was in the background |
| `tap` | A person tapped a button, tab or section header | `control`, `label`, `mode`; from website v2.9, `research` (for research only): `sending`, `fastTurns`, `lights` (per light: `light`, `alias`, `lightId`, `status`, `rssi`, `rssiAgeMs`, `writes`, `avgMs`, `maxMs`, `skipped`, `waiting`, `how`, `slow`) and `pose` (the phone's last `face`, `pitch`, `roll`, `heading`, `q`, `at`) |
| `set` | A person changed a slider, colour, tick box or list. Logged once, when the finger lifts | `control`, `label`, `value`, `mode`. The colour wheel adds `from`, and `brightness` and `brightnessFrom` if the ring moved. From website v2.9, `research` as for `tap` |
| `light_down` | A light stopped being connected | `light` (its own name), `alias` (if renamed), `lightId`, `connected`, `of`, `manual`; from website v2.9, `research` (`rssi`, `rssiAgeMs`, `avgMs`, `sending`, `fastTurns`) |
| `light_up` | A light connected | `light`, `alias`, `lightId`, `connected`, `of`, and `downFor` (ms) if it had dropped since the page opened; `research` as for `light_down` |
| `light_stuck` | Website v3.1: a light looked connected but wasn't taking changes, or fell far behind, so the page reconnected it | `light`, `alias`, `lightId`, `reason` (`write_stalled`, `writes_failing` or `lagging`), `error` or `avgMs` and `othersMs`, `kind`, `research` |
| `app_exit` | App v1.9: why Android ended the app's last run, reported once on the next fresh start and placed at the time it happened | `reason` (`low_memory`, `crash`, `user_requested`, `user_stopped`, `excessive_resource_usage`, `freezer`, `other`…), `description`, `importance`, `status`, `pssKb`, `rssKb`, `reportedAt` |
| `rename` | A person renamed a light (website v2.9) | `light` (its own name), `lightId`, `alias` (missing when set back to its own name) |
| `all_down` | The last connected light went | `of`, `manual` |
| `all_up` | Every light is connected | `of`, and `downFor` (ms) if they had all been down |
| `heartbeat` | Every 5 minutes (the first 30 s after opening), even when nothing happened, from every device (website v2.6+) | `foreground`, `lights` and `of` (connected of added), `battery` (%) and `charging` where the browser tells, `online`, `network` (for example `4g`), `waiting` (events not sent yet), `lastSentAt` (last batch that reached Firebase), `lastError` (why the last send failed), `mode`, `unrestricted` (app only: battery saving off); from website v2.8, `sending` (the Bluetooth sending setting: `today`, `confirmed` or `throttled`) and `sent`, one row per light since the last heartbeat: `light`, `lightId`, `how`, `writes`, `avgMs` and `maxMs` (time until each write was done), `skipped` (changes replaced by a newer one before they went out); from website v2.9, `backgroundRestricted` (app only) and `research` as for `tap`; from app v1.9, `power`: `batterySaver`, `deviceIdle`, `bucket` (Android's standby bucket), `unrestricted`, `backgroundRestricted` |
| `position` | Android app v1.7+: where the phone is, once soon after launch | `face` (`face_up`, `upright`, `face_down`, `upside_down`, `left_side`, `right_side`), `pitch`, `roll`, `heading` (degrees, 0 = north), `q` (the 3D rotation as 4 numbers), `signal` |
| `motion` | Android app v1.7+: one movement, from when the phone starts turning (2° or more) to when it has been still for 1.5 s. A quick look at a notification counts, even if the phone ends where it started. Movements longer than a minute come in one-minute pieces | `from` (when it started), `seconds`, `maxDegrees` (largest swing), `netDegrees` (start to end), `startFace`, then the end position as in `position`, `path` (`{at, q}`: a reading every half second or every 1° turned, up to 120), `ongoing` (a piece of a longer movement), `signal` |
| `tilt` | Android app v1.6 only (replaced by `position` and `motion`): the phone turned more than 10° since the last one (at most once a minute) | `face` (`face_up`, `upright`, `face_down`, `upside_down`, `left_side`, `right_side`), `pitch`, `roll`, `heading` (degrees, `heading` 0 = north), `q` (the 3D rotation as 4 numbers), `path` (that minute's readings, `{at, q}` every 5 s while it moved), `signal` |
| `signal` | Android app v1.6+: only the lights' signal changed (by 5 dBm or more) in a minute | `signal`: `[{ light, id, rssi }]`, `rssi` in dBm (closer to 0 is stronger) |
| `ble_scan` | Android app v1.6+: a 10-second scan for nearby Bluetooth devices, every 5 minutes and after a light drops (screen on only) | `seconds`, `reason` (`timer` or `light_drop`), `devices`: up to 100 of `{ address, name, rssi, maker, makerId, kind, rotating, connectable, yours }` |
| `battery_warning` | Android app only: the setup block (a red warning before website v2.9) appeared (`shown: true`) or went away once Android allowed the app to run in the background (`shown: false`) | `shown`; from website v2.9, `unrestricted` and `backgroundRestricted` |

What the fields mean:

- **`control`:** which control. Its id, like `brightness`, or its section and data attributes, like `controls tab=scenes`.
- **`label`:** the words on it, like `Scenes`.
- **`value`:** the new value. **Typed text is never logged**, only that it changed, and the access key box is never logged at all.
- **`mode`:** what the lights were doing at the time, like `colour`, `scene` or `screen`.
- **`connected` / `of`:** how many lights are connected, out of how many added.
- **`manual: true`:** someone pressed Disconnect or Forget. `false` means it really dropped.
- **`lightId`:** the end of the light's id (its Bluetooth address in the app), so two lights with the same name, like the two `GATT--DEMO`, can be told apart.

### Example batch

```json
{
  "device": { "id": "d557dzsn3", "kind": "app", "browser": "Android WebView 156", "os": "Android 15",
              "model": "Pixel 9 Pro", "maker": "Google", "name": "Prabhashwara's Pixel", "page": "1.8", "app": "1.2" },
  "from": 1793045700000,
  "to": 1793045733200,
  "events": [
    { "type": "tap", "at": 1793045700000, "control": "controls tab=scenes", "label": "Scenes", "mode": "colour" },
    { "type": "set", "at": 1793045712000, "control": "wheel", "label": "Colour wheel", "value": "#d6ffad", "from": "#ff0000", "mode": "colour" },
    { "type": "light_down", "at": 1793045730000, "light": "MELK-OA10 46", "connected": 3, "of": 4, "manual": false },
    { "type": "light_up", "at": 1793045733200, "light": "MELK-OA10 46", "connected": 4, "of": 4, "downFor": 3200 },
    { "type": "all_up", "at": 1793045733200, "of": 4 }
  ]
}
```

### How do we know it was done by hand?

The browser marks every event caused by a real finger or mouse as **trusted** (`event.isTrusted`), and code can't fake that flag. Replays, screen colour following and the page's own code change the lights directly, without any tap. So they never show up as `tap` or `set`. Everything in `tap` and `set` was done by a person.

### Never lost, never counted twice

- **Waiting events are kept in the browser** (`led-lights-log`), so a reload or the app closing doesn't lose them. Up to 1,000 are kept.
- **A batch can only be created, never changed** (the rules say so). If a batch is sent twice, for example because the page closed mid-send, the second try is refused. The page then sees the batch already exists and drops its copy.

### Always on

There's no switch for it, and Settings doesn't mention it: every device sends sync events, because the mobile and desktop sync depends on them.

---

## 7. How a colour travels, step by step

You're watching a forest scene on the computer, and the phone is following.

```
 1. Computer looks at the screen       10 times a second (the scan rate setting)
            │
            ▼
 2. Shrinks it to 64 × 36 pixels       2,073,600 pixels → 2,304
            │
            ▼
 3. Picks ONE colour                   using your active preset
            │                          (see screen-colour-algorithm.md)
            ▼
 4. "Is it worth sending?"             most scans say NO, see below
            │ yes
            ▼
 5. setDoc → users/prabhashwara/live/screen  =  { h:110, s:870, b:83, t:… }
            │
            ▼  Firestore pushes the change to everyone listening (usually well under a second)
 6. Phone's onSnapshot fires
            │
            ▼
 7. Phone checks it                    all four numbers present? sent in the last 45 s?
            │
            ▼
 8. Phone fades the lights             over the Smoothness time (default 0.8 s),
                                       up to 20 small steps a second over
                                       Bluetooth (only when a step changes),
                                       the short way round the colour wheel
```

### Step 4 in detail: when does the computer actually write?

Writing on every scan would be 10 writes a second, which would burn through the free plan in about half an hour. So each scan asks "why would I send this?", and there are only four good reasons:

| Reason | When | Example |
|---|---|---|
| `first` | Nothing sent yet since you pressed Send | You just started |
| `jump` | A **big** change: hue moves 40° or more, colour ↔ white, or brightness moves 25 points. Sent as soon as the "at most every" slider allows (default 1 s). | A cut from ocean to forest |
| `drift` | A **small but visible** change: hue moves 5° or more, saturation 6.0% or more, or brightness 4 points. Waits for the "every" setting (default 3 s). | A sunset slowly getting redder |
| `checkin` | Nothing changed for 30 s | A paused movie. This tells the phone "I'm still here". |

Anything else is **not sent**. Scanning faster makes the colour more accurate, but it doesn't send more writes.

### Step 7 in detail: why the phone ignores old colours

The document keeps its last colour forever, even after the computer stops. If you press **Follow my screen** tomorrow, the phone would read yesterday's colour. So:

- **Older than 45 seconds:** the phone ignores it and shows "Waiting for Prabhashwara's computer…".
- **No news for 45 seconds while following:** the phone shows "No colour from … Is it still sending?". That's why the computer checks in every 30 s, even on a still screen.

**Clocks matter:** `t` comes from the computer's clock and the phone compares it with its own. If one clock is minutes off, the phone may treat fresh colours as old. Normally both clocks are set automatically, so this isn't a problem.

---

## 8. How presets sync, step by step

Say you're on the phone, tuning, while the computer sends.

```
 PHONE                                   FIRESTORE                        COMPUTER
 You drag Green to 0
   │
   ▼
 Waits 1 s for you to stop dragging
 (status: "Saving…")
   │
   ▼
 setDoc(config/screen, whole doc) ───►  config/screen updated  ───►  onSnapshot fires
 (status: "Saved for Prabhashwara.")                                     │
                                                                         ▼
                                                                 cleans the presets, keeps
                                                                 them in memory + localStorage
                                                                         │
                                                                         ▼
                                                                 the next scan (≤ 0.1 s later)
                                                                 uses the new preset
```

In testing, the computer had the change 1.1–1.6 seconds after the edit, and 1 second of that is the deliberate wait.

### Small details that prevent bugs

- **The 1-second wait (a "debounce").** Dragging a slider fires dozens of events. Without the wait, each one would be a write.
- **"My unsaved edit wins."** While the phone has an edit waiting to be saved, it ignores incoming snapshots. Otherwise its own old data could bounce back and undo the slider under your finger.
- **Switching person saves first.** If you change from Prabhashwara to Chamuth during the 1-second wait, the pending edit is saved to **Prabhashwara** (the person it belonged to), then the app starts watching Chamuth.
- **It only listens when needed.** The app watches `config/screen` only while the computer is sending or Colour tuning is open, which saves reads.
- **The local copy starts things fast.** The last presets are kept in the browser (`led-lights-tune`), so the app has them instantly at start-up, then Firestore's version takes over.

---

## 9. Who's allowed in

There are three layers.

### Layer 1: the access key

The Firebase `apiKey` is **not** in the code or the GitHub repo. On each device you paste it once, as `AIza…---Prabhashwara`:

- **The part before `---`** is the key. It's saved in that browser only (localStorage `led-lights-apikey`).
- **The part after `---`** picks whose lights these are.

No key means no Firebase at all.

### Layer 2: anonymous sign-in

With the key, the app signs in anonymously, so each browser gets a guest account. The rules below only let signed-in visitors in.

### Layer 3: the security rules ([firebase/firestore.rules](firebase/firestore.rules))

Here they are, in plain English:

```
users/<anyone>/live/screen:
  read  → only if signed in
  write → only if signed in AND the document has exactly h, s, b, t AND all four are numbers

users/<anyone>/config/screen:
  read  → only if signed in
  write → only if signed in AND it has only active, presets, t
                            AND active is text of 40 characters or fewer
                            AND presets is a map of 20 or fewer
                            AND t is a number

users/<anyone>/mobile_and_desktop_sync_events/<any id>:
  read   → only if signed in
  create → only if signed in AND it has only device, from, to, events
                             AND device is a map of 20 fields or fewer
                             AND from and to are numbers
                             AND events is a list of 50 or fewer
  change or delete → refused (a batch is written once, then left alone)

anything else → refused (Firestore refuses whatever no rule allows)
```

So nobody can stuff extra fields in, create other collections, store 1,000 presets, or rewrite the log.

### What the rules do NOT do (be honest about this)

**The name is a label, not a lock.** The rules check the **shape** of the data, not **who** you are. Anyone who has the access key could write to any person's documents, for example Chamuth's computer could write to `users/prabhashwara/live/screen`. That's fine for a few friends sharing one key. Making it strict would need real sign-in (for example Google accounts) and rules that compare `request.auth.uid`.

### Golden rules

- **Never run `firebase deploy`.** Firebase Hosting would publish the full config, key included.
- **Republish the rules after editing them.** Copy [firebase/firestore.rules](firebase/firestore.rules) into Firebase console → Firestore Database → Rules → **Publish**. Editing the file alone changes nothing.

---

## 10. What is NOT in Firebase

Each of these lives in the browser's localStorage, on that device only:

| localStorage key | What |
|---|---|
| `led-lights-apikey` | The access key |
| `led-lights-settings` | Your settings: whose lights, send intervals, scan rate, feature switches and so on |
| `led-lights-tune` | A local copy of the presets, for a fast start |
| `led-lights-writes` | How many writes this computer made today, for the counter in the "Sent to Firebase" step |
| `led-lights-log` | Sync events not sent yet |
| `led-lights-device` | This device's random id for sync events |

The Bluetooth lights themselves, the paired strips and the shows never touch Firebase either.

---

## 11. The free plan budget

The project is on Firebase's free plan (Spark). The daily limits are shared by everyone in the project and reset at **midnight US Pacific time**.

| | Free per day | What uses it |
|---|---|---|
| **Writes** | 20,000 | Every colour the computer sends, every preset save, and every sync events batch |
| **Reads** | 50,000 | Every colour the phone receives (one per write, per phone listening), and every preset update received. The app never reads the log back. |

Writes run out first. Some rough numbers at the default settings:

| Situation | Writes per hour |
|---|---|
| Paused or still screen (check-ins only) | 120 |
| Action movie, worst case (a jump every second) | 3,600 |
| **20,000 writes lasts at least** | **about 5.6 hours of non-stop worst case** |
| Sync events, per device in use | One per burst of activity (a burst ends after 10 s of quiet); 60 with non-stop activity |
| Heartbeats, per device that's open | 12 (one every 5 minutes, about 288 a day), even when idle |

**Stored data (1 GiB free):** sync events are mostly the phone's movements and nearby scans, a few MB a day per busy phone at most. Firestore normally builds a searchable index of every field, which can double what's stored; the `events` and `device` fields are left out of it (see [firebase/firestore.indexes.json](firebase/firestore.indexes.json)), since the viewer only ever searches by `from`.

If the limit is hit, Firestore answers `resource-exhausted`. The app stops sending and says "Firebase's free daily limit is used up…" until midnight US Pacific time.

---

## 12. See it for yourself

### In the real Firebase console

1. Go to console.firebase.google.com → project **neon-controller** → **Firestore Database**.
2. Click `users` → `prabhashwara` → `live` → `screen`.
3. Start **Send this screen** on the computer. You'll see `h`, `s`, `b` and `t` change live.
4. Open `config` → `screen` and change a slider on the phone. The `presets` map updates about a second later.
5. Open `mobile_and_desktop_sync_events`. Tap around on a device, and a new batch appears 10 seconds after you stop, or as soon as you switch away from the page.

### On your own machine (no real data touched)

```bash
cd firebase
firebase emulators:start --only firestore,auth --project neon-controller-cf97b
```

Then serve the folder and open the page with `?emulator=127.0.0.1` on the end, for example `http://localhost:8765/index.html?emulator=127.0.0.1`. Paste any key ending in `---Test`. Everything goes to the local fake Firestore on port 8080 instead of Google's.

---

## 13. Where it lives in the code

Everything is in [index.html](index.html). Search for these names:

| Name | What it does |
|---|---|
| `FIREBASE_CONFIG`, `PEOPLE` | Project ids (not secret) and the list of people |
| `connectFirebase()` | Loads the Firebase SDK, signs in anonymously and returns `{ db, fs }` |
| `parseAccessKey()`, `unlockFirebase()` | Splits `AIza…---Name` and saves the key |
| `screenDoc()` | Builds the path `users/<name>/live/screen` |
| `scanScreen()` | Advanced detector: scan, pick, maybe `setDoc` |
| `sampleScreen()` | Simple detector: the same idea |
| `screenSendReason()` | Decides `first`, `jump`, `drift`, `checkin` or nothing |
| `startScreenFollow()`, `onScreenColour()` | Phone: the `onSnapshot` listener and the 45-second check |
| `screenFollowTick()` | Phone: the fade, 20 times a second |
| `tuneDoc()` | Builds the path `users/<name>/config/screen` |
| `watchTune()`, `applyTuneData()`, `cleanPreset()` | Listen for presets, then check and apply them |
| `saveTune()`, `writeTune()` | The 1-second wait, then `setDoc` |
| `logEvent()` | Notes one sync event |
| `flushLog()`, `sendLog()` | Pack waiting events into batches, then write each as a new document |
| `logLights()` | Spots lights dropping and coming back (called on every render request) |
| `setupUsageLog()` | The tap and change listeners (trusted only), the colour wheel and the battery warning |
| `renderSetupBlock()` | Android app: shows or hides the full-screen setup block (battery Unrestricted, background usage allowed), and logs `battery_warning` when it changes |
| `researchNow()`, `lightResearch()` | FOR RESEARCH ONLY: the per-light Bluetooth data and phone position added to sync events |
| `logDevice()`, `detailLogDevice()` | The `device` map |

Related reading:

- [screen-colour-algorithm.md](screen-colour-algorithm.md): how the computer picks the colour (step 3 in section 7).
- [README.md](README.md): setup, including the one-time Firebase console steps.
