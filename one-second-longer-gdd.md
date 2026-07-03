# ONE SECOND LONGER — "The Reject Bin"
## Complete Game Design Document · v1.0

*Builds directly on the shipped playable prototype (one-second-longer.html). Sections marked ✅ exist in the build; everything else is specced here for implementation.*

---

# 1 · Game Concept

## Premise

You are locked inside **the Reject Bin** — the room where a game studio dumps every object that failed QA. Every few seconds, a chute in the ceiling drops another reject into the room: a spike that was "too mean," a power-up that was "buggy," a platform that "had attitude," an enemy that "wouldn't stop hugging testers." Some rejects are dangerous. Some are useless. Some are *mislabeled* — and the label is the lie.

The room's compile timer reads **5:00**. When it hits zero, the **SHIP IT door** unlocks. Your job is simple: still be functional when it opens, and cross whatever the room has become.

Dying doesn't reset the clock. Dying just respawns you into a room that has only ever gotten worse.

## Why this theme

The premise isn't decoration — it *licenses every mechanic in the brief*:

- **A mixed-quality spawn stream** (helpful/harmful/neutral/chaotic) is canon: rejects were rejected for different reasons.
- **Trap-disguised-as-reward** is canon: a mislabeled reject isn't the game cheating, it's the object's documented defect.
- **The anti-idle rule** is canon: the QA system flags AFK testers.
- **Infinite respawns** are canon: you're in a test chamber; the chamber doesn't care.
- **The 5:00 door** is canon: builds take five minutes.

## Design pillars

1. **The room is the villain, the clock is its heartbeat.** Difficulty is never authored per-second by a designer's hand; it *accumulates*. Every individual object is fair. The pile is not.
2. **Colors never lie. Labels do.** (The Honesty Split — see §3.3.) Every mechanical channel — color, shape, motion, telegraph — is always truthful. Every *linguistic* channel — names, signs, labels, arrows — is allowed to lie. Players learn to trust physics and distrust marketing. This is the entire troll contract in one rule.
3. **Punish the spot, not the player.** Consequences prefer to poison *places* and *choices* rather than delete progress. (See the Monument, §2.4.)
4. **You cannot lose. You can only still be in the room.** No game-over screen exists. The failure state is the room itself.

## Player emotion targets

The spawn ecosystem is deliberately tuned to cycle four emotions — every object in the catalog (§4–5) is tagged with its primary target:

- **FUN** — skill-rewarding, satisfying interactions
- **TROLL** — playful misdirection with a readable tell
- **RAGE** — high-tension, punishing-but-fair pressure
- **ENGAGE** — curiosity, reactivity, risk/reward decisions

---

# 2 · Core Rules

## 2.1 Movement ✅

Values from the shipped prototype (960×540 logical canvas):

| Parameter | Value | Note |
|---|---|---|
| Run speed | 232 px/s | constant, no acceleration ramp |
| Jump velocity | −624 px/s | ~105 px apex, ~0.67 s air time |
| Gravity | 1700 px/s² | fall capped at 760 px/s |
| Coyote time | 0.09 s | jump grace after leaving a ledge |
| Jump buffer | 0.10 s | early-press grace before landing |
| Player size | 22×30 px | one hitbox, no crouch in this game |

Movement is deliberately *simple and honest* — no wall-jump, no dash by default (those arrive only as temporary power-ups). The player's baseline verb set must stay small so that spawned objects are what changes, never the controls.

## 2.2 The clock

- The session clock runs **0:00 → 5:00** in real time and **never rewinds or pauses on death**. ✅
- **Time tokens** (±10s, §4.4) can edit the *remaining* time. Net subtraction is capped at **60 s** — the door can never open before 4:00. Net addition is uncapped (the room is happy to keep you).
- Spawning ceases at 5:00. The saturated room persists — the final crossing happens through everything that accumulated. ✅

## 2.3 Death & respawn ✅

