# Procedural visuals with no image assets

All game visuals are generated at runtime via Phaser's `Graphics` API, baked into reusable textures with `generateTexture()` at boot. No PNGs, no spritesheets, no atlas pipeline, no image editor in the toolchain. This is a permanent art direction, not placeholder art.

Considered alternatives: Aseprite pixel art with TexturePacker atlases (industry standard for 2D web games), or hand-drawn PNGs. Both were rejected because the game's visual identity is geometric and color-coded (the Honesty Split's coral/mint/gold/bronze palette), not illustrative. Procedural generation eliminates the entire asset pipeline (load-time budget, atlas packing, artist tooling, contributor workflow), which is disproportionately valuable for a solo/small-team project targeting web portals with load-time constraints.

The trade-off: artistic range is permanently constrained to what `Graphics` primitives can express. Complex character art, detailed textures, and hand-drawn animation are off the table. This is acceptable because the GDD's visual language is deliberately simple — shapes and colors carry gameplay meaning, not aesthetic detail.

Clarification: dynamic `Phaser.Text` objects (e.g., Bus Stop countdown signage, HUD numbers) are not "image assets" in the sense this decision prohibits. They are runtime-generated text renders, not pre-authored art files. This ADR constrains the art pipeline, not the use of Phaser's text rendering API.
