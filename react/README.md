# @bc-forge/react

React hooks and components for the bc-forge SDK.

Supported Node, React, Stellar SDK, and package ranges are in the [compatibility matrix](../docs/COMPATIBILITY.md).

## Visual snapshots

Storybook stories for Alert, Badge, Dropdown, and the product components live in `react/stories/`. Playwright compares each story to a committed baseline under `react/visual/__screenshots__/`.

From the repository root:

```bash
npm run test:visual --workspace @bc-forge/react
```

That builds Storybook and fails if a screenshot differs from its baseline (`maxDiffPixels: 0`). An unapproved UI change fails the check. CI runs the same command on `ubuntu-24.04-arm` in `.github/workflows/react-visual.yml`, which matches the `linux-chromium-arm64` baseline directory.

To replace baselines after an intentional visual change, regenerate them in that same environment (the Playwright container image is `mcr.microsoft.com/playwright:v1.63.0-noble` on arm64) and commit the new PNGs:

```bash
npm run test:visual:update --workspace @bc-forge/react
```

Do not hand-edit the PNG files. `updateSnapshots` is `none` in `react/playwright.visual.config.ts`, so a normal `test:visual` run never rewrites baselines.
