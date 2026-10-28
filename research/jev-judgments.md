# Jev questions for the three Focus judgments

Research date: 2026-10-28. Resolves radimsem/omadock#3. Sources: live TypeSafe docs (`docs.typesafe.ai/llms.txt`, pages read as `.md`), Ollama's decision docs and Nimble page, the freedesktop notification spec, and omaone's code, ADRs 0001 and 0002, `docs/decisions.md`, `docs/contracts.md`, README settings and its two 2026-10-09 research notes. No paid API call was made and no secret was read. Nothing here is measured; every number is either cited or marked as an estimate.

## Short answer

- **Use Noul for the tags, Choice for "which Focus".** One shared Choice, `belongs_to`, has one option per Focus plus `none`. It serves all three judgments. Independent conditions (time-sensitive, needs an answer, promotional, matches any Focus) are Nouls. No Score is needed. Score is also the weakest primitive on the local model (Nimble 54.6% vs 80.2% for Noul on Bespoke Labs' public sets), and omaone's decider adapter implements only Choice and Noul.
- **The generated tags become a code-owned enum plus verbatim facts.** "Time-sensitive · needs your answer before 3pm" becomes tags `time_sensitive` and `needs_reply` (from Nouls) plus a quote that code cuts out of the message ("before 3pm"). "API docs for an incident tool" becomes the page's own title. No Jev output is shown as text.
- **Ask `belongs_to` without telling the model the current Focus.** The answer is then Focus-independent: store the probabilities with the held item, and re-decide on every Focus switch in code, with no new call.
- **One request per subject** (one notification, one window, one agent question), holding every question that subject needs: 4 questions for a notification, 2 for a window or site, 1 for an agent question. Never pack several subjects into one state.
- **Cost on cloud is negligible.** About 450 to 600 input tokens per request (estimate) at $0.042 per million input tokens, output free: about $0.00002 per call. **Latency has no published figure for this request shape.** The nearest numbers are 70 to 500 ms vendor claim and 236 to 256 ms p50 from France (third party). Measure from Czechia before promising anything.
- **Local `nimble-9b` can serve all three** (Choice up to 26 options, Noul, text only). Cost: Focus descriptors become strings, requests must fit omaone's 1024-token local cap, and Nimble's boolean accuracy is a few points under Jev's (80.2% vs 84.6% on Bespoke Labs' benchmark, not on our tasks). LAN works with any `/v1/systemone` endpoint; Laya cannot serve these questions zero-shot.
- **When the service is down, code decides and never adds friction.** Notifications from outside the Focus stay held and are marked "Not checked"; windows and sites show nothing and start no countdown; agent questions are admitted.

## 1. Shared design

### Code first, Jev for the leftover

TypeSafe says to keep known rules, exact lookups and arithmetic in code and use the model for semantic judgment ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one.md)). Jev 1.13 reads dates as text and does not compare or count reliably ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md)). So:

- App-to-Focus lookup, an exact known host, the freedesktop `urgency` hint (a BYTE hint, [spec](https://specifications.freedesktop.org/notification-spec/latest/hints.html)), a user's always-admit list and repo-to-Focus rules are code and need no call.
- Deadline parsing ("before 3pm") is code. Jev is never asked whether a time has passed.

### The Focus descriptor

Every option for `belongs_to` carries the same field names ([Choice: structured criteria](https://docs.typesafe.ai/primitives/choice.md), [How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one.md)):

```json
{"name": "Building", "about": "<user text, 1 sentence>", "apps": ["Ghostty", "Neovim"], "kept": ["github.com", "docs.rs"]}
```

`kept` is the hosts and titles the user chose to keep in that Focus (Stay or Switch answers). It plays the role of omaone's `past_requests`, which that project found to be its strongest evidence (`omaone/docs/research/2026-10-09-omaone-static-session-context.md`). A 6-Focus roster is about 52 tokens per Focus (estimate, chars/4 like omaone's `descriptor.tokens`). Ollama needs the descriptor as a compact JSON string, not an object (`descriptor_format: 'string'` in `omaone/helper/omaoned/deciders/__init__.py`).

### The shared Choice

```json
"belongs_to": {"type": "choice",
  "instructions": {"question": "Which activity does `<subject>` belong to? Judge by what it is about, not by the app that sent it.",
                   "rules": ["Pick none when no listed activity fits."]},
  "criteria": {"Building": <descriptor>, "Writing": <descriptor>, "Social": <descriptor>, "none": "No listed activity"}}
```

- **Why Choice:** the Focuses are mutually exclusive destinations and the probabilities compare them against each other, which is how omaone's `target` question works ([Choice](https://docs.typesafe.ai/primitives/choice.md), 255 options max; Nimble allows 2 to 26).
- **Why not name the current Focus:** the model "leans toward the option that comes first" ([jaggedness #8](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md)), and a "current" hint would bias it toward the Focus the user is already in. Shuffle the option order per request, and on a close call that would admit or refuse, re-ask once with the order reversed, as omaone's `router.py` does.
- **`none` caveat:** TypeSafe warns "a `none of the above` option may not work well" and suggests a companion Noul ([Choice](https://docs.typesafe.ai/primitives/choice.md)). Judgment 2 includes that companion because "matches no Focus" is a visible outcome. The eval should compare both.
- **Threshold use:** reuse omaone's two per-profile knobs, `t` (top probability) and `m` (lead over the runner-up), unchanged in Settings (Decision 7, README `backend.<profile>.t/.m`). Note that at the defaults T 0.80 and M 0.40, `m` never binds: a top of at least 0.80 leaves the runner-up at 0.20 or less, a lead of at least 0.60. It only matters if `t` is lowered below 0.70.

### Policy constants (starting guesses)

Thresholds are tied to a model version (omaone ADR 0002) and have not been tuned on Focus data. The ticket "Evaluation and thresholds" in map #1 is still open, so these are placeholders to replace.

| Constant | Start | Why |
|---|---|---|
| Choice gate | `t` per profile (0.80), `m` (0.40) | False friction and false holds cost trust; same as omaone |
| Noul yes for a tag shown | `t` | A wrong tag is visible copy |
| Noul yes for `time_sensitive` admit | 0.60 | Missing a real urgent message is worse than one extra interruption. TypeSafe: lower the threshold when missing a true yes is expensive ([Noul](https://docs.typesafe.ai/primitives/noul.md)) |
| Noul yes for `promotional` veto | 0.50 | Cheap to be wrong; held items stay reachable |
| Noul default band | yes 0.70, no 0.30 | TypeSafe's starting point ([Noul](https://docs.typesafe.ai/primitives/noul.md)) |

Per-profile `t` and `m` mean Nimble gets its own values (omaone Decision 7).

## 2. Judgment 1: hold or admit a notification

**Primitive.** Noul for each independent condition (several can be true at once, and each is one literal yes/no, which suits Jev's literal reading), plus the shared Choice. Score is rejected: urgency is a spectrum in principle, but the decision is binary, graded scores add little, and Score is the weak spot on both models (Jev 50.1%, Nimble 54.6% on Bespoke Labs' Score subsets, [Nimble page](https://ollama.com/library/nimble)).

**State fields.** From the Notify call ([spec](https://specifications.freedesktop.org/notification-spec/latest/protocol.html)): `app_name`, `summary`, `body`. Optional hints `category` and `desktop-entry` ([hints](https://specifications.freedesktop.org/notification-spec/latest/hints.html)). Whether Omarchy's notification plugin exposes all of them is ticket #5's question.

```json
{"notification": {"app": "Slack", "summary": "Maya Ortiz", "body": "Can you confirm the launch copy before 3pm? ...", "category": null}}
```

Cap `body` at about 600 characters. Omit timestamps and any `now` field. Jev does not do date arithmetic.

**Questions (sketch).**

```json
"time_sensitive": {"type": "noul", "instructions": "Does `notification.body` say the user must act or answer soon: a deadline, today, now, or marked urgent or asap?"},
"needs_reply":    {"type": "noul", "instructions": "Does `notification.body` ask the user a direct question or ask for a reply, approval or decision?"},
"promotional":    {"type": "noul", "instructions": "Is `notification` marketing, a newsletter, a promotion or an automated digest?"},
"belongs_to":     <shared Choice, subject = `notification`>
```

Each question is phrased so that a high value means yes ([Noul: writing a question](https://docs.typesafe.ai/primitives/noul.md)). `time_sensitive` and `needs_reply` stay separate so the combination is code's.

**Fixed tag set and verbatim facts.**

| Tag id | Copy | Source |
|---|---|---|
| `time_sensitive` | Time-sensitive | Noul at or above threshold |
| `needs_reply` | Needs your answer | Noul at or above `t` |
| `promotional` | Promotion | Noul (veto) |
| `fits_focus` / `for_focus` | Fits {Focus} / For {Focus} | Choice top, from the config name |
| `unjudged` | Not checked | code, when the call failed |

Verbatim facts: the sender (`summary`) and app name as received, and a **quote** found by code: the first regex hit for a deadline phrase (`before 3pm`, `by 15:00`, `in 10 minutes`, `today`, `asap`), shown only when `time_sensitive` is a tag. With no hit, show no quote. Rendered: `Slack · Time-sensitive · Needs your answer · "before 3pm"`.

**Probabilities to hold or admit** (first match wins):

1. Code gates, no call: app is in the current Focus's `apps`, hint `urgency` is critical, or the app is on the always-admit list. Admit.
2. Call failed or malformed: hold, tag `unjudged` (Section 7).
3. `belongs_to` top is the current Focus, `p1 >= t`, lead `>= m`: admit, tag `fits_focus`.
4. `promotional >= 0.50`: hold.
5. `time_sensitive >= 0.60`: admit, tags `time_sensitive`, plus `needs_reply` when it is `>= t`.
6. Otherwise hold. Label "Held for {Focus}" when the Choice gate passes for another Focus, else "Held". Sort the held list by `time_sensitive` descending, so a 0.30 to 0.60 item sits at the top rather than vanishing (TypeSafe's three-way split, [Confidence](https://docs.typesafe.ai/confidence.md)).

Store the raw probabilities with each held item. On a Focus switch, re-run rules 3 to 6 in code. Thresholds and the current Focus live in code, so changing them needs no new inference when the evidence and question meanings are unchanged (the weights-in-code idea of [Composite scoring](https://docs.typesafe.ai/patterns/composite-scoring.md)). Re-ask only if the Focus config changed since judging.

**Batch.** All four questions in one request over one notification. A burst of notifications goes out as concurrent requests, one per notification: a combined state is "full of irrelevant detail" that lowers accuracy ([jaggedness #5](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md)).

## 3. Judgment 2: does a window or site fit the Focus

**Primitive.** Choice for the three-way outcome (the shared `belongs_to`: top is the current Focus, another Focus, or `none`) plus a companion Noul `matches_any` for "matches no Focus", per TypeSafe's own advice about `none`.

**State fields.** Signals are ticket #4's job; this is the fixed shape.

```json
{"subject": {"kind": "site", "host": "tidepool.dev", "title": "Tidepool API reference: incidents, pages and on-call",
             "description": "REST API for the Tidepool incident tool.", "opened_from": "Firefox"}}
```

For a native window: `kind: "window"`, `class`, `title`, and the `.desktop` `Name` and `Comment`. Cap `title` at 120 and `description` at 200 characters. Keep the URL path and query out unless a later privacy decision says otherwise. Only the host and page-supplied title and description are needed.

**Questions (sketch).**

```json
"belongs_to":  <shared Choice, subject = `subject`, "Judge by `subject.title` and `subject.description`">,
"matches_any": {"type": "noul", "instructions": {"question": "Does `subject` serve at least one of these activities?",
                "activities": {"Building": "<about>", "Writing": "<about>", "Social": "<about>"}}}
```

The Noul repeats only each Focus's one-sentence `about`, because Noul cannot see Choice criteria.

**Fixed outcome enum and facts.** `fits`, `other_focus(id)`, `none`, `unsure`, `unjudged`. Copy comes from templates:

- `{host} · Fits {Focus} · "{title}"`
- `{host} belongs to {Focus}. Stay focused, or switch?`
- `{host} · Matches no Focus`

`host` is code-parsed, `title` is the page's own, cut at a word boundary. The demo's "API docs for an incident tool" is the one string we cannot reproduce, by design (ADR 0001).

**Probabilities to outcomes.**

| Condition | Outcome | Friction |
|---|---|---|
| Exact host in a Focus's known domains (code) | `fits` or `other_focus` | per Focus config, no call |
| `matches_any <= 0.30` | `none` | none by default (owner decision) |
| Choice top is current, `p1 >= t`, lead `>= m` | `fits` | none |
| Choice top is X, not current, `p1 >= t`, lead `>= m`, `matches_any >= 0.70` | `other_focus(X)` | countdown with Stay (friction, never a block) |
| anything else | `unsure` | none, no chip |

Only a confident claim for another Focus adds friction, since enforcement is friction (map #1). A site that fits two Focuses has a split distribution and is `unsure`, which is the safe side.

**Batch.** Both questions in one request per window or site. Cache the verdict per `(host, title, Focus-config hash)` for the life of the window. A per-host cache for multi-purpose hosts (YouTube, GitHub) would hide title differences, so do not add one before measuring.

## 4. Judgment 3: hold or admit an agent's question

**Primitive.** The shared Choice `belongs_to`, subject `session`. The question is about the agent's session, not the question text, so the model never has to read the (possibly long) question to place it. A session-to-Focus rule in config (repo path, harness tag) is code and skips the call.

**State fields.** Reuse omaone's Descriptor, redacted by its own `redact_descriptor` (`omaone/helper/omaoned/descriptor.py`):

```json
{"session": {"title": "Fix flaky export test", "repo": "aside", "branch": "main",
             "last_prompt": "make the export test deterministic", "harness": "claude"},
 "question": {"text": "<verbatim, cap 600 chars>", "options": ["Fixture", "Clock argument"]}}
```

`question` is context only; `strict` mode (omaone `backend.cloud.strict`, which drops `last_prompts` and `pending_question`) still leaves `title`, `repo` and `branch`, so `belongs_to` keeps working. Whether cloud may see any of this is the open privacy-envelope ticket in map #1.

**Questions.** One: `belongs_to`. Optional second, only for free-text questions: a Choice `kind` with options `approve`, `choose`, `provide`, `confirm`. Code derives `kind` for structured prompts (permission dialog means `approve`, options present means `choose`), so most requests carry one question. This is the display tag.

**Fixed tags and facts.** `kind` tag (Permission, Choice, Needs input, Confirm), `fits_focus` or `for_focus`, `unjudged`. Verbatim: the agent's question text and options as omaone's Inbox already shows them (ADR 0001 consequence), plus session name, repo and branch.

**Probabilities to hold or admit.**

| Condition | Outcome |
|---|---|
| Rule match (code) or Choice top is current, `p1 >= t`, lead `>= m` | admit |
| Choice top is another Focus X, `p1 >= t`, lead `>= m` | hold; item stays in the Inbox, reachable, labelled "For {Focus}" |
| `none`, split distribution, or call failed | admit, tag `unjudged` when it failed |

Ambiguity admits here but holds in Judgment 1 on purpose. The agent is the user's own and is blocked while it waits, and every held item is reachable anyway. This asymmetry is an owner decision to confirm.

**Batch.** One question, one request per agent question. The omaone router already judges per session event.

## 5. Cost and latency (cloud, Jev)

**Price.** $42 per billion input tokens ($0.042 per million), output free; `jev-1.13.0` and `jev-1.14.0` cost the same ([Models](https://docs.typesafe.ai/models.md)). Limits: 64k tokens per request, 32k per question, no limit on question count; rate limits are per organization, shown in the console, and `429`/`529` are retried by the SDKs ([Models](https://docs.typesafe.ai/models.md), [errors](https://docs.typesafe.ai/api/errors.md)).

**Estimated size per request** (sketches above, six Focuses, chars/4 like omaone's `tokens()`; the real count is `usage.input_tokens`):

| Judgment | Questions | Est. input tokens | Est. cost per call |
|---|---|---|---|
| Notification | 4 | about 560 | $0.000024 |
| Window or site | 2 | about 590 | $0.000025 |
| Agent question | 1 | about 450 | $0.000019 |

At an assumed 300 judgments a day this is about $0.007 a day, roughly $0.21 a month. The 300 is invented to show scale, not a measured figure.

**Latency.** No figure is published for this request shape, and none was measured here.

| Source | Figure | Caveat |
|---|---|---|
| [TypeSafe announcement](https://typesafe.ai/blog/introducing-system-one-models-and-jev) | "70ms-500ms" end to end | Run from the vendor's West Coast laptops; no percentile |
| [AbdelStark/jev-benchmarks](https://github.com/AbdelStark/jev-benchmarks) | 236 to 256 ms p50 (4 and 6 labels), 246 ms (72 labels) | Third party, called from France |
| [Twitch chat filter cookbook](https://docs.typesafe.ai/cookbooks/twitch_chat_filter.md) | median about 0.4 s per message | Multi-question request per message |
| [Skill suggestion cookbook](https://docs.typesafe.ai/cookbooks/skill_suggestion.md) | 0.09 to 0.31 s per call, 182 options | Printed run output |
| [Parallel questions cookbook](https://docs.typesafe.ai/cookbooks/parallel_questions.md) | 13 questions, about 54k characters: 0.27 s and $0.000497 vs 2.71 s and $0.006090 for 13 calls | Priced at jev-1.12 |

Batching questions about one subject is cheaper because the state is paid once, and it adds little time since questions run in parallel ([Fan-out](https://docs.typesafe.ai/patterns/fan-out.md)). Plan on a budget of 1.5 s per call, which is omaone's decider timeout (`router.py` `_ask`; `docs/contracts.md` lists 1.5 s). A notification is "pending" until it answers or times out, so a genuine urgent message waits at most that long.

**Model version.** `jev-latest` now points to `jev-1.14.0` (from 2026-10-14, [Models](https://docs.typesafe.ai/models.md)). omaone pins `jev-1.13.0` and only a 1.13 jaggedness page exists. Pin 1.13.0 and treat 1.14 as an eval candidate.

## 6. Local and LAN profiles

| Aspect | Jev 1.13 (cloud) | `nimble-9b` (Ollama) |
|---|---|---|
| Choice, Noul | yes, 255 options | yes, 2 to 26 options ([Ollama decision](https://github.com/ollama/ollama/blob/main/docs/capabilities/decision.mdx)) |
| Criteria as objects | yes | strings only, so serialize descriptors |
| Questions per request | unlimited within context | 1 to 64; "reads the prompt once per question" ([Nimble page](https://ollama.com/library/nimble)), so local cost grows with question count |
| Accuracy (Bespoke Labs, 13 public sets) | Boolean 84.6%, Choice 82.9%, Score 50.1% | Boolean 80.2%, Choice 81.6%, Score 54.6% ([Nimble page](https://ollama.com/library/nimble)) |
| Latency | see above | "under 100ms" on an M5 Max; this host is untested |
| Price | per input token | $0 |

- **All three judgments are servable** with Choice (about 6 to 12 options) and Noul.
- **Accuracy figures** come from Bespoke Labs and are not our tasks. The Focus eval must run per profile.
- **Context:** the Nimble library page now lists a 256K window, but omaone caps Local at 1024 input tokens (`make_decider`, `max_input_tokens`). The sketches above fit under that, up to roughly 10 to 14 Focuses at 52 tokens each. Give Local a tighter Focus descriptor cap (about 30 tokens; omaone already trims its session descriptors from 250 tokens on cloud to 180 locally, `CAP_CLOUD` and `CAP_LOCAL`).
- **This host:** `ollama` is not installed (checked with `command -v`). Nimble needs Ollama 0.35.0 or later. Latency and VRAM fit on the RTX 3060 are unmeasured.
- **LAN:** any `/v1/systemone` server works. Nimble through Ollama on another machine is the same wire. Laya zero-shot cannot serve this: its English budget is 512 tokens per question with a 192-token share for all options, and its base checkpoint scored 0.362 against a 0.461 majority baseline ([model card](https://huggingface.co/convaiinnovations/laya)). A Focus roster alone is about 312 tokens.
- **Privacy:** Local and LAN are the profiles that keep notification bodies, URLs and agent questions on the user's network. Which judgments may run only there belongs to the privacy-envelope ticket.

## 7. Failure behaviour

omaone's decider raises `timeout`, `auth`, `no_key`, `http`, `malformed` or `unreachable`; the router turns any of them (and any non-`DeciderError`) into a code-chosen `Ask` outcome and logs the row (`router.py`, `route`). Choice answers whose probabilities do not sum to 0.98 to 1.02 are `malformed` (`validate_choice`). Focus does the same, and the rule per judgment is "never add friction":

| Judgment | On failure | Reason |
|---|---|---|
| Notification | Code gates still apply. Otherwise hold, tag `unjudged` ("Not checked"), re-judge when the service returns | Hold is the Focus default, and held items are reachable |
| Window or site | Outcome `unjudged`: no chip, no countdown. Exact-host lookups still work | A failed check cannot become a reason to interrupt |
| Agent question | Admit, tag `unjudged` | The agent is blocked and the user started it |

- A missing key (`no_key`) is surfaced by the existing Settings "Test connection" (`backend_test`).
- Log every request with state, questions, probabilities and thresholds, like omaone's `audit.jsonl`, so thresholds can be calibrated.
- No retry loop or circuit breaker is proposed. The SDK's own retry on `429`/`529` is enough within a 1.5 s budget. A breaker would be new machinery for a failure that has not happened yet.

## 8. Not verified

- Latency from this host to `api.typesafe.ai`, and Nimble's latency and fit on this GPU.
- Accuracy and calibration on Focus data. Every threshold above is a starting value; TypeSafe publishes no calibration error, and neither do the vendors' benchmark pages for these tasks.
- Whether Omarchy's notification stack passes `category` and `desktop-entry` (ticket #5) and which window signals exist (ticket #4).
- Whether `none` or the companion `matches_any` Noul separates "no Focus" better. Needs the eval set.
- Nimble's real context limit: the library page says 256K, omaone's earlier note found 8,192, and the Ollama API page only says input must fit the loaded window and 64 KiB of text.
- Prompt caching or prefix reuse for Nimble across questions: not documented.

## Sources

- TypeSafe: [index](https://docs.typesafe.ai/llms.txt), [Models](https://docs.typesafe.ai/models.md), [State](https://docs.typesafe.ai/concepts/state.md), [Primitives](https://docs.typesafe.ai/primitives.md), [Choice](https://docs.typesafe.ai/primitives/choice.md), [Noul](https://docs.typesafe.ai/primitives/noul.md), [Score](https://docs.typesafe.ai/primitives/score.md), [Confidence](https://docs.typesafe.ai/confidence.md), [How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one.md), [Fan-out](https://docs.typesafe.ai/patterns/fan-out.md), [Jev 1.13 jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md), [API errors](https://docs.typesafe.ai/api/errors.md), [announcement](https://typesafe.ai/blog/introducing-system-one-models-and-jev).
- Local models: [Ollama decision guide](https://github.com/ollama/ollama/blob/main/docs/capabilities/decision.mdx), [Ollama System One API](https://docs.ollama.com/api/systemone), [Nimble](https://ollama.com/library/nimble), [Laya card](https://huggingface.co/convaiinnovations/laya).
- Platform: [freedesktop Notify](https://specifications.freedesktop.org/notification-spec/latest/protocol.html), [hints](https://specifications.freedesktop.org/notification-spec/latest/hints.html).
- omaone (local checkout, private repo): `docs/adr/0001-no-generative-model.md`, `docs/adr/0002-code-decides-with-a-fuse.md`, `docs/decisions.md` (Decisions 7, 12, 14), `docs/contracts.md`, `README.md` settings table, `helper/omaoned/router.py`, `policy.py`, `descriptor.py`, `deciders/__init__.py`, `deciders/systemone.py`, `docs/research/2026-10-09-omaone-router-decision-models.md`, `docs/research/2026-10-09-omaone-static-session-context.md`.
