# Spec: Rewrite Agent Laboratory as an Actor-Based Rust System

**Status:** Draft v2 (audited against the paper and the source) · **Scope:** whole codebase (~4.1k lines of Python across 9 modules: `ai_lab_repo.py`, `agents.py`, `mlesolver.py`, `papersolver.py`, `tools.py`, `utils.py`, `inference.py`, `app.py`, `common_imports.py`) · **Approach:** TDD, experiment-gated, incremental

---

## 1. Problem Statement

Agent Laboratory is an LLM-driven research pipeline of seven subtasks in four phase groups (`ai_lab_repo.py:60`): literature review → plan formulation → experimentation (data preparation, running experiments) → results interpretation (results interpretation, report writing, report refinement). It is implemented as one orchestrator class (`LaboratoryWorkflow`) that sequentially drives six agent objects (five `BaseAgent` roles plus a `ReviewersAgent`) through shared mutable state, with these structural problems:

- **Shared mutable state, no isolation.** `set_agent_attr` mutates the five role agents at once; the shared `notes` list is one object aliased by every agent; phase transitions are a chain of `if subtask == "..."` blocks in `perform_research` (`ai_lab_repo.py:144`); the refinement loop recurses into `perform_research()`.
- **Fragile persistence.** State is saved by pickling the whole workflow object (`save_state`) to `state_saves/Paper{n}.pkl` after every subtask; nothing in the codebase loads it back, so runs cannot be resumed at all today.
- **Unsafe, hard-to-test execution.** LLM-generated code runs via `exec` in a `multiprocessing.Process` with string-match guards (`exit(`, `load_dataset('pubmed`) in `tools.py:306`; stdout is swapped globally; output is never truncated (the `MAX_LEN` parameter is unused), and output larger than the pipe buffer deadlocks and is reported as a timeout.
- **Untestable LLM boundary.** `query_model` (`inference.py`) is a ~200-line if/elif chain over model names that mutates `os.environ` with API keys and keeps a process-global token ledger; a second, unused family of `query_*` functions in `utils.py` exists only for agent-generated code. There are no tests.
- **Duplicated code.** `Command`/`Replace`/`Edit` exist in both `mlesolver.py` and `papersolver.py`; `get_score` exists in both `agents.py` and `mlesolver.py`.
- **Parallel labs are unfinished.** The `parallel-labs` branch runs labs in a `ThreadPoolExecutor` sharing CWD artefacts and the global ledger and ends in `raise NotImplementedError` (`ai_lab_repo.py:828`); real multi-lab runs are separate processes with distinct `lab-index` YAMLs sharing `uploads/` and the Flask hub (`app.py`).
- **Latent config bugs.** Several YAML keys are read under a different spelling than the shipped configs use, so copilot mode and the AgentRxiv config do not actually switch on; one phase name is misspelled in the per-phase model map, pinning report refinement to the default backend (§7.2).

**Who is affected:** maintainers and contributors (every new model/phase/tool touches several files), and researchers running long, expensive (LLM-cost) pipelines who lose progress on crashes or cannot resume reliably.

**Cost of not solving it:** each new backend or phase increases regression risk with no safety net; long runs remain non-resumable and non-observable.

> Assumption: this spec was written from a read of the code and the stated goal ("actor-based idiomatic Rust, TDD, experiment-based"). Items I could not determine are in §9 Open Questions.

---

## 2. Goals

| # | Goal | Measure |
|---|------|---------|
| G1 | **Behavioral parity** on the supported workflow | In `--parity-mode`, replay of recorded LLM transcripts produces byte-identical prompts, parsed commands, and phase artifacts to Python for 100% of golden scenarios; any diff is a bug unless it is a §7.2 row with that row's test passing in both modes |
| G2 | **Actor architecture** with explicit message protocols, no shared mutable state | 0 `Arc<Mutex<_>>` over domain state; every actor owns its state; all inter-actor interaction is typed messages |
| G3 | **Resumable runs** | Kill the process at any point; restart resumes from the last completed step with ≤ 1 LLM call repeated. Verified by a crash-injection test at every phase boundary |
| G4 | **Test-first, deterministic core** | ≥ 90% line coverage on pure/domain crates; the whole pipeline runs in CI with zero network and zero real LLM calls in < 60 s |
| G5 | **Safe experiment execution** | LLM-generated code runs in a supervised subprocess with enforced timeout, output cap, and kill-on-drop; no in-process `exec`, no string-matching guards as the primary defense |
| G6 | **Pluggable LLM backends** behind one trait | Adding a backend = one new impl + one config entry, with zero changes to agents/phases |
| G7 | **Zero Python in the end state** | `git ls-files` contains no `.py`; agents generate and run Rust experiments; Pixi/Python env deleted (Phase 8 gate) |
| G8 | **Every change in results/performance is recorded and diffable** | Any two commits can be compared with one command (`dvc metrics diff` / `dvc exp diff`) covering quality, cost, tokens, latency, and parity |

---

## 3. Non-Goals

1. **Rewriting the generated experiments in Rust in the first release.** v1 runs agent-written Python as a subprocess (Pixi env). *Reason: parity first.* **Deferred, not dropped:** the long-term goal is to remove ALL Python, including the code agents generate (Phases 7–8, E11).
2. **New research features or new phases.** No behavior changes beyond parity (bug fixes recorded as explicit deviations). *Reason: parity is the only way to validate a rewrite.*
3. **A new UI.** The Flask `app.py` surface is kept as an HTTP API at most (only `/api/search`, `/uploads/<path>` and `/update` work today; the HTML routes reference a `templates/` directory that is not in the repo); no new frontend. *Reason: separate initiative.*
4. **Replacing LaTeX tooling.** `pdflatex` stays an external dependency invoked as a subprocess.
5. **Big-bang cutover.** Python stays runnable until Phase 5 gate passes. *Reason: risk.*
6. **Translating the 16 localized READMEs or media.** Out of scope.

---

## 4. Target Architecture

### 4.1 Actor model

Hand-rolled Tokio actors (task + `mpsc` inbox + `oneshot` replies, per the "actor = handle + task" pattern): small, no framework lock-in, trivially testable. A framework (`ractor`, `kameo`) is evaluated in Experiment E1 before commitment.

| Actor | Replaces (Python) | Owns | Key messages |
|-------|-------------------|------|--------------|
| `Supervisor` | `LaboratoryWorkflow.perform_research` | phase state machine, restart policy, run lifecycle | `StartRun`, `Resume`, `Cancel`, `PhaseDone`, `ActorFailed` |
| `PhaseActor` (one per phase, spawned per run) | `literature_review`, `plan_formulation`, … | phase-local loop & step counter | `Begin`, `AgentReply`, `ToolResult`, `HumanDecision` |
| `AgentActor` × 5 (PhD, Postdoc, Professor, ML Eng, SW Eng) | `BaseAgent` & subclasses | role prompt, 15-entry history with `EXPIRATION` decay, notes, phase context | `Infer{phase, step, feedback, temp} → Reply`, `Reset` |
| `ReviewPanel` | `ReviewersAgent` (not a `BaseAgent`) | three fixed reviewer personas, each a `get_score` call | `Review{plan, report} → [ReviewScore; 3]` |
| `LlmGateway` | `inference.query_model` only | HTTP clients, retry, cost ledger, key config | `Complete{model, system, prompt, temp} → Result<String, LlmError>` |
| `SearchActor` | `ArxivSearch` (`SemanticScholarSearch` is dead code) | HTTP client, rate-limit sleeps | `FindPapers{query, n}`, `FullText{id}` |
| `DatasetSearchActor` | `HFDataSearch` (TF-IDF over `nkasmanoff/huggingface-datasets`) | dataset index, split-size lookups | `Retrieve{query}` |
| `ExperimentRunner` | `tools.execute_code` as called from `Replace.parse_command` / `Edit.execute_command` and from data preparation (`MLESolver.run_code` is dead code) | child-process lifecycle | `Run{code, timeout} → ExecResult` |
| `SolverActor` (MLE, Paper) | `MLESolver`, `PaperSolver` | code/paper state, edit/replace command application, scoring | `Solve`, `Step`, `Score` |
| `LatexActor` | `utils.compile_latex` | `pdflatex` subprocess | `Compile → Pdf \| Log` |
| `StateStore` | `save_state` (pickle) | append-only event log (JSONL) + snapshots | `Append(Event)`, `Load(run_id)` |
| `AgentRxivHub` | `AgentRxiv` + `app.py` | shared paper index across labs | `Upload`, `Search`, `Fetch` |
| `HumanGate` | `human_in_loop` | pending approvals | `Request`, `Respond` |

### 4.2 Design rules