- One-touch death. Respawn is instant (~0.2 s) at the **spawn pocket** (left wall, x < 100).
- The spawn pocket is a guaranteed no-spawn zone, verified in simulation: an idle player standing there is never killed by any scheduled object across the full 5:00. *(Amended by the anti-idle rule below — the pocket grants a grace ticket, not immunity.)*
- Un-banked collectibles scatter on death (§4.5). Nothing else is lost. The clock does not care.

## 2.4 The Anti-Idle Rule — "The Monument"

**Rule:** the player must displace at least **48 px horizontally** within every rolling **5.0-second** window. Vertical hops in place do not count — the check is horizontal, so wiggle-cheese and bunny-hop-camping both fail.

**Escalation ladder:**

| t (stationary) | Consequence |
|---|---|
| 3.0 s | Warning: an **"AFK?"** tag appears over the player; the floor tile underfoot glows gold. Fully readable, fully escapable. |
| 5.0 s | **The Monument.** A chalk statue *of the player* erupts from the ground exactly where they stood — a permanent spike-hazard for the rest of the session, sculpted in their honor. The player is shoved sideways (small knockback), **not killed**. |
| Repeat offenses | Every Monument after the first also releases a **pigeon** — a slow homing bird that harasses the player for 6 s before despawning. Serial campers get a statue *and* an escort. |

**Exemptions:**
- **Respawn grace:** 8 s of idle allowance after every respawn (so deaths never chain into Monument punishment).
- **Bus Stops** (§4.3): sanctioned shelters that grant a visible countdown ticket.
- **Coffee Breaks** (§4.4): the *only* free idling in the game is during the union-sanctioned break. Even the anti-idle system respects the union.

**Why this mechanic (reasoning the brief asked for):**

Straight damage-on-idle was rejected because it feels arbitrary — a death with no object to blame reads as the game cheating, violating pillar 2. Forced movement (conveyor floors, push-winds) was rejected because it takes the controls away, violating the free-movement goal. The Monument was chosen because it converts the punishment into *worldbuilding*: the consequence of camping is that the camper personally makes the room worse, permanently, in the shape of their own laziness. It is self-documenting (a statue of you explains itself with zero UI), it compounds naturally with the room-filling fantasy, it's darkly funny (your "trophies" accumulate), and it punishes the *spot* rather than the player — the first offense costs you a safe location, not your life. It also feeds the exit-accessibility system: Monument placement passes through the same solvability veto as every permanent spawn (§6.2), so campers can degrade the room but never brick it.

## 2.5 Win / loss conditions

- **Win:** be touching the SHIP IT door at any moment after it unlocks at 5:00 (or earlier via −time tokens, floor 4:00). ✅
- **Loss:** none. There is no fail state, no lives system, no game over. Quitting mid-session banks your best stats.
- **Score (session summary / share card):** furthest-% reached · deaths · bits banked · hazards outlived · net time edited · Monuments erected (shame stat).

## 2.6 Seeding & modes

- **Classic:** one fixed seed forever ✅ — the room is learnable, mastery is memory + execution.
- **Daily Bin:** date-based seed, everyone worldwide gets the same room for 24 h; leaderboard on furthest-% then deaths. (Directly imports "The Daily Devil" concept from earlier brainstorming — the watercooler mode.)
- Determinism is non-negotiable in both: the *stream* is fixed; only the player varies. Chaos objects (§4.7) get their randomness from the seed too — "random" means *unknowable in advance*, not *different per run*.

---

# 3 · The Spawn System (Director)

## 3.1 Architecture

A deterministic **schedule generator** (seeded PRNG ✅) emits a timeline of spawn events before the session starts. Each event has: type, position, parameters, spawn time. The generator works in three layers:

1. **Cadence curve** — interval between spawns shrinks from ~8 s to ~1 s across the five minutes ✅.
2. **Category budgets** — each phase (§3.2) has a % mix across the seven categories; the generator draws types against those weights.
3. **Placement constraints** — reserved corridor, spawn pocket, solvability veto (§6.2), and minimum-spacing rules filter every placement.

