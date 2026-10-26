# Sync events: next features (spec, 10/26/26)

> **Built** in app v1.6, website v2.4 and the dev viewer, with the recommended choices: tilt at most once a minute when it turned more than 10°, light signal strength included, a nearby scan every 5 minutes plus one after a light drop, compass kept. One change from the spec: `path` rows are `{ at, q }` maps, because Firestore can't store a list inside a list.
>
> **Changed in app v1.7 / website v2.5:** the once-a-minute `tilt` is replaced by `position` (at launch) and one `motion` event per movement (2° or more, ending after 1.5 s still), so quick pickups are caught; see firebase-gorilla-guide.md, section 6.

Covers everything discussed so far: app state (done), phone tilt, nearby Bluetooth devices, and how the viewer shows them. Everything here is a sync event in `users/<name>/mobile_and_desktop_sync_events`, batched as today.

Versions when built: app v1.6, website v2.4, plus the local viewer (`dev/sync-events-viewer.html`).

---

## 1. App state: done (app v1.5, website v2.3)

| Event | When | Extra |
|---|---|---|
| `open` | Fresh start: the app launched from closed, or the website loaded | — |
| `background` | Left the app | `reason`: `left_app`, `screen_off`, `closing` (`tab_hidden` in a browser) |
| `foreground` | Came back while it was still running | `awayFor` (ms) |

---

## 2. Phone tilt and orientation

**Goal:** research how the phone's position affects the Bluetooth connection to the lights, and how fast drops happen.

### What the app reads

The app's own Android code reads the motion sensors in its background service, so it keeps working with the app in the background. No permission is needed.

| Value | Meaning | Example |
|---|---|---|
| `face` | Which side is up | `face_up`, `face_down`, `upright`, `upside_down`, `left_side`, `right_side` |
| `pitch` | Tilt forward or back, in degrees | `-12` |
| `roll` | Tilt left or right, in degrees | `4` |
| `heading` | Compass direction the top of the phone points to, in degrees (0 = north) | `275` |
| `q` | The full 3D position as 4 numbers (a quaternion). It drives the 3D phone in the viewer | `[0.71, 0.02, -0.03, 0.70]` |

### When it's sent

- **Read** about 5 times a second, kept only on the phone.
- **Sent at most once a minute**, and only if the position changed by more than 10° since the last one sent. A phone lying still sends nothing.
- **Each `tilt` event carries that minute's path:** a reading every 5 seconds while the phone moved (up to 12). The viewer can then replay the movement, not just jump between minutes.
- **Cost:** up to 60 writes an hour while the phone keeps moving, and 0 while it's still.

```json
{ "type": "tilt", "at": 1793045760000, "face": "upright", "pitch": -12, "roll": 4, "heading": 275,
  "q": [0.71, 0.02, -0.03, 0.70],
  "path": [[0, 0.70, 0.05, -0.02, 0.71], [5, 0.71, 0.03, -0.03, 0.70]] }
```

`path` rows are `[seconds into the minute, q]`.

### Recommended addition: light signal strength

To study tilt against connectivity, the signal of each connected light matters as much as the tilt. The app can read it (RSSI, in dBm, where closer to 0 is stronger) from each light's connection. The proposal is to add it to each `tilt` event, and also send it once a minute while it's changing:

```json
"signal": { "MELK-OA10 46": -61, "MELK-OA10 63": -74, "GATT--DEMO": -80 }
```

### Limits

- **With the screen off**, Android may sleep between readings. The app takes a reading whenever it wakes, and Bluetooth traffic wakes it often. If readings turn out to be too sparse, a "wake on motion" sensor can be added.
- **The compass (`heading`)** is thrown off by metal, magnets and LED power supplies. Tilt and which side is up aren't affected.

---

## 3. Nearby Bluetooth devices

### What the app reads

- **When:** a 10-second scan every 5 minutes while the app runs, plus one right after any light drops.
- **What's saved:** one `ble_scan` event per scan, with up to 100 devices. For each device:

| Field | Meaning |
|---|---|
| `address` | Its Bluetooth address |
| `name` | Its name, if it shares one |
| `rssi` | Signal strength in dBm (closer to 0 = nearer) |
| `maker` | From its maker code: Apple, Samsung, Google, Microsoft… |
| `kind` | Best guess from what it advertises: headphones, watch, TV, LED light… |
| `rotating` | A privacy address that changes every few minutes (most phones, watches and earbuds) |
| `yours` | One of your lights |

- **Cost:** about 12 scans an hour, which is a few writes an hour once batched.
- **Limits:** many devices share no name; rotating addresses make one device look like several over time; some beacons are hidden from apps that don't ask for location, which ours doesn't.

---

## 4. The viewer

All sections share one time cursor: hovering any timeline moves a line across every timeline and updates the 3D phone.

### Header

Person, period and device filters as today. The selected device shows its model, for example "Google Pixel 9 Pro · Android 16 · app v1.6".

### Main timeline (existing), with two new lanes

- **App:** open, background (with reason) and foreground, as dots.
- **Phone position:** a band coloured by `face` (face up, upright, face down…), so long stretches read at a glance.
- **Nearby:** a dot the first time a device appears, for example "New: JBL Flip 5 · Samsung · strong signal".
- Light drops already show in the Lights lane, and also draw a thin red line across every timeline.

### Tilt section

- **Line chart:** pitch, roll and heading over time, with light drops marked. Optionally the signal strength of each light on a second axis below, sharing the time axis.
- **Hover:** a tooltip with the time, `face`, pitch, roll, heading and each light's signal.
- **The 3D phone:** a box drawn like a phone. The front has a dark screen with a camera dot; the back has a camera bump and the label "Back". Edges are visible, so it's clear which way it faces.
  - It rotates to the exact position under the cursor as you hover.
  - **Play:** replays the selected period's movement smoothly, using each minute's `path`, at 1×, 10× or 60× speed, with a clock showing the time being replayed. Lights that drop during the replay flash red beside the phone.
  - A drag on the chart selects the period to replay.

### Nearby devices section

1. **Tiles:** in range now, new today, named devices, your lights in range.
2. **Device timeline:** one row per device, with a bar for each stretch it was seen, coloured by signal (stronger = closer). Your lights are pinned on top. Rotating-address devices are grouped into one "Unnamed phones and earbuds" row with a count.
3. **Device table:** name, maker, type, first seen, last seen, times seen, strongest signal, and a small signal sparkline. Search, plus filters: hide unnamed, hide rotating, close only.
4. **Device detail:** click a device to see its signal over time, with light drops and phone position on the same time axis.

---

## 5. Decisions needed

1. **Tilt sending:** at most once a minute when it changed by more than 10°, with 5-second readings inside each event. Agreed?
2. **Light signal strength:** add it to the tilt data, as recommended above?
3. **Nearby scan:** 10 seconds every 5 minutes, plus one after each light drop. Agreed?
4. **Compass heading:** keep it, knowing it's unreliable near metal and magnets?
