# mm-track-widget

The GitHub Pages site the Medically Modern marketing site embeds. Pages deploys from
`main` (classic deploy-from-branch): **merging to `main` is deploying**. Pages serves the
HTML with a 10-minute cache, so changes are live within ~10 minutes of the merge.

| File               | What it is                                                              |
| ------------------ | ----------------------------------------------------------------------- |
| `intake-form.html` | **The live patient intake form** — the file the marketing site iframes at `medicallymodern.com/intake-form/`. |
| `index.html`       | Drop-off tracker widget for JotForm-hosted forms. Runs only inside JotForm's own pages, so the form above never loads it; kept while any JotForm form is still live, delete when JotForm is retired. |

## Editing the form

`intake-form.html` is the whole form: one self-contained HTML file, no build step. Edit
it, merge to `main`, and the live form updates — that is the entire deploy story. Its own
tracking is built in: Microsoft Clarity (project `wlf9luix56`, same dashboard as before)
and partial-lead beacons to the backend on every step, so a drop-off still produces a
monday row.

The backend is `server/` in
[dtc-mm-form-H7eG34s](https://github.com/medically-modern/dtc-mm-form-H7eG34s) — Express
on Railway, writing to monday. Form changes that add or rename answer options usually need
a matching update to `server/src/labelMap.js` there. That repo also carries the same form
file as its `index.html`, served standalone at its own Pages URL (where texted resume
links fall back to) — when either copy changes, copy the file across so the two stay
byte-identical: `diff` between them should print nothing.
