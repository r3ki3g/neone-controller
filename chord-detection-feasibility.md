# Chord detection and song recognition: feasibility

Checked 10/23/26.

## Short answer

| Idea | Feasible? | Notes |
|---|---|---|
| Show the chord playing right now, from the phone mic | ✅ Yes, in real time | Light on the CPU. Accuracy is the catch: good for simple major/minor chords in clear music, shaky with heavy drums, loud vocals or a noisy room. |
| Recognise a song from its chord progression | ❌ Not reliably | Thousands of songs share the same progressions. For example, I–V–vi–IV is in countless pop songs. |
| Recognise a song you recorded a show for, and start that show | ✅ Yes | Uses audio fingerprinting (the Shazam method), matched only against your own saved shows, on the phone. |
| Recognise any song in the world | ⚠️ Only with a paid cloud service | Sends audio off the phone. Out of scope. |

## How it would work

1. The mic feeds the Web Audio API, which phone-mic mode already uses.
2. About 10 times a second, the page measures how strongly each of the 12 musical notes is sounding. This is called a "chroma" profile.
3. That profile is compared with every major and minor chord. The page shows the best match only when it has been confident for about 0.5 s, and "—" otherwise. Expect roughly half a second of lag.
4. Optionally, each chord gets its own colour, as a "Chord colours" music pattern.
5. For song matching, recording a show also saves a fingerprint of the song. Later, about 5–10 s of listening finds the matching show and starts it at the right point.

## Libraries

| Library | What it does | Licence |
|---|---|---|
| [Meyda](https://meyda.js.org) | Real-time chroma in the browser | MIT |
| [Tonal](https://github.com/tonaljs/tonal) | Turns a set of notes into a chord name | MIT |
| [Essentia.js](https://essentia.upf.edu) | Full chord estimation (`ChordsDetection`), compiled to WebAssembly | AGPL-3.0 |
| [Olaf](https://github.com/mediainbox/Olaf) | Acoustic fingerprinting, with a working [browser demo](https://0110.be/posts/Acoustic_fingerprinting_in_the_browser_with_Olaf) | AGPL-3.0. Its README warns about patents in some regions. |

AGPL is fine here because the repo is already public. Anything built on these libraries must stay open source.

## Will the browser crash?

Very unlikely. The analysis runs on small, fixed-size audio buffers, in a background audio thread (AudioWorklet) rather than the page's main thread. The realistic risks:

- **Battery and heat:** analysing 10 times a second is modest, but it adds up over an hour. The feature would be off by default.
- **Download size:** Essentia.js and Olaf add a few MB. They load only when the feature is switched on.
- **Worst case:** the page gets sluggish, or Android reloads the tab when memory is low. That's not a browser crash, and switching the feature off fixes it.
- **Screen off:** pauses the analysis, the same as phone-mic mode today.

## Recommendation

1. **Phase 1:** live chord display plus a "Chord colours" pattern, using Meyda and Tonal. These are small and MIT-licensed.
2. **Phase 2:** recognise songs for your saved shows with Olaf, only if Phase 1 turns out accurate enough in your room.

## Update 10/23/26: built, then removed

Both phases were built, without any of the libraries above. They worked in simulated tests but didn't work with real music through the phone's mic, so chord detection, colouring by chord and song recognition were removed from the app the same day. Recording colour changes and replaying them stay.
