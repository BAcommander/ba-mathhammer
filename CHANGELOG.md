# BA Mathhammer - Changelog

A WH40K damage calculator. Two calculation engines are kept in agreement: an
**averaged/analytic** engine (the main results) and a **Monte-Carlo** engine
(used for the battlegroup/BGV analysis view).

---

## v1.02 - Feel No Pain is now rolled per wound, as written (2026-08-17)

### Player-facing patch notes (copy for Discord/YouTube)
- 🛡️ **Feel No Pain now works the way the rulebook says it does.** The rule is "each time a model
  would lose a wound, roll one D6" - one roll for every single point of damage. The calculator was
  rolling it once per attack instead, so a Damage 3 hit was either fully shrugged or did all 3.
  Now that same hit rolls three separate FNP saves and can cost the model 0, 1, 2 or 3 wounds.
  - This changes results, sometimes by a lot, and it goes both ways. Against 3-wound models with
    a 5+++: a Damage 2 weapon was under-reporting by about 17%, a Damage 6 weapon by about 27%,
    and a Damage 3 weapon was over-reporting by about 18%. The bigger the weapon's damage compared
    to the target's wounds, the more the old numbers were wrong.
  - The reason it mattered so much is overkill. Rolling FNP once per attack made damage
    all-or-nothing, so wounds got wasted in chunks. Rolling per point spreads it out - a shrugged
    point is not a wasted attack any more, it is just one wound the target kept.
  - Both views were fixed together, and the dice-sim view now matches an exact calculation of the
    rule to 3 decimal places. If you do not use FNP anywhere, nothing in your results changes.
  - The Save Phase "Wounds After FNP" line has been replaced with **"Damage After FNP"**, with a
    "?" explaining it. Counting wounds there stopped being useful once FNP is rolled per point -
    a Damage 3 hit vs a 5+++ target is only shrugged outright 1 time in 27, so the old line barely
    moved and read like a damage figure. The new line shows the damage that actually survives the
    FNP rolls, so you can follow unsaved wounds - damage after FNP - effective damage in order.

### Fixed
- **FNP is now RAW 24.12 - one roll per wound LOST, applied after the damage modifiers.** Both
  engines rewritten; see CLAUDE.md for the full rules note.
  - **Analytic** (`calcProfile`): new `applyFnpToPmf(pmf, fnp, cap)` thins the post-modifier
    damage pmf - damage `d` becomes `Binomial(d, q)` where `q` is the chance a wound is not
    shrugged. That is exactly equivalent to rolling until the model dies, because the rolls RAW
    never makes would only have landed on wasted overkill. `instances` is now the count of
    **unsaved wounds** (it used to be the post-FNP count) and the pmf carries 0-damage outcomes,
    so `expectedInstancesToKill` gained a `/(1 - p0)` term for instances that never advance the
    wound track. `applyDamageMods`' `expectedRolls` (the Re-roll N Damage budget) also moved to
    the pre-FNP wound count, which is when damage is actually rolled.
  - Values at or above `cap` are lumped into one bucket: exact for the allocation DP (any instance
    that meets the model's remaining wounds is identical to it) and it bounds the expansion cost
    for hostile dice strings. The **mean is taken pre-lump** as `mean × q`, so `totalDamage` and
    the per-weapon breakdown figure stay exact.
  - **MC** (`runMonteCarlo`): new `applyFnpDamage(dmg, remaining, fnp)` rolls one D6 per point and
    stops at the model's last wound. Shared by the unit loop and `applyEventToChar` so the unit,
    spill and Precision paths cannot drift. FNP moved to after the damage roll in both.
  - `charPipeline` no longer returns `fnpFail` - the character's FNP lives inside its pmf and mean
    now, so the three call sites (Precision absorb, spill damage, spill-to-kill) dropped it. The
    unit-wipe threshold likewise divides by the failed-save rate only, since the pooled pmf
    already accounts for shrugged instances (`instancesToWipe`, renamed from `finalWoundsToWipe`).
  - Accuracy, vs an exact DP over the RAW rule: the MC engine now matches to 3dp in every case
    tested. The analytic engine's error on effective damage drops from **-27%…+18%** to **under
    3%** (the residual is the pre-existing `expectedInstancesToKill` accumulation approximation,
    already documented). Regression: with FNP off, 42 damage scenarios and 96 character /
    Precision / sergeant-extra-wound scenarios are **byte-identical** to v1.01.
- **`FNP_TIP`** added to the shared results-tooltip consts and wired to the Save Phase "Wounds
  After FNP" row, which now counts unsaved wounds that removed at least one wound.

---

## v1.01 - Army roster import, Precision, Extra Attacks, Re-roll Attacks (2026-08-11)

### Player-facing patch notes (copy for Discord/YouTube)
- 📥 **Import your army from New Recruit** - no more typing every weapon in by hand. Export your
  list as JSON from New Recruit (or BattleScribe), hit Import in the header, and pick what you
  want: weapons go into your weapon library so any card's Load button can pull them, and units
  become defender presets with Toughness, Save, Invuln, Wounds, model count and points already
  filled in. A 2000pt list came in as 84 weapons and 53 units in one go.
  - Weapon keywords come across ticked - Lethal Hits, Devastating Wounds, Sustained Hits
    (including "Sustained Hits D3"), Torrent, Blast, Psychic, Ignores Cover, Twin-linked, Heavy,
    Melta, Rapid Fire, Anti-X and Lance (which is the +1 to Wound box).
  - The situational ones are ticked as printed on the datasheet, so untick what does not apply:
    Melta and Rapid Fire assume half range, Heavy assumes you moved 3" or less, Lance assumes a
    charge, and Anti-X assumes the target actually has the keyword.
  - Precision is the exception: it comes in ticked but set to "Aim: None", because who you snipe
    is a choice you make per shot, not something the datasheet decides. Importing therefore never
    changes a result on its own - pick a character on the card when you want to use it.
  - Anything already in your library is flagged "already saved" and left unticked, so re-importing
    an updated list will not fill you with duplicates.
- 🎯 **New weapon ability: Precision** - wounds from that weapon go onto an attached Character
  straight away, instead of having to wipe the bodyguard unit first. Until now there was no way
  to touch a character while its unit was alive, so sniping a leader could not be worked out at
  all. Tick Precision to record that the weapon has it, then use the dropdown to say who you are
  aiming at THIS shot: "Aim: None" shoots the unit as normal, or pick Char 1 / Char 2. If your
  target dies mid-volley the rest of the wounds go back onto the unit - by the rule, the
  Precision pick ends the moment that character is destroyed, it does not hop to a second
  character. With no character attached it does nothing either way.
  - The dice-sim view is the accurate one here. The averaged view is close away from the
    character's kill threshold, but right at it the averages tip over and call the character dead
    a bit early - if a result looks borderline, check it in the sim view.
  - The dice-sim view also reads properly when you are sniping: character rows now say "Chance
    of Killing Character" instead of "Wiping Unit + Character" (with Precision the character can
    die while the unit stands), each character shows its average damage taken, and the risk
    badge counts a successful snipe - no more "Minimal Damage" on a 98% character kill.
