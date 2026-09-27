# mail_label_draft

Labels Gmail on a schedule, and writes a draft reply for the handful of emails a person is actually waiting on. The
mail domain's only scheduled workflow. ⟦REVIEW src: workflow `mail_label_draft`, id `tCzqvauu1uCOzt8O`, read
2026-09-27⟧

**It labels and drafts. It never archives, deletes, or sends.** A draft cannot leave by accident, and nothing is ever
hidden from the inbox. ⟦REVIEW⟧

## Why it exists

An inbox of 6,000-plus messages has no shape. Gmail's own tabs sort by sender behaviour, not by what the mail is *to
you* — a rejection and a job digest both land in Updates. The useful question is "what kind of thing is this, and does
someone want an answer", and both halves are answerable locally. ⟦REVIEW src: Gmail account state, 2026-09-26⟧

## Flow

```mermaid
flowchart TD
    T1["<b>Schedule Trigger</b><br/>every 30 min"]
    T2["When clicking 'Execute workflow'<br/>manual, for testing"]
    H["<b>history_lookup</b><br/>Gmail · q: label:sorted · 100"]
    M["<b>build_sender_map</b><br/>sender → usual category"]
    F["<b>fetch_mail</b><br/>Gmail · q: -label:sorted in:inbox newer_than:7d · 25"]
    B["<b>trim_body</b><br/>strip URLs · 700 chars · attach senderHint"]
    C["<b>mail_categorizer</b><br/>sub-workflow · returns category + labelId"]
    D["<b>dedupe_and_flag</b><br/>one per id · build label list"]
    A["<b>apply_labels</b><br/>Gmail · Add Label"]
    R["<b>filter_repliable</b><br/>category gate + sender address"]
    K["<b>reply_check</b><br/>Text Classifier · needs_reply?"]
    W["<b>write_draft</b><br/>Basic LLM Chain"]
    O["<b>ollama_reply</b><br/>qwen3:8b · thinking off · temp 0.1"]
    G["<b>create_draft</b><br/>Gmail · Draft → Create"]
    T1 --> H
    T2 --> H
    H --> M --> F --> B --> C --> D
    D --> A
    D --> R --> K --> W --> G
    K -.-> O
    W -.-> O
    style C fill:#e7f1ff,stroke:#0d6efd
    style A fill:#d4edda,stroke:#28a745
    style G fill:#d4edda,stroke:#28a745
```

⟦REVIEW src: workflow `mail_label_draft`, 14 nodes⟧

Published, Schedule Trigger every 30 minutes. The manual trigger sits alongside it for testing and for deliberate
backlog sweeps. ⟦REVIEW src: `active: true`, `minutesInterval: 30`⟧

Two branches leave `dedupe_and_flag`. The labelling branch handles every email; the draft branch handles the one or
two that survive `filter_repliable`. They run independently. ⟦REVIEW⟧

## The labelling branch

### `history_lookup` — Gmail, Get Many

```
q:        label:sorted
limit:    100
Simplify: on
```

Fetches mail already labelled, as raw material for the sender map. ⟦REVIEW⟧

**It sits on the trigger, not on the mail.** A Gmail node runs once per input item; the trigger emits exactly one, so
this is one API call per run rather than one per email. ⟦REVIEW⟧

**Simplify on here, unlike `fetch_mail`.** Only the sender and the labels are needed, and Simplify returns both while
keeping the payload small. ⟦REVIEW src: measured node output, 2026-09-26⟧

**No leading minus.** The exact inverse of `fetch_mail` — this wants what has already been handled. ⟦REVIEW⟧

### `build_sender_map` — Code

Counts what each sender usually gets, and emits **one item** holding the map. ⟦REVIEW⟧

```js
if (!l.id.startsWith('Label_') || l.name === 'sorted') continue;
```

**`Label_` separates your labels from Gmail's.** Every message carries `INBOX`, `UNREAD` and often a `CATEGORY_*`;
none are informative. User labels are the only ones with `Label_` ids, so one check does the filtering — and a label
added later needs no edit here. ⟦REVIEW⟧

```js
if (total >= 3 && n / total >= 0.8) senderMap[from] = { top, n, total };
```

**The threshold guards against self-reinforcement.** The map is built from labels this workflow applied, so a mistake
could teach itself. Three emails and 80% agreement means one bad label among good ones cannot flip a sender.
⟦REVIEW src: threshold taken from n8n template 11518⟧

**Correcting a label by hand is the training signal.** The map reads whatever is in Gmail, so a relabel propagates on
the next run. ⟦REVIEW⟧

