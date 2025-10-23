# urob/zmk-config — Coding Style Guide

This is a concise guide describing the conventions used across this repo. It focuses on the non-obvious, repo-specific and newer/uncommon techniques (ZMK features, keymap-drawer conventions, Justfile automation, build / nix integration, and small SVG/keymap conventions). It intentionally omits generic best-practices.

---

## ZMK configuration conventions

- Prefer "timer-less" style Homerow Mods (HRMs)
  - Use a large tapping-term and the `balanced` flavor:
    - tapping-term-ms ≈ 280
    - flavor = "balanced"
    - quick-tap-ms ≈ 175
    - require-prior-idle-ms ≈ 150
  - Use positional hold-tap with `hold-trigger-on-release` to:
    - force same-side next-key taps to resolve as taps, and
    - delay the hold/tap decision until the next key RELEASE so multiple modifiers can be combined.
  - Example (devicetree macro style):
    - ZMK_HOLD_TAP(hml,
        flavor = "balanced";
        tapping-term-ms = <280>;
        quick-tap-ms = <175>;
        require-prior-idle-ms = <150>;
        bindings = <&kp>, <&kp>;
        hold-trigger-key-positions = <KEYS_R THUMBS>;
        hold-trigger-on-release;
      )

- Use positional and hand-aware rules
  - Define hand/position constants via #define (e.g. KEYS_L, KEYS_R, THUMBS) and reference them in hold-tap macros or positional checks.

- Avoid tap-dance for double-tap behaviors that must be instant
  - Use a mod-morph (or equivalent) for double-tap (capsword/double-shift) instead of a tap-dance to avoid tap delays.

- Combos instead of a symbol layer
  - Prefer combos (and homerow-mod-combos where needed) to surface symbols, leveraging `require-prior-idle-ms` to avoid most misfires.
  - When ZMK lacks tap-only combo support, implement combos as homerow-mod-combos to reduce edge cases.

- Smart/auto layers implemented as modules
  - Numword / Smart-Mouse: implement as an auto-layer module where:
    - a single TAP activates a sticky/auto layer,
    - continuous numeric typing keeps it active,
    - holding the activator produces a non-sticky layer.
  - Provide a dedicated CANCEL key on Nav layer to forcibly deactivate modes.

- Tri-state and swapper behaviors
  - Use Nick Conway’s tri-state behavior (PR/patch) for single-handed Alt-Tab style swapping (Swapper).
  - Document and pin any ZMK patches required to preserve higher-level behavior.

- Prefer helper macros and small modules
  - Use zmk-helpers (or local helper macros) to simplify devicetree syntax, key-label inclusion, and readable key bindings.

---

## Devicetree / .dtsi style

- Macro-first layout
  - Centralize position lists:
    - #define KEYS_L …, #define KEYS_R …, #define THUMBS …
  - Use short, mnemonic macro names: e.g. hml / hmr for left/right homerow-hold-taps.

- ZMK macro style and syntax
  - Use the ZMK_HOLD_TAP macro (or similar) with semicolon-separated properties inside parentheses.
  - Use angle-bracket integer syntax where ZMK expects numeric DT properties: tapping-term-ms = <280>;
  - Keep bindings as bare <&kp> tokens when using helper macros.

- Keep semantics explicit
  - Document why a non-default ZMK behavior is chosen (balanced vs hold-preferred/tap-preferred, hold-trigger-on-release, etc.) next to macro definitions.

---

## keymap-drawer / draw/config.yaml conventions

- Use a small preamble and additional includes
  - zmk_additional_includes for helper headers.
  - zmk_preamble to #include key-label headers (e.g. key-labels/34.h) and general compile defines (e.g. CONFIG_WIRELESS).

- raw_binding_map: map internal bindings to human-friendly labels
  - Use raw_binding_map to translate complicated ZMK bindings into concise labels used in the SVG/legend.
  - Support structured mappings where a binding has multiple textual forms:
    - e.g. "&num_dance": { "t": "Numword", "s": "Sticky Num" }

- Icon insertion
  - Use the $$mdi:icon$$ placeholder in zmk_keycode_map to insert SVG glyphs (these are picked up by the drawing pipeline and placed via <use> in the SVG).
    - Example: C_MUTE: $$mdi:volume-off$$

- Normalize keycode labels
  - Provide a comprehensive zmk_keycode_map that normalizes multiple synonyms to the same short label (e.g. LALT/RALT → Alt, BPST → Bspc).
  - Prefer concise labels (Gui, Alt, Shift, Ctrl, Bspc, Del, Spc, Tab, PgUp, PgDn).