- 🗡️ **New weapon ability: Extra Attacks** - for weapons like a Servo-arm that are swung IN
  ADDITION to a normal melee weapon. Give it its own card as usual, and tick this so a unit-wide
  "+X Attacks" from your battlegroup is not applied to it - an Extra Attacks weapon's attacks
  cannot be modified by other rules. Imports ticked automatically.
- 🎲 **New weapon ability: Re-roll Attacks** - for weapons with a random Attacks characteristic
  (D6, D6+3, 2D3 and so on), tick this when a rule lets you re-roll the dice that decide how
  many attacks you get. Handy for Knights in particular. It sits with the other Damage & Misc
  abilities on each weapon card. It works the same way as Re-roll Damage: a below-average roll
  gets re-rolled and you keep the new number (you cannot keep the original). Ticking it on a
  weapon with a fixed Attacks value does nothing, so it is safe to leave on.
- 🔢 **"Total Attacks" now shows the fraction** - a D6-attack weapon reads 3.5 instead of being
  rounded to 4. The maths was always using the true average, only the displayed number was
  rounded, so your damage numbers do not change. Whole numbers still show without a decimal.
- ❓ **Points Efficiency now explains itself** - hover the new "?" next to Trade Ratio, Points
  Destroyed and Points Remaining. Short version: Trade Ratio is the defender points you removed
  divided by what the attacking unit costs, so 1.00 is an even trade, above 1.00 you came out
  ahead, and below 1.00 the attacker costs more than it killed. It covers ONE round of attacks,
  so a unit you expect to shoot for several turns will show a low ratio per round and still
  trade fine over a game. The "?" bubbles are stripped from clipboard image exports.

### Added
- **`ABILITY_CONFIG` value inputs can now be a fixed choice**: `val.options: [[value, label], ...]`
  renders a `<select>` instead of a text box. Everything downstream (collect, apply, save/restore,
  battlegroup grey-out) already went through `.value`, so no other code changed.
- **`precision` + `precisionTarget` per-profile ability** (`ABILITY_CONFIG`, `dmg` group).
  `precisionTarget` is a dropdown: 0 = None (ticked but not aimed - identical to unticked, verified
  byte-for-byte), 1/2 = that attached character. Splitting "has Precision" from "aiming it this
  shot" means the roster import can record the keyword without silently changing any result.
  MC: `precisionCharIndex` picks the target and `applyEventToChar` (extracted from the spill
  branch, so the two paths cannot drift) resolves it. Analytic: precision profiles are held out
  of the unit's `dmgPool`, `charPipeline` derives their damage under the character's own
  defences, and an `overflowFrac` of their instances is added back to the unit for the wounds the
  character could not absorb. Ignored entirely when no character is attached (verified: unit
  damage identical ticked vs unticked). Inherits the documented deterministic-mean gap near the
  character's kill threshold - see CLAUDE.md.
