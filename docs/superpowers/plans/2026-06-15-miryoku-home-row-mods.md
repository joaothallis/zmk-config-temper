# Miryoku Home Row Mods Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move modifiers from the bottom row to the home row in Miryoku GACS order, with positional hold-tap tuning, so holding Command (and the others) is a resting-finger motion and Command+Tab can be done relaxed and cross-hand.

**Architecture:** A single-file change to `boards/shields/temper/temper.keymap`. Add three preprocessor position-group defines and two positional hold-tap behaviors (`hml`, `hmr`); rewrite the default layer's home row to use them and revert the bottom row to plain keys. The existing `hm` behavior stays as-is for the thumb keys. Everything else (other layers, combos, macros, conf, overlays) is untouched.

**Tech Stack:** ZMK firmware `v0.3` (devicetree keymap), GitHub Actions for build (`build.yml`) and keymap image (`doc.yml`).

**Verification model:** ZMK keymaps have **no local unit-test harness and no local build** here. The two real gates are:
1. **CI compile** — pushing the branch runs the `build firmware` workflow; it must go green (this catches devicetree syntax errors and unknown properties).
2. **Manual hardware validation** — flash the produced `.uf2` and run the checklist in Task 5. This step is performed by the user; the agent's responsibility ends at "CI green + draft PR open."

Per-task verification before the push is a careful static self-check (brace balance, binding count, spelling of keycodes/positions).

---

## File Structure

| File | Change | Responsibility |
|------|--------|----------------|
| `boards/shields/temper/temper.keymap` | Modify | All keymap logic: defines, behaviors, layers, combos, macros |
| `docs/superpowers/specs/2026-06-15-miryoku-home-row-mods-design.md` | (exists) | Approved design reference |

Position reference (used by the trigger groups):

```
 0   1   2   3   4  |  5   6   7   8   9
10  11  12  13  14  | 15  16  17  18  19
20  21  22  23  24  | 25  26  27  28  29
        30  31  32  | 33  34  35
```

Home row keys: `A`=10 `S`=11 `D`=12 `F`=13 `G`=14 | `H`=15 `J`=16 `K`=17 `L`=18 `;`=19

---

## Task 1: Create feature branch

**Files:** none (git only)

- [ ] **Step 1: Confirm clean-ish state and current branch**

Run:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper status --short --branch
```
Expected: branch shows `## main`. (Untracked `CLAUDE.md` and the new `docs/` files may be listed — that's fine; they won't be in our commits.)

- [ ] **Step 2: Create and switch to the branch**

Run:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper switch -c miryoku-home-row-mods
```
Expected: `Switched to a new branch 'miryoku-home-row-mods'`

---

## Task 2: Add position defines and `hml` / `hmr` behaviors

**Files:**
- Modify: `boards/shields/temper/temper.keymap` (defines block near top; `behaviors {}` block)

- [ ] **Step 1: Add the position-group defines**

Edit `boards/shields/temper/temper.keymap`.

Find this exact line:
```
#define ______ &trans
```

Replace it with:
```
#define ______ &trans

// Home row mod positional trigger groups
#define KEYS_L 0 1 2 3 4 10 11 12 13 14 20 21 22 23 24
#define KEYS_R 5 6 7 8 9 15 16 17 18 19 25 26 27 28 29
#define THUMBS 30 31 32 33 34 35
```

- [ ] **Step 2: Add the two positional hold-tap behaviors**

In the same file, find this exact block (the `hm` behavior followed by the closing brace of the `behaviors` node):
```
        hm: homerow_mods {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            tapping-term-ms = <200>;
            quick-tap-ms = <0>;
            flavor = "tap-preferred";
            bindings = <&kp>, <&kp>;
        };
    };
```

Replace it with (keeps `hm` untouched for the thumb keys, adds `hml` and `hmr` before the node closes):
```
        hm: homerow_mods {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            tapping-term-ms = <200>;
            quick-tap-ms = <0>;
            flavor = "tap-preferred";
            bindings = <&kp>, <&kp>;
        };

        // Positional home row mods (left hand): hold triggers only with a
        // right-hand or thumb key, so same-hand rolls stay plain letters.
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

        // Positional home row mods (right hand): mirror of hml.
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
    };
