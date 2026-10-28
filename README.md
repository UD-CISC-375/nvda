# nvda-dynamic-testing-framework

This repository is going to be used to develop the testing framework for NVDA. There will be a separate repository linked shortly (https://github.com/mlorang/nvda-dynamic-testing-webpage) to the testing website with DOM dynamic componets for testing Dynamic DOM states to implement into NVDA.

## Menu button: accessibility before and after a DOM change

The first scenario we test is Team Y's menu button:
https://mlorang.github.io/nvda-dynamic-testing-webpage/#/disclosure/menu?variant=accessible

For each variant (`accessible` and `broken`) the test:

1. Opens the page and runs an axe-core scan (the **before** baseline).
2. Starts a `MutationObserver` on `.menu-wrapper`.
3. Clicks the menu button, either automatically or by you (see below).
4. Waits for the observer to report the expected change: the button's `aria-expanded` going to `"true"` and the menu's `hidden` attribute being removed.
5. Runs the same axe scan again (the **after** scan) and reports any violations the change introduced.

### Setup

```
npm install
npx playwright install chromium
```

### Running

| Command | What it does |
|---|---|
| `npm test` | Playwright clicks the button itself (headless) |
| `npm run test:headed` | Same, but you can watch the browser |
| `npm run test:manual` | A browser opens and waits for **you** to click the menu button (2 min per test) |
| `npx playwright show-report` | Reopens the HTML report from the last run |

The HTML report opens in your browser automatically when a run finishes (press Ctrl+C in the terminal to stop the report server). In the report, each test has these attachments:

- `axe-before.json` and `axe-after.json`: the full axe results (violations, passes, incomplete) for each scan.
- `stdout`: what the test logged, including the DOM changes the observer recorded.

### What the results show

| Variant | Before | DOM change detected? | After |
|---|---|---|---|
| accessible | 0 violations | Yes: `aria-expanded` false → true, `hidden` removed | 0 violations |
| broken | `button-name` (icon button has no name) | No: the menu `<ul>` is just inserted, no state attributes | `button-name`, **nothing new** |

The tests assert these outcomes, so `npm test` passes when the broken variant fails in the expected ways.

Note that **axe reports nothing new after the broken menu opens**. axe has no way to know the button should announce "expanded". Only the DOM watcher catches that the screen reader is never told the state changed. A scan before and after on its own is not enough. We also need to check that the right kind of change happened.

### Internal results.json schema (version 0)

The shared TypeScript contract is [`results/schema.ts`](results/schema.ts);
[`examples/results.json`](examples/results.json) is a hand-written menu
example for both variants, not a captured test run. Its timestamp, commit,
tree node IDs and tree contents are illustrative. This format is internal to
this repository, is expected to change, and is not yet emitted by the tests.

The top level has `schemaVersion: 0` and `run`: an ISO 8601 `timestamp`,
git `commit`, base `siteUrl`, and `scenarios`. Each scenario identifies the
scenario and variant, has a `verdict` (`passed`, `failed`, or `skipped`),
and contains ordered `states`. A skipped scenario must have a nonempty
`skippedReason` and may have no states. Verdicts describe the scenario's
accessibility outcome, not whether a Playwright test asserting an expected
failure passed.

State indices start at zero and follow array order. Each state has a
human-readable `trigger` and `domChangeSummary`, a `treeDiff`, `axe` results,
and `stable` (whether the state was observed to settle, not whether it
passed). The first state is the baseline: its tree diff and axe `new` and
`resolved` lists are empty. Subsequent diffs compare with the previous state.

`treeDiff` contains `added` and `removed` subtrees with their parent ID and
zero-based child index, `changed` properties by node ID, and `reordered`
child ID sequences by parent ID. Node IDs must identify the same tree node
across states within a scenario; a null parent denotes the root. Subtrees
include their children recursively. Property changes omit `before` or `after`
when a property is absent; JSON `null` is a value, not absence. Reorder
sequences describe the children shared by both states, before and after the
reorder; additions and removals are recorded separately.

`axe.new` and `axe.resolved` contain rule/target pairs, with `target` preserving
axe's selector array. `totalCount` counts current rule/target pairs, not
distinct rules; the baseline count includes existing violations even though
its diff lists are empty.

### Files

- `playwright.config.ts`: points Playwright at Team Y's site (`baseURL`) and defines the `chromium` (automatic) and `manual` projects.
- `tests/menu.spec.ts`: the menu tests.
- `tests/inline-validation.spec.ts`: checks blur-triggered error feedback for accessible and broken variants, including the live DOM changes and axe scans. The semantic WCAG checks are asserted directly because axe cannot infer whether an error message belongs to a field.
- `tests/helpers/axe.ts`: `runAxe()` (scan + report attachment) and `diffViolations()` (violations that are new since the before scan).
- `tests/helpers/watch-dom.ts`: `watchDom()`, a MutationObserver in the page that reports each change back to the test, and `waitFor()`, which waits until the change we expec