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

## Protocols

Both brands use service `fff0` and write to characteristic `fff3`:

| Brand | Frame | Source |
|---|---|---|
| LotusLamp X | `7e … ef`, RGB colour, brightness 0–100 | [elkbledom](https://github.com/dave-code-ruiz/elkbledom) (MELK-OA10 model) |
| MR Star | `bc … 55`, hue/saturation colour, brightness 0–1,000 | [homey-mrstar](https://github.com/smarthomesven/homey-mrstar) |

The **Diagnostics** section has the original scanner: environment checks, "Pick any device", passive scan, service listing and a log.