```

- [ ] **Step 3: Static self-check**

Run:
```
grep -nE "hml:|hmr:|KEYS_L|KEYS_R|THUMBS|require-prior-idle-ms|hold-trigger-key-positions" /Users/joao/Developer/joaothallis/zmk-config-temper/boards/shields/temper/temper.keymap
```
Expected: the two define lines (`KEYS_L`, `KEYS_R`, `THUMBS`), `hml:` and `hmr:` labels, and two occurrences each of `require-prior-idle-ms` and `hold-trigger-key-positions`.

Also confirm braces still balance:
```
awk '{o+=gsub(/{/,"{"); c+=gsub(/}/,"}")} END{print "open="o" close="c}' /Users/joao/Developer/joaothallis/zmk-config-temper/boards/shields/temper/temper.keymap
```
Expected: `open` equals `close`.

- [ ] **Step 4: Commit**

Run:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper add boards/shields/temper/temper.keymap
git -C /Users/joao/Developer/joaothallis/zmk-config-temper commit -m "feat: add positional home row mod behaviors (hml/hmr)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```
Expected: one file changed.

Note: this is a valid intermediate state — `hml`/`hmr` are defined but not yet used, which compiles fine.

---

## Task 3: Move modifiers to the home row in the default layer

**Files:**
- Modify: `boards/shields/temper/temper.keymap` (`default_layer` bindings — the home row and the bottom row)

- [ ] **Step 1: Rewrite the home row to use `hml` / `hmr`**

