# Karpathy → Sheets · Cowork Handoff

You're taking over an interactive-courseware build: **Andrej Karpathy's "Neural Networks: Zero to Hero" Lecture 1 (micrograd), rebuilt as a collaborative quiz inside Google Sheets**, with diagrams authored in Google Slides and embedded as live images. Everything below is enough to run, screenshot, and iterate. Read `PROJECT.md` for deep background; this file is the operational quickstart.

> 🔑 **Secrets are NOT in this file.** The service-account key lives locally at `~/.config/gcp/sheets-sa.json` (perms 600). GitHub auth is via `gh auth token`. Transfer the key file securely between machines — never commit it or paste it into a public doc.

---

## 0. Mission & current focus
- **What:** a per-tab quiz card UI in one Google Sheet. Each tab = one question. A two-person **answer-relay** threads the cards (your answer to one step is your partner's input to the next), teaching the forward pass + chain rule by hand.
- **Right now:** we just added a **MOTIVATION section** to the card template (`build_17_template.py` → tab `TEMPLATE ✦`) — a "WHY THIS STEP" panel + a `▶ watch @ MM:SS` deep-link into the lecture. **Your job: clean it up visually, then apply it to all 8 side-A cards and build side B.**

## 1. IDs & access
| thing | value |
|---|---|
| Google Sheet ("Kaparthy") | `1EHkTbf_u_-Ja905jy3CGT8-dTJpxM661WGag9cbyWvY` |
| Google Slides deck | `1Q_fdiQsk1aGuhI_VY7LdvNjtQDO7p6iLj0w3VbyBDaQ` |
| Service account | `tunrout-sheet@gen-lang-client-0578503752.iam.gserviceaccount.com` (key: `~/.config/gcp/sheets-sa.json`) |
| APIs enabled (project `gen-lang-client-0578503752`) | Sheets, Slides. (Drive: SA has **no storage quota** — can't create/upload files.) |
| GitHub asset repo (public) | `pr0fessor-Dan/kaparthy-sheet-assets` — hosts slide PNGs |
| Lecture-1 video id | `VMj-3S1tku0`  (deep-link: `https://youtu.be/VMj-3S1tku0?t=<seconds>`) |
| Tab deep-link | `https://docs.google.com/spreadsheets/d/<SHEET>/edit#gid=<sheetId>` |

## 2. Environment
- macOS, Python 3.9, deps installed `--user`: `gspread`, `google-api-python-client`, `google-auth`, `openpyxl`.
- All build scripts live in `~/karpathy-zero-to-hero-sheets/sheets-port/`.
- Auth pattern (every script): `Credentials.from_service_account_file("~/.config/gcp/sheets-sa.json", scopes=["https://www.googleapis.com/auth/spreadsheets","…/presentations","…/drive"])`. The shared helper module is **`slide_pipeline.py`** (exposes `sp.sheets`, `sp.slides`, `sp.SHEET`, `sp.PRES`, `sp.KEY`, and the image pipeline funcs).
- Run a script: `cd ~/karpathy-zero-to-hero-sheets/sheets-port && python3 build_16_card_tabs.py` (warnings on stderr are harmless; filter with `2>&1 | grep -v -E "NotOpenSSL|warnings.warn|FutureWarning|api_core"`).

## 3. The image pipeline (Slides → Sheet)
`slide_pipeline.py`:
- `snapshot_and_host(slide_id, prefix)` → renders a slide to PNG (`getThumbnail`), pushes it to GitHub under a **unique timestamped filename**, returns the raw URL.
- Convert raw→CDN: `.replace("raw.githubusercontent.com/","cdn.jsdelivr.net/gh/").replace("/main/","@main/")`. **Embed the jsDelivr URL** (Google's IMAGE proxy fetches CDN reliably; raw-github lazy-fetches and looks stale).
- In a cell: `=IMAGE("<jsdelivr-url>", 1)` for fit-to-merged-cell, or `=IMAGE(url, 4, h, w)` for explicit px.

### IMAGE() gotchas (all learned the hard way — don't re-discover)
- URL must be **anonymously fetchable** (CDN/raw OK; Slides `contentUrl` is auth-gated + expires → fails).
- **Mode 1 renders fine inside a merged cell.** (An earlier "#REF in merged" note was situational.)
- A formula cell's value reads **blank whether or not the image loaded** → can't verify rendering via API; check visually (or read the local `/tmp/<prefix>.png` the pipeline downloads).
- **"Show formulas" (⌘+`)** hides all images & shows formula text — first thing to check if a sheet "looks broken."
- Caches: unique filename defeats CDN/path cache; append `?v=<n>` to defeat Google's proxy cache. Both = guaranteed-fresh.
- **Hidden helper cells (answer keys) must live OUTSIDE every merged range** — writes to a merged-away cell are silently dropped (this bug made every checker read ✗; key was moved to col `Q`).

## 4. The card template (what to perfect)
`build_17_template.py` builds tab **`TEMPLATE ✦`** — the canonical single-question card. Sections, top→bottom (left column cols C–H; right column cols J–O):
1. **Breadcrumb** — `NEURAL NETWORKS · ZERO TO HERO — Lecture 1` | `R9 · 7/8`
2. **◆ MOTIVATION panel** (blue tint, left accent rail): `WHY THIS STEP` label · why-text · `=HYPERLINK("https://youtu.be/VMj-3S1tku0?t=241","▶ watch @ 4:01")`
3. **Question** (big, goal-framed)
4. **Rule** (muted italic local-derivative hint)
5. **`📨 enter partner's …`** (relay cards) — yellow input feeds the checker
6. **Equation** (Roboto Mono; use `x1/w1`, NOT subscripts — mono lacks subscript glyphs)
7. **Answer** (yellow) + **feedback** (`✓/✗/🔒`, colored by conditional format) + **reveal** (`type SHOW` → shows the key cell)
8. Right: **DIAGRAM** (merged box, `IMAGE(…,1)`) + **SCRATCH** panel

Design tokens: gridlines OFF; near-mono palette + one blue accent + red/green/amber feedback; **set an explicit pixel height for every layout row** (unset rows collapse). Checker pattern: `=IF(partner="","🔒 enter partner value",IF(ans="","—",IF(ABS(ans-KEY)<0.02,"✓ correct","✗ try again")))`; KEY cell holds a formula referencing the partner cell.

## 5. 🔧 Visual cleanup TODO (from the latest screenshot — DO THESE FIRST)
On `TEMPLATE ✦` / `build_17_template.py`:
1. **Question overflows** — "How much does the rain move the verdict? ( ∂o/∂x₁ )" wraps and the `(∂o/∂x₁)` collides with the rule line below. Fix: taller/again-merged question row, or shorten, or drop the parenthetical to a second muted line. Ensure no overlap with row 9.
2. **Partner row is cramped/misaligned** — the `📨 enter partner's …` label (C12) sits on top of the equation (C13); the yellow input (F12) floats far right, detached. Fix: give the partner prompt its own clean row with the input immediately beside the label, clearly above the equation.
3. **Diagram is small with dead whitespace** — the neuron sits in a 16:9 canvas with big margins. Consider re-rendering a tighter diagram (crop / fill more), or enlarge the box.
4. General: align answer / feedback / reveal onto a tidy baseline; tighten vertical rhythm.

## 6. Curriculum (apply template to these — side A)
8 question cards. Per-card data (id · watch sec · WHY · reframed question) is in `PLAN-motivation-lecture1.md` (the per-card table) and the card list in `build_16_card_tabs.py`. The neuron: `o=tanh(x1·w1+x2·w2+b)`, x1=2,w1=−3,x2=0,w2=1,b=6.8814 → n=0.88, o=0.71, **∂o/∂x1=−1.5**. Relay spine R1–R9 + concept C1,C2 + verify P. Side A solves R1·R3·R5·R7·R9 + all C/P; side B solves R2·R4·R6·R8.

## 7. How to iterate / verify
- **Rebuild a card:** edit the card dict in `build_16_card_tabs.py` (or `build_17_template.py`) and re-run; it deletes+recreates the tab.
- **Verify a checker without eyes:** write test values via `sp.sheets…values().update`, read back the check cell via `…values().get` (see `build_16` test snippets in the chat history / `git log`). Confirm `🔒 → ✓ → ✗` transitions.
- **Screenshot/iterate:** open the tab deep-link (`#gid=`), screenshot, adjust the generator, re-run. The pipeline also saves each rendered slide to `/tmp/<prefix>.png` — you can `Read` that image directly to inspect diagrams.
- **Get a tab's gid:** `sp.sheets.spreadsheets().get(spreadsheetId=SHEET, fields="sheets(properties(title,sheetId))")`.

## 8. Decisions locked (don't relitigate)
two separate workbooks (currently tabs in one file as a stand-in) · strict gates + `type SHOW` reveal · per-step ✓ (no final code) · mix of computed + MCQ · framing = abstract+concrete (picnic) · Socratic (rule + hint) · one question per tab · diagram persistent · finish by verifying backprop against a numerical nudge.

## 9. File map
- `PROJECT.md` — full project state & history
- `PLAN-lecture1-collab-quiz.md` — the relay/quiz design (v2)
- `PLAN-motivation-lecture1.md` — the motivation/why-chain + per-card watch/why map
- `CONTENT-lecture1-relay.md` — full card content script
- `sheets-port/slide_pipeline.py` — auth + image pipeline (import this)
- `sheets-port/build_12_live.py` — live neuron renderer (`draw_live`) used for diagrams
- `sheets-port/build_16_card_tabs.py` — current per-tab card generator (side A)
- `sheets-port/build_17_template.py` — **the redesigned template (with motivation) — start here**
- `transcripts/01-micrograd.tsv` — timestamped transcript (for picking `watch` anchors)