- **`extraAttacks` per-profile ability** (`ABILITY_CONFIG`, `dmg` group, id `extra-attacks`).
  Both engines skip `bgPlusAttacks` for a profile carrying it (`calcProfile` and `simProfile`);
  verified identical - 5 models, A2, battlegroup +2 Attacks gives 20 attacks off / 10 on in both.
  Per-weapon abilities printed on the weapon itself (Blast, Cleave, Rapid Fire) are deliberately
  NOT blocked: the RAW carve-out is about *other* rules modifying it.
- **`rerollAttacks` per-profile ability** (`ABILITY_CONFIG`, `dmg` group, id `reroll-attacks`).
  Config-driven, so the checkbox, tooltip, ability summary, weapon-library persistence and
  save/restore come for free.
- **New Recruit / BattleScribe roster import.** `importData` now sniffs the file: a roster
  (`obj.roster.forces`) routes to `parseNrRoster` + the picker modal instead of being rejected
  with "not a BA Mathhammer export file" (which is what players hit first). `parseNrRoster` walks
  the `selections` tree per top-level unit, collecting `profiles` by `typeName` ("Ranged/Melee
  Weapons", "Unit"); weapons dedupe case-insensitively by name+melee (New Recruit's own data has
  `Absolvor Bolt Pistol` / `Absolvor bolt pistol`), units take the most numerous stat line as
  representative and sum `number` for the model count. `nrWeaponToData` / `nrUnitToData` emit the
  exact shapes `captureWeaponProfile` / `captureDefenderData` produce, so the entries are
  indistinguishable from hand-made presets. `NR_MAX_NODES` caps the tree walk (hostile file), and
  every name/stat is `esc()`d into the picker.
- **`tipHtml(title, body)`** results-row tooltip helper + shared `TRADE_RATIO_TIP` /
  `PTS_DESTROYED_TIP` / `PTS_REMAINING_TIP` copy, used by the Points Efficiency rows in BOTH the
  averaged view and the dice-sim view. Emits the existing `.ab-tip`/`.tip-text` markup, so the
  delegated hover handler positions it with no extra wiring. Body text is `esc()`d, `\n` -> `<br>`.
  `copyResultsToClipboard` strips `.ab-tip` from its clone so the bubbles stay out of exports.

### Fixed
- **Precision overflow now matches the rule ([PRECISION] 24.28).** The sim originally cascaded
  leftover Precision wounds into the second attached character before the unit. Per RAW the
  chosen CHARACTER group is the current allocation group only "until that CHARACTER group is
  destroyed" - then the override ends and normal allocation resumes, so the overflow goes back
  onto the unit. `precisionCharIndex` now returns -1 once the target is dead (the event falls
  through to the normal unit path). The analytic engine already sent its `overflowFrac` to the
  unit's pool, so this also removes a documented engine divergence rather than adding one.
- **In-app Patch Notes swallowed sub-bullets.** `renderPatchNotes` treated an indented `  - `
  line as a wrapped continuation and glued it onto its parent, so a bullet with sub-points came
  out as one run-on paragraph with stray hyphens (this release's import bullet was 1482 chars;
  v1.0's accuracy bullet had the same problem). Nested bullets now render as a nested `<ul>`,
  styled smaller and dimmer so the headline bullet still leads. The block is also split on
  `\r?\n` now: with CRLF line endings a trailing `\r` defeats a `$` anchor, because JS `.` does
  not match a carriage return.
- **Total Attacks display** used `toFixed(0)`, so a fractional average was rounded (D6 read "4",
  and Re-roll Attacks on 2D3 still read "4" while the hits below it correctly rose). Now prints
  one decimal when the value is fractional. Display-only - no engine change.

### Engine notes
- **Analytic (`calculateBattle`)**: `parseDamage(atkStr)` already returned a RAW re-roll
  expectation (`reroll`), so the profile's `attacks` now takes `reroll` instead of `avg` when
  the box is ticked. Attacks enter the maths linearly (`totalAtk = models * modAttacks`), so
  this substitution is exact rather than an approximation.
- **Monte-Carlo (`simProfile`)**: the per-model attacks roll re-rolls once when it comes in
  below `profile.attacks` (the parsed average) - the same threshold the analytic side uses.
- **Cross-validated** over 2M iterations per case (1, 3, D3, D6, D6+3, 2D6, 2D3+1, D6+2, 3D6):
  the two engines agree to within 0.03%, and fixed Attacks values are unchanged.

---

## v1.0 - Datasheet-style inputs, save confirmations, accuracy + hardening pass (2026-08-06)

### Player-facing patch notes (copy for Discord/YouTube)
- 🎲 **Type it like the datasheet** - stat boxes now accept the notation printed on your cards.
  BS/WS, Sv, Invuln, FNP, Anti-X and Crit boxes take "3+" as well as "3"; AP takes "-1" as
  well as "1"; dice values work with spaces ("D6 + 3"); Sustained Hits accepts "D3+1". Before
  this, several of those silently fell back to a default - a Demolisher with "D6 + 3" attacks
  was being calculated as ONE attack, and an AP typed as "-2" actually IMPROVED the target's
  save. If a result ever looked weirdly low, this was probably why.
- 💾 **Saving feels safe now** - saving under an existing name offers to update that record
  instead of blocking; every save/update pops a small confirmation toast; and if your browser
  storage is full you get told, instead of a fake success. Weapon card Save also remembers
  its library link after a refresh (Upd/Del no longer vanish).
- 🎯 **Averaged engine accuracy** (the dice-sim view was always the exact reference, and the
  averaged view now tracks it much closer):
  - Attached-character damage from big multi-damage weapons no longer massively overshoots -
    the maths now models overkill waste when deciding the unit is wiped (a D6-damage volley
    used to "kill" a bodyguard the dice sim said took ~30% damage)
  - Re-roll 1s + Re-roll N together now stack like the dice do (was up to -21% wounds)
  - Re-roll N Wounds is now correct alongside Lethal Hits / Sustained Hits
- 📦 **Import/export hardening** - imported files are validated (a corrupted or malicious
  file can no longer wedge the app on every page load), and your weapon-card links survive
  restoring a backup on a fresh machine.
- 🧰 Smaller fixes: clearing a character's Invuln box now means "no invuln" (was silently
  4++), clearing a Models box no longer silently drops that weapon, first visit starts with
  Battlegroups ON to match the Clear buttons, and the external CDN scripts are pinned with
  integrity hashes.