- **Message-passing only.** No shared domain state; actors hold `Sender` handles, never references to each other's data.
- **Event sourcing for state.** Every completed step emits an immutable `Event` to `StateStore`; actor state is a fold over events. This replaces pickling, enables resume (G3), and makes golden-test assertions easy.
- **Pure core, effectful edge.** Everything that is pure Python logic today becomes a plain function/crate with no async and no I/O: prompt assembly (including the Python tuple/list `repr` quirks, §7.2 D9), fenced-command extraction (`extract_prompt` in `utils.py`, `extract_json_between_markers` in `agents.py`), `Edit`/`Replace` command parsing & application, line numbering (`generate_code_lines`/`generate_paper_lines`), history formatting and the 15-entry/`EXPIRATION` history cap, `format_review`/`add_review` parsing, arXiv query truncation (`_process_query`), `HFDataSearch.results_str`, review-score parsing and weighting, the MATH answer normalizer family (`strip_string`, `is_equiv`, `fix_fracs`, `fix_a_slash_b`, `remove_right_units`, `fix_sqrt`, `remove_boxed`, `last_boxed_only_string`, `clean_answer`; used only by generated code, so needed for the runner prelude until Phase 8). `clip_tokens`/`count_tokens` are never called and are ported only as dead-code parity if at all.
- **Typed errors** (`thiserror`) at every boundary; no stringly-typed `"[CODE EXECUTION ERROR]"` sentinels — those become enum variants rendered to text only when fed back to the LLM.
- **Bounded mailboxes + cancellation tokens** everywhere; graceful shutdown drains in-flight LLM calls.
- **Supervision:** transient failures (HTTP 5xx, timeout) retried inside the owning actor; panics/crashes restart the actor from the event log.

### 4.3 Proposed workspace layout

```
agentlab/                      (cargo workspace)
  crates/
    core/        pure domain: prompts, parsing, commands, scoring, MATH normalizer   (no async)
    protocol/    message & event types, serde
    llm/         LlmGateway + Backend trait + impls (openai, deepseek, anthropic, gemini)
    tools/       search, dataset search, experiment runner, latex
    actors/      supervisor, phases, agents, solvers
    store/       event log + snapshots
    hub/         AgentRxiv HTTP service (axum)
    cli/         arg/YAML parsing (clap, serde_yaml) → binary `agentlab`
  tests/
    golden/      recorded LLM transcripts ("cassettes") + expected outputs
    fixtures/
  experiments/   one dir per experiment: hypothesis.md, harness, results.md
```

---

## 5. TDD & Experiment Methodology

### 5.1 TDD rules (apply to every requirement)

1. **Red first.** No production code merges without a failing test that precedes it in git history (enforced by review convention: test commit, then implementation commit).
2. **Characterization before translation.** For every Python function ported, first capture its behavior as fixtures by running the *Python* implementation (the oracle) over real and adversarial inputs; the Rust test asserts identical output.
3. **Differential testing** for pure functions: property-based tests (`proptest`) generate inputs, run both Rust and the Python oracle (via a small harness script), and compare. Primary targets: `strip_string`/`is_equiv`, `extract_prompt`, `Edit` line-range semantics, `clip_tokens`.
4. **Actors tested through their mailbox.** Each actor test sends messages and asserts on replies/emitted events, with all collaborators replaced by test doubles (`FakeLlm` returning scripted responses, `FakeRunner`, `FakeSearch`). Time is controlled with `tokio::time::pause`.
5. **No real network in CI.** LLM and search responses are replayed from cassettes recorded once from the Python implementation (§5.3).
6. **Mutation testing** (`cargo-mutants`) run on `core` per phase gate to check tests actually bite.

### 5.2 Experiment protocol

Uncertain design/technology decisions are resolved by time-boxed experiments, not argument. Each experiment lives in `experiments/EXX-name/` with:

- `hypothesis.md` — what we believe, *and the pass/fail criterion written before running anything*
- a runnable harness (itself test-first where feasible)
- `results.md` — measured numbers, decision, date
- **Time-box:** ≤ 2 working days each. If inconclusive at the box, pick the simpler option and record why.

Experiment registry:

| ID | Question | Hypothesis | Pass criterion | Fallback |
|----|----------|------------|----------------|----------|
| **E1** | Hand-rolled Tokio actors vs `ractor` vs `kameo` | Hand-rolled is sufficient and smaller | Implement the `AgentActor`+`LlmGateway` slice in all three; compare LOC, test ergonomics (can we test a mailbox in <10 lines?), supervision support, compile time | Adopt `ractor` if supervision/linking needs exceed ~150 LOC hand-rolled |
| **E2** | Can `core` parsers match Python exactly? | Yes for all inputs in corpus | 100% match on ≥ 5k corpus samples + proptest differential run (≥ 100k cases) with 0 diffs, or each diff justified | Document intentional deviations |
| **E3** | Event log vs snapshot-only persistence | Event log gives needed resume with acceptable size | Crash-inject at every event boundary across a golden run; resume succeeds each time; log ≤ 50 MB for a full run | Snapshots at phase boundaries only |
| **E4** | Subprocess sandbox for generated Python (inside the E10 env) | Plain child process + `kill_on_drop` + `rlimit` is enough | Corpus of hostile snippets (infinite loop, fork bomb, huge stdout, `exit()`, `os.system`, memory balloon) all terminate within timeout+1 s, output capped at limit, host unaffected | Run inside container/`nsjail`/`bwrap` (Linux) — see Open Q |
| **E5** | LLM backend abstraction | One `Backend` trait covers OpenAI (incl. the o-series quirks: no `temperature`, system prompt concatenated into the single user message), DeepSeek (OpenAI-compatible, custom base URL), Anthropic, Gemini | Trait + 4 impls pass a shared contract test suite run against a mock HTTP server (`wiremock`); request bodies for every alias in §7.2 R2 are golden-tested | Per-backend request-shaping hook |
| **E6** | Mailbox backpressure & parallel labs | N parallel labs (AgentRxiv) scale without deadlock | Run 8 simulated labs (the `num-parallel-labs` default) against `AgentRxivHub` with `FakeLlm` for 10 min: 0 deadlocks, bounded memory, p99 hub latency < 50 ms | Single-writer hub with batched writes |
| **E7** | TF-IDF dataset search in Rust | `HFDataSearch` is scikit-learn `TfidfVectorizer` + `linear_kernel` cosine over dataset descriptions filtered by likes ≥ 3 and downloads ≥ 50, min-max normalised, weights `sim_w=1.0, like_w=0.0, dwn_w=0.0`; a Rust re-implementation of sklearn's default tokenizer (`(?u)\b\w\w+\b`, lowercase, smooth idf, sublinear off, l2 norm) reproduces the ranking | Top-10 list identical on ≥ 95% of 50 queries, same order on ≥ 80%; `load_dataset_builder` split-size lookups replicated via the HF Hub API | Keep it as a Python sidecar behind the `DatasetSearchActor` interface |
| **E8** | PDF text extraction | A Rust PDF crate matches `pypdf` (lit review) and `PyPDF2` (AgentRxiv hub) text closely enough, including the `--- Page n ---` markers and the 50 000-char cap | Normalized text similarity ≥ 0.9 on 30 arXiv PDFs | Shell out to `pdftotext` |
| **E12** | Sentence-embedding search for the AgentRxiv hub | `all-MiniLM-L6-v2` cosine ranking (`app.py`) can be served from Rust via `fastembed`/`ort` with the same model weights | Top-5 overlap with Python ≥ 90% on 50 queries over 100 uploaded PDFs | Hub keeps a Python sidecar for embeddings (dev-only Python until Phase 8), or swap to TF-IDF as a recorded deviation |
| **E9** | Resume determinism | Replaying the event log reproduces actor state bit-for-bit | Property test: fold(events) == live state for random event prefixes | Add explicit state snapshots |
| **E11** | Can agents write correct experiments in Rust? | With a curated crate set (`polars`, `ndarray`, `linfa`, `candle`/`burn`, `plotters`, `serde`), pass rate approaches Python's on the MATH config tasks | On a fixed task set (≥ 30 tasks from the cassettes): compile-and-run success within the existing step budget ≥ 90% of the Python pass rate; compile-error feedback loop converges (median ≤ 3 repair steps); per-experiment wall time incl. compile reported | Keep Python for ML-heavy tasks and scope G7 to orchestration; or provide a prebuilt "experiment SDK" crate wrapping the common patterns |
| **E10** | Nix (toolchain) + pixi (Python env) | Nix flake supplies Rust, `pdflatex`, `pixi`; `pixi.lock` supplies Python + numpy, pandas, sklearn, torch, datasets, matplotlib, … on macOS arm64 and Linux | `nix develop` + `pixi install --locked` succeed on both; the cassette experiment code runs unmodified; warm start < 5 s; lockfile reproducible across two machines. Also try the single-tool variants (Nix-only Python; pixi-only incl. `texlive-core`) and compare setup effort and reliability | Drop Nix (pixi + system TeX) or drop pixi (Nix Python) depending on which half fails |

### 5.3 Golden "cassette" corpus (the parity oracle)

Before any Rust is written: instrument the Python implementation to record, for several full runs (including `experiment_configs/MATH_agentlab.yaml` and an AgentRxiv run), every LLM request/response, search call, code-execution input/output, and phase output. These cassettes are the shared test fixture for every later phase. Because LLM output is nondeterministic, parity is asserted on **replay** (same responses in → same decisions/artifacts out), not on fresh runs.

Recording caveats found in the audit (§7.2): the shipped `MATH_agentrxiv.yaml` has no `agentRxiv:` key and sets `compile-latex: False`, so an AgentRxiv cassette needs a corrected YAML (`agentRxiv: True`, `compile-latex: True`, since upload copies `tex/temp.pdf`); copilot cassettes need `copilot_mode` spelled with an underscore; importing the Python oracle pulls in `common_imports.py` (torch, tensorflow, transformers, spacy, …) and `from app import *` loads a MiniLM model, so the oracle harness runs in the Pixi env and only its recorded outputs enter CI.

---

## 6. User Stories

**Maintainer / contributor**
- As a maintainer, I want every actor testable through its mailbox with fakes so that I can change a phase without running a real experiment.
- As a contributor, I want to add an LLM backend by implementing one trait so that I don't edit agent or phase code.
- As a maintainer, I want a typed message protocol so that illegal phase transitions fail to compile or fail fast in tests.

