# Why one light lags behind (gorilla version, 10/28/26)

> **Built in website v2.8:** Settings → Bluetooth sending (*As today* / *Confirmed* / *Throttled*), a per-light readout in Diagnostics, and the setting plus each light's numbers in the heartbeat. Fixes 3 and 4 aren't built.
>
> **Website v2.9–v3.1, app v1.8–v1.9:** faster Bluetooth turns (fix 3) is a setting; the Mixed sending option, a lag watchdog that reconnects a light falling behind, and pausing nearby scans while lights stream were added too.

## 1. What you saw

- **Dragging the colour wheel:** most strips jump straight to the final colour. Now and then one strip walks through the in-between colours (red → green → blue), late.
- **Follow my screen:** the same, but worse. One strip replays old colours (red, blue…) for seconds after the others are already white.
- **Your questions:** should we throttle? Do the lights tell us they got a command?

## 2. What's happening

Think of every colour change as a **letter**.

- **The page waits per light:** it gives each light one letter at a time and waits for the phone to say "done" before giving it the next. While it waits, a newer colour replaces an unsent one, so the page itself never piles up letters.
- **"Done" doesn't mean delivered:** for colour and brightness we use *write without response*, so "done" from Android only means **"it's in my Bluetooth outbox"**. The light hasn't got it yet.
- **Bluetooth never loses or reorders a letter while connected.** If a light's signal is weak (distance, walls, the way the strip faces), the radio re-sends until it gets through. So nothing is lost, it's just **late and in order**. The lagging light shows every old colour one by one: that's your "historical curve".
- **We can't empty that outbox:** it lives inside Android's Bluetooth system, below our app, so the app can't take old letters back out.
- **Why it's one random light:** whichever light has the weakest signal at that moment. The healthy ones empty their outbox right away.
- **Why Follow is worse:**

  | | Letters per second, per light | With your 5 lights |
  |---|---|---|
  | Dragging the wheel | up to 20 (colour) | up to 100 |
  | Follow my screen, while fading | up to 40 (colour + brightness) | up to 200 |

  All 5 lights share one phone radio.
- **Not the cause:** the 20-second catch-up. It keeps only the *newest* change for a light that's offline, never a history.

## 3. Is there feedback?

Bluetooth has two ways to write:

| Way | Does the light answer "got it"? | We use it for |
|---|---|---|
| Write **with** response | Yes: the light's Bluetooth chip confirms each one, one at a time | On/off, scenes, mic mode |
| Write **without** response | No, only the phone's "it's in my outbox" | Colour and brightness (fast streams) |

"Got it" means the light's Bluetooth chip received it, not that the LEDs changed. It's still the best signal a light gives.

## 4. Fixes people use (best first)

1. **Confirmed sending, latest wins (recommended).** Send colour and brightness *with* response. Each light then has at most one colour on its way. While it waits, newer colours replace older ones; the page already does that part. Healthy lights stay smooth, about 15–30 updates a second. A weak light simply **skips to the newest colour** instead of replaying history. This is called back-pressure, the same trick video calls use: drop old frames rather than lag. It keeps your smooth drag.
   *Catch:* a light that can't do write-with-response falls back to fix 2.
2. **Throttle each light**, for example 10 colours a second. The simplest fix: fewer letters means a smaller pile-up. But a really weak light can still fall behind, and healthy lights look a bit steppier.
3. **Faster radio turns** (Android's "high connection priority") while dragging or following. Each light gets a turn every ~11–15 ms instead of every ~30–50 ms, so outboxes empty faster. It costs some battery, and some lights refuse it. It helps alongside the others but isn't a fix on its own. It needs a new APK; the others are website-only.
4. **Fewer letters in Follow:** fade in 10 steps a second instead of 20, and skip steps the eye can't see. That roughly halves the load.

## 5. Plan for the A/B test

- **Setting:** "Bluetooth sending" with three choices: *As today* / *Confirmed (latest wins)* / *Throttled (10 a second)*. It stays on *As today* until you've compared.
- **Diagnostics, per light:** updates sent per second, average time until "got it", and colours skipped.
- **Sync events:** the chosen setting goes into the heartbeat, so the viewer can compare the two.
- **The test:** stand near your weakest strip and drag red → green → blue fast, then try Follow on a busy video. With *Confirmed*, every strip should land on blue at the same moment.

## Sources

- [Write requests vs write commands](https://punchthrough.com/?p=11464) (Punch Through)
- [Connection interval and throughput](https://punchthrough.com/ble-connection-interval-throughput/) (Punch Through)
- [High priority gives an 11.25–15 ms interval; phones can still refuse](https://devzone.nordicsemi.com/f/nordic-q-a/18579/slow-ble-throughput-with-nexus-5x) (Nordic DevZone)
- [`requestConnectionPriority`](https://developer.android.com/reference/android/bluetooth/BluetoothGatt) (Android docs)
- [Android drops writes sent without waiting for the callback](https://devzone.nordicsemi.com/f/nordic-q-a/4038/delay-in-writing-characteristics-in-android-4-3) (Nordic DevZone)
- [A BLE LED that plays queued commands slowly, in turn](https://mbientlab.com/community/discussion/comment/1313) (MbientLab forum, the same symptom on another device)