Find this exact line (the default layer's home row, all plain `&kp`):
```
        &kp A         &kp S         &kp D         &kp F         &kp G             &kp H         &kp J         &kp K         &kp L       &kp SEMI
```

Replace it with:
```
        &hml LGUI A  &hml LALT S  &hml LCTRL D  &hml LSHFT F    &kp G             &kp H    &hmr RSHFT J  &hmr RCTRL K  &hmr RALT L  &hmr RGUI SEMI
```

- [ ] **Step 2: Revert the bottom row to plain letters**

Find this exact line (the default layer's bottom row with `&hm` mods):
```
     &hm LSHFT Z   &hm LALT X   &hm LCTRL C   &hm LGUI V      &kp B             &kp N      &hm RGUI M  &hm RCTRL COMMA &hm RALT DOT  &hm RSHFT FSLH
```

Replace it with:
```
        &kp Z         &kp X         &kp C         &kp V         &kp B             &kp N         &kp M       &kp COMMA      &kp DOT      &kp FSLH
```

- [ ] **Step 3: Static self-check — default layer still has exactly 36 bindings**

Run:
```
awk '/default_layer/{f=1} f&&/bindings = </{b=1} b{print} b&&/>;/{exit}' /Users/joao/Developer/joaothallis/zmk-config-temper/boards/shields/temper/temper.keymap | grep -oE "&[a-z_]+" | wc -l
```
Expected: `36` (10 top + 10 home + 10 bottom + 3 left thumb + 3 right thumb).

Confirm the bottom row no longer references `&hm` and the home row references the new behaviors:
```
grep -nE "&hml |&hmr " /Users/joao/Developer/joaothallis/zmk-config-temper/boards/shields/temper/temper.keymap
```
Expected: 8 matches, all on the default layer's home row line.

- [ ] **Step 4: Commit**

Run:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper add boards/shields/temper/temper.keymap
git -C /Users/joao/Developer/joaothallis/zmk-config-temper commit -m "feat: move modifiers to home row (Miryoku GACS)

Home row mods A/S/D/F = GUI/Alt/Ctrl/Shift (mirrored on right);
bottom row reverts to plain letters. Reduces sustained-hold strain
for Command+Tab and other modifier chords.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```
Expected: one file changed.

---

## Task 4: Push, verify CI build, open draft PR

**Files:** none (git/CI only)

- [ ] **Step 1: Push the branch**

Run:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper push -u origin miryoku-home-row-mods
```
Expected: branch pushed; this triggers the `build firmware` and `draw keymap` workflows.

- [ ] **Step 2: Watch the firmware build**

Run:
```
gh run list --repo joaothallis/zmk-config-temper --workflow build.yml --branch miryoku-home-row-mods --limit 1
```
Take the run ID from the output, then:
```
gh run watch <run-id> --repo joaothallis/zmk-config-temper --exit-status
```
Expected: workflow concludes **success**. If it fails, open the logs (`gh run view <run-id> --log-failed`) — the most likely cause is a devicetree typo or, less likely, the ZMK `v0.3` pin not supporting a property (it should support both `require-prior-idle-ms` and `hold-trigger-key-positions`). Fix in the keymap, commit, push, re-watch.

- [ ] **Step 3: Pull the auto-generated keymap image**

The `draw keymap` workflow commits an updated image to the branch. Sync it locally:
```
git -C /Users/joao/Developer/joaothallis/zmk-config-temper pull --ff-only
```
Expected: fast-forwards to include the bot's `doc: update keymap image` commit (or "Already up to date" if it hasn't run yet — not blocking).

- [ ] **Step 4: Open the draft PR**

Run:
```
gh pr create --repo joaothallis/zmk-config-temper --draft \
  --title "Miryoku home row mods (reduce Command+Tab hand strain)" \
  --body "Moves modifiers from the bottom row to the home row in Miryoku GACS order with positional hold-tap tuning (hml/hmr, require-prior-idle-ms, balanced flavor, quick-tap, hold-trigger-on-release).

Motivation: left-hand pain from the sustained Command hold during Command+Tab. Command now rests on a home-row finger; the comfortable motion is hold \`;\` (right pinky) + tap left-thumb Tab.

Design: docs/superpowers/specs/2026-06-15-miryoku-home-row-mods-design.md

Verification: CI firmware build green. On-hardware validation pending (see plan Task 5).

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```
Expected: a draft PR URL is printed.

---

## Task 5: Manual on-hardware validation (performed by the user)

**Files:** none — this is hardware testing. The agent hands off here.

Download the `.uf2` artifacts from the successful `build firmware` run, flash each half (double-tap reset → copy `.uf2`), then check:

- [ ] **No accidental mods while typing** — type fast, hitting same-hand rolls: "as", "sad", "ask", "fads", "load", "kill", "jak". Expect plain letters — no Save dialog, no stray Ctrl, no capitalization.
- [ ] **Deliberate mods work** — pause, then hold a mod + an opposite-hand key: `A`+c → ⌘C, `F`+c → ⇧C, `;`+c → ⌘C, `J`+a → ⇧A.
- [ ] **Command+Tab (the goal)** — hold `;` (right pinky) + tap left-thumb Tab → switch; tap thumb repeatedly → cycle; release → commit. Confirm it feels low-strain on the left hand.
- [ ] **Combos intact** — F+G → `[`, S+D → one-shot Shift, J+K → `Esc`, L+`;` → `'`.
- [ ] **Key repeat** — tap-then-hold a home-row letter repeats it (e.g. hold `a` after a quick tap).

Tuning if needed (edit, commit, push, re-flash):
- Accidental mods appearing → raise `require-prior-idle-ms` (e.g. 150 → 200) and/or `tapping-term-ms`.
- Deliberate mods missed / sluggish → lower those values.

When validation passes, mark the PR ready for review / merge.

---

## Self-Review Notes

- **Spec coverage:** layout relocation (Task 3), `hml`/`hmr` behaviors with all four tuning guards + position groups (Task 2), thumb `hm` left unchanged (Task 2 keeps it verbatim), combos untouched (no task modifies them), validation plan (Task 5), cross-hand Cmd+Tab (Task 5 + PR body). All spec sections map to a task.
- **Placeholder scan:** none — every edit shows exact old/new text and every command shows expected output. `<run-id>` and `<filename>`-style tokens are runtime values, not placeholders.
- **Consistency:** behavior labels `hml`/`hmr`, define names `KEYS_L`/`KEYS_R`/`THUMBS`, and position numbers are identical across Task 2's behavior definitions and the position reference. Home-row binding line in Task 3 references exactly the behaviors defined in Task 2.
