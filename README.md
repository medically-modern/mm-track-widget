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

## Layouts (one form, different question orders for ads)

The same file serves every version of the form; `?v=<name>` on its URL picks the
question order. No `v` (or one the form doesn't know) is the main form, unchanged.

| `?v=`     | Order                                                                                   |
| --------- | --------------------------------------------------------------------------------------- |
| *(none)*  | Main form: reason → contact → what they need → doctor → insurance → coverage → confirm.  |
| `insulin` | Opens on the CGM qualifying question ("Which of the following applies to you?" — insulin / low blood sugar / neither), then the main form; step 3 doesn't ask it again. Every answer continues. |

What stays the same in every layout, on purpose:

- **The questions and the monday columns they fill.** A layout only reorders.
- **The step numbers sent to monday.** Each screen reports its main-form step, so the
  Drop-off Step labels mean the same thing on every row. A screen asked before step 1
  is step 0, which writes no label (no row exists that early).

What a layout carries with it:

- **Form Layout** column on monday says which one the patient used.
- **Saved progress** is kept per layout on the device, so the insulin form never
  reopens a half-finished main form (or the other way round).
- **Resume links** remember the layout: texted links, drop-off nudges and the
  card-upload "finish your form" button all reopen the order the patient started in.

**Adding a layout** is a change in both repos: `LAYOUTS` in `intake-form.html`, and
`FORM_LAYOUTS` (plus `CGM_REASON_FIRST_LAYOUTS` if it moves that question) in
`server/src/labelMap.js`. The backend ignores a layout name it doesn't know.

## Ad landing pages

Each ad lands on its own page on medicallymodern.com that embeds the form with its
layout, the same way `/intake-form/` does:

```html
<iframe src="https://medically-modern.github.io/mm-track-widget/intake-form.html?v=insulin"
        style="width:100%;height:100%;border:none;" allow="geolocation; microphone; camera" allowfullscreen></iframe>
```

**Ad tags.** The form sends `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`,
`utm_content` and `gclid` to the backend, which writes them to the UTM Source / Medium /
Campaign / Term / Content and GCLID columns. They are only ever written, never blanked, so
a patient who comes back through a texted link keeps the ad that first brought them.
Google puts these tags on the *landing page's* URL, not the iframe's, so the page has to
hand them over — that is the first job of the listener below. (Tags written into the
iframe `src` itself win over the page's.)

**Google Ads conversions.** The Google tag runs on the landing page and can't see inside
the iframe. The form posts two messages to the page, and the listener turns them into
dataLayer events: `mm_intake_lead` when a patient first gets past the contact step, and
`mm_intake_submit` when the form is filed — each once, with `form_layout` set. Messages
never carry an answer or anything that identifies the patient.

### The listener (install once, in Google Tag Manager)

Container `GTM-NXS6X6L2` on medicallymodern.com: add a **Custom HTML** tag with the code
below, firing on **All Pages** (Initialization). Until it is installed, the form still
works — the ad-tag hand-off and the conversion events just don't happen.

```html
<script>
/* Medically Modern intake form <-> this page (mm-track-widget README). */
(function(){
  var FORM_ORIGIN = 'https://medically-modern.github.io';
  window.addEventListener('message', function(e){
    if(e.origin !== FORM_ORIGIN || !e.data || typeof e.data.type !== 'string') return;
    if(e.data.type === 'mm-intake:hello'){
      e.source.postMessage({ type: 'mm-intake:page', search: location.search }, FORM_ORIGIN);
    } else if(e.data.type === 'mm-intake:lead' || e.data.type === 'mm-intake:submit'){
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        event: e.data.type === 'mm-intake:lead' ? 'mm_intake_lead' : 'mm_intake_submit',
        form_layout: e.data.layout || ''
      });
    }
  });
})();
</script>
```

Then, in GTM, a **Custom Event** trigger on `mm_intake_lead` and/or `mm_intake_submit`
fires the Google Ads conversion tag. A Data Layer Variable on `form_layout` lets a
conversion be split by layout.