- 🛒 **BA Commander Store** - new Merch button in the header (and slide-out menu) linking to
  the official merch shop: https://ba-commander-shop.fourthwall.com/

### Added
- **Datasheet notation parsing**: roll-target inputs (profile BS/WS; defender and character
  Sv/Invuln/FNP; ability values like Anti-X, Crit on X+, FNP mod) are now `type="text"`
  `inputmode="numeric"` so "3+" survives to the parser (`parseInt` strips the trailing `+`;
  `collectProfileRules` still clamps configured min/max). AP fields accept negatives - both
  engines read `Math.abs(...)`. `parseSustained`/`rollSustainedValue` accept `NdX+M`.
- **`resolveSaveName`** shared save-name flow: clash -> "already exists. Update it?" dialog
  (Update overwrites in place, keeping id + stored name; decline reopens the name modal).
- **`showSaveToast`** (`#save-toast`, textContent-only, `aria-live=polite`) fired from all
  save/update paths; `putPresets` now returns success and `storageError()` reports failures.
- **SRI**: integrity + crossorigin on the two Firebase script tags and the dynamically loaded
  html2canvas (`loadScript(src, integrity)`).

### Fixed
- **Dice parsers** tolerate internal whitespace ("D6 + 3"); sanity cap `num<=50, sides<=1000`
  in BOTH engines (a hostile "9999D6" import used to hang the analytic convolution).
- **Analytic character spill**: unit-wipe threshold and per-character chaining now use the
  `expectedInstancesToKill` DP on the pooled (per-character re-modified) damage pmf instead of
  totalHP / mean damage, so per-model overkill waste is respected. Removes the large
  multi-damage overstatement (a reported-dead character vs ~29% in the sim); remaining bias is
  the documented deterministic-mean Jensen underestimate near the threshold.
- **Analytic re-roll branches**: new combined Re-roll 1s + Re-roll N branch for hits and
  wounds (free re-roll on 1s, budget spent on the other failures - mirrors the sim); Re-roll N
  Wounds trials pool corrected to the actual wound dice (normal hits + sustained extras;
  lethal auto-wounds never roll), fixing +7%/-4% drift with Lethal/Sustained.
- **Import**: `state` blob validated (plain object, profiles capped at 20 and type-checked -
  a 500k-profile file used to re-hang the app on EVERY load until site data was cleared);
  preset names coerced to strings (a non-string name aborted the whole import silently);
  category lists capped at 500; `state.profiles[].weaponPresetId` remapped to the freshly
  assigned weapon ids so card->library ties survive a backup restore.
- **Persistence**: weapon Save/Delete now `saveState()` so the card's library tie survives
  refresh; unticked Sustained value text survives save/load round-trips (captures keep the
  raw text; restore skips legacy `''`); first visit keeps both Battlegroup toggles ON;
  after a clash-update the name field syncs to the stored preset name; character Invuln
  empty-box fallback fixed to 0 (was 4); profile Models empty-box fallback fixed to 1 (was
  0, which silently dropped the weapon); defender models clamped to 500 (display-loop DoS).

---

## v0.6 - Profile toggles, crit-fishing, points efficiency, leader + support (2026-07-28)

### Player-facing patch notes (copy for Discord/YouTube)
- ⚡ **Weapon profile On/Off switches** - every weapon card now has an "On" tick in its
  header. Untick it to keep the weapon loaded but drop it from the maths - so you can have
  your whole JDC kit plus Lemartes/Astorath weapons loaded at once and flick them on and off
  to compare leaders and buffs, no reloading.
- 🎣 **Fishing for crits** - "Re-roll Non-6s" is now "Re-roll Non-Crit Hits" (it respects
  Crit on X+) and works with Lethal Hits as well as Sustained Hits. New "Re-roll Non-Crit
  Wounds" ability - re-roll every wound roll that isn't a crit (even successes) to fish for
  Devastating Wounds; works with Anti-X thresholds too. Both model committing to the fishing
  plan - whether it's actually worth it for your matchup is exactly what the calculator will
  show you.
