# Python, One Day at a Time

A tiny, friendly site that teaches **one Python concept a day** — 30 days, one idea at a
time — with real code you run yourself, right in the browser. A daily WhatsApp nudge
reminds the learner to open the site and either *warm up* with yesterday's idea or *learn
today's concept*.

## What it is

The entire app is a single, self-contained **`index.html`** — plain HTML, CSS, and vanilla
JS, no build step and no framework. You can:

- **Open it locally** — double-click `index.html` (most things work offline; the code
  runner needs a network connection, see below), or
- **Serve it on GitHub Pages** — the recommended way to use it day to day.

Progress (your name, completed days, and streak) is saved in your browser via
`localStorage`, so it survives refreshes. A **Reset progress** link on each screen clears it.

## Deploy to GitHub Pages

1. Push this repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**.
4. Select branch **`main`** and folder **`/ (root)`**, then **Save**.
5. After a minute, your site is live at `https://<your-username>.github.io/<repo-name>/`.

## The daily WhatsApp reminder

A scheduled GitHub Action — [`.github/workflows/daily-reminder.yml`](.github/workflows/daily-reminder.yml) —
sends one WhatsApp message a day through **[CallMeBot](https://www.callmebot.com/)** (a free
service). It also supports a manual run for testing and a light monthly keep-alive commit so
GitHub doesn't auto-disable the schedule after 60 days of repo inactivity.

### One-time setup

First, on the learner's phone, do the free **CallMeBot WhatsApp activation** (send the
one-time message they describe to get an API key). The free tier only delivers to the number
that authorised it, so this must be done on the learner's own phone.

Then, in the repo, go to **Settings → Secrets and variables → Actions** and add:

| Type         | Name                | Value                                                   |
| ------------ | ------------------- | ------------------------------------------------------- |
| **Secret**   | `CALLMEBOT_APIKEY`  | The API key CallMeBot gave the learner                  |
| **Secret**   | `LEARNER_PHONE`     | The learner's number with country code, e.g. `+1404...` |
| **Variable** | `LEARNER_NAME`      | The learner's first name                                |
| **Variable** | `SITE_URL`          | The live GitHub Pages URL (fill in after Pages is on)   |

To test it right away: **Actions → Daily Python reminder → Run workflow**.

### Changing the reminder time

The schedule lives in the `cron` line of the workflow:

```yaml
- cron: "0 22 * * *"
```

Cron is **always in UTC** and does not follow daylight saving. `0 22 * * *` is 22:00 UTC,
which is about 6:00 PM in Atlanta during summer (EDT) and 5:00 PM in winter (EST). Change the
hour to suit the learner's timezone. (GitHub's scheduled runs can be delayed a few minutes at
busy times — fine for a daily nudge.)

## Changing the lessons

All 30 lessons live in the `LESSONS` array near the top of the `<script>` in `index.html`.
Each lesson is one object (`day`, `emoji`, `title`, `idea`, `example`, `practice`, `review`).
Edit the text, add days, or tweak the practice — it's just data.

## A note on the code runner

The in-browser Python is powered by **[Pyodide](https://pyodide.org/)** (real CPython
compiled to WebAssembly), loaded from the official jsDelivr CDN only on the first time you
press **Run**. If Python can't load (e.g. you're offline or previewing inside an app that
blocks it), the rest of the page still works and shows a friendly note — the runner comes to
life once the site is served over the web.
