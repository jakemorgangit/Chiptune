# Chiptune

![No dependencies](https://img.shields.io/badge/dependencies-none-93f28a?style=flat-square)
![Web Audio API](https://img.shields.io/badge/built%20on-Web%20Audio%20API-5fe1e8?style=flat-square)
![Size](https://img.shields.io/badge/size-15%20KB-ffd35c?style=flat-square)

A tiny NES-style music and sound-effect engine for browser games. Everything is synthesised
live with the Web Audio API: no audio files and no dependencies.

![The demo page playing the Boss Fight song](docs/demo.gif)

Four channels mirror the NES sound chip:

| Channel | Sound | Typical use |
|---|---|---|
| `pulse1`, `pulse2` | Square waves, 12.5 / 25 / 50 / 75% duty | Lead melody, harmony, echo |
| `triangle` | Triangle wave | Bass |
| `noise` | Filtered noise | Drums: `k` kick, `s` snare, `h` closed hat, `o` open hat, `c` crash |

## How it works

Each note is a short-lived oscillator or noise burst with its own volume envelope. A look-ahead
scheduler queues notes about 120 ms ahead of the audio clock, so timing stays tight even when the
game's main thread is busy.

```mermaid
flowchart LR
    song["Song text<br/>C5:2 E5:2 G5:4"] --> sched["Look-ahead<br/>scheduler"]
    sched --> p1["Pulse 1<br/>lead"]
    sched --> p2["Pulse 2<br/>harmony / echo"]
    sched --> tri["Triangle<br/>bass"]
    sched --> nz["Noise<br/>drums"]
    p1 & p2 & tri & nz --> music["Music bus<br/>per-channel mute"]
    fx["chip.sfx('coin')"] --> sfxbus["SFX bus"]
    music & sfxbus --> master["Master volume"]
    master --> scope["Analyser<br/>oscilloscope"]
    master --> out(["Speakers"])
```

## Demo

Open `index.html` in a browser. Press **Space** to play or stop, and keys **1–8** for sound effects.
The page shows a tracker view of the song, per-channel mutes and a live oscilloscope.

![The demo page: tracker, song picker, channel strip, oscilloscope and sound-effect pad](docs/screenshot.png)

## Use it in a game

```html
<script src="chiptune.js"></script>
<script>
  const chip = new ChipTune();

  // Browsers block audio until the player clicks or presses a key.
  startButton.onclick = () => {
    chip.unlock();
    chip.play(ChipTune.songs.overworld);
  };

  // In your game logic
  if (player.touches(coin)) chip.sfx('coin');
  if (boss.awake) chip.play(ChipTune.songs.boss);
  if (paused) chip.stop();
</script>
```

### API

| Call | Does |
|---|---|
| `chip.unlock()` | Starts the audio context. Call it from a click or key handler. |
| `chip.play(song)` | Plays a song on loop, replacing any song already playing. |
| `chip.stop()` | Fades out and stops the music. |
| `chip.sfx(name)` | Plays a sound effect over the music. |
| `chip.setVolume(0..1)` | Sets the master volume. |
| `chip.setMuted(channel, bool)` | Mutes or unmutes one music channel. |
| `chip.onStep(fn)` | Calls `fn({ step, time, song })` for every sixteenth note, for syncing visuals. |

Included songs: `overworld`, `dungeon`, `boss`.
Included sound effects: `coin`, `jump`, `laser`, `hit`, `explosion`, `powerup`, `levelup`, `select`.

## Write your own song

One step is a sixteenth note. Pitched lines are `NOTE:steps` tokens with `r` for a rest.
Drum lines take one character per step, with `.` for silence. Shorter lines repeat to fill the song.

```js
const myTheme = {
  name: 'Title',
  bpm: 132,
  pulse1:   { duty: 0.25, vol: 0.15, line: 'C5:2 E5:2 G5:4 A5:2 G5:2 E5:4' },
  triangle: { vol: 0.3, line: 'C3:4 G2:4 A2:4 G2:4' },
  noise:    { vol: 0.3, line: 'k.h.s.h.k.h.s.hh' },
};
chip.play(myTheme);
```

Channel options: `vol`, `duty` (pulse only), `sustain` (0–1 level after the attack),
`gate` (fraction of each note that sounds) and `delay` (shift a line by N steps, handy for echoes).

## Limits

Browsers slow down timers in background tabs, so music can stutter while the tab is hidden.
