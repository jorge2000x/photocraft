# SPEC DRAFT: Distribute and Space for layers (#10715)

Status: **draft**. Open for improvements and reworks. This PR is intentionally partial: it is probably missing features that would fit in it, and nothing here is final.

Tracking issue: storytold/photocraft#10715 ("Add distribute (even spacing) for multiple selected layers").

## Summary

Add four commands that spread the selected layers evenly, matching Affinity's behaviour:

- **Space Horizontally / Space Vertically**: equal gaps between layer edges.
- **Distribute Horizontally / Distribute Vertically**: equal distances between layer centres.

Both work inside the bounds of the selection. **Auto** is on by default and computes the spacing. When it is off, a fixed gap is typed in. Changes preview live, and each application is a single undo step.

Scope note: the issue asks for "canvas or selection". Affinity only distributes within the selection bounds, so this spec follows Affinity. Canvas-relative distribution is left as a possible extension, not part of v1.

## Reference behaviour (Affinity)

Taken from Affinity's public help pages and one third-party tutorial. Only observed behaviour is used. No Affinity code, assets or UI copy.

| Behaviour | Affinity | Source |
| --- | --- | --- |
| Distribute | Equal distance between object **centres** | Affinity Designer 2 help, "Distributing/spacing objects" |
| Space | Equal distance between object **edges** | Same page |
| Directions | Space Horizontally and Space Vertically; can be applied together | Affinity Designer iPad help |
| Reference | Always the selection bounds. "Align to" does not apply | Affinity Designer iPad help; FontBundles tutorial |
| Auto | On by default: distributes within selection bounds. Off: a width/gap field appears | Affinity Photo 2 iPad help, "Spacing" |
| Preview | Options update in real time (iPad). Desktop v1 uses Apply | Affinity Photo iPad help; Designer v1 help |
| Key object | Modifier + tap sets a key object. Spacing is based on the gap to the leftmost item, or to the rightmost item with the modifier | Affinity Photo 2 iPad help |
| Not enough room | Overlap is spread evenly, not a gap. Observed, not documented | Scrapbook Campus tutorial (2026) |

Affinity Photo's layer documentation only describes **Space**. Distribute by centres is documented for Designer. This spec implements both, as the issue asks.

## Requirements

| ID | Requirement |
| --- | --- |
| RF-1 | Four commands (Space H/V, Distribute H/V), listed next to the existing align commands in the menu and the align panel. |
| RF-2 | Enabled with two or more selected layers. With two layers in Auto mode nothing moves, so the result is neutral. |
| RF-3 | Reference is the union of the layer boxes in the selection. There is no "Align to" option for these commands. |
| RF-4 | **Auto** is on by default. When it is off, a numeric field appears for the gap, in document units. |
| RF-5 | In Auto mode the outermost layers on each side stay where they are. Only the inner layers move. |
| RF-6 | Space uses equal edge-to-edge gaps. Distribute uses equal centre-to-centre steps. |
| RF-7 | Horizontal and vertical can be applied together. Each axis is computed independently. |
| RF-8 | Live preview when the option or gap value changes. Apply commits, cancel restores the original positions. |
| RF-9 | Each application is one undo/redo entry, like the existing align commands. |
| RF-10 | Same layer eligibility as the existing align commands: groups, locked and hidden layers are handled the same way (to be confirmed in code). |
| RF-11 | If the layers do not fit, a negative gap is applied (uniform overlap) and no error is raised. |
| RF-12 | (v1.1) Key object: a modifier-click sets the reference layer. Spacing is based on the gap to the leftmost/topmost item, or to the rightmost/bottommost one with the modifier. |

## Algorithm

Each axis is independent. For horizontal, *start* is `x` and *length* is `width`. For vertical, use `y` and `height`. Boxes are axis-aligned.

1. Sort layers by **centre** (ties: start, then original order), so the result is deterministic.
2. Selection bounds: `start = min(start)`, `end = max(start + length)`.
3. **Space, Auto:** `gap = (end - start - sum(length)) / (n - 1)`. The first layer goes to `start`, each next one to `previous.end + gap`.
4. **Space, fixed gap:** same sequential placement from `start`, using the typed gap.
5. **Distribute:** `step = (lastCentre - firstCentre) / (n - 1)`. Layer *k* gets centre `firstCentre + k * step`. The first and last centres do not move.
6. Round final positions to the document's pixel grid. The error (at most 0.5 px per layer) is accepted and not redistributed.

```ts
// Reference only. The implementation language is Rust (see AGENTS.md).
function space(items, gap /* undefined = Auto */) {
  const o = [...items].sort((a, b) => (a.s + a.l / 2) - (b.s + b.l / 2) || a.s - b.s);
  const start = Math.min(...items.map(x => x.s));
  const end = Math.max(...items.map(x => x.s + x.l));
  const g = gap ?? (end - start - o.reduce((a, x) => a + x.l, 0)) / (o.length - 1);
  let cur = start;
  return o.map(x => { const r = { id: x.id, s: cur }; cur += x.l + g; return r; });
}

function distribute(items) {
  const o = [...items].sort((a, b) => (a.s + a.l / 2) - (b.s + b.l / 2) || a.s - b.s);
  const c0 = o[0].s + o[0].l / 2, c1 = o.at(-1).s + o.at(-1).l / 2;
  const step = (c1 - c0) / (o.length - 1);
  return o.map((x, k) => ({ id: x.id, s: c0 + k * step - x.l / 2 }));
}
```

### Verification

A prototype of these two functions was tested in Node with 5,000 random sets of 2 to 8 layers:

- Space gaps and Distribute steps are equal in every case.
- Selection bounds are preserved.
- With gap ≥ 0, applying the command again changes nothing (4,596 cases).
- With overlap (gap < 0, 404 cases), applying it again changed the result in 36 of them, because the centre order can change after the first pass. This is accepted for v1 and listed under edge cases.

These checks are on the prototype only. Rust unit and property tests are part of the acceptance criteria below.

## Edge cases

| Case | Proposed behaviour | Origin |
| --- | --- | --- |
| Layers do not fit | Negative gap, uniform overlap, no error | Observed in Affinity (not documented) |
| Re-applying with overlap | May change the result, because centre order changes. Accepted for v1 | Measured in prototype |
| Large layer in the middle | First and last in **centre** order go to the selection bounds, so an outer edge layer can move. RF-5 applies to centre order | Decision in this spec |
| Equal centres | Tie-break by start, then original order | Decision in this spec |
| Rotated or transformed layers | Axis-aligned bounding box, same as align | Assumption: confirm in code |
| Groups | Treated as one item with the group's box | Assumption: confirm in code |
| Locked or hidden layers | Same rule as align | Assumption: confirm in code |
| All layers the same size | Space and Distribute give the same result | Math |
| Negative fixed gap | Allowed (intentional overlap) | Decision in this spec |
| Two layers in Auto | Nothing moves, command stays enabled | Decision in this spec |
| Layer outside the canvas | Included normally; reference is the selection, not the canvas | Affinity parity |
| Pixel layers | Same content bounds as align | Assumption: confirm in code |

## UI

- **Menu and align panel:** a new group with Space Horizontally, Space Vertically, Distribute Horizontally and Distribute Vertically, after the align buttons.
- **Options:** an **Auto** checkbox (on by default). When it is off, a numeric field appears next to it. With Auto on, the field shows the computed gap so the current value is visible.
- **Preview:** changing the command, direction or value moves the layers immediately. Apply commits. Escape, or changing the selection, discards.
- **Disabled state:** greyed out with fewer than two layers selected, with a tooltip explaining why.
- **Shortcuts:** none in v1. Affinity has no documented default shortcut for these commands that this spec could confirm.

## Acceptance criteria (v1)

- [ ] Four layers of different widths: Space Horizontally in Auto gives equal gaps and keeps the selection bounds.
- [ ] Same four layers: Distribute Horizontally gives equal centre steps. The visible gaps differ from Space.
- [ ] Auto off, gap 5 px: layers sit 5 px apart, starting from the first one.
- [ ] Space Vertically works in the same cases, swapping x/width for y/height.
- [ ] Horizontal and vertical applied together move each axis independently.
- [ ] Preview updates when the gap changes. Cancel restores the original positions.
- [ ] One undo reverts the whole application. Redo reapplies it.
- [ ] Fewer than two selected layers: commands are disabled.
- [ ] Layers that do not fit produce a uniform overlap, with no error.
- [ ] Groups, locked and hidden layers behave the same as with align.

## Tests (planned)

- **Unit tests** for the two pure functions: fixed cases from this spec, plus property tests over random inputs (equal gaps, preserved bounds, idempotence when gap ≥ 0).
- **Integration test:** apply, undo and redo on the history stack across several selections.
- **Graceful failure:** empty selection, one layer, zero-length layers, and out-of-range params return `Err`, never panic (see the "Never crash" rule in AGENTS.md).
- **Visual check:** compare final positions with Affinity on the same objects.

## Open questions

- [ ] **Canvas:** the issue says "canvas or selection". Confirm with the author whether canvas-relative distribution is needed, since Affinity does not do it.
- [ ] **Key object (v1.1):** the Affinity docs do not say exactly which gap is used between the key object and its neighbours. Test in real Affinity before implementing.
- [ ] **Sort order:** does Affinity sort by centre, by start edge, or by selection order? The docs do not say. This spec uses centre.
- [ ] **Auto field:** show the computed gap (this spec's proposal), or leave it empty?
- [ ] **Fixed gap anchor:** the first layer stays in place (this spec's proposal). Does Affinity anchor on the leftmost layer or the selection centre?
- [ ] **Related work:** issue #2349 and `storytold/vectorcraft#1199` were mentioned in the ticket. Check for reusable logic.

## Sources

Consulted on 2026-11-01:

- [Affinity Designer 2: Distributing/spacing objects](https://affinity.help/designer2/en-US.lproj/pages/ObjectControl/distribute.html)
- [Affinity Designer v1: Distributing/spacing objects](https://affinity.help/designer/en-US.lproj/pages/ObjectControl/distribute.html)
- [Affinity Designer iPad: Distribute](https://affinity.help/designeripad/en-US.lproj/pages/ObjectControl/distribute.html)
- [Affinity Photo 2 iPad: Spacing (layers, key object, Auto)](https://affinity.help/photo2ipad/en-US.lproj/pages/LayerOperations/distribute.html)
- [Affinity Photo v1: Spacing (layers)](https://affinity.help/photo/en-US.lproj/pages/LayerOperations/distribute.html)
- [Affinity Photo iPad: Distribute layers](https://affinity.help/photoipad/English.lproj/pages/LayerOperations/distribute.html)
- [Tutorial: Alignment option in Affinity Designer (FontBundles)](https://fontbundles.net/design-school/accessing-the-alignment-option-in-affinity-designer)
- [Scrapbook Campus: Align, space and distribute in Affinity](https://scrapbookcampus.com/2026/07/align-space-distribute-affinity/)

Rows marked "Observed" or "Decision in this spec" in the tables above are not from the official documentation.