**Researcher (operator)**
- As a researcher, I want a crashed or interrupted run to resume from its last completed step so that I don't repay LLM cost.
- As a researcher, I want to see an ordered event log of what each agent did so that I can audit and debug a run.
- As a researcher, I want per-run cost and token accounting so that I can budget runs.
- As a researcher, I want to approve/redirect at phase boundaries (human-in-the-loop) exactly as today.
- As a researcher running several labs, I want parallel labs sharing papers via AgentRxiv so that agents build on each other's work.

**Security-minded operator**
- As an operator, I want generated code to run with enforced time, memory, and output limits so that a bad generation cannot hurt my machine.

---

## 7. Requirements

### P0 — Must have (parity & foundations)

**R0. Parity harness and cassettes** (blocks everything)
- [ ] Python instrumented to record cassettes for ≥ 3 full runs, including one per paper research topic (Appendix A.2) and the 10 MLE-Bench tasks (A.3)
- [ ] Oracle script exposes each pure Python function over stdin/stdout for differential tests
- *Given* a cassette, *when* the Rust pipeline replays it, *then* phase artifacts equal the recorded ones (or listed deviations)

**R1. `core` crate — pure logic ported test-first**
- [ ] `extract_prompt` (joins *all* matching fenced blocks with `\n`, then `strip()`; the fence word is interpolated unescaped into the regex), `extract_json_between_markers` (a `json` fence first, then a non-greedy `\{.*?\}` fallback that fails on nested JSON; control characters stripped on retry), command parsing, `Edit`/`Replace` application, line numbering, history formatting, review-score weighting, MATH normalizer (§4.2 list)
- [ ] `Edit N M` semantics: `N`/`M` parsed from the first line of the fence; lines `N..=M` removed and the new lines inserted at `N`; out-of-range → failure, negative indices silently accepted (Python list semantics); `EDIT` wins when both fences are present
- [ ] Edge cases explicitly tested: nested/unterminated fences, multiple fences, empty bodies, out-of-range and negative `EDIT` indices, unicode, `\boxed{}` nesting, `python` fences rewritten to `REPLACE` by `MLESolver.clean_text`
- [ ] Differential/property tests green (E2)

**R2. `LlmGateway` with backend trait**
- [ ] Alias → pinned-model table exactly as `inference.query_model`: `gpt-4o-mini`/`gpt4omini`/`gpt-4omini`/`gpt4o-mini` → `gpt-4o-mini-2024-07-18`; `gpt4o`/`gpt-4o` → `gpt-4o-2024-08-06`; `o1-mini` → `o1-mini-2024-09-12`; `o1` → `o1-2024-12-17`; `o1-preview` → `o1-preview`; `o3-mini` → `o3-mini-2025-01-31`; `deepseek-chat` → `deepseek-chat` at `https://api.deepseek.com/v1`; `claude-3.5-sonnet` → `claude-3-5-sonnet-latest`; `gemini-1.5-pro` → `gemini-1.5-pro`; `gemini-2.0-pro` → `gemini-2.0-pro-exp-02-05`. Newer models are added as config entries, not aliases in code
- [ ] Request shaping: `temperature` sent only for gpt-4o, gpt-4o-mini and deepseek-chat and only when set; o-series send one `user` message containing `system_prompt + prompt` (no system role); Anthropic and Gemini pass the system prompt natively. The Anthropic path must add `max_tokens` (missing today, so that path raises on every call: recorded deviation D4)
- [ ] Retry semantics: Python is 5 tries with a fixed 5 s sleep, catching every exception, and `tries` is never overridden by a caller; that is the parity baseline. Exponential backoff on 429/5xx and cancellation-awareness are allowed as an additive deviation that cannot change the request sequence on replay
- [ ] API keys passed via config/env at construction, never written to process env at call time; YAML `api-key`/`deepseek-api-key` and env `OPENAI_API_KEY`/`DEEPSEEK_API_KEY` (env wins) are the only key sources today; Anthropic/Gemini keys gain config entries (additive)
- [ ] Cost/token ledger replaces the global `TOKENS_IN`/`TOKENS_OUT` dicts and `curr_cost_est`: per-run, per-model, thread-safe; USD per 1M tokens in/out as in `inference.py:13-32` (gpt-4o 2.50/10.00, gpt-4o-mini 0.15/0.60, o1-preview 15/60, o1-mini 3/12, o1 15/60, o3-mini 1.10/4.40, deepseek-chat 1/5, claude-3-5-sonnet 3/12) stored in `params.yaml`; token counts via `tiktoken` cl100k/o200k-compatible counting (Rust `tiktoken-rs`). Python's ledger silently skips claude (cost-map key mismatch), deepseek (invalid encoding name) and gemini (unknown model); the Rust ledger counts all of them and the difference is a recorded deviation D5
- *Given* a backend returns 429 twice then 200, *then* caller receives the 200 body and ledger records 3 attempts
- *Given* `o1-mini`, *then* the request has no `temperature` and a single user message beginning with the system prompt