- 💰 **Points efficiency** - optional Points fields on both sides (attacker loadout, defender
  unit, each attached character); leave at 0 if you don't care. Fill them in and the results
  show points destroyed (part-dead models count), a trade ratio (points destroyed ÷ attacker
  cost - above 1.00 is a good trade) and points remaining. The probability view shows the
  average points destroyed and trade ratio across 1,000 simulated battles. Cheap scouts into
  Reapers finally looks like the bargain it is - and an expensive hammer unit into grots
  looks like the waste it is.
- 👥 **Leader + Support** - the defender can now take two attached characters, matching this
  edition's leader + support attachments. Spill damage hits Character 1 first, then
  Character 2, each rolling their own saves/FNP/damage reduction - and both get their own bar
  in the probability view.
- 📜 **Patch notes in the app** - the menu now has a Patch Notes entry showing every
  release's notes - no more digging through Discord history.
- As always, both the averaged and dice-sim engines were updated together and cross-checked
  against each other.

### Added
- **Per-profile `On` checkbox** (`${pid}-active`, default on) in the profile card header; both
  engines skip unticked cards; the card body greys out (`.profile-card.inactive`). Card state
  only - weapon library Save/Load never captures or touches it; persisted in the full form
  state (missing = on, so old states restore unchanged).