- Transparent keys and combo layout hints
  - Use trans_legend to control how transparent keys are shown.
  - Control combo rendering via zmk_combos metadata (alignment, offsets, hidden).

- Drawing config
  - append_colon_to_layer_header: false (no trailing colon)
  - footer_text: keep repo/author string consistent.

---

## Justfile and build scripting conventions

- High-level structure
  - Use Justfile variables for repository-local absolute paths:
    - config := absolute_path('config')
    - build := absolute_path('.build')
    - out := absolute_path('firmware')
    - draw := absolute_path('draw')

- Shell recipes
  - Place a shebang and strict mode inside every recipe that runs a shell script:
    - #!/usr/bin/env bash
    - set -euo pipefail
  - Use here-doc style templating functions ({{ … }}) to resolve Justfile variables into the shell.

- Dynamic configuration from source artefacts
  - Dynamically compute build-time config values by parsing generated files (uncommon but central here):
    - Example: parse config/combos.dtsi to set CONFIG_ZMK_COMBO_MAX_COMBOS_PER_KEY and CONFIG_ZMK_COMBO_MAX_KEYS_PER_COMBO in config/*.conf via sed -Ei.
    - Rationale: avoid exhausting combo slots; let the source define limits.

- Target parsing & orchestration
  - Parse build.yaml with yq to produce (board, shield, snippet) tuples and feed them to a _build_single recipe.
  - Build artifacts named by combining shield and board; copy zmk.uf2 if present else fall back to zmk.bin.

- Testing harness
  - Provide a test recipe that:
    - builds against native_posix_64,
    - runs the emulator and captures keycode events,
    - filters events with a patterns file and diffs against a snapshot,
    - supports flags: --no-build, --verbose, --auto-accept.

- Keep UI terse for common tasks
  - Default just target lists (e.g. @just --list --unsorted).
  - Provide helper recipes: init (west init/update/export), draw (keymap generation pipeline), update, upgrade-sdk.

---

## Shell / scripting small conventions

- Use minimal but explicit bash idioms in scripts:
  - Prefer set -euo pipefail and explicit variable quoting when interpolating file names or paths.
  - Prefer builtin tools (yq, sed, grep, awk) for parsing YAML / DT source artifacts.

- Filenames and artifacts
  - Construct artifact directory names as: "${shield:+${shield// /+}-}${board}" (replace spaces with + in shields).

---

## Nix / Repo / CI conventions

- Pin the environment
  - Keep the build environment pinned with flake.lock and test the pinned environment in CI.
  - Use direnv + nix-direnv to provide an isolated dev environment that activates on cd.

- .gitattributes
  - Force syntax highlighting for certain non-standard files:
    - *.dtsi text linguist-language=C++
    - *.keymap text linguist-language=C++

---

## SVG / Keymap-export conventions

- Centralized glyph defs
  - Place icon glyphs in <defs> and reference them with <use> and xlink:href for small inline icons.
  - Use classes (e.g. key, held, trans, combo, combo-separate) to control fill / stroke via a single stylesheet block.

- Class-based semantics
  - Annotate keys with classes to show semantics:
    - text.tap, text.hold, text.shifted, text.trans
    - rect.key, rect.combo, rect.held, rect.trans
  - Use layer group names and ids that match layer names (e.g., class="layer-Base" and an element with id="Base") — this supports clickable SVG layer navigation.

- Consistent typography
  - Use a monospace face and consistent default sizes; shifted / hold labels use smaller font-sizes and vertical offsets.

---

## Naming conventions

- Keycodes and combos
  - Keycode labels are uppercase with underscores for internal names; map to short human labels via zmk_keycode_map.
  - Combo names use a clear prefix: combo_*, hm_combo_*, mmv_* for mouse move, msc_* for mouse scroll, mkp_* for mouse buttons.

- Macros and behaviors
  - Short macro names for left/right mirrored constructs: hml / hmr (homerow left/right), mmv for mouse-move, msc for scroll.

---

## Documentation + Rationale

- Always include a short rationale for any non-default or timing-sensitive setting (e.g., why tapping-term is large; why require-prior-idle-ms is tuned). The README contains domain knowledge that is considered part of the "code" for this project and should be kept aligned with actual parameter values.

---

If you want, I can:
- Extract the actual templated values (e.g. all ZMK_HOLD_TAP macros) from the repo and produce a single annotated snippet showing each uncommon option, or
- Produce a checklist for adding a new key/hold-tap/combo that follows these conventions.