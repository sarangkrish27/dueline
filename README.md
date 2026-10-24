# Dueline

A small, quiet website that lists upcoming TDS (Tools in Data Science, IIT Madras BS) deadlines in date order: graded assignments, take-home projects, and each question's phase.

> Not an official IIT Madras or TDS website. Always confirm deadlines on the course pages.

## Why

TDS deadlines are phased. Some questions of a graded assignment close this Sunday, the rest next week, and a single project question can have two phases a few days apart. They are easy to lose track of. Dueline puts them in one list, soonest first, with a "Next up" line and a countdown.

## How it works

- Vue 3 + Vite, no backend and no database.
- Deadlines are read in the browser from one XML file, `public/deadlines.xml`.
- Hosted on Vercel; every push to GitHub redeploys the site.
- All times are IST.

## Editing `deadlines.xml`

One line per deadline:

```xml
<deadline date="2026-10-25" time="23:59" kind="assignment" title="GA3" part="Questions 1–6" />
```

| Attribute | Meaning |
| --- | --- |
| `date` | `YYYY-MM-DD`, in IST |
| `time` | `HH:MM`, 24-hour, in IST. Optional, defaults to `23:59`. Note that `11:59` is 11:59 AM |
| `kind` | `assignment` or `project` |
| `title` | e.g. `GA3`, `Project 1` |
| `part` | which questions or phase (optional) |

Commit and push, and Vercel redeploys. Entries with the same title, date and time are shown once. An entry with an invalid date or time is skipped, and a warning naming it appears in the browser console.

## Run locally

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # production build in dist/
```

## Found an error?

Send a gchat to 24f2005600@ds.study.iitm.ac.in