**R3. `StateStore` event log + resume** (E3, E9)
- [ ] JSONL append with fsync policy; schema-versioned events
- [ ] Crash-injection test at every event boundary resumes successfully
- [ ] Old Python pickles are **not** loaded: there is no loading code in the repository (the README's "load-existing" tip and the YAML `load-existing`/`load-previous` keys are inert), so nothing is lost
- [ ] Per-subtask timing in events matches Python's `statistics_per_phase` semantics, including the quirk that time is measured from the *phase-group* start (so "running experiments" includes data preparation). The Rust store records both the raw quirk-compatible value and a corrected per-subtask value (deviation D10)

**R4. Agent actors**
- [ ] Five role actors (`PhDStudentAgent`, `PostdocAgent`, `ProfessorAgent`, `MLEngineerAgent`, `SWEngineerAgent`) plus the `ReviewPanel` (`ReviewersAgent`), with per-phase context/prompt/commands matching Python byte-for-byte (prompts live as versioned template files, not string-concatenation in code)
- [ ] The exact `BaseAgent.inference` layout: system = `You are {role} \nTask instructions: {phase_prompt}\n{command_descriptions}`; user = `{context}\n~~~~~~~~~~\nHistory: {history}\n~~~~~~~~~~\nCurrent Step #{step}, Phase: {phase}\n{complete_str}\n[Objective] Your goal is to perform research on the following topic: {topic}\nFeedback: {feedback}\nNotes: {notes}\nYour previous command was: {prev_comm}. Make sure your new output is very different.\nPlease produce a single command below:\n`; `complete_str` is the paper's completion nag when `step/(max_steps-1) > 0.7`; `clean_text` removes the newline that follows a closing fence
- [ ] Python `repr` quirks reproduced verbatim: `context()` returns tuples, `sr_str` is a tuple on the second round, `MLESolver.phase_prompt` is a one-tuple, `notes_str` is a `list[dict]` repr, `PaperSolver` interpolates `str(lit_review)[:20000]`. A `PyRepr` formatter in `core` is differential-tested against Python (D9)
- [ ] History: at most 15 entries (`pop(0)` when full) plus `EXPIRATION n` countdown entries removed when their counter goes below 0; `Reset` clears history and `prev_comm` only (plan, lit review, etc. persist); the shared `notes` list and `human_in_loop` note appends covered by tests
- [ ] Command fences per role and phase as in A.1; the Professor's `LATEX`/`DIALOGUE` commands are described in its prompt but never parsed (report writing has no agent dialogue): keep the prompt text, parse nothing
- [ ] `ReviewPanel`: three `get_score` calls with the three fixed reviewer persona strings, joined as `Reviewer #1:\n{r1}, \nReviewer #2:\n{r2}, \nReviewer #3:\n{r3}` where each `rN` is the Python tuple repr `(score, text, True)` (D9)

**R5. Phase actors and `Supervisor`**
- [ ] Explicit enum state machine for the 7 subtasks in their 4 phase groups (§1); human-in-the-loop is a per-subtask gate flag (all seven set from one `copilot_mode` value), not a state
- [ ] Loop semantics: `while repeat: repeat = subtask()`, a human rejection re-runs the whole subtask; `set_model(phase_models[subtask])` before each subtask; `save_state` after each
- [ ] Hard-coded `max_steps = 100` bounds literature review, plan formulation, data preparation and results interpretation; exhaustion behaviour differs per subtask and is preserved: plan formulation → raise if `except-if-fail` else `plan = "No plan specified."`; literature review → raise if `except-if-fail` else finish if enough papers, otherwise return with no summary set; data preparation and results interpretation always raise
- [ ] Refinement path: when report refinement asks to go back, the supervisor sets `second_round` and the four `prev_*` snapshots on every role agent and reopens **plan formulation** through report refinement (not literature review, and not "experiments" only), **without recursion**
- [ ] Known Python behaviour reproduced in parity mode and fixed otherwise (§7.2): with defaults (`review_override = True`, `review_total_steps = 0`) refinement always answers "n", so the loop is unreachable in autonomous mode (D6); in copilot mode the human's answer is discarded and the subtask returns `None`, treated as "finished" (D7); the running-experiments human gate is labelled `"data preparation"`, so rejection notes land on the wrong phase (D8)
- [ ] Per-phase model map: today all seven entries are the single `llm-backend` value, `lit-review-backend` is parsed but unused, and the key `"paper refinement"` is misspelled so report refinement (and the three reviewers) always run on the default `o3-mini` (D1). Rust accepts a true per-phase map; parity mode reproduces D1 exactly
- [ ] Illegal transition → typed error, tested

**R6. `ExperimentRunner`** (E4)
- [ ] `PythonEnv` provider trait (Pixi default, Uv, Container); runner is provider-agnostic
- [ ] Prelude `from utils import *` reproduced as a pinned `agentlab_prelude.py` in the Pixi env exposing what generated code relies on today: `query_gpt4omini`/`query_gpt4o`/`query_deepseekv3`/`query_qwen`/`query_gemini*` (note the Python signature bug D11: `api_key` is a required positional argument, so the MATH YAML's documented `query_gpt4omini(prompt=..., system=...)` raises), the MATH helpers, `compile_latex`, and `matplotlib.use('Agg')`
- [ ] Timeouts by call site: 60 s for `SUBMIT_CODE` in data preparation, 120 s for ML-engineer `python` fences (prefixed with the submitted dataset code), 600 s for mle-solver runs (`dataset_code + "\n" + code`)
- [ ] String guards kept for parity: `load_dataset('pubmed` and `exit(` return the exact Python error strings; sandbox limits (R13) are the real defence
- [ ] Output: stdout captured (stderr is not captured in Python; capturing it is deviation D12); Python has **no** output cap and deadlocks on output larger than the OS pipe buffer, reporting a timeout (D13). Rust streams output and caps it at a configured size, recording the cap as a deviation visible in the event log
- [ ] Error results are a typed `ExecResult` enum (`Ok`, `Error{traceback}`, `Timeout`, `Rejected{guard}`, `MissingDependency`) rendered to the Python strings (`[CODE EXECUTION ERROR]: …`, the timeout message) only at the LLM boundary
- [ ] Side effects preserved: `remove_figures()` deletes `*Figure_*.png` in the run directory before every mle-solver command; generated figures and `downloaded-paper.pdf` are per-run paths in Rust, not CWD (D14, fixes the parallel-lab clobbering)
- [ ] Hostile-snippet corpus (seeded from A.8) passes

**R7. Solver actors (MLE, Paper) + Latex**
- [ ] `MLESolver` as in `mlesolver.py`: `initial_solve` loops **unbounded** until a scored program exists (error history capped at 5); `solve()` runs `mlesolver-max-steps − 1` times, each picking `random.choice(best_codes)`, prompting with the 2-entry `st_history` at temp 1.0, trying ≥ `min_gen_trials = 1` attempts until a valid score, keeping the best, replacing `best_codes[-1]` only when better, and calling `reflect_code()` whenever the pool is full (always, since `max_codes = 1`); `process_command` tries `[Edit, Replace]` and up to `GLOBAL_REPAIR_ATTEMPTS = 2` repair rounds (`code_repair`, temp 0.8 for `REPLACE` and 0.2 for `EDIT`); reward model `get_score` is a float 0–1 in a `SCORE` fence at temp 0.6, single attempt. Result strings (`Code was successfully edited.`, `Code editing FAILED due to the following error: …`, `Command not supported, choose from existing commands`) byte-identical
- [ ] `PaperSolver` as in `papersolver.py`: `initial_solve` uses `PaperReplace` only; `gen_initial_report` builds `scaffold, abstract, introduction, related work, background, methods, experimental setup, results, discussion` in that order, with an LLM-generated arXiv query (≤ 6 attempts, N = 10) for introduction, related work, background, methods and discussion, replacing `[SECTION HERE]` placeholders, retrying while `documentclass{article}`/`usepackage{` appear, looping unbounded until the section compiles; `solve()` runs `papersolver-max-steps` times (no `−1`), `random.choice(best_report)`, prompt `\nNow please enter a command: ` at temp 1.0, ≥ `min_gen_trials = 2` attempts, **EDIT only** (Replace is commented out), no history, no repair loop; score = one NeurIPS review via `agents.get_score` with weights clarity .1, quality .1, overall 1.0, soundness .1, confidence .1, originality .1, significance .1, contribution .4, presentation .2, normalised by 2.2 × 10, temp 0.0, single attempt; `\includegraphics{Figure_N.png}` rewritten to absolute paths; `PaperEdit` success test is `"error" in latex_ret.lower()`; system prompt includes the word-count nag toward 4000 words and figure-include instructions when `Figure_1.png`/`Figure_2.png` exist
- [ ] Title regex `\\title\{([^}]*)\}` with a 10-random-digit fallback; `generate_readme()` (strips the `markdown` fence); outputs `readme.md`, `report.txt`, `src/load_data.py`, `src/run_experiments.py` (model code without the dataset prelude), `experiment_output.log`, `tex/temp.tex`, `tex/temp.pdf`
- [ ] Single shared `Command` implementation (removes current duplication); the unused `Arxiv` command class in `papersolver.py` is dropped
- [ ] `LatexActor` = `compile_latex`: injects the fixed 30-line `\usepackage` block after `\documentclass{article}`, always writes `tex/temp.tex` (even with compilation off, returning `Compilation successful`), runs `pdflatex -interaction=nonstopmode temp.tex` in the tex dir with a 30 s timeout, returns the three exact Python strings; a missing `pdflatex` is a typed error instead of the uncaught `FileNotFoundError` (D15)
- [ ] `report_writing` resets `reference_papers` to `[]` before building the solver, so `ref_papers` is always empty (D16: fixed by default, empty in parity mode)

**R8. Search tools**
- [ ] arXiv as in `ArxivSearch`: query truncated at a word boundary to 300 chars, `abs:` prefix, relevance sort, `N` = 5 from the lit review and 10 from the paper solver (20 is only the unused default), 3 retries with `sleep(2 × retry)`, `sleep(2.0)` after every call (rate-limit courtesy, kept), `None` on total failure; summary format `Title: …\nSummary: …\nPublication Date: YYYY-MM-DD\narXiv paper ID: …\n` joined by `\n`
- [ ] Full text: download, `--- Page n ---` markers, `EXTRACTION FAILED` on a page error, cap 50 000 chars; the lit-review feedback wraps it in an `EXPIRATION 3` fence; `ADD_PAPER` re-downloads the PDF (D17: cached by default, re-fetched in parity mode)
- [ ] Semantic Scholar is dead code (never instantiated) and is **not** ported; listed as a P2 backend
- [ ] Response caching is additive and keyed so replay is unaffected
- [ ] AgentRxiv client: `num_papers()` = count of files in `uploads/`; search = `GET /api/search?q=…` on the lab's hub (port `5000 + lab-index`), download of each hit to a per-lab temp file, 5-sentence summary with a hard-coded `gpt-4o-mini`; ids `AgentRxiv:ID_{n}`; lit review branches on the substring `"AgentRxiv"` in the query

**R9. CLI/config parity**
- [ ] The only CLI flag today is `--yaml-location` (default `experiment_configs/MATH_agentlab.yaml`); README examples with `--llm-backend`/`--compile-latex` are stale. Rust keeps `--yaml-location`, adds `--parity-mode` (also `parity_mode: true` in YAML; default off; see §7.2), and may add flags that mirror YAML keys
- [ ] YAML keys consumed, with defaults, accepted unchanged: `copilot_mode` (False; **underscore**, while both shipped YAMLs write `copilot-mode`, so copilot never switches on: D2), `research-topic`, `api-key`, `deepseek-api-key`, `compile-latex` (True), `llm-backend` (`o3-mini`), `lit-review-backend` (`gpt-4o-mini`, unused), `language` (`English`; non-English appends a translation note to all seven phases), `num-papers-lit-review` (5), `mlesolver-max-steps` (3), `papersolver-max-steps` (5), `task-notes` (keys mapped with `-`→space), `num-papers-to-write` (100), `parallel-labs` (False), `num-parallel-labs` (8), `except-if-fail` (False), `agentRxiv` (False), `construct-agentRxiv` (unused), `agentrxiv-papers` (5), `lab-index` (0), `load-previous` (unused; the YAMLs write `load-existing`). The string `"true"` is accepted for booleans
- [ ] Rust accepts both spellings for the mismatched keys and warns; parity mode reproduces the Python reading (D2, D3)
- [ ] Output layout preserved: `MATH_research_dir/research_dir_{paper}_lab_{lab_index}/{src,tex}`, `agent_times_{lab_index}.txt`, `state_saves/`; the research dir is removed at start unless `agentRxiv` is on

### 7.1 R16 — Results & performance tracking (Git + DVC)

Principle: **git stores code and small text; DVC stores data, artifacts, and the numbers that prove a change helped or hurt.** Every commit that touches behavior carries its measured effect.

- [ ] `dvc.yaml` pipeline with stages: `record_cassettes` → `replay_parity` → `unit_bench` → `live_run` (opt-in, costs money) → `report`
- [ ] `params.yaml`: llm backend/model per phase, temperature, seed, N runs, timeouts, dataset slice — one source of truth, diffable in git
- [ ] **Metrics** (`metrics/*.json`, tracked by git, produced by stages): replay parity %, deviation count, MATH accuracy, review score, tokens in/out, USD cost, wall time per phase, retries, crash-resume success, peak RSS
- [ ] **Paper-derived metrics (arXiv 2501.04227), required in `metrics/`:**

  | Group | Metric | Paper definition / value to reproduce |
  |-------|--------|----------------------------------------|
  | Runtime | Wall time per subtask and per workflow (s) | 7 subtasks; gpt-4o 1165.4 s, o1-mini 3616.8 s, o1-preview 6201.3 s total |
  | Cost | USD per subtask and per workflow | gpt-4o $2.33, o1-mini $7.51, o1-preview $13.10 |
  | Reliability | Success rate per subtask; average subtask success rate | gpt-4o 94.3%, o1-mini 92.8%, o1-preview 95.7%; lit review 60/70/80%, data prep 100/80/90% |
  | Auto review | Overall (1–10), Originality, Quality, Significance, Clarity, Soundness, Presentation, Contribution (1–4 each), Confidence (1–5), Decision | NeurIPS-style LLM reviewer adapted from The AI Scientist; 1 reviewer inside paper-solver, 3 in refinement; automated overall ≈ 6.1 |
  | Auto review | mle-solver internal score (0–1, LLM reward model on the dev split) | Selects top programs each step |
  | Human eval | Experiment quality, report quality, usefulness (1–5) | gpt-4o 2.6/3.0/4.0; o1-mini 3.2/3.2/4.3; o1-preview 2.9/3.4/4.4 |
  | Human eval | Same NeurIPS-style criteria as above, scored by humans | Overall 3.5 / 3.8 / 4.0 (gpt-4o / o1-mini / o1-preview) |
  | Human vs auto | Overall score gap (auto − human) | −2.3 points; clarity 3.6 vs 2.4 |
  | Reference | Gap to mean NeurIPS 2024 accepted paper (5.85) | Autonomous ≈ −2.05, co-pilot ≈ −1.45 |
  | Co-pilot | Utility, continuation, satisfaction, usability (1–5) | 3.5 / 3.75 / 3.63 / 4.0; self vs external review (4.13 vs 4.38 overall) |
  | MLE-Bench | Medals (gold/silver/bronze), count above human median, valid-submission rate, score per challenge | mle-solver 4 medals, 6/10 above median, valid on 10/10; 10 text/tabular low-complexity tasks, baselines MLAB/OpenHands/AIDE |
  | Config | Table 1 hyperparameters recorded as `params.yaml`, paper value and repository value side by side (A.10) | e.g. lit-review summaries 5, mle-solver steps 3, repair attempts 2, max top codes 2 (paper) vs 1 (code), lit-review temp 0.8 (paper) vs 0.4 (code), 1 vs 3 reviewers |

  Human-evaluation rows are collected out-of-band (surveys) and stored via DVC as CSV; all other rows are produced automatically by the pipeline. The MATH accuracy metric comes from this repo's AgentRxiv config (not the original paper) and is kept as an additional metric.
- [ ] **Plots** (`dvc plots`): per-phase latency, cost, and score across commits
- [ ] **Artifacts in DVC** (not git): cassettes, run event logs, generated code/papers/PDFs, Python baseline runs; remote storage configured (S3/GCS/other — Open Q)
- [ ] **Baseline lineage:** Python baseline results are committed once (tag `py-baseline`) and never regenerated; each later milestone has a tag (`rust-parity`, `rust-native-exp`, …) so `dvc metrics diff py-baseline rust-parity` is the headline comparison
- [ ] **Experiments:** each E1–E11 runs as a `dvc exp` so alternatives are compared side by side and the winner is applied to the branch (`dvc exp apply`)
- [ ] **CI:** every PR runs `dvc repro` on the offline stages and posts `dvc metrics diff` + `dvc params diff` against `main` as a PR comment; a regression beyond a configured threshold fails the check
- [ ] **Benchmarks** for the Rust hot paths via `criterion`, results exported into `metrics/` so they diff like everything else
- [ ] `docs/results/` is generated (never hand-edited): a table per milestone linking commit, params, and metrics
- *Given* two commits, *when* a developer runs `dvc metrics diff <a> <b>`, *then* every metric above is shown with absolute and relative change
- *Given* a PR that changes a prompt, *then* CI shows its parity and score delta before review

- [ ] **R19. Paper metrics suite (P0 for Phase 0 baseline):** every metric in the §7.1 table and Appendix A is emitted by the pipeline (automated rows) or ingested from CSV in the A.9 schema (human rows) and recorded for the `py-baseline` tag before any Rust ships

### 7.2 Deviation register (P0; owned by R0)

The Python implementation has defects that a byte-identical port would reproduce. Decision (Q5e): **every row is fixed in Rust, and every fix sits behind one switch, `--parity-mode`** (`parity_mode: true` in YAML), which reproduces the Python behaviour exactly. Default off. Each row has a characterization test that runs twice, once per mode, asserting the Python behaviour in parity mode and the corrected behaviour otherwise. The register is the only place a difference from Python is allowed to live, and no row may be fixed without its switch.

Disposition column meaning: **Fixed** = default behaviour differs from Python, parity mode restores it. **Preserved** = identical in both modes (no defect, or a quirk that is cheaper to keep); the row stays here because it looks like a bug to a reader. **Replaced** = the Python mechanism does not exist in Rust at all (no parity-mode equivalent is possible, so the cassette asserts only the observable outputs). **Not ported** = dead code.

Rows marked Fixed that change which LLM calls are made (D1, D2, D6, D7) also change the request sequence, so a cassette recorded in one mode only replays in that mode; cassettes carry their mode in their header.

| ID | Python behaviour (file:line) | Disposition |
|----|-------------------------------|-------------|
| D1 | `agent_models` key `"paper refinement"` instead of `"report refinement"` (`ai_lab_repo.py:781`) → report refinement and reviewers always use `o3-mini` | Fixed: per-phase map honoured. Parity: refinement and reviewers forced to `o3-mini` |
| D2 | `copilot_mode` read with underscore (`:671`); shipped YAMLs use `copilot-mode` → copilot never enabled | Fixed: both spellings accepted. Parity: only the underscore key is read, hyphen key ignored with a warning |
| D3 | `load-existing` (YAML) vs `load-previous` (code), `construct-agentRxiv`, `lit-review-backend` all inert; `lab_index` int-conversion checks the wrong variable (`:727`) | Fixed: `lit-review-backend` honoured, inert keys warn. Parity: all ignored |
| D4 | Anthropic call omits `max_tokens` (`inference.py:99-102`) → raises on every call | Fixed: `max_tokens` sent. Parity: request rejected with the same error class before any HTTP call |
| D5 | Ledger skips claude (key mismatch), deepseek (`encoding_for_model("cl100k_base")` is invalid), gemini (unknown model) | Fixed: all models counted. Parity: those three models record zero tokens |
| D6 | `review_override = True`, `review_total_steps = 0` → refinement loop unreachable in autonomous mode (`:45-46, 220-225`) | Fixed: the PhD agent is asked, as the paper describes; `review_total_steps` configurable. Parity: override on, answer is always "n" |
| D7 | Copilot answer in `report_refinement` discarded; function returns `None` (`:214-216`) | Fixed: "y" reopens plan formulation. Parity: answer discarded, treated as finished |
| D8 | Running-experiments human gate labelled `"data preparation"` (`:333`) | Fixed: note attached to running experiments. Parity: attached to data preparation |
| D9 | Prompts contain Python tuple/list reprs (`agents.py:373-385, 461-464, 588-617`; `mlesolver.py:517-522`; `papersolver.py:522`) | Fixed: plain text. Parity: `PyRepr` formatter reproduces the repr byte-for-byte |
| D10 | Subtask time measured from phase-group start (`:145, 203-205`) | Fixed: true per-subtask time. Both values are always emitted; parity mode reports the quirk value as the headline |
| D11 | `utils.query_*` have `api_key` positional-required and pass `attempt+1` as the key on retry (`utils.py:26-124`) → retries always fail; MATH YAML's documented call raises | Fixed in the prelude. Parity: prelude keeps the Python signatures |
| D12 | stderr not captured by `execute_code` | Fixed: captured separately. Parity: stderr discarded from the LLM feedback |
| D13 | No output cap; large stdout deadlocks and is reported as a timeout (`tools.py:313-325`) | Fixed: streamed and capped. Parity: outputs over the pipe-buffer size surface as `Timeout`, never truncated |
| D14 | Figures, `downloaded-paper.pdf`, `_tmp_{lab}.pdf` written to CWD and shared across threads | Fixed: per-run directories. Parity: same per-run directories (path layout is not observable by the LLM except in `\includegraphics`, which is rewritten to absolute paths anyway) |
| D15 | Missing `pdflatex` → uncaught `FileNotFoundError` (`utils.py:150`) | Fixed: typed error. Parity: same typed error (a crash has no replayable equivalent) |
| D16 | `reference_papers` reset to `[]` before `PaperSolver` (`:250`) | Fixed: first full text passed as `ref_papers`. Parity: empty |
| D17 | `ADD_PAPER` re-downloads the PDF (`agents.py:714-731`) | Fixed: cached from the `FULL_TEXT` call. Parity: re-fetched (cassette records both fetches) |
| D18 | `parallel-labs` branch is a `ThreadPoolExecutor` ending in `NotImplementedError` (`:788-828`) | Replaced by R10 |
| D19 | `MLESolver.run_code`/`feedback`/`reflection`, `SemanticScholarSearch`, `papersolver.Arxiv`, `clip_tokens`, `count_tokens`, `requirements_txt` are dead code | Not ported |
| D20 | `report_writing` reads module globals `research_topic`/`compile_pdf` (`:251`) | Fixed: passed explicitly. Parity: identical (not observable) |
| D21 | `save_state` pickles after every subtask; no loader exists | Replaced by R3 |
| D22 | AgentRxiv hub recomputes MiniLM embeddings for every stored paper on every query; `papers.db` wiped at start; `flask`, `flask_sqlalchemy`, `sentence-transformers`, `tensorflow` absent from `requirements.txt` | Fixed in R10 (embed once per upload, persistent index). Parity: identical ranking, since the scores are the same either way; Pixi env pins the missing packages |

### P1 — Nice to have (fast follow)

- **R10. `AgentRxivHub`** over axum with parallel labs (E6, E12), replacing the Flask `app.py` server: wire-compatible `GET /api/search?q=` returning `{query, results: [{id, filename, similarity, pdf_url}]}` sorted by descending cosine similarity over `all-MiniLM-L6-v2` embeddings, `/uploads/<file>`, `/update`; embeddings computed once per upload, not per query (D22); one hub per `lab-index` on port `5000 + lab-index` for the multi-process mode, or one shared hub for in-process labs
- **R11. `DatasetSearchActor`** native TF-IDF implementation (E7), otherwise Python sidecar
- **R12. Structured tracing** (`tracing` + JSON) and a `agentlab events <run>` inspector command
- **R13. Stronger sandbox** (container / `bwrap`) if E4 shows plain subprocess is inadequate
- **R16. Git + DVC results tracking** (see §7.1) — promoted to P0 for Phase 0
- **R17. Rust-native experiments:** agents generate Rust (cargo project) instead of Python; runner compiles and runs under the same limits (E11)
- **R18. Remove remaining Python:** dataset search, PDF reading, MATH normalizer oracle, bindings decision
- **R15. Python bindings:** PyO3/maturin module exposing run/resume/inspect so existing Python callers keep working
- **R14. Budget guard:** abort when ledger exceeds a configured spend

### P2 — Future considerations (design must not preclude)

- Pluggable phases defined in config (custom pipelines)
- Distributed actors across machines (message types are already `Serialize`)
- MCP/tool-protocol based tools instead of fenced-command parsing
- Local-model backends (llama.cpp / Ollama) and Qwen via HF Inference (today only `utils.query_qwen`, reachable from generated code, never from the agents)
- Semantic Scholar as a second `SearchActor` backend (dead code today)
- Web dashboard fed by the event stream

---

## 8. Phasing, Gates, and Timeline

Each phase ends with a **gate**: all tests green, experiment results committed, and a short review. A failed gate blocks the next phase.

| Phase | Content | Gate (exit criteria) | Rough effort |
|-------|---------|----------------------|--------------|
| **0 – Oracle** | R0: cassettes, oracle harness, workspace skeleton, CI with fake-only tests; §7.2 characterization tests in Python; `py-baseline` and `py-baseline-fixed` recorded; Fig. 5–9 values transcribed into Appendix A | ≥ 3 cassettes recorded; every D-row has a failing-then-passing characterization test; CI green on empty workspace; E1 decided | 1.5 wk |
| **1 – Pure core** | R1 + E2 | 0 unexplained diffs; mutation score on `core` ≥ 80% | 1–1.5 wk |
| **2 – Edges** | R2 (E5), R6 (E4), R8, E8, LaTeX | Contract tests green against mock HTTP; hostile corpus passes | 2 wk |
| **3 – Actors & state** | R3 (E3, E9), R4, R5, R7 | Full single-lab pipeline replays one cassette end-to-end offline; crash-injection green | 2–3 wk |
| **4 – CLI parity & live shake-out** | R9; run Rust and Python side by side on the same MATH config with real models; compare artifacts/quality/cost | Rust completes a full real run; live-run outcomes are statistically indistinguishable from Python (see §9 Q4 resolution) | 1 wk |
| **5 – Parallel labs & extras** | R10, R11 (E6, E7, E12), P1 items | 8-lab soak test passes; hub wire-compatibility contract tests green | 1–2 wk |
| **6 – Cutover (orchestrator)** | Docs; Rust is the product; Python orchestrator deleted | Sign-off; `py-baseline` vs `rust-parity` report published | 0.5 wk |
| **7 – Rust-native experiments** | E11; experiment SDK crate; `RustRunner` (cargo build + run under same limits); prompts for Rust codegen; Rust dataset search/PDF/embedding (E7/E8 resolved natively) | Rust-native pass rate and quality within the Phase 4 band, per DVC metrics | 4–6 wk (high uncertainty) |
| **8 – Python removal** | Delete Pixi env, `PythonEnv::Pixi/Uv`, oracle harness, remaining sidecars; decide on bindings (R15) | G7 met; final `dvc metrics diff` vs `py-baseline` published | 1 wk |

Total ≈ 9–11 weeks for one engineer to Phase 6, plus ≈ 5–7 weeks for Phases 7–8 (E11-dependent); Phases 1 and 2 can overlap with two people. Python remains the source of truth until the Phase 4 gate.

---

## 9. Success Metrics

**Leading (measured during the rewrite)** — all paper-derived metrics in §7.1 / Appendix A are recorded for every milestone via DVC

- Cassette replay parity rate: target 100% (exact); deviations only via the signed-off deviation log
- Differential-test diffs: 0 unexplained
- `core` line coverage ≥ 90%, mutation score ≥ 80%
- CI wall time for full offline suite: < 60 s
- Crash-injection resume success: 100% of boundaries

**Lagging (after cutover, 1–3 months)**
- Mean time to add an LLM backend: ≤ 1 hour / one trait impl + one config entry (today: a new `elif` branch, a cost-map entry and a tiktoken mapping in `inference.py`)
- Wasted LLM spend on crashed runs: ↓ ≥ 80% vs. Python baseline (resume)
- Defects escaping to real runs per month: tracked, trending down
- Paper-metric deltas vs. `py-baseline` (Appendix A): wall time and cost per subtask, success rates, automated review scores, MLE-Bench medals — each within the Python run-to-run band (Q4), none worse
- Real-run end-to-end wall time on the MATH config: ≤ Python (target ≥ 1.0× parity; the LLM dominates latency, so no speedup is promised)

---

## 10. Open Questions

**Blocking**
1. ~~**[Owner]** Replacement or dual implementation?~~ **Resolved:** full replacement. The Python oracle is scaffolding and is deleted at cutover (Phase 6). A thin Python interface (PyO3/maturin bindings exposing `agentlab` as a Python module) is allowed if needed; tracked as P1 R15.
2. ~~**[Owner]** Python runtime for generated experiment code~~ **Resolved (pending E10):** Nix + pixi. A Nix flake pins the toolchain (Rust, TeX Live for `pdflatex`, `pixi`); pixi owns the Python environment via a committed `pixi.lock` (conda-forge, with PyPI fallback through its uv integration). `ExperimentRunner` takes a `PythonEnv` provider trait with impls `Pixi` (default Python), `Uv` (no-Pixi fallback) and `Container` (opt-in hardening). Embedded Python (PyO3) is rejected for execution.
3. ~~**[Engineering]** Which LLM backends are mandatory?~~ **Resolved:** all of them. Audit correction: the agents' `query_model` has no Qwen path (Qwen exists only in `utils.query_qwen`, callable from generated code), so P0 for R2 is OpenAI (gpt-4o, gpt-4o-mini, o1, o1-mini, o1-preview, o3-mini), DeepSeek, Anthropic (claude-3.5-sonnet), Gemini (1.5-pro, 2.0-pro); Qwen is kept in the runner prelude (R6) and is a P2 agent backend.
4. ~~**[Owner]** Quality parity tolerance~~ **Resolved:** exact, or as close to exact as possible. Exactness is enforced where it is achievable: replay parity is 100% (byte-identical prompts, requests, parsed commands, artifacts). Live runs cannot be bit-exact (LLM nondeterminism, even Python vs. Python), so the bar is: Rust's outcome distribution is indistinguishable from Python's. Method: run Python N times to measure its own run-to-run variance (MATH accuracy, score, cost), then require Rust's results to fall within that observed band over N runs, with temperature pinned and seeds set wherever the backend supports it. Choose N (suggest ≥ 5 each) once the Python baseline variance is measured in Phase 0.

5a. **[Owner]** DVC itself is a Python tool. Accept it as dev/CI-only tooling (not part of the shipped product, so G7 still holds), or require a non-Python alternative? Assumed: accepted.
5b. **[Owner]** DVC remote storage location and budget for cassettes/artifacts (S3, GCS, SSH, local)?
5c. **[Engineering]** Phase 7 risk: Rust ML ecosystem lacks parity with sklearn/torch for arbitrary agent-written experiments. Is it acceptable that the experiment domain narrows (e.g., classical ML + small neural nets) if E11 shows a gap?
5d. **[Owner]** Is the Python-bindings interface (R15) still wanted once the end state has no Python? Assumed: only until Phase 8, then drop or keep as a published wheel.

5e. ~~**[Owner]** Deviation dispositions in §7.2~~ **Resolved:** fix every defect in Rust, behind a `--parity-mode` switch that reproduces the Python behaviour. Default is fixed; parity mode is used for cassette replay against `py-baseline` and is what G1's byte-identical claim is measured in. See §7.2 for the rule.

**Non-blocking**
5. ~~**[Engineering]** Is loading existing `state_saves/*.pkl` a requirement?~~ **Resolved by audit:** no loader exists in the repository, so there is nothing to be compatible with.
6. **[Engineering]** Target platforms (macOS/Linux only? Windows)? Affects sandbox options (E4) and `pdflatex` handling.
7. **[Security]** Threat model for LLM-generated code (the paper reports generated `subprocess.run()` calls on the host, A.8 F6): trusted-machine convenience vs. untrusted-input hardening — decides whether R13 moves to P0.
8. **[Data]** Can cassettes include real API outputs from the maintainers' runs, or must they be synthesized (licensing/PII/cost)?
9. ~~**[Engineering]** `HFDataSearch` depends on embeddings~~ **Resolved by audit:** it is TF-IDF (E7); the embedding dependency is the AgentRxiv hub (E12). Open sub-question: is a Python embedding sidecar for the hub acceptable until Phase 8?
10. **[Engineering]** Should AgentRxiv's HTTP API stay wire-compatible with the current Flask server so existing clients/the public site keep working? (R10 assumes yes.)
11. **[Engineering]** Python 3.12 and the pins in `requirements.txt` (`openai==1.55.1`, `anthropic==0.39.0`, `tiktoken==0.8.0`, `arxiv==2.1.3`, `datasets==3.1.0`, `torch==2.5.1`, `transformers==4.46.3`, `keras==3.7.0`) are the baseline for the Pixi env (E10); `tensorflow`, `flask`, `flask_sqlalchemy`, `sentence-transformers` must be added. Is pinning to these exact versions for the `py-baseline` cassettes acceptable, with upgrades tracked as separate DVC experiments?

---

## 11. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| LLM nondeterminism makes parity hard to prove | High | Med | Replay-based parity (§5.3); live comparison only as a statistical check |
| Prompts drift subtly when ported | Med | High | Prompts as template files, byte-for-byte golden tests |
| Hidden behavior in Python (global state, `os.environ` mutation, implicit `from utils import *` in generated code, tuple-repr prompts, misspelled config keys) | High | Med | Characterization tests; the §7.2 deviation register; the runner prelude is specified and tested |
| Parity target is itself buggy: fixing D1/D2/D6 changes which model runs refinement and whether copilot/refinement happen at all, so live-run metrics will move | High | Med | Record `py-baseline` twice: as shipped, and with the deviation fixes applied in Python (`py-baseline-fixed`); Rust parity is measured against the latter |
| Actor framework regret | Low | Med | E1 decision, behind a thin `spawn`/`Handle` abstraction |
| Generated code imports a package missing from the locked env | High | Med | Detect `ModuleNotFoundError` as a typed `ExecResult::MissingDependency` fed back to the LLM; curated package set in `pixi.toml`; `Uv` fallback; decide in E10 |
| Dataset/embedding search not portable | Med | Low | Sidecar fallback (E7) |
| Rust-native experiments (Phase 7) cannot match Python ML ecosystem | High | High | E11 gates the phase; curated experiment SDK; narrow the supported experiment domain explicitly; keep Phases 0–6 independently valuable |
| DVC metric noise from LLM variance causes false regressions | Med | Med | Compare against Python's measured run-to-run band (Q4), N≥5 runs, thresholds in `params.yaml` |
| Scope creep ("while we're at it" features) | Med | High | Non-goals; any addition requires equal removal or schedule extension |

---

## Appendix A — Reference values from the original paper (arXiv 2501.04227v2, 17 Jun 2025)

Used as the benchmark to reproduce and track in DVC. Values are as stated in the paper's text; per-cell values only visible in figures (e.g. Fig. 5–9 tables) are to be transcribed from the PDF in Phase 0. Where the paper and the repository disagree, the repository is the parity oracle and the disagreement is noted here.

### A.1 Setup
- Backends evaluated: gpt-4o, o1-mini, o1-preview. Hardware: 2023 MacBook Pro, Apple M3 Max, 36 GB.
- 5 research topics × 3 backends = 15 autonomous papers; 10 volunteer PhD students each reviewed 3 randomly assigned papers.
- Co-pilot: o1-mini for all phases except literature review; each researcher produced 1 custom-topic and 1 preselected-topic paper; external researchers rated them.
- Hyperparameters (Table 1): lit review — 5 paper summaries, full-text history decay 3 steps, agent temp 0.8; data prep — timeout 120 s; experiments — mle-solver steps 3, repair attempts 2, max top codes 2, error history 5, code history 2, comparison trials 2, timeout 600 s, score temp 0.6, repair temp 0.8, initial code temp 1.0, solver temp 1.0; paper writing — paper-solver steps 5, max top papers 1, history 10, reviewers 1, comparison trials 2, solver temp 1.0, initial paper temp 0.8; refinement — 3 reviewers.
- Known inconsistency: Table 1 lists 2 code-repair attempts but §3.2 text says N_rep = 3. The repository value (A.10) is authoritative for parity; `params.yaml` records both the paper's and the code's value.
- Workflow shape (paper §3): three stages, seven subtasks measured in §4.3 (Literature Review, Plan Formulation, Data Preparation, Running Experiments, Results Interpretation, Report Writing, Report Refinement). Literature review retrieves the top 20 abstracts per `SUMMARY` query and finalises once N papers are added. mle-solver: sample a program from the top-N pool, apply `EDIT n m` or `REPLACE`, compile, repair up to N_rep times, score 0–1 with an LLM reward model over (plan, code, output), self-reflect on success or failure, and run N modifications per step keeping the best. paper-solver: scaffold with exactly eight sections (Abstract, Introduction, Background, Related Work, Methods, Experimental Setup, Results, Discussion), optional arXiv lookup per section, `EDIT`-driven refinement with a LaTeX compile check before each accepted edit, then NeurIPS-style review scoring. Refinement: three reviewer agents, then the PhD agent chooses to finish or return to planning, experiments or interpretation.
- Prompt skeleton (paper Appendix B, parity-critical for R4): system prompt = role description + phase prompt + command descriptions; user prompt = context prompt + `History:` lines (`Step #{step}, Phase: {phase}, Feedback: {feedback}, Your response: {model_resp}`) + `Current Step #{step}` + `Phase: {phase}` + a completion nag appended once the step count passes 70% of the phase cap (`You must finish this task and submit as soon as possible!`) + objective + `Feedback:` + `Notes:` + `Your previous command was: {prev_comm}. Make sure your new output is different.` + `Please produce a single command below:`. On a second pass after refinement a "previous experiments" block (prior code, results, interpretation, report, reviewer response) is prepended to the context.
- Command fences by phase (paper B.6): lit review `SUMMARY`, `FULL_TEXT`, `ADD_PAPER`; plan formulation `DIALOGUE`, `PLAN` (Postdoc submits); data preparation `python`, `DIALOGUE`, `SEARCH_HF` (ML Eng), `SUBMIT_CODE` (PhD; code must load a HuggingFace dataset); results interpretation `DIALOGUE`, `INTERPRETATION` (Postdoc submits); mle-solver `EDIT N M`, `REPLACE`, reward model answers in a `SCORE` fence; paper-solver `EDIT N M`, `REPLACE`; reviewer answers `THOUGHT:` then `REVIEW JSON:` with a `json` fence.

### A.2 The five research topics (fixture tasks for cassettes)
1. Do language models exhibit cognitive biases such as confirmation or anchoring bias?
2. Are image transformers more or less sensitive to pixel noise than convolutional networks?
3. Do language models improve accuracy on MedQA when asked to perform differential diagnosis?
4. Are language models sensitive to word order in multiple-choice benchmarks?
5. Does gender role play affect accuracy of language models answering math questions?

### A.3 MLE-Bench subset
10 low-complexity text/tabular Kaggle challenges; 80/20 train/dev split of the original training set; data supplied as numpy arrays; best-dev-score code evaluated once on the Kaggle test set. Baselines: MLAB (gpt-4o), OpenHands (gpt-4o), AIDE (o1-preview). Results to reproduce: mle-solver 4 medals (2 gold, 1 silver, 1 bronze), above human median on 6/10, valid submission on 10/10 within 2 h; OpenHands 2 medals / 2 of 10; AIDE 2 medals / 5 of 10; MLAB 0 / 0.

### A.4 Runtime, cost, reliability (gpt-4o / o1-mini / o1-preview)
| Quantity | gpt-4o | o1-mini | o1-preview |
|----------|--------|---------|------------|
| Total workflow time (s) | 1165.4 | 3616.8 | 6201.3 |
| Total workflow cost (USD) | 2.33 | 7.51 | 13.10 |
| Avg subtask success rate | 94.3% | 92.8% | 95.7% |
| Literature review success | 60% | 70% | 80% |
| Data preparation success | 100% | 80% | 90% |
| Running experiments time (s) | 417.8 | 2082.5 | 4036.2 |
| Report writing time (s) | 572.5 | 827.7 | 1854.2 |
| Report writing cost (USD) | 1.73 | 2.58 | 9.58 |
| Data preparation cost (USD) | 0.09 | 3.03 | 0.30 |
| Literature review time (s) / cost | 92.9 / 0.12 | 56.8 / 0.16 | — |
| Plan formulation time (s) / cost | 23.3 / 0.03 | — | 33.1 / 0.04 |

Context from the paper: prior autonomous workflow (The AI Scientist) ≈ $15 per paper with gpt-4o, i.e. 6.4× the gpt-4o cost here; the abstract states this as an 84% cost reduction. gpt-4o is 3.2× faster than o1-mini and 5.3× faster than o1-preview end to end. Per-phase cells not quoted in the text (the "—" cells above, plus results interpretation and report refinement for every backend, and all per-phase success rates other than those listed) are in Fig. 8 only and are transcribed in Phase 0.

### A.5 Human-evaluated quality, autonomous mode (1–5)
| Metric | gpt-4o | o1-mini | o1-preview |
|--------|--------|---------|------------|
| Experimental quality | 2.6 | 3.2 | 2.9 |
| Report quality | 3.0 | 3.2 | 3.4 |
| Usefulness | 4.0 | 4.3 | 4.4 |

Topic notes: word-order topic highest report quality (3.8) and usefulness (4.5) but, with image-noise, lowest experiment quality (2.7); cognitive-bias topic highest experiment quality (3.2); image-noise topic highest backend variance (experiment quality 1.5 gpt-4o vs 4.0 o1-mini; usefulness 2.5 vs 4.5). Per-topic × per-backend cells are in Fig. 5 only and are transcribed in Phase 0.

Survey wording (Appendix C of the paper, reproduced in A.9) is the schema for the human-eval CSV in R19.

### A.6 NeurIPS-style scores
- Human overall (1–10): gpt-4o 3.5, o1-mini 3.8, o1-preview 4.0. Human quality (1–4): gpt-4o 1.8, o1-mini 2.3; significance 2.2–2.5 across backends; soundness o1-preview 2.2, o1-mini 1.8, gpt-4o 1.7; mean contribution 2.1.
- Human clarity (1–4): gpt-4o 2.6, o1-mini 2.1. Presentation and contribution follow the same ordering (o1-preview best overall at 4.0/10).
- Automated overall 6.1 vs human 3.8 (−2.3); automated clarity 3.6 vs human 2.4; the gap "holds for all criterion".
- Reference: mean accepted NeurIPS 2024 paper 5.85 (paper §4.1.1 also rounds this to 5.9); autonomous human overall ≈ −2.05 below.
- Reviewer-agent provenance: the reviewer prompt and NeurIPS form are taken from The AI Scientist (Lu et al. 2024). Two calibration figures are quoted from that work, not measured here: on 500 ICLR 2022 OpenReview papers the LLM reviewer reached 65% accuracy vs 66% for humans and F1 0.57 vs 0.49 after calibration. The paper's own limitations section separately reports lower LLM self-evaluation agreement than humans, 53.3% vs 56.1%.

### A.7 Co-pilot results
- Tool quality (1–5), all: utility 3.5, continuation 3.75, satisfaction 3.63, usability 4.0. Custom: 3.75 / 4.0 / 3.75 / 3.75. Preselected: 3.25 / 3.5 / 3.5 / 4.25.
- Paper quality (1–5), all: experiment 2.38, report 3.13, usefulness 3.75; custom report 3.5, usefulness 4.0; preselected experiment 2.5.
- Co-pilot paper quality vs the o1-mini autonomous papers: report −0.07, usefulness −0.55, experiment quality −0.82 (attributed to difficulty steering the agents to the researcher's exact vision).
- NeurIPS overall: autonomous 3.8 → co-pilot self-eval 4.13 → external 4.38 (still −1.45 vs 5.85).
  - Self-eval vs autonomous: quality +0.13, clarity +0.48, soundness +0.35, presentation +0.33, significance −0.3, contribution −0.1.
  - External vs self-eval: quality +0.62, significance +0.25, overall +0.25, soundness +0.13, contribution +0.13, presentation +0.0, clarity −0.25. External reviewers rated preselected topics above custom (quality +0.5, significance +0.5); self-evaluators rated custom higher.
  - External co-pilot vs external autonomous: quality +0.75, soundness +0.48, overall +0.58, clarity +0.23, presentation +0.33, significance −0.05, contribution +0.03.
- Survey feedback response rate: 75%; requested improvements were a GUI, inspection of intermediate results, more figures, and a better literature review phase.

### A.8 Failure modes reported by the paper (§5.1–5.2)

These are parity-relevant behaviours and the seed list for the E4 hostile-snippet corpus and the deviation log. Each must be either reproduced, or recorded as an intentional deviation with a test.

| # | Observed behaviour | Where it lands in this spec |
|---|--------------------|-----------------------------|
| F1 | Lit review: agents repeat `SUMMARY` until the phase step cap and the phase terminates | R5 step caps; parity on the "max steps reached" path |
| F2 | Retrieved full texts and printed experiment output push some models past their token limit | R1 `clip_tokens`; R8 full-text cap; R6 output cap |
| F3 | mle-solver sometimes ends with 0% accuracy for every method and does not recover before steps run out | A.1 prompt line "MUST not get 0% accuracy" kept byte-for-byte; metric recorded |
| F4 | mle-solver edits line 0 far more than other lines, so `REPLACE` compiles more often than `EDIT` | R1 `Edit` semantics; metric: EDIT vs REPLACE success rate |
| F5 | Generated code calls `exit()`, which killed the whole Python process; detected and removed manually | R6 `ExecResult::Rejected`; E4 corpus |
| F6 | Generated code runs host commands via `subprocess.run()` | R6/R13 sandbox; E4 corpus; Open Q 7 |
| F7 | paper-solver arXiv searches could loop up to ~100 tries; a cap of 5 attempts was added | R7/R8 search-attempt cap of 5 |
| F8 | Only two figures (`Figure_1.png`, `Figure_2.png`) are carried into the paper; no repository-level file management | Non-goal (parity); P2 |
| F9 | Hallucinated experimental details in weaker backends' reports | Human-eval rows (R19) are the only detector; no automated check promised |

### A.9 Survey schema (paper Appendix C) for the human-eval CSV (R19)

All ratings are integers. One row per (paper, rater, mode).

| Column | Scale | Question (abridged) |
|--------|-------|----------------------|
| `experimental_quality` | 1–5 | Perceived quality of the experimental results |
| `report_quality` | 1–5 | Perceived quality of the report writing |
| `usefulness` | 1–5 | Usefulness of an AI tool that produced this report |
| `quality`, `clarity`, `significance` | 1–4 | NeurIPS criteria (low … very high) |
| `soundness`, `presentation`, `contribution` | 1–4 | NeurIPS criteria (poor … excellent) |
| `overall` | 1–10 | NeurIPS overall (very strong reject … award quality) |
| `confidence` | 1–5 | Reviewer confidence |
| `decision` | Accept/Reject | Binary decision |
| `usability`, `satisfaction`, `utility`, `continuation` | 1–5 | Co-pilot only: ease of building, enjoyment, usefulness for research, likelihood of reuse |
| `topic`, `mode`, `rater_kind` | text | Topic (A.2 index or custom text); `autonomous` / `copilot-custom` / `copilot-preselected`; `self` / `external` |
| `feedback` | text, optional | "How could Agent Laboratory be improved for your research?" |

The automated reviewer emits the same NeurIPS fields plus `originality` (1–4), `summary`, `strengths`, `weaknesses`, `questions`, `limitations`, and `ethical_concerns` (bool); these are stored per review so human and automated scores diff on identical columns.

### A.10 Paper vs. repository: hyperparameter reconciliation

The repository is the parity oracle. Where the paper's Table 1 and the code differ, `params.yaml` carries both values; replay parity uses the code value, and the paper value is available as a named DVC experiment.

| Hyperparameter | Paper (Table 1) | Repository (file:line) |
|----------------|-----------------|------------------------|
| Lit review paper summaries | 5 | 5 (`arxiv_num_summaries`, `ai_lab_repo.py:55`); `num-papers-lit-review` default 5 |
| Full-text history decay | 3 steps | 3 (`arxiv_paper_exp_time`, `:47`, emitted as `EXPIRATION 3`) |
| Lit review agent temperature | 0.8 | **0.4** (`:473, :524`) |
| Data preparation timeout | 120 s | **60 s** for `SUBMIT_CODE` (`:369`), 120 s for ML-engineer `python` blocks (`:403`) |
| mle-solver steps | 3 | `mlesolver-max-steps` default 3 (`initial_solve` + 2 `solve` calls) |
| Code repair attempts | 2 (text says 3) | 2 (`GLOBAL_REPAIR_ATTEMPTS`, `mlesolver.py:25`) |
| Maximum top codes | 2 | **1** (`max_codes`, `:213`) |
| Error history length | 5 | 5 (`:260`) |
| Code history length | 2 | 2 (`st_hist_len`, `:214`) |
| Comparison trials (mle) | 2 | **1** (`min_gen_trials`, `:215`) |
| Experiment timeout | 600 s | 600 s (`execute_code` default, `tools.py:306`) |
| Score temperature | 0.6 | 0.6 (`:157`) |
| Repair temperature | 0.8 | 0.8 for `REPLACE`, **0.2** for `EDIT` (`:179, :199`) |
| Initial code temperature | 1.0 | 1.0 (`:267`) |
| Solver temperature | 1.0 | 1.0 (`:289`) |
| paper-solver steps | 5 | `papersolver-max-steps` default 5 (`initial_solve` + 5 `solve` calls) |
| Maximum top papers | 1 | 1 (`papersolver.py:260`) |
| Paper history length | 10 | 10 (`:261`, never used in a prompt) |
| Reviewers (paper-solver) | 1 | 1 (`get_score` once per candidate) |
| Comparison trials (paper) | 2 | 2 (`min_gen_trials`, `:262`) |
| Solver temperature (paper) | 1.0 | 1.0 (`:280`) |
| Initial paper temperature | 0.8 | 0.8 (`:369`) |
| Reviewers (refinement) | 3 | 3 (`ReviewersAgent`, `agents.py:184-201`) |
| Phase step cap | not stated | 100 (`max_steps`, `ai_lab_repo.py:20`), completion nag above 70% |
| Default backend | gpt-4o / o1-mini / o1-preview evaluated | `o3-mini` (`DEFAULT_LLM_BACKBONE`, `:13`) |
| arXiv results per query | 20 | 5 in the lit review, 10 in paper-solver (20 is the unused default) |
