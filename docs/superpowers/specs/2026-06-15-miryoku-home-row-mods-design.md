# Miryoku-style Home Row Mods — Design

**Date:** 2026-06-15
**Status:** Approved design, pending spec review
**Scope:** `boards/shields/temper/temper.keymap` only

## Problem

Left-hand pain, attributed to Command+Tab. On the current layout the only `TAB`
lives on the left thumb (`&hm LGUI TAB`), so `TAB` is always a left-thumb tap.
Command therefore has to come from a *different* key — and in practice it comes
from `&hm LGUI V` (left **index**, bottom row). Doing Command+Tab means curling
the left index down to `V` and **holding that claw** while the left thumb taps
`TAB` repeatedly to cycle apps.

The strain source is the **sustained hold in an awkward bottom-row position**,
not the taps.

## Goal

Make holding a modifier (Command especially) a low-effort, resting-finger motion,
and enable a relaxed cross-hand Command+Tab. Do this by moving modifiers to the
home row in the Miryoku GACS arrangement, with tuning robust enough that the mods
stay invisible during normal typing.

## Approach (chosen)

Miryoku-style **home row mods** in **GACS** order (pinky→index: GUI, Alt, Ctrl,
Shift), with **full hold-tap tuning** (positional hold-tap + `require-prior-idle-ms`
+ balanced flavor + quick-tap).

Alternatives considered and rejected: the "swapper" tri-state behavior (eliminates
the hold entirely but adds an external module and a new motion — user preferred a
native Miryoku layout); sticky Command (only half-solves, no good cycling);
two-handed rebalance (keeps the hold).

> The swapper remains a viable future addition if a resting-pinky Command hold for
> frequent app-switching still bothers the hand later.

## Detailed Design

### 1. Default layer — modifier relocation

**Home row** gains GACS home row mods. **Bottom row** reverts to plain letters.
Top row and thumbs are **unchanged**.

```
                    Left                      |                 Right
top (unchanged):  Q    W    E    R    T       |   Y    U    I    O    P
home (NEW):       A⌘   S⌥   D⌃   F⇧   G       |   H   J⇧   K⌃   L⌥   ;⌘
bottom (PLAIN):   Z    X    C    V    B       |   N    M    ,    .    /
thumb (unchanged):     ⌥   NUM  ⌘/TAB         |   SYM/SPC  ⌃/BSPC  ⇧/ENTER
```

New home-row bindings:

```
&hml LGUI A  &hml LALT S  &hml LCTRL D  &hml LSHFT F  &kp G   \
&kp H  &hmr RSHFT J  &hmr RCTRL K  &hmr RALT L  &hmr RGUI SEMI
```

New bottom-row bindings (mods removed):

```
&kp Z  &kp X  &kp C  &kp V  &kp B   \
&kp N  &kp M  &kp COMMA  &kp DOT  &kp FSLH
```

- `G` (pos 14) and `H` (pos 15) stay plain — inner index columns have no mod, per
  Miryoku.
- Left mods use `L*` keycodes, right mods use `R*` keycodes.

### 2. Behaviors

Add two positional hold-tap behaviors for the home row. **Keep the existing `hm`
behavior unchanged** for the thumb keys (`&hm LGUI TAB`, `&hm LCTRL BSPC`,
`&hm RSHFT ENTER`) — thumbs are not typed during normal text, so they don't need
positional restriction, and leaving them preserves their current feel.

Position groups (added as `#define`s near the layer defines):

```
#define KEYS_L 0 1 2 3 4 10 11 12 13 14 20 21 22 23 24
#define KEYS_R 5 6 7 8 9 15 16 17 18 19 25 26 27 28 29
#define THUMBS 30 31 32 33 34 35
```

```
hml: home_row_mod_left {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    flavor = "balanced";
    tapping-term-ms = <200>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <150>;
    hold-trigger-key-positions = <KEYS_R THUMBS>;
    hold-trigger-on-release;
    bindings = <&kp>, <&kp>;
};

hmr: home_row_mod_right {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    flavor = "balanced";
    tapping-term-ms = <200>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <150>;
    hold-trigger-key-positions = <KEYS_L THUMBS>;
    hold-trigger-on-release;
    bindings = <&kp>, <&kp>;
};
```

Why each setting:

- **`hold-trigger-key-positions`** — a home row mod only resolves to *hold* when the
  next key is on the opposite hand or a thumb. Same-hand rolls ("as", "df", "we")
  stay pure letters. Left mods trigger on right + thumbs; right mods on left + thumbs.
  Thumbs are included in both so Command+Tab works from either Command (cross-hand
  `;`+thumb, or same-hand `A`/thumb-Command + thumb-Tab).
- **`require-prior-idle-ms = 150`** — if another key was pressed within the last
  150 ms, the home-row key is forced to tap. Fast typing never produces mods; a mod
  engages only after a deliberate pause + hold.
- **`flavor = "balanced"`** — clean resolution for intentional chords.
- **`quick-tap-ms = 175`** — tap-then-hold repeats the letter (key repeat survives).
- **`hold-trigger-on-release`** — lets multiple home row mods combine (e.g. ⌘⇧key).

Values are sensible starting points and are trivial to retune after living with them.

### 3. Cmd+Tab after the change

- Comfortable / recommended: hold **`;` (right pinky → Command)**, tap **left-thumb
  `TAB`**. Cross-hand; the left hand only does a light thumb tap. Tap the thumb again
  to cycle.
- Still available: the left thumb keeps its hold-Command / tap-Tab role for other
  GUI shortcuts.

### 4. Combos — unchanged

All combos are kept as-is. They remain compatible because the combo window (50 ms)
is far shorter than the hold timing. Worth a sanity-test after flashing because they
overlap home-row-mod keys:

- `F`+`G` (pos 13,14) → `[`
- `S`+`D` (pos 11,12) → one-shot Right Shift (now somewhat redundant with `F`=Shift,
  but harmless)
- `J`+`K` (pos 16,17) → `Esc`
- `L`+`;` (pos 18,19) → `'`

## Out of Scope / Not Changing

- `config/temper.conf`, overlays, `west.yml` — no changes.
- Other layers (NUM, SYM, FUNC) — no changes.
- Thumb keys and the `hm` behavior they use — unchanged.
- Combos and macros — unchanged.

## Validation Plan

After flashing both halves:

1. **No accidental mods while typing** — type a fast paragraph; specifically hit
   same-hand rolls: "as", "sad", "ask", "fads", "load", "kill", "jak". Expect plain
   letters, no Save dialogs / stray Ctrl / caps.
2. **Deliberate mods work** — pause, then hold each mod + an opposite-hand key:
   `A`+`c`→⌘C, `F`+`c`→⇧C, `;`+`c`→⌘C, etc.
3. **Cmd+Tab** — hold `;` + tap left-thumb `TAB` → switch; tap thumb repeatedly →
   cycle; release → commit.
4. **Combos** — F+G→`[`, S+D→shift, J+K→`Esc`, L+;→`'`.
5. **Key repeat** — tap-then-hold a home-row letter repeats it (quick-tap).

If accidental mods appear: raise `require-prior-idle-ms` / `tapping-term-ms`.
If deliberate mods feel sluggish or get missed: lower them.

## Risks

- **Muscle-memory transition** — modifiers move from bottom row to home row; a few
  days of adjustment expected.
- **Combo overlap on F+G and S+D** — covered by the validation plan; the 50 ms combo
  window makes conflicts unlikely.