- **`Re-roll Non-Crit Wounds`** ability (`rerollNonCritWounds`, wound group), gated on
  Re-roll Wounds + Devastating Wounds. Analytic: per die `P(crit) = cw + (1−cw)·cw`,
  `P(wound) = cw + (1−cw)·w` with `cw` from the Anti-aware crit threshold; sim mirrors by
  re-rolling successful non-crit wound rolls. Takes precedence over the other wound re-roll
  branches. Lethal-hit auto-wounds are never fished (they don't roll).
- **Points inputs** `attacker-points`, `defender-points`, `char-points`/`char2-points`
  (0/empty = hide). Efficiency block in the analytic results and avg points destroyed / avg
  trade ratio in the probability view. Pure post-processing - no engine math involved.
  Defender/character points ride along in unit presets; attacker points in the form state.
- **Second attached character** (`char2-*` form block). New `readCharacter(prefix)` /
  `charEffectiveSave` / `charFailSaveChance` helpers feed both engines; the Monte-Carlo
  allocator fills characters in order, the analytic engine chains its deterministic-mean spill
  approximation (char 1's kill threshold consumes spill before char 2 sees any). Results and
  probability views render a characters array (`+char`, `+char2` bars).

- **In-app Patch Notes** - new menu entry opens a modal (`openPatchNotesModal`) that fetches
  `CHANGELOG.md` (`cache: 'no-store'`) and renders each release's "Player-facing patch notes"
  bullets (`renderPatchNotes`; `esc()` first, then minimal `**bold**`/`` `code` `` markdown).
  `build.js` now copies `CHANGELOG.md` into `dist/` so the deployed site serves it. Keep
  writing the player-facing section per release - it IS the in-app patch notes.

### Changed
- **"Re-roll Non-6s" renamed "Re-roll Non-Crit Hits"** (label/summary only - the stored
  `rerollNonCrits` prop is unchanged, so saved states/presets are unaffected) and its gate
  relaxed from Sustained-only to Sustained **or** Lethal Hits (tooltip updated). The new name
  matches "Re-roll Non-Crit Wounds" and is accurate under Crit on X+.
- `saveState`/`getCurrentState`/`captureDefenderData` and their restores now share
  `captureCharState`/`restoreCharState(prefix)`; `clearDefender` resets both characters
  through the same helper. Old saved states/presets/exports load unchanged (character 2
  simply restores as unattached; no schema bump needed).

### Validation
- Node cross-validation harness (stub DOM driving the real `calculateBattle` +
  `runMonteCarlo`): 10 scenarios × pre-save wounds / dev wounds / effective damage / per-char
  spill, 200k MC iterations - analytic within MC noise everywhere except the documented
  approximation zones (near-wipe effective damage, near-threshold character spill).

---

## v0.5 - Sorted ability lists, per-side Clear (2026-07-10)

### Player-facing patch notes (copy for Discord/YouTube)
- **Abilities are sorted now** - every ability list (weapon abilities, both battlegroups, and
  the defensive modifiers) is grouped into columns by the roll it affects: Hit / Wound /
  Damage & Misc on the attack side, Hit & Wound / Saves / Damage on the defence side. Inside
  each column the big stuff is at the top (full re-rolls before re-roll 1s), so what you need
  is where you'd expect it.
- **Clear only clears its own side** - the attacker's Clear button no longer wipes the defender
  (and vice versa). Set up a defender once and try different attackers against it freely.
- **Font fix** - the Attacks and Damage inputs now use the same font as the other stat fields
  (they looked different because they accept dice notation like D3/2D6).
- **"Re-roll Wounds" is now "Re-roll Wounds / Twin-linked"** (weapon ability and attacker
  battlegroup) so you can find it under the name printed on your datasheet.

### Changed
- **All five ability lists grouped into headed columns** (weapon abilities, attacker BG,
  defender BG, unit defensive mods, character defensive mods). `ABILITY_CONFIG` entries carry a
  `group` tag (`hit`/`wound`/`dmg`) and the array is ordered most-impactful-first within each
  group; `appendAbilities` renders one `.ab-col` per `ABILITY_GROUPS` entry. The static HTML
  lists got the same `.abilities-list.grouped` / `.ab-col` markup by hand. Columns stack
  vertically on mobile (≤768px). Consumers are id/prop-keyed, so save/restore, presets and the
  BG greyout are unaffected; results ability summaries now list in the new grouped order.
- **Attacker Clear is now side-scoped** - `clearAll()` (which reset the defender, character and
  both battlegroups too) replaced by `clearAttacker()`: attacker battlegroup, unit name and
  weapon profiles only. Button id `btn-clear-all` → `btn-clear-attacker`. There is no longer a
  whole-app reset button.

### Fixed
- **Attacks/Damage inputs rendered in the wrong font** (Inter, left-aligned) because the
  name-field styling targeted every `input[type="text"]`; it's now scoped to `.unit-name` rows,
  so dice-notation inputs match the mono, centred number fields.

---

## v0.43 - Per-weapon library, named battlegroups, styled dialogs (2026-07-07)

### Player-facing patch notes (copy for Discord/YouTube)
- **Weapon library** - every weapon card now has its own Load/Save buttons. Save a weapon once,
  drop it into any profile slot later. Your old saved loadouts were automatically split into
  individual weapons, so nothing is lost.
- **Named battlegroups** - give your attacker/defender battlegroups a name and save them with
  one click, no popup.
- **More accurate maths** - fixed several cases where the average results drifted from the
  dice-simulation view: "Critical Hit on 5+" style abilities under hit penalties, limited
  damage re-rolls, battlegroup-wide Sustained Hits D3/D6, and damage spilling onto an attached
  Character (the Character now takes its own saves, and Devastating Wounds bypass them).
- **FNP fix** - the Feel No Pain defensive modifier now saves with your setup, presets and
  history instead of quietly resetting.
- **Sharing improvements** - export/import files now carry your weapon library, importing old
  files recovers your weapons, and shared files can no longer duplicate entries.
- **Nicer dialogs** - all remaining browser popups replaced with in-app dialogs.

### New
- **Per-weapon library (replaces whole-loadout weapon presets).** Each weapon profile card now
  has its own **Load / Save / Upd / Del** buttons (Upd/Del appear once the card is tied to a
  saved weapon). Save stores a single weapon to a library and ties the card to it; Load drops a
  library weapon into that card; Upd overwrites the tracked weapon's stats/abilities (keeps its
  name); Del removes it from the library. The old top-of-panel "Load/Save Weapons" (whole loadout
  at once) is gone.
  - **One-time migration** (`migratedWeaponV1`): every weapon profile inside existing
    whole-loadout attacker presets is lifted into the new single-weapon library (names deduped),
    so nothing saved is lost.
  - Export/Import now also carries the `weapon` library.
- **Battlegroups have a name field.** Inline "Battlegroup name" input on both the attacker and
  defender battlegroup rows. **Save BG saves under that name - no popup.** The name persists
  across reloads and through Export/Import, and Load fills it back in. (Empty/duplicate names
  fall back to the styled name modal.)
- **Styled in-app name modal** (`openNameModal`) replaces the native `window.prompt()` used when
  naming a preset - matches the app theme, supports Enter/Esc/click-outside, and shows duplicate
  -name errors inline instead of a separate `alert`.

### Changed
- **Layout (attacker):** order is now Battlegroup → battlegroup abilities → Unit Name → weapon
  profiles (Unit Name moved below the battlegroup).
- **Preset controls sit with their content:** each section's Load/Save lives on its own header
  row (Battlegroup row, Unit row, per-weapon card) rather than a single ambiguous set in the
  panel header; the panel header keeps only **Clear**.

### Fixed (2026-07-07 review sweep - engines cross-validated with a Node harness)
**Engines (both `calcProfile` and the Monte-Carlo engine, kept in agreement):**
- **Crit on X+ must also hit** - the analytic engine counted critical hits that missed (e.g.
  Crit 5+ while only 6s hit under Cover + −1 to Hit), overcounting by up to +67% and letting
  `normalHits` go negative with Lethal Hits. Crit chance is now capped at the hit chance.
- **Re-roll N Damage (limited)** was treated as re-roll ALL in the analytic engine (~+17%).
  Now blends the pmf by `E[min(N, below-average rolls)]` via `expectedCappedRerolls`.
- **Battlegroup Sustained Hits X value** never won the ability merge - a profile's default "1"
  beat the BG's value (BG "Sustained D3" silently computed as Sustained 1). A profile now only
  claims a sustained value when its own Sustained box is ticked; the BG greyout also disables
  the per-profile value input.
- **Attached-character spill** had contradictory semantics (analytic double-saved spill wounds;
  Monte-Carlo applied no character save and rolled FNP twice at different granularities). Both
  engines now agree: spill wounds roll the **character's** save (dev wounds skip it), then the
  character's FNP **once per wound**, then damage with the character's own reductions applied
  per damage value. FNP is one roll per wound everywhere (deliberate simplification).
- Sim restructured so saves/FNP/damage are rolled at allocation time (`simProfile` emits
  pre-save wound events); analytic no longer ignores the sergeant's extra wound in spill.

**Persistence / data:**
- **FNP modifier (defender + character) is now saved** - it was missing from saveState/restore,
  Defender presets, calc-log recall, and both Clear buttons (it silently survived Clear and
  vanished on reload).
- **Importing a pre-v0.43 export now lifts old whole-loadout attacker presets into the weapon
  library** instead of leaving them invisible (no UI for that category any more).
- **Names are HTML-escaped everywhere they're rendered** (preset lists, results titles,
  per-weapon breakdown, history) - a shared import file with a hostile preset/unit name could
  previously run script (stored XSS).
