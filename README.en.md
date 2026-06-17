# clean-transcript-to-report — Use Case

🌐 [Tiếng Việt](README.md) · **English**

A single web app (one container) that turns **raw meeting / interview transcripts** into
**structured HTML reports**. Two steps in one pipeline, sharing a single LLM endpoint:

1. **Clean** — clean up a raw transcript (`.vtt` from Teams / `.docx` / `.txt`).
2. **Analyze** — turn the cleaned transcript into a report of type `cs` / `ux` / `meeting`.

> This document focuses on **what it's for** and **how to use it**.
> For architecture, the technical pipeline, and deployment, see `docs/01-agent-README.md` and `DEPLOY.md`.

---

## 1. The problem

After a meeting, a user interview (UX research), or a customer support session, you usually end up
with a raw transcript — full of timestamps, repeated speaker names, filler words ("um", "uh"),
half-finished sentences, and recording-tool noise. Reading it by hand to extract insights is slow
and inconsistent.

This agent automates the two most tedious parts:

- **Cleaning** the transcript into something readable while preserving the speaker's intent
  (favoring *Precision > Recall*).
- **Analyzing** the cleaned transcript into a structured report, tightly anchored to the
  **research objectives** the user defines.

---

## 2. Who uses it & when

| Role | Situation | Report type |
|---|---|---|
| **UX researcher** | After a user interview, needs pain points / insights tied to research questions | `ux` |
| **CS / operations** | After a support session, needs a summary of issues + resolution direction | `cs` |
| **PM / lead / note-taker** | After a meeting, needs minutes: decisions, action items, owners | `meeting` |

Common thread: you have a recording/transcript and need a clean, structured report — **fast**.

---

## 3. Usage flow (2-step wizard)

Open the endpoint → a single-page wizard with two steps:

**Step 1 — Clean**
1. Upload `.vtt` / `.docx` / `.txt` (parsed right in the browser).
2. Toggle the 9 cleaning rules on/off (+ add a custom rule if needed).
3. Click clean → the LLM processes it via a server-side proxy (`POST /api/chat`).
4. **Review / hand-edit** the cleaned transcript before moving to step 2.

**Step 2 — Analyze**
1. Choose the report type: `cs` / `ux` / `meeting`.
2. Enter the **objectives** (goals / research questions) — required.
3. Click analyze → receive an **HTML report** rendered from validated JSON.

> Each analysis takes **exactly 1 cleaned transcript**. There is a minimum length requirement
> (ux ≥ 500 words · cs ≥ 300 words · meeting ≥ 500 words), checked **before** any LLM call to avoid wasting quota.

---

## 4. Input / output

| | Content |
|---|---|
| **Input** | 1 transcript file (`.vtt` / `.docx` / `.txt`) + report type + objectives |
| **Intermediate** | Cleaned transcript (user can review/edit) |
| **Output** | Structured HTML report by type (`cs` / `ux` / `meeting`) |

---

## 5. Two ways to call it

- **UI (recommended):** open `https://<ENDPOINT>/` → use the Step 1 → Step 2 wizard.
- **Direct API (skip Clean):** if you already have a clean transcript, call directly:

```bash
curl -s -X POST https://<ENDPOINT>/invocations \
  -H 'Content-Type: application/json' \
  -d '{"type":"meeting","mode":"single","transcripts":[{"participant":"x","text":"..."}],"objectives":"RQ1 ..."}'
```

---

## 6. Limitations (current scope)

- **Single-file** only (Route A). The entire Route B (multi-file ladder) has been removed from this agent.
- **Objectives are required** for the Analyze step — without them, the gate blocks the request.

---

## 7. Quick local run

```bash
cd src
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # fill in real LLM_API_KEY / LLM_BASE_URL / LLM_MODEL
python -m app.server            # http://localhost:8080
```

---

## 8. Related docs

| File | Content |
|---|---|
| `docs/01-agent-README.md` | Architecture, analysis pipeline, detailed run/deploy |
| `docs/02-clean-pipeline-9-rules.md` | Spec for the 9 cleaning rules (behind Step 1) |
| `docs/prompts/{cs,ux,meeting}.md` | Brain prompt for each report type |
| `HANDOVER.md` | Handover + security |
| `DEPLOY.md` | Build/push image + create GreenNode AgentBase runtime |