Every spawn telegraphs with a **0.7 s gold blink** at its position before becoming live ✅. Nothing in the room ever materializes lethally on top of the player without warning.

## 3.2 The five-minute arc — phase table

| Phase | Window | Cadence | Category bias | Intended feel |
|---|---|---|---|---|
| **Naive Room** | 0:00–0:45 | ~8 s | 40% platforms/coins · 50% simple hazards · 10% relief | "this is cute" |
| **The Fill** | 0:45–2:00 | ~5 s | hazards ramp to 55% · first enemy · first Mystery Crate | leaning in |
| **Crowding** | 2:00–3:15 | ~3 s | enemies + chaos to 25% · relief pulses *before* density spikes | sweating |
| **The Flood** | 3:15–4:30 | ~1.5 s | everything · Buoy Platforms guaranteed · Bus Stops rarer | panic |
| **Last Words** | 4:30–5:00 | ~1 s | maximum chaos, then a cruel 5 s of total silence before the door | held breath |
| **The Crossing** | 5:00+ | spawning stops | the frozen, saturated room is the final exam | the run that matters |

Relief is deliberately scheduled *just before* difficulty spikes, not after — a breather you enjoy while watching the next wave telegraph is tension; a breather after the wave is just downtime.

## 3.3 The Honesty Split (readability law)

Carried over from the Jump Once trap grammar and now codified as the studio-wide contract:

**Mechanical channels always tell the truth:**
- **Coral** = lethal. **Mint** = safe/helpful. **Gold** = value. **Dull bronze** = fake value. Gold blink = incoming spawn.
- Motion telegraphs (wind-up, tremble, blink cadence) always precede harm.

**Linguistic channels are allowed to lie:**
- Labels, names, signs, arrows, and NPC claims may all be wrong — and every lying label carries a *physical* tell (a crooked sticker, an asterisk, a shadow showing the true value ✅ — the "+1 that casts a −1 shadow" from Jump Once L7 is the canonical example).

One sentence for the tutorial card: ***"Read the physics, not the marketing."***

---

# 4 · Game Objects Catalog

Every object below is tagged with its primary emotion target (**FUN / TROLL / RAGE / ENGAGE**) and, where it can deceive, its honest tell. ✅ = already in the prototype.

## 4.1 Platforms

| Object | What it does | Tell / counter-play | Emotion |
|---|---|---|---|
| **Buoy Platform** | Floats at a fixed height above the *tallest hazard beneath it* — as the floor saturates, it rises. The chaos builds its own high road. | Bobbing motion shows it's a buoy; watch what's under it. | ENGAGE |
| **Ghost Scaffold** | Solid only while you're airborne; vanishes the instant you're grounded. Enables jump-chains across the room. | Translucent when you're grounded. | FUN |
| **The Apologetic Platform** | Collapses on landing — then re-forms *under you* mid-fall for exactly one save, as if to say sorry. | Cracks on contact, mint glow on the catch. | FUN |
| **Grudge Plank** | Fine with soft landings. Land on it at high fall speed and it remembers — next touch, it tilts and dumps you. | Visible crack + angry face after a hard landing. | TROLL |
| **Fair-Weather Platform** | Solid while its neighborhood is calm; turns intangible when local hazard density gets high — exactly when you want it most. | It visibly trembles before phasing out. | RAGE |
| **Subscription Platform** | Free for 5 s, then blinks "SUBSCRIBE" and despawns unless you feed it one bit. | Trial countdown printed on its face. | TROLL |
| **Escalator Steps** | A rising stack of steps that carries you up — toward the ceiling spikes if you ride it idly. Transportation, not residence. | Constant upward tick; ceiling is visible. | ENGAGE |
| **Union Platform** | A normal moving platform that stops during Coffee Breaks. It has rights. | Tiny hard-hat decal. | FUN |

## 4.2 Power-ups (and their rejected siblings)

