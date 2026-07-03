# Raw Web Audio API over audio libraries

Audio is built on the raw Web Audio API (OscillatorNode, GainNode, BiquadFilterNode) for sustained/continuous sounds, with ZzFX-style parameter arrays for one-shot SFX. No Tone.js, Howler.js, or Phaser's built-in audio manager.

The GDD requires two fundamentally different audio patterns: one-shot triggers (jump, death, pickup) and sustained tones that start/stop in sync with game state (Magnet Hands hum, saw buzzing, Genie Lens tick). ZzFX handles one-shots well but has no concept of "keep playing until I say stop" — it generates fixed-duration buffers. Tone.js handles both but pulls in a full music-sequencing framework (~150KB) with transport scheduling and effects chains the game doesn't need.

Raw Web Audio gives direct control over frequency-band routing (hazards-low/value-high/relief-mid enforced by per-category BiquadFilterNode chains), oscillator lifecycle (create on sound start, discard on stop, cap at 16-24 concurrent voices), and pause integration (AudioContext.suspend()/resume() tied to the portal SDK adapter's ad-break mechanism).

The trade-off: every sound is programmed, not designed in a GUI tool. This is accepted as consistent with the procedural-visuals decision (ADR-0002) — the entire game is code-generated, audio included.
