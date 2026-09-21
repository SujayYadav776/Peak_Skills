---
name: watchdog
description: Autonomously completes courses in the browser on course/certification platforms — LinkedIn Learning, Udemy, Coursera, edX/Harvard CS50, and similar — by playing every video lesson to the end at the maximum speed the player accepts (auto-probed, 16× down to 2×), skipping assessments only when the certificate allows it, chaining lesson→lesson with no manual babysitting, and verifying completion from server state. Use when the user shares or attaches a course tab/URL and asks to complete/watch/finish it.
version: 2.1.0
---

# Course Watchdog (multi-platform)

Identify the platform from the URL, load its adapter card below, then run the universal loop: **inventory lessons → play each pending lesson to natural `ended` at the maximum probed speed → auto-advance → verify from server state**. Never stop between lessons, never ask per-lesson confirmation, never re-watch completed lessons, resume partially-watched ones. Stop only when all lessons are verified complete or a genuine platform block occurs (then report exactly where you got stuck). For an unlisted platform, run the **Discovery protocol** first.

## Universal hard rules

- Play at the **maximum speed the player accepts** — don't hardcode 2×. The supervisor probes candidates `[16,8,6,4,3,2.5,2]`, keeps the highest `playbackRate` the video element retains, stores it in `window.__rate`, and hammers it every 600 ms (players reset rate to 1× on load and platform code may clamp it back down). Re-arm on every page load.
- **Completion-rate safety**: some platforms only register completion up to a cap (often 2×–4× UI max even when the element accepts more). If verification (Phase 3) shows a lesson finished at high rate but NOT marked complete/viewed, add it to a `retryQueue` and replay those lessons at 2×. Once a run reveals the platform's real completion-capable max, note it in that adapter card.
- If `playbackRate` >1× causes constant buffering stalls (readyState < 3 for > 15 s at `paused:false`), step the probe list down one notch and re-arm.
- Completion = natural `ended` event (or the platform's own ≥90%-watched marker). Do NOT seek to the end.
- The supervisor JS timer dies on every navigation → re-install it on EVERY fresh lesson page. Make the installer idempotent.
- In-page JS waits must stay **≤ 22 s** — the browser tool's V2 timeout fires around 25–30 s. Longer gaps: Bash `sleep` (retry once if it exits 1), then a status read.
- Every status read must echo `location.pathname` so lesson transitions are unambiguous.
- Skip assessments (quizzes/exams/ungraded checks) ONLY when the platform's certificate/progress doesn't need them — check the certificate caveat in the adapter card. If required, ask the user once: skip assessments (no cert) vs attempt them.
- Constraints: no profile changes, no posts/comments/messages, no purchases, no subscription changes, don't touch unrelated tabs.

## Universal supervisor (same-document `<video>` players)

Idempotent installer, parameterized with the lesson URL base and the NEXT lesson's full URL (`N=''` on the final lesson → no auto-advance):

```js
(function(){var N='<next-lesson-full-url>';
var v=document.querySelector('video');if(!v)return 'novid';
try{Object.defineProperty(document,'visibilityState',{get:function(){return 'visible'}});
Object.defineProperty(document,'hidden',{get:function(){return false}});
Object.defineProperty(document,'hasFocus',{value:function(){return true}});}catch(e){}
v.pause=function(){};                     // neutralize hidden-tab pause guards
if(!window.__rate){var cand=[16,8,6,4,3,2.5,2],ok=1;   // probe: highest playbackRate the element retains
  cand.forEach(function(c){if(ok>=c)return;try{v.playbackRate=c;if(Math.abs(v.playbackRate-c)<0.01)ok=c}catch(e){}});
  window.__rate=ok>1?ok:2;}
if(window.__bTimer)clearInterval(window.__bTimer);
window.__bTimer=setInterval(function(){var vv=document.querySelector('video');if(!vv)return;
  vv.playbackRate=window.__rate;if(vv.paused&&!vv.ended)vv.play();
  if(vv.ended){clearInterval(window.__bTimer);window.__bTimer=null;if(N)setTimeout(function(){location.href=N},300)}},600);
v.playbackRate=window.__rate;v.play();
return JSON.stringify({armed:location.pathname.split('/').pop(),rate:window.__rate,t:+v.currentTime.toFixed(1),d:+v.duration.toFixed(1)})})()
```

- If `novid`: poll `document.querySelector('video') && v.readyState>0` every 700 ms for up to ~15 s, then arm.
- Why this is needed: most platforms (LinkedIn, Udemy, YouTube) pause playback whenever the tab is hidden or the window loses focus, via internal `checkPlayback` handlers — plain `play()` gets re-paused within ~1 s. Spoofing `visibilityState`/`hidden`/`hasFocus` + no-op'ing `pause()` + hammering `play()`/rate from a 600 ms interval is what makes the probed max rate hold in practice (LinkedIn v1 empirically held 2× → ≈2–2.5× effective; higher rates where the platform allows it).
- **OS foreground fallback** (needed for YouTube-based lectures and Chrome/Opera media suspension that JS spoofing can't beat): if playback stalls at `readyState:0`/`networkState:2` and only advances while you screenshot, the browser window lost real OS focus. Fix with a PowerShell user32 call that `ShowWindow(SW_RESTORE)` + `AttachThreadInput` + `SetForegroundWindow` on the window's hwnd — match by window TITLE (the user's browser may be Opera, so enumerating `chrome` processes finds nothing), then re-arm. Re-run if a later lesson stalls at `t≈0`.
- **Stall detection**: `paused:false` but `currentTime` frozen and `readyState<4` → keep calling play every 500 ms for ~30 s, else reload the lesson (most platforms resume near the stall).

## Universal monitoring & advance

- One bounded wait call per poll: `(function(){var v=document.querySelector('video');return new Promise(res=>{var i=setInterval(()=>{if(v.ended){clearInterval(i);res('ended')}},400);setTimeout(()=>{clearInterval(i);res(JSON.stringify({path:location.pathname,rate:window.__rate,t:+v.currentTime.toFixed(1),d:+v.duration.toFixed(1)}))},20000)})})()`
- `EXECUTION_ERROR: Inspected target navigated or closed` → the supervisor auto-advanced; run the ready-wait installer for the expected next lesson (verify `location.pathname` matches your queue position first).
- If a read shows `t≈d` with no navigation, set `location.href` to the next lesson manually.
- If the page landed on an unexpected lesson, don't fight it — lesson order is irrelevant for completion; requeue the skipped lesson and continue the chain from wherever you are.
- Interstitial/survey/modal blocking the player → click the minimal Next/Close control and resume.
- Assessment page appears by accident (no video) → leave immediately via URL to the next video lesson; never answer it.

## Adapter cards

### LinkedIn Learning — proven recipe (v1 hard-won)

- URL: lesson pages `https://www.linkedin.com/learning/<course-slug>/<lesson-slug>?u=<userid>` — grab the course slug and `u=` from the tab URL; `?u=` keeps the right account on shared machines. `resume=true` is the default (partially watched lessons continue).
- Inventory: click `.classroom-sidebar-toggle` (Contents), wait ~1.2 s, enumerate `li[data-toc-content-id]`. The `data-toc-content-id` URN classifies the item: `urn:li:learningApiVideo:...` = lesson, `urn:li:learningApiAssessment:...` = QUIZ → exclude and URL-hop across it. Lesson slug from its `<a href>`; status from item text: `(Viewed)` = done, `(In progress)` = resume, else pending. Returns `[]` → sidebar collapsed, click the toggle again.
- Player is same-document; the universal supervisor works verbatim.
- Cert caveat: certificate of completion fires on 100% **video** watched; chapter quizzes are NOT required → safe to skip.
- Verify (never skip): full `navigate` reload of any lesson page (fresh server-rendered sidebar), toggle sidebar, wait 1.5 s, count every non-`learningApiAssessment` `li[data-toc-content-id]` containing `(Viewed)` — expect `pendingCount === 0`. Anything not Viewed goes back in the queue even if you "saw" it end. Optionally confirm on `/learning/my-learning` that the course shows completed.

### Udemy

- URLs: course `https://www.udemy.com/course/<slug>/`; video lessons link to `.../lecture/<id>/` (or `?lecture_id=<id>`). Assessments to skip: `.../take-quiz/...`, `.../take-practice-test/...`, coding exercises/assignments. Inventory = expandable curriculum sidebar; collect every `a[href*="/lecture/"]` in DOM order; the sidebar's own progress text ("X of Y lectures", per-row check icon) is the state source — on first run, confirm one completed row's marker differs from a pending row's and record the exact selector.
- Player is same-document `<video>` → universal supervisor applies. Udemy pauses hidden tabs (defeated by the spoof) and may auto-advance on its own at the end; that's fine — the installer is idempotent, just re-arm on whatever lesson URL you land on and keep the queue straight.
- Cert caveat: Udemy completion = all lecture videos watched; quizzes are instructor add-ons and generally NOT required for the certificate → safe to skip.
- Verify: sidebar/lecture-header shows 100% (or "Course complete" panel). Don't declare success from the final video ending alone.

### Coursera

- URLs: video lessons `https://www.coursera.org/learn/<slug>/lecture/<id>`; graded quizzes/assignments are `.../assignment/<id>` — these **count toward the certificate and often have deadlines**. Do NOT silently skip them: warn the user that a Coursera certificate requires passing graded work, and ask whether to (a) just complete videos (progress <100%, no cert) or (b) stop and hand graded parts over.
- **Big gotcha**: the video player is a cross-origin iframe (`classplayer.cloud.coursera.org`), so top-document JS cannot reach its `<video>`. Fall back ladder:
  1. If the browser tool can target frames/iframes, inject the universal supervisor inside the player frame.
  2. Else grab the iframe's `src` and open that player URL directly in the same tab, run the supervisor there, and on `ended` navigate the top document to the next lesson URL yourself.
  3. Else UI automation: enable Coursera's "Autoplay next" toggle once, set the player's speed control to the highest option it offers and Play via coordinates/aria, then just poll the sidebar for completion markers between Bash sleeps, keeping OS foreground.
- Verify: sidebar lesson tiles turn into checkmarks; course progress % shown in the header reaches its video-only maximum.

### edX / Harvard CS50x

- edX courses (`edx.org/course-v1:...` or `/learn/...`): videos live in same-document or `vendor-frame` iframes; progress is only recorded while enrolled and logged in. **Cert caveat (important)**: edX verified certificates require passing graded assignments + payment — video-only completion will NOT yield a cert. Tell the user upfront; automate the video lessons only, and handle graded items per their instruction (same ask-user rule as Coursera).
- CS50 specifically: current CS50x lectures are **YouTube-hosted** (course site links out, or embeds YouTube). Treat as a YouTube player: universal supervisor + visibility spoof usually works, but YouTube throttles/suspends background media — expect to need the **OS foreground fallback**. Quizzes/problem sets and the CS50 `submit50`/a50 exercises are real coding work — impossible to "watch" into completion; report them as user-side requirements rather than faking them.
- Verify: per-video watched/complete markers on the platform page; on YouTube-only course sites there is usually NO completion tracking — instead verify you enumerated and ended every lecture in your inventory list, and say so honestly in the report.

### Any other platform — Discovery protocol (run before looping)

1. Classify URL patterns: which sidebar links are videos vs assessments (quiz/exam/assignment/ungraded-problem).
2. Confirm `<video>` is reachable from top-document JS; if it's inside an iframe, check `iframe.contentDocument` — cross-origin ⇒ use the Coursera fallback ladder.
3. Identify the completion state marker: complete exactly one short lesson naturally, reload, and diff the sidebar DOM to find what changes (check icon class, text like "Viewed"/"Completed", progress counter).
4. Test hidden-tab behavior: arm the supervisor, Bash `sleep 60`, status-read; if `currentTime` didn't advance, apply the OS foreground fallback.
5. Certificate requirements: determine whether skipping assessments blocks the certificate; if yes, ask the user before proceeding.
6. After a successful run, patch this skill (`qwenwork_skill_manage` → patch) with a new/updated adapter card containing the verified selectors and gotchas, so the next run is cheap.

## Phase 1 — Inventory (once)

`tabs_context` → tabId + platform + course URL. Apply the adapter card's inventory recipe to build the ordered queue of pending video lessons (exclude assessments per the cert caveat). Wall-clock estimate ≈ Σ remaining durations ÷ probed rate (fall back to ÷4 until the first probe reports `window.__rate`). Report the queue size and estimate once, then go quiet.

## Phase 2 — Per-lesson cycle

Navigate (adapter URL pattern) → arm universal supervisor (or fallback ladder) → bounded monitor → auto-advance. Follow the universal monitoring & advance rules above.

## Phase 3 — Verify (never skip)

Reload from the server (fresh sidebar), recount completed vs pending via the adapter's state marker, requeue-and-replay anything not marked complete even if you "saw" it end, then check the course-level progress/certificate surface. Never declare success from the final video ending alone.

## Reporting

Quietly loop; no per-lesson narration. End with a short summary: platform, lessons completed, assessments skipped (and whether that forfeits the certificate), wall-clock, verification counts (e.g. "50/50 lessons Viewed, 6 chapter quizzes skipped by URL jump"). If truly stuck, state the exact lesson, symptom, and last successful step.
