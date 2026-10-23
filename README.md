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

## Features

- **Colour:** one colour picker and presets for every light, plus a shared brightness slider that goes down to Off. On phones it's a built-in colour wheel that updates the lights live while you drag, because Android Chrome's own picker only reports the colour when its dialog closes. Its outer ring is a dimmer from Off to Max. Desktop can switch to the wheel with "Use colour wheel".
- **Replays (PRO):** press ● Record colour changes, play with colours and brightness (up to 10 minutes), then save. The Replays tab lists every replay with a colour ribbon and brightness line, and plays it back after a 3-2-1 countdown. Playback can be as recorded, smooth or extra smooth, at 0.5×, 1× or 2×, with Repeat and Replay all. Replays can be renamed, deleted (with a confirmation), and exported to or imported from a backup file. They're saved in this browser only.
- **Settings:** every feature can be turned off for a simpler app. Features that need another one are greyed out while it's off. Sections fold away and stay folded.
- **Always reconnecting:** lights that drop out are retried every few seconds (never more than 10 s apart), and straight away when the page comes back on screen or Bluetooth turns back on.
- **Catch-up:** a change made while a light is disconnected waits up to 20 s for it. A bar shows "3 of 4 updated · … waiting", then whether every light got the change or one missed it.
- **Scenes:** "Together" scenes that pick the closest match on each brand, each brand's full scene list, and a shared speed slider.
- **Music:**
  - *Lights' own mics:* each controller listens with its own mic. LotusLamp X uses the chosen pattern (Energy, Rhythm, Spectrum, Scroll 1/2); MR Star uses its built-in music mode. One sensitivity slider covers both.
  - *This phone's mic:* the page listens and sends the same colour and brightness to every light about 20 times a second, so all lights stay in sync. Patterns: Pulse, Beat colours, Spectrum. The page keeps the screen on while listening.
- **Diagnostics:** environment checks, "Pick any device", passive scan, service listing, a log, and a command tester for sending one scene number or raw bytes to a single light.

## Protocols

Both brands use service `fff0` and write to characteristic `fff3`.

| Brand | Frame | Source |
|---|---|---|
| LotusLamp X | `7e … ef`: RGB colour, brightness 0–100, scenes `7e 05 03 NN 06 …`, mic patterns `7e 05 03 8N 04 …`, mic on/off, sensitivity and speed 0–100 | [elkbledom](https://github.com/dave-code-ruiz/elkbledom) (MELK-OA10 model) |
| MR Star | `bc CMD LEN DATA 55`: hue/saturation colour, brightness 0–1,000, scenes `bc 06`, speed `bc 08`, mic mode `bc 0f`, sensitivity `bc 12` | [homey-mrstar](https://github.com/smarthomesven/homey-mrstar), [ha-findn-led-ble](https://github.com/hirotasoshu/ha-findn-led-ble), [Tira-led](https://github.com/Luiscrespo-bot/Tira-led) |

Not yet confirmed on real hardware: the names of LotusLamp X scenes 2–5, the order of mic patterns `84`–`87`, MR Star's mic patterns, and MR Star's "leave mic mode" command (`bc 0f 01 00 55` is a guess).
