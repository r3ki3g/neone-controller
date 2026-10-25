# How the screen's main colour is picked

The goal is the colour an ambient TV light would choose: the strongest real colour on screen, not the average. Averaging a half-red, half-green screen gives a muddy brown; both detectors below pick red.

There are two detectors. **Advanced colour detection** (Settings, on by default) is the tunable one. Turning it off gives the simple one. With **Show how the colour is picked** on (also the default), the page shows each step below live on the computer while it sends, with the formulas behind a Show the maths switch.

## Advanced detection

On every scan (2–20 times a second, default 10, set per computer):

1. **Grab the newest frame** and **shrink it to 64 × 36 pixels**. A still screen sends no new frames, so the last one counts again.
2. **Sort the pixels:**
   - **Colour:** saturation 20.0% or more. Weight = *saturation² × lightness*, so pale colours count much less than vivid ones.
   - **Tint:** 5.0–20.0% saturation, like a bluish or warm white. Kept apart, only for step 6.
   - **Grey and black** (under 5.0% saturation, or brightest channel under 12.0%): ignored.
   - Colours go into 36 hue bins of 10°; tints into the 12 bands. Brightness is the average perceived lightness, as below.
3. **Soften over time:** every bin and the brightness is a running average with the preset's **Softening** as its time constant (Off–5 s, default 1 s). Averaging bins rather than pixels means a cut from red to green fades from one to the other, without a yellow that was never on screen.
4. **Boost:** each bin is multiplied by its band's **colour boost** (0× ignores the band, up to 3×). Each band's slider covers the three bins nearest its centre.
5. **Pick the strongest 30° band** (a bin with its two neighbours). A new colour has to beat the one showing by 25.0% to take over, so near-equal colours don't flip back and forth.
6. **Not enough colour?** If less of the screen than **Minimum colour** (default 3.0%) is clearly coloured in that band, the **favourites** stand in. Each band scores *favourite level (0–5) × (1 + 3 × share of the screen's faint tint in that band)*, so a warm-white room tips towards a warm favourite. The winner is shown at the preset's **strength** (default 60.0% saturation). All favourites at 0 gives white. The same 25.0% margin applies here, and between colour and no colour.
7. **The final colour:** the boosted average hue inside the band, and its average saturation × 1.3 (capped at 100%), because LEDs look washed out at screen saturation.

One scan takes about 0.1 ms.

**Presets** hold the boost, favourites, strength, minimum colour and softening. Built-in: Neutral (the default), Nature (green and lime up, cyan and sky down) and Warm (red and orange up, cyan and blue down, orange favourite). Changing a built-in makes your own copy, saved per person in Firestore at `users/<name>/config/screen`, so it can be tuned from the phone while the computer sends.

### Examples (Neutral unless stated)

| Scene | Simple | Advanced |
|---|---|---|
| Trees under a bluish-white sky | sky blue (212°) | green (110°) |
| Trees under a light-blue sky | sky blue (210°) | green (110°) |
| Half red, half green | red | red; green with Nature |
| Grey office | white | white; orange with Warm |
| Warm-white room, favourites Purple 5 and Orange 3 | white | orange |
| Cut from red to green | jumps | slides over, switches 0.8 s after the cut |

## Simple detection (Advanced colour detection off)

Twice a second, or as often as the "at most every" slider allows sending:

1. Shrink the frame to 64 × 36 and measure brightness: the average of `0.2126 R + 0.7152 G + 0.0722 B`, turned into 10–100% with a square-root curve, so dark scenes dim the lights without switching them off.
2. Skip near-black pixels (brightest channel under 12.0%) and greys and whites (channels within 8.0% of each other).
3. Weigh the rest by *saturation × lightness* and sort them into 36 hue bins of 10°.
4. Pick the strongest bin with its two neighbours. Under 3.0% of the screen's weight gives **white**.
5. Hue is the weighted average in that band; saturation the weighted average × 1.3.

## After that (both detectors)

- **Sending:** a big change (hue moves 40° or more, colour↔white, or brightness moves 25 points) is sent as soon as the "at most every" slider allows. Smaller visible changes wait for the "every" interval. An unchanged screen checks in every 30 s. Scanning more often doesn't send more.
- **Fading:** the phone fades the lights to each new colour over the Smoothness time, taking the short way round the colour wheel.
