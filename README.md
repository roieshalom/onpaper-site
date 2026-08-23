# onpaper-site

Standalone front-end for **OnPaper** — the job-fit gap finder — served on
GitHub Pages at **https://onpaper.fit**.

- Single static `index.html` (self-contained: markup, styles, script inline).
- Calls the private backend at `https://onpaper-api.vercel.app/api`
  (`/analyze` + `/redeem`), which holds the Anthropic key and the prompt.
- The same tool also lives as the **OnPaper tab** in the `job-triage-portal`
  site (triage.roiesh.com); this repo is the dedicated home. Keep the two
  front-ends in sync until OnPaper is split off for good.

`onpaper.fit` must stay in the backend's CORS allowlist (`onpaper-api/lib/http.js`).
