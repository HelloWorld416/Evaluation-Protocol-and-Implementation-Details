# C. Evaluation Protocol and Implementation Details

This section specifies the evaluation contract used for the final
`StreamDriveBench-1935-v1` results. The implementation is pinned by the
`sdb1935_causal_window15_v1` evaluation context and the
`streamdrivebench_causal_qa_1hz_v1` protocol.

## C.1 Release and task composition

The release contains 1,935 public questions from five driving-video sources.
The three task families are mapped as follows:

| Family in the release | Paper name | Questions | Response protocol |
|---|---|---:|---|
| HGI | History QA | 700 | One answer at the query time |
| AD | Prediction QA | 552 | One answer at the query time |
| ETP | Event QA | 683 | One answer at each monitoring second until termination |

The 683 Event QA questions contain 572 positive events and 111 negative
windows in which the event is not observed. The release also contains 444
complete H/P/E groups for joint metrics. These groups comprise 66
source-marked strictly linked groups and 378 context-associated groups.

The public input contains questions and choices only. Ground-truth answers,
event trigger times, evidence frames, and the positive/negative event label
are kept in the private evaluation files and are never inserted into a model
prompt.

## C.2 Causal input construction

### Public query format

Each query is a JSON object with the following fields:

```json
{
  "query_id": "<unique question id>",
  "episode_id": "<episode id>",
  "dataset": "<source dataset>",
  "family": "HGI | AD | ETP",
  "query_second": 726,
  "model_input": {
    "question": "<question shown to the model>",
    "choices": {"A": "<option>", "B": "<option>", "C": "SILENT"},
    "target_description": "<target at injection>",
    "predicate": "<event or state predicate>",
    "monitoring_instruction": "<optional event instruction>",
    "prediction_horizon_seconds": 3,
    "prediction_semantics": "<optional prediction semantics>",
    "prediction_quantity": "<optional quantity>"
  },
  "input_policy": {
    "prefix_end_second": 726,
    "history_start_second": 724,
    "online_monitor_end_second": 732,
    "evaluation_session": "independent_per_question"
  }
}
```

`HGI` and `AD` use `online_monitor_end_second = query_second` and are
single-shot questions. `ETP` has one `SILENT` choice and may continue until
`online_monitor_end_second`. Choice labels are canonical (`A`, `B`, ...),
unique, and there are two to five choices per question.

### 15-second, 1-FPS causal window

For a response at absolute second `t`, the supplied timestamps are

\[
  \{\max(h, t-15),\;\max(h, t-15)+1,\ldots,t\},
\]

where `h` is the selected clip's effective history start. Thus a full
window spans 15 seconds and contains at most 16 frames, including the current
frame. The current frame is always included; frames after `t` are never
visible. The effective history start is
`ceil(selection_start_sec)`. At a clip boundary, the window is truncated at
that start without padding, time shifting, or synthetic frames.

The same rule is applied independently to every ETP poll. A poll at second
`t` therefore sees only the causal window ending at `t`, even when an earlier
poll has already been answered. Prepared frames are accepted only when their
source timestamp is within 0.5 seconds of the requested integer second and
the observation is valid.

## C.3 Prompt construction

The following text is the exact shared autonomous header used by visual
models (line breaks are preserved):

```text
You are a driving video assistant viewing chronological forward camera images at 1 FPS. Use only observations supplied through the current time. For AD, predict the requested future state without future images. For HGI, use visible history to interpret the current state. For ETP, respond with the option meaning SILENT until the requested NEW event becomes visibly established after injection; then answer immediately. Turn excludes ordinary lane changes and road-following curvature unless explicitly stated otherwise. Return ONLY one uppercase supplied option letter, without text, punctuation or explanation.
```

For a query at current second `t`, the header is followed by:

```text
Task: {family}. Injection/query time: {query_second} seconds. Current time: {t} seconds.
Only absolute times [{first_supplied_second}, {t}] seconds are supplied. Native video time zero corresponds to absolute {first_supplied_second} seconds. No earlier memory or previous answers are supplied.
Target at injection: {target_description}
{question}
{monitoring_instruction_if_present}
Prediction semantics: {prediction_semantics_if_AD}
Review ALL supplied historical frames before predicting at the unchanged query time. Available history spans {t-first_supplied_second} seconds; the requested maximum is 15 seconds. No future frames are supplied.
Your first non-SILENT response is final and cannot be corrected. Select your response NOW.
A. {choice_A}
B. {choice_B}
...
```

The AD-specific lines are included only for AD, the ETP final-response line
only for autonomous ETP polling, and the choice list contains exactly the
public choices in canonical order. HGI answers use all supplied causal
history to interpret the state at the query time. AD answers are made once at
the unchanged query time and predict the requested future state without
future images. In the release, AD labels use the annotated future interval
specified by `prediction_horizon_seconds` and `prediction_semantics`.

