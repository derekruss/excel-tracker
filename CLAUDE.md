# Excel High School Family Tracker

## Context (from owner, carried forward)

Purpose: Tracks grades and assignments for Logan and Peyton at Excel High School, which uses the Learn Stage platform. Goal is fully automated, multi-device, no manual data entry.

Architecture:
- Frontend: single `index.html` (Firebase + localStorage), repo github.com/derekruss/excel-tracker, branch `main`, served by GitHub Pages at school.derkinc.io (Cloudflare CNAME to derekruss.github.io, proxy off)
- Database: Firebase Realtime DB at https://school-tracker-7e84c-default-rtdb.firebaseio.com, grade data at `/grades/logan` and `/grades/peyton`
- Sync: Node.js + Puppeteer script `sync.js`, run by Windows Task Scheduler job `LearnStageNightlySync` nightly at 2 AM. Logs in as parent (Ashley), scrapes each student's gradebook, pushes numeric grades and assignment data to Firebase. A copy also exists as a Gist at gist.github.com/derekruss
- Home page: intended 6-card grid (Logan's Grades, Peyton's Grades, Upcoming Assignments, Exam Countdowns, Logan's Today, Peyton's Today). Sidebar student buttons are hardcoded in HTML on purpose so they always render
- Assignment due dates and exam dates are entered manually because Learn Stage doesn't expose dates in scraped data

