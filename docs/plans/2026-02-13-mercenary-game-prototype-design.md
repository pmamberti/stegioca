# Stegioca: Mercenary Mission Game Prototype

## Overview

A browser-based game prototype where the player selects 2 mercenaries from a roster of 6, sends them on a mission, and sees the outcome. First iteration focuses on the core selection/mission loop with procedurally generated 1-bit pixel art characters.

## Tech Stack

- **Single `index.html` file** with inline CSS and JS
- **Canvas 2D** for rendering procedural pixel art portraits
- **No dependencies**, no build step — open in browser and play
- Rendering layer is intentionally decoupled from generation logic so it can be swapped to WebGL/WebGPU when the full game is built

## Architecture

Three logical layers, all in one file:

1. **Generation layer** — pure functions that procedurally create mercenary data and pixel art portraits. Takes a seed, returns a character object. Decoupled from rendering for future portability.
2. **Rendering layer** — Canvas 2D. Draws pixel grids onto `<canvas>` elements scaled up with `image-rendering: pixelated` for crisp 1-bit look. Handles UI layout.
3. **Game logic layer** — manages state: current screen, selected cards, mission outcome.

## Game Flow

### Screen 1: Selection

- 6 cards in a 3x2 grid
- Click a card to select (highlighted), click again to deselect
- Counter shows "0/2 selected" -> "1/2 selected" -> "2/2 selected"
- "Start Mission" button, disabled until exactly 2 cards are selected

### Screen 2: Waiting

- Two selected cards displayed side by side
- "Mission in progress..." message with animated ellipsis or pulse effect
- 5-second timer, then auto-transitions to result

### Screen 3: Result

- Two selected cards still visible
- Brief randomly-picked narrative sentence (names slotted in)
- "Mission Success" or "Mission Failed" heading (50/50 dice roll)
- "New Mission" button returns to selection with fresh cards (new seeds)

## Procedural Character Generation

### Races

| Race | Silhouette | Flavour |
|------|-----------|---------|
| Human | Standard proportions, round head | The baseline, everywhere in the galaxy |
| Tharn | Broad, heavy, cranial ridges | Engineered heavy-worlders, built for war |
| Cephalid | Bulbous head, face tentacles | Psychic deep-space dwellers |
| Gravborn | Short, wide, visor/respirator | High-gravity miners, dense and tough |
| Insectoid | Antennae, segmented body | Hive-species, expendable and fearless |
| Construct | Angular, boxy, mechanical | Autonomous war machines |

### Generation Algorithm

1. Each race defines a **base template** — a rough pixel grid outline at 32x32 that establishes the silhouette (head shape, body proportions, distinguishing features)
2. A **seed** (integer) drives per-character randomness — slight variations in features (ridge size, tentacle length, visor shape) so no two characters of the same race look identical
3. **Symmetry** — left half is generated, then mirrored to right for classic pixel-art character feel
4. **Name generation** — syllable combiner per race with race-appropriate phonemes (Tharn get guttural sounds, Cephalid get wet/flowing ones, etc.)

### Character Data Structure

```
{
  name: "Devendra Ray",
  race: "Human",
  portrait: Uint8Array (32x32, 1 = black, 0 = white),
  seed: 48291
}
```

### Card Rendering

- Portrait canvas at top (32x32 scaled up with nearest-neighbor)
- Name below the portrait
- Race label as a tag
- All in 1-bit black and white

## Result Narratives

A pool of template sentences with `{name1}` and `{name2}` slots. 50/50 dice roll selects success or failure, then a random template from that pool.

### Success examples

- "{name1} drew their fire while {name2} secured the objective. Clean extraction."
- "{name2} almost blew the cover, but {name1} improvised. Mission complete."
- "Textbook operation. {name1} and {name2} made it look easy."

### Failure examples

- "{name1} took a hit in the first corridor. {name2} dragged them out. No payload."
- "Bad intel. {name1} and {name2} walked into a trap. Barely made it back."
- "{name2} froze under fire. {name1} called the abort. Live to fight another day."

~10 of each in the final implementation.

## Future Considerations (Not in Prototype)

- Stats per character (STR/INT/CHA) that influence mission success probability
- WebGL/WebGPU rendering with shader effects (CRT scanlines, dithering)
- Multiple mission types with different difficulty/requirements
- Persistent roster across missions
- Credits/economy system