For ETP, monitoring begins at the injection second. The model must output the
`SILENT` option while the event is not visibly established, then produce the
first non-`SILENT` option immediately after the event becomes established.
That first non-silent response is terminal and cannot be revised. A negative
ETP question must remain validly silent through the entire monitoring window.

### Oracle-trigger diagnostic prompt

The oracle diagnostic uses the following separate header:

```text
You are a driving video assistant viewing chronological forward camera images at 1 FPS. This is a given-trigger-time diagnostic: the monitoring condition is confirmed to occur at the current supplied time. Use only observations through this time and select the non-SILENT option describing the observed outcome. You do not need to decide when to respond. Return ONLY one uppercase supplied option letter, without text, punctuation or explanation.
```

The prompt then includes the same causal timestamp declaration, target,
question, and choices, followed by:

```text
ORACLE-TIME DIAGNOSTIC ONLY: The condition is confirmed at the current supplied time. Answer the observed outcome NOW; do not wait for a later frame.
```

Oracle evaluation is run only on the 572 positive ETP questions. The input
cutoff is the private annotated trigger second, and the prompt explicitly
states that the event occurrence is confirmed. Oracle answer accuracy
therefore measures answer recognition given both the trigger time and an
event-occurrence cue; it is not a pure measurement of temporal localization.

## C.4 Execution protocol

Every query is evaluated in an independent session. Before a call, the
adapter is reset with `start_episode`; frames are observed in chronological
order, the prompt is issued, and `end_episode` clears the model state. No
answer, hidden state, or visual frame is carried from one question or one
ETP poll to another.

HGI and AD make one model call at `query_second`. ETP starts at
`query_second`, polls at every integer second, and keeps only active queries.
Polling stops at the first non-`SILENT` response, at an invalid response, or
after the monitoring deadline. Each poll receives the fresh causal window
ending at that second. Batch inference, where available, batches independent
questions only; it never batches future ETP ticks into the current input.

The runner writes each poll to a per-query checkpoint immediately. A resumed
run first validates the saved timestamps and visible frame list against the
same causal protocol. This prevents an interrupted run from silently
changing the observation window.

## C.5 Visual representation and transport

The benchmark supplies the prepared `CAM_FRONT` image as RGB. The source
dimensions and aspect ratio are preserved, and the runner adds a 36-pixel
black header containing
`FRONT | source time {absolute} seconds`. This timestamp header is part of
the image seen by the model and makes the absolute frame time explicit.

For API models, each frame is encoded as a lossless PNG. The request content
contains the prompt followed by chronological `Frame at {t} seconds` text and
the corresponding image. Requests use the high-detail image setting for the
GPT-5.6 Sol evaluation, `store=false`, and no cross-window memory. Local VLM
processors may apply their model-specific resize or normalization after this
common representation; they receive the same ordered frames and timestamps.

## C.6 Models and inference settings

The final comparison includes the following model identifiers. We report the
identifier used by the local checkpoint or API adapter so that the exact
model revision can be pinned by the accompanying contract files.

| Evaluation family | Model identifiers | Decoding and batching |
|---|---|---|
| Local visual VLMs | Qwen3-VL-8B, Qwen3.5-9B, InternVL3.5-8B, MiniCPM-V4.5-8B, Molmo2-8B, Penguin-VL-8B, InternVideo3-8B, Cosmos-Reason2-8B | Greedy decoding (`do_sample=false`), `max_new_tokens=16`, batch size 2, bfloat16 where supported, FlashAttention 2 when supported |
| Streaming VLMs | StreamForest-Qwen2-7B (`StreamForest`), VideoChatOnline-4B (`VideoChat-Online`), OneStreamer-4B (`OneStreamer`) | One independent stream per query, batch size 1, chronological contiguous frames, `max_new_tokens=16`, greedy decoding, no cross-window memory |
| API model | GPT-5.6 Sol (`gpt-5.6-sol`) | Responses API, reasoning effort `medium`, `max_output_tokens=16384` including reasoning, `image_detail=high`, concurrency 4, `store=false` |

Local visual runs use a 512 MB frame cache and `attention=auto`; the final
contracts record whether FlashAttention 2 was enabled. Streaming adapters
also validate that their memory bank retained every supplied frame in order.
The GPT-5.6 Sol adapter accepts only a completed response from the requested
model family and retries transient transport failures with bounded attempts;
an incomplete or refused response is saved as an invalid model output rather
than replaced by a different answer.

The repository also contains two controlled ablations. The text-only
ablation uses the public query fields and no images, with the same greedy
16-token budget and strict parser. The matched oracle ablation uses the
visual prompt text without images and supplies the true positive trigger
cutoff and occurrence cue; it excludes the 111 negative ETP questions and is
reported separately from the main proactive evaluation.

## C.7 Output parsing and invalid-output handling

The required output is one uppercase option letter. The parser applies the
following rules:

1. An exact offered letter (`A`–`E`) is valid.
2. A letter followed by `.` or `)` and the exact offered choice text is also
   valid; a final period is ignored for this comparison.