Hard-won scraper rules (do NOT regress these):
- Parent login: identity.learnstage.com. SIS: live.learnstage.com. LMS courses: live.lms.learnstage.com/exceled/excelhighschool/sis/lms/courses
- Course cards selector is `.box-round.box-bg.cursor-pointer` (NOT `.course_instance`, that's the grid wrapper)
- Gradebook URL pattern: `/sis/lms/course/details/gradebook/{courseTypeId}/{enrollCourseId}`
- Overall weighted course grade is in the LAST `thead th`, not tbody rows
- Only one accordion section can be expanded at a time; expand sequentially
- React LMS pages need `waitUntil: 'networkidle2'` with 60s timeout; `domcontentloaded` leaves cards unrendered
- Detect card navigation with `waitForFunction(() => location.href.includes('/course/details/'), { timeout: 8000 })`
- Use a fresh `puppeteer.launch()` per student; logout URLs and cookie clearing both failed to isolate sessions
- Grades MUST be stored as floats via `parseFloat()`. Storing strings like `"97.61%"` made every grade render as F

Known open issue: a family member got a Firebase access error even after rules were confirmed open. Unresolved.

## Working rules

- Only make changes directly requested. No refactors, new files, new dependencies, or framework migrations.
- Ask before: deleting any file, adding a dependency, changing Firebase data structure or rules, editing the scheduled task, or pushing to `main`.
- Never run `sync.js` against live Learn Stage without asking first.
- Never put passwords, tokens, or Firebase secrets in this file or in commits.
- Pushing to `main` deploys to school.derkinc.io immediately (GitHub Pages).

## Locations (verified 2026-10-06)

| What | Path |
|---|---|
| Repo clone | `C:\Users\derek\.openclaw\workspace\excel-tracker` |
| Live frontend | `index.html` at repo root (byte-identical to school.derkinc.io, ignoring CRLF) |
| Sync project (NOT in repo) | `C:\Users\derek\Downloads\learnstage-sync\learnstage-sync\` (`sync.js`, `package.json`, `run.bat`, `setup.bat`, `README.md`, `node_modules`) |
| Scheduled task action | `"C:\Users\derek\Downloads\learnstage-sync\learnstage-sync\run.bat"` |

Other copies that are NOT the live code (don't edit by mistake):
- `tracker/index.html` in the repo: an older variant (served at school.derkinc.io/tracker/), lacks the module progress bar.
- `C:\Users\derek\.openclaw\workspace\tracker\index.html`: same content as repo `tracker/index.html`.
- `C:\Users\derek\Downloads\sync.js`: older sync version (Apr 11).
- `C:\Users\derek\Downloads\index.html` / `index_1.html` (Mar 19): the only copies with the 6-card home grid (Logan's/Peyton's Today). Never committed.
- Gist `df0b9a87…` "LearnStage Scraper v10 — Firebase Push Edition" (`learnstage-scraper-v10.js`), secret gist, not the same as the current `sync.js`.

## Reality vs. intended design (found 2026-10-06)

Frontend (`index.html`):
- Home grid on the live site has **4 cards**, not 6 (no "Today" cards).
- Firebase access is plain REST `fetch` against the RTDB `.json` endpoints, with no Firebase SDK and no auth. Rules must allow unauthenticated read+write for it to work. `/.settings/rules.json` returns 401 without an admin secret.
- Reads on load: `/grades`, `/planner_logan`, `/planner_peyton`.
- Writes: `saveGrades()` **PUTs the entire `/grades` node** (both students) on any checkbox toggle or import. `savePlanner()` PUTs `/planner_{logan|peyton}`.
- localStorage keys: `ehs_grades_v2`, `ehs_planner_v2` (Logan), `ehs_planner_peyton_v2`.
- **Clobber bug:** on a device with empty localStorage, `loadAll()` → `seedAll()` → `saveGrades()` PUTs demo data over `/grades` (and the Logan planner) *before* `initFirebase()` reads. Any new device or browser can wipe real data. As of 2026-10-06, Firebase `/grades` contains only `logan` with demo data (English Literature / Algebra II); `/grades/peyton` does not exist.
- Unused top-level Firebase keys also exist: `/students`, `/planner`.

Sync (`sync.js`) vs. frontend contract:
- Credentials: parent email and password are **hardcoded in `CONFIG` in `sync.js`** (no env vars or `.env`). This needs moving out of source eventually.
- Writes: one PUT per student to `/grades/{logan|peyton}` (replaces that student's node, so frontend checkmarks there are lost each night).
- `currentGrade` is pushed as a **string** like `"97.61%"`. There is no `parseFloat`, which violates the rule above. The frontend `parseFloat`s it for averages but the course card uses the raw value (`g>=90` on a string) → renders F.
- Field-name mismatches: sync writes `possiblePoints` but the frontend reads `outOf`. Sync writes `gradePercent` (always null) but the frontend reads `percentage`. Sync writes `lastUpdated` but the frontend reads `scrapedAt` for "Last synced". Sync never writes `subject`, `code`, `moduleProgress`, or `status`.
- No accordion expansion code exists in current `sync.js`; it reads `tbody tr` directly.
- Gradebook URL is built as `page.url().replace('/home/', '/gradebook/')`, and that `goto` uses a 20s timeout (courses page uses 60s).
- Course card selector, `waitForFunction` nav detection, fresh browser per student, and last-`thead th` grade all match the rules.

Data shapes as actually written:
```
/grades/{student}  (from sync.js)
  { studentName: "Logan Russ", lastUpdated: ISO, totalAssignments, completedAssignments,
    courses: [ { name, currentGrade: "97.61%" | null, totalAssignments, completedAssignments,
                 sections: [ { name, assignments: [
                   { name, receivedPoints: number|null, possiblePoints: number|null,
                     gradePercent: null, completed: bool } ] } ] } ] }

/grades/{student}  (shape the frontend expects / demo seed)
  { studentName, scrapedAt?, courses: [ { name, code, subject, currentGrade: number, status,
      moduleProgress: "4/8", sections: [ { name, assignments: [
        { name, receivedPoints, outOf, percentage: "90%", completed } ] } ] } ] }

/planner_{student}
  { assignments: [ { id, name, subject, type, dueDate: "YYYY-MM-DD", completed } ],
    exams:       [ { id, name, subject, date: "YYYY-MM-DD", notes } ],
    classes:     [ { id, name, subject, day: 0-4, startHour, startMin, endHour, endMin } ] }
```

Scheduled task `LearnStageNightlySync`:
- Daily 2:00 AM, runs as `derek`, **Logon Mode: Interactive only** (runs only while derek is logged in), "Stop On Battery Mode".
- Runs `run.bat`, which ends with `pause > nul` and writes **no log file**. Output goes to a console window, so there's no record of what each run did. "Last Result: 0" reflects `run.bat`, not sync success.
- Created by `setup.bat` (which deletes and recreates the task).

Environment: Node v22, puppeteer ^22 installed in the sync folder's `node_modules`.
