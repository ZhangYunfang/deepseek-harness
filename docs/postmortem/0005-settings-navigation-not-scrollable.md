# Post-mortem 0005: The Settings navigation rail clipped its own overflow, hiding installed sections

English | [中文](0005-settings-navigation-not-scrollable.zh.md)

Status: resolved upstream in `47ee6080` (`fix(web): allow overflowing settings navigation to scroll`); backported onto `dsh-v0.1.2-alpha.2`

## Executive summary

The Settings dialog's section rail could not scroll. Its wrapper pinned one panel height and clipped whatever exceeded it, and no element inside the rail owned that overflow, so as soon as installed plugins pushed the registered sections past the panel the last rows became permanently unreachable: no scrollbar, and no element for the wheel to scroll. It escaped because the shipped section count fit the panel and no test ever forced the rail past it. The durable lesson is that a fixed-height flex container with a variable-length child list needs an explicit overflow owner, and the check for it has to overflow the container deliberately rather than trust the section count a build happens to ship.

## Summary

The Settings shell renders a fixed-height modal panel (`height: min(800px, calc(100vh - 48px))`) split into a navigation rail and a content column. The rail stacks a title and a list of section cells, one per registered `settings.section` contribution: four from the shipped client packages, plus one for each installed plugin that contributes a page.

With enough sections the list is taller than the rail, and the bottom rows are cut off by the panel's `overflow: hidden` with no way to reach them. Nothing looks broken at the shipped section count, which is why it survived review: the defect depends on how many plugins a user installs, not on the code as built. A second contributor made the failure look intentional. The panel's layout comment stated that the content which does not fit scrolls in `.options`, which holds for the content column and is silently false for the rail beside it.

The fix is one declaration, `overflow-y: auto` on the section list, because in flex layout a non-`visible` overflow also zeroes the item's automatic minimum size. That is the whole mechanism: before the declaration the list could not shrink below its content height, and `overflow-y: auto` is what stops it from being clamped there.

## Impact

A user with enough plugin-contributed settings pages could not open the last few sections at all: not by scrolling, not by keyboard, not through any affordance the dialog offered. Those sections are where a plugin exposes its configuration, so the practical effect was that the plugin became unconfigurable, with no error and no hint that content existed below the fold.

Measured on a local `0.1.2-alpha.2` installation with 19 registered sections at a 1440x800 viewport: the panel is 752 px tall and the rail's list needs 832 px, so 144 px of cells, the last three or four, sat outside the panel's clip. At 1440x620 the same list loses 324 px.

## Timeline

- 2026-08-30: `dsh-v0.1.2-alpha.2` ships with the rail's list owning no overflow.
- 2026-09-20: upstream `47ee6080` adds `overflow-y: auto` to the section list and an end-to-end check that shrinks the viewport to force the rail to overflow.
- 2026-10-28: reproduced on a local `0.1.2-alpha.2` installation (19 sections, 144 px clipped at 1440x800), confirmed against the bytes the running server actually serves, backported the upstream declaration, and verified in a real browser with ten assertions, including that the last cell becomes clickable after scrolling and that every cell keeps its 40 px height.

## Root cause

Three declarations cooperate to hide the content, and each is correct on its own:

```css
.panel { height: min(800px, calc(100vh - 48px)); overflow: hidden; }  /* fixed height, clips */
.nav { flex: none; display: flex; flex-direction: column; }           /* owns no overflow */
.navList { display: flex; flex-direction: column; gap: 4px; }         /* owns no overflow */
```

The panel is a definite-height flex row, so the rail is stretched to the panel's height. The rail is a column flex container, and its list is a flex item with `overflow: visible`, so the list's automatic minimum size resolves to its content height: the list refuses to shrink and overflows the rail, and the panel's `overflow: hidden` clips that overflow away. No element in the chain is a scroll container, so there is nothing for the browser to attach a scrollbar to and nothing to route wheel deltas to.

The one-declaration fix works because the automatic minimum size applies only while the item's overflow is `visible`:

```css
.navList { display: flex; flex-direction: column; gap: 4px; overflow-y: auto; }
```

With `overflow-y: auto` the item's automatic minimum size resolves to zero, the list shrinks into the space left under the title, its content overflows inside it, and the list becomes the scroll container. The title stays fixed while only the cells move. An explicit `min-height: 0` on the same item is equivalent; the shipped fix relies on the property that also provides the scrolling, which is why one line is sufficient.

## Why every safety net missed it

- **The failure scales with the installed plugin set.** Component tests mount the shell with a handful of synthetic sections, which fit the panel. The defect needs more sections than the panel height allows, so nothing about the built code is wrong until a user's plugin set is large enough.
- **The panel is deliberately content-independent.** Its fixed height is the feature: a content-sized panel would resize under the pointer on every navigation click. The same decision means the panel can never grow to fit a longer rail, so the rail must own its overflow. The layout comment described the content column's scrolling and was read as a specification of both columns.
- **A clipped region looks like a short list.** `overflow: hidden` produces no scrollbar, no scroll shadow, and no DOM hint. Reviewing the dialog at the shipped section count shows a tidy, complete list.
- **Nothing asserted scrollability.** The end-to-end suite opened the dialog and changed sections, but never inspected the rail's geometry, so "the list fits" and "the list can scroll" were indistinguishable to the suite.

## Guardrails added

- The upstream fix carries an end-to-end check that sets a 320x240 viewport to guarantee overflow, then asserts that the rail's `scrollHeight` exceeds its `clientHeight`, that wheeling over the rail moves only the rail (its `scrollTop` grows while the title's bounding box and the content column's scroll offset stay put), and that the last cell is hittable and selects its section.
- A regression check for this layout must create the overflow it tests. Reproducing with an overflowing viewport, never with the shipped section count, is what makes the check meaningful.
- "The container is fixed-height" carries an obligation to name the overflow owner. A fixed-height flex container whose children vary in number or length states which descendant scrolls, or it will clip.

## Lessons

A layout that clips on purpose is indistinguishable, from the outside, from a layout that lost content by accident. The decision to clip belongs with an explicit statement of what remains reachable, and the reachability claim needs a test that overflows the container, because a container that fits today's content proves nothing about tomorrow's.

## Backport note

`dsh-v0.1.2-alpha.2` predates the upstream fix, so this branch carries the upstream declaration verbatim (the same one-line change to the section list and the same correction to the panel's layout comment) plus the documentation on this page. Runtime verification used a headless-browser harness against a live installation rather than the upstream end-to-end suite, because that suite belongs to the later layout: its ten assertions cover the facts the upstream check establishes (overflow exists, the rail scrolls, the title stays fixed, the last cell selects its section) and add the row-height and short-viewport cases.