**One item out** is what keeps `fetch_mail` to a single call downstream. ⟦REVIEW⟧

### `fetch_mail` — Gmail, Get Many

```
q:        -label:sorted in:inbox newer_than:7d
limit:    25
Simplify: off
```

Three clauses, three jobs. `-label:sorted` is the idempotency key — anything already handled is invisible, so Gmail
filters server-side and those messages are never fetched. `in:inbox` ignores mail already filed by hand.
`newer_than:7d` caps the blast radius so a run cannot touch years of history. ⟦REVIEW⟧

Gmail search excludes Spam and Trash by default. ⟦REVIEW src: Gmail search behaviour⟧

**Limit 25, not 100.** At ~16–18s per email a batch of 100 runs ~26 minutes and overruns the schedule. 25 finishes in
about 7 minutes and still covers 1,200 emails a day. ⟦REVIEW src: measured, 2026-09-26⟧

**Simplify off is required.** See [Measured findings](#measured-findings). ⟦REVIEW⟧

### `trim_body` — Code

Turns a 126 KB raw Gmail message into the few hundred characters the model reads. ⟦REVIEW⟧

- **Strips URLs.** A bulk email's plain-text body is largely tracking links; without this, 700 characters of "body" is
  600 characters of `click4.example.com/CL0/…`. ⟦REVIEW src: measured, 2026-09-26⟧
- **Collapses whitespace, truncates to 700 characters.** Long enough to reach the decisive sentence, short enough to
  keep an 8B model accurate. ⟦REVIEW⟧
- **Drops `html` and `textAsHtml`.** The large payload dies here. ⟦REVIEW⟧
- **Attaches `senderHint`** by looking the sender up in the map. Unknown senders get an empty string. ⟦REVIEW⟧

Output field names are deliberately `Subject` and `From`, matching Simplify's spelling, so downstream expressions read
the same either way. ⟦REVIEW⟧

### `mail_categorizer` — Execute Sub-workflow

```
Mode:                            Run once with all items
Wait For Sub-Workflow Completion: on
```

Sends the batch out and gets it back with `category` and `labelId` added. Detail in
[`mail_categorizer.md`](mail_categorizer.md). ⟦REVIEW⟧

**Mode matters.** On `Run once for each item` the sub-workflow starts 25 separate times — 25 model startups instead of
one batched run. ⟦REVIEW⟧

**The node stores `mappingMode: defineBelow` with an empty value object, and this is not a bug.** The sub-workflow's
trigger runs `Accept all data`, so n8n skips input mapping entirely and every field passes through. Verified by
inspecting what `dedupe_and_flag` receives. ⟦REVIEW src: measured, 2026-09-27⟧

### `dedupe_and_flag` — Code

Keeps one copy per message id, then builds `labelsToAdd` — always `Label_11` (`sorted`), plus the category label when
there is one. ⟦REVIEW⟧

`sorted` goes on every email including the undecided ones. That is what stops the next run re-reading them. ⟦REVIEW⟧

### `apply_labels` — Gmail, Add Label

```
Message ID:         {{ $json.id }}
Label Names or IDs: {{ $json.labelsToAdd }}
```

The multi-select label field accepts an expression returning an array, which lets one node cover all ten categories
instead of ten nodes. ⟦REVIEW src: n8n-mcp `get_node` for `nodes-base.gmail`, `labelIds` property⟧

Gmail's add-label call is idempotent — adding a label already present is a no-op, not an error. ⟦REVIEW⟧

## The draft branch

Three gates, cheapest first. Each one exists so the next is affordable. ⟦REVIEW⟧

### 1. `filter_repliable` — Code, no model

Two questions, both answerable from data already in hand. ⟦REVIEW⟧

**Is the category one that could ever need a reply?** Only `general`, `job_application` and `job_interview`. A bill or
a security alert already has a loud label; a second flag adds nothing. ⟦REVIEW⟧

**Is the sender a real mailbox?** The part before the `@` is checked against a list of words that mean nobody is
reading — `noreply`, `notifications`, `alerts`, `daemon` and about thirty more. ⟦REVIEW⟧

```js
const mailbox = (from.split('<').pop() ?? from).split('@')[0];
```

**Only the local part is searched, never the domain.** `careers@infosys.com` becomes `careers`. Matching the whole
string would find `info` and `sys` in the domain and silently drop a recruiter. ⟦REVIEW⟧

This cuts 25 emails to 0, 1 or 2. That reduction is what makes the two model calls below cheap enough to justify.
⟦REVIEW⟧

### 2. `reply_check` — Text Classifier

Two categories — `needs_reply` and `no_reply_needed` — plus an unwired Other branch. Only `needs_reply` continues.
⟦REVIEW⟧

**Why a model and not more rules.** Whether an email wants an answer is a question about intent, and keyword matching
gets it wrong in both directions: *"Please do not reply to this email"* contains `please`, while *"Send over your
notice period"* contains no question mark. Of 2,061 workflows in the community collection, not one attempts this with
rules. ⟦REVIEW src: `../n8n-workflow-ideas/workflows`, searched 2026-09-27⟧

**Six rules in the system prompt**, the last of which sets the failure direction: ⟦REVIEW src: workflow
`mail_label_draft`, reply_check⟧

```
- When genuinely unsure, choose no_reply_needed.
```

A missing draft costs nothing — the email is still labelled and sitting in the inbox. An unwanted draft costs
attention. ⟦REVIEW⟧

### 3. `write_draft` — Basic LLM Chain

**A Chain, not an Agent.** One email in, one block of text out. An Agent can choose a tool, choose the wrong one, or
loop; a Chain has no tools to reach for. Same principle as using a Text Classifier for routing. ⟦REVIEW⟧

**The rule that matters:** ⟦REVIEW⟧

```
You do not know Swapnil's availability, notice period, salary, address,
schedule or opinions. Never state any of them. Write a named placeholder
in square brackets instead, like [YOUR AVAILABILITY].
```

Without it, `qwen3:8b` invents. Measured on a test email asking for availability and notice period, the model
confidently answered *"Monday, Wednesday, or Thursday"* and *"my current notice period is 15 days"* — both fabricated,
both stated as fact. ⟦REVIEW src: measured, 2026-09-27⟧

**Rules alone did not fix it. Two worked examples did.** The same finding as `memory_agent`: rules describe the
outcome, an example gives the shape. After adding them the same email produced `[YOUR AVAILABILITY]` and
`[YOUR NOTICE PERIOD]`, with no preamble, no subject line and no markdown. ⟦REVIEW src: measured, 2026-09-27⟧

The examples are **summarised rather than quoted in full**, and one is deliberately not about jobs, so the model
copies the shape without dragging job framing into ordinary mail. ⟦REVIEW⟧

### 4. `create_draft` — Gmail, Draft → Create

```
Subject:            =Re: {{ $('reply_check').item.json.Subject }}
Message:            {{ $json.text }}
options.threadId:   {{ $('reply_check').item.json.threadId }}
options.sendTo:     {{ $('reply_check').item.json.From }}
```

**`threadId` is the difference between a useful draft and a lost one.** Without it the draft sits alone in Drafts,
disconnected from the email. With it, the draft appears inside the conversation, where you would look for it.
⟦REVIEW src: n8n community workflow 1479, Gmail AI auto-responder⟧

**`$('reply_check')` rather than `$json`.** `write_draft` outputs only `{ text: ... }` — the subject, thread and
sender were left behind, so the expression reaches back to the last node that still had them. ⟦REVIEW⟧

**Draft → Create only.** There is no send operation anywhere in this workflow. Safe by construction rather than by
rule, the same reasoning as the `:ro` mount on `briefing/`. ⟦REVIEW⟧

## Safety

The risk here is not a wrong label. It is a **hidden** email. ⟦REVIEW⟧

A mislabelled email is still in the inbox, still visible, still findable. An archived one is not. So every guard sits
on the archive step, and this workflow does not have one. ⟦REVIEW⟧

Rules agreed before building, still in force: ⟦REVIEW src: design decisions 2026-09-26⟧

1. **Labels and drafts only. No archiving at all.** Enable archiving per label once accuracy is known.
2. **Nothing under `jobs/*` is ever archived**, at any stage. A buried recruiter email costs far more than scrolling
   past job alerts.
3. **The archive set was cut** from `{jobs/rejected, jobs/alert, newsletter}` to `{newsletter}` only.
4. **A sender allowlist** of ATS domains — greenhouse, lever, workday, ashby, smartrecruiters, icims — will skip
   archiving regardless of category. Deterministic, so no model can talk it out of it.
5. **Never archive what Gmail already flagged** `IMPORTANT` or `STARRED`.
6. **Uncertainty is visible, never silent** — the Other branches on both classifiers.

**Everything is reversible.** `label:sorted` finds every message this workflow has touched; select all and remove
labels to undo it entirely. Drafts are deleted like any other draft. Nothing is destroyed. ⟦REVIEW⟧

**Drafts are never written for `jobs/*` beyond `job_application` and `job_interview`**, and never for a rejection — a
mediocre auto-reply to a recruiter costs a real opportunity. ⟦REVIEW src: PLAN.md Phase 4.1⟧

## Measured findings

### Gmail's `snippet` truncates before the point

Simplify-on returns no body, only Gmail's ~200-character `snippet`. For an Amazon rejection the snippet ended:

```
Hi Swapnil, Thank you for your application for the position of
Software Development Engineer I, IESP Merchant Tech (ID: …).
After careful consideration a
```

One word before *"decided to progress with other candidates"*. The model never saw the rejection, and what it did see
reads exactly like `job_application`. ⟦REVIEW src: measured, 2026-09-26⟧

Systematic rather than unlucky: rejections open with "thank you for your application" and put the decision afterwards,
so **every** rejection looks like an application update from the first 200 characters. Fixed by Simplify off plus
`trim_body`. ⟦REVIEW⟧

### Simplify changes the shape of the fields

| | Simplify on | Simplify off |
|---|---|---|
| Labels | `labels: [{id, name}]` | `labelIds: ['INBOX', …]` |
| Subject | `Subject` | `subject` |
| Sender | `From` (string) | `from.text`, `from.value[0].address` |
| Body | none, only `snippet` | `text`, `html`, `textAsHtml` |

A future `IMPORTANT`/`STARRED` guard therefore reads `$json.labelIds.includes('IMPORTANT')` after `fetch_mail`, but
`$json.labels.some(l => l.id === 'IMPORTANT')` after `history_lookup`. ⟦REVIEW src: measured both modes, 2026-09-26⟧

### An 8B model invents facts about you unless shown otherwise

Covered under `write_draft` above. The general lesson is the one already recorded for `memory_agent`: when rules fail
on a small model, an example is the next lever, not more rules. ⟦REVIEW⟧

### Header-based bulk detection is weaker than it looks

RFC 3834 defines `Auto-Submitted`, and bulk senders are meant to set `Precedence: bulk` or carry `List-Unsubscribe`.
In practice a measured Instahyre message carried **none** of them — only a `Feedback-ID` from Amazon SES and a
`noreply@` address. ⟦REVIEW src: measured raw headers, 2026-09-26⟧

`filter_repliable` therefore checks the sender address, which caught that message where the headers did not. The
header check remains available if address matching proves insufficient. ⟦REVIEW⟧

## Cost

| | |
|---|---|
| Per email, classification | ~16–18s, ~1,430 tokens |
| Per email, draft | ~20s, only for emails passing all three gates |
| Per run | 25 emails ≈ 7 minutes |
| Schedule | every 30 minutes |
| Gmail quota | ~4,000 units/run against a 15,000/minute per-user limit ⟦REVIEW src: Gmail API quota reference, 2026⟧ |

`history_lookup` fetches 100 messages every run whether or not there is new mail — nothing in quota terms, 20–40
seconds of work on a quiet run. The fix, if it ever matters, is an `If` that skips the rest when `fetch_mail` returns
nothing. ⟦REVIEW⟧

**The backlog is not being swept.** `newer_than:7d` caps it at one week. A full sweep of 6,000-plus messages would be
~27 hours of continuous local inference — a deliberate, separate decision, worth deferring until the classifier is
trusted, since labelling everything now bakes in today's error rate. ⟦REVIEW⟧

## What this does not do yet

| Gap | Where it lands |
|---|---|
| No Slack logging | Next. One card per run to `#tars_logs` — counts, breakdown, undecided, drafts written ⟦REVIEW⟧ |
| No `needs_reply` label | The label exists (`Label_10`) but nothing applies it. It belongs after `create_draft`, so it means "a draft is waiting" rather than "a human sent this" ⟦REVIEW⟧ |
| No archiving | Deliberate. `newsletter` only, once the labels are trusted ⟦REVIEW⟧ |
| Nothing in Obsidian | `mail_digest` → `dashboard/mail_tracker.md` ⟦REVIEW src: PLAN.md Phase 4.1⟧ |
| No chat access | `mail_agent`, its own trigger and system message ⟦REVIEW src: PLAN.md Phase 4.1⟧ |
| Sender map sampled at 100 | Recent history only. Correct for now — recency is what matters — but a sample, not the whole record ⟦REVIEW⟧ |
| A failed draft never retries | The labelling branch applies `sorted` regardless, so the email is not refetched. Remove `sorted` by hand to reconsider it ⟦REVIEW⟧ |
