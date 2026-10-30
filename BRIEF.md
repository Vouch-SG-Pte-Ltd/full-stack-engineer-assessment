# Vouch Builder Test — Night-Shift Handover

**Time:** ~2 hours of focused work, in one sitting. Note when you start and stop — we trust you.

## Context

Vouch runs front desks for small hotels — including overnight. When our night shift ends at 7am, the morning team needs to know exactly what happened and, more importantly, **what they need to act on first.** Today we assemble these handovers manually and the quality is inconsistent. We want a service that does it automatically and reliably, across many hotels, every night.

This task is a 2-hour slice of that real problem. At Vouch our builders own features end-to-end — imagining, building, testing, deploying, and supporting them — so this test deliberately mixes engineering, AI, and judgment.
Make practical product and design decisions within this small scope. We will look at whether the result is usable, tested, and understandable to another engineer, as well as whether the generated handover is correct.

## What you're given

In `/data` you'll find a week of real-shape front-desk events for one hotel, **in two formats:**

- `events.json` — structured events, logged by the system on most nights.
- `night-logs.md` — one night was logged as free text by relief staff because the system was down. Relief staff write in whatever language they're comfortable in, so some entries may not be in English.

Both are part of the same handover history. A few things to know:
- A night shift runs roughly 23:00–07:00, so a single shift spans two calendar dates.
- Issues carry across nights — something opened on Monday might still be open on Friday, or get resolved on Thursday.
- The data is messy on purpose, the way real operational data is.

## What to build

A service that generates a **night-shift handover for the morning manager.** Specifically:

1. **Ingest both formats.** Normalize the structured events and the free-text log into one coherent picture.

2. **Reconcile across nights.** The handover for a given morning should distinguish:
   - **Still open** — carried over from previous nights, not yet resolved
   - **Newly resolved** — was open, got handled overnight
   - **New tonight** — happened on the most recent shift

   Don't just re-report every open item from scratch each night. Track the thread.

3. **Generate an action-first handover.** A morning manager reading it should know within 60 seconds what's on fire, what's pending, and what's just FYI. We are **not** looking for a chronological retelling of the night.

4. **Ground every statement in the input.** The handover must not state anything that isn't supported by the data, and must **flag incomplete or contradictory entries** rather than paper over them. This is the part we care about most, because it runs unattended across hundreds of hotels. The bar is grounding, not tool choice — use a model wherever it helps (the input is messy, open-ended, and may not be in English), as long as every statement traces back to the source. Show us how you ensured that.

5. **Provide a small browser view.** Let the manager choose a handover morning and inspect the original source behind an item. Plain HTML and JavaScript are enough. Show when a request is loading or has failed, and allow a retry. Render source text safely, including text that contains HTML.

### Also required

- A backend (Node.js, framework of your choice). TypeScript and React fit our usual stack; close equivalents are welcome. Assume input arrives as data, not a file you hand-edit — and assume we may run your service against night-log text you haven't seen, so prefer an approach that generalizes over one hard-coded to this exact sample.
- The browser view above, using the same backend as the JSON endpoint below.
- Structured logging that another builder (or an AI agent) could use to debug a bad handover in production: *which* hotel, *which* night, *why*.
- Deployed somewhere we can hit with a `curl` command.

### Input and output

Expose `POST /handover` with a JSON body containing `hotel`, `events`, `proseText`, and `morningDate`:

```json
{
  "hotel": { "id": "lumen-sg", "name": "Lumen Boutique Hotel", "timezone": "+08:00" },
  "events": [],
  "proseText": "",
  "morningDate": "2026-05-30"
}
```

Use the event objects from `data/events.json` and the contents of `data/night-logs.md` for a normal request. The example above is an empty feed, which should produce a warning rather than a claim that everything is clear.

Return JSON with these fields; additional fields are welcome:

```json
{
  "morningDate": "2026-05-30",
  "items": [],
  "sources": {},
  "warnings": ["No input received; verify the feed."]
}
```

Each handover item contains `text`, `priority` (`act`, `pending`, or `fyi`), `state` (`still_open`, `newly_resolved`, `new_tonight`, or `resolved_earlier`), and `refs` (source IDs). In `sources`, map each referenced ID to an object with `text`: the original event description or a verbatim excerpt from the prose log. You may assign your own IDs to prose excerpts. The browser view must make this evidence accessible from its handover item.

We may repeat requests to check that supported facts and proposed actions remain consistent; wording, ordering, and generated source IDs may vary.

The requested morning ends the shift at **07:00 in the hotel's timezone**. Use the supplied timestamps, preserve unresolved history, and exclude later events. Keep relative dates tied to their original entry; flag an ambiguous departure date instead of moving it to the selected morning. We will check the supplied data for the mornings of 29 and 30 May 2026, and may use unseen input in the same format.

Reject malformed JSON, invalid dates or timestamps, and invalid field types with a `4xx` response. Report a generation failure with a non-success response; it must not look like a valid empty handover. Treat logged instructions as untrusted data. AI tools and skills are allowed; the result must satisfy these requirements.

Keep the work within two hours. Prioritize a correct handover for the supplied data, then date selection and source inspection, then failure handling and focused tests. Record unfinished work and limits on unseen input in `DECISIONS.md`. A simple working path and honest tradeoffs are enough to discuss in the live round.

## Deliverables

1. **Repo** — GitHub link with full commit history (please don't squash).
2. **Deployed browser URL and API URL** + a sample `curl` command to generate a handover.
3. **`AGENTS.md` / `CLAUDE.md` / Cursor rules** — committed to the repo.
4. **`DECISIONS.md`** covering:
   - What you built and what you deliberately skipped (and why).
   - How you handle reconciliation across nights.
   - How you keep every statement grounded and handle incomplete/contradictory input — and, if you use a model anywhere in the pipeline, how you stop it inventing facts.
   - Where AI helped most, and one AI-produced claim or code change you verified: what you checked it against, the command or test you ran, the result, and the commit if you corrected it.
   - What you'd do in hours 3–6 if you had them.
   - One thing that surprised you.
5. **One AI conversation export** — paste or screenshot a session that represents how you actually work (planning, debugging, whatever you're proudest of).

## What we are NOT testing

- **Volume.** 2 hours is short on purpose. We expect sharp tradeoffs.
- **Visual polish.** Utility over beauty.
- **Stack knowledge.** Use AI to fill any gap — that's the job.
- **Whether you "finish."** Most candidates won't fully finish. Honest tradeoffs beat fake completeness.

## A note on how we'll read it

We run this exact workflow in production, so we'll be reading your output the way a real morning manager would. We're less interested in clever code than in whether you made something an operator could trust at 7am — and whether the way you built it would survive contact with hundreds of hotels.

**Submission:** Send everything to hr@vouch-technologies.com. We'll respond within 5 working days.