Every power-up has a chance to spawn as its **reject variant** — same silhouette, crooked label, defective behavior. The Honesty Split applies: the *label* lies, the *physical tell* never does.

| Object | What it does | Reject variant (the lie) | Tell | Emotion |
|---|---|---|---|---|
| **Double Jump** | One mid-air jump, 15 s. | **Double Jump\*** — second jump fires *downward* (sign error). | The asterisk on the label. | FUN / TROLL |
| **Speed Boost** | +60% run speed, 10 s. | **Speed Boost (recalled)** — boosts speed *and removes stopping*: ice physics until it expires. | "RECALLED" stamp, skid marks on the box. | FUN / RAGE |
| **Shield Bubble** | Absorbs one hit — but you're in a literal bubble: elastic bounces off every wall while it lasts. | **The Colander** — a shield with holes; blocks ~80% of hits. You never know which 20%. | Visible holes. | ENGAGE |
| **Tiny Mode** | Half size for 12 s — fit under crushers, thread saw gaps; knockback doubles. | — | — | FUN |
| **Magnet Hands** | Pulls collectibles to you in a radius… and also pulls the Magpie's attention (it hears the jingling from farther away). | — | Magnet hum audible to you *and* it. | ENGAGE |
| **Snow Globe** | Freezes all hazard *motion* 4 s (still lethal to touch, but predictable). | — | — | FUN |
| **Slow-Mo Lens** | Slows the room 50% for 5 s. The 5:00 clock keeps real time — the lens is pure gift. | **Genie Lens** — slows the room *and the clock*. You asked for more time; you got it. | Genie Lens ticks audibly slower. | FUN / TROLL |
| **Grapple (targeting bug)** | A grappling hook that only latches onto *hazards*. Genuinely powerful, permanently terrifying: zip toward the saw, release early. | — (it IS the reject; that's the whole item) | Reticle only highlights coral objects. | ENGAGE / RAGE |
| **Extra Life** | In a game with infinite respawns: does nothing. Enormous fanfare on pickup. A trophy of pure comedy. | — | Its label says "FINALLY, VALUE." | TROLL (affectionate) |
| **Moonwalk Boots** | You move normally; your sprite faces backwards. Zero mechanical effect. Chaos for streamers. | — | — | TROLL (harmless) |

## 4.3 Relief & Recovery

| Object | What it does | Design note | Emotion |
|---|---|---|---|
| **Bus Stop** | A small shelter: no spawns land inside, and it grants a visible **idle ticket** (8 s of sanctioned stillness). A sign reads "NEXT BREAK: 0:47" — the next Coffee Break, truthfully. | The pressure valve for the anti-idle rule. Rarer in late phases. | FUN |
| **The Intern's Broom** | Sweeps one horizontal lane clear — despawns the oldest hazards in that row. The room visibly gets *cleaned* for the only time all session. | Single-use, spawns mid-session; feels miraculous by 3:00. | FUN |
| **Complaint Form** | Pick it up and the hazard type that *last killed you* gets recalled for 20 s — every instance deactivates and wears a traffic cone. | Relief personalized to the player's actual pain. | ENGAGE |
| **Ghost Pepper** | 3 s of invincibility — but you're forced to run at full speed the whole time. Relief that refuses to let you rest. | Anti-idle-compatible mercy. | RAGE / FUN |
| **The Referee** | An NPC strolls across blowing a whistle; all *enemies* freeze and pretend to be doing nothing until he exits. | Comedy relief; hazards unaffected, only actors. | FUN |
| **Sympathy Pizza** | See Mercy Rule (§4.4). Worth points. It's also just nice. | — | FUN |

## 4.4 Cooling-Down Systems (the level "breathes")

| System | What it does | Design note |
|---|---|---|
| **Coffee Breaks** ("union breaks") | At fixed schedule points (~1:30 and ~3:00), a 10 s window: spawning pauses and all *moving* hazards stop and sip tiny coffees. Static spikes remain (they're not in the union). The **only** time idle is free. | Scheduled valleys make the pacing legible; players learn to plan routes during breaks. Fixed times = learnable in Classic, part of the daily puzzle in Daily Bin. |
| **Time Tokens (±10 s)** | Spawned tokens that edit the master clock. **−10 s** tokens are gold-bright, rare, and always placed somewhere expensive (deep in danger). **+10 s** tokens are the room's favorite lie — shiny, convenient, and they *add* to your sentence. | The brief's "add or subtract countdown" requirement. Net −60 s cap (door floor at 4:00). Honest tell: −tokens tick *down* audibly, +tokens tick *up*. Labels may lie; the ticking never does. |
| **Mercy Rule** | 5 deaths inside 15 s → the room feels bad: 6 s spawn pause + a Sympathy Pizza spawns + banner: "WOW. OK. BREAK TIME." | Dynamic difficulty done *loudly* — hidden rubber-banding erodes trust; announced pity is comedy and mercy at once. |
| **Last Words silence** | The final 5 s before 5:00: total spawn silence. The room holds its breath with you. | The cruelest kindness — a designed calm that makes the door-opening moment land. |

## 4.5 Collectibles & Economy

| Object | What it does | Risk design | Emotion |
|---|---|---|---|
| **Bits** (screws & sparks) | Base currency/points. Carrying them makes you *jingle* — audible to the Magpie. | Hoard = louder = hunted. | ENGAGE |
| **The Piggy Bank** | Spawns periodically; deposit carried bits to make them permanent. | Banking runs create voluntary detours through danger. | ENGAGE |
| **Death scatter** | Un-banked bits scatter on death — and *roll toward the nearest hazard*, because the room is cruel. Recoverable if you're brave and fast. | The brief's "can be taken away," version 1. | RAGE |
| **Golden Sticker** | Rare, high value, and literally sticky: −8% acceleration while carried, until banked. | Value with weight — sprint home or waddle rich. | ENGAGE |
| **The Bill** | Coin-shaped at a distance; subtracts points on pickup. | Tell: real coins *spin*; the Bill *flutters* like paper. | TROLL |
| **Dev Coin** | A coin with the dev's face. Collect 3 in one session → unlocks a cosmetic. | Meta-progression breadcrumb. | FUN |

## 4.6 Enemies / Attackers

| Enemy | Behavior | Counter-play | Emotion |
|---|---|---|---|
| **The Magpie** | Flying thief. Dives for un-banked bits; if you're broke, steals your active power-up; if you have *nothing*, displays a speech bubble: "broke." then leaves. | Bank often; bait its dive and sidestep — it needs a straight line. | ENGAGE / TROLL |
| **The Fan** | Sprints at you to *hug* you. The hug does no damage — it pins you for 1.5 s. Being hugged counts as stationary for the anti-idle clock, and the room is full of saws. | Jump over the lunge; it can't jump. It cries a little. | RAGE (funny) |
| **The Inspector** | Slow patroller. Catches you idling >2 s → writes a ticket (points fine) *before* the Monument ladder reaches you. A walking soft-warning. | Keep moving; he only fines the stationary. | TROLL |
| **The Copycat** | Mirrors your inputs from the opposite side of the room. Harmless to touch — but it *collects any bits it walks over*. A rival, not a killer. | Route so your mirror-path avoids coin clusters; or use it: mirror it into collecting the Bill. | ENGAGE |
| **The Fixer** | Never touches you. Walks the room *repairing* things: re-arms spent crushers, refills used relief spawns as hostile, patches broom-cleaned lanes. Makes the room worse, forever. | No combat exists — lure it over a pit or into a Monument. Indirect murder only. | RAGE / ENGAGE |
| **The Balloon Seller** | Wanders across selling balloons. Touch one → you float uncontrollably upward for 3 s. Sometimes that's death by ceiling spikes; sometimes it's the only way over a saturated floor. | It's a tool wearing an enemy costume. Timing is everything. | ENGAGE / TROLL |

## 4.7 Chaotic / Random Spawns

Determinism note (§2.6): all "randomness" is seeded — unknowable in advance, identical across runs of the same seed. Chaos you can *study*.

| Object | What it does | The gamble | Emotion |
|---|---|---|---|
| **Mystery Crate ?** | Contains any object from any category. Its shake animation is a learnable language — gentle wobble = benign, violent rattle = hazard. | Open it or route around it; late-game crates block lanes. | ENGAGE |
| **The Gacha Door** | A second door that appears somewhere, opens for 2 s. Entering teleports you to a seeded-random location: sometimes a clutch escape, sometimes inside a saw's orbit. | "Do you trust the door?" is a question the room asks forever. | ENGAGE / RAGE |
| **Schrödinger's Coin** | Visibly oscillates coin ↔ coin-shaped-spike once per second. Touching it locks whichever state it's in. | Not RNG — a *timing* test dressed as a gamble. | FUN / TROLL |
| **The Regifter** | A present box. Opening it spawns the last object type you visibly *avoided* this session. The room noticed your cowardice. | Face your fears or leave the box forever sealed. | TROLL |
| **Weather Vane** | Touch to trigger a seeded 5 s room-wide modifier: low gravity · icy floor · dark room (spotlight on you) · big-head mode (cosmetic) · reversed conveyor floor. | Player-consented chaos — it only fires if touched. | ENGAGE |
| **The Dice** | A giant die bounces through; the face it lands on dictates the next 10 s of the spawn table (ALL PLATFORMS / ALL HAZARDS / ALL COINS / NOTHING / DOUBLE SPEED / ???). | Public randomness — everyone sees the roll and braces together. | ENGAGE |

---

# 5 · New Original Hazard Brainstorm

Twelve hazards built to be *not commonly seen* in 2D platformers — each punishes a **state, choice, or history** rather than mere position, which is where the genre's unexplored space lives. All obey the Honesty Split.

1. **The Auditor** — a slow scanning beam that fines (points) or zaps anything it catches *airborne*. The inverse of every laser ever: it punishes jumping, not standing. Creates grounded-rhythm windows in a game that otherwise screams "jump." *Avoid:* be on the floor when it sweeps; its schedule is printed on its housing.
2. **The Metronome** — a pendulum blade whose swing speed **mirrors your movement speed**. Sprint and it becomes a blur; walk and it lazes. You are your enemy's tempo. *Avoid:* approach slow, pass slow — patience is the dodge.
3. **The Souvenir Crown** — a session rule, not an object: any hazard that kills you gains a tiny crown and +10% size for the rest of the session. Your killers become famous. The room's final form is a museum of your specific failures. *Avoid:* stop dying to the same thing (the game's most honest advice).
4. **Poltergeist Furniture** — a chair that only moves when you're *facing away* (facing direction, not camera). Turn around and it's frozen mid-scoot, feigning innocence. Weeping-angel logic in a 2D room. *Avoid:* moonwalk past it; keep it on-screen-side of your eyes.
5. **The Echo** — your own movement from 10 seconds ago replayed as a lethal chalk ghost. Camp a lane twice and you collide with your own history. *Avoid:* never repeat a route within its memory; the anti-idle rule's spiritual sibling.
6. **The Understudy** — spawns as a blank mannequin that *watches*. After 20 s it transforms into a copy of whichever hazard type has killed you most, and wears that hazard's mask while studying — so you always know what's coming. *Avoid:* it telegraphs its final form the whole time; pre-plan its neighborhood.
7. **Velcro Walls** — wall patches that grab you on airborne contact. Being stuck counts as *stationary* for the anti-idle clock. Mash to tear free. Two systems (walls, idle rule) conspiring is scarier than either alone. *Avoid:* the patches glisten; don't wall-hug.
8. **The Queue** — a line of tiny NPCs shuffling across the floor. Harmless, but **solid** — a slow-moving wall you cannot push through, because they are queueing and you will *not* cut in line. Terrain as comedy. *Avoid:* jump the queue (the only place that phrase is allowed).
9. **The Fire Alarm** — a pullable lever. Pulling it makes every hazard panic and retract for 4 s (total relief *now*) and silently adds **+3 hazards** to the future spawn schedule (debt *later*). A payday loan you can pull mid-crisis. *Avoid/Use:* it's never mandatory; it's always tempting.
10. **Tide with Manners** — a hazardous floor-goo that creeps in from one wall, but *politely pauses for 2 s whenever it would corner you*, tipping an imaginary hat, then continues. The pause guarantees a near-miss escape window — constant almost-death, engineered. *Avoid:* use its manners; it never lies about the pause.
11. **The Balloon Cloud** — burst any balloon (see Balloon Seller) near it and it sneezes a 3-balloon flurry that lifts *hazards* off the floor and drops them elsewhere. Terrain-shuffling via slapstick. *Avoid:* pop responsibly.
12. **Static Cling Storm** — a 6 s field: collectibles within it stick to your body (great) and so do hazard *fragments* (spike tips, saw teeth — each one a small hurtbox you now wear). Exit the storm to shed them, in reverse pickup order. Greed becomes literal armor of consequences. *Avoid:* shop fast, leave fast.

---

# 6 · Level / Arena Design Guidance

## 6.1 Single-screen principles

- **One room, no scroll** ✅. Every death, every spawn, every mistake is permanently visible — the room *is* the progress bar. Never break this: the fantasy is watching emptiness become impossibility in one unbroken shot.
- **The floor is the stage, the air is the escape.** Ground-level saturates first (spikes, pits, queues); verticality (buoys, scaffolds, balloons) becomes the late-game — the room teaches you to leave the ground the way water teaches you to swim.
- **Fixed landmarks:** spawn pocket (left), SHIP IT door (right), chute (top-center). Everything else is spawned. Players need exactly three permanent truths to build mental maps around.
- **Density ≠ difficulty by itself.** The prototype curve (5 → 10 → 23 → 40 → 65 → 107 objects ✅) works because early objects are *simple* (static spikes) and complexity ramps *with* count. Never spawn the Fixer in the Naive Room.

## 6.2 Exit accessibility — the three guarantees (brief requirement 3)

1. **The Reserved Corridor** ✅ (partially): the door-approach column (x > 895) and the spawn pocket (x < 110) never receive *permanent* spawns. Mobile hazards (saws, enemies, the Queue) may *transit* — the exit is never quiet, but it is never *walled*.
2. **The Solvability Veto:** before any permanent spawn commits (pit, monument, static spike cluster), the director runs a coarse reachability check — a nav-grid flood-fill using the real jump arc (105 px apex / 140 px flat reach) from pocket to door. Blocked → the spawn relocates to its nearest legal position. This is cheap at our scale and we have already proven the methodology: the shipped prototypes were validated with exactly this kind of simulation harness, which caught three real geometry bugs by *playing* the game headlessly.
3. **The Emergent High Road:** from The Flood onward, the director guarantees ≥1 Buoy Platform per 45 s, so a ceiling route assembles itself precisely as the floor becomes untenable. The room closes the ground door and builds the sky door — accessibility maintained *diegetically*, not by fiat.

Plus one soft valve: if furthest-% hasn't passed 50% in 60 s, helpful-category spawn bias shifts toward the right half of the room. Announced nowhere, felt everywhere.

## 6.3 Readability under saturation

At 100+ objects, fairness lives or dies on legibility:

- Hard cap on *simultaneously animating telegraph blinks* (stagger the schedule so ≤2 spawns telegraph at once).
- Color budget enforcement: coral is *never* used decoratively ✅; late-game visual pressure comes from the vignette, not from hue pollution.
- Every mobile object's path is either periodic (saws, crushers — learn the loop) or player-relative (Magpie, Fan — read the wind-up). No true erratic movers, ever: erratic + dense = slot machine, and we are not a slot machine.
- Audio layering: each category owns a frequency band (hazards low, value high, relief mid) so ears parse what eyes can't.

## 6.4 Where authored design still lives

The director spawns; it doesn't *compose*. Authorship survives in: the phase budgets, the Coffee Break placement, the guaranteed reliefs before spikes, the Last Words silence, and the placement *costs* (−10s tokens always guarded, Bus Stops always slightly out of the way). Systemic games die when designers mistake "procedural" for "unauthored" — the schedule is a level, written in probabilities.

---

# 7 · Mandatory Hazard Engagement — Evaluation (brief requirement 4)

**The question:** should the design force the player to encounter/clear every hazard type at least once per session, versus allowing full routing freedom?

**Pros of a hard mandate:** guarantees content exposure (nothing in the catalog is wasted work); teaches the full hazard vocabulary, which matters because the final crossing tests all of it; prevents degenerate single-route play; makes mastery claims meaningful ("I beat it" means *all* of it).

**Cons:** it fundamentally fights this game's identity. In an open arena, "engage" is hard to even define (what constitutes engaging a saw — proximity? survival? damage taken?); enforcement requires either invisible checklists (feels arbitrary when the door refuses you) or visible ones (HUD clutter that turns a survival spectacle into an errand list); it punishes creative routing, which is the genre's core joy; and it converts the room from a *place that happens to you* into a *syllabus*, deflating the fantasy.

**Recommendation: reject the hard mandate for this game — adopt guaranteed exposure instead.** Three layers deliver the mandate's benefits without its costs:

1. **Exposure by schedule:** the deterministic director guarantees every hazard type *spawns* in every session (already true in the prototype's phase tables). Players see everything; they choose what to touch.
2. **De-facto engagement at the Crossing:** the 5:00 traversal through a 100-object room forces meaningful contact with most types anyway — the mandate emerges from density, not decree.
3. **The Field Guide (optional meta):** a collection journal — "survive a near-miss with each hazard type" — rewarding voluntary engagement across sessions with cosmetics. The completionists get their checklist; the speedrunners never see it.

**The nuance worth keeping:** the answer flips in *authored-level* games. In Jump Once (fixed levels, one trap as thesis per level), mandatory engagement is correct — each level exists to teach its setpiece. Mandate engagement where levels are *sentences*; guarantee only exposure where the level is *weather*.

---

# 8 · Implementation Priorities

| Tier | Contents | Rationale |
|---|---|---|
| **P0 (already shipped)** ✅ | Room, clock, door, 6 hazard types, deterministic scheduler, telegraphs, spawn pocket, death/respawn, HUD, vignette, title/win screens, mobile controls | The playable spine |
| **P1 — the brief's core** | Anti-idle Monument ladder · Bus Stops · Coffee Breaks · Time Tokens (±10s, capped) · Bits + Piggy Bank + death-scatter · Magpie · Mystery Crate · solvability veto | Converts hazard-hose into ecosystem; every brief requirement lands here |
| **P2 — flavor & depth** | Reject power-up variants · The Fan, Inspector, Fixer · Buoy Platforms + Ghost Scaffold · Mercy Rule · 4 of the new hazards (Auditor, Metronome, Souvenir Crown, Echo) | The emotional spectrum fills out |
| **P3 — retention** | Daily Bin seed + leaderboard · Field Guide · Dev Coins/cosmetics · remaining catalog | The reasons to return |

*Rewarded-ad hooks (portal monetization, from earlier sessions): "respawn at your furthest point" offered on deep deaths after 3:00, and "+1 Complaint Form" offered after three deaths to the same hazard type. Interstitial only at session end. Never gate the door itself.*

---

*End of document. Companion prototypes: `one-second-longer.html` (playable spine of this design) and `jump-once.html` (the authored-level sibling; shares the visual grammar and the Honesty Split).*
