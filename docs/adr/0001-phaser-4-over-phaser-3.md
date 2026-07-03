# Phaser 4 over Phaser 3

Phaser 4 (4.2.0, stable since April 2026) was chosen over Phaser 3 (last release 3.90.0, May 2025) for all new development. Phaser 3 is in maintenance mode with no further updates; `npm install phaser` defaults to v4; the official Vite+TS template targets v4.

The deciding factors were not feature differences (both cover the GDD's requirements) but lifecycle position: starting a new project on a dead branch means accumulating unpatched bugs and missing future ecosystem improvements. Phaser 4's SpriteGPULayer (1M+ sprites, single draw call), unified Filter system, and 6-mode tinting are bonuses, not the reason — avoiding a version migration mid-project is the reason.