3. Any other prose, lower-case or multi-letter answer, a missing choice,
   mismatched explanation, or the literal word `SILENT` without its option
   letter is invalid and maps to `answer = None`.

For HGI and AD, an invalid answer is a completed failure. For ETP, an
invalid output is a terminal invalid response and receives no credit. Before
the terminal response, every poll must parse to the supplied `SILENT` letter;
otherwise the timeline is invalid. A noncausal timestamp, missing poll,
extra poll, or poll after a terminal response is also rejected. Invalid
outputs and misses remain in the fixed denominator; no metric is conditioned
on the model having emitted a valid answer.

For a positive ETP question, let `t_g` be the annotated trigger second and
`t_r` the first terminal response second. The response is timely exactly
when

\[
  0 \leq t_r - t_g \leq 1\text{ second}.
\]

The event is successful only when the terminal option is correct and timely.
For a negative ETP question, success requires valid `SILENT` at every poll
through the deadline; any non-silent or invalid terminal response is a false
alarm/failure.

## C.8 Metrics

Let `S_H`, `S_P`, and `S_E` be the numbers of successful HGI, AD, and ETP questions,
where ETP success includes both positive correct-and-timely answers and
negative uninterrupted silence.

### Individual QA metrics

| Metric | Definition | Denominator |
|---|---|---:|
| History Accuracy | `S_H / 700` | 700 HGI questions |
| Prediction Accuracy | `S_P / 552` | 552 AD questions |
| Event Accuracy | `S_E / 683` | 683 ETP questions |
| Overall Accuracy | `(S_H + S_P + S_E) / 1935` | all 1,935 questions |
| Average Accuracy | `(Acc_H + Acc_P + Acc_E) / 3` | unweighted task mean |

Average Accuracy is computed from the unrounded task accuracies and rounded
only for display. Overall Accuracy uses the pooled question count, so it is
not the same as the task mean when task sizes differ.

### Joint task metrics

For the same set `G` of 444 complete H/P/E groups, define

\[
J_{XY}=\frac{1}{|G|}\sum_{g\in G}
  \mathbf{1}[\text{all rows of family }X\text{ and }Y\text{ in }g\text{ succeed}],
\]

and analogously

\[
J_{HPE}=\frac{1}{|G|}\sum_{g\in G}
  \mathbf{1}[H_g\land P_g\land E_g].
\]

The reported metrics are `J_HP`, `J_HE`, `J_PE`, and `J_HPE`.
Every one uses the same 444 complete groups and a group succeeds only when
all indicated questions succeed. These are joint success rates, not products
of marginal accuracies.

### Event QA diagnostics

The diagnostic table separates event recognition, timing, and silence:

| Diagnostic | Definition | Denominator |
|---|---|---:|
| Correct + Timely | Positive answer is correct and the first response is in \([t_g,t_g+1]\) | 572 positive ETP questions |
| Answer Accuracy | Positive terminal answer is correct, ignoring response timing | 572 positive ETP questions |
| On-time Rate | Positive response is valid and occurs in \([t_g,t_g+1]\), regardless of answer correctness | 572 positive ETP questions |
| Correct Silence | All monitoring polls for a negative question are valid `SILENT` | 111 negative ETP questions |
| Oracle Answer Accuracy | Correct answer at the true trigger cutoff with the explicit occurrence cue | 572 positive ETP questions |

Misses, invalid outputs, early responses, and late responses count as failures
in the fixed denominators. The main-table Event Accuracy is

\[
\frac{\#\text{positive correct-and-timely}
      +\#\text{negative correct-silence}}{572+111}.
\]

Oracle-trigger evaluation supplies two pieces of information: the correct
trigger-time cutoff and a statement that the event is confirmed at that
time. Its accuracy therefore should not be interpreted as visual localization
accuracy or as an intervention that changes timing alone.

## C.9 Reproducibility artifacts and implementation basis

The protocol is implemented in the following source files in the released
project:

- `src/sdb_release/protocol.py`: prompt templates, causal frame selection,
  parser, timelines, and per-question scoring;
- `src/sdb_release/data.py`: public/private field separation and release
  validation;
- `src/sdb_release/evaluation_context.py`: clip-boundary history policy and
  the 15-second context snapshot;
- `src/sdb_release/runner.py`: frame construction, state reset, streaming
  polling, checkpoints, oracle runs, and group scoring;
- `src/sdb_release/adapters.py` and `scripts/streaming_models.py`: model
  adapter contracts and streaming implementations;
- `scripts/gpt56_api.py`: GPT-5.6 Sol request construction and response
  validation;
- `reports/paper_table_1935_updated_20261101/build_table.py`: main-table
  individual and joint metrics;
- `reports/paper_event_diagnostics_updated_20261101/build_table.py`:
  proactive Event diagnostics and oracle metrics.

The final results are accompanied by per-model `contract.json` files,
per-query checkpoints, input/context hashes, and audit CSV/JSON files. A
reproduction must use the pinned release version, the 15-second context
manifest, the same model revision and decoding parameters, the same strict
parser, and the fixed denominators above.