- Import / calc-log recall now clears the active-preset indicators (Upd could overwrite the
  previously loaded preset with the imported form).
- Upd no longer renames a preset to an empty or duplicate name.
- Deleting a weapon from the Load modal now unties any profile card still pointing at it.
- The card↔library-weapon tie survives a page reload (Upd/Del reappear).
- Save BG / Save Unit now mark the new preset active immediately (Upd/Del appear without a
  round-trip through Load).
- Name modal: re-entrancy guard (double-click can't stack two dialogs / lose a save), focus
  trap, Escape works anywhere in the dialog, focus returns to the opener.
- All remaining native `confirm()`/`alert()` dialogs converted to styled modals
  (`openConfirmModal` / `openInfoModal`).
- Dead code from the removed whole-loadout attacker presets deleted (`PRESET_CATS.atk`,
  orphaned buttons wiring, `captureAttackerData`/`restoreAttackerFromData`).

**Build / tooling:**
- `deploy.bat` now aborts on a failed clone/pull/cd/copy - previously a failure fell through
  and could commit un-obfuscated source (with Firebase placeholders) to the main repo, or push
  a mismatched index.html/app.js pair.
- `build.js` resolves paths from its own directory and fails loudly on a missing Firebase
  config key or an unreplaced `%%FIREBASE_*%%` token.
- `start.bat` opens the browser after the server is up and errors clearly if Python is missing.

### Fixed (second adversarial review pass over the sweep itself)
- Analytic character spill now lets **Devastating** spill wounds bypass the character's save
  (the sim already did; the analytic under-reported char damage up to 2× with dev weapons).
- Calc-log snapshots (`getCurrentState`) now carry the FNP modifier, so log recall reproduces
  the result.
- Sustained Hits value escaped in the hit-results breakdown (last missed XSS sink).
- v0.43 exports no longer duplicate legacy weapons on import: the migration clears the lifted
  `attacker` list, and import only lifts `attacker` entries from files without a weapon library.
- Message modal: Enter on a focused Cancel/close button cancels instead of confirming.
- Sustained ticked with an empty value box now means "1" in both engines (the empty string
  could previously pull in the battlegroup's value in one engine only).
- The FNP defensive modifier now shows in the results ("Wounds After FNP" row, defender
  FNP+++ suffix, summary tags) via a shared `effectiveDefFnp()` used by engines and display.
- deploy.bat/start.bat rewritten with CRLF endings (cmd's goto/label scanning is unreliable
  with LF-only files); start.bat probes `python --version` (the `where` check passed on the
  Windows-Store alias stub).

Released after a manual browser test (2026-07-07) plus the automated engine cross-validation
(16 cases) and jsdom persistence/import/XSS/dialog suite (33 checks).

---

## v0.42 - Saveable battlegroups + JSON export/import (2026-06-29)

### New
- **Battlegroups are now their own saveable presets** - separate **Attacker Battlegroup**
  and **Defender Battlegroup** lists, each with Save / Load / Update / Delete (buttons on the
  battlegroup row). Attacker (weapons) and Defender (unit + character) presets are now
  **focused** - loading one no longer changes your battlegroup.
- **Export / Import** (buttons in the top toolbar) - Export downloads a single JSON file
  (`ba-mathhammer-profiles.json`) containing your current setup **and** all saved presets,
  versioned. Import reads it back and **merges** presets (name clashes get an "(imported)"
  suffix, nothing is overwritten) and restores the saved setup into the form. Works across
  browsers/PCs and for sharing.
- **One-time migration**: any existing attacker/defender preset that had a battlegroup baked in
  has its battlegroup lifted into the new battlegroup lists (named "<preset> BG"), so nothing is
  lost by the split.

---

## v0.41 - 11th-edition damage modifier order (2026-06-29)

### Fixed
- **Damage modifiers now follow the 11th-edition order:** multiplication → addition
  → division → subtraction → **round up any fractions at the end** (minimum 1).
  For the calculator's damage modifiers that means **+Melta, then ÷2 (Half Damage),
  then −1 (−1 Damage), then round up**.
  - **Melta is now affected by Half Damage** - the melta bonus is added *before* the
    halving, so it gets halved too. Previously melta was added last (never halved).
    Example: D6 + Melta 2 vs a Half-Damage target averaged 4.0 before, now **3.0**.
  - **Half + −1 Damage** changed too (division before subtraction, single rounding at
    the end): e.g. a D6 hit averages **1.33** now vs 1.67 before.
  - **Melta + −1 only** (no halving) is unchanged.
- Applied consistently in both engines (averaged and Monte-Carlo), which agree
  exactly on per-instance damage.

---

## v0.4 - Effective damage, limited re-rolls & RAW damage re-roll (2026-06-29)

### Effective damage - no overkill spillover (significant fix)
- **Damage no longer spills between models.** Previously both engines pooled all
  damage into a single health bar and did `floor(total / wounds)`, which let
  overkill from a dying model carry onto the next one - inflating kills and the
  damage %. Now each wound that gets through is allocated to one model; damage
  accumulates *within* a model across hits, but overkill on the killing blow is
  **wasted** (it does not carry over).
  - Example: a flat **3-damage** weapon landing ~2.89 hits on **2-wound** models
    previously reported **4 kills**; it now correctly reports **~2.89** (each hit
    kills one model, 1 damage wasted).
  - Biggest corrections: high-damage weapons into low-wound models (e.g. D6 into
    1-wound chaff, multi-damage guns into 2-wound infantry).
- **"Total Damage" is now "Effective Damage"** - Models Killed, Damage %, and the
  per-model bars all derive from effective (non-wasted) damage.
- **How it's modelled:** the **Monte-Carlo / BGV engine** is exact (allocates each
  instance per model, discards overkill; character spill is the instances arriving
  after the unit is wiped). The **averaged engine** is distribution-aware (uses the
  damage roll's full pmf to find the expected instances to kill a model) - exact
  when per-hit damage ≥ wounds (the main overkill case); for per-hit damage **<**
  wounds (pure accumulation) it slightly over-counts, where the BGV view stays exact.

### New
- **Re-roll N Hits / Wounds / Damage** (per-weapon-profile, configurable N).
  Models rules like *"each time this model shoots, you can re-roll one Hit roll,
  one Wound roll and one Damage roll"* (e.g. Lancer, Prism). You only get a
  limited number of re-rolls, spent on failures - not a blanket re-roll-all.
  - At low shot counts (2 shots) this is close to, but slightly less than,
    "re-roll all"; the difference grows the more dice you roll.
  - If "Re-roll all" (Hits/Wounds/Damage) is also enabled, it takes precedence.

### Changed / Fixed
- **Re-roll Damage is now RAW.** Previously the averaged engine modelled it as
  *"keep the better of two rolls"*; the rules actually force you to keep the
  re-rolled result. It now models *"re-roll a below-average roll, keep the new
  result."* This **lowers existing Re-roll Damage results by ~5%** to the correct
  value (e.g. D6 average 4.47 → 4.25) and makes both engines agree.
- **Wound re-roll accuracy.** Limited wound re-rolls now use an attack-level
  binomial, removing a fractional-dice rounding error (~3.5%) in the averaged
  engine.

### Under the hood
- Cross-validated the averaged engine against the Monte-Carlo engine across all
  re-roll scenarios - agreement within ~2%. Remaining small residuals (limited
  wound re-rolls combined with Sustained/Lethal Hits; the "re-roll the duds"
  damage model at high shot counts) are documented in code.

---

## v0.38 - New-edition rules update (2026-06-26)

Brings the calculator in line with the current WH40K core rules.

### Cover & to-hit modifiers (the big change)
- **Cover now worsens BS by 1 (−1 BS characteristic)** instead of giving +1 to
  the save. Negated by **Ignores Cover**; ignored by **Psychic**.
- **New "−1 to Hit" defender toggle** - a penalty to the *hit roll itself*, a
  different modifier type from Cover's −1 BS. The two **stack** (both on = −2 to
  hit). Not removed by Ignores Cover.
- **New "+1 Sv" defender toggle** - improves the armour save by 1 (not invuln),
  for rules that still grant a save bonus. Available on the unit, the attached
  character, and the battlegroup.

### New abilities
- **Cleave X** (per-profile) - +X attacks per 5 models in the target unit (single
  target), à la Blast.
- **Psychic** (per-profile) - ignores hit-roll / BS modifiers (e.g. Cover,
  −1 to Hit).
- **+X AP** (offensive battlegroup) - improves the AP of all weapons by X
  (applied before the defender's −1 AP).

### Fixes
- **+X Attacks** battlegroup option now persists across reloads and presets
  (was silently lost before).
- Character spill-calc no longer used the old "cover = +1 Sv" behaviour.
- Monte-Carlo engine now enforces *unmodified 6 always hits / 1 always fails*,
  so a high-BS unit in Cover + −1 to Hit can still hit on a 6.

---

*Earlier versions (≤ v0.35) predate this changelog.*
